# Part 2.5: PKI Fundamentals — Trust, Certificates, and the Handshake

> ⏱️ Estimated time: 90–120 min | Difficulty: Moderate

## Assumed State

- Part 2 complete. Bearer token auth works; you can `curl` the API server with `Authorization: Bearer $TOKEN`.
- Control-plane node, Debian 12, `openssl` ≥ 3.0 (`openssl version`).
- No Kubernetes commands in this part. This is a detour — you leave the cluster alone and come back to it in Part 3.

## Why this part exists

Part 3 replaces the bearer token with a CA, client certs, and RBAC. If the mechanics of a CA aren't solid, every command in Part 3 is a spell you're copying, not something you understand. This part builds a CA, a server cert, and a client cert by hand, breaks them on purpose, and runs a real TLS handshake — until none of it is magic.

## Priming questions — don't answer yet, just hold them

1. If a CA's private key is just a private key, why can't anyone with *some* private key forge a cert that your CA would accept?
2. In a TLS connection, who proves identity to whom — always the server? Always both?
3. You have a valid cert issued to `server-a`. What actually stops you from copying it onto `server-b` and using it there?
4. What field does a client actually check against the hostname it's connecting to?

You'll answer all four cold by the end. Work happens on the control-plane node, `~/pki-lab/`.

```bash
mkdir -p ~/pki-lab && cd ~/pki-lab
```

---

## 1. The one paragraph you need

A private key signs. The matching public key verifies. You cannot derive the private key from the public one, and you cannot forge a valid signature without the private key. That's it — that's the entire mathematical trust anchor. Everything below is what you build *on top* of that one fact. You don't need to know how the math does this, only that it does.

---

## 2. Build a CA by hand

A CA is nothing but a private key plus a self-signed certificate. "Self-signed" means Issuer == Subject — it vouches for itself because nothing vouches for it. It's the root; trust starts there or it doesn't start.

```bash
# CA private key
openssl genrsa -out ca.key 2048

# self-signed CA certificate (this key vouching for itself)
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -out ca.crt -subj "/CN=kubernetes-by-hand-ca"
```

Inspect it:

```bash
openssl x509 -in ca.crt -text -noout
```

Find and understand two lines:
- `Issuer` and `Subject` are identical → self-signed.
- `X509v3 Basic Constraints: CA:TRUE` → this cert is *allowed to sign other certs*. Without this flag, a cert can't act as a CA even if you have its key.

**Checkpoint:** you now hold the only thing that can turn a CSR into a trusted cert — `ca.key`. Guard it like you'd guard a root password, because it functionally is one.

---

## 3. Issue a server certificate

A CSR is a request: "I generated this keypair, here's my public key, here's who I claim to be — please sign it." The CA never sees or needs the requester's private key.

```bash
# server's own keypair
openssl genrsa -out server.key 2048

# CSR, with SAN — modern TLS clients ignore CN and check SAN instead
openssl req -new -key server.key -out server.csr \
  -subj "/CN=server-a" \
  -addext "subjectAltName=DNS:server-a,IP:10.0.0.11"

# CA signs it → this is the moment trust gets conferred
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt -days 365 -sha256 -copy_extensions copy
```

Inspect the result:

```bash
openssl x509 -in server.crt -text -noout
```

Now `Issuer = kubernetes-by-hand-ca`, `Subject = server-a`, and the SAN block is present. The CA never touched `server.key` — it only ever saw the CSR (public info) and signed it with its own private key.

---

## 4. Verify the chain — and break it on purpose

```bash
openssl verify -CAfile ca.crt server.crt
# -> server.crt: OK
```

Now break it two ways, and watch it fail both times. This is answering priming question 3 experientially.

```bash
# tamper: flip a byte in a copy, verify against the original
cp server.crt server-tampered.crt
sed -i 's/A/B/1' server-tampered.crt
openssl verify -CAfile ca.crt server-tampered.crt   # fails — signature no longer matches content
rm server-tampered.crt   # server.crt itself was never touched, no need to regenerate

# wrong CA: sign with an unrelated CA and verify against the original
openssl genrsa -out rogue-ca.key 2048
openssl req -x509 -new -nodes -key rogue-ca.key -days 365 -out rogue-ca.crt -subj "/CN=rogue-ca"
openssl x509 -req -in server.csr -CA rogue-ca.crt -CAkey rogue-ca.key -CAcreateserial -out rogue-server.crt -days 365 -copy_extensions copy
openssl verify -CAfile ca.crt rogue-server.crt   # fails — signed by a key ca.crt doesn't trust
```

