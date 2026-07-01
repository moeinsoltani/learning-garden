---
title: "Lesson 10 — Revocation: CRL, OCSP, Stapling"
nav_order: 10
parent: "Phase 2: PKI & Certificates"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 10: Revocation — CRL, OCSP, Stapling

## Concept

Certificates have expiry dates, but sometimes you must **untrust one early** — the private key leaked, or
it was mis-issued. **Revocation** is how a CA says "this certificate is no longer valid even though it
hasn't expired." It turns out to be one of the hardest problems in PKI, and the industry's answer has
shifted over time.

```
   Certificate compromised before notAfter → CA must announce "revoked"
   CRL:   download a big list of revoked serials       (bulky, stale)
   OCSP:  ask "is serial X revoked?" per cert          (privacy + latency cost)
   Stapling: server fetches its own OCSP proof, attaches it (fixes OCSP's problems)
```

---

## How it works

**Why revocation is needed.** If a server's private key is stolen, the attacker can impersonate it until
the cert expires — possibly a year. Revocation lets the CA invalidate it immediately. But the client must
*find out*, and that's the hard part.

**CRL (Certificate Revocation List).** The CA publishes a signed list of revoked serial numbers. Clients
download it and check. Problem: CRLs grow **large**, are fetched infrequently so they're **stale**, and
downloading megabytes per check doesn't scale.

**OCSP (Online Certificate Status Protocol).** Instead of a whole list, the client asks the CA's OCSP
responder "is *this* serial revoked?" in real time. Problems: it adds **latency** to every connection, it
**leaks** which sites you visit to the CA (privacy), and if the responder is down clients often
**fail open** (ignore the check) — undermining the whole point.

**OCSP stapling** fixes much of this: the **server** periodically fetches a signed, time-stamped OCSP
response for its own cert and **staples** it to the TLS handshake. The client gets fresh revocation proof
with no extra request, no privacy leak, and no dependency on the responder being up at connect time.
**Must-staple** is a cert flag requiring a stapled response, closing the fail-open gap.

**The modern direction: short-lived certs.** Because revocation is so unreliable, the industry is moving
to **short-lived certificates** (e.g. 90 days, soon shorter) issued via automation (Lesson 13). If a cert
only lives days, the window of a compromised-but-unrevoked cert is small, reducing reliance on revocation
infrastructure altogether.

{: .note }
> **Why revocation "doesn't really work"**
> CRLs are too big/stale; OCSP is slow, privacy-leaking, and usually fails open; so a determined attacker
> with a stolen key often stays usable. That uncomfortable truth is exactly why the ecosystem pivoted to
> <em>short lifetimes + automation</em>: don't fix revocation, make certificates expire fast.

---

## Lab

```bash
# See where revocation info lives in a real cert
$ echo | openssl s_client -connect wikipedia.org:443 -servername wikipedia.org 2>/dev/null \
    | openssl x509 -noout -ext crlDistributionPoints,authorityInfoAccess
# crlDistributionPoints = CRL URL; authorityInfoAccess = OCSP responder URL

# Make a live OCSP query for a cert
$ openssl s_client -connect wikipedia.org:443 -servername wikipedia.org -status </dev/null 2>/dev/null \
    | grep -A2 'OCSP response'
# With stapling, the server returns an OCSP "good" status inline in the handshake.
```

---

## Further Reading

