---
title: "Lesson 20 — Inspecting & Hardening TLS"
nav_order: 20
parent: "Phase 3: TLS & SSL"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 20: Inspecting & Hardening TLS

## Concept

Knowing TLS theory is one thing; **auditing and tightening a real deployment** is the skill that prevents
breaches. This lesson is the practical checklist: how to inspect what a server actually offers, and how to
configure it so only strong, modern options remain.

```
   Audit:    what versions / ciphers / cert / extensions does the server REALLY offer?
   Harden:   disable old versions, keep only ECDHE+AEAD, add HSTS, fix cert/key hygiene
   Verify:   re-scan → clean grade
```

---

## How it works

**Inspect with the right tools.** `openssl s_client` shows one connection's negotiated parameters;
**`testssl.sh`** and the **SSL Labs** scanner systematically probe *all* offered versions and suites and
grade the result. Auditing means enumerating the whole surface, not just one successful connection.

**The hardening checklist:**
- **Versions:** offer only **TLS 1.2 + 1.3**; disable SSL 3, TLS 1.0/1.1 (Lesson 14).
- **Cipher suites:** keep only **ECDHE + AEAD** (forward secrecy, Lesson 17); drop RC4, 3DES, CBC,
  static-RSA.
- **HSTS** (HTTP Strict Transport Security): a response header telling browsers "only ever use HTTPS for
  this site," preventing downgrade-to-HTTP and SSL-stripping attacks.
- **Certificate/key hygiene:** strong key (≥2048-bit RSA or EC), full chain installed (Lesson 7), correct
  SANs (Lesson 6), reasonable lifetime + automated renewal (Lesson 13), private key permissions locked
  down.
- **OCSP stapling** enabled (Lesson 10) for efficient, private revocation.

**Defense in depth.** No single setting is "secure"; hardening is the *combination* — modern versions
+ forward-secret AEAD + HSTS + clean certs + stapling. A scanner's letter grade is a quick proxy for how
many of these are right.

{: .note }
> **"Use safe defaults, then verify"**
> Modern web servers (and TLS 1.3) default to sane settings, but defaults drift and old configs linger.
> The discipline is: configure conservatively (only 1.2/1.3, only ECDHE+AEAD), then <em>scan to confirm</em>
> — never assume. The scan is the proof, the same way a packet capture proves a network claim.

---

## Lab

```bash
# One-connection view
$ echo | openssl s_client -connect wikipedia.org:443 2>/dev/null | grep -E 'Protocol|Cipher|Verify'

# Full audit with testssl.sh (clones from github) — enumerates versions, suites, vulns, grade
$ ./testssl.sh --quiet --color 0 wikipedia.org | grep -E 'TLS 1|cipher|HSTS|Rating'

# Check HSTS header presence
$ curl -sI https://wikipedia.org | grep -i strict-transport-security
strict-transport-security: max-age=...; includeSubDomains; preload

# Confirm weak protocols are refused
$ for v in tls1 tls1_1; do echo -n "$v: "; echo | openssl s_client -connect wikipedia.org:443 -$v 2>/dev/null \
    | grep -q Protocol && echo "ENABLED (bad)" || echo "disabled (good)"; done
```

---

## Further Reading

