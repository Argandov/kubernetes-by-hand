# Part 1 — etcd alone

*Kubernetes By Hand*

Kubernetes has a reputation for dizzying complexity. So it is worth sitting with a strange fact:
the one thing every other part of Kubernetes trusts completely — the single source of truth for
the entire cluster — is **just a key-value store**. Put a key, get a key. That's the shape of it.

In this chapter there is **no Kubernetes at all**. We run `etcd` by itself on `cp` and poke it
with your bare hands until it holds no mystery. Everything Kubernetes does later — scheduling,
self-healing, "watching" for changes — is built on the handful of primitives you're about to
touch directly. Meet them now, alone and unhurried, and the rest of the course has solid ground
to stand on.

> ⏱️ **Estimated time:** 60–90 min · **Difficulty:** conceptually dense, low-friction

---

## Priming questions

Read these. Do **not** look anything up. Let your mind make guesses — the guesses are the point.

1. A key-value store "remembers keys and values." So does a plain hashmap. So does Redis. What
   could etcd have that those don't, that a system whose whole job is *remembering the desired
   state of a cluster* specifically needed?
2. Later, when you run `kubectl get pods`, some bytes are physically read from somewhere. **From
   where?** And is that "where" the same machine `kubectl` is running on?
3. etcd offers three features named **watch**, **lease**, and **revision**. Before reading their
   definitions: for a system whose job is *remember what I want, and notice when reality drifts
   from it*, guess what each of those three might be **for**.
4. Suppose etcd stops on a fully running cluster. What do you think **freezes** — and what do you
   think **keeps running anyway**? (This one is a trap. The answer teaches you the entire shape of
   Kubernetes, so commit to a guess before you read on.)
5. Two different controllers try to update the same object at the same instant. How might a plain
   key-value store help them avoid silently clobbering each other?

---

## Assumed state

You should be resuming from a passed **Part 0** gate.

