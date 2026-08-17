# Part 3 — kubectl & auth

*Kubernetes By Hand*

At the end of Part 2 you had a working API server with two embarrassing holes: it trusted
**anyone** (`--authorization-mode=AlwaysAllow`), and requests carried **no real identity** (the
audit log couldn't say who did anything). In this chapter we close both, and in doing so you'll
learn the two questions the API server asks about *every single request*:

1. **Authentication** — *"Who are you?"* (prove your identity)
2. **Authorization** — *"Are you allowed to do this?"* (RBAC decides)

You'll build a tiny certificate authority, mint yourself a client certificate that *is* your
identity, switch the API server from "allow everyone" to "check RBAC," and watch your own requests
get **denied** — then write the rule that allows them and watch them succeed. Feeling a request go
from `403 Forbidden` to `200 OK` because of a rule *you* wrote is how RBAC stops being abstract.

Only at the very end do we introduce **`kubectl`** — and by then you'll see it clearly for what it
is: a comfortable client wrapped around the exact HTTP API you've been driving by hand with `curl`.
No magic. Just a nicer `curl`.

> ⏱️ **Estimated time:** 90–150 min · **Difficulty:** most moving parts so far
> Expect a potential 20-30 min rabbit hole on Certs

> **[fwd]** flags continue: a term marked **[fwd]** is a placeholder you'll fully meet later.

---

## Priming questions

Guess first. Don't look anything up.

1. The API server needs to know *who* is making a request. It has no user database, no passwords
   file you've set up. So how can an HTTP client **prove who it is** using only the TLS connection
   itself? (You made a certificate in Part 2. What if a certificate could carry a *name*?)
2. "Authentication" and "authorization" sound like synonyms in English but are two distinct steps
   here. What's the difference, and why must they be separate? What breaks if you conflate them?
3. RBAC stands for Role-Based Access Control. Before reading its details: if you had to design
   "who can do what" for a system with thousands of object types, what are the *minimum* pieces you'd
   need? (Hint: a *verb*, a *thing*, and a *someone* — how do you connect them?)
4. `kubectl get pods` and `curl -k https://.../api/v1/.../pods` return the same data. So what,
   precisely, is `kubectl` *doing* that `curl` isn't — and what must it read to know where your API
   server is and who you are?
5. In Part 2 the audit log's `user` field was anonymous. After this chapter, what will it say — and
   *why* will it suddenly know?

---

## Assumed state

Resuming from a passed **Part 2** gate.

- You're on **`cp`**.
- etcd runs and is healthy; `kube-apiserver` (v`KVER`) starts and serves on `6443`.
- You can create a Namespace via `curl` and find it in etcd under `/registry`.
- You have the throwaway `~/pki/apiserver.crt|key` and `~/pki/sa.*` from Part 2, and a
  `part-02-apiserver` snapshot.

Check on `cp`:

```bash
hostname                                  # -> cp
etcdctl endpoint health                   # healthy (start etcd if needed)
# start the apiserver from Part 2 if needed, then:
curl -k https://127.0.0.1:6443/api/v1 >/dev/null && echo "apiserver OK"
```

---

## The mental model

Every request to the API server runs a gauntlet, in this fixed order:

```
request --> [ Authentication ] --> [ Authorization ] --> [ Admission (fwd) ] --> etcd
             "who are you?"         "may you do it?"       "any last edits?"
```

- **Authentication** turns an anonymous connection into a *named* one. It does **not** decide
  permissions — only identity. Its entire output is "you are `argv`, member of groups `[x,y]`" (or
  "you are `system:anonymous`"). We'll authenticate with a **client certificate**: the API server
  trusts a CA, you present a cert *signed by that CA*, and the **Common Name (CN)** in your cert
  becomes your username, the **Organization (O)** fields become your groups. Identity carried by
  cryptography — no password file needed. (That answers priming Q1.)
- **Authorization** takes that identity and asks the configured authorizer, *"may this user perform
  this verb on this resource?"* We'll use **RBAC**. By default RBAC denies everything — you are
  allowed only what a rule explicitly grants. (Q2: separating the steps means you can change *who
  you are* and *what you may do* independently, and reason about each alone.)
- **Admission [fwd: Part 10]** can mutate or reject valid, authorized requests via plugins. Ignore
  it this chapter; just know it's the next stage.

RBAC itself is four object types, and they're simpler than they sound (Q3):

- **Role** / **ClusterRole** — a *list of permissions*: "these verbs, on these resources." A Role
  is scoped to one Namespace; a ClusterRole is cluster-wide. A permission is literally
  `{verbs, resources, apiGroups}` — e.g. "get,list,watch on pods."
- **RoleBinding** / **ClusterRoleBinding** — *glue*: "this **subject** (a user/group/service
  account) has this Role." A permission list connected to a someone. That's it: a *verb+thing* list
  (Role) bound to a *someone* (Binding).

