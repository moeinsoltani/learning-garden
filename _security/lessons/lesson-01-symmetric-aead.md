---
title: "Lesson 1 — Symmetric Encryption & AEAD"
nav_order: 1
parent: "Phase 1: Cryptography Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 1: Symmetric Encryption & AEAD

## Concept

**Symmetric encryption** uses *one shared key* to both encrypt and decrypt. It's the fast workhorse
of all crypto — TLS, disk encryption, VPNs all use it for the bulk data. The whole challenge it
*doesn't* solve is "how do both sides get the same key" (that's Lesson 3).

```
   plaintext ──[ encrypt with key K ]──► ciphertext ──[ decrypt with key K ]──► plaintext
                       same K on both sides
```

The modern twist is **AEAD** — encryption that also *proves the data wasn't tampered with*.

---

## How it works

**Block vs stream ciphers.** A block cipher (AES) encrypts fixed-size blocks; a stream cipher
(ChaCha20) produces a keystream XORed with the data. In practice you don't use a block cipher raw —
you use a **mode of operation** that defines how blocks chain together.

**Why ECB is broken.** The naive mode, ECB, encrypts each block independently, so *identical plaintext
blocks produce identical ciphertext blocks* — patterns in the data show through (the infamous
"ECB penguin"). Real modes mix in position/randomness so identical plaintext encrypts differently.

**The IV/nonce.** Secure modes take a **nonce** ("number used once") or IV so that encrypting the same
message twice gives different ciphertext. Reusing a nonce with the same key is catastrophic for many
modes (it can leak plaintext or break authentication) — nonce uniqueness is a hard rule.