- You are working on the **`cp`** node (this is where the cluster's database lives).
- `clean-base` snapshots exist for all nodes.
- `cp` can reach the internet (we download one binary).

Quick check on `cp`:

```bash
hostname                       # -> cp
ping -c1 -W2 deb.debian.org    # internet OK
```

If either is wrong, return to Part 0.

---

## The mental model

An ordinary hashmap forgets everything when the process dies, lives in one program's memory, and
lets the last writer silently win. A cluster's brain can afford none of that. etcd is a
key-value store hardened with exactly the properties such a brain needs:

- **Durable & consistent.** Data survives restarts, and etcd is built to give every reader the
  same answer rather than stale or conflicting ones. (etcd achieves this with a consensus
  algorithm called Raft across multiple etcd members. In this lab we run a **single** member, so
  consensus is trivial — but keep in mind that in a real cluster the "source of truth" is itself
  a small replicated cluster, precisely so losing one machine doesn't lose the brain.)
- **Watchable.** A client can say "tell me the instant anything under this key changes" and etcd
  *pushes* the change. No polling. **This is the single most important etcd feature for
  understanding Kubernetes** — hold onto it.
- **Leased (TTL).** A key can be tied to a lease that expires unless refreshed, so the key
  *deletes itself* when whoever owned it stops checking in.
- **Versioned (MVCC).** Every change bumps a global **revision** counter, and old revisions are
  retained (until compaction). etcd remembers not just the current value but the *history*.
- **Transactional.** "Write this **only if** that condition still holds" — atomic compare-and-set.

Read that list again through Kubernetes-colored glasses and you can almost see the whole system:
desired state lives here durably; components *watch* for changes instead of polling; nodes prove
they're alive with *leases*; concurrent writers stay safe via *revisions* and *transactions*.
We're going to touch every one of these by hand.

---

## The build

### Step 1 — Install a single etcd

We fetch the official upstream binary rather than a distro package — it's the same artifact real
clusters use, it's version-pinned and reproducible, and it sidesteps packaging quirks. On `cp`:

```bash
# Pick a recent etcd v3.5.x release. This manual uses v3.5.16 as a concrete example;
# any recent v3.5.x works. (amd64 build — see the architecture note in Part 0.)
ETCD_VER=v3.5.16

cd /tmp
curl -LO "https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz"
tar xzf "etcd-${ETCD_VER}-linux-amd64.tar.gz"
sudo cp "etcd-${ETCD_VER}-linux-amd64/etcd"    /usr/local/bin/
sudo cp "etcd-${ETCD_VER}-linux-amd64/etcdctl" /usr/local/bin/

etcd --version
etcdctl version
```

You now have two binaries: **`etcd`** (the server — the database itself) and **`etcdctl`** (a
command-line client that talks to it). We'll run the server in one pane and the client in another.

### Step 2 — Run etcd in the foreground and watch it think

Open a `tmux` session (or two SSH windows). In the **first** pane, start etcd in the foreground so
you can see its log:

```bash
cd ~
etcd
```

Read the startup log instead of ignoring it. Two lines matter enormously:

- It creates a data directory named **`default.etcd/`** in your current folder. **That directory
  is the cluster's entire memory.** Everything Kubernetes will ever know ends up as files in a
  directory just like this one. Sit with that — the "brain" is a folder on a disk.
- It reports listening for clients on **`127.0.0.1:2379`**. Port **2379** is etcd's client port;
  memorize it, you'll see it constantly. (You'll also see `2380` mentioned — that's the port etcd
  members use to talk to *each other*. With one member it's unused, which itself tells you
  something: peer traffic only matters once the brain is replicated.)

Leave etcd **running** in this pane. In the **second** pane, confirm the client can reach it:

```bash
etcdctl endpoint health
```

Expected: something like `127.0.0.1:2379 is healthy`. If so, you are now talking to the future
brain of your cluster — which currently knows nothing. Let's teach it something.

> All `etcdctl` commands below run in the **second** pane. Leave etcd running in the first.

### Step 3 — put / get (the whole store, in two verbs)

```bash
etcdctl put /demo/greeting "hello"
etcdctl get /demo/greeting
etcdctl get /demo/greeting --print-value-only
```

That's the entire core of a key-value store: `put` a value at a key, `get` it back. Note the key
looks like a filesystem path (`/demo/greeting`). It isn't a filesystem — but the *convention* of
slash-separated hierarchical keys is exactly how Kubernetes organizes everything. Foreshadow: in
Part 2 you'll watch real Kubernetes objects appear as keys under **`/registry/...`** in this very
store.

Write a few keys and list a whole range by prefix:

```bash
etcdctl put /demo/fruit/apple  "red"
etcdctl put /demo/fruit/banana "yellow"
etcdctl get /demo/fruit/ --prefix                 # every key under the prefix
etcdctl get /demo/fruit/ --prefix --keys-only     # just the keys
```

`--prefix` returns an entire key range at once. Keep it in mind: "list all pods" will later be, at
bottom, exactly this — a prefix read over a key range.

### Step 4 — watch (the feature the whole system is built on)

This is the important one. In your **second** pane, start watching a key range and leave it
blocking:

```bash
etcdctl watch /demo/fruit/ --prefix
```

It hangs, waiting. Now open a **third** pane (or another SSH window) and change something:

```bash
etcdctl put /demo/fruit/apple "green"
etcdctl put /demo/fruit/cherry "dark-red"
etcdctl del /demo/fruit/banana
```

Watch the second pane: each change **appears instantly**, pushed to you — `PUT` events with the
new values, a `DELETE` event for the removed key. You never asked "did anything change?" in a
loop. etcd *told* you, the moment it happened.

Stop the watch with `Ctrl-C`.

**Why this is the crucial primitive.** Almost every Kubernetes component is, in essence, a program
that opens a watch and reacts to what streams in. The scheduler watches for Pods with no assigned
node. The kubelet on each worker watches for Pods assigned to *it*. Controllers watch the objects
they own. None of them poll. They all lean on exactly the push-on-change behavior you just saw —
so the entire cluster reacts to change in near real-time without hammering the database. When you
meet "informers" and "watches" later, remember: it's this.

### Step 5 — lease (a key that deletes itself)

A **lease** is a timer. Attach a key to a lease and the key lives only as long as the lease is
kept alive; let the lease lapse and the key **vanishes on its own**.

```bash
# Grant a 15-second lease. etcd prints a lease ID (a hex number) — copy it.
etcdctl lease grant 15
# -> lease 694d7... granted with TTL(15s)

# Attach a key to that lease (use YOUR lease id):
etcdctl put --lease=694d7... /demo/heartbeat "worker-1 alive"

etcdctl get /demo/heartbeat        # present now
```

Now **do nothing** for ~15 seconds (watch the clock), then:

```bash
etcdctl get /demo/heartbeat        # gone — empty output
```

The key deleted itself because nothing renewed its lease. Now see the *other* half — keeping a
lease alive:

```bash
etcdctl lease grant 15
# copy the new id, then:
etcdctl put --lease=<new-id> /demo/heartbeat "worker-1 alive"
etcdctl lease keep-alive <new-id>     # streams a refresh every few seconds; blocks
```

While that `keep-alive` runs in one pane, `etcdctl get /demo/heartbeat` in another keeps returning
the value — indefinitely — because the lease is being refreshed. `Ctrl-C` the keep-alive, wait
15s, and the key disappears again.

**Why this matters.** This is exactly how a cluster tracks liveness. A worker node will hold
something like a lease and continuously refresh it to say "I'm still here." The moment the node
dies or is partitioned, it stops refreshing, the lease lapses, and the system *automatically*
concludes the node is gone — which is what later triggers its workloads to be rescheduled
elsewhere. "Node NotReady" is, underneath, a lease that stopped being renewed. The same mechanism
underlies **leader election**: whoever holds the lease is the leader; if they die, the lease frees
and someone else grabs it.

### Step 6 — revision (etcd remembers history)

Every write to etcd — anywhere, to any key — increments one **global revision** counter. Old
revisions are kept (until compacted), so etcd can show you the past.

```bash
etcdctl put /demo/config "v1"
etcdctl put /demo/config "v2"
etcdctl put /demo/config "v3"

etcdctl get /demo/config                 # v3 (current)
etcdctl get /demo/config -w json         # look at the numbers
```

In the JSON, note the fields on the key: **`create_revision`** (the revision when this key was
first created), **`mod_revision`** (the revision of its most recent change), and **`version`**
(how many times *this key* has been written). Distinguish two counters: the **global revision**
advances on every change to the whole store; a key's **version** counts writes to that one key.

Now read the *past* by asking for an earlier global revision:

```bash
# Find a revision number from the JSON above (mod_revision of an earlier put),
# or just try a small number you saw, e.g.:
etcdctl get /demo/config --rev=<an-earlier-revision>    # shows the older value
```

You just read a value that no longer exists as "current." etcd is a little time machine.

**Why this matters.** This versioning is the ancestor of Kubernetes' **`resourceVersion`**, the
number stamped on every object. Two payoffs you'll cash in later: (1) a client can watch *starting
from a specific revision*, so if it disconnects it resumes exactly where it left off without
missing or double-processing events — that's how watches survive network blips; (2) it's the basis
of safe concurrent updates, next step.

### Step 7 — transaction (safe updates without clobbering)

Two writers, same key, same instant: how do you stop one from silently overwriting the other? With
a **transaction** that says *write only if the value hasn't changed since I read it.* etcd's `txn`
does compare → then-succeed → else-fail, atomically.

```bash
etcdctl put /demo/counter "1"

# Interactive transaction:
etcdctl txn -i
```

At the prompts, express: *"compare: value of `/demo/counter` equals `1`; if true: put
`/demo/counter` `2`; if false: get `/demo/counter`."* A minimal session looks like:

```
compares:
value("/demo/counter") = "1"

success requests (get, put, del):
put /demo/counter "2"

failure requests (get, put, del):
get /demo/counter
```

Run it once → the compare holds → it writes `2` (SUCCESS). Run the **exact same** transaction
again → now the value is `2`, not `1`, so the compare fails → it takes the failure branch instead
(FAILURE). You just performed a **compare-and-swap**: the update only applied while your assumption
about the current state was still true.

**Why this matters.** This is precisely how the Kubernetes API server prevents two controllers
from clobbering each other. Each update is conditional on the object's `resourceVersion` being
unchanged; if someone else wrote first, your update is rejected and you must re-read and retry.
"Optimistic concurrency," and it's this `txn` primitive underneath. Conflict-safety in a system
with dozens of concurrent loops falls out of one key-value-store feature.

### Step 8 — answer the trap (what if etcd dies?)

Priming Q4. You have etcd running in pane one. Go there and stop it with `Ctrl-C`. In pane two:

```bash
etcdctl endpoint health      # now fails — the brain is unreachable
```

Reason it through with what you now know. If this were a *live* cluster and only etcd died:

- **What freezes:** anything that needs to *read or change desired state*. No new scheduling, no
  updates, no self-healing, and `kubectl` writes fail — because the loops can't reach the database
  they compare against.
- **What keeps running anyway:** the programs **already running** on the workers. Their containers
  don't ask etcd for permission to keep executing; the component on each worker already knows what
  it was told to run and just... keeps running it.

That gap is the deepest structural fact in Kubernetes: the **control plane** (database + loops
that decide) is *decoupled* from the **data plane** (your actual running programs). The brain can
go dark and the body keeps standing — it just stops *adapting*. Internalize this and half of
Kubernetes' behavior under failure becomes predictable instead of surprising.

Restart etcd (`etcd` in pane one) before moving on, and confirm `etcdctl endpoint health` is happy
again.

---

## Verification gate

You pass Part 1 when you can do the following **without looking back at the steps** — the doing is
the proof you understood, not memorized:

1. Start `etcd`, and from another pane confirm `etcdctl endpoint health` reports healthy.
2. `put` a key and `get` it back; list a set of keys by `--prefix`.
3. Run a `watch` on a prefix in one pane and cause a `PUT` and a `DELETE` to appear in it live
   from another pane.
4. Create a key on a short lease, and **explain out loud, correctly**, why it disappears — and
   demonstrate keeping it alive with `keep-alive`.
5. Show that writing the same key three times advances revisions, and read a previous value with
   `--rev`.
6. State, in one or two sentences each, **what freezes and what keeps running** if etcd dies on a
   live cluster — and *why*.

If #4 and #6 come out fluently in your own words, you have the core of this chapter. That fluency —
not command recall — is the gate.

A quick self-test one-liner (watch + live change), if you want a scripted confirmation:

```bash
# pane A:
etcdctl watch /gate/ --prefix
# pane B:
etcdctl put /gate/x 1 && etcdctl put /gate/y 2 && etcdctl del /gate/x
# pane A should show: PUT /gate/x, PUT /gate/y, DELETE /gate/x
```

---

## Troubleshooting

- **`etcdctl` says `context deadline exceeded` or connection refused.** The server isn't running
  or isn't reachable. Confirm `etcd` is still up in its pane (foreground process — did it get
  `Ctrl-C`'d or did the pane close?), and that it's listening on `127.0.0.1:2379`
  (`ss -ltnp | grep 2379`).
- **etcd won't start: "listen tcp 127.0.0.1:2379: bind: address already in use."** A previous etcd
  is still running. Find and stop it: `ss -ltnp | grep 2379`, then `kill` the PID (or reuse that
  running instance).
- **etcd won't start: permission denied / cannot create `default.etcd`.** You're in a directory
  you can't write to. `cd ~` and start etcd from your home directory.
- **`etcdctl` behaves oddly / flags rejected.** This course assumes the **v3** API, which is the
  default in etcd v3.4+. If you're on an unusually old build, prefix commands with `ETCDCTL_API=3`.
  Confirm with `etcdctl version` (the "API version" line).
- **Lease key didn't disappear.** A `keep-alive` you started earlier is still running in some pane
  and refreshing it, or you `get` a *different* key than you leased. Check panes; re-check the key
  path.
- **You've made a mess of `/demo` keys and want a clean slate.** `etcdctl del "" --from-key=true`
  wipes all keys (fine in this throwaway lab), or simply stop etcd, delete the `default.etcd/`
  directory, and start fresh — vivid proof that that directory *is* the entire state.

---

## What just happened (close the loop)

You ran the cluster's future brain in isolation and touched, by hand, every primitive the rest of
Kubernetes is built on:

- **put/get/prefix** — the store itself; "list all pods" is a prefix read.
- **watch** — push-on-change; the reason every component reacts in real-time without polling.
- **lease** — self-deleting keys; the mechanism behind node liveness and leader election.
- **revision (MVCC)** — history and `resourceVersion`; resumable watches and safe retries.
- **txn** — compare-and-swap; optimistic concurrency that keeps concurrent loops from clobbering.
- **the decoupling** — kill etcd and the control plane freezes while running programs carry on:
  brain vs. body, the single most clarifying fact in the system.

Notice what's still missing. There's no nice API here — you're speaking etcd's raw key-value
dialect. There's no notion of a "Pod" or a "Deployment," no permissions, no validation: etcd will
store any bytes at any key. Something has to sit *in front of* this store to impose structure,
identity, authorization, and a clean interface — and to be the *only* thing allowed to touch etcd
directly. That something is the **API server**, and it's Part 2. You'll talk to it with `curl`
before you're allowed a single `kubectl` command — and you'll watch the objects you create land as
keys in exactly this store.

---

## Lab log

Snapshot each node as `part-01-etcd` (or at minimum snapshot `cp`), and add a row to `LAB_LOG.md`:

```
| 2025-XX-XX | 01 | Y | part-01-etcd | ran etcd solo; watch/lease/revision/txn by hand; understood control/data-plane split |
```

---

**Next → Part 2 — kube-apiserver** *(coming next release)*

**[← Part 0](00-the-lab.md)** | **[Index](README.md)** | **[→ Part 2](02-apiserver.md)**
