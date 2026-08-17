# Part 2 — kube-apiserver

*Kubernetes By Hand*

In Part 1 you spoke to etcd in its raw dialect: bytes at keys, no structure, no rules, no
identity. etcd will happily store `"potato"` at `/registry/pods/whatever` and never object,
because it has no idea what a Pod is. Something has to sit *in front of* that store and impose
everything etcd lacks: a real API with typed objects, validation, identity, permissions, and — the
feature you'll care about most — a **watch endpoint** that hands clients the same push-on-change
power you felt in Part 1, but safely.

That something is the **kube-apiserver**, and it is the center of the entire system. Say it once
and let it sink in: **the API server is the only component that ever talks to etcd directly.**
Everything else — every controller, every worker, every `kubectl` — talks to the *API server*.
Learn this one process well and the rest of Kubernetes is variations on "another client of the API
server."

In this chapter you'll run the API server on `cp`, pointed at the etcd you already know, and talk
to it with nothing but `curl`. No `kubectl` yet — that's deliberate, and it's Part 3. First you
meet the API as what it actually is: an HTTP server.

> **Forward-reference flags.** From here on, when I use a term you haven't formally met yet, I mark
> it **[fwd]** — it means "placeholder, you'll build a real understanding of this in a later
> chapter; take the one-line gloss for now and keep moving." You flagged in Part 1 that undefined
> terms threw you; this is the fix.

---

## Priming questions

Read them, guess, don't look anything up.

1. You already have etcd. Why put *another* program in front of it? List everything etcd
   **cannot** do that a cluster's front door clearly needs.
2. The thing is called an API **server**, and you'll talk to it with `curl`. So what *is* the
   Kubernetes API, physically? If it's "just" an HTTP server, what are the URLs, and what do GET
   and POST to those URLs *do*?