**AEAD — the modern default.** [Authenticated Encryption with Associated Data](https://en.wikipedia.org/wiki/Authenticated_encryption)
(AES-GCM, [ChaCha20-Poly1305](https://en.wikipedia.org/wiki/ChaCha20-Poly1305)) does two jobs at once:
it **encrypts** (confidentiality) and produces an **authentication tag** (integrity + authenticity).
On decryption the tag is checked first; if the ciphertext (or the "associated data" — unencrypted-but-
authenticated headers) was altered, decryption *fails* rather than returning garbage. This is why every
modern protocol uses AEAD, never plain encryption.

{: .note }
> **"128-bit security"**
> A 128-bit key means ~2¹²⁸ possible keys — brute force is computationally infeasible forever with
> classical computers. Key *size* isn't the usual weak point; nonce reuse, bad randomness (Lesson 5),
> and key management are. AES-128 and AES-256 are both fine; 256 buys margin (incl. some quantum
> hedging).

---

## Lab

Encrypt with AEAD and watch tampering get caught.

### Step 1 — Encrypt a file with AES-256-GCM

```bash
$ echo "transfer \$1000 to account 42" > msg.txt
$ openssl enc -aes-256-gcm -pbkdf2 -in msg.txt -out msg.enc -k correct-horse
$ openssl enc -d -aes-256-gcm -pbkdf2 -in msg.enc -k correct-horse
transfer $1000 to account 42        # correct key + intact data → plaintext
```

### Step 2 — Tamper and watch authentication fail

```bash
# Flip a byte in the ciphertext, then try to decrypt:
$ printf '\x00' | dd of=msg.enc bs=1 seek=20 count=1 conv=notrunc 2>/dev/null
$ openssl enc -d -aes-256-gcm -pbkdf2 -in msg.enc -k correct-horse
bad decrypt                          # the auth tag rejects the modified ciphertext
```

### Step 3 — See why ECB leaks patterns (conceptual)

```bash
# Encrypt a file full of repeated blocks with ECB vs CBC; in ECB the repetition is visible.
$ head -c 256 /dev/zero | openssl enc -aes-128-ecb -nosalt -K 00112233445566778899aabbccddeeff | xxd | head
# Repeated identical ciphertext lines appear — the leak ECB is infamous for.
```

---

## Further Reading

| Topic | Link |
|---|---|
| Symmetric-key algorithm | [Wikipedia — Symmetric-key algorithm](https://en.wikipedia.org/wiki/Symmetric-key_algorithm) |
| Authenticated encryption (AEAD) | [Wikipedia — Authenticated encryption](https://en.wikipedia.org/wiki/Authenticated_encryption) |
| Block cipher mode of operation | [Wikipedia — Block cipher mode of operation](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation) |
| AES | [Wikipedia — Advanced Encryption Standard](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard) |

---

## Checkpoint

**Q1. What does AEAD provide beyond plain encryption, and why does every modern protocol use it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Plain encryption gives only <strong>confidentiality</strong> — it hides the data but doesn't tell you
whether it was modified. <strong>AEAD</strong> (Authenticated Encryption with Associated Data) adds
<strong>integrity and authenticity</strong>: alongside the ciphertext it computes an authentication tag
over both the encrypted data and any "associated data" (headers that are authenticated but not
encrypted). On decryption the tag is verified first, so any tampering with the ciphertext or the
associated data — or use of the wrong key — makes decryption <em>fail</em> instead of returning
manipulated plaintext. Modern protocols use it because encryption without integrity is dangerous
(attackers can flip bits or forge messages); AEAD delivers both guarantees in one efficient step.
</details>

---

**Q2. Why is ECB mode insecure, and what does a nonce/IV fix?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>ECB</strong> encrypts each block independently with no chaining or randomization, so
<em>identical plaintext blocks always produce identical ciphertext blocks</em>. That leaks structure —
repeated patterns in the data remain visible in the ciphertext (the classic "ECB penguin"), revealing
information without breaking the cipher itself. A <strong>nonce/IV</strong> fixes this by injecting
unique per-message randomness/position into the encryption, so the same plaintext encrypts to different
ciphertext each time and identical blocks no longer map to identical output. The catch is that the
nonce must be <strong>unique per key</strong> — reusing a nonce with the same key undermines the mode
(and can be catastrophic for AEAD).
</details>

---

**Q3. Symmetric encryption is fast and secure — so what problem does it leave unsolved?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Key distribution</strong>: symmetric encryption requires both parties to already share the same
secret key, but it provides no way to <em>establish</em> that shared key over an insecure channel in the
first place. If two strangers on the internet want to communicate, they can't just "send the key" —
anyone watching would capture it. This is exactly the gap that asymmetric cryptography and Diffie-Hellman
key exchange (Lesson 3) fill: they let two parties agree on a shared symmetric key without ever
transmitting it, after which fast symmetric AEAD does the bulk encryption. (This pairing is called
hybrid encryption and is how TLS and VPNs work.)
</details>

---

## Homework

Encrypt the same short message twice with AES-256-GCM using `openssl`, once letting openssl pick a fresh
salt/IV and once forcing the same key+IV (`-K`/`-iv`), and compare the ciphertext bytes. Explain what
you observe and why nonce/IV uniqueness is a security requirement, not a convenience.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
When openssl uses a fresh salt/IV each time, the two ciphertexts <strong>differ</strong> even though the
plaintext and passphrase are identical — the randomized IV makes each encryption unique. When you force
the <strong>same key and IV</strong>, the two ciphertexts come out <strong>identical</strong>. That
identical output is the danger: with a repeated key+nonce, an observer can tell that the same message was
sent (and for stream-cipher-based AEAD like GCM, the keystream is reused, so XORing the two ciphertexts
cancels the keystream and leaks the relationship/contents of the plaintexts — and the authentication
guarantee can be broken, allowing forgeries). So nonce/IV uniqueness isn't cosmetic: reuse degrades
confidentiality and can completely break integrity. The rule "never reuse a nonce with the same key" is
a hard cryptographic requirement, which is why protocols carefully derive or counter their nonces.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 2 — Hashing, MACs & Password Storage →](lesson-02-hashing-macs){: .btn .btn-primary }
