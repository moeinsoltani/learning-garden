---
title: "Lesson 49 — VPN Cryptography Building Blocks"
nav_order: 49
parent: "Phase 16: Tunnels & VPNs"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 49: VPN Cryptography Building Blocks

{: .note }
> **Scope of this lesson**
> This is a deliberately *thin, VPN-focused primer* so the rest of Phase 16 stands on its own.
> The full, hands-on treatment of cryptography — ciphers, hashing, signatures, randomness — lives
> in the **Security & Identity** track, Phase 1. If you want depth, go there; if you just need
> enough to understand WireGuard and IPsec, this lesson is enough.

## Concept

A plain tunnel (Lesson 48) hides nothing — anyone on the path can read and modify the inner
packet. A **VPN** adds three guarantees, all from cryptography:

- **Confidentiality** — eavesdroppers can't read the contents.
- **Integrity** — tampering is detected and rejected.
- **Authenticity** — you're really talking to the intended peer, not an impostor.

```
   plain tunnel:   [ outer | inner ]            ← readable, forgeable
   VPN tunnel:     [ outer | ENCRYPTED+SEALED ] ← unreadable, tamper-evident, peer-verified
```

Note what a VPN does **not** give you: it doesn't hide *that* you're talking, or *how much*
(traffic volume/timing), and it doesn't protect the inner data once it leaves the far end.

---

## How it works

Three primitives do all the work:

