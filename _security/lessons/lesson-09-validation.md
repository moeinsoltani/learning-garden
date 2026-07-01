---
title: "Lesson 9 — Certificate Validation"
nav_order: 9
parent: "Phase 2: PKI & Certificates"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 9: Certificate Validation

## Concept

Receiving a certificate isn't enough — the client must **validate** it before trusting the connection.
Validation is a checklist; skipping any item creates a hole. The famous insecurity isn't usually broken
crypto, it's *clients that didn't check something* ("it's encrypted" ≠ "it's the right server").

```
   Got a cert? Run the checklist:
   ☐ chains to a trusted root   ☐ within validity dates   ☐ hostname matches a SAN
   ☐ each signature valid       ☐ CA flags/constraints OK ☐ key usage permits this use
```

---

## How it works

Every check a client must pass:

1. **Chain to a trusted root** (Lesson 7) — build leaf → intermediate(s) → trusted root, verifying each
   **signature**.
2. **Validity dates** — `notBefore ≤ now ≤ notAfter` for every cert in the chain (clock-dependent —
   cf. networking time-sync lesson).
3. **Hostname matching** — the name you connected to must match a **SAN** entry (exact or wildcard).
4. **Basic Constraints** — issuers must have `CA:TRUE`; the leaf must not be used as a CA; path-length
   limits respected.
5. **Key Usage / EKU** — e.g. a server cert must have "TLS Web Server Authentication" EKU; a cert
   without the right usage is rejected for that purpose.
6. **Revocation** — not revoked (Lesson 10).

**Name constraints** can restrict an intermediate to issuing only under certain domains — a guard so a
delegated sub-CA can't mint certs for the whole internet.

**The classic validation bugs.** Real breaches came from clients that: skipped hostname verification
(accepting any valid cert for *any* site), didn't check Basic Constraints (so a leaf could sign others),
or trusted expired/wrong-usage certs. Libraries now do this by default — but only if you don't disable
it (the infamous `verify=False`).

{: .note }
> **"Encrypted" is not "authenticated"**
> A connection can be perfectly encrypted to an <em>attacker</em> if you skip validation — that's a
> man-in-the-middle. Confidentiality without verifying <em>who</em> you're encrypting to is worthless.
> Validation is what ties the encryption to the right identity, which is the entire point of the PKI.

---

## Lab

```bash
# Full validation against the system trust store, with hostname check
$ openssl s_client -connect wikipedia.org:443 -servername wikipedia.org -verify_hostname wikipedia.org \
    </dev/null 2>/dev/null | grep -E 'Verify return code|Verification'
Verification: OK

# Force a hostname mismatch and watch it fail
$ openssl s_client -connect wikipedia.org:443 -servername wikipedia.org \
    -verify_hostname wrong.example.com </dev/null 2>/dev/null | grep -i verify
# verification fails — the SAN doesn't include wrong.example.com

# Inspect validity dates and EKU of a cert
$ openssl x509 -in leaf.pem -noout -dates -ext extendedKeyUsage,basicConstraints
```

---

## Further Reading