Hold the shape: **Role = what may be done; Binding = who may do the Role.** Every RBAC setup you'll
ever see is combinations of those two ideas.

---

## The build

### Step 1 — Become a certificate authority

A CA is just a keypair you decide to trust as the root of identity. The API server will be told
"trust certs signed by this CA," and anyone holding a CA-signed cert gets the identity written
inside it. On `cp`:

```bash
cd ~/pki

# The CA: a self-signed root you control.
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -days 3650 \
  -subj "/CN=kubernetes-by-hand-ca" \
  -out ca.crt

ls -1 ca.crt ca.key      # your certificate authority
```

`ca.crt` is public (the API server needs it to verify signatures). `ca.key` is the crown jewels —
whoever holds it can mint *any* identity. In a real cluster this key is guarded fiercely; here it
lives next to everything else because the whole lab is disposable.

### Step 2 — Mint YOUR identity as a client certificate

Now issue yourself a client cert whose **CN is your username** and whose **O is your group**. This
cert literally *is* your identity to the cluster.

```bash
cd ~/pki

# 1) your private key
openssl genrsa -out argv.key 2048

# 2) a signing request: CN=argv (username), O=byhand-admins (group)
openssl req -new -key argv.key \
  -subj "/CN=argv/O=byhand-admins" \
  -out argv.csr

# 3) the CA signs it -> a cert the API server will trust and read your name from
openssl x509 -req -in argv.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -days 365 -out argv.crt

# inspect what identity it encodes:
openssl x509 -in argv.crt -noout -subject
# -> subject=CN=argv, O=byhand-admins
```

You now hold three files that matter: `argv.crt` + `argv.key` (your identity) and `ca.crt` (to
verify the *server*). Keep the names — `CN=argv`, `O=byhand-admins` — in mind; they'll appear as
your username and group everywhere from here on.

### Step 3 — Restart the API server to trust your CA and enforce RBAC

Two changes from Part 2's command: point the API server at your CA for client authentication, and
flip authorization from `AlwaysAllow` to `RBAC`. Stop the API server (`Ctrl-C`) and restart with:

```bash
cd ~
sudo kube-apiserver \
  --etcd-servers=http://127.0.0.1:2379 \
  --service-cluster-ip-range=10.96.0.0/16 \
  --authorization-mode=Node,RBAC \
  --bind-address=127.0.0.1 \
  --secure-port=6443 \
  --tls-cert-file=$HOME/pki/apiserver.crt \
  --tls-private-key-file=$HOME/pki/apiserver.key \
  --client-ca-file=$HOME/pki/ca.crt \
  --service-account-key-file=$HOME/pki/sa.pub \
  --service-account-signing-key-file=$HOME/pki/sa.key \
  --service-account-issuer=https://kubernetes.default.svc \
  --audit-policy-file=$HOME/audit-policy.yaml \
  --audit-log-path=/tmp/audit.log \
  2>&1 | tee /tmp/apiserver.log
```

The two lines that changed everything:

- `--client-ca-file=$HOME/pki/ca.crt` — **authentication**: "trust client certs signed by this CA,
  and read the caller's identity (CN/O) from them."
