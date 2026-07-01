---
title: "Lesson 18 — Mutual TLS (mTLS)"
nav_order: 18
parent: "Phase 3: TLS & SSL"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 18: Mutual TLS (mTLS)

## Concept

In ordinary TLS, only the **server** proves its identity with a certificate; the client stays anonymous
(it logs in separately, with a password or token). **Mutual TLS (mTLS)** adds the reverse: the **client
also presents a certificate**, so *both* ends are cryptographically authenticated by the TLS layer itself.

```
   Ordinary TLS:  client ──verifies──► server's cert        (server authenticated; client anonymous)
   mTLS:          client ◄──verifies──► server              (both present + verify certificates)
```

This is the foundation of service-to-service authentication (Phase 6).

---

## How it works

**The extra handshake step.** During the handshake, the server sends a **CertificateRequest**, asking the
client for a cert. The client responds with its **Certificate** and a **CertificateVerify** (a signature
proving it holds the matching private key). The server validates the client's cert against *its* trusted
CA (often a private CA, Lesson 12), exactly as the client validates the server's. Both directions use the
same chain-of-trust validation (Lesson 9).

**What it proves.** Ordinary TLS answers "is this the real server?" mTLS additionally answers "**is this a
legitimate client?**" — at the transport layer, before any application logic runs. The client's identity is
its certificate (e.g. a service name), not a bearer token that could be stolen and replayed.

**Where it's used.**
- **Service-to-service** auth inside a system / mesh — each service has a cert; only certified peers
  connect (Phase 6, SPIFFE/SPIRE).
- **Device identity** — IoT/managed devices authenticate with client certs.
- **High-security APIs / admin access** — require a client cert in addition to other factors.

**Trust config on both ends.** Each side needs: its own cert+key, and the **CA(s) it trusts** for the
*other* side. Commonly an internal private CA issues all the client and server certs, and both sides trust
that CA's root.

{: .note }
> **Why mTLS beats bearer tokens for machines**
> A bearer token (API key, JWT) is "something you have" that, if stolen, anyone can replay. An mTLS client
> proves possession of a <em>private key that never leaves the client</em> — there's no replayable secret
> on the wire, and the identity is bound to the key. That's why service meshes use mTLS for workload
> identity rather than shared API keys (Lesson 35).

---

## Lab

```bash
# Using the private CA from Lesson 12, issue a CLIENT cert too (client.key/client.crt),
# signed by the same CA as the server cert.

# Start a TLS server that REQUIRES a client certificate
$ openssl s_server -accept 4433 -cert server.crt -key server.key \
    -CAfile root.crt -Verify 1 -www &

# Connect WITHOUT a client cert → rejected
$ echo | openssl s_client -connect localhost:4433 2>&1 | grep -i 'alert\|verify\|handshake fail'
# the server demands a client cert and the handshake fails

# Connect WITH a client cert → success
$ echo | openssl s_client -connect localhost:4433 -cert client.crt -key client.key -CAfile root.crt 2>/dev/null \
    | grep -E 'Verify return code|Cipher'
Verify return code: 0 (ok)
```

---

## Further Reading

| Topic | Link |
|---|---|
| Mutual authentication | [Wikipedia — Mutual authentication](https://en.wikipedia.org/wiki/Mutual_authentication) |
| Client certificate | [Wikipedia — Client certificate](https://en.wikipedia.org/wiki/Client_certificate) |
| Certificate validation | [Lesson 9 — Certificate validation](lesson-09-validation) |
| Workload identity | [Lesson 35 — mTLS & workload identity](lesson-35-workload-identity) |

---

## Checkpoint

**Q1. What does mTLS prove that ordinary server-only TLS does not?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Ordinary TLS authenticates only the <strong>server</strong> — the client verifies the server's certificate
but the client itself is anonymous at the TLS layer (it proves who it is later, e.g. via a password or
token). <strong>mTLS</strong> additionally authenticates the <strong>client</strong>: the client presents
its own certificate and proves possession of the matching private key, and the server validates it against
a trusted CA. So mTLS proves "this is a legitimate, identified client" at the transport layer, before any
application logic — both ends are cryptographically authenticated to each other, rather than just the
server to the client.
</details>

---

**Q2. What extra handshake messages make mTLS happen, and how does each side establish trust?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The server adds a <strong>CertificateRequest</strong> to the handshake, asking the client to authenticate.
The client then sends its <strong>Certificate</strong> plus a <strong>CertificateVerify</strong> — a
signature over the handshake proving it holds the private key for that certificate. Each side establishes
trust the same way (Lesson 9): the client validates the server's certificate chain against the CAs it
trusts, and the server validates the client's certificate chain against the CA(s) <em>it</em> trusts
(often an internal/private CA that issued all the client certs). So both ends need their own cert+key and
a configured set of trusted CAs for verifying the other side — typically a shared private-CA root.
</details>

---

**Q3. Why is an mTLS client certificate generally a stronger machine credential than a bearer token?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A bearer token (API key, JWT) is a secret that grants access to <em>anyone who presents it</em> — if it's
stolen (leaked logs, intercepted, exfiltrated), an attacker can simply replay it. An mTLS client
certificate authenticates by <strong>proving possession of a private key that never leaves the client</strong>:
the client signs a handshake-specific value (CertificateVerify), so there's no reusable secret transmitted
that an eavesdropper could capture and replay, and the identity is cryptographically bound to the key. To
impersonate the client an attacker would have to steal the private key itself (which can be hardware-
protected and isn't sent anywhere). That's why service-to-service / workload authentication (Lesson 35)
favors mTLS over shared bearer secrets — it removes the replayable-credential-on-the-wire problem.
</details>

---

## Homework

Set up an mTLS server that trusts your private CA, then test three clients: one with a valid cert from that
CA, one with a cert from a *different* CA, and one with no cert. Record what happens in each case and
explain how the server decides to accept or reject, mapping it to the certificate-validation checklist
(Lesson 9).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Results: the client with a <strong>valid cert from the trusted CA</strong> connects successfully; the
client with a cert from a <strong>different CA</strong> is rejected; the client with <strong>no cert</strong>
is rejected (the handshake fails at CertificateRequest). The server decides exactly by running the
certificate-validation checklist (Lesson 9) against the <em>client's</em> certificate: does it
<strong>chain to a CA the server trusts</strong>? (The different-CA cert fails here — no path to the
server's trusted root.) Is it <strong>within its validity dates</strong>, not revoked, and does it have the
appropriate <strong>key usage / EKU</strong> (client authentication)? And the client must prove
<strong>key possession</strong> via CertificateVerify. The no-cert client fails simply because it provides
nothing to validate. So mTLS is just the standard chain-of-trust validation applied in the client→server
direction: accept only certificates that chain to a trusted CA and pass all the usual checks, which is why
the issuing CA (your private CA) is the linchpin of who's allowed in.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 19 — SNI, ALPN, Resumption & 0-RTT →](lesson-19-sni-alpn-resumption){: .btn .btn-primary }
