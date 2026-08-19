# Part 4 — the scheduler

*Kubernetes By Hand*

> ⏱️ **Estimated time:** 60–100 min · **Difficulty:** conceptually rich, mechanically light
> Much less fiddly than the cert chapter — you reuse Part 3's cert skills once, then the payoff is
> watching a single field get filled in. Times assume "understand and move," not deep rabbit-holing.

So far you've built a place to *remember* desired state (etcd) and a guarded door to *read and
write* it (the API server). But nothing in your cluster **does** anything. Objects land in etcd and
just sit there. Recall the one-sentence model from Part 0:

> Kubernetes is a database plus a set of **loops** that work to make reality match the database.

You've built the database. This chapter you meet your **first loop** — the **scheduler** — and it
does exactly one small, clarifying thing: it looks for Pods **[fwd: think "one or more containers
that run together"; full treatment in Part 8]** that haven't been assigned to a machine yet, picks
a machine, and writes that choice down. That's it. And the way we're going to prove what scheduling
*is* (and just as importantly, what it *isn't*) is by scheduling a Pod onto a machine **that doesn't
physically exist**. When it works — when the scheduler happily "places" a Pod on a ghost — you'll
understand something most people who use Kubernetes for years never quite see: **scheduling never
touches the actual machine. It's a pure bookkeeping decision recorded in etcd.**

> **You have `kubectl` now.** From this chapter on we mostly use it instead of raw `curl` — you
> proved in Part 3 that it's just a client of the same API, so there's nothing left to demystify.
> We'll still peek at etcd directly when seeing the raw truth matters.

---

## Priming questions

Guess first, don't look anything up.

1. A Pod object exists in etcd but hasn't been told which machine to run on. What *one field* on
   that Pod is empty — and what would "scheduling" therefore *concretely mean* in terms of that
   field?
2. The scheduler is "a loop that watches for unscheduled Pods and assigns them." Given what you saw
   in Parts 1–2, what is it **watching**, and when it decides, what does it physically **do** to
   record the decision? (It has no special power — it's just another API client.)
3. We're about to schedule a Pod onto a node that isn't a real machine — no OS, no container
   runtime, nothing. It'll "succeed." What does that tell you about the relationship between
   *scheduling* a Pod and *running* it?
4. After the scheduler assigns our Pod, its status will still say `Pending`, not `Running`. Why?
   What's missing that would move it to `Running` — and which chapter do you think adds it?
5. The scheduler needs to talk to the API server like any client. From Part 3, what does it need in
   order to be *allowed* to list Pods and create bindings — and what would happen if it presented
   the wrong identity?

---

## Assumed state

Resuming from a passed **Part 3** gate.

- You're on **`cp`**.
- etcd is healthy; `kube-apiserver` runs with `--authorization-mode=Node,RBAC` and your CA
  (`--client-ca-file`).
- You have the CA (`~/pki/ca.crt` + `~/pki/ca.key`), your `argv` and `admin` client certs, and a
  working `~/.kube/config`. `kubectl` is installed.
- Snapshot `part-03-kubectl-auth` exists.

Check on `cp`:

```bash
hostname                                  # -> cp
etcdctl endpoint health                   # healthy (start etcd if needed)
# start the apiserver (Part 3 command) if needed, then:
kubectl --context admin get namespaces >/dev/null 2>&1 && echo "apiserver + admin OK" \
  || echo "start apiserver / check admin context"
```

> If you don't yet have an `admin` context, Step 0 below creates it. You need admin (superuser)
> power for this chapter — recall `argv` was deliberately given almost none in Part 3.

---

## The mental model

The scheduler is the simplest possible illustration of a **control loop**, and every other
controller you meet later is a fancier version of the same three-beat rhythm:

```
   watch  ->  decide  ->  write
   (find Pods with     (pick a       (record the choice
    no nodeName)        machine)      by creating a Binding)
```

Concretely:

- **watch** — the scheduler opens a watch (the exact mechanism from Parts 1–2) for Pods whose
  `spec.nodeName` is empty. Those are "unscheduled."
- **decide** — for each such Pod, it runs the candidate nodes through *filters* ("can this Pod even
  fit here?") and *scores* ("which fit is best?"), and picks a winner. In a real cluster this is
  where resource requests, affinity, and taints matter **[fwd: Part 8]**. For us, with one bare
  node and a Pod that asks for nothing, the decision is trivial.
- **write** — it records the decision by creating a **Binding** object for the Pod, which sets the
  Pod's `spec.nodeName`. That write goes through the API server into etcd, like every write.

Notice what's *not* in that loop: contacting the chosen machine. The scheduler never SSHes anywhere,
never starts a container, never pings the node. It **only writes a name into a field.** Something
*else* — running on each machine — has to notice "a Pod was assigned to me" and actually run it.
That something is the kubelet, and it's Part 5. This chapter deliberately stops at the write, so you
feel the seam between *deciding* and *doing*.

That seam is why we can schedule onto a ghost node. The scheduler's whole world is objects in etcd;
if an object called `Node` exists, it's a valid target, whether or not any real hardware backs it.

---

## The build

### Step 0 — Become admin for these experiments

Part 3 left `kubectl` pointed at `argv`, who can barely do anything (that was the RBAC lesson). To
run experiments you need superuser power, so add an `admin` context backed by the `system:masters`
cert you made in Part 3:

```bash
cd ~/pki

kubectl config set-credentials admin \
  --client-certificate=$HOME/pki/admin.crt \
  --client-key=$HOME/pki/admin.key \
  --embed-certs=true

kubectl config set-context admin --cluster=byhand --user=admin
kubectl config use-context admin

kubectl get namespaces        # works — you're system:masters now
```

(If you'd rather be explicit each time, skip `use-context` and append `--context admin` to
commands. Either is fine.)

### Step 1 — Prove the absence: a Pod with nowhere to go

Before building the scheduler, feel why it's needed. Create a Pod — note the blanket **toleration**
**[fwd: a way of saying "this Pod may be placed even on a node marked not-ready"; taints/tolerations
get their real treatment later]** and the absence of resource requests, both of which make the Pod
trivially placeable so nothing *else* can be blamed later:

```bash
cat > ~/scheduler-demo.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: scheduler-demo
  namespace: default
spec:
  tolerations:
    - operator: "Exists"        # tolerate ANY taint (keeps a bare node from blocking us)
  containers:
    - name: app
      image: nginx:stable       # a valid spec; it will NOT actually run (no kubelet yet)
EOF

kubectl apply -f ~/scheduler-demo.yaml
```

Now look at it:

```bash
kubectl get pod scheduler-demo -o wide
# STATUS: Pending    NODE: <none>

kubectl get pod scheduler-demo -o jsonpath='{.spec.nodeName}{"\n"}'
# (empty line — no node assigned)
```

The Pod is real, stored in etcd, valid — and **stuck**. `spec.nodeName` is empty and nothing is
going to fill it, because the loop that fills it doesn't exist yet. Leave it Pending; you'll watch
it get rescued in a moment. (Answering priming Q1: "scheduling this Pod" means, concretely,
*setting that empty `spec.nodeName` field*.)

### Step 2 — Create a node that doesn't exist

The scheduler needs at least one `Node` object to consider. We give it one — a **ghost**, backed by
no hardware — precisely to prove scheduling is bookkeeping:

```bash
cat > ~/ghost-node.yaml <<'EOF'
apiVersion: v1
kind: Node
metadata:
  name: ghost-1
EOF

kubectl apply -f ~/ghost-node.yaml
kubectl get nodes
# NAME      STATUS     ROLES    AGE   VERSION
# ghost-1   NotReady   <none>   ...   <empty>
```

It shows `NotReady` — of course; there's no kubelet **[fwd: Part 5]** posting health for it,
because there's no machine. That's fine. The scheduler can still target it (and our Pod's blanket
toleration means the not-ready state won't block placement).

### Step 3 — Give the scheduler an identity and a binary

The scheduler is just an API client, so — exactly like `kubectl` in Part 3 — it needs a client
cert and a kubeconfig. Its identity matters: if it authenticates as the user
**`system:kube-scheduler`**, the API server's built-in RBAC already grants it precisely the
permissions a scheduler needs (there's a default `ClusterRole`/binding of that name, installed
automatically by the API server). So we mint a cert with that exact CN:

```bash
cd ~/pki

# client cert whose identity is the built-in scheduler user
openssl genrsa -out scheduler.key 2048
openssl req -new -key scheduler.key -subj "/CN=system:kube-scheduler" -out scheduler.csr
openssl x509 -req -in scheduler.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -days 365 -out scheduler.crt

# a dedicated kubeconfig for the scheduler (same shape you learned in Part 3)
KCFG=$HOME/pki/scheduler.kubeconfig
kubectl config set-cluster byhand \
  --server=https://127.0.0.1:6443 \
  --certificate-authority=$HOME/pki/ca.crt --embed-certs=true \
  --kubeconfig=$KCFG
kubectl config set-credentials system:kube-scheduler \
  --client-certificate=$HOME/pki/scheduler.crt \
  --client-key=$HOME/pki/scheduler.key --embed-certs=true \
  --kubeconfig=$KCFG
kubectl config set-context byhand \
  --cluster=byhand --user=system:kube-scheduler --kubeconfig=$KCFG
kubectl config use-context byhand --kubeconfig=$KCFG
```

Download the scheduler binary (**same `KVER`** as your API server):

```bash
KVER=v1.31.0
cd /tmp
curl -LO "https://dl.k8s.io/${KVER}/bin/linux/amd64/kube-scheduler"
chmod +x kube-scheduler && sudo mv kube-scheduler /usr/local/bin/
kube-scheduler --version
```

### Step 4 — Start the scheduler and watch the ghost get chosen

Open a **new pane**. In one pane, start a live watch on the Pod so you *see* the moment it flips:

```bash
kubectl get pod scheduler-demo -o wide -w
```

In another pane, start the scheduler:

```bash
sudo kube-scheduler \
  --kubeconfig=$HOME/pki/scheduler.kubeconfig \
  --leader-elect=false \
  --bind-address=127.0.0.1 \
  -v=2 \
  2>&1 | tee /tmp/scheduler.log
```

- `--kubeconfig` — the identity + endpoint you just built. This is how it reaches and is authorized
  by the API server.
- `--leader-elect=false` — with one scheduler there's no election to hold; disabling it keeps the
  lab simple **[fwd: leader election, Part 6]**.
- `-v=2` — verbose enough to see the decision in the log.

Within a second or two, the scheduler log prints something like *"Successfully bound pod
default/scheduler-demo to ghost-1."* And your watch pane updates:

```
NAME             READY   STATUS    ...   NODE
scheduler-demo   0/1     Pending   ...   ghost-1
```

The `NODE` column just went from `<none>` to `ghost-1`. Confirm the field directly:

```bash
kubectl get pod scheduler-demo -o jsonpath='{.spec.nodeName}{"\n"}'
# ghost-1

kubectl get pod scheduler-demo -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"\n"}{end}'
# PodScheduled=True
```

And see it in the raw store, tying back to Parts 1–2:

```bash
etcdctl get /registry/pods/default/scheduler-demo | strings | grep -A0 ghost-1
# you'll spot "ghost-1" embedded in the stored object — the decision, persisted
```

### Step 5 — Sit with the two facts that matter

Look carefully at the status:

```bash
kubectl get pod scheduler-demo -o wide
# STATUS: Pending      NODE: ghost-1
```

Two things are simultaneously true, and holding both is the entire lesson of this chapter:

1. **The Pod is scheduled.** `spec.nodeName=ghost-1`, `PodScheduled=True`. The scheduler did its
   whole job: it watched, decided, and wrote.
2. **The Pod is not running.** Status is still `Pending`. Nothing has pulled the image or started a
   container — because `ghost-1` is a ghost, and even on a real machine, *the scheduler doesn't run
   anything*. Running is somebody else's job.

You scheduled a Pod onto a machine that does not exist, and the control plane is perfectly happy,
because **scheduling is a decision recorded in etcd, not an action taken on hardware** (priming Q3).
The thing that will look at `spec.nodeName`, notice "that's me," and actually start the container is
the **kubelet** — a program that runs *on the machine* — and that's exactly what Part 5 builds. When
you add it, `Pending` finally becomes `Running` (priming Q4).

---

## Verification gate

You pass Part 4 when, without copying:

1. You create a Pod and show it stuck `Pending` with an empty `spec.nodeName`, and explain in one
   sentence what "unscheduled" means at the field level.
2. You start `kube-scheduler` and show the **same** Pod flip to `spec.nodeName=<node>` with
   `PodScheduled=True` — and you can point to the line in the scheduler log where it bound.
3. You can explain why the Pod is now **scheduled but still `Pending`/not running**, and name what's
   missing (the kubelet) and roughly when it arrives (Part 5).
4. You can state the scheduler's three-beat loop — **watch (unscheduled Pods) → decide (pick a node)
   → write (a Binding that sets nodeName)** — and note that it never contacts the machine.
5. You can say what identity the scheduler authenticated as and why that mattered (RBAC granted the
   `system:kube-scheduler` user its permissions).

If #3 and #4 are fluent, you've internalized the decide-vs-do seam that the rest of the control
plane is organized around.

---

## Troubleshooting

- **Pod stays `Pending` with `nodeName` empty after the scheduler starts.** The scheduler isn't
  binding. Read `/tmp/scheduler.log`:
  - `... cannot list resource "pods" ... forbidden` or similar **403** → the scheduler's identity is
    wrong. Its cert CN must be exactly `system:kube-scheduler` (case-sensitive). Re-check
    `openssl x509 -in ~/pki/scheduler.crt -noout -subject`.
  - `0/1 nodes are available: 1 node(s) had untolerated taint {node.kubernetes.io/not-ready ...}` →
    the ghost node carries a not-ready taint and your Pod isn't tolerating it. Confirm the
    `tolerations: [{operator: "Exists"}]` block is present in the Pod (Step 1). Re-apply the Pod.
  - `0/1 nodes are available: ... Insufficient cpu/memory` → your Pod has resource requests the
    (capacity-less) ghost can't satisfy. Remove requests for this demo.
- **`kube-scheduler` won't start / can't reach the API server.** Wrong `--server` in
  `scheduler.kubeconfig`, or the API server isn't running. Verify with
  `kubectl --kubeconfig ~/pki/scheduler.kubeconfig get pods` (should at least authenticate).
- **Scheduler log shows warnings about its own `healthz`/secure serving / authorization.** Harmless
  in this lab. Those refer to the scheduler's *own* metrics/health endpoint auth, not to scheduling.
  Scheduling only needs `--kubeconfig`. Ignore them.
- **`kubectl get nodes` shows `ghost-1` as `NotReady`.** Expected — there's no kubelet posting
  status. Not a problem for scheduling.
- **You expected `Running` and got `Pending`.** That is the **correct** result for this chapter, not
  a bug. No kubelet exists yet to run the container. Part 5.
- **Tangled things up?** Restore `part-03-kubectl-auth` and redo from Step 0. Nothing here is
  precious.

---

## What just happened (close the loop)

- **Q1 — what "scheduling" concretely is:** filling the empty `spec.nodeName` field on a Pod.
- **Q2 — what the scheduler watches and does:** it watches (via the API server) for Pods with no
  `nodeName`, and records its decision by creating a Binding — an ordinary authorized write, no
  special powers.
- **Q3 — scheduling vs. running:** you placed a Pod on a non-existent machine and it "succeeded,"
  proving scheduling is a control-plane bookkeeping decision that never touches hardware.
- **Q4 — why still `Pending`:** nothing has *run* the container. The component that watches for
  "Pods assigned to me" and actually starts them — the kubelet — doesn't exist yet. Part 5.
- **Q5 — the scheduler's identity:** it authenticated as `system:kube-scheduler`, which RBAC
  already empowers; a wrong CN would have been authenticated-but-forbidden (403), and it would have
  bound nothing.

You now have a database, a guarded API, and your first working **loop**. The shape you just
learned — *watch → decide → write* — is the template for every controller in Kubernetes. The
scheduler is special only in that its decision is "which node," recorded in one field.

But the whole thing is still inert on the *machine* side: decisions pile up in etcd with nothing to
enact them. In **Part 5** we finally cross from the control plane to a real worker: you'll install a
container runtime **[fwd: containerd]** on `worker-1`, run a **kubelet** there, watch it *register
itself* as a real `Node` (goodbye ghost), and watch it notice the Pod assigned to it and — at last
— pull the image and **run the container**, flipping `Pending` to `Running`. That's the moment the
cluster stops being a database and starts being a machine that does things.

---

## Lab log

Snapshot `cp` as `part-04-scheduler` and add a row to `LAB_LOG.md`:

```
| 2025-XX-XX | 04 | Y | part-04-scheduler | scheduler as first control loop; bound a Pod to a GHOST node; scheduled != running; nodeName is the whole job |
```

---

**Next → Part 5 — kubelet & a real node** *(coming next release)*

**[← Part 3](03-kubectl-auth.md)** · **[Index](README.md)**
