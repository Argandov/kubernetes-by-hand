# Part 0 — The Lab

*Kubernetes By Hand*

Before any Kubernetes, we build the ground it stands on: three Linux machines, a discipline
for not losing your work, and a mental model to hang everything else on. This chapter installs
**nothing** Kubernetes-related — on purpose. Every prerequisite Kubernetes needs (disabling
swap, loading a kernel module, opening a port) will be introduced later, in the chapter of the
component that actually needs it, so you learn *why* it's required instead of copying a setup
script you don't understand.

> ⏱️ **Estimated time:** 45–75 min · **Difficulty:** mechanical

---

## How to read these manuals

You'll do this in every chapter, so here's the rhythm once:

- **Read the priming questions first and do not look anything up.** They're supposed to be
  unanswerable right now. Just let your mind guess. That's the mechanism working.
- **Run the assumed-state check** to confirm your lab is where the chapter expects.
- **Read the mental model**, then **do the build** with a terminal open beside this text.
- **Hit the verification gate.** Its output is your proof. Passed = done.
- **Snapshot, and write one line in `LAB_LOG.md`.**

Rule for when things break: if you're stuck for more than ~30 minutes on something this manual
didn't warn you about, **restore your last snapshot and redo the section** rather than grinding.
The environment is disposable by design; use that.

---

## Priming questions

Let these sit. You'll answer them by the end.

1. Kubernetes can run on one machine. So why would a course insist on **three**? What is it
   that a single machine physically *cannot* demonstrate about how Kubernetes works?
2. Two of these three machines have the identical job ("worker"). Why two and not one? What
   becomes visible with a second identical worker that a single worker hides?
3. We're about to make all three machines **clones of one template**. Beyond saving time, why
   might "these nodes are cheap and identical and re-creatable in seconds" be a *learning*
   decision and not just a convenience?
4. We will install nothing Kubernetes-related in this chapter. Why might *withholding* the
   setup be better teaching than front-loading it?

---

## The one-sentence mental model

Hold this in your head for the whole course; everything else is detail hung on it:

> **Kubernetes is a database plus a set of loops that work to make reality match the database.**

