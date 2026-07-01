---
title: "Lesson 17 — Cipher Suites & Forward Secrecy"
nav_order: 17
parent: "Phase 3: TLS & SSL"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 17: Cipher Suites & Forward Secrecy

## Concept

A **cipher suite** is the bundle of algorithms a TLS connection agrees to use. Reading one tells you
exactly how a connection is protected — and whether it has **forward secrecy**, the property that past
traffic stays safe even if the server's long-term key is later stolen.

```
   TLS 1.2:  ECDHE - RSA - AES256_GCM - SHA384
             │       │     │            └ hash / PRF
             │       │     └ symmetric AEAD cipher
             │       └ authentication (cert/signature type)
             └ key exchange (ECDHE → forward secrecy)

   TLS 1.3:  TLS_AES_256_GCM_SHA384   (key exchange/auth negotiated separately; always (EC)DHE)
```

---

## How it works

**Anatomy (TLS 1.2).** A 1.2 suite names four things: **key exchange** (how the shared secret is
established — ECDHE/DHE/RSA), **authentication** (the server's cert/signature type — RSA/ECDSA), the
**symmetric cipher** (ideally an AEAD like AES-GCM or ChaCha20-Poly1305), and the **hash** (for the
KDF/PRF, e.g. SHA-256/384). All four must be agreed.

**TLS 1.3 simplified this.** Because 1.3 mandates (EC)DHE and AEAD, the suite name only specifies the
**AEAD cipher + hash** (e.g. `TLS_AES_128_GCM_SHA256`); key exchange and signature algorithms are
negotiated in separate extensions. There are just a handful of suites — far less to get wrong.

**Forward secrecy.** A suite provides [forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy)
**iff** it uses an *ephemeral* key exchange — **ECDHE** or **DHE** (the "E" = ephemeral). Each session
uses throwaway DH keys, so even if the server's long-term private key is compromised later, recorded past
sessions can't be decrypted (no single key unlocks them). A suite with plain **RSA** key exchange has *no*
forward secrecy — one stolen key decrypts all past captured traffic. TLS 1.3 makes FS universal by allowing
only ephemeral exchange.

**Choosing & ordering.** Servers should offer only strong AEAD suites with ECDHE, and the ordering
expresses preference. The modern advice is simple: TLS 1.3 first, plus a curated TLS 1.2 set of
ECDHE+AEAD suites; drop everything legacy (RC4, 3DES, CBC, static RSA).

{: .note }
> **Read FS straight off the name**
> The fastest security check on a TLS 1.2 connection: does the cipher suite start with
> <strong>ECDHE</strong> or <strong>DHE</strong>? If yes → forward secrecy. If it's a static
> <strong>RSA</strong> key exchange → no forward secrecy, a recording-now-decrypt-later risk. In TLS 1.3
> you don't even have to check — it's always ephemeral.

---

## Lab

```bash
# What suite gets negotiated, and does it have forward secrecy?
$ echo | openssl s_client -connect wikipedia.org:443 2>/dev/null | grep -E 'Protocol|Cipher'
# ECDHE-* (1.2) or TLS_AES_*-GCM (1.3) → forward secrecy.

# List the cipher suites your openssl supports, filtered to forward-secret AEAD ones
$ openssl ciphers -v 'ECDHE+AESGCM:ECDHE+CHACHA20' | awk '{print $1, $3, $4, $5}' | head

# Test whether a server still offers a WEAK suite (should fail/return nothing)
$ echo | openssl s_client -connect wikipedia.org:443 -cipher 'RC4' 2>/dev/null | grep -i cipher || echo "RC4 not offered (good)"
```

---

## Further Reading

| Topic | Link |
|---|---|
| Cipher suite | [Wikipedia — Cipher suite](https://en.wikipedia.org/wiki/Cipher_suite) |
| Forward secrecy | [Wikipedia — Forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy) |
| AEAD | [Lesson 1 — Symmetric encryption & AEAD](lesson-01-symmetric-aead) |
| `openssl ciphers` | [openssl ciphers](https://docs.openssl.org/master/man1/openssl-ciphers/) |

---

## Checkpoint

**Q1. What four things does a TLS 1.2 cipher suite specify, and how did TLS 1.3 simplify this?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A TLS 1.2 cipher suite specifies: (1) the <strong>key exchange</strong> method (e.g. ECDHE, DHE, or static
RSA — how the shared secret is established), (2) the <strong>authentication</strong> / signature type (e.g.
RSA or ECDSA — what the server's certificate uses), (3) the <strong>symmetric cipher</strong> for bulk data
(ideally an AEAD like AES-GCM or ChaCha20-Poly1305), and (4) the <strong>hash</strong> used for the key-
derivation/PRF (e.g. SHA-256/384). TLS 1.3 <strong>simplified</strong> this by mandating (EC)DHE key
exchange and AEAD ciphers, so the suite name only encodes the <strong>AEAD cipher + hash</strong> (e.g.
<code>TLS_AES_128_GCM_SHA256</code>); key exchange and signature algorithms are negotiated separately. The
result is a handful of suites instead of dozens, drastically reducing misconfiguration.
</details>

---

**Q2. What is forward secrecy, and how can you tell from a cipher suite whether a connection has it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Forward secrecy</strong> means that compromising the server's long-term private key in the future
does <em>not</em> let an attacker decrypt past sessions, because each session's keys were derived from
<strong>ephemeral</strong> Diffie-Hellman values that were discarded. You can tell from the cipher suite by
looking at the <strong>key-exchange portion</strong>: if it's <strong>ECDHE</strong> or <strong>DHE</strong>
(the "E" stands for ephemeral), the connection has forward secrecy; if it uses static <strong>RSA</strong>
key transport (no DHE/ECDHE), it does <em>not</em>. In TLS 1.3 you don't need to check at all — only
ephemeral key exchange is allowed, so every 1.3 connection has forward secrecy by design.
</details>

---

**Q3. Why is a non-forward-secret suite (static RSA key exchange) dangerous even if the cipher is strong?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because the strength of the symmetric cipher is irrelevant if a <em>single</em> long-lived key can unlock
everything. With static RSA key exchange, the client encrypts the session's pre-master secret to the
server's long-term RSA public key, so that one private key effectively protects <em>all</em> sessions. An
attacker can <strong>record encrypted traffic now and decrypt it later</strong> ("harvest now, decrypt
later") the moment they obtain the server's private key — through a future breach, theft, legal compulsion,
or eventual cryptanalysis. Forward-secret (ECDHE/DHE) suites defeat this because each session's key is
ephemeral and gone, so a key compromise exposes nothing recorded in the past. A strong AES-GCM cipher with
static RSA is therefore still a serious risk; forward secrecy is what protects the archive of past traffic.
</details>

---

## Homework

Use `openssl ciphers` and a scanner to compare two servers' offered cipher suites: one modern, one older.
Identify which offer forward secrecy and which (if any) still allow non-FS or weak suites. Then write the
cipher configuration you would deploy for a new service and justify each choice (versions, key exchange,
ciphers).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Comparing the two, the modern server should offer only <strong>ECDHE-based AEAD suites</strong> (and TLS
1.3's <code>TLS_*_GCM/CHACHA20</code>), all forward-secret, while an older server may still list
<strong>static-RSA suites</strong> (no FS) or weak ciphers (RC4, 3DES, CBC) — which a scanner flags. The
configuration I'd deploy for a new service: <strong>enable only TLS 1.3 and TLS 1.2</strong> (disable 1.0/
1.1 and all SSL); for 1.2, restrict to <strong>ECDHE key exchange with AEAD ciphers</strong> —
<code>ECDHE-ECDSA/RSA-AES128/256-GCM</code> and <code>ECDHE-*-CHACHA20-POLY1305</code> — and remove static
RSA, CBC, 3DES, and RC4 entirely; prefer ECDSA certificates (smaller/faster) with strong curves; and let
TLS 1.3's built-in suites handle the rest. Justification: only modern versions avoids downgrade/legacy
attacks (Lesson 14); ECDHE everywhere guarantees forward secrecy so recorded traffic stays safe after a key
compromise; AEAD ciphers provide integrity+confidentiality and avoid CBC/MAC-then-encrypt flaws; and
removing weak suites shrinks the attack and misconfiguration surface. This mirrors what TLS 1.3 enforces by
default, applied to the 1.2 fallback.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 18 — Mutual TLS (mTLS) →](lesson-18-mtls){: .btn .btn-primary }
