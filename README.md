# Kubernetes By Hand

Follow along in a nice website: [https://argandov.github.io/kubernetes-by-hand/](https://argandov.github.io/kubernetes-by-hand/)

> Build a Kubernetes cluster by hand, one component at a time, until nothing about it is magic.

This is not a "get a cluster running fast" guide. Tools like `minikube` and `kind` are
excellent — but they hand you a finished cluster the way a live-USB hands you a finished
Linux desktop. That's the opposite of the goal here. **The goal is to understand the
system**, so that if you never touch it again you keep the understanding, and if you do
touch it again nothing surprises you.

We build the control plane from its parts. You will run `etcd` by itself before Kubernetes
exists at all. You will talk to the API server with `curl` before you're allowed to use
`kubectl`. You will create a Pod that hangs forever because there is no scheduler yet — and
then start the scheduler and watch it move. Each component is introduced **at the moment you
feel its absence**, because that is when its job becomes obvious.

Security and observability are **not** separate topics here. They are not the point, and they
are not bolted on at the end. They fall out of the mechanism where they naturally live: audit
logging appears when we meet the API server, because that is the thing being audited; network
policy appears when we build the network, because that is the thing enforcing it; Falco and
seccomp appear once you understand the syscall path, because only then is it obvious what they
are watching. By the time you meet them, they are evident rather than mysterious.

---

## Who this is for

Someone comfortable on a Linux terminal who wants to understand Kubernetes as a *system* —
the control plane, the reconciliation model, why each moving part exists — rather than
memorize `kubectl` incantations. No prior Kubernetes knowledge is assumed. This is
deliberately **not** exam prep; if you want the CKA, get a CKA course. This is for
understanding.

---

## The method

Every chapter follows the same shape. Once you've read Part 0, the rhythm is automatic.

1. **Priming questions.** Each chapter opens with questions you *cannot yet answer*. Do not
   look anything up. Just read them and let your mind form guesses. This is the entire point:
   a primed mind reads *hunting* for answers instead of passively absorbing text. The questions
   get answered naturally as you build. (The technique is sometimes called "advance organizers"
   or "open loops" — you're creating a felt gap so the material has somewhere to land.)

2. **Assumed state.** A short precondition check — a couple of commands — that confirms your
   lab matches where the chapter expects to begin. This is your re-entry point after a break.

3. **The mental model.** A short conceptual frame *before* the hands-on work, so you know what
   you're looking at.

4. **The build.** You do the work, in a terminal, alongside the manual. Read in one pane, work
   in another.

5. **The verification gate.** A concrete command whose output *proves* you understood. This is
   binary. Gate passed = chapter done, move on. Gate not passed = you're not done yet. There is
   no ambiguity about whether you finished.

6. **Troubleshooting.** The handful of failure modes this specific chapter tends to produce,
   and how to reason about each from first principles.

7. **Lab log entry.** One line in `LAB_LOG.md`. Sixty seconds. This is what lets you return
   after a week without reconstructing where you were.

**Depth calibration:** we go deep on *mechanism* (how a watch stream works, how a reconcile
loop converges, how a packet physically crosses nodes) and deliberately shallow on *feature
surface* (we will not enumerate every field of every object — once you understand one
controller, you understand the pattern). Understanding, not coverage.

---

## The anti-drift system

Self-directed hands-on courses rarely die from difficulty. They die from four structural
failures. Three of them are engineered out of existence here; the fourth is yours.

- **"Am I done?" ambiguity** → killed by the **verification gate**. Binary definition of done.
- **Lost state between sessions** → killed by the **assumed-state check** at the top of each
  chapter plus the **lab log**. You never have to reconstruct where you were.
- **The debugging death-spiral** (burn three hours on a wrong certificate, get demoralized,
  quit) → killed by the **snapshot-per-chapter escape hatch**. Because your nodes are clones of
  a template and you snapshot at every chapter boundary, *no session can ever leave you in an
  unrecoverable hole*. Wedged beyond repair? Restore the snapshot or reclone and reapply the
  chapter. The environment can never trap you. Hard rule: **stuck more than ~30 minutes on
  something the manual didn't warn you about → restore the snapshot and start the section
  fresh** rather than grinding.
- **Showing up** → yours. Nothing can manufacture the willingness to open the file. What this
  design *can* do is make re-entry frictionless: each chapter's assumed-state check tells you
  your single next action. A small predictable slot beats an ambitious one you keep postponing.

---

## The lab

A vendor-neutral contract. Run it on Proxmox, libvirt, VirtualBox, or three cloud VMs — the
manuals never assume a specific hypervisor. See **[Part 0](00-the-lab.md)** for the full build.

| Role          | Count | vCPU | RAM  | Disk  | Why it exists                                              |
|---------------|:-----:|:----:|:----:|:-----:|-----------------------------------------------------------|
| control-plane |   1   |  2   | 4 GB | 20 GB | will run etcd + apiserver + scheduler + controller-manager |
| worker        |   2   |  2   | 2 GB | 20 GB | **two**, so scheduling is a real choice and cross-node networking is actually exercised |

Baseline OS: **Debian 12 (bookworm), x86_64**. Requirements: the three nodes share one flat
L2 network, can reach each other and the internet, and your platform can snapshot a VM.

---

## Syllabus

| #   | Module                              | What you'll actually understand                                          |
| --- | ----------------------------------- | ------------------------------------------------------------------------ |
| 00  | [The Lab](00-the-lab.md)            | Why 3 nodes; the build/snapshot/log discipline                           |
| 01  | [etcd alone](01-etcd.md)            | The source of truth is *just* a key-value store                          |
| 02  | [kube-apiserver](02-apiserver)      | Talk to it raw with `curl`; the REST + watch model; audit logging        |
| 03  | [kubectl & auth](03-kubectl-auth)   | Certs, users, RBAC — how the API answers "can this caller do this?"      |
| 04  | scheduler                           | Watch a Pod hang `Pending`, then get bound                               |
| 05  | kubelet & a real node               | CRI/containerd, node registration, CSR approval                          |
| 06  | controller-manager & reconciliation | The heart: kill pods, watch them return                                  |
| 07  | networking                          | The packet's physical path; kube-proxy; Services; CoreDNS; NetworkPolicy |
| 08  | the workload object model           | Deployments/StatefulSets/DaemonSets/Jobs; probes; QoS                    |
| 09  | storage                             | Volumes, PV/PVC, CSI, the attach/mount dance                             |
| 10  | extension machinery                 | ConfigMaps/Secrets, admission webhooks, CRDs *(optional Go appendix)*    |
| 11  | runtime & the kernel boundary       | The syscall path; now seccomp/Falco are obvious                          |
| 12  | observability                       | metrics-server, kube-state-metrics, Prometheus, control-plane telemetry  |
| 13  | capstone: failure forensics         | Expired certs, corrupt etcd, wedged scheduler — diagnose cold            |

Modules are released a few at a time, because each one assumes the exact lab state the previous
one produced.

---

## How to follow along

Clone the repo and open the folder as an **Obsidian vault**, read on GitHub, or read on [https://argandov.github.io/kubernetes-by-hand/](https://argandov.github.io/kubernetes-by-hand/). Keep the
manual in one pane and a terminal in another. Copy `LAB_LOG.md` and start filling it from
Part 0 onward.

Start here → **[Part 0 — The Lab](00-the-lab.md)**
