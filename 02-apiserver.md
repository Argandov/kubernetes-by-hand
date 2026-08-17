# Part 2 — kube-apiserver

*Kubernetes By Hand*

> ⏱️ **Estimated time:** 75–120 min · **Difficulty:** first finicky chapter
> Getting the server to *boot* is the fiddly part (certs + token file). Once it's up, the rest is
> plain `curl`. Times assume "understand and move," not deep rabbit-holing.

In Part 1 you spoke to etcd in its raw dialect: bytes at keys, no structure, no rules, no
identity. etcd will happily store `"potato"` at `/registry/pods/whatever` and never object,
because it has no idea what a Pod is. Something has to sit *in front of* that store and impose
everything etcd lacks: a real API with typed objects, validation, and — the feature you'll care
about most — a **watch endpoint** that hands clients the same push-on-change power you felt in
Part 1, but safely.

That something is the **kube-apiserver**, and it is the center of the entire system. Say it once
and let it sink in: **the API server is the only component that ever talks to etcd directly.**
Everything else — every controller, every worker, every `kubectl` — talks to the *API server*.
Learn this one process well and the rest of Kubernetes is variations on "another client of the API
server."

In this chapter you run the API server on `cp`, pointed at the etcd you already know, and talk to
it with nothing but `curl`. No `kubectl` yet — that's deliberate, and it's Part 3. First you meet
the API as what it actually is: an HTTP service.