| Topic | Link |
|---|---|
| HSTS | [Wikipedia — HTTP Strict Transport Security](https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security) |
| testssl.sh | [testssl.sh](https://testssl.sh/) |
| Mozilla TLS config generator | [ssl-config.mozilla.org](https://ssl-config.mozilla.org/) |
| `openssl s_client` | [openssl s_client](https://docs.openssl.org/master/man1/openssl-s_client/) |

---

## Checkpoint

**Q1. Name three things you'd check or set to call a TLS deployment "hardened."**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Any three of: (1) <strong>Only modern protocol versions</strong> — TLS 1.2 and 1.3 enabled, SSL 3/TLS 1.0/
1.1 disabled (to prevent downgrade and legacy-version attacks). (2) <strong>Only strong cipher suites</strong>
— ECDHE key exchange with AEAD ciphers (forward secrecy), with RC4/3DES/CBC/static-RSA removed. (3)
<strong>HSTS</strong> enabled so browsers refuse to use plain HTTP / can't be SSL-stripped. (4) <strong>Clean
certificate/key hygiene</strong> — strong key, full chain installed, correct SANs, automated renewal, and
locked-down private-key file permissions. (5) <strong>OCSP stapling</strong> for efficient revocation. The
point is it's the <em>combination</em> (defense in depth), confirmed by a scanner, not any single switch.
</details>

---

**Q2. What does HSTS protect against, and how?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
HSTS (HTTP Strict Transport Security) protects against <strong>downgrade-to-HTTP / SSL-stripping
attacks</strong>. Without it, a user's first request to a site might go over plain HTTP (e.g. typing the
domain without https://), and an active attacker on the path can intercept that and keep the user on
unencrypted HTTP — stripping away the redirect to HTTPS and reading/altering traffic. With HSTS, the server
sends a response header (<code>Strict-Transport-Security: max-age=...</code>) that tells the browser
"<strong>for this domain, only ever connect over HTTPS</strong>" for the specified duration. The browser
then refuses to make plaintext HTTP requests to that site at all (it upgrades them internally), so there's
no HTTP request for an attacker to hijack. The <code>preload</code> option goes further by baking the rule
into browsers in advance, protecting even the very first visit.
</details>

---

**Q3. Why is "scan to confirm" part of the hardening discipline rather than just configuring good settings?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because configuration and reality often diverge: defaults change between software versions, multiple config
layers (load balancer, web server, framework) can each override settings, old virtual hosts or fallback
listeners may still offer weak versions/ciphers, and a typo can silently leave something enabled. Just
setting "good" options doesn't prove the server actually <em>presents</em> only those to clients. A
<strong>scan</strong> (testssl.sh / SSL Labs) empirically enumerates what's truly offered across all
versions and suites, catches accidentally-enabled legacy options, missing HSTS, incomplete chains, and weak
parameters, and gives an objective grade. It's the same principle as proving a network claim with a packet
capture rather than trusting the config — verify the observable behavior, don't assume the intent took
effect.
</details>

---

## Homework

Run `testssl.sh` (or SSL Labs) against your own service (or a test server you configure) and produce a
before/after: capture the initial findings/grade, apply the hardening checklist (disable old versions,
restrict to ECDHE+AEAD, add HSTS), and re-scan. Document each change and the specific finding it resolved.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A complete answer shows a measurable improvement. <strong>Before:</strong> the scan flags issues such as
TLS 1.0/1.1 enabled, weak/legacy cipher suites offered (e.g. 3DES, CBC, or non-FS static-RSA), missing
HSTS, and possibly an incomplete chain — yielding a mediocre grade. <strong>Changes applied, each tied to a
finding:</strong> (1) disable SSL3/TLS 1.0/1.1, leaving only 1.2+1.3 → resolves the "obsolete protocol /
downgrade risk" finding; (2) restrict cipher suites to ECDHE+AEAD and remove RC4/3DES/CBC/static-RSA →
resolves "weak cipher / no forward secrecy"; (3) add the <code>Strict-Transport-Security</code> header →
resolves "HSTS not set / SSL-stripping risk"; (4) install the full intermediate chain and ensure correct
SANs → resolves "chain issues / name mismatch"; (5) enable OCSP stapling → resolves "revocation checking."
<strong>After:</strong> the re-scan shows only TLS 1.2/1.3 with forward-secret AEAD suites, HSTS present,
clean chain, and a top grade. The exercise demonstrates the discipline: configure conservatively, then
<em>verify with the scanner</em> that each weakness is actually gone.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 21 — TLS Attacks & the Lessons They Taught →](lesson-21-tls-attacks){: .btn .btn-primary }