- `--authorization-mode=Node,RBAC` — **authorization**: "stop allowing everyone; check RBAC rules."
  (`Node` is an additional authorizer for workers **[fwd: Part 5]**; harmless now.)

Leave it running.

### Step 4 — Feel the denial (this is the lesson)

First, present your certificate identity to a normally-safe read. Note we now provide *three* PEM
files: your cert, your key, and the CA (so curl trusts the server too — no more `-k`):

```bash
cd ~/pki

curl --cert argv.crt --key argv.key --cacert ca.crt \
  https://127.0.0.1:6443/api/v1/namespaces
```

You are **authenticated** (the server knows you're `argv`) but **not authorized** — RBAC grants
nothing by default. Expect a `403 Forbidden` whose message names you:

```
... "forbidden: User \"argv\" cannot list resource \"namespaces\" ... "
```

Read that message closely — it's RBAC being explicit and honest: *this user, this verb, this
resource, denied.* You have a valid identity and zero permissions. That's the secure default: deny
unless explicitly allowed. Now go grant yourself something.

### Step 5 — Grant a permission, watch 403 become 200

We need to write RBAC objects — but *you* can't yet (you have no permissions), a classic
bootstrapping knot. We cut it the way real clusters bootstrap: use a **superuser** the API server
trusts implicitly. The simplest lab approach: temporarily talk to the API as a member of the
built-in `system:masters` group, which is hardwired to full access.

Mint a one-off admin cert in `system:masters` (this group is the break-glass identity):

```bash
cd ~/pki
openssl genrsa -out admin.key 2048
openssl req -new -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 365 -out admin.crt
```

As that admin, create a **ClusterRole** (what may be done) and a **ClusterRoleBinding** (who may do
it) that grant `argv` read access to namespaces. We POST these as JSON, same as any object:

```bash
cd ~/pki

# ClusterRole: "get,list,watch on namespaces"
curl --cert admin.crt --key admin.key --cacert ca.crt \
  -X POST https://127.0.0.1:6443/apis/rbac.authorization.k8s.io/v1/clusterroles \
  -H 'Content-Type: application/json' \
  -d '{
    "apiVersion":"rbac.authorization.k8s.io/v1",
    "kind":"ClusterRole",
    "metadata":{"name":"ns-reader"},
    "rules":[{"apiGroups":[""],"resources":["namespaces"],"verbs":["get","list","watch"]}]
  }'

# ClusterRoleBinding: bind that role to user "argv"
curl --cert admin.crt --key admin.key --cacert ca.crt \
  -X POST https://127.0.0.1:6443/apis/rbac.authorization.k8s.io/v1/clusterrolebindings \
  -H 'Content-Type: application/json' \
  -d '{
    "apiVersion":"rbac.authorization.k8s.io/v1",
    "kind":"ClusterRoleBinding",
    "metadata":{"name":"argv-ns-reader"},
    "subjects":[{"kind":"User","name":"argv","apiGroup":"rbac.authorization.k8s.io"}],
    "roleRef":{"kind":"ClusterRole","name":"ns-reader","apiGroup":"rbac.authorization.k8s.io"}
  }'
```

Now repeat the **exact** request that was denied in Step 4, as `argv`:

```bash
curl --cert argv.crt --key argv.key --cacert ca.crt \
  https://127.0.0.1:6443/api/v1/namespaces | jq '.items[].metadata.name'
```

`403` → `200`. You now see the namespace list. **Nothing about your identity changed** — the same
cert, the same user. The only difference is a rule you wrote connecting the verb `list`, the
resource `namespaces`, and the subject `argv`. That is the entire mental model of RBAC, felt
directly: *Role = permitted verbs+resources; Binding = attach them to a someone.*

Try a verb you *didn't* grant and watch the denial return — proving grants are precise:

```bash
# you were granted get/list/watch, NOT delete:
curl --cert argv.crt --key argv.key --cacert ca.crt \
  -X DELETE https://127.0.0.1:6443/api/v1/namespaces/byhand
# -> 403 Forbidden: argv cannot delete namespaces
```

### Step 6 — Confirm the audit log now names you (closing Part 2's thread)

Priming Q5. Back in Part 2 the audit `user` was anonymous. Look now:

```bash
tail -n 20 /tmp/audit.log | jq -r 'select(.objectRef.resource=="namespaces") | "\(.verb) \(.objectRef.resource) by \(.user.username) -> \(.responseStatus.code // "?")"' | tail -5
```

You'll see lines attributing actions to **`argv`** (and your denied delete showing a `403`). The
audit log didn't change — *authentication* did. The moment requests carry a real identity, the
front door's ledger records *who*. That's the whole reason audit logging lived in the API server
chapter: it was only ever waiting for identity to become meaningful.

### Step 7 — Finally, `kubectl`: a client of everything you just built

You've been the client by hand. `kubectl` automates it. It needs to know three things — *where* the
API server is, *how to trust it* (CA), and *who you are* (your cert+key) — all of which live in a
single file called a **kubeconfig**. Install `kubectl` and build one.

```bash
# same KVER as your apiserver!
KVER=v1.31.0
cd /tmp
curl -LO "https://dl.k8s.io/${KVER}/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client
```

Assemble a kubeconfig from the pieces you already have — notice every value maps to a concept you
now understand:

```bash
cd ~/pki

kubectl config set-cluster byhand \
  --server=https://127.0.0.1:6443 \
  --certificate-authority=$HOME/pki/ca.crt \
  --embed-certs=true \
  --kubeconfig=$HOME/.kube/config

kubectl config set-credentials argv \
  --client-certificate=$HOME/pki/argv.crt \
  --client-key=$HOME/pki/argv.key \
  --embed-certs=true \
  --kubeconfig=$HOME/.kube/config

kubectl config set-context byhand \
  --cluster=byhand --user=argv \
  --kubeconfig=$HOME/.kube/config

kubectl config use-context byhand --kubeconfig=$HOME/.kube/config
```

Open `~/.kube/config` and read it. There are no secrets you haven't met: a **cluster** (server URL
+ CA), a **user** (your client cert+key), and a **context** tying them together. `kubectl` is just
a program that reads this file and makes the HTTPS calls you were typing by hand.

