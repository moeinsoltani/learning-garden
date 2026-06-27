---
title: "Lesson 6 — X.509 Certificates Anatomy"
nav_order: 6
parent: "Phase 2: PKI & Certificates"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 6: X.509 Certificates Anatomy

## Concept

A public key alone proves nothing — anyone can generate one and claim to be `google.com`. A
**certificate** binds a public key to an **identity** (a hostname), and a trusted third party (a CA,
Lesson 7) **signs** that binding. An [X.509](https://en.wikipedia.org/wiki/X.509) certificate is the
standard format for that signed statement.

```
   "The public key  ABC...  belongs to  www.example.com,
    valid 2026-01-01 to 2027-01-01"           ← the binding
   — signed by: Example CA                     ← the signature (Lesson 4)
```

A certificate is essentially *a signed claim about who owns a public key*.

---

## How it works

**The core fields:**
- **Subject** — who the cert is *for* (the entity/hostname).
- **Issuer** — who signed it (the CA).
- **Validity** — `notBefore` / `notAfter` dates.
- **Public key** — the key being bound.
- **Serial number** — unique per issuer (used in revocation).
- **Signature** — the issuer's signature over all the above.

**Extensions** carry the modern, important stuff:
- **SAN (Subject Alternative Name)** — the list of hostnames/IPs the cert is valid for. *This* is what
  browsers check, not the old CN field.
- **Key Usage / Extended Key Usage (EKU)** — what the key may be used for (e.g. "TLS server auth,"
  "digital signature").
- **Basic Constraints** — notably `CA:TRUE/FALSE`: is this a CA cert allowed to sign other certs?

**CN is dead, SAN is everything.** Historically the hostname lived in the Subject's Common Name (CN).
Modern clients **ignore CN** and require the hostname in the **SAN** extension. A cert without a matching
SAN entry fails, no matter what the CN says.

**Encoding: DER vs PEM.** The same certificate can be stored as binary **DER** or as base64-wrapped
**PEM** (the `-----BEGIN CERTIFICATE-----` text you see everywhere). PEM is what you'll usually handle.

{: .note }
> **A cert is just signed data — trust comes from the signature**
> The certificate file itself isn't secret and contains only public information. Its value is entirely
> that a CA you trust <em>signed</em> it (Lesson 4). Strip away the signature and it's an unverifiable
> claim. That's why the whole next lesson is about <em>who</em> signs and <em>why</em> you'd trust them.

---

## Lab

```bash
# Fetch and decode a real certificate
$ echo | openssl s_client -connect wikipedia.org:443 -servername wikipedia.org 2>/dev/null \
    | openssl x509 -noout -text | sed -n '1,30p'
# Read off: Issuer, Subject, Validity, and under X509v3 extensions the Subject Alternative Name list.

# Just the SANs (what actually authorizes the hostname)
$ echo | openssl s_client -connect wikipedia.org:443 -servername wikipedia.org 2>/dev/null \
    | openssl x509 -noout -ext subjectAltName

# Convert PEM <-> DER
$ openssl x509 -in cert.pem -outform der -out cert.der
$ openssl x509 -inform der -in cert.der -text -noout | head
```

---

## Further Reading

| Topic | Link |
|---|---|
| X.509 | [Wikipedia — X.509](https://en.wikipedia.org/wiki/X.509) |
| Subject Alternative Name | [Wikipedia — Subject Alternative Name](https://en.wikipedia.org/wiki/Subject_Alternative_Name) |
| PKI | [Wikipedia — Public key infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) |
| `x509` | [openssl x509](https://docs.openssl.org/master/man1/openssl-x509/) |

---

## Checkpoint

**Q1. What does a certificate actually bind together, and what makes it trustworthy?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A certificate binds a <strong>public key</strong> to an <strong>identity</strong> (typically one or more
hostnames), along with a validity period and usage constraints. On its own that binding is just a claim —
what makes it trustworthy is that a <strong>certificate authority you trust has signed it</strong> with
its private key (a digital signature, Lesson 4). Because you can verify that signature with the CA's
known public key, and you trust the CA to only sign correct bindings, you can believe "this public key
really belongs to this hostname." The certificate file itself is all public information; its security
value comes entirely from the signature, not from secrecy.
</details>

---

**Q2. Which field does a browser use to match a certificate to a hostname, and which field is obsolete for this?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Browsers match the hostname against the <strong>Subject Alternative Name (SAN)</strong> extension — the
list of DNS names (and/or IPs) the certificate is valid for. The old <strong>Common Name (CN)</strong>
field in the Subject is <strong>obsolete</strong> for hostname matching: modern clients ignore it
entirely and require the name to appear in the SAN. So a certificate whose CN says
<code>www.example.com</code> but has no matching SAN entry will be rejected — the SAN is authoritative.
</details>

---

**Q3. What does the Basic Constraints `CA:TRUE` flag signify, and why does it matter for security?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<code>CA:TRUE</code> in the Basic Constraints extension marks a certificate as a <strong>CA
certificate</strong> — one that is permitted to <em>sign other certificates</em> (i.e. act as an issuer
in a chain). <code>CA:FALSE</code> (or absent) marks a leaf/end-entity certificate that may not sign
others. It matters enormously for security: if clients didn't enforce this flag, anyone holding an
ordinary leaf certificate (e.g. for their own website) could use it to sign a certificate for
<em>any</em> hostname and have it accepted — a total break of the trust model. Enforcing Basic
Constraints ensures only deliberately-designated CA certs can issue, and path length constraints can
further limit how deep a sub-CA may delegate. (This exact check was the basis of a famous early TLS
vulnerability when some clients failed to verify it.)
</details>

---

## Homework

Inspect the certificates of three different websites and tabulate their issuer, validity period (how
many days), signature algorithm, and number of SAN entries. Note any patterns (e.g. short lifetimes,
ECDSA vs RSA, wildcard SANs) and explain what each pattern tells you about modern certificate practice.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Across three sites you'll typically observe patterns such as: <strong>short validity periods</strong>
(often ~90 days, e.g. Let's Encrypt, or capped near a year by industry rules) — reflecting the move to
short-lived, auto-renewed certs that limit the damage window of a compromised key and reduce reliance on
revocation (Lesson 10); a mix of <strong>signature algorithms</strong>, increasingly <strong>ECDSA</strong>
(smaller/faster, Lesson 4) alongside RSA; <strong>multiple SAN entries</strong> and sometimes
<strong>wildcard SANs</strong> (<code>*.example.com</code>) covering many subdomains or apex+www, showing
how one cert serves many names; and well-known <strong>intermediate issuers</strong> rather than roots
(Lesson 7). The takeaways: modern practice favors short lifetimes + automation (ACME), elliptic-curve
keys for efficiency, SAN-based multi-name coverage, and issuance from intermediates — all visible just by
decoding the certificates.
</details>
