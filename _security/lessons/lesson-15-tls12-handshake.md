---
title: "Lesson 15 — The TLS 1.2 Handshake"
nav_order: 15
parent: "Phase 3: TLS & SSL"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 15: The TLS 1.2 Handshake

## Concept

The handshake is where TLS authenticates the server and both sides agree on keys, before any application
data flows. Understanding TLS 1.2's handshake message-by-message makes TLS 1.3's improvements (Lesson 16)
obvious. It takes **two round trips** before data can flow.

```
   Client                                   Server
     │── ClientHello ─────────────────────►│   (versions, ciphers, client random)
     │◄──────────── ServerHello ───────────│   (chosen cipher, server random)
     │◄──────────── Certificate ───────────│   (server's cert chain)
     │◄──────────── ServerKeyExchange ─────│   (ECDHE params, signed)  [if ECDHE]
     │◄──────────── ServerHelloDone ───────│
     │── ClientKeyExchange ────────────────►│   (client's ECDHE share / encrypted secret)
     │── ChangeCipherSpec, Finished ───────►│
     │◄──────────── ChangeCipherSpec, Finished
     │═══════════ encrypted application data ═══════════
```

---

## How it works

**ClientHello / ServerHello.** The client offers its supported TLS versions, cipher suites, and a random
nonce; the server picks one cipher suite and version and sends its own random. These randoms feed into key
derivation so each session's keys are unique.

**Certificate.** The server sends its certificate chain (Lesson 7). The client validates it (Lesson 9) —
this is where the server is **authenticated**.

**Key establishment — two styles.**
- **RSA key transport (legacy):** the client encrypts a "pre-master secret" with the server's RSA public
  key; only the server can decrypt it. Simple, but **no forward secrecy** — if the server's key later
  leaks, all past sessions decrypt.
- **ECDHE (modern):** the server sends signed ephemeral Diffie-Hellman parameters (ServerKeyExchange),
  the client sends its DH share (ClientKeyExchange), and both derive the shared secret. The ephemeral keys
  give **forward secrecy** (Lesson 17). This is what you want.

**Deriving keys & Finished.** Both sides combine the pre-master secret + the two randoms into the
**master secret**, then into the symmetric AEAD keys. **ChangeCipherSpec** signals "switching to
encrypted," and **Finished** (a MAC over the whole handshake) lets each side confirm the handshake wasn't
tampered with — anti-downgrade integrity.

{: .note }
> **Two round trips is the cost TLS 1.3 attacks**
> Notice the data can't flow until after ClientHello→ServerHello→…→client Finished — two full round trips
> of latency before the first byte of application data. On a high-RTT link that's a visible delay. TLS 1.3
> (Lesson 16) restructures the handshake to do it in <em>one</em> round trip (and 0-RTT on resumption).

---

## Lab

```bash
# Force TLS 1.2 and watch the handshake; -msg shows the handshake messages
$ openssl s_client -connect wikipedia.org:443 -servername wikipedia.org -tls1_2 -msg </dev/null 2>/dev/null \
    | grep -E 'ClientHello|ServerHello|Certificate|ServerKeyExchange|Finished'

# See the negotiated cipher suite (look for ECDHE = forward secrecy)
$ echo | openssl s_client -connect wikipedia.org:443 -tls1_2 2>/dev/null | grep -E 'Cipher|Protocol'
# e.g. ECDHE-RSA-AES256-GCM-SHA384 → ECDHE key exchange, AES-GCM AEAD
```

---

## Further Reading

