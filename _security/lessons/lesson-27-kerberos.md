---
title: "Lesson 27 — Kerberos"
nav_order: 27
parent: "Phase 4: Authentication Fundamentals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 27: Kerberos

## Concept

[Kerberos](https://en.wikipedia.org/wiki/Kerberos_(protocol)) is the classic **enterprise single sign-on**
protocol — the engine behind Windows Active Directory logins. You authenticate **once** to a central
authority and receive a **ticket** that lets you access many services **without re-entering (or transmitting)
your password**. It's ticket-based, symmetric-crypto SSO predating the web's OAuth/OIDC.

```
   1. authenticate to KDC once  → get a Ticket-Granting Ticket (TGT)
   2. present TGT to get a service ticket for each service you want
   3. present the service ticket to the service → access, no password sent
```

---

## How it works

**The three parties.** The **client**, the **service** (e.g. a file server), and the trusted **KDC** (Key
Distribution Center, with an Authentication Server + Ticket-Granting Server). The KDC shares a long-term
secret key with every principal (derived from the user's password and each service's key).

**The ticket flow.**
1. **Get a TGT:** the client authenticates to the KDC; the KDC returns a **Ticket-Granting Ticket**
   (encrypted so only the KDC can read it later) plus a session key. Crucially, the password is used to
   derive a key *locally* — it's **not sent over the network**.
2. **Get a service ticket:** to reach a service, the client presents its TGT to the KDC, which issues a
   **service ticket** encrypted with that *service's* key.
3. **Access the service:** the client presents the service ticket; the service decrypts it with its own key,
   trusts the KDC's vouching, and grants access — all **without contacting the KDC** and **without the
   password**.

**Why tickets avoid sending passwords around.** The password never traverses the network; instead, time-
limited, encrypted **tickets** prove identity. A service trusts a ticket because only the KDC could have
produced one encrypted with the service's key. SSO falls out naturally: one TGT → many service tickets.

**Constraints.** Kerberos needs a **central KDC** (a trusted bottleneck) and **synchronized clocks** (tickets
are time-stamped to prevent replay — clock skew beyond a few minutes breaks it, tying back to the networking
time-sync lesson). It's designed for **enterprise/intranet** environments (Active Directory), not the open
web — where OAuth/OIDC (Phase 5) took over.

{: .note }
> **Kerberos vs OAuth/OIDC**
> Both give single sign-on, but Kerberos is <strong>symmetric-key, intranet, ticket-based</strong> (great
> inside a controlled domain with a KDC and synced clocks), while OAuth/OIDC (Phase 5) are
> <strong>token-based over HTTP for the open web / cross-organization</strong> using TLS and signed tokens.
> Same goal — authenticate once, access many — different era and trust model.

---

## Lab

```bash
# On a Kerberos-enabled host (e.g. joined to a realm / AD):
# 1. Authenticate and obtain a TGT (password used locally, not sent)
$ kinit alice@EXAMPLE.COM
Password for alice@EXAMPLE.COM:        # verified by deriving a key; not transmitted

# 2. List your tickets — note the TGT (krbtgt) and any service tickets
$ klist
Ticket cache: ...
Default principal: alice@EXAMPLE.COM
Valid starting     Expires            Service principal
01/01/26 09:00:00  01/01/26 19:00:00  krbtgt/EXAMPLE.COM@EXAMPLE.COM   ← the TGT

# 3. Access a Kerberized service (e.g. SSH with GSSAPI) → a service ticket appears in klist, no password
$ ssh -o GSSAPIAuthentication=yes fileserver.example.com
$ klist        # now also shows host/fileserver.example.com@EXAMPLE.COM (service ticket)

# 4. Destroy tickets
$ kdestroy
```

---

## Further Reading

| Topic | Link |
|---|---|
| Kerberos | [Wikipedia — Kerberos (protocol)](https://en.wikipedia.org/wiki/Kerberos_(protocol)) |
| Key Distribution Center | [Wikipedia — Key distribution center](https://en.wikipedia.org/wiki/Key_distribution_center) |
| Single sign-on | [Wikipedia — Single sign-on](https://en.wikipedia.org/wiki/Single_sign-on) |
| Active Directory | [Wikipedia — Active Directory](https://en.wikipedia.org/wiki/Active_Directory) |

---

## Checkpoint

**Q1. How does Kerberos let you access many services after authenticating once?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You authenticate to the central <strong>KDC</strong> a single time and receive a <strong>Ticket-Granting
Ticket (TGT)</strong>. From then on, whenever you want to use a service, you present the TGT to the KDC and
get back a <strong>service ticket</strong> for that specific service (encrypted with the service's key); you
hand that ticket to the service, which decrypts and trusts it, granting access. So one initial authentication
(one TGT) can be exchanged for <strong>many service tickets</strong> without re-entering your password — that's
the single sign-on. Each service trusts the ticket because only the KDC could have produced one encrypted with
that service's key.
</details>

---

**Q2. Why is it significant that the password isn't sent over the network in Kerberos?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because it removes the password as something an attacker on the network can capture or replay. In Kerberos
the password is used <strong>locally</strong> to derive a key (used to decrypt the KDC's response); it never
traverses the wire. Authentication instead relies on time-limited, encrypted <strong>tickets</strong>. So even
an eavesdropper on the network — or a compromised service — never sees the user's password; at most they might
see tickets, which are encrypted, time-stamped (replay-protected), and scoped to specific services. This is a
major improvement over schemes that transmit a password (even hashed) to each service, and it means a single
service compromise doesn't yield the user's reusable credential.
</details>

---

**Q3. What two infrastructure requirements does Kerberos impose, and why?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) A <strong>central, trusted KDC</strong> (Key Distribution Center) — every principal shares a long-term
key with it, and it issues all tickets, so it's a required trusted authority (and a potential bottleneck/
single point of failure that must be highly available and protected). (2) <strong>Synchronized clocks</strong>
across participants — tickets are <strong>time-stamped to prevent replay</strong>, and the protocol enforces a
small allowable clock skew (commonly ~5 minutes); if a client's or service's clock drifts beyond that,
authentication fails. (This is exactly why time synchronization, NTP/PTP, is a security dependency — see the
networking time-sync lesson.) These requirements make Kerberos well-suited to controlled enterprise/intranet
environments (Active Directory) with a managed KDC and time sync, rather than the open, decentralized web.
</details>

---

## Homework

On a Kerberos-enabled system, run `kinit` then `klist` and observe your TGT, access a Kerberized service and
watch a service ticket appear, then `kdestroy`. (No realm? Describe the flow from the lab.) Then explain how
Kerberos's ticket model compares to a stateless JWT (Lesson 31): what's similar about "present a token to get
access," and what's fundamentally different about the trust/crypto model.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The flow: <code>kinit</code> authenticates you and <code>klist</code> shows a <strong>TGT</strong>
(<code>krbtgt/...</code>); accessing a Kerberized service adds a <strong>service ticket</strong> to the cache
without a password prompt; <code>kdestroy</code> clears them. <strong>Similar to a JWT:</strong> both are
"present a token to gain access" models — after an initial authentication you carry a credential (ticket / JWT)
that a relying party validates instead of re-checking your password, and both are time-limited. <strong>Fundamentally
different:</strong> (1) <strong>Crypto model</strong> — Kerberos tickets use <em>symmetric</em> keys (each
ticket is encrypted with the target service's secret key, shared with the KDC), whereas a JWT is typically
<em>signed</em> with the issuer's key and verified with its public key (asymmetric), so any holder of the
public key can validate it without a shared secret. (2) <strong>Trust topology</strong> — Kerberos requires a
<em>central online KDC</em> and shared keys within a realm (intranet, synced clocks), while JWTs are designed
for <em>decentralized, cross-domain</em> use over HTTP/TLS where services validate tokens independently using
the issuer's published keys (JWKS). (3) <strong>Scoping/issuance</strong> — Kerberos issues a distinct,
service-encrypted ticket per service from the KDC on demand, whereas a JWT is often one bearer token presented
to many services. So both achieve SSO-style "token in, access out," but Kerberos is symmetric, KDC-centric,
and intranet-oriented, while JWT/OIDC is signature-based, decentralized, and web-oriented (Phase 5).
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 28 — OAuth 2.0 Fundamentals →](lesson-28-oauth2){: .btn .btn-primary }
