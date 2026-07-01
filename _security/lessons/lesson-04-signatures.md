---
title: "Lesson 4 — Digital Signatures"
nav_order: 4
parent: "Phase 1: Cryptography Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 4: Digital Signatures

## Concept

A **digital signature** is the asymmetric mirror of a MAC (Lesson 2): instead of a shared key, you
sign with your **private** key and anyone verifies with your **public** key. It proves three things at
once — the message is **authentic** (came from the key-holder), **intact** (not modified), and offers
**non-repudiation** (the signer can't credibly deny it, since only they hold the private key).

```
   sign:   message → hash → encrypt-the-hash with PRIVATE key → signature
   verify: recompute hash, check it against signature using PUBLIC key → valid / invalid
```

Signatures are the backbone of certificates (Phase 2), JWTs (Phase 5), and software updates.

---

## How it works

**You sign a hash, not the message.** Asymmetric operations are slow and size-limited, so signing
hashes the message first (Lesson 2) and signs the small fixed-size digest. Verification recomputes the
hash of the received message and checks it against the signature with the public key. If even one bit
of the message changed, the recomputed hash won't match — so the signature covers the *whole* message
despite only operating on the digest.

**Algorithms.** [RSA-PSS](https://en.wikipedia.org/wiki/Probabilistic_signature_scheme) (modern RSA
signing), **ECDSA**, and **Ed25519** (fast, modern, misuse-resistant elliptic-curve signatures).
Ed25519 is a good default today.

**What it proves — and doesn't.** A signature gives authenticity, integrity, and non-repudiation. It
does **not** give confidentiality — a signed message is still readable by anyone (signing ≠ encrypting).
To get both you sign *and* encrypt.

**Signature vs MAC.** Both prove integrity+authenticity, but a **MAC** uses a shared secret (both
parties can produce and verify it, so *no non-repudiation* — either could have made it), while a
**signature** uses a private key only the signer holds (so it *does* give non-repudiation, and verifiers
need only the public key, not a shared secret). Use a MAC between two trusting parties sharing a key;
use a signature for one-to-many or when you need provable authorship.

{: .note }
> **Why this powers the whole web of trust**
> A certificate authority (Phase 2) <em>signs</em> your certificate with its private key; browsers
> <em>verify</em> with the CA's well-known public key. A JWT (Phase 5) is <em>signed</em> by an identity
> provider and verified by the app. Software releases are signed by the vendor and verified by your
> package manager. All of it is this one primitive: sign-private, verify-public over a hash.

---

## Lab

```bash
# Make a keypair, sign a file, verify it
$ openssl genpkey -algorithm Ed25519 -out signer.pem
$ openssl pkey -in signer.pem -pubout -out signer.pub
$ echo "release v1.0 binary" > artifact.txt
$ openssl pkeyutl -sign -inkey signer.pem -rawin -in artifact.txt -out artifact.sig

# Verify with the PUBLIC key → success
$ openssl pkeyutl -verify -pubin -inkey signer.pub -rawin -in artifact.txt -sigfile artifact.sig
Signature Verified Successfully

# Tamper with the artifact, verify again → failure
$ echo "release v1.0 BACKDOORED" > artifact.txt
$ openssl pkeyutl -verify -pubin -inkey signer.pub -rawin -in artifact.txt -sigfile artifact.sig
Signature Verification Failure
```

---

## Further Reading

| Topic | Link |
|---|---|
| Digital signature | [Wikipedia — Digital signature](https://en.wikipedia.org/wiki/Digital_signature) |
| Ed25519 / EdDSA | [Wikipedia — EdDSA](https://en.wikipedia.org/wiki/EdDSA) |
| Non-repudiation | [Wikipedia — Non-repudiation](https://en.wikipedia.org/wiki/Non-repudiation) |
| MAC vs signature | [Wikipedia — Message authentication code](https://en.wikipedia.org/wiki/Message_authentication_code) |

---

## Checkpoint

**Q1. Why do you sign a hash of a message rather than the message itself?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because asymmetric signing operations are <strong>slow and limited in input size</strong>, so signing a
large message directly would be inefficient or impossible. Hashing first reduces any-size message to a
small fixed-size digest, which is what actually gets signed. This loses nothing: the hash is collision-
resistant, so the signature still effectively covers the entire message — verification recomputes the
hash of the received message and any change, even one bit, produces a different digest that fails the
check. So "sign the hash" gives you full-message integrity at the cost of a single cheap signing
operation on a tiny input.
</details>

---

**Q2. What three properties does a digital signature provide, and which one does a MAC lack?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A digital signature provides <strong>authenticity</strong> (it came from the holder of the private key),
<strong>integrity</strong> (the message wasn't altered), and <strong>non-repudiation</strong> (the
signer can't later deny signing, because only they possess the private key). A <strong>MAC</strong>
provides authenticity and integrity too, but it <strong>lacks non-repudiation</strong>: a MAC uses a
<em>shared</em> secret key, so <em>either</em> party holding that key could have produced the tag — you
can't prove which one did, so it isn't binding proof of authorship. Non-repudiation requires that only
one party can create the proof, which is exactly what a private-key signature gives. (Note: neither a
signature nor a MAC provides confidentiality — they don't hide the message.)
</details>

---

**Q3. A vendor signs a software release. Walk through how your package manager verifies it and what an attacker can and cannot do.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The vendor computes a hash of the release and signs it with their <strong>private</strong> key,
publishing the signature alongside the artifact; your package manager already trusts the vendor's
<strong>public</strong> key (shipped with the distro/keyring). To verify, the package manager hashes the
downloaded artifact and checks that hash against the signature using the vendor's public key — if it
validates, the file is authentic (from the vendor) and intact (unmodified). What an attacker
<strong>cannot</strong> do without the private key: produce a valid signature for a modified/malicious
artifact — any tampering changes the hash and fails verification, and they can't forge the vendor's
signature. What an attacker <strong>can</strong> still do: serve the file unsigned or with an invalid
signature (you must actually <em>enforce</em> verification), strip signing entirely if the client doesn't
require it, or compromise the vendor's private key / trick you into trusting a rogue public key — which
is why key protection and trust distribution matter. The signature secures integrity/authenticity, not
the trust-establishment of the key itself.
</details>

---

## Homework

Sign the same file with both an Ed25519 key and an RSA-PSS key, compare the signature sizes and the
signing/verification speed (`openssl speed ed25519 rsa3072`). Then explain why certificate authorities
and protocols have been migrating toward elliptic-curve signatures, citing your measurements.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You'll observe that the <strong>Ed25519 signature and key are far smaller</strong> than the RSA-3072
ones, and that Ed25519 <strong>signs much faster</strong> while both verify quickly (RSA verification is
actually fast, but its signing and key generation are slow, and its keys/signatures are large). The
migration toward elliptic-curve signatures follows directly from these numbers: smaller keys and
signatures mean <strong>less bandwidth and storage</strong> (important when certificates and signatures
are sent in every TLS handshake and embedded in tokens), faster signing reduces <strong>server CPU
cost</strong> at scale, and EC gives <strong>equivalent security with a fraction of the bits</strong>
(a 256-bit curve ≈ 3072-bit RSA). For CAs and protocols issuing and verifying enormous volumes of
signatures, those efficiency gains — smaller, faster, equally secure — are compelling, which is why
ECDSA/Ed25519 are increasingly the default over RSA.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 5 — Randomness & Entropy →](lesson-05-randomness){: .btn .btn-primary }
