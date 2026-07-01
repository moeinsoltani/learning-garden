---
title: "Lesson 2 — Hashing, MACs & Password Storage"
nav_order: 2
parent: "Phase 1: Cryptography Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 2: Hashing, MACs & Password Storage

## Concept

Three different things get sloppily called "hashing." Keeping them straight is foundational:

```
   Hash:     data ─► fixed-size digest        (integrity check; no key)
   MAC:      data + KEY ─► tag                 (integrity + authenticity; shared key)
   Password hash: password + salt ─► slow digest (storage; deliberately SLOW)
```

A cryptographic **hash** is a one-way fingerprint. A **MAC** is a keyed fingerprint (proves *who*).
A **password hash** is a deliberately *slow* function for storing secrets safely.

---

## How it works

**Cryptographic hashes** (SHA-256, SHA-3) map any input to a fixed-size digest with three properties:
**preimage resistance** (can't reverse the digest to find input), **second-preimage resistance**
(can't find another input with the same digest), and **collision resistance** (can't find *any* two
inputs colliding). They're fast and unkeyed — used for integrity checks, fingerprints, signatures
(Lesson 4), and commitments. (MD5/SHA-1 are broken — collisions found — so avoid them.)

**MACs and HMAC.** A plain hash proves *integrity* but not *authenticity* — anyone can recompute a hash
over modified data. A **MAC** mixes in a secret key, so only key-holders can produce a valid tag,
proving the data came from someone who knows the key. **HMAC** is the standard construction
(`HMAC(key, message)`), designed to be safe even with hashes vulnerable to **length-extension attacks**
(where, for naive `hash(secret‖message)`, an attacker can append data and forge a valid digest without
knowing the secret — HMAC's nested structure prevents this).

**Password hashing is different on purpose.** Storing passwords with a *fast* hash like SHA-256 is a
serious mistake: attackers who steal the database can try **billions of guesses per second** (and use
precomputed rainbow tables). Password hashing functions — **bcrypt, scrypt, [argon2](https://en.wikipedia.org/wiki/Argon2)**
— are deliberately **slow and memory-hard**, so each guess costs real time/RAM, throttling brute force.
Two more essentials:
- **Salt:** a unique random value per password, stored alongside it, so identical passwords hash
  differently and rainbow tables are useless.
- **Pepper:** an optional secret added to all passwords, kept *outside* the database (e.g. in an HSM/
  config), so a DB-only leak still isn't crackable.

{: .note }
> **The one-sentence rule**
> Use a fast hash (SHA-256) for *integrity*, an HMAC for *authenticated integrity with a shared key*,
> and a slow salted password hash (argon2/bcrypt/scrypt) for *storing passwords* — never the reverse.

---

## Lab

```bash
# A plain cryptographic hash
$ echo -n "hello" | openssl dgst -sha256
SHA2-256(stdin)= 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824

# An HMAC (keyed) — changes entirely if the key changes
$ echo -n "hello" | openssl dgst -sha256 -hmac "secretkey"

# A password hash with argon2 (install argon2): slow, salted, tunable cost
$ echo -n "hunter2" | argon2 "$(head -c16 /dev/urandom | base64)" -id -t 3 -m 16 -p 1
# Note the deliberate delay vs the instant SHA-256 above.
```

Run the SHA-256 a million times vs argon2 once and feel the *intended* speed difference — that slowness
is the security feature for passwords.

---

## Further Reading

| Topic | Link |
|---|---|
| Cryptographic hash function | [Wikipedia — Cryptographic hash function](https://en.wikipedia.org/wiki/Cryptographic_hash_function) |
| HMAC | [Wikipedia — HMAC](https://en.wikipedia.org/wiki/HMAC) |
| Key derivation / password hashing | [Wikipedia — Key derivation function](https://en.wikipedia.org/wiki/Key_derivation_function) |
| Argon2 | [Wikipedia — Argon2](https://en.wikipedia.org/wiki/Argon2) |

---

## Checkpoint

**Q1. What is the difference between a hash and a MAC?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A <strong>hash</strong> is an <em>unkeyed</em> one-way function: anyone can compute <code>hash(data)</code>,
so it proves <strong>integrity</strong> (the data matches the digest) but not <em>who</em> produced it —
an attacker who modifies the data can simply recompute a new valid hash. A <strong>MAC</strong> mixes in a
<strong>secret key</strong> (<code>MAC(key, data)</code>), so only someone holding the key can produce a
valid tag; it proves both <strong>integrity and authenticity</strong> (the data is intact <em>and</em>
came from a key-holder). HMAC is the standard MAC construction built from a hash. In short: hash =
fingerprint anyone can make; MAC = fingerprint only key-holders can make.
</details>

---

**Q2. Why is SHA-256 a bad choice for storing passwords, and what should you use instead?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
SHA-256 is <strong>fast</strong> — which is exactly wrong for passwords. If a database of SHA-256 hashes
leaks, an attacker can compute billions of guesses per second on commodity GPUs (and use precomputed
rainbow tables for unsalted hashes), cracking weak/common passwords almost instantly. Passwords should
instead be stored with a <strong>deliberately slow, memory-hard password hashing function</strong> —
<strong>argon2</strong> (preferred), bcrypt, or scrypt — with a <strong>unique random salt per
password</strong> (so identical passwords differ and rainbow tables fail), and optionally a
<strong>pepper</strong> kept outside the database. The slowness/memory cost makes each guess expensive,
turning billions/sec into a tiny fraction of that, which is what actually protects against offline
cracking after a breach.
</details>

---

**Q3. What is a length-extension attack, and how does HMAC avoid it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A <strong>length-extension attack</strong> targets the naive MAC construction <code>hash(secret ‖
message)</code> using Merkle–Damgård hashes (like SHA-256): because the hash's output <em>is</em> its
internal state after processing the input, an attacker who knows <code>hash(secret ‖ message)</code> and
the message length can <strong>resume hashing</strong> and append extra data, producing a valid digest
for <code>secret ‖ message ‖ extra</code> <em>without knowing the secret</em> — forging an authenticated
message. <strong>HMAC</strong> avoids it with a nested, two-pass construction —
<code>hash(key2 ‖ hash(key1 ‖ message))</code> — so the attacker never sees a raw hash state they can
extend; the outer hash with a separate key wraps the inner result. This is why you use HMAC rather than
hand-rolling <code>hash(secret ‖ message)</code>.
</details>

---

## Homework

Generate two argon2 hashes of the *same* password with two *different* salts and confirm the outputs
differ. Then explain, to someone who says "we already hash passwords with SHA-256, isn't that enough?",
exactly what a salted slow hash defends against that a fast unsalted hash does not — name the specific
attacks.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The two argon2 hashes of the same password come out <strong>completely different</strong> because each
used a distinct salt — proving salts make identical passwords store differently. The explanation for the
skeptic: a fast unsalted SHA-256 is vulnerable to several concrete attacks that a salted slow hash
defeats. (1) <strong>Rainbow-table / precomputation attacks</strong> — with no salt, an attacker uses
precomputed tables of hash→password to reverse millions of hashes instantly; a unique per-password
<em>salt</em> makes every hash require its own computation, rendering precomputed tables useless. (2)
<strong>Offline brute-force / dictionary attacks</strong> — SHA-256's speed lets a GPU try billions of
guesses per second against a stolen database; a <em>slow, memory-hard</em> function (argon2/bcrypt/
scrypt) makes each guess cost real time and RAM, cutting the attacker's rate by orders of magnitude. (3)
<strong>Identical-password detection</strong> — unsalted hashes reveal which users share a password
(same hash); salts hide this. So "we hash with SHA-256" protects against essentially none of the
post-breach cracking that actually happens; salting + a slow KDF is what does.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 3 — Asymmetric Crypto & Diffie-Hellman →](lesson-03-asymmetric-dh){: .btn .btn-primary }
