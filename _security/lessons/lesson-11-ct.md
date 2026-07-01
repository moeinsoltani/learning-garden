---
title: "Lesson 11 — Certificate Transparency"
nav_order: 11
parent: "Phase 2: PKI & Certificates"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 11: Certificate Transparency

## Concept

The CA system has a scary failure mode: any trusted CA can issue a certificate for **any** domain. If a
CA is compromised or coerced, it could mint a valid cert for `yourbank.com` without the owner ever
knowing. **Certificate Transparency (CT)** fixes the "without knowing" part: every issued certificate
must be recorded in **public, append-only logs**, so domain owners (and the world) can *watch* for
certificates issued for their names.

```
   CA issues cert ──► must be logged in public CT logs ──► anyone can monitor the logs
                                                           ↳ domain owner spots a rogue cert
```

CT doesn't prevent mis-issuance; it makes it **detectable**.

---

## How it works

**The mis-issuance problem CT solves.** Before CT, a mis-issued certificate could be used in targeted
attacks invisibly — only the victims (if lucky) might notice. There was no global record of what had been
issued. The 2011 DigiNotar compromise (rogue Google certs used against Iranian users) drove CT's creation.

**Append-only Merkle-tree logs.** A CT log is a public, cryptographically **append-only** structure based
on a [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree): entries can be added but never altered or
removed without detection, because the tree's root hash would change. This means a log can't quietly erase
a certificate it once recorded — the history is tamper-evident.

**SCTs (Signed Certificate Timestamps).** When a cert is submitted to a log, the log returns an **SCT** —
a signed promise "I've recorded this." Browsers require certificates to come with SCTs (embedded in the
cert, or delivered via the TLS handshake/OCSP) from a sufficient number of trusted logs, otherwise they
reject the cert. So **enforcement** is at the browser: no SCTs, no trust.

**Monitoring.** Because logs are public, anyone can run (or use) a **CT monitor** that watches for
certificates issued for their domains and alerts on unexpected ones. Tools/services like `crt.sh` let you
search all logged certs for a domain.

{: .note }
> **Detect, don't prevent — and why that's powerful**
> CT can't stop a CA from issuing a bad cert, but by guaranteeing it'll be <em>publicly visible</em>, it
> creates accountability: mis-issuance gets caught quickly, the offending CA faces consequences (even
> removal from trust stores), and that deterrent makes CAs far more careful. Transparency turns an
> invisible risk into a monitored one.

---

## Lab

```bash
# Search public CT logs for every cert ever issued for a domain (crt.sh)
$ curl -s 'https://crt.sh/?q=example.com&output=json' | python3 -m json.tool | head -40
# Each entry = a logged certificate: issuer, validity, SANs. You can spot unexpected issuers.

# See SCTs embedded in a live certificate
$ echo | openssl s_client -connect wikipedia.org:443 -servername wikipedia.org 2>/dev/null \
    | openssl x509 -noout -text | grep -A4 'CT Precertificate SCTs'
# Lists the logs and timestamps proving the cert was publicly recorded.
```

---

## Further Reading

| Topic | Link |
|---|---|
| Certificate Transparency | [Wikipedia — Certificate Transparency](https://en.wikipedia.org/wiki/Certificate_Transparency) |
| Merkle tree | [Wikipedia — Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree) |
| crt.sh (CT search) | [crt.sh](https://crt.sh/) |
| DigiNotar incident | [Wikipedia — DigiNotar](https://en.wikipedia.org/wiki/DigiNotar) |

---

## Checkpoint

**Q1. What problem does Certificate Transparency solve, and what does it explicitly NOT do?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
CT solves the <strong>invisibility of certificate mis-issuance</strong>. Any trusted CA can technically
issue a certificate for any domain, and before CT a compromised or coerced CA could mint a valid rogue
certificate for, say, your bank, and use it in targeted attacks <em>without anyone knowing</em>. CT
requires every issued certificate to be recorded in <strong>public, append-only logs</strong>, so domain
owners and the public can detect certificates issued for their names. What it explicitly does <strong>not
do</strong> is <em>prevent</em> mis-issuance — a CA can still issue a bad certificate; CT only guarantees
the issuance becomes <strong>publicly visible and therefore detectable</strong>, enabling rapid response
and accountability rather than silent abuse.
</details>

---

**Q2. How does Certificate Transparency let you detect a certificate mis-issued for your domain?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because all certificates are recorded in <strong>public CT logs</strong>, you (or a monitoring service)
can continuously <strong>watch those logs for any certificate naming your domain</strong>. When a cert is
logged for <code>yourbank.com</code> that you didn't request — say from an unexpected CA — a CT
<strong>monitor</strong> flags it, and you can investigate and get it revoked / report the CA. The logs
are append-only Merkle structures, so a CA can't issue a cert and then quietly remove the record to hide
it. Practically, you can search a service like <code>crt.sh</code> for your domain or subscribe to alerts,
turning "did anyone issue a cert for us?" from unanswerable into a routine, near-real-time check.
</details>

---

**Q3. What is an SCT, and how do browsers use it to enforce CT?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An <strong>SCT (Signed Certificate Timestamp)</strong> is a signed receipt issued by a CT log when a
certificate (or precertificate) is submitted, amounting to a cryptographic promise "this certificate has
been recorded in my public log at this time." Browsers <strong>enforce CT</strong> by requiring that a
certificate be accompanied by a sufficient number of valid SCTs from <strong>trusted logs</strong>
(delivered embedded in the certificate, via a TLS extension, or via OCSP). If a certificate arrives
<strong>without adequate SCTs</strong>, the browser rejects it as untrusted — even if it's otherwise valid
and chains to a trusted CA. This makes public logging effectively mandatory: a CA that issues a cert
without logging it produces something browsers won't accept, so mis-issuance can't avoid the public record.
</details>

---

## Homework

Use `crt.sh` to look up all certificates issued for a domain you control (or a well-known one). Identify
how many distinct CAs have issued for it, the typical validity periods, and any wildcard or unexpected
entries. Then describe how you would set up ongoing monitoring and what you'd do if an unexpected
certificate appeared.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
On <code>crt.sh</code> you'll see the full logged history for the domain: the set of issuing CAs (often
one or two expected ones — e.g. your ACME provider plus maybe a CDN's CA), the validity periods (commonly
short, ~90 days, reflecting automated issuance), and any wildcard SANs or subdomains. Anything from a CA
you don't use, or for a hostname you didn't provision, is a red flag. To set up <strong>ongoing
monitoring</strong>, subscribe to a CT-monitoring service (or crt.sh's RSS/alerts, or run a monitor) that
notifies you whenever a new certificate naming your domain is logged. If an <strong>unexpected
certificate</strong> appeared, you'd: (1) confirm it's truly unauthorized (not a forgotten internal/CDN
issuance), (2) <strong>report it to the issuing CA and request revocation</strong>, (3) investigate how it
was obtained (was domain control validation bypassed? account compromised? DNS hijacked?), (4) tighten
controls — e.g. add a <strong>CAA DNS record</strong> restricting which CAs may issue for your domain, and
secure your registrar/DNS and ACME accounts. CT gives you the early-warning signal; CAA and account
hardening reduce the chance of recurrence.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 12 — Running a Private CA →](lesson-12-private-ca){: .btn .btn-primary }