Verification is arithmetic, not vibes. A cert is only ever as trustworthy as the CA key that signed it — copying a valid cert to another machine (question 3) doesn't help you without the matching *private key*, which never left `server.key`.

---

## 5. One-way TLS, live

This is what happens every time you hit an HTTPS site. Only the server proves identity.

Terminal A:
```bash
openssl s_server -accept 8443 -cert server.crt -key server.key -CAfile ca.crt
```

Terminal B:
```bash
openssl s_client -connect localhost:8443 -CAfile ca.crt
```

Look for `Verify return code: 0 (ok)` in terminal B's output. What just happened:
1. Client connects, server sends `server.crt`.
2. Client checks: is this signed by a CA I trust (`ca.crt`)? Yes.
3. Client checks: does the SAN match the host I dialed? (`localhost` vs `server-a` — in a real setup this must match, that's answering question 4).
4. Session key gets negotiated using the server's public key. Server proved identity. **Client proved nothing.**

Ctrl+C both sides.

---

## 6. Mutual TLS (mTLS) — this is what Kubernetes runs everywhere

Same pattern, other direction: the client also gets a cert, and the server demands it.

```bash
# client keypair + CSR + sign — note the O= field, you'll need it in a minute
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr -subj "/CN=client-a/O=developers"
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out client.crt -days 365 -sha256 -copy_extensions copy
```

Terminal A — server now *requires* a verified client cert:
```bash
openssl s_server -accept 8443 -cert server.crt -key server.key -CAfile ca.crt -Verify 1
```

Terminal B — client presents its cert:
```bash
openssl s_client -connect localhost:8443 -CAfile ca.crt -cert client.crt -key client.key
```

Both sides now show verify OK. Both sides proved identity with the same mechanism, just in opposite directions. This *is* mTLS — nothing more exotic happens inside a Kubernetes cluster.

---

## 7. Map it back onto Part 3

| What you just built | What it becomes in Part 3 |
|---|---|
| `ca.key` / `ca.crt` | The cluster CA. `kube-apiserver --client-ca-file=ca.crt` |
| Server cert + `-Verify 1` requirement | `kube-apiserver`'s own serving cert, and its requirement that clients present a cert |
| `client.crt` with `CN=client-a` | The **CN of a client cert becomes the Kubernetes username** — no separate user database exists |
| `client.crt` with `O=developers` | The **O field becomes group membership**, which RBAC `RoleBindings` bind against |
| `client.key` + `client.crt`, presented on connect | Exactly the `client-certificate` / `client-key` fields in your kubeconfig |

That CN → username, O → group mapping is the single biggest unlock for Part 3. Kubernetes doesn't have a user table. Identity *is* whatever the CA vouched for in the cert.

---

## Stop condition — self-check

Answer the four priming questions now, out loud, no notes:

1. Why can't anyone forge a cert your CA accepts, even with some other private key?
2. Who proves identity in one-way TLS vs mTLS?
3. Why doesn't a copied `server.crt` let you impersonate the original server?
4. What field does a client check against the hostname?

If any answer is shaky, redo only that section — not the whole part. If all four are solid, stop here. Do **not** chase revocation (CRL/OCSP), cert-manager, Vault's PKI engine, or intermediate CA chains right now — those are real topics, but they're not blocking Part 3, and they're exactly the kind of thing that eats a weekend if you let curiosity drive instead of the stop condition. Park them in `BACKLOG.md` if you want a marker.

## LAB_LOG.md entry

```
YYYY-MM-DD - Part 2.5 (PKI interlude) - built CA, server cert, client cert by hand;
verified chain, broke it twice on purpose, ran live one-way and mutual TLS handshakes
with openssl s_server/s_client. Key unlock: CN->k8s username, O->k8s group, both come
from the CA-signed cert, not a user database.
```

## Snapshot checkpoint

Snapshot the control-plane node now, before touching Part 3: `pki-interlude-complete`. Everything in `~/pki-lab/` was scratch work for this part only — Part 3 builds its own CA and certs from scratch inside the actual cluster config paths, it doesn't reuse these files.