The database remembers what you *want* ("I want 3 copies of this program running"). The loops
are programs that constantly compare want-versus-reality and take action to close the gap ("only
2 are running — start one more"). That's it. That's the whole shape. The database is `etcd`
(Part 1). The first loop you'll meet is the scheduler (Part 4); the loops' true home is the
controller-manager (Part 6). Every mysterious Kubernetes behavior you've ever seen is some
version of *a loop noticing reality drifted from the database and correcting it.*

Keep asking, chapter to chapter: **where's the database here, and which loop is acting?**

---

## Why three nodes (answering priming Q1 and Q2 up front, because it shapes the build)

A single-node cluster teaches you a *degenerate* version of Kubernetes — one where the most
important behaviors are invisible because they're trivial:

- **Scheduling becomes a non-choice.** The scheduler's job is deciding *which machine* runs a
  workload. With one machine there is no decision, so you never see the mechanism. With **two**
  workers, the scheduler makes a real choice you can observe and influence.
- **Networking becomes fake.** The hard, interesting problem is a program on machine A talking to
  a program on machine B across a real network. On one machine everything is loopback and the
  genuinely tricky part never happens. Two workers force *real* cross-node traffic.
- **Resilience becomes unobservable.** A core idea is "drain a machine and watch its work flee to
  another machine." That needs somewhere for the work to flee *to*.

So: **one control-plane node** (the brain — later it runs the database and the loops) and
**two worker nodes** (the muscle — where your programs actually run). Three total.

The **control-plane** node gets more RAM (4 GB) because it will eventually run the database plus
several coordinating processes. The **workers** are lean (2 GB) because early on they run almost
nothing. This isn't a production sizing; it's the smallest lab where nothing important is hidden.

---

## The lab contract (vendor-neutral)

Build this however you like — a type-1 hypervisor, `libvirt`/`virt-manager`, VirtualBox, or three
small cloud instances. The manuals never assume a specific platform. What they *do* assume:

| Property        | Requirement                                                              |
|-----------------|--------------------------------------------------------------------------|
| Nodes           | 3 total: `cp` ×1, `worker` ×2                                             |
| OS              | Debian 12 (bookworm), **x86_64** (amd64)                                  |
| control-plane   | 2 vCPU, 4 GB RAM, 20 GB disk                                              |
| each worker     | 2 vCPU, 2 GB RAM, 20 GB disk                                              |
| Network         | one flat L2 segment; all nodes on the same subnet                        |
| Reachability    | every node can reach every other node **and** the internet              |
| Access          | SSH to each, and a user with `sudo`                                       |
| **Snapshots**   | your platform can snapshot/restore a VM — **non-negotiable** (this is the safety net) |

> **Architecture note.** This course targets **x86_64/amd64**. If your nodes are ARM (e.g.
> Apple silicon, a Pi cluster), the *concepts* are identical but every binary download URL in
> later chapters differs (`arm64` instead of `amd64`). Pick one architecture and keep all three
> nodes on it.

We'll refer to the machines by these hostnames throughout:

| Hostname   | Role          | Example IP (yours will differ) |
|------------|---------------|--------------------------------|
| `cp`       | control-plane | `10.0.0.10`                    |
| `worker-1` | worker        | `10.0.0.11`                    |
| `worker-2` | worker        | `10.0.0.12`                    |

Write your three real IPs down now — you'll use them constantly.

---

## The build

### Step 1 — A reusable Debian 12 template

The single most valuable habit in this course: **build one golden image, clone it three times.**
Identical nodes remove a whole class of "works on one, not the other" confusion, and — more
importantly — make every node **disposable**. A node you can recreate in under a minute is a node
that can never trap you in a debugging spiral. That's priming Q3: cheapness and sameness are a
*learning* decision, because they make failure free.

Create one Debian 12 VM to serve as the template:

- Minimal install — **no desktop**. You want a plain headless server.
- Create a non-root user with `sudo`.
- Install SSH so you can reach it, plus a few tools you'll want everywhere.
- **Do not** hardcode a hostname or a static per-node identity into the template itself; each
  clone will get its own. (If your platform has cloud-init, use it to set hostname/network per
  clone. If not, you'll set them by hand in Step 3 — equally fine for a lab.)

On the template, install the baseline tools we rely on and update:

```bash
sudo apt-get update && sudo apt-get -y upgrade
sudo apt-get -y install \
  curl wget vim tmux jq \
  ca-certificates gnupg lsb-release \
  net-tools iproute2 dnsutils tcpdump

sudo apt-get -y autoremove
sudo apt-get clean
```

What each earns its place for: `curl`/`wget` (fetch component binaries later), `jq` (read API
JSON — heavily used from Part 2), `tmux` (run a component in the foreground in one pane while you
poke it in another — essential when we run things by hand), `tcpdump`/`iproute2`/`net-tools`/
`dnsutils` (see the network with your own eyes in Part 7). That's the whole toolkit; we add
component binaries only when their chapter needs them.

Shut the template down cleanly once it's prepared.

### Step 2 — Clone three nodes

From the template, create three clones. Give them the sizes from the contract:

- `cp` — 2 vCPU / 4 GB / 20 GB
- `worker-1` — 2 vCPU / 2 GB / 20 GB
- `worker-2` — 2 vCPU / 2 GB / 20 GB

A **full/independent clone** (not a linked clone tied to the template) is safest for a learning
lab, so deleting or rebuilding the template later can't disturb your nodes.

### Step 3 — Give each node its identity

On each clone, set the hostname (skip if cloud-init already did):

```bash
# on cp:
sudo hostnamectl set-hostname cp
# on worker-1:
sudo hostnamectl set-hostname worker-1
# on worker-2:
sudo hostnamectl set-hostname worker-2
```

Confirm each node has a **stable IP** on the shared subnet. A DHCP reservation or a static lease
is ideal — you do not want a node's address changing between sessions, because later chapters
bake IPs and certificate names into config. Note the three IPs.

### Step 4 — Make the nodes resolve each other by name

Kubernetes components refer to nodes by name; you'll want to as well. Without a DNS server in the
lab yet, the simplest correct thing is a shared `/etc/hosts`. **On all three nodes**, append your
real IPs:

```bash
sudo tee -a /etc/hosts >/dev/null <<'EOF'

# --- kubernetes-by-hand lab ---
10.0.0.10  cp
10.0.0.11  worker-1
10.0.0.12  worker-2
EOF
```

Replace the addresses with yours. (When we build CoreDNS in Part 7 you'll understand exactly what
this static file is standing in for — and why real clusters don't rely on it.)

### Step 5 — Prove the lab is wired correctly

From `cp`:

```bash
ping -c1 worker-1 && ping -c1 worker-2      # name resolution + L2 reachability
ping -c1 1.1.1.1                            # internet reachability (raw IP)
ping -c1 deb.debian.org                     # internet + DNS working
```

From each worker, ping `cp` by name. Every ping should succeed. If any fails, fix it now —
everything downstream assumes this works. See Troubleshooting.

### Step 6 — Snapshot the clean baseline

This is the habit that makes the whole course safe. On **each** node, take a snapshot named
exactly:

```
clean-base
```

`clean-base` is your floor. No matter how badly a later chapter goes, you can always return three
nodes to this known-good, tool-equipped, freshly-networked state in seconds. **You will snapshot
at the end of every chapter from here on**, each with the chapter's name — building a ladder of
restore points.

---

## Verification gate

You pass Part 0 when **all** of these are true. This is binary.

1. Three nodes exist: `cp`, `worker-1`, `worker-2`, each with the right sizing.
2. From `cp`, `ping -c1 worker-1` and `ping -c1 worker-2` both succeed **by name**.
3. From each worker, `ping -c1 cp` succeeds **by name**.
4. From every node, `ping -c1 deb.debian.org` succeeds (internet + DNS).
5. `hostname` on each node prints the expected name.
6. A snapshot named `clean-base` exists for **each** node.

One combined check to run on `cp`:

```bash
hostname && \
for h in worker-1 worker-2; do ping -c1 -W2 "$h" >/dev/null && echo "$h OK" || echo "$h FAIL"; done && \
ping -c1 -W2 deb.debian.org >/dev/null && echo "internet OK" || echo "internet FAIL"
```

Expected output: `cp`, then `worker-1 OK`, `worker-2 OK`, then `internet OK`. Run the equivalent
worker→`cp` ping on each worker. When all green **and** all three `clean-base` snapshots exist,
the gate is passed.

---

## Troubleshooting

- **`ping worker-1` fails with "Name or service not known".** DNS/hosts problem, not a network
  problem. The `/etc/hosts` entry is missing or misspelled on the node you're pinging *from*.
  Re-check Step 4 on that node; confirm with `getent hosts worker-1`.
- **`ping worker-1` resolves but times out.** Name resolved, packets don't arrive — an L2/network
  issue. Confirm both nodes are on the same subnet (`ip addr` — same network prefix), the IPs
  match what's in `/etc/hosts`, and no host firewall is dropping ICMP. Two VMs on different
  virtual networks is the classic cause.
- **`ping 1.1.1.1` works but `ping deb.debian.org` fails.** Raw internet is fine; DNS is not. Your
  node has no working resolver. Check `cat /etc/resolv.conf` has a reachable `nameserver`.
- **Clone came up with the template's hostname / a duplicate IP.** You skipped or mis-ran Step 3,
  or DHCP handed two clones the same lease. Set the hostname explicitly; give each node a distinct
  reserved address.
- **You changed something and now a node is misbehaving in a way you can't explain.** This is
  exactly what `clean-base` is for. Restore it and redo the affected steps. Do not grind.

---

## What just happened (close the loop on the priming questions)

- **Q1/Q2 — why three, why two workers:** so scheduling is a real choice, cross-node networking
  is real traffic, and resilience is observable. A single node hides all three.
- **Q3 — clones as a learning decision:** identical, disposable nodes make failure *free*, which
  is what defeats the debugging death-spiral. Cheap to destroy = safe to experiment.
- **Q4 — withholding the setup:** every Kubernetes prerequisite arrives with the component that
  needs it, so you'll understand *why* swap is disabled or a kernel module is loaded instead of
  trusting a script. Understanding compounds; copied setup evaporates.

You now have the database's future home (`cp`) and the muscle (`worker-1`, `worker-2`) — but no
Kubernetes, no database, nothing running. In Part 1 we put the very first thing on `cp`: `etcd`,
alone, and discover that the foundation the entire system trusts is *just a key-value store*.

---

## Lab log

Add a row to `LAB_LOG.md`:

```
| 2025-XX-XX | 00 | Y | clean-base | 3 nodes up, name resolution + internet verified |
```

---

**Next → [Part 1 — etcd alone](01-etcd.md)**