| Topic | Link |
|---|---|
| Certificate validation / path validation | [Wikipedia — Certification path validation algorithm](https://en.wikipedia.org/wiki/Certification_path_validation_algorithm) |
| Hostname verification | [Wikipedia — Transport Layer Security](https://en.wikipedia.org/wiki/Transport_Layer_Security) |
| Extended Key Usage | [Wikipedia — X.509](https://en.wikipedia.org/wiki/X.509) |
| `openssl verify` | [openssl verify](https://docs.openssl.org/master/man1/openssl-verify/) |

---

## Checkpoint

**Q1. List the checks a client must pass before trusting a server certificate.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <strong>Chain to a trusted root</strong> — build and verify leaf → intermediate(s) → a root in the
trust store, checking each <strong>signature</strong>. (2) <strong>Validity dates</strong> — current time
within every certificate's notBefore/notAfter. (3) <strong>Hostname matching</strong> — the connected
hostname matches a <strong>SAN</strong> entry (exact or wildcard). (4) <strong>Basic Constraints</strong>
— issuers are marked <code>CA:TRUE</code>, the leaf isn't usable as a CA, path-length limits respected.
(5) <strong>Key Usage / Extended Key Usage</strong> — the certificate is authorized for this purpose (e.g.
TLS server authentication). (6) <strong>Revocation status</strong> — the cert hasn't been revoked
(CRL/OCSP, Lesson 10). (Plus any <strong>name constraints</strong> on intermediates.) Failing any one of
these means the certificate must be rejected.
</details>

---

**Q2. Why is "the connection is encrypted" not sufficient for security?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because encryption only guarantees that the data is unreadable to passive eavesdroppers — it says nothing
about <em>who</em> is on the other end. If the client skips certificate validation (or accepts any
certificate), it can establish a perfectly encrypted channel <strong>to an attacker</strong> who is
sitting in the middle, decrypting and relaying your traffic — a man-in-the-middle. The attacker presents
their own cert/key, you encrypt to them, and "encrypted" gives you false confidence. Validation —
verifying the certificate chains to a trusted root and the hostname matches — is what binds the encryption
to the <em>correct identity</em>. Without it, confidentiality is meaningless because you might be
confidentially talking to the wrong party. Encryption needs authentication to be useful.
</details>

---

**Q3. Name two historic certificate-validation bugs and what an attacker could do because of them.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Two classic classes: (1) <strong>Missing hostname verification</strong> — many clients/libraries verified
that a certificate chained to a trusted CA but failed to check that its SAN matched the hostname being
visited. An attacker with <em>any</em> valid CA-issued certificate (e.g. for their own domain) could then
present it for <em>another</em> site and the client would accept it, enabling man-in-the-middle on
arbitrary HTTPS connections. (2) <strong>Not checking Basic Constraints (<code>CA:TRUE</code>)</strong> —
some early implementations didn't verify that an issuing certificate was actually permitted to be a CA, so
an attacker holding an ordinary leaf certificate could use it to <strong>sign certificates for any
hostname</strong> and have them accepted — a total break of the trust hierarchy (the basis of an early,
widely-publicized SSL flaw). Both show the theme: the danger is usually a <em>skipped check</em>, not
broken cryptography — which is why libraries now enforce these by default and disabling them (e.g.
<code>verify=False</code>) is so dangerous.
</details>

---

## Homework

Use a deliberately broken certificate (expired, wrong hostname, or self-signed) — `badssl.com` offers
test endpoints like `expired.badssl.com`, `wrong.host.badssl.com`, `self-signed.badssl.com`. For each,
run `openssl s_client` (or curl) and record exactly which validation check fails and the error message.
Then explain why a developer using `curl -k` / `verify=False` to "make it work" is dangerous.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Each endpoint fails a <em>specific</em> check: <code>expired.badssl.com</code> fails the
<strong>validity-date</strong> check (error like "certificate has expired"); <code>wrong.host.badssl.com</code>
fails <strong>hostname matching</strong> (the SAN doesn't include the connected name — "certificate
subject name does not match"); <code>self-signed.badssl.com</code> fails the <strong>chain-to-trusted-
root</strong> check ("self-signed certificate" / "unable to get local issuer certificate"). Each maps to
one item on the validation checklist. Using <strong><code>curl -k</code> / <code>verify=False</code></strong>
is dangerous because it disables <em>all</em> of these checks at once: the connection is still encrypted,
but you'll now accept expired certs, wrong-host certs, untrusted/self-signed certs, and — critically —
<strong>an attacker's certificate</strong>, turning the connection into an easy man-in-the-middle target.
It converts "verified, authenticated TLS" into "encrypted to whoever answered," which is barely better
than plaintext against an active attacker. The right fix is to install the proper cert/chain or trust the
correct CA, never to switch off verification.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 10 — Revocation: CRL, OCSP, Stapling →](lesson-10-revocation){: .btn .btn-primary }