| Topic | Link |
|---|---|
| TLS handshake | [Wikipedia — TLS handshake](https://en.wikipedia.org/wiki/Transport_Layer_Security#TLS_handshake) |
| Forward secrecy | [Wikipedia — Forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy) |
| ECDHE | [Lesson 3 — Asymmetric crypto & DH](lesson-03-asymmetric-dh) |
| `openssl s_client` | [openssl s_client](https://docs.openssl.org/master/man1/openssl-s_client/) |

---

## Checkpoint

**Q1. At a high level, what is exchanged in each round trip of a TLS 1.2 handshake?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>First round trip:</strong> the client sends <strong>ClientHello</strong> (offered versions, cipher
suites, client random); the server responds with <strong>ServerHello</strong> (chosen version/cipher,
server random), its <strong>Certificate</strong> chain, <strong>ServerKeyExchange</strong> (signed
ephemeral DH parameters, for ECDHE), and <strong>ServerHelloDone</strong>. <strong>Second round trip:</strong>
the client validates the cert and sends <strong>ClientKeyExchange</strong> (its DH share / encrypted
pre-master), then <strong>ChangeCipherSpec</strong> + <strong>Finished</strong>; the server replies with
its own <strong>ChangeCipherSpec</strong> + <strong>Finished</strong>. Only after that can encrypted
application data flow — i.e. <strong>two full round trips</strong> of latency before the first byte.
</details>

---

**Q2. What is the difference between RSA key transport and ECDHE for establishing the session key, and why does it matter?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With <strong>RSA key transport</strong>, the client generates a pre-master secret and encrypts it with the
server's <em>long-term RSA public key</em>; only the server's private key can decrypt it. With
<strong>ECDHE</strong>, the two sides perform an <em>ephemeral</em> Diffie-Hellman exchange (the server
signs its DH parameters with its certificate key to authenticate them) and both derive the shared secret.
The crucial difference is <strong>forward secrecy</strong>: RSA key transport has none — if the server's
private key is ever compromised, an attacker who recorded past traffic can decrypt the pre-master secrets
and thus all those sessions. ECDHE uses throwaway keys per session, so a later key compromise can't decrypt
past recordings. That's why ECDHE is required/preferred and RSA key transport was removed in TLS 1.3.
</details>

---

**Q3. What is the purpose of the Finished message?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The <strong>Finished</strong> message is each side's <strong>integrity check over the entire handshake</strong>:
it contains a MAC/PRF computed over all the preceding handshake messages using the freshly established keys.
Both sides send one, and each verifies the other's. This proves two things: (1) both ends derived the
<em>same</em> keys (so the key agreement succeeded), and (2) <strong>no attacker tampered with the
handshake</strong> — if a man-in-the-middle had altered the ClientHello/ServerHello (e.g. to downgrade the
cipher or version), the hash of the transcript wouldn't match and Finished verification would fail,
aborting the connection. So Finished provides handshake authentication and downgrade/tamper protection
before any application data is sent.
</details>

---

## Homework

Capture a TLS 1.2 handshake in Wireshark (or `openssl s_client -msg`) and identify each message from the
diagram. Find the negotiated cipher suite and decode what each part means (key exchange, authentication,
cipher, MAC/hash). Then explain whether the connection has forward secrecy and how you can tell from the
cipher suite name alone.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
In the capture you can map the messages: ClientHello → ServerHello → Certificate → ServerKeyExchange →
ServerHelloDone → ClientKeyExchange → ChangeCipherSpec/Finished (each way). The negotiated cipher suite —
e.g. <code>ECDHE-RSA-AES256-GCM-SHA384</code> — decodes as: <strong>ECDHE</strong> = key-exchange method
(ephemeral elliptic-curve Diffie-Hellman), <strong>RSA</strong> = authentication (the server's certificate/
signature is RSA), <strong>AES256-GCM</strong> = the symmetric AEAD cipher protecting the data, and
<strong>SHA384</strong> = the hash used for key derivation / PRF (GCM provides its own integrity, so this is
the handshake/KDF hash). You can tell it has <strong>forward secrecy from the name alone</strong>: the
presence of <strong>ECDHE</strong> (or DHE) in the key-exchange portion means ephemeral Diffie-Hellman is
used, which gives forward secrecy. If instead the suite started with plain <code>RSA</code> (key transport,
e.g. <code>RSA-AES256-GCM-SHA384</code> with no DHE/ECDHE), there would be <em>no</em> forward secrecy. So
"ECDHE/DHE present → forward secrecy; static RSA key exchange → none."
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 16 — The TLS 1.3 Handshake →](lesson-16-tls13-handshake){: .btn .btn-primary }