> **A note on what we are and aren't doing with security here.** Modern `kube-apiserver` serves
> **HTTPS only** — there is no plain-HTTP mode anymore (it was removed years ago). It also refuses
> to answer fully anonymous requests in the configuration we want. So two small, *deliberately
> opaque* credentials are unavoidable just to get the server to boot and answer you:
>
> 1. a **serving certificate** — the server proving its own identity to you. You will generate one
>    throwaway file and then **ignore it entirely** with `curl -k` (which means "don't verify the
>    server's cert"). You never think about it again.
> 2. a **bearer token** — a one-line throwaway password that says "the caller is `me`." You put it
>    in a file and send it as an HTTP header.
>
> **This is not PKI, and it is not the lesson.** A real certificate authority, client-certificate
> *identity*, trust chains, and **RBAC** deciding who-may-do-what are the entire content of **Part
> 3**, and none of it happens here. Think of the token as a turnstile you badge through to reach the
> thing you're actually here to study: the API. Badge through, don't overthink it.

> **Forward-reference flags.** From here on, when I use a term you haven't formally met yet, I mark
> it **[fwd]** — "placeholder, you'll build a real understanding of this later; take the one-line
> gloss for now and keep moving." You flagged in Part 1 that undefined terms threw you; this is the
> fix.

---

## Priming questions

Read them, guess, don't look anything up.

1. You already have etcd. Why put *another* program in front of it? List everything etcd
   **cannot** do that a cluster's front door clearly needs.
2. The thing is called an API **server**, and you'll talk to it with `curl`. So what *is* the
   Kubernetes API, physically? If it's "just" an HTTP service, what are the URLs, and what do GET
   and POST to those URLs *do*?
3. In Part 1, `watch` pushed changes to you live. If the API server is the only thing allowed to
   touch etcd, but controllers **[fwd: the loops from Part 0's model]** still need to react to
   changes in real-time — how might the API server give them that, without letting them near etcd?
4. When you POST a new object to the API server and it succeeds, where does that object physically
   end up? (You already know this store intimately. You'll go *look*.)
5. A cluster's front door sees every create, update, and delete anyone ever attempts. If you
   wanted a record of "who tried to do what, when" — where is the one obvious place in the whole
   architecture to capture it?

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

The API server's job, in one breath: accept HTTPS requests for typed objects, figure out **who is
asking** (authentication — for us, the token), **validate** the object is well-formed, **persist**
it into etcd (as a key under `/registry/...`), and **serve watches** so clients learn about changes
without polling and without ever touching etcd.

Two properties to hold onto, because they explain almost everything later:

- **The API is RESTful and resource-oriented.** Every kind of thing (a Pod, a Namespace **[fwd]**,
  a ServiceAccount **[fwd]**) is a *resource type* with a URL. You GET a URL to read, POST to
  create, DELETE to remove. If you know HTTP, you already half-know the Kubernetes API.
- **The API server is mostly stateless itself.** It holds no cluster state in its own memory of
  note — the state lives in etcd. The API server is a *translator and gatekeeper* between HTTP and
  etcd. Kill it and restart it and nothing is lost, because it never owned the data. (Contrast with
  etcd, which *is* the data.)

For this chapter we run with authorization set to `AlwaysAllow` — once the token tells the server
*who* you are, it lets you do *anything*. That "anyone-authenticated-can-do-everything" stance is
obviously not how you'd run a real cluster; replacing it with real permissions (**RBAC**) is Part
3. Learn the shape of the API before the locks.

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

### Step 2 — Generate the three things the server needs to boot

The server refuses to start without a **serving cert** and a **service-account keypair**, and it
won't know who you are without a **token file**. Make all three now. Remember: these are opaque
throwaways, not a lesson — Part 3 is where certificates become meaningful.

```bash
cd ~
mkdir -p pki && cd pki

# (a) Serving certificate — the server proving itself to clients. Self-signed, throwaway.
#     SANs include 127.0.0.1 so tools that DO check the cert are happy; we mostly use -k anyway.
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout apiserver.key -out apiserver.crt -days 365 \
  -subj "/CN=kube-apiserver" \
  -addext "subjectAltName=IP:127.0.0.1,DNS:localhost,DNS:kubernetes.default.svc"

# (b) Service-account signing keypair — the server requires one even in our minimal setup.
openssl genrsa -out sa.key 2048
openssl rsa -in sa.key -pubout -out sa.pub

# (c) Token file — one line: token,username,uid
#     This is your throwaway "password". The caller presenting it becomes user "me".
echo 'letmein-byhand,me,1' > tokens.csv

ls -1 apiserver.crt apiserver.key sa.key sa.pub tokens.csv
```

That's the whole "security" setup for this chapter: one serving cert you'll ignore, one keypair the
server insists on, one token that names you.

### Step 3 — Start the API server, pointed at etcd

With etcd healthy in its own pane, open a **new pane** and start the server. If you're restarting
after a previous attempt, kill any old instance first (this is a real gotcha — a stale server keeps
port 6443 and your "new" one silently fails to bind):

```bash
sudo pkill -f kube-apiserver 2>/dev/null ; sleep 2
ss -ltnp | grep 6443        # should print NOTHING before you start. If it does, pkill again.
```

Now start it:

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
  --token-auth-file=$HOME/pki/tokens.csv \
  --service-account-key-file=$HOME/pki/sa.pub \
  --service-account-signing-key-file=$HOME/pki/sa.key \
  --service-account-issuer=https://kubernetes.default.svc \
  2>&1 | tee /tmp/apiserver.log
```

What each flag is doing, plainly:

- `--etcd-servers` — *where the brain is.* This is the wire between Part 1 and Part 2. The API
  server stores everything through this connection.
- `--service-cluster-ip-range` — a pool of virtual IPs for Services **[fwd: Part 7]**. Required to
  boot; ignore its meaning for now.
- `--authorization-mode=AlwaysAllow` — once you're authenticated, you may do anything. Part 3
  replaces this with **RBAC** and you'll watch requests start getting *denied*.
- `--tls-cert-file` / `--tls-private-key-file` — the throwaway **serving** cert from Step 2. This
  is why the server is HTTPS and why clients use `-k`.
- `--token-auth-file` — the token file from Step 2. This is how the server learns you're `me`.
- `--service-account-*` — keys the server refuses to start without. Opaque for now.

Leave it running. Skim the log — it will mention connecting to etcd, loading admission controllers,
and finally `Serving securely on 127.0.0.1:6443`. That last line is your green light.

> **Reading the log is a skill this course is training.** If the server *exits*, the reason is
> almost always one clear line near the bottom of `/tmp/apiserver.log`. Two you might see:
> a warning that a flag combination was rejected, or a `connect: connection refused` for etcd
> (meaning etcd isn't running). Look there first, always.

### Step 4 — Talk to the API: set your token, hit the discovery endpoints

Open a **new pane**. Every request from here on carries your token in an `Authorization` header, so
set it once as a shell variable to keep commands clean:

```bash
export TOKEN=letmein-byhand
```

The API server is HTTPS with a self-signed cert, so `curl -k` (skip cert check — fine for this
throwaway). Ask the server what it offers:

```bash
# The "legacy" core group (Pods, Namespaces, Services, ...) lives under /api/v1:
curl -k -H "Authorization: Bearer $TOKEN" \
  https://127.0.0.1:6443/api/v1 | jq '.resources[].name' | sort | head -30

# Everything else is grouped under /apis/<group>/<version>:
curl -k -H "Authorization: Bearer $TOKEN" \
  https://127.0.0.1:6443/apis | jq '.groups[].name'
```

Read that first list. Those are the **resource types** the API server knows — `pods`,
`namespaces`, `services`, `configmaps`, `serviceaccounts`, and more. Each becomes a URL. This
"ask the server what it supports" mechanism is called **discovery**, and it's how `kubectl` later
knows what you can do without hardcoding anything.

> **If you drop the header**, e.g. `curl -k https://127.0.0.1:6443/api/v1`, you'll get a `401
> Unauthorized`. That's correct: no token means no identity, and the server won't answer. The `401`
> is the server saying "I don't know who you are" — remember that; it's diagnostic.

### Step 5 — Create an object over HTTP, then find it in etcd with your own eyes

This is the payoff of the whole chapter. You'll POST a Namespace **[fwd: think "a folder that
groups objects"; full treatment later]** to the API, then go look at etcd and watch it appear as a
key — proving the three-layer picture is literally true.

First, list namespaces (there are a couple of built-in ones):

```bash
curl -k -H "Authorization: Bearer $TOKEN" \
  https://127.0.0.1:6443/api/v1/namespaces | jq '.items[].metadata.name'
```

Now create one. The API takes JSON describing the object:

```bash
curl -k -H "Authorization: Bearer $TOKEN" \
  -X POST https://127.0.0.1:6443/api/v1/namespaces \
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
curl -k -H "Authorization: Bearer $TOKEN" \
  https://127.0.0.1:6443/api/v1/namespaces/byhand | jq '.metadata.name'
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

### Step 6 — The watch endpoint (Part 1's superpower, now over HTTP)

Priming Q3. Controllers **[fwd]** need to react to changes in real time but must not touch etcd. The
API server solves this by exposing etcd's watch *through HTTP*. In one pane, open a watch on
namespaces (note `?watch=true` and `-N` to keep the stream open) — it will hang, streaming:

```bash
curl -k -N -H "Authorization: Bearer $TOKEN" \
  "https://127.0.0.1:6443/api/v1/namespaces?watch=true"
```

In another pane (set `export TOKEN=letmein-byhand` there too), cause changes:

```bash
curl -k -H "Authorization: Bearer $TOKEN" \
  -X POST https://127.0.0.1:6443/api/v1/namespaces \
  -H 'Content-Type: application/json' \
  -d '{"apiVersion":"v1","kind":"Namespace","metadata":{"name":"watch-demo"}}' >/dev/null

curl -k -H "Authorization: Bearer $TOKEN" \
  -X DELETE https://127.0.0.1:6443/api/v1/namespaces/watch-demo >/dev/null
```

The first pane streams JSON events — `ADDED`, then `MODIFIED`/`DELETED` — one per line, live. This
is *exactly* the etcd watch you ran in Part 1, but now: typed, filtered to one resource kind,
guarded by the API server, and available to any HTTP client. **This single endpoint is the engine
of the whole control plane.** Every controller, the scheduler, every worker's agent — they all sit
on a watch like this and react to what streams out. When you hear "informer," "controller,"
"reconcile loop" in later chapters, picture this stream. `Ctrl-C` to stop.

### Step 7 — Audit logging: the front door writes down who did what

Priming Q5. Every attempted create/update/delete passes through this one process — so it's the
natural, and only sensible, place to record them. This isn't a "security chapter"; it's just an
obvious property of being the single front door. Turn it on to see it.

Stop the API server (`Ctrl-C` in its pane). Create a tiny audit policy:

```bash
cat > ~/audit-policy.yaml <<'EOF'
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata      # record WHO/WHAT/WHEN for every request, without request bodies
EOF
```

Restart the API server with the **same command as Step 3**, adding these two lines:

```bash
  --audit-policy-file=$HOME/audit-policy.yaml \
  --audit-log-path=/tmp/audit.log \
```

Now make a request and watch it get recorded:

```bash
curl -k -H "Authorization: Bearer $TOKEN" \
  -X POST https://127.0.0.1:6443/api/v1/namespaces \
  -H 'Content-Type: application/json' \
  -d '{"apiVersion":"v1","kind":"Namespace","metadata":{"name":"audited"}}' >/dev/null

tail -n 5 /tmp/audit.log | jq '{verb, resource: .objectRef.resource, name: .objectRef.name, user: .user.username, when: .requestReceivedTimestamp}'
```

You'll see a structured record: the **verb** (`create`), the **resource** (`namespaces`), the
**name** (`audited`), and — note this — a **user** field that says **`me`**. That's your token
identity showing up in the ledger. The front door knows who called because the token told it, and
it wrote that down.

Hold onto this thread: right now "who" is just a throwaway token user. In **Part 3**, when you have
a real certificate identity and RBAC, this same audit stream will name your *real* user and record
which actions were *allowed vs denied*. The audit log isn't changing — the richness of "who" is.
That's the whole idea behind a cluster audit log, and later behind tools that watch it. Nothing
mysterious once you see where it lives: it's just the front door keeping a ledger.

---

## Verification gate

You pass Part 2 when you can, without copying from above:

1. Start etcd, then start `kube-apiserver` pointed at it, and get the resource list back from
   `curl -k -H "Authorization: Bearer $TOKEN" https://127.0.0.1:6443/api/v1`.
2. **Create** a Namespace by POSTing JSON to the API with `curl` (any name), and GET it back.
3. **Find that same object as a key in etcd** under `/registry/namespaces/...` — and explain, in
   one sentence, why it's there and how it got there.
4. Open a `?watch=true` stream and make an `ADDED` and a `DELETED` event appear in it live by
   POSTing/DELETEing from another pane.
5. State the one-sentence rule that is the spine of this whole course: **which single component
   talks to etcd, and what does everyone else talk to?**
6. Explain what a bare `curl -k https://127.0.0.1:6443/api/v1` (no header) returns and **why**.

If #3, #5, and #6 are fluent, you own the center of Kubernetes. Everything after this is "another
client of the thing you just built."

---

## Troubleshooting

Read this section top to bottom once — these are the exact traps this chapter produces.

- **`Error: unknown flag: --insecure-port` (or `--port`, `--insecure-bind-address`).** These flags
  were **removed** in Kubernetes v1.24. There is no plain-HTTP mode on modern versions. Use the
  HTTPS + token setup in this chapter (Steps 2–3); don't try to resurrect insecure serving.

- **`curl` returns `401 Unauthorized`.** Authentication failed — the server doesn't know who you
  are. Almost always: **you forgot the `-H "Authorization: Bearer $TOKEN"` header**, or `$TOKEN` is
  empty in this pane (run `echo $TOKEN`; re-`export TOKEN=letmein-byhand` — shell variables don't
  cross panes). Also confirm the token in your header matches the first field of `~/pki/tokens.csv`.

- **`curl` returns `403 Forbidden`.** Different failure: the server *knows* who you are but won't
  let you do it. You shouldn't see this in Part 2 (we're on `AlwaysAllow`). If you do, a stray auth
  flag slipped in or you're not actually on `AlwaysAllow`. **Learn the 401-vs-403 distinction now —
  it's diagnostic all course: 401 = "I don't know you," 403 = "I know you, but no."**

- **Log says `AnonymousAuth is not allowed with the AlwaysAllow authorizer. Resetting AnonymousAuth
  to false`.** This is why fully anonymous requests get 401 under `AlwaysAllow`, and why we use a
  token instead of anonymous access. Expected; not an error. Just send the token.

- **Server exits immediately complaining about certs or service-account keys.** You skipped part of
  Step 2. It needs `--tls-cert-file`/`--tls-private-key-file` (the serving cert) **and**
  `--service-account-key-file`/`--service-account-signing-key-file`. Regenerate all of Step 2.

- **`curl` fails with an SSL certificate error.** You forgot `-k`. The server uses a self-signed
  serving cert; `-k` tells curl to skip verifying it. (In Part 3 you'll stop needing `-k` because
  you'll trust a real CA.)

- **`connection refused` on 6443.** The API server isn't running — check its pane (did it crash?
  read `/tmp/apiserver.log`) — **or** a stale server is holding the port and your new one failed to
  bind. Always do the `pkill -f kube-apiserver ; sleep 2 ; ss -ltnp | grep 6443` dance before
  starting. `ss` should show nothing, then you start.

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
  identity, and no safe multi-client watch. The API server adds every one of those and becomes the
  single guarded doorway to the store.
- **Q2 — what the API physically is:** an HTTPS service whose URLs are resources. GET reads, POST
  creates, DELETE removes, `?watch=true` streams changes. If you know HTTP, you know its shape.
- **Q3 — real-time reactions without touching etcd:** the API server re-exposes etcd's watch as an
  HTTP stream. Every controller lives on that stream. You saw the raw version of the engine that
  drives the whole control plane.
- **Q4 — where a created object goes:** into etcd, as a key under `/registry/...`. You POSTed over
  HTTP and then *found the exact key* — both ends of the three-layer model, visible at once.
- **Q5 — where to record who-did-what:** at the one front door everything passes through. Audit
  logging is just the API server keeping a ledger; it named `me` because the token gave the request
  an identity.

The honest state of things: you have a real, if minimal, front door — it types, validates,
persists, and streams, and it knows who's calling (via a throwaway token). But two big pieces are
stubbed: identity is a *shared password*, not a real per-user credential, and authorization is
`AlwaysAllow` — anyone who badges in can do anything. That's not a cluster you'd ever run.

In **Part 3** we replace both properly: you'll build a real **certificate authority**, mint
yourself a **client certificate** that *is* your identity (no shared token), turn on **RBAC** so the
API server actually decides *"may this caller do this?"* — you'll watch a request go from `403` to
`200` because of a rule you wrote — and finally retire raw `curl` for `kubectl`, which you'll see is
nothing more than a well-dressed client of the exact HTTP API you just drove by hand.

---

## Lab log

Snapshot `cp` as `part-02-apiserver` and add a row to `LAB_LOG.md`. **Record your `KVER`** — you
need the same version for every future component.

```
| 2025-XX-XX | 02 | Y | part-02-apiserver | apiserver in front of etcd; token auth; created ns via curl, found it in /registry; watched the stream; audit named "me". KVER=v1.31.0 |
```

---

**Next → [Part 3 — kubectl & auth](03-kubectl-auth.md)**

**[← Part 1](01-etcd.md)** · **[Index](README.md)**
