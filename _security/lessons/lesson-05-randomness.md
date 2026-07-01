---
title: "Lesson 5 — Randomness & Entropy"
nav_order: 5
parent: "Phase 1: Cryptography Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 5: Randomness & Entropy

## Concept

Every primitive so far — keys, IVs/nonces, DH private values, signature nonces, salts — assumes
**unpredictable random numbers**. If an attacker can guess or reproduce your "random" values, the
strongest cipher collapses. Randomness is the silent foundation, and historically the most common place
real systems break.

```
   good randomness → unpredictable keys/nonces → crypto holds
   bad randomness  → guessable keys/nonces      → crypto silently broken (cipher still "fine")
```

---

## How it works

**CSPRNG vs ordinary PRNG.** A normal PRNG (like C's `rand()`) is fast but **predictable** — given some
output, you can compute the rest. A **CSPRNG** (cryptographically secure PRNG) is designed so that
observing output reveals nothing about past or future output. Crypto must use a CSPRNG, never `rand()`.

**Where to get it on Linux.** The kernel CSPRNG is exposed via **`/dev/urandom`** and the
**`getrandom(2)`** syscall, seeded from hardware entropy sources (interrupt timings, hardware RNG). The
modern guidance is simple: **use `getrandom()`/`/dev/urandom`** — once seeded, it's secure and
non-blocking. (The old `/dev/random` "blocking for entropy" model caused more bugs than it solved.)

**Entropy at boot — the real risk.** The danger is using the CSPRNG **before it's seeded** — early in
boot, or on embedded devices/VMs with no entropy sources. `getrandom()` blocks until seeded precisely to
prevent this. Famous failures came from generating keys at first boot with insufficient entropy, so many
devices produced **identical or predictable keys**.

**Nonce reuse = catastrophe.** This ties back to earlier lessons: reusing a nonce in AES-GCM can leak
plaintext and break authentication (Lesson 1); reusing the per-signature nonce `k` in ECDSA lets an
attacker **recover the private key** from two signatures (the famous PlayStation 3 break). Ed25519
avoids this by deriving its nonce deterministically — a design response to how often randomness goes
wrong.

{: .note }
> **The meta-lesson**
> Cryptographic breaks rarely come from breaking the cipher math; they come from <strong>randomness,
> nonce reuse, and key management</strong>. When you see "use the library, don't roll your own," a huge
> part of why is that libraries handle entropy and nonces correctly and you probably won't.

---

## Lab

```bash
# Get cryptographic random bytes the right way
$ head -c 32 /dev/urandom | xxd | head      # 256 bits of CSPRNG output for a key

# Demonstrate the danger of a predictable PRNG: seeded the same way, output repeats
$ python3 -c 'import random; random.seed(1234); print([random.randint(0,99) for _ in range(5)])'
$ python3 -c 'import random; random.seed(1234); print([random.randint(0,99) for _ in range(5)])'
# Identical lists — a seeded non-crypto PRNG is fully reproducible (NEVER use for keys).

# The crypto-safe source is different every time:
$ python3 -c 'import secrets; print(secrets.token_hex(16))'
$ python3 -c 'import secrets; print(secrets.token_hex(16))'
# Different each run — `secrets` uses the OS CSPRNG (getrandom).
```

---

## Further Reading

| Topic | Link |
|---|---|
| CSPRNG | [Wikipedia — Cryptographically secure PRNG](https://en.wikipedia.org/wiki/Cryptographically_secure_pseudorandom_number_generator) |
| /dev/random | [Wikipedia — /dev/random](https://en.wikipedia.org/wiki//dev/random) |
| `getrandom` | [man7.org — getrandom(2)](https://man7.org/linux/man-pages/man2/getrandom.2.html) |
| Entropy (computing) | [Wikipedia — Entropy (computing)](https://en.wikipedia.org/wiki/Entropy_(computing)) |

---

## Checkpoint

**Q1. What is the difference between a CSPRNG and an ordinary PRNG, and why must crypto use the former?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An ordinary PRNG (e.g. <code>rand()</code>) is designed for speed and statistical spread, not secrecy —
it's <strong>predictable</strong>: from some output (or the seed) an attacker can reconstruct past and
future values. A <strong>CSPRNG</strong> is designed so that observing any amount of output gives an
attacker <strong>no feasible way to predict</strong> other output or recover its internal state. Crypto
must use a CSPRNG because keys, nonces, IVs, and salts derive their security entirely from being
unpredictable; if they come from a predictable PRNG, an attacker can reproduce them and the encryption/
signatures are defeated regardless of how strong the cipher is. On Linux that means <code>getrandom()</code>/
<code>/dev/urandom</code>, never <code>rand()</code>.
</details>

---

**Q2. What is the real-world risk around entropy "at boot," and how does `getrandom()` address it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The risk is generating cryptographic material <strong>before the CSPRNG has been seeded with enough
real entropy</strong> — which happens early in boot or on embedded devices/VMs that lack rich entropy
sources (no disks, no user input, predictable timing). If keys are generated then, the "random" values
can be weak or even <strong>identical across many devices</strong>, making the keys guessable (this has
caused real fleet-wide key collisions). <code>getrandom(2)</code> addresses it by <strong>blocking until
the kernel CSPRNG is properly seeded</strong>, after which it returns secure bytes without blocking
again. So instead of silently returning low-entropy output, it forces callers to wait until randomness
is trustworthy — preventing the "weak keys at first boot" class of failures.
</details>

---

**Q3. Why is reusing a nonce so dangerous? Give an example.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A nonce ("number used once") must be unique per key; reusing one breaks the cryptography even though the
algorithm is unchanged. Two examples: (1) In <strong>AES-GCM</strong> (Lesson 1), reusing a nonce with
the same key reuses the keystream — XORing the two ciphertexts cancels it and leaks relationships between
the plaintexts, and it also allows forging the authentication tag, destroying both confidentiality and
integrity. (2) In <strong>ECDSA</strong>, the per-signature random value <code>k</code> must be unique;
if the same <code>k</code> is used for two different signatures, an attacker can solve a simple equation
to <strong>recover the signer's private key</strong> outright (this is how the PlayStation 3 signing key
was extracted). Because nonce reuse is so catastrophic and so easy to do accidentally, modern designs
like Ed25519 derive the nonce deterministically to remove the chance of reuse.
</details>

---

## Homework

Research one real-world cryptographic failure caused by bad randomness (e.g. the Debian OpenSSL 2008
entropy bug, the PS3 ECDSA `k` reuse, or weak embedded-device keys). Summarize what went wrong at the
randomness layer and how it was exploited, then state which lesson-1-through-5 concept would have
prevented it.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Example — the <strong>Debian OpenSSL bug (2008)</strong>: a well-meaning code change removed a line that
fed entropy into OpenSSL's PRNG, so on Debian/Ubuntu systems the PRNG was effectively seeded only by the
process ID (~32,768 possibilities). Every key generated on those systems for ~2 years (SSH host/user
keys, TLS keys, etc.) came from this tiny space, so attackers could <strong>precompute all possible keys
and simply look up the match</strong> — instantly compromising affected keys without breaking any cipher.
(Alternative — <strong>PS3 ECDSA</strong>: Sony reused the same signature nonce <code>k</code>, letting
researchers recover the master private key from two signatures.) What went wrong at the randomness layer:
the CSPRNG had almost no real entropy, making its output predictable. What would have prevented it: the
core principle of this lesson — <strong>use a properly seeded CSPRNG (getrandom/urandom) and never reduce
its entropy</strong>, plus, for the PS3 case, <strong>never reuse a nonce</strong> (or use deterministic
nonces as Ed25519 does). The unifying point: the cipher math was fine; the randomness wasn't — which is
why entropy and nonce handling are treated as first-class security concerns.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 6 — X.509 Certificates Anatomy →](lesson-06-x509){: .btn .btn-primary }