3. In Part 1, `watch` pushed changes to you live. If the API server is the only thing allowed to
   touch etcd, but controllers **[fwd: the loops from Part 0's model]** still need to react to
   changes in real-time — how might the API server give them that, without letting them near etcd?
4. When you POST a new object to the API server and it succeeds, where does that object physically
   end up? (You already know this store intimately. You'll go *look*.)
5. A cluster's front door sees every create, update, and delete anyone ever attempts. If you
   wanted a tamper-evident record of "who tried to do what, when" — where is the one obvious place
   in the whole architecture to capture it?

---

## Assumed state

Resuming from a passed **Part 1** gate.

- You're on **`cp`**.
- `etcd` and `etcdctl` are installed (`/usr/local/bin`), and you can start etcd and see it healthy.
- You have a `part-01-etcd` snapshot (or at least `clean-base`) to fall back to.

Check on `cp`:

```bash
hostname                      # -> cp
etcd --version                # etcd present
# start etcd if it isn't running (foreground, its own pane), then:
etcdctl endpoint health       # -> healthy
```

Leave **etcd running** in its own pane for this whole chapter — the API server needs it.

---

## The mental model

Picture three layers, bottom to top:

```
        you / curl / kubectl / controllers / workers          <- clients
                          |
                     kube-apiserver          <- the ONLY thing that talks to etcd
                          |
                        etcd                  <- the raw key-value store from Part 1
```

The API server's job, in one breath: accept HTTP requests for typed objects, **authenticate**
[fwd: Part 3] who's asking, **authorize** [fwd: Part 3] whether they may, **validate** the object
is well-formed, **persist** it into etcd (as a key under `/registry/...`), and **serve watches** so
clients learn about changes without polling and without ever touching etcd.

Two properties to hold onto, because they explain almost everything later:

- **The API is RESTful and resource-oriented.** Every kind of thing (a Pod, a Namespace **[fwd]**,
  a ServiceAccount **[fwd]**) is a *resource type* with a URL. You GET a URL to read, POST to
  create, DELETE to remove. If you know HTTP, you already half-know the Kubernetes API.
- **The API server is mostly stateless itself.** It holds no cluster state in its own memory of
  note — the state lives in etcd. The API server is a *translator and gatekeeper* between HTTP and
  etcd. Kill it and restart it and nothing is lost, because it never owned the data. (Contrast with
  etcd, which *is* the data.)

For this chapter we run the API server in a deliberately stripped-down, **insecure-for-learning**
mode so you can `curl` it without wrestling certificates first. That's backwards from real life on
purpose: you'll meet the naked HTTP API now, then Part 3 bolts on the identity and permissions
that a real API server never runs without. Learn the shape before the locks.

---

## The build

### Step 1 — Get the API server binary

The API server ships as a single binary, `kube-apiserver`, from the official Kubernetes release
artifacts. On `cp`:

```bash
# Pick a recent stable Kubernetes version. This manual uses v1.31.0 as a concrete example;
# any recent v1.3x.y works. Keep this SAME version for every K8s binary in later chapters.
KVER=v1.31.0

cd /tmp
curl -LO "https://dl.k8s.io/${KVER}/bin/linux/amd64/kube-apiserver"
chmod +x kube-apiserver
sudo mv kube-apiserver /usr/local/bin/

kube-apiserver --version      # -> Kubernetes v1.31.0 (or your chosen version)
```

> **Pin the version.** Write `KVER` in your lab log. Every Kubernetes component you download later
> (scheduler, controller-manager, kubelet…) must be the *same* version to avoid subtle skew. This
> is the one place in the course where "just grab latest each time" will bite you.

### Step 2 — Start the API server, pointed at etcd, in insecure learning mode

We need one decision up front: the API server serves HTTPS on port **6443** in real life, which
means certificates. To meet the *API* before the *PKI*, we start it with an unauthenticated local
port so plain `curl` works. Everything here is loopback-only and thrown away in Part 3.

With etcd healthy in its own pane, open a **new pane** and run:

```bash
sudo kube-apiserver \
  --etcd-servers=http://127.0.0.1:2379 \
  --service-cluster-ip-range=10.96.0.0/16 \
  --authorization-mode=AlwaysAllow \
  --insecure-port=0 \
  --bind-address=127.0.0.1 \
  --secure-port=6443 \
  --client-ca-file=/dev/null \
  --service-account-key-file=/dev/null \
  --service-account-signing-key-file=/dev/null \
  --service-account-issuer=https://kubernetes.default.svc \
  --tls-cert-file="" --tls-private-key-file="" \
  2>&1 | tee /tmp/apiserver.log
```

> **Reality note — this will fight you a little, and that's fine.** Modern `kube-apiserver`
> *really* wants TLS and service-account keys; a truly "insecure HTTP" mode was removed years ago.
> Rather than paper over that, we lean into it: in **Step 2b** we generate the *minimum* self-signed
> cert so the server starts, and `curl` with `-k`. This is closer to the truth (the API server
> speaks HTTPS) while still letting you skip real auth until Part 3. If the command above refuses to
> start (it may, depending on the exact patch version), go straight to Step 2b — that's the
> supported path.

### Step 2b — Minimal TLS so the server starts, `curl -k` so you can talk to it

Generate a throwaway self-signed cert for the API server, and a service-account keypair it insists
on:

```bash
cd ~
mkdir -p pki && cd pki

# API server serving cert (self-signed, throwaway) with the right SANs:
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout apiserver.key -out apiserver.crt -days 365 \
  -subj "/CN=kube-apiserver" \
  -addext "subjectAltName=IP:127.0.0.1,DNS:localhost,DNS:kubernetes.default.svc"

# Service-account signing keypair (the API server requires one, even in our stub):
openssl genrsa -out sa.key 2048
openssl rsa -in sa.key -pubout -out sa.pub
```

Now start the API server for real (in its own pane, etcd still running):

```bash
cd ~
sudo kube-apiserver \
  --etcd-servers=http://127.0.0.1:2379 \
  --service-cluster-ip-range=10.96.0.0/16 \
  --authorization-mode=AlwaysAllow \
  --bind-address=127.0.0.1 \
  --secure-port=6443 \
  --tls-cert-file=$HOME/pki/apiserver.crt \
  --tls-private-key-file=$HOME/pki/apiserver.key \
  --service-account-key-file=$HOME/pki/sa.pub \
  --service-account-signing-key-file=$HOME/pki/sa.key \
  --service-account-issuer=https://kubernetes.default.svc \
  2>&1 | tee /tmp/apiserver.log
```

What each flag is doing, plainly:

- `--etcd-servers` — *where the brain is.* This is the wire between Part 1 and Part 2. The API
  server will store everything through this connection.
- `--service-cluster-ip-range` — a pool of virtual IPs for Services **[fwd: Part 7]**. Required to
  boot; ignore its meaning for now.
- `--authorization-mode=AlwaysAllow` — **the big learning shortcut.** "Anyone who reaches me may do
  anything." This is why raw `curl` works with no credentials. Part 3 replaces this with real
  authorization (RBAC) and you'll watch requests start getting *denied*.
- `--tls-*` — the throwaway serving cert. Real clusters use a proper CA; you'll build one in Part 3.
- `--service-account-*` — keys the server refuses to start without. Stubbed for now.

Leave it running. Read the log it prints — it will mention initializing, connecting to etcd, and
serving on `6443`.

### Step 3 — Talk to the API with curl: the discovery endpoints

New pane. The API server is HTTPS with a self-signed cert, so `curl -k` (skip cert check — fine for
this throwaway). Start at the root:

```bash
curl -k https://127.0.0.1:6443/
```

You get a big JSON list of **paths**. This is the API telling you what it offers. Two entry points
matter:

```bash
# The "legacy" core group (Pods, Namespaces, Services, ...) lives under /api/v1:
curl -k https://127.0.0.1:6443/api/v1 | jq '.resources[].name' | sort | head -30

# Everything else is grouped under /apis/<group>/<version>:
curl -k https://127.0.0.1:6443/apis | jq '.groups[].name'
```

Read that first list. Those are the **resource types** the API server knows — `pods`,
`namespaces`, `services`, `configmaps`, `serviceaccounts`, and more. Each becomes a URL. This
"ask the server what it supports" mechanism is called **discovery**, and it's how `kubectl` later
knows what you can do without hardcoding anything.

### Step 4 — Create an object over HTTP, then find it in etcd with your own eyes

This is the payoff of the whole chapter. You'll POST a Namespace **[fwd: think "a folder that
groups objects"; full treatment later]** to the API, then go look at etcd and watch it appear as a
key — proving the three-layer picture is literally true.

First, list namespaces (there are a couple of built-in ones):

```bash
curl -k https://127.0.0.1:6443/api/v1/namespaces | jq '.items[].metadata.name'
```

Now create one. The API takes JSON describing the object:

```bash
curl -k -X POST https://127.0.0.1:6443/api/v1/namespaces \
  -H 'Content-Type: application/json' \
  -d '{
        "apiVersion": "v1",
        "kind": "Namespace",
        "metadata": { "name": "byhand" }
      }' | jq '{name: .metadata.name, uid: .metadata.uid, rv: .metadata.resourceVersion}'
```

Look at what came back: the server *added* fields you didn't send — a **`uid`**, a
**`resourceVersion`** (remember Part 1's revision? this is its descendant), a `creationTimestamp`.
The API server enriched and validated your object, then stored it. Confirm over HTTP:

```bash
curl -k https://127.0.0.1:6443/api/v1/namespaces/byhand | jq '.metadata.name'
```

Now the moment of truth. Go to your **etcd** pane's client and look directly in the store:

```bash
# List everything the API server has written. Kubernetes stores objects under /registry:
etcdctl get /registry/ --prefix --keys-only | head -40

# Find your namespace specifically:
etcdctl get /registry/namespaces/byhand
```

There it is — your object, sitting in etcd as a key under `/registry/namespaces/byhand`, exactly
where Part 1 said cluster state lives. (The value looks like binary/protobuf, not the JSON you
sent — the API server stores objects in an efficient encoding. Same object, different wire format.)

Sit with what you just proved: **you spoke HTTP to the API server, and it turned that into a write
in the key-value store you learned last chapter.** The three-layer model isn't a metaphor. You can
see both ends.

### Step 5 — The watch endpoint (Part 1's superpower, now over HTTP)

Priming Q3. Controllers **[fwd]** need to react to changes in real time but must not touch etcd. The
API server solves this by exposing etcd's watch *through HTTP*. In one pane, open a watch on
namespaces (note `?watch=true`) — it will hang, streaming:

```bash
curl -k -N "https://127.0.0.1:6443/api/v1/namespaces?watch=true"
```

In another pane, cause changes:

```bash
curl -k -X POST https://127.0.0.1:6443/api/v1/namespaces \
  -H 'Content-Type: application/json' \
  -d '{"apiVersion":"v1","kind":"Namespace","metadata":{"name":"watch-demo"}}' >/dev/null

curl -k -X DELETE https://127.0.0.1:6443/api/v1/namespaces/watch-demo >/dev/null
```

The first pane streams JSON events — `ADDED`, then `MODIFIED`/`DELETED` — one per line, live. This
is *exactly* the etcd watch you ran in Part 1, but now: typed, filtered to one resource kind,
guarded by the API server, and available to any HTTP client. **This single endpoint is the
engine of the whole control plane.** Every controller, the scheduler, every worker's agent — they
all sit on a watch like this and react to what streams out. When you hear "informer," "controller,"
"reconcile loop" in later chapters, picture this stream. `Ctrl-C` to stop.

### Step 6 — Audit logging: the front door writes down who did what

Priming Q5. Every attempted create/update/delete passes through this one process — so it's the
natural, and only sensible, place to record them. This isn't a "security chapter"; it's just an
obvious property of being the single front door. Turn it on to see it.

Stop the API server (`Ctrl-C` in its pane). Create a tiny audit policy, then restart with two extra
flags:

```bash
cat > ~/audit-policy.yaml <<'EOF'
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata      # record WHO/WHAT/WHEN for every request, without request bodies
EOF
```

Restart the API server with the same command as Step 2b, adding:

```bash
  --audit-policy-file=$HOME/audit-policy.yaml \
  --audit-log-path=/tmp/audit.log \
```

Now make a request and watch it get recorded:

```bash
curl -k -X POST https://127.0.0.1:6443/api/v1/namespaces \
  -H 'Content-Type: application/json' \
  -d '{"apiVersion":"v1","kind":"Namespace","metadata":{"name":"audited"}}' >/dev/null

tail -n 2 /tmp/audit.log | jq '{verb, resource: .objectRef.resource, name: .objectRef.name, user: .user.username, when: .requestReceivedTimestamp}'
```

You'll see a structured record: the **verb** (`create`), the **resource** (`namespaces`), the
**name** (`audited`), and — note this — a **user** field. Right now it says something anonymous,
because we have no authentication yet. Hold that thought: the instant Part 3 gives requests real
identities, this same audit stream starts naming *who* did each thing. That's the entire idea
behind the cluster audit log, and later behind tools that watch it — but it's just the front door
keeping a ledger. Nothing mysterious once you see where it lives.

---

## Verification gate

You pass Part 2 when you can, without copying from above:

1. Start etcd, then start `kube-apiserver` pointed at it, and get a healthy response from
   `curl -k https://127.0.0.1:6443/api/v1`.
2. **Create** a Namespace by POSTing JSON to the API with `curl` (any name), and GET it back.
3. **Find that same object as a key in etcd** under `/registry/namespaces/...` — and explain, in
   one sentence, why it's there and how it got there.
4. Open a `?watch=true` stream and make an `ADDED` and a `DELETED` event appear in it live by
   POSTing/DELETEing from another pane.
5. State the one-sentence rule that is the spine of this whole course: **which single component
   talks to etcd, and what does everyone else talk to?**

If #3 and #5 are fluent, you own the center of Kubernetes. Everything after this is "another client
of the thing you just built."

---

## Troubleshooting

- **`kube-apiserver` exits immediately complaining about certs or service-account keys.** Modern
  versions refuse to start without TLS and SA keys. Use **Step 2b** (generate the throwaway cert
  and SA keypair). The bare-HTTP path in Step 2 is shown to explain *why* 2b exists, not as the
  supported route on current versions.
- **`curl` fails with an SSL certificate error.** You forgot `-k`. The server uses a self-signed
  cert; `-k` tells curl to skip verification. (In Part 3 you'll stop needing `-k` because you'll
  trust a real CA.)
- **`curl` connects but everything returns `401 Unauthorized` / `403 Forbidden`.** You're not in
  `AlwaysAllow` mode, or a stray auth flag slipped in. For this chapter, `--authorization-mode`
  must be `AlwaysAllow` and there should be no `--client-ca-file` pointing at a real CA. Re-check
  the Step 2b command exactly.
- **`connection refused` on 6443.** The API server isn't running (check its pane — did it crash?
  read `/tmp/apiserver.log`) or it bound a different address. Confirm with `ss -ltnp | grep 6443`.
- **API server logs `dial tcp 127.0.0.1:2379: connect: connection refused`.** etcd isn't running.
  This is the dependency made visible — no brain, no front door. Start etcd first, always.
- **`/registry` prefix in etcd is empty even though curl created objects.** Your API server is
  pointed at a *different* etcd (or a different data dir) than the one you're inspecting. Confirm
  `--etcd-servers` matches the etcd you're querying, and that you didn't start a second etcd from a
  different directory (different `default.etcd/`).
- **Made a mess?** Restore `part-01-etcd` (clean etcd) or `clean-base` and re-run this chapter. The
  API server owns no state, so restoring etcd effectively resets the cluster.

---

## What just happened (close the loop)

- **Q1 — why another program in front of etcd:** because etcd offers no types, no validation, no
  identity, no permissions, and no safe multi-client watch. The API server adds every one of those
  and becomes the single guarded doorway to the store.
- **Q2 — what the API physically is:** an HTTPS server whose URLs are resources. GET reads, POST
  creates, DELETE removes, `?watch=true` streams changes. If you know HTTP, you know its shape.
- **Q3 — real-time reactions without touching etcd:** the API server re-exposes etcd's watch as an
  HTTP stream. Every controller lives on that stream. You saw the raw version of the engine that
  drives the whole control plane.
- **Q4 — where a created object goes:** into etcd, as a key under `/registry/...`. You POSTed over
  HTTP and then *found the exact key* — both ends of the three-layer model, visible at once.
- **Q5 — where to record who-did-what:** at the one front door everything passes through. Audit
  logging is just the API server keeping a ledger; it will name real users the moment Part 3 gives
  requests identities.

The glaring hole you should feel: right now **anyone** who reaches the API server can do
**anything** (`AlwaysAllow`), and requests have no real identity (the audit log couldn't say *who*).
That's not a cluster you'd ever run. In **Part 3** we build a real certificate authority, give
ourselves a genuine identity, turn on **RBAC** so the API server actually decides *"can this caller
do this?"*, and — finally — retire raw `curl` for `kubectl`, which is nothing more than a
well-dressed client of the exact HTTP API you just used by hand.

---

## Lab log

Snapshot `cp` as `part-02-apiserver` and add a row to `LAB_LOG.md`. **Record your `KVER`** — you
need the same version for every future component.

```
| 2025-XX-XX | 02 | Y | part-02-apiserver | apiserver in front of etcd; created ns via curl, found it in /registry; watched the stream; audit on. KVER=v1.31.0 |
```

---

**Next → [Part 3 — kubectl & auth](03-kubectl-auth.md)**

**[← Part 1](01-etcd.md)** · **[Index](README.md)**
