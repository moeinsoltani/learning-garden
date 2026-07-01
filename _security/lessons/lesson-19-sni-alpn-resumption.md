---
title: "Lesson 19 — SNI, ALPN, Resumption & 0-RTT"
nav_order: 19
parent: "Phase 3: TLS & SSL"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 19: SNI, ALPN, Resumption & 0-RTT

## Concept

TLS extensions in the ClientHello make the modern web work: **SNI** lets many sites share one IP, **ALPN**
negotiates which application protocol to speak, and **session resumption / 0-RTT** make repeat connections
faster. These are the practical features you'll configure and debug constantly.

```
   ClientHello extensions:
     SNI:  "I want  www.example.com"   → server picks the right certificate
     ALPN: "I speak  h2, http/1.1"     → server picks the protocol (e.g. HTTP/2)
     (later) resumption ticket          → skip the full handshake next time
```

---

## How it works

**SNI (Server Name Indication).** One IP can host thousands of TLS sites, but the server must choose
*which certificate* to present **before** it knows which site you want — so the client states the hostname
in the **SNI** field of the ClientHello. The catch: SNI is sent **in plaintext** (it's needed before
encryption), so it leaks which site you're visiting — exploited for filtering/censorship. **ECH (Encrypted
Client Hello)** encrypts it to close that leak.

**ALPN (Application-Layer Protocol Negotiation).** The client lists the app protocols it supports
(`h2`, `http/1.1`, etc.) and the server picks one — all within the TLS handshake, so there's no extra
round trip to decide whether to speak HTTP/2 vs HTTP/1.1. (HTTP/3 over QUIC is also selected via ALPN.)

**Session resumption.** A full handshake is expensive. After the first connection, the server can issue a
**session ticket** (an encrypted blob the client stores). On reconnect, the client presents the ticket and
both **resume** the session with the previously negotiated secrets — skipping the certificate exchange and
much of the key agreement. Faster reconnections, less CPU.

**0-RTT (and its danger).** TLS 1.3 adds **0-RTT**: on resumption, the client can send *application data
in its very first message*, before the handshake completes — zero round trips of latency. The risk:
**replay**. Because 0-RTT data isn't tied to a fresh handshake, an attacker can capture and **re-send** it,
and the server might process it twice. So 0-RTT must be limited to **idempotent** requests (safe to repeat,
like a GET), never state-changing ones (a payment), and servers apply anti-replay measures.

{: .note }
> **The recurring theme: convenience vs leakage/replay**
> Each feature trades something. SNI enables shared hosting but leaks the hostname (→ ECH). Resumption
> saves CPU but ties sessions across connections. 0-RTT removes latency but introduces replay risk.
> Understanding the trade-off is what lets you enable them <em>safely</em>.

---

## Lab

```bash
# SNI: the same IP returns DIFFERENT certs depending on the name you send
$ echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
    | openssl x509 -noout -subject
$ echo | openssl s_client -connect example.com:443 -servername www.iana.org 2>/dev/null \
    | openssl x509 -noout -subject     # different servername → potentially different cert

# ALPN: ask for HTTP/2 and see what the server selects
$ echo | openssl s_client -connect cloudflare.com:443 -alpn h2,http/1.1 2>/dev/null | grep ALPN
ALPN protocol: h2

# Resumption: openssl reports whether a session was reused
$ openssl s_client -connect wikipedia.org:443 -reconnect 2>/dev/null | grep -E 'Reused|New'
```

---

## Further Reading

