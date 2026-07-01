---
title: "Lesson 7 — Certificate Authorities & Chains of Trust"
nav_order: 7
parent: "Phase 2: PKI & Certificates"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 7: Certificate Authorities & Chains of Trust

## Concept

A certificate is only as good as *who signed it*. **Certificate Authorities (CAs)** are the trusted
third parties that sign certificates. Trust is **delegated in a chain**: a small number of **root** CAs
(whose public keys are pre-installed in your OS/browser) sign **intermediate** CAs, which sign your
**leaf** certificate. Verifying a leaf means walking the chain up to a trusted root.

```
   Root CA  (trusted, in your trust store)
      │ signs
   Intermediate CA
      │ signs
   Leaf cert  (www.example.com)   ← verify by walking UP to a root you trust
```

---

## How it works

**Root, intermediate, leaf.** The **root** is a self-signed certificate whose public key ships in every
trust store. Roots are kept **offline** and rarely used directly — instead they sign **intermediates**,
which do the day-to-day issuing. Your **leaf** (server) cert is signed by an intermediate. Each
certificate's *issuer* points to the one above it.

**Trust stores.** Your OS and browsers carry a curated list of trusted **root** CA certificates (e.g.
the Mozilla CA program). A root becomes trusted only by passing audits and being added to these
programs. You trust a root → therefore you trust what it (transitively) signs.

**Path building and validation.** To validate a leaf, the client builds a **chain** from leaf →
intermediate(s) → a trusted root, verifying each signature with the next certificate's public key, and
checking validity dates and constraints (Lesson 9) at every hop. If the chain reaches a trusted root,
it's valid; if not, untrusted.

**Why intermediates exist.** Roots are extremely sensitive — if a root key leaks, *everything* it
anchored is compromised and the root must be removed from billions of devices (catastrophic). So roots
stay **offline**, and revocable, replaceable **intermediates** do the actual signing. If an intermediate
is compromised, it can be revoked without touching the root.

{: .note }
> **Servers must send the intermediates, not the root**
> A common deployment bug: serving only the leaf. Clients have the <em>root</em> but usually not your
> <em>intermediate</em>, so the server must send leaf + intermediate(s) to let the client build the chain
> up to the root it already trusts. It must <em>not</em> send the root (clients ignore it — they trust
> their own copy). "Chain not trusted" errors are very often a missing intermediate.

---

## Lab

```bash
# Show the chain a server presents (leaf + intermediates)
$ echo | openssl s_client -connect wikipedia.org:443 -servername wikipedia.org -showcerts 2>/dev/null \
    | grep -E 's:|i:'
# 's:' = subject, 'i:' = issuer for each cert — read the chain leaf→intermediate.

# Verify a chain explicitly against a CA bundle
$ openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt -untrusted intermediate.pem leaf.pem
leaf.pem: OK

# Inspect a root in your trust store (self-signed: subject == issuer)
$ openssl x509 -in /etc/ssl/certs/ca-certificates.crt -noout -subject -issuer | head
```

---

## Further Reading

| Topic | Link |
|---|---|
| Certificate authority | [Wikipedia — Certificate authority](https://en.wikipedia.org/wiki/Certificate_authority) |
| Chain of trust | [Wikipedia — Chain of trust](https://en.wikipedia.org/wiki/Chain_of_trust) |
| Root certificate | [Wikipedia — Root certificate](https://en.wikipedia.org/wiki/Root_certificate) |
| Mozilla CA program | [wiki.mozilla.org/CA](https://wiki.mozilla.org/CA) |

---

## Checkpoint

**Q1. Why do servers send intermediate certificates but not the root?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
To validate a leaf, a client must build a chain from the leaf up to a root it trusts. Clients already
have the <strong>root</strong> in their trust store, but they usually do <em>not</em> have your specific
<strong>intermediate</strong> — so the server must supply the intermediate(s) to complete the path from
leaf to the trusted root. Sending the <strong>root</strong> is pointless and even discouraged: the client
ignores any root the server sends and uses its own trusted copy (trusting a server-supplied root would be
circular). So: send leaf + intermediates (the parts the client lacks), omit the root (the part the client
already independently trusts). A missing intermediate is the classic cause of "certificate chain not
trusted" errors.
</details>

---

**Q2. Why are root CAs kept offline and used only to sign intermediates?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because a root's private key is the ultimate anchor of trust: it's pre-installed in billions of devices
and underpins every certificate beneath it. If a root key were compromised, an attacker could forge trust
for <em>anything</em>, and remediation means removing/replacing that root in every trust store worldwide
— slow and catastrophic. To minimize that risk, roots are kept <strong>offline</strong> (air-gapped,
rarely touched) and used only to sign a handful of <strong>intermediate</strong> CAs. The intermediates
do the high-volume day-to-day issuing and can be <strong>revoked and replaced</strong> if compromised
without disturbing the root or the trust stores. This isolates the most valuable key from operational
exposure.
</details>

---

**Q3. Describe the steps a client takes to validate a leaf certificate via the chain of trust.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The client performs <strong>path building and validation</strong>: (1) Start with the leaf certificate
and look at its <em>issuer</em>; find the certificate that issued it (an intermediate, supplied by the
server). (2) Verify the leaf's <strong>signature</strong> using that intermediate's public key, and check
the leaf's validity dates and constraints. (3) Repeat up the chain — verify each certificate's signature
with the next one's public key — until reaching a certificate that is, or is signed by, a <strong>root in
the client's trust store</strong>. (4) At every hop also enforce checks like validity period, Basic
Constraints (<code>CA:TRUE</code> for issuers), key usage/EKU, and name matching at the leaf (Lesson 9).
If a complete chain to a trusted root is built and every signature and constraint checks out, the leaf is
trusted; if the chain can't reach a trusted root (or any check fails), it's rejected.
</details>

---

## Homework

Deliberately break a chain: configure a local server (or use `openssl verify`) with only the leaf
certificate and no intermediate, observe the failure, then add the intermediate and watch it succeed.
Explain exactly what the client could and couldn't do in each case, and why this is one of the most
common real-world TLS misconfigurations.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With <strong>only the leaf</strong>, <code>openssl verify</code> (or a browser) fails with an error like
"unable to get local issuer certificate": the client can verify the leaf's signature <em>only if</em> it
can find the issuing intermediate, but it doesn't have it — the trust store contains the root, not your
intermediate — so it cannot build a path from leaf to a trusted root and rejects the cert. After you
<strong>add the intermediate</strong> (server sends leaf + intermediate, or you pass <code>-untrusted
intermediate.pem</code>), the client can chain leaf → intermediate → trusted root, verify every signature,
and validation succeeds. This is one of the most common real-world TLS misconfigurations because it
<strong>often appears to work in the admin's own browser</strong> (which may have cached the intermediate
from another site) while failing for other clients, fresh systems, and many tools — so it slips through
casual testing. The fix is always to install the full chain (leaf + all intermediates) on the server.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 8 — CSRs & the openssl Toolkit →](lesson-08-csr-openssl){: .btn .btn-primary }