Prove the equivalence:

```bash
kubectl get namespaces
```

Same list as your `curl` in Step 5 — because it *is* the same request. And RBAC still applies to
`kubectl` exactly as it did to `curl` (same identity, same rules):

```bash
kubectl delete namespace byhand
# -> Error from server (Forbidden): ... argv cannot delete ... namespaces
```

`kubectl` didn't grant itself anything. It's a client. The API server is still the one deciding.
That's the sentence to leave this chapter with.

Optional, to *feel* that kubectl is a thin wrapper: ask it to show the raw HTTP it makes:

```bash
kubectl get namespaces -v=8 2>&1 | grep -E 'GET https|Request Headers|Response Status' | head
```

You'll see it performing the very `GET https://127.0.0.1:6443/api/v1/namespaces` you did by hand.

---

## Verification gate

You pass Part 3 when, without copying:

1. You can explain the difference between **authentication** and **authorization** in one sentence
   each, and name what carries your identity (a client cert; CN=user, O=group).
2. You make a request as `argv` that is **denied** by RBAC (`403`), then write a Role/Binding that
   makes the *same* request **succeed** (`200`) — and can articulate that only the *rule* changed,
   not your identity.
3. You can state the RBAC shape: **Role/ClusterRole = permitted verbs on resources; Binding =
   attaches a Role to a subject.**
4. `kubectl get namespaces` works through your kubeconfig, and you can name the three things a
   kubeconfig holds (cluster/CA, user cert+key, context).
5. You can explain why `kubectl delete` is *also* denied — i.e. why `kubectl` has no more power than
   your `curl` did.

If #2 and #5 are fluent, RBAC and the client/server relationship are yours.

---

## Troubleshooting

