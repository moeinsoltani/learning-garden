---
title: "Lesson 24 — Passwords & Credential Storage"
nav_order: 24
parent: "Phase 4: Authentication Fundamentals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 24: Passwords & Credential Storage

## Concept

Passwords are the most common — and most commonly mishandled — credential. Doing them right is mostly about
**how you store them** (so a breach isn't catastrophic) and **how you resist guessing** (online and offline).
This applies the hashing lesson (Lesson 2) to a real system.

```
   NEVER store:   plaintext, or fast-hash(password)
   DO store:      slow_salted_hash(password)   e.g. argon2id(password, unique_salt)
```

---

## How it works

**Storage: slow, salted, modern.** Never store plaintext or reversible-encrypted passwords, and never a
*fast* hash (SHA-256) — a stolen DB of fast hashes is cracked at billions/sec. Use a **password hashing
function** — **argon2id** (preferred), bcrypt, or scrypt — which is deliberately slow and memory-hard, with
a **unique random salt per user** (so identical passwords differ and rainbow tables are useless), as covered
in Lesson 2. Optionally a **pepper** kept outside the DB adds a layer a DB-only breach can't crack.

**Resisting online guessing.** Even with perfect storage, attackers try logins directly: **credential
stuffing** (reusing leaked username/password pairs from other breaches) and brute force. Defenses:
**rate limiting**, **account lockout / exponential backoff**, CAPTCHAs, and MFA (Lesson 25). Detect reused
breached passwords with **k-anonymity / Have I Been Pwned** (check a password's hash prefix against the
breach corpus without revealing the password).

**Password policies that actually help.** Modern guidance (NIST) flips old advice: **length beats
complexity** (a long passphrase > forced symbols), **don't force periodic rotation** (it leads to weaker,
predictable passwords), **screen against known-breached passwords**, and allow paste/managers. The arbitrary
"1 upper, 1 symbol, change every 90 days" rules harmed usability without improving security.

**The real fix is reducing reliance on passwords.** Passwords are phishable and reusable; the trajectory is
toward MFA (Lesson 25) and passwordless **passkeys** (Lesson 26).

{: .note }
> **Online vs offline attacks — two different defenses**
> <strong>Offline</strong> (attacker has your hash DB) is defended by <em>how you store</em> — slow salted
> hashing makes cracking expensive. <strong>Online</strong> (attacker guesses at your login endpoint) is
> defended by <em>how you respond</em> — rate limiting, lockout, breached-password screening, MFA. You need
> both; they protect against different attacker positions.

---

## Lab

```bash
# Store a password the right way: argon2id with a unique salt
$ SALT=$(head -c16 /dev/urandom | base64)
$ echo -n "correct horse battery staple" | argon2 "$SALT" -id -t 3 -m 16 -p 1 -e
$argon2id$v=19$m=65536,t=3,p=1$...      # slow, salted, memory-hard hash to store

# Check a password against the Have I Been Pwned corpus WITHOUT sending the password (k-anonymity)
$ pw="password123"
$ hash=$(echo -n "$pw" | sha1sum | awk '{print toupper($1)}')
$ prefix=${hash:0:5}; suffix=${hash:5}
$ curl -s "https://api.pwnedpasswords.com/range/$prefix" | grep -i "$suffix"
# A match (with a big count) = this password is in known breaches → reject it.
```

---

## Further Reading

