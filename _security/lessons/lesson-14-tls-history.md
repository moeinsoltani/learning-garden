---
title: "Lesson 14 — SSL → TLS History & Versions"
nav_order: 14
parent: "Phase 3: TLS & SSL"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 14: SSL → TLS History & Versions

## Concept

[TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security) (Transport Layer Security) is the protocol
that secures almost everything — HTTPS, email, APIs, VPNs. It's the practical application of everything in
Phases 1–2: symmetric AEAD, key exchange, signatures, and certificates, combined into one handshake +
record protocol. Knowing the version history matters because **old versions are insecure and must be
disabled**.

```
   SSL 2.0 (1995) ─ SSL 3.0 ─ TLS 1.0 ─ TLS 1.1 ─ TLS 1.2 (2008) ─ TLS 1.3 (2018)
   └──────── broken / deprecated ────────┘        └─── current, use these ───┘
```

---

## How it works

**The lineage.** "SSL" was Netscape's original (SSL 2.0, 3.0); it was renamed **TLS** when standardized.
Each version fixed flaws in the last:
- **SSL 2.0/3.0** — fundamentally broken (POODLE killed SSL 3.0); long disabled.
- **TLS 1.0 / 1.1** — inherited weaknesses (weak MAC constructions, vulnerable to BEAST-style attacks);
  **deprecated** by browsers and standards bodies — do not use.
- **TLS 1.2 (2008)** — still secure *if configured well*; added AEAD support and modern hashes. Widely
  deployed; acceptable but being superseded.
- **TLS 1.3 (2018)** — a major cleanup: removed all the legacy/weak crypto, made forward secrecy
  mandatory, encrypted most of the handshake, and cut it to 1 round trip (Lesson 16).

**Where TLS sits.** TLS runs **between the transport (TCP) and the application** — the app hands it
plaintext, TLS encrypts it into records over TCP. (On the wire you saw this from the networking side in
networking Lesson 70; here we go inside the protocol.) QUIC (networking Lesson 69) bakes TLS 1.3 directly
into the transport.

**Two phases of TLS.** Every TLS connection has a **handshake** (authenticate, agree keys — uses the
asymmetric crypto of Phase 1–2) followed by the **record protocol** (bulk data, symmetric AEAD). Lessons
15–16 dissect the handshakes.

{: .note }
> **Why old versions must die, not just "be avoided"**
> Leaving TLS 1.0/1.1 (or SSL 3.0) enabled is dangerous even if clients prefer newer versions, because of
> <strong>downgrade attacks</strong> (Lesson 21): an active attacker can force both sides down to the
> weakest commonly-supported version, then exploit it. The only safe posture is to <em>disable</em> the
> old versions entirely so they can't be negotiated at all.

---

## Lab

```bash
# Which versions does a server support? Probe each explicitly.
$ for v in tls1 tls1_1 tls1_2 tls1_3; do
    echo -n "$v: "; echo | openssl s_client -connect wikipedia.org:443 -$v 2>/dev/null \
      | grep -q "Protocol" && echo "supported" || echo "rejected"; done
# Good servers: tls1/tls1_1 rejected, tls1_2/tls1_3 supported.

# See the negotiated version of a normal connection
$ echo | openssl s_client -connect wikipedia.org:443 2>/dev/null | grep -E 'Protocol|Cipher'
```

---

## Further Reading

| Topic | Link |
|---|---|
| Transport Layer Security | [Wikipedia — Transport Layer Security](https://en.wikipedia.org/wiki/Transport_Layer_Security) |
| TLS 1.3 | [Wikipedia — TLS 1.3](https://en.wikipedia.org/wiki/Transport_Layer_Security#TLS_1.3) |
| POODLE (killed SSL 3.0) | [Wikipedia — POODLE](https://en.wikipedia.org/wiki/POODLE) |
| TLS on the wire | networking Lesson 70 (TLS at the packet level) |

---

## Checkpoint

**Q1. Where does TLS sit in the stack, and what are its two phases?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
TLS sits <strong>between the transport layer (TCP) and the application</strong>: the application hands TLS
its plaintext, and TLS turns it into encrypted records carried over TCP (so the app gets a secure channel
without implementing crypto itself). Its two phases are (1) the <strong>handshake</strong> — where the
peers authenticate (via certificates), agree on parameters, and establish shared keys, using the
asymmetric crypto and PKI of Phases 1–2 — and (2) the <strong>record protocol</strong> — where the bulk
application data is encrypted and authenticated with fast symmetric AEAD using the keys from the handshake.
Handshake = set up trust and keys (expensive, once); record protocol = move data (cheap, continuous).
</details>

---

**Q2. Which TLS versions are acceptable today, and which must be disabled?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Acceptable: <strong>TLS 1.2</strong> (secure when configured with strong cipher suites) and
<strong>TLS 1.3</strong> (the modern best choice). Must be disabled: <strong>SSL 2.0, SSL 3.0, TLS 1.0,
and TLS 1.1</strong> — these are deprecated and carry known weaknesses (broken constructions, BEAST/POODLE-
class vulnerabilities). The goal is to offer only 1.2 and 1.3, with 1.3 preferred, and to turn the older
versions off entirely rather than merely deprioritizing them.
</details>

---

**Q3. Why isn't it enough to simply prefer newer TLS versions while leaving old ones enabled?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because of <strong>downgrade attacks</strong>. The TLS version is negotiated, and an <em>active</em>
man-in-the-middle attacker can interfere with that negotiation to force both endpoints down to the
<strong>weakest version they both still support</strong>, even if both would have preferred a newer one.
If an obsolete version like TLS 1.0 or SSL 3.0 remains enabled "just in case," the attacker can push the
connection onto it and then exploit that version's known flaws (e.g. POODLE on SSL 3.0). So preference
isn't protection — the only safe posture is to <strong>disable the old versions entirely</strong> so they
can't be negotiated under any circumstances. (TLS 1.3 also adds explicit downgrade-protection signaling,
but removing the weak options is the fundamental fix.)
</details>

---

## Homework

Use an online scanner (SSL Labs) or `testssl.sh` against a website and record which TLS versions and
cipher suites it offers, plus its grade. Identify anything that would lower the grade (old versions, weak
ciphers) and explain the remediation. Tie at least one finding back to a downgrade or known-attack risk.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A scan reports the protocol versions offered (ideally only TLS 1.2 + 1.3), the cipher suites, key
strength, certificate details, and an overall grade. Grade-lowering findings typically include:
<strong>TLS 1.0/1.1 still enabled</strong> (remediation: disable them, leaving only 1.2/1.3),
<strong>weak/legacy cipher suites</strong> like RC4, 3DES, or non-forward-secret RSA key exchange
(remediation: restrict to modern AEAD suites with ECDHE), missing forward secrecy, or weak DH parameters.
Tying to risk: if the scan shows <strong>TLS 1.0 enabled</strong>, that's a downgrade-attack and BEAST
exposure — an attacker can force the connection onto 1.0 and exploit it, so even though modern clients
wouldn't choose it, leaving it on is a real vulnerability; the fix (disable it) directly removes that
attack surface. Similarly, an enabled RC4 cipher invites known keystream-bias attacks and should be
removed. The general remediation theme: offer only modern versions and AEAD+ECDHE cipher suites, which is
exactly what TLS 1.3 enforces by default.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 15 — The TLS 1.2 Handshake →](lesson-15-tls12-handshake){: .btn .btn-primary }