- **`curl` gives `403` even as `admin`.** Your admin cert's Organization must be exactly
  `system:masters` (that literal string is the hardwired superuser group). Re-check
  `openssl x509 -in admin.crt -noout -subject` → must show `O=system:masters`.
- **`curl` gives an SSL error now that you dropped `-k`.** Either `--cacert ca.crt` is missing/wrong
  (curl can't verify the server) or the server cert's SANs don't include `127.0.0.1` (regenerate the
  apiserver cert per Part 2 Step 2b with the IP SAN). During learning you *can* fall back to `-k`,
  but fixing it teaches the trust chain.
- **`401 Unauthorized` (not 403).** Authentication failed — the server didn't accept your client
  cert at all. Usually `--client-ca-file` doesn't match the CA that signed `argv.crt`, or you're
  presenting the wrong `--cert/--key` pair. 401 = "I don't know who you are"; 403 = "I know you, but
  no." Learn to read which one you got.
- **RBAC change seems to have no effect.** RBAC is additive and near-instant, but confirm the
  **subject name matches exactly** — `name":"argv"` must equal the cert CN `argv`, case-sensitive.
  A binding to `Argv` or `argv@x` won't match.
- **`kubectl` can't find the cluster / `connection refused`.** Wrong `--server` in the kubeconfig,
  or the API server isn't running. `kubectl config view` to inspect; `ss -ltnp | grep 6443` to
  confirm the server.
- **Everything is denied and you locked yourself out.** Use the `admin`/`system:masters` cert with
  `curl` (or make a kubeconfig for it) to fix bindings. This is your break-glass path. Worst case,
  restore `part-02-apiserver` (which ran `AlwaysAllow`) and redo from Step 3.

---

## What just happened (close the loop)

- **Q1 — proving identity over TLS:** a client certificate *signed by a CA the server trusts*
  carries your username in its CN and groups in its O. Cryptography is the credential; no password
  store needed.
- **Q2 — auth vs. authz:** authentication establishes *who* (identity only); authorization decides
  *whether* (permissions only). Separating them lets you reason about, and change, identity and
  permission independently — you saw a fixed identity go from denied to allowed by changing only a
  rule.
- **Q3 — RBAC's minimum pieces:** a permission list (verbs × resources = Role) connected to a
  subject (Binding). You built exactly that and watched it take effect.
- **Q4 — what kubectl does that curl doesn't:** it reads a kubeconfig (server + CA + your identity)
  and makes the same HTTPS calls for you. `-v=8` showed the identical request underneath.
- **Q5 — why the audit log now names you:** because authentication finally gave requests an
  identity; the ledger was always recording, it just had no name to write until now.

Step back and see how far the picture has come. You have: a durable store (etcd), a single guarded
front door that types/validates/persists/streams and now *knows who's calling and enforces what
they may do* (the API server), and a proper client (`kubectl`) that is nothing but a well-dressed
HTTP caller. That is a real, if minimal, **control plane** — a database with a guarded API in front
of it.

But it's still just a database with an API. Nothing is *acting* on the objects yet. If you created
a Pod **[fwd]** object right now, it would land in etcd as a key and then… sit there. Nothing
schedules it; nothing runs it. The cluster can *remember* desired state but can't yet *move reality
toward it*. In **Part 4** we add the first of the **loops** from Part 0's mental model — the
**scheduler** — and you'll watch it do exactly one thing that suddenly makes the whole "reconcile
reality to desired state" idea concrete: take a Pod that's sitting unassigned and decide which node
runs it.

---

## Lab log

Snapshot `cp` as `part-03-kubectl-auth` and add a row to `LAB_LOG.md`:

```
| 2025-XX-XX | 03 | Y | part-03-kubectl-auth | built CA; argv identity via client cert; RBAC deny->allow; kubectl via kubeconfig; audit now names argv |
```

---

**Next → Part 4 — the scheduler** *(coming next release)*

**[← Part 2](02-apiserver.md)** | **[Index](README.md)** | **[→ Part 4](02-apiserver.md)**