| Topic | Link |
|---|---|
| Password / credential storage | [Wikipedia — Password](https://en.wikipedia.org/wiki/Password) |
| Argon2 | [Wikipedia — Argon2](https://en.wikipedia.org/wiki/Argon2) |
| Credential stuffing | [Wikipedia — Credential stuffing](https://en.wikipedia.org/wiki/Credential_stuffing) |
| Have I Been Pwned | [haveibeenpwned.com](https://haveibeenpwned.com/) |

---

## Checkpoint

**Q1. Why is per-user salting essential even when using a slow hash?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A slow hash makes each individual guess expensive, but <strong>without a unique salt</strong> an attacker
can still amortize work across many users: identical passwords produce identical hashes, so (1) precomputed
<strong>rainbow tables</strong> or a single cracking run can be matched against <em>every</em> user at once,
and (2) you can instantly see which users share a password. A <strong>unique per-user salt</strong> forces
the attacker to crack <em>each</em> hash separately — no precomputation carries over, and identical passwords
hash differently — so the slow-hash cost is paid per user rather than once for the whole database. Slowness
limits the rate per guess; salting prevents one guess from covering many accounts. You need both.
</details>

---

**Q2. What's the difference between defending against offline and online password attacks?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An <strong>offline attack</strong> assumes the attacker has stolen your password <em>database</em> and is
cracking hashes on their own hardware — defended by <strong>how you store</strong> credentials: slow,
memory-hard, salted hashing (argon2/bcrypt/scrypt) makes each guess so expensive that mass cracking is
infeasible. An <strong>online attack</strong> is the attacker guessing at your <em>live login endpoint</em>
(brute force, credential stuffing) — defended by <strong>how you respond</strong>: rate limiting, account
lockout/backoff, CAPTCHAs, screening against known-breached passwords, and MFA. They target different
positions (stolen data vs your running service), so they need different, complementary defenses; neither
alone is sufficient.
</details>

---

**Q3. Modern guidance says "length beats complexity" and discourages forced periodic rotation. Why?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Length beats complexity</strong> because password strength against guessing scales with entropy, and
a long passphrase delivers far more entropy than a short password padded with mandated symbols — while being
easier to remember. Forced complexity rules ("1 upper, 1 number, 1 symbol") produce predictable patterns
(<code>Password1!</code>) that attackers' cracking rules already anticipate, adding little real strength.
<strong>Forced periodic rotation is discouraged</strong> because it pushes users toward weak, incremental,
predictable changes (<code>Spring2024!</code> → <code>Summer2024!</code>) and password reuse/writing-down,
<em>reducing</em> security and usability; rotation should instead be triggered by evidence of compromise.
Modern guidance (NIST) therefore favors <strong>long passphrases, screening against breached passwords,
allowing password managers/paste, and rotating only on compromise</strong> — optimizing for both real
security and human behavior.
</details>

---

## Homework

Implement (in pseudocode or a small script) a secure password registration + verification flow: hashing with
argon2id + unique salt on registration, constant-time verification on login, breached-password screening,
and rate limiting. Then explain which step defends against which specific attack (offline cracking, credential
stuffing, timing, reused-breached passwords).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A correct flow: <strong>Registration</strong> — generate a unique random salt, compute
<code>argon2id(password, salt, cost params)</code>, store the salt + hash (never the password). <strong>Login</strong>
— look up the user's salt/hash, recompute argon2id over the submitted password, and compare using a
<strong>constant-time</strong> comparison; before accepting registration/changes, <strong>screen the password
against a breached-password list</strong> (e.g. HIBP k-anonymity); and apply <strong>rate limiting / lockout</strong>
to the login endpoint. Mapping steps to attacks: (1) <strong>argon2id + unique salt</strong> defends
<strong>offline cracking</strong> — a stolen DB is slow/expensive to crack and salts prevent rainbow-table/
cross-user amortization. (2) <strong>Rate limiting / lockout</strong> defends <strong>online brute force and
credential stuffing</strong> — it caps how many guesses an attacker can make against the live endpoint. (3)
<strong>Constant-time comparison</strong> defends <strong>timing attacks</strong> — it prevents leaking how
much of a hash matched via response-time differences. (4) <strong>Breached-password screening</strong> defends
specifically against <strong>credential stuffing / reused leaked passwords</strong> — it rejects passwords
already known to attackers. Layering them covers both the stored-data and live-endpoint threat positions, and
the constant-time detail closes a subtle side channel.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 25 — Multi-Factor Authentication (TOTP/HOTP) →](lesson-25-mfa){: .btn .btn-primary }
