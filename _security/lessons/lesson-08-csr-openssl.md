---
title: "Lesson 8 — CSRs & the openssl Toolkit"
nav_order: 8
parent: "Phase 2: PKI & Certificates"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 8: CSRs & the openssl Toolkit

## Concept

To get a certificate you send the CA a **Certificate Signing Request (CSR)**: a bundle containing your
**public key** and the identity you're requesting (hostnames), **signed by your private key** to prove
you hold it. The CA validates your claim, then issues a certificate. Crucially, your **private key never
leaves your machine**.

```
   you:  generate keypair → build CSR (public key + desired SANs, self-signed) → send CSR
   CA:   validate identity → issue certificate (CA adds issuer, serial, dates, signs it)
```

---

## How it works

**What's in a CSR vs what the CA adds.** The CSR carries the **subject/SANs you want** and your
**public key**, self-signed to prove key possession. The **CA decides and adds** the issuer, serial
number, validity dates, and the actual trust-conferring signature — and may override/ignore fields you
requested per its policy. So you *request* identity; the CA *grants* it.

**The openssl tools you'll use repeatedly:**
- `openssl genpkey` — generate a private key (RSA or EC).
- `openssl req` — create a CSR (or a self-signed cert).
- `openssl x509` — inspect/convert/sign certificates.
- `openssl ca` — operate as a CA, signing CSRs (Lesson 12).

**SANs go in the CSR via config.** Since SAN is what matters (Lesson 6), you specify the hostnames in
the CSR — typically through a config file's `[alt_names]` section or the modern `-addext
"subjectAltName=DNS:..."` flag.

**Keep the private key private.** The whole model rests on the private key never being disclosed. It's
generated locally, used to sign the CSR, then used by your server — it is never sent to the CA. Anyone
who obtains it can impersonate you until the cert is revoked.

{: .note }
> **Why "key possession" is proven by self-signing the CSR**
> The CSR is signed by the very private key whose public half it contains. The CA verifies that
> self-signature, proving the requester actually holds the private key for the public key being
> certified — otherwise someone could request a certificate for <em>your</em> public key. It binds the
> request to key ownership.

---

## Lab

```bash
# 1. Generate a private key (EC P-256)
$ openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out server.key

# 2. Create a CSR with SANs (modern one-liner)
$ openssl req -new -key server.key -out server.csr \
    -subj "/CN=www.example.test" \
    -addext "subjectAltName=DNS:www.example.test,DNS:example.test"

# 3. Inspect the CSR — note the requested SANs and your public key, and that it's self-signed
$ openssl req -in server.csr -noout -text | sed -n '1,20p'
$ openssl req -in server.csr -noout -verify       # "verify OK" = valid self-signature (key possession)
```

---

## Further Reading

| Topic | Link |
|---|---|
| CSR | [Wikipedia — Certificate signing request](https://en.wikipedia.org/wiki/Certificate_signing_request) |
| PKCS #10 (CSR format) | [Wikipedia — PKCS](https://en.wikipedia.org/wiki/PKCS) |
| `openssl req` | [openssl req](https://docs.openssl.org/master/man1/openssl-req/) |
| `openssl genpkey` | [openssl genpkey](https://docs.openssl.org/master/man1/openssl-genpkey/) |

---

## Checkpoint

**Q1. What does a CSR contain, and what does the CA add when issuing the certificate?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A <strong>CSR contains the requester's public key and the identity they want</strong> (subject and,
importantly, the SAN hostnames), and it is <strong>self-signed with the corresponding private key</strong>
to prove key possession. The CA, after validating the request, <strong>adds the parts that confer
trust</strong>: the issuer (itself), a unique serial number, the validity period (notBefore/notAfter),
any policy-mandated extensions, and — critically — its <strong>signature</strong> over the whole
certificate. The CA may also enforce its own policy, overriding or ignoring fields you requested. In
short: you supply the public key and requested identity; the CA decides issuer/serial/dates and signs.
</details>

---

**Q2. The private key is central to all of this — where does it go during the CSR process, and why does that matter?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The private key is <strong>generated locally and never leaves your machine</strong>. It's used to
self-sign the CSR (proving you hold it) and later by your server to perform TLS, but it is <em>not</em>
sent to the CA — the CA only ever sees your public key. This matters because the entire security model
depends on the private key staying secret: anyone who obtains it can impersonate your service (decrypt/
sign as you) until the certificate is revoked and replaced. Sending the private key anywhere, or letting
a third party generate it for you, breaks that guarantee. So CSR-based issuance is designed precisely so
the secret never has to be shared.
</details>

---

**Q3. Why is a CSR self-signed with the requester's own private key?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
To <strong>prove possession of the private key</strong> corresponding to the public key in the request.
The CSR is signed by the very private key whose public half it carries, and the CA verifies that
self-signature before issuing. Without this, an attacker could submit a CSR containing <em>someone
else's</em> public key and obtain a certificate binding that key to a name they control (or vice versa),
causing confusion or enabling attacks. The self-signature binds the request to actual ownership of the
key pair, so the CA knows the entity requesting the certificate genuinely controls the private key it's
about to certify.
</details>

---

## Homework

Create two CSRs from the same private key: one using only a CN (no SAN) and one with proper SANs. Have a
local CA (or `openssl x509 -req`) sign both, then try to use each with `openssl verify`/a browser against
the hostname. Explain why the CN-only certificate is rejected by modern clients and what that says about
where identity must be expressed.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Both CSRs use the same key, so both can be signed into valid-looking certificates, but they behave
differently at validation: the <strong>SAN certificate is accepted</strong> for its listed hostnames,
while the <strong>CN-only certificate is rejected</strong> by modern clients for hostname matching —
browsers and current TLS libraries <em>ignore the Common Name</em> and require the hostname in the
<strong>Subject Alternative Name</strong> extension. So even though the CN-only cert is cryptographically
well-formed and properly signed, a client checking <code>www.example.test</code> finds no matching SAN and
fails with a name-mismatch error. The lesson: identity (the hostnames a cert is valid for) <strong>must
be expressed in the SAN</strong>, not the CN — the CN is legacy/cosmetic, and any tooling or process that
still relies on it produces certificates that won't work with real clients.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 9 — Certificate Validation →](lesson-09-validation){: .btn .btn-primary }
