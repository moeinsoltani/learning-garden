---
title: "Lesson 16 — The TLS 1.3 Handshake"
nav_order: 16
parent: "Phase 3: TLS & SSL"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 16: The TLS 1.3 Handshake

## Concept

TLS 1.3 redesigned the handshake to be **faster and safer**: **one round trip** instead of two, with most
of the handshake **encrypted**, and all the legacy/weak options removed. The trick is that the client
*guesses* the key-exchange parameters up front, so the server can finish keying in a single exchange.

```
   Client                                   Server
     │── ClientHello (+ key share guess) ──►│
     │◄─ ServerHello (+ key share)          │   keys established here
     │◄─ {EncryptedExtensions, Certificate, │   ← these are ENCRYPTED now
     │    CertificateVerify, Finished}      │
     │── {Finished} ───────────────────────►│
     │═══════ encrypted application data ═══════  (after ONE round trip)
```

---

## How it works

**One round trip (1-RTT).** In TLS 1.2 the client had to wait to learn the server's chosen parameters
before sending its key share. TLS 1.3 has the client send its **key share(s) in the ClientHello** (an
optimistic guess at the group, e.g. X25519). The server picks from those, replies with its share in
ServerHello, and **both can derive keys immediately** — so the server can send its certificate and
Finished in the *same* flight. Data flows after one round trip. (If the client guessed the wrong group,
the server sends a HelloRetryRequest and it costs an extra trip — rare.)

**Encrypted handshake.** Once the ServerHello establishes keys, everything after it — **EncryptedExtensions,
Certificate, CertificateVerify, Finished** — is **encrypted**. So a passive observer no longer sees the
server's certificate on the wire (contrast TLS 1.2; cf. networking Lesson 70). Less metadata leaks.

**Mandatory forward secrecy, legacy removed.** TLS 1.3 **only** allows (EC)DHE key exchange — RSA key
transport is gone — so **every** connection has forward secrecy. It also dropped weak ciphers, MAC-then-
encrypt, renegotiation, and compression. The cipher-suite list shrank to a handful of AEAD suites
(Lesson 17), eliminating whole classes of misconfiguration and downgrade attacks.

**The key schedule.** TLS 1.3 derives keys through a structured HKDF-based "key schedule," producing
separate keys for handshake encryption and application data, plus resumption secrets — a cleaner, more
analyzable design than 1.2's PRF.

{: .note }
> **Faster <em>and</em> safer is unusual**
> Normally security and performance trade off; TLS 1.3 improved both at once. The lesson: a lot of TLS
> 1.2's cost and risk came from <em>flexibility</em> (negotiable legacy options, extra round trips). By
> <em>removing choices</em> — one round trip, mandatory FS, a tiny AEAD-only suite list — 1.3 got faster
> and closed attack surface simultaneously. Simplicity is a security feature (echoing WireGuard,
> networking Lesson 51).

---

## Lab

```bash
# Force TLS 1.3 and observe the handshake
$ openssl s_client -connect wikipedia.org:443 -servername wikipedia.org -tls1_3 -msg </dev/null 2>/dev/null \
    | grep -E 'ClientHello|ServerHello|EncryptedExtensions|Certificate|Finished'
# Note: the Certificate appears AFTER key establishment (it's encrypted on the wire).

# Confirm the negotiated suite is a TLS 1.3 AEAD suite
$ echo | openssl s_client -connect wikipedia.org:443 -tls1_3 2>/dev/null | grep -E 'Protocol|Cipher'
# e.g. Protocol: TLSv1.3  Cipher: TLS_AES_256_GCM_SHA384
```

---

## Further Reading