| Topic | Link |
|---|---|
| Certificate revocation list | [Wikipedia — Certificate revocation list](https://en.wikipedia.org/wiki/Certificate_revocation_list) |
| OCSP | [Wikipedia — Online Certificate Status Protocol](https://en.wikipedia.org/wiki/Online_Certificate_Status_Protocol) |
| OCSP stapling | [Wikipedia — OCSP stapling](https://en.wikipedia.org/wiki/OCSP_stapling) |
| Short-lived certs | [Wikipedia — Certificate authority](https://en.wikipedia.org/wiki/Certificate_authority) |

---

## Checkpoint

**Q1. Why is revocation necessary, and why is it fundamentally hard?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It's <strong>necessary</strong> because a certificate can become untrustworthy before its expiry — most
importantly if its private key is compromised (or it was mis-issued) — and the CA needs a way to declare
it invalid <em>now</em> rather than waiting up to a year for it to expire, during which an attacker could
impersonate the service. It's <strong>hard</strong> because the CA can revoke, but every <em>client</em>
must reliably <em>learn</em> about it at connection time, and the mechanisms for that all have problems:
CRLs are large and stale, OCSP adds latency and leaks browsing to the CA and usually <strong>fails
open</strong> when the responder is unreachable (so a blocked check just gets ignored). The result is that
revocation is often unreliable in practice — a stolen key may keep working — which is the core difficulty.
</details>

---

**Q2. What problems does OCSP stapling solve compared to plain OCSP?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Plain OCSP makes the <em>client</em> contact the CA's responder on each connection, causing (1)
<strong>latency</strong> (an extra round trip before the page loads), (2) a <strong>privacy leak</strong>
(the CA learns which sites each client visits), and (3) a <strong>reliability/fail-open problem</strong>
(if the responder is down, clients typically skip the check). <strong>OCSP stapling</strong> moves the
work to the server: the server periodically fetches a fresh, CA-signed OCSP response for its own
certificate and <strong>attaches ("staples") it to the TLS handshake</strong>. So the client gets current
revocation proof with <em>no extra request</em> (no latency), <em>no contact with the CA</em> (no privacy
leak), and <em>no dependence on the responder being reachable at connect time</em>. Adding the
<strong>must-staple</strong> flag forces a stapled response to be present, closing the fail-open gap.
</details>

---

**Q3. Why is the industry shifting to short-lived certificates, and how does that relate to revocation?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because revocation has proven unreliable (CRLs stale, OCSP slow/privacy-leaking/fail-open), the practical
window during which a compromised-but-unrevoked certificate can be abused is often large. <strong>Short-
lived certificates</strong> attack the problem from the other side: if a certificate is only valid for,
say, 90 days (and the trend is even shorter), then even with <em>no working revocation</em>, a stolen key
becomes useless within that short window when the cert simply expires. This trades a hard, unreliable
problem (getting every client to learn a cert is revoked) for an easy, reliable one (certs expiring on a
schedule) — at the cost of needing <strong>automation</strong> (ACME, Lesson 13) to reissue constantly.
So short lifetimes reduce dependence on revocation infrastructure by shrinking the danger window
automatically.
</details>

---

## Homework

Find a certificate that supports OCSP stapling and one that doesn't (test several sites with
`openssl s_client -status`). Compare what the client must do in each case to learn revocation status, and
the privacy implication. Then argue whether you'd rather rely on stapling or on short-lived certs for a
high-traffic service, and why.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With a <strong>stapling</strong> site, <code>-status</code> shows an inline "OCSP Response ... good"
delivered in the handshake — the client learns revocation status with no extra connection and the CA never
sees the client's request. With a <strong>non-stapling</strong> site, no OCSP response is returned inline,
so a client that wants to check must contact the CA's OCSP responder itself (extra latency) or fetch a
CRL — and contacting the responder <strong>reveals to the CA which site the user is visiting</strong> (a
privacy leak), and if it's unreachable the client usually fails open and checks nothing. For a high-traffic
service, the strongest answer is to <strong>prefer short-lived certificates</strong> (with stapling as a
complement): stapling improves the OCSP experience but still depends on revocation machinery and correct
configuration, whereas short lifetimes shrink the compromise window automatically and reliably even if
revocation checks are skipped entirely. Stapling optimizes a fragile mechanism; short lifetimes reduce the
need for it — so for resilience at scale, automation + short lifetimes is the more robust strategy, ideally
with both.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 11 — Certificate Transparency →](lesson-11-ct){: .btn .btn-primary }
