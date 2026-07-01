---
title: "Lesson 3 — Asymmetric Crypto & Diffie-Hellman"
nav_order: 3
parent: "Phase 1: Cryptography Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 3: Asymmetric Crypto & Diffie-Hellman

## Concept

Symmetric crypto (Lesson 1) needs a shared key but can't establish one over an open wire.
**Asymmetric (public-key) cryptography** solves this with a **key pair**: a *public* key you give to
everyone and a *private* key you keep secret. What one key does, only the other can undo.

```
   encrypt with PUBLIC  → decrypt with PRIVATE   (confidentiality: anyone can send you a secret)
   sign with PRIVATE    → verify with PUBLIC      (authenticity: only you could have signed) [Lesson 4]
```

And **Diffie-Hellman** lets two strangers derive a *shared secret* over a public channel without ever
sending it.

---

## How it works

**RSA vs elliptic curve.** [RSA](https://en.wikipedia.org/wiki/RSA_(cryptosystem)) relies on the
difficulty of factoring huge numbers; [elliptic-curve](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography)
crypto (ECDSA, Ed25519, X25519) relies on the elliptic-curve discrete-log problem and gives equivalent
security with **much smaller keys** (a 256-bit EC key ≈ a 3072-bit RSA key), so it's faster and now
preferred. Asymmetric operations are *slow*, so they're used to set up keys, not to bulk-encrypt.

**Public-key encryption vs key exchange.** You *can* encrypt directly to someone's public key, but in
practice protocols use **key exchange**: agree on a symmetric key, then switch to fast AEAD. This is
**hybrid encryption** and is how TLS, PGP, and VPNs actually work.

**Diffie-Hellman.** Each party picks a private value and computes a public value from it.
They exchange public values, and each combines its *own private* with the *other's public* to compute
the **same** shared secret — which never traversed the wire. An eavesdropper sees both public values
and still can't derive the secret (that's the discrete-log hardness). The modern elliptic-curve form is
**X25519**. Using fresh, throwaway DH keys per session gives **forward secrecy** (Lesson covered again
in TLS).

{: .note }
> **Hybrid encryption: the universal pattern**
> Asymmetric crypto is slow and size-limited; symmetric crypto is fast but needs a shared key. So every
> real system combines them: use asymmetric/DH to establish a symmetric session key, then encrypt all
> the actual data with symmetric AEAD. "Public key to bootstrap, symmetric to do the work."

---

## Lab

```bash
# Generate an RSA and an Ed25519 keypair, compare sizes
$ openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:3072 -out rsa.pem
$ openssl genpkey -algorithm Ed25519 -out ed.pem
$ wc -c rsa.pem ed.pem      # the Ed25519 key is tiny by comparison

# Derive the public keys
$ openssl pkey -in rsa.pem -pubout -out rsa.pub
$ openssl pkey -in ed.pem  -pubout -out ed.pub

# Diffie-Hellman (X25519): two parties derive the SAME secret from public exchange
$ openssl genpkey -algorithm X25519 -out alice.pem
$ openssl genpkey -algorithm X25519 -out bob.pem
$ openssl pkey -in alice.pem -pubout -out alice.pub
$ openssl pkey -in bob.pem   -pubout -out bob.pub
$ openssl pkeyutl -derive -inkey alice.pem -peerkey bob.pub  | sha256sum
$ openssl pkeyutl -derive -inkey bob.pem   -peerkey alice.pub | sha256sum
# Identical digests: Alice and Bob agreed on a secret without transmitting it.
```

---

## Further Reading

| Topic | Link |
|---|---|
| Public-key cryptography | [Wikipedia — Public-key cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography) |
| Diffie-Hellman | [Wikipedia — Diffie–Hellman key exchange](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange) |
| Elliptic-curve cryptography | [Wikipedia — Elliptic-curve cryptography](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography) |
| RSA | [Wikipedia — RSA](https://en.wikipedia.org/wiki/RSA_(cryptosystem)) |

---

## Checkpoint

**Q1. In public-key cryptography, what is each key used for in (a) confidentiality and (b) authenticity?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The key pair works in complementary directions. For <strong>(a) confidentiality</strong>: you
<strong>encrypt with the recipient's public key</strong>, and only the recipient's matching
<strong>private key</strong> can decrypt — so anyone can send a secret that only the key-owner can read.
For <strong>(b) authenticity</strong> (Lesson 4): you <strong>sign with your private key</strong>, and
anyone can <strong>verify with your public key</strong> — since only you hold the private key, a valid
signature proves it came from you (and that the data is intact). The rule of thumb: public key for
sending <em>to</em> someone secretly, private key for proving something came <em>from</em> you.
</details>

---

**Q2. How does Diffie-Hellman let two strangers agree on a secret over a channel an attacker is reading?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Each party generates a private value and derives a public value from it, then they exchange
<em>only the public values</em>. Each side then combines <strong>its own private value with the other
party's public value</strong>, and the mathematics yields the <strong>same shared secret</strong> on
both ends — a secret that was never sent over the wire. An eavesdropper sees both public values but
cannot combine them into the shared secret without one of the private values, because doing so requires
solving the discrete-logarithm (or elliptic-curve discrete-log) problem, which is computationally
infeasible. The agreed secret is then used as a symmetric key for fast AEAD. The modern form is X25519.
</details>

---

**Q3. Why do real systems use "hybrid encryption" rather than just encrypting everything with public keys?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because asymmetric crypto is <strong>slow and size-limited</strong> — public-key operations are orders
of magnitude more expensive than symmetric ones, and you can only directly encrypt a small amount of data
with them. Symmetric crypto (AEAD) is <strong>fast and handles bulk data</strong> but needs a pre-shared
key. <strong>Hybrid encryption</strong> takes the best of both: use asymmetric crypto / Diffie-Hellman
<em>once</em> to establish (or transport) a symmetric session key, then encrypt all the actual data with
fast symmetric AEAD. This is how TLS, PGP/encrypted email, and VPNs work — "public key to bootstrap the
key, symmetric key to do the work" — giving you public-key convenience for key setup and symmetric speed
for throughput.
</details>

---

## Homework

Compare the cost of the two paradigms: time 10,000 RSA-3072 signatures vs 10,000 AES-GCM encryptions of
a small block using `openssl speed rsa3072` and `openssl speed -evp aes-256-gcm`. Use the numbers to
explain, quantitatively, why TLS uses RSA/ECDHE only during the handshake and switches to symmetric
encryption for the session data.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<code>openssl speed</code> shows asymmetric operations measured in <em>hundreds to a few thousand per
second</em> while symmetric AES-GCM runs at <em>gigabytes per second</em> — a difference of several
orders of magnitude per unit of data. That gap explains TLS's design directly: if every byte of a
connection were protected with RSA/ECDHE-style public-key math, throughput would be catastrophically low
and CPU usage enormous. Instead, TLS confines the expensive asymmetric work to the <strong>handshake</strong>,
where it's done a small fixed number of times to authenticate the peer and establish a shared symmetric
key (via ECDHE), and then encrypts <em>all the bulk session data</em> with fast symmetric AEAD. So the
costly public-key operations happen once per connection, and the cheap symmetric operations happen per
packet — which is exactly the hybrid-encryption pattern, justified by the measured performance gap.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 4 — Digital Signatures →](lesson-04-signatures){: .btn .btn-primary }