**1. Symmetric encryption (AEAD).** A single shared key both encrypts and decrypts. Modern VPNs
use **AEAD** ciphers ([ChaCha20-Poly1305](https://en.wikipedia.org/wiki/ChaCha20-Poly1305),
AES-GCM) that encrypt *and* authenticate in one step — so confidentiality and integrity come
together. Fast, but both sides need the same secret key.

**2. Asymmetric crypto + Diffie-Hellman.** How do two strangers get a shared key over a wire an
attacker is watching? [Diffie-Hellman](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange)
key exchange: each side has a key *pair* (public + private), they exchange public halves, and each
independently computes the *same* shared secret — without that secret ever crossing the wire. The
attacker sees both public values and still can't derive it. WireGuard uses the elliptic-curve
variant, **Curve25519**.

**3. Authentication.** Encryption alone doesn't tell you *who* is on the other end. Peers prove
identity with either a **pre-shared key (PSK)** — a secret configured on both ends — or
**public-key identity**: you know the peer's public key in advance, so only the holder of the
matching private key can complete the handshake. WireGuard uses the public-key model (peers list
each other's public keys); IPsec/IKE can use PSKs or certificates.

{: .note }
> **Forward secrecy**
> Good VPNs derive a fresh *ephemeral* key per session via Diffie-Hellman, then throw it away.
> So even if your long-term private key is stolen later, yesterday's captured traffic stays
> unreadable — there's no single key that decrypts everything. This is why WireGuard re-keys
> every couple of minutes.

**Why authentication is non-negotiable.** Encryption *without* authentication lets an attacker
flip bits in the ciphertext to silently corrupt or manipulate the decrypted result, or act as a
**man-in-the-middle**, terminating your "encrypted" session and relaying it. AEAD prevents this:
any modification breaks the authentication tag and the packet is dropped.

---

## Lab

No new tunnel here — just look at the primitives with `openssl` so they're concrete.

### Step 1 — Symmetric AEAD encryption round-trip

```bash
$ echo "secret payload" > msg.txt
$ openssl enc -aes-256-gcm -pbkdf2 -in msg.txt -out msg.enc -k hunter2
$ openssl enc -d -aes-256-gcm -pbkdf2 -in msg.enc -k hunter2
secret payload          # decrypts correctly with the right key
```

### Step 2 — A Diffie-Hellman style key agreement (X25519)

```bash
# Two parties each generate a keypair
$ openssl genpkey -algorithm X25519 -out alice.pem
$ openssl genpkey -algorithm X25519 -out bob.pem
$ openssl pkey -in alice.pem -pubout -out alice.pub
$ openssl pkey -in bob.pem   -pubout -out bob.pub

# Each derives the SAME shared secret from their private + the other's public
$ openssl pkeyutl -derive -inkey alice.pem -peerkey bob.pub | sha256sum
$ openssl pkeyutl -derive -inkey bob.pem   -peerkey alice.pub | sha256sum
# both hashes match — the shared secret never crossed the wire
```

The two `sha256sum` outputs are identical: Alice and Bob computed the same secret from public
exchange. This is the exact mechanism behind a WireGuard handshake.

---

## Further Reading

| Topic | Link |
|---|---|
| Diffie-Hellman key exchange | [Wikipedia — Diffie–Hellman](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange) |
| AEAD | [Wikipedia — Authenticated encryption](https://en.wikipedia.org/wiki/Authenticated_encryption) |
| ChaCha20-Poly1305 | [Wikipedia — ChaCha20-Poly1305](https://en.wikipedia.org/wiki/ChaCha20-Poly1305) |
| Curve25519 | [Wikipedia — Curve25519](https://en.wikipedia.org/wiki/Curve25519) |
| Forward secrecy | [Wikipedia — Forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy) |
| (Full treatment) | Security & Identity track, Phase 1 |

---

## Checkpoint

**Q1. Two peers need a shared symmetric key but can only communicate over a wire an attacker is reading. How do they establish one?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With a <strong>Diffie-Hellman key exchange</strong>. Each peer generates a key pair and sends only its <em>public</em> key over the wire. Each then combines its own private key with the other's public key, and the math yields the <em>same</em> shared secret on both sides — without that secret ever being transmitted. The attacker sees both public keys but cannot compute the shared secret (the security rests on the discrete-log / elliptic-curve problem). The agreed secret is then used as (or to derive) the symmetric AEAD key. WireGuard does exactly this with Curve25519.
</details>

---

**Q2. Why is encryption without authentication dangerous, and how do AEAD ciphers address it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Encryption alone gives confidentiality but not <em>integrity</em> or <em>authenticity</em>: an attacker who can't read the traffic can still modify ciphertext (bit-flipping can predictably alter the decrypted plaintext) or impersonate an endpoint / mount a man-in-the-middle. The receiver would happily decrypt forged or tampered data. <strong>AEAD</strong> (Authenticated Encryption with Associated Data) ciphers like ChaCha20-Poly1305 or AES-GCM produce an authentication tag over the ciphertext; on decryption the tag is verified first, and any modification (or wrong key) makes verification fail so the packet is rejected. So AEAD delivers confidentiality and integrity together, which is why every modern VPN uses it.
</details>

---

**Q3. What is forward secrecy, and what concrete attack does it defeat?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Forward secrecy means each session's encryption key is derived from <em>ephemeral</em> Diffie-Hellman values that are discarded after use, so no single long-lived key can decrypt past sessions. The attack it defeats is "<strong>capture now, decrypt later</strong>": an adversary records your encrypted traffic today and steals your long-term private key months later. Without forward secrecy that key could decrypt all the recorded traffic; with it, each session used a throwaway key that no longer exists, so the recordings stay unreadable. This is why WireGuard re-keys roughly every two minutes rather than using one static key forever.
</details>

---

## Homework

Using the `openssl pkeyutl -derive` X25519 example above, deliberately give Bob the *wrong* public
key for Alice (use a third keypair's public key) and observe that the derived secrets no longer
match. Then explain, in terms of the three guarantees (confidentiality/integrity/authenticity),
which guarantee a VPN loses if it skips verifying the peer's public key, and what real-world attack
becomes possible.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With a mismatched public key the two derived secrets differ, so any AEAD traffic encrypted by one side fails to authenticate/decrypt on the other — communication simply breaks. The guarantee at stake is <strong>authenticity</strong>: if a VPN encrypts to whatever public key it's handed without verifying it belongs to the intended peer, an attacker can insert <em>their own</em> public key, complete a key exchange with each side separately, and sit in the middle decrypting and re-encrypting everything — a <strong>man-in-the-middle attack</strong>. The traffic is still "encrypted" (confidentiality and integrity hold on each leg), but you're talking to the attacker, not your peer. This is why WireGuard requires you to configure each peer's public key out-of-band: knowing the correct public key in advance is what authenticates the far end.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 50 — IPsec and the xfrm Framework →](lesson-50-ipsec-xfrm){: .btn .btn-primary }