| Topic | Link |
|---|---|
| Server Name Indication / ECH | [Wikipedia — Server Name Indication](https://en.wikipedia.org/wiki/Server_Name_Indication) |
| ALPN | [Wikipedia — Application-Layer Protocol Negotiation](https://en.wikipedia.org/wiki/Application-Layer_Protocol_Negotiation) |
| TLS session resumption | [Wikipedia — Transport Layer Security](https://en.wikipedia.org/wiki/Transport_Layer_Security#Resumed_TLS_handshake) |
| 0-RTT replay | [Wikipedia — TLS 1.3](https://en.wikipedia.org/wiki/Transport_Layer_Security#TLS_1.3) |

---

## Checkpoint

**Q1. What problem does SNI solve, why is it sent in plaintext, and what does ECH change?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
SNI solves <strong>virtual hosting over TLS</strong>: many websites can share one IP, but the server must
choose which certificate to present, and it needs to know the requested hostname to do so. The client
therefore puts the hostname in the <strong>SNI</strong> field of the ClientHello — which is sent
<strong>in plaintext because it's needed before the encrypted channel exists</strong> (the server can't
decrypt anything until it picks the right cert and keys). The downside is a privacy/censorship leak: anyone
on-path can read which site you're visiting. <strong>ECH (Encrypted Client Hello)</strong> changes this by
encrypting the ClientHello (including SNI) to a public key the server publishes via DNS, so on-path
observers can no longer see the hostname — closing TLS's last major plaintext identity leak.
</details>

---

**Q2. What is session resumption, and what does it save?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Session resumption lets a client reconnect to a server <strong>without redoing the full handshake</strong>.
After the first connection, the server issues a <strong>session ticket</strong> (an encrypted blob holding
the session's secrets) that the client stores; on a later connection the client presents the ticket and both
sides <strong>resume</strong> using the already-established secrets. This <strong>saves</strong> the
expensive parts of a fresh handshake — the certificate exchange and the asymmetric key-agreement work —
reducing both latency (fewer/cheaper steps) and server CPU. It's especially valuable at scale, where a busy
server would otherwise spend significant CPU on full handshakes for returning clients.
</details>

---

**Q3. Why is 0-RTT data subject to replay, and how is that mitigated?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
0-RTT lets a resuming client send application data in its <em>first</em> message, before the handshake
completes, using keys derived from the previous session's resumption secret. Because that early data isn't
bound to a fresh, live handshake exchange, an attacker who captures the 0-RTT message can simply
<strong>re-send (replay) it</strong>, and the server may process the same request twice — dangerous for
state-changing operations (e.g. a transfer executed twice). Mitigations: restrict 0-RTT to
<strong>idempotent requests</strong> (safe to repeat, like a GET) and never use it for state-changing ones;
and have servers apply <strong>anti-replay defenses</strong> (single-use ticket tracking, tight freshness/
time windows, or rejecting 0-RTT for sensitive endpoints). So 0-RTT's latency win is only taken where replay
is harmless or specifically guarded against.
</details>

---

## Homework

Capture a fresh TLS connection and a resumed one to the same server (use `-reconnect`) and compare what
appears on the wire (full handshake with certificate vs abbreviated resumption). Then describe a concrete
application request where enabling 0-RTT would be unsafe and one where it's fine, justifying each with the
replay risk.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The <strong>fresh</strong> connection shows a full handshake — ClientHello/ServerHello plus the
Certificate exchange and key agreement — while the <strong>resumed</strong> connection is abbreviated:
openssl reports "Reused," and there's no certificate re-sent because both sides restore the prior session's
secrets from the ticket, so it's faster and lighter. For <strong>0-RTT safety</strong>: it would be
<strong>unsafe</strong> for a state-changing request like <code>POST /transfer?amount=1000</code> or
"place order" — if an attacker replays the captured 0-RTT data, the transfer/order could execute
<em>twice</em>, causing real harm. It's <strong>fine</strong> for an <strong>idempotent</strong> request
like <code>GET /index.html</code> or fetching a static asset: re-sending it just returns the same content
again with no side effects, so a replay is harmless. The rule: only put idempotent, non-state-changing
requests in 0-RTT, and route sensitive operations through the completed (1-RTT) handshake, optionally with
server-side anti-replay tracking.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 20 — Inspecting & Hardening TLS →](lesson-20-tls-hardening){: .btn .btn-primary }