| Topic | Link |
|---|---|
| TLS 1.3 | [Wikipedia — TLS 1.3](https://en.wikipedia.org/wiki/Transport_Layer_Security#TLS_1.3) |
| HKDF (key schedule) | [Wikipedia — HKDF](https://en.wikipedia.org/wiki/HKDF) |
| Forward secrecy | [Wikipedia — Forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy) |
| TLS 1.2 handshake | [Lesson 15](lesson-15-tls12-handshake) |

---

## Checkpoint

**Q1. How does TLS 1.3 cut the handshake to one round trip?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
By having the client <strong>send its key-exchange share(s) in the ClientHello</strong> — an optimistic
guess at which (EC)DHE group the server will accept — instead of waiting to learn the server's choice
first (as in TLS 1.2). The server picks one of the offered groups, returns its own share in the
ServerHello, and at that point <strong>both sides can derive the session keys immediately</strong>. That
lets the server send the rest of its handshake (certificate, CertificateVerify, Finished) in the
<em>same</em> flight, so application data can flow after just <strong>one round trip</strong>. If the
client's guessed group is unsupported, the server sends a HelloRetryRequest and it falls back to an extra
round trip, but that's the uncommon case.
</details>

---

**Q2. What parts of the handshake does TLS 1.3 encrypt that TLS 1.2 did not, and why does it matter?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Once the ServerHello establishes keys, TLS 1.3 <strong>encrypts the remainder of the handshake</strong> —
EncryptedExtensions, the server's <strong>Certificate</strong>, CertificateVerify, and Finished — whereas
in TLS 1.2 those (notably the certificate) were sent in the clear. It matters because it <strong>reduces
the metadata a passive observer can collect</strong>: a sniffer can no longer read the server's certificate
(and thus its identity details) off the wire, shrinking what surveillance/middleboxes can see (cf.
networking Lesson 70). Combined with ECH (which also hides SNI), TLS 1.3 leaves much less identifying
information visible, improving privacy and resistance to traffic analysis.
</details>

---

**Q3. Name two things TLS 1.3 removed and the security benefit of removing each.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Any two of: (1) <strong>RSA key transport</strong> — removing it makes <strong>forward secrecy
mandatory</strong> (only ephemeral (EC)DHE remains), so a future key compromise can't decrypt past
traffic. (2) <strong>Legacy/weak ciphers</strong> (RC4, 3DES, CBC MAC-then-encrypt modes) — removing them
eliminates known cipher-level attacks and leaves only modern AEAD suites. (3) <strong>Renegotiation</strong>
— removing it kills a class of renegotiation-based attacks. (4) <strong>TLS compression</strong> — removing
it prevents compression-oracle attacks like CRIME. (5) The huge negotiable cipher-suite matrix — shrinking
it to a few AEAD suites <strong>reduces downgrade/misconfiguration surface</strong>. The unifying benefit:
each removal deletes a category of attacks or misconfigurations, so "fewer options" directly means "fewer
ways to be insecure."
</details>

---

## Homework

Capture both a TLS 1.2 and a TLS 1.3 handshake to the same server and compare in Wireshark: count the
round trips before application data, and note which handshake messages are readable vs encrypted in each.
Then explain how TLS 1.3 manages to be both faster and more private, referencing the specific design
changes.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The TLS 1.2 capture shows <strong>two round trips</strong> before application data (ClientHello/ServerHello
+ Certificate/KeyExchange, then ClientKeyExchange/Finished), with the <strong>certificate and most
handshake messages readable</strong> in the clear. The TLS 1.3 capture shows <strong>one round trip</strong>
(ClientHello with key share → ServerHello + the rest), and after the ServerHello the
<strong>Certificate/CertificateVerify/Finished are encrypted</strong>, appearing as opaque handshake data.
TLS 1.3 is <strong>faster</strong> because the client front-loads its key share in the ClientHello, letting
keys be derived in a single exchange so the server completes its half immediately (1-RTT instead of 2). It
is <strong>more private</strong> because keys are established right after ServerHello, so everything
afterward — including the certificate that revealed the server's identity in 1.2 — is encrypted on the
wire. Both gains come from the same restructuring (send key share early, derive keys sooner) plus removing
legacy options; with ECH layered on, even the SNI is hidden, completing the privacy improvement.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 17 — Cipher Suites & Forward Secrecy →](lesson-17-cipher-suites){: .btn .btn-primary }
