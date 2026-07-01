---
title: "Lesson 40 — Threat Modeling Authentication Systems"
nav_order: 40
parent: "Phase 6: Applied Security & Identity"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 40: Threat Modeling Authentication Systems

## Concept

The capstone skill: instead of hoping a system is secure, **systematically reason about how it could fail**.
**Threat modeling** asks, for a given design: *what are we building, what can go wrong, and what do we do
about it?* This lesson applies that lens to authentication, pulling together every weakness seen across the
track into one disciplined process.

```
   1. What are we building?   (data-flow: actors, credentials, trust boundaries)
   2. What can go wrong?      (enumerate threats systematically — STRIDE)
   3. What do we do about it? (mitigations) → 4. Did it work? (validate)
```

---

## How it works

**STRIDE — a checklist for "what can go wrong."** [STRIDE](https://en.wikipedia.org/wiki/STRIDE_model) names
six threat categories; walk each against your auth system:
- **Spoofing** (identity) → defended by strong authentication (MFA/passkeys, mTLS).
- **Tampering** (data) → signatures/MACs, TLS integrity.
- **Repudiation** → audit logging, signatures (non-repudiation).
- **Information disclosure** → encryption, hashing secrets, minimal token contents.
- **Denial of service** → rate limiting, lockout, cookie/anti-DoS mechanisms.
- **Elevation of privilege** → authorization checks, least privilege, scope enforcement.

**Map the data flow and trust boundaries.** Draw how credentials and tokens move — user → app → IdP →
resource server — and mark **trust boundaries** (where data crosses from less- to more-trusted, e.g. the
network, the browser, a third-party). Threats concentrate at boundaries.

**Attack trees.** For a goal like "log in as another user," branch the ways to achieve it: guess/crack the
password, phish credentials, steal a session token (XSS), forge a JWT (`alg:none`), replay a 0-RTT request,
abuse password reset, intercept an SMS OTP… Each branch is a threat from earlier lessons; the tree shows your
**weakest link**.

**The track in one frame.** Every vulnerability you studied slots into this: weak password storage (Spoofing/
disclosure), missing cert validation (Spoofing/MITM), JWT `alg:none` (Tampering/Spoofing), CSRF/XSS (Spoofing/
EoP), TOTP phishing (Spoofing), token theft & revocation gaps (Spoofing). Threat modeling is how you make sure
you've considered them *all* for *your* design — and the OWASP Top 10 is a ready catalog of the common ones.

{: .note }
> **Find the weakest link, because attackers will**
> A login flow's security is the security of its <em>easiest</em> bypass, not its strongest control. MFA is
> moot if password reset emails to a hijacked inbox, or if a forgeable JWT sidesteps login entirely. Threat
> modeling's value is forcing you to enumerate <em>every</em> path and harden the weakest — which is the whole
> track's content, applied as a method.

---

## Lab

```
# Threat-model a username + password + TOTP login. Build a small attack tree for
# GOAL: "authenticate as victim", and tag each leaf with a defense from the track.

GOAL: log in as victim
├─ know the password
│   ├─ guess/brute force      → defense: rate limiting, lockout, strong hashing (L24)
│   ├─ credential stuffing    → defense: breached-password screening, MFA (L24/L25)
│   └─ phish it               → defense: passkeys / phishing-resistant MFA (L26)
├─ bypass the second factor
│   ├─ phish TOTP in real time→ defense: WebAuthn origin binding (L26)
│   ├─ SIM-swap an SMS OTP    → defense: avoid SMS; use app/passkey (L25)
│   └─ MFA-bomb push          → defense: number matching (L25)
├─ steal an active session
│   ├─ XSS reads token        → defense: HttpOnly cookies, CSP (L23)
│   └─ CSRF rides cookie      → defense: SameSite, CSRF tokens (L23)
├─ forge a token
│   └─ JWT alg:none / confusion → defense: pin algorithm, verify aud/iss/exp (L31)
└─ abuse account recovery
    └─ password-reset hijack  → defense: secure reset flow, verify identity (L24)
```

The exercise: for *your* system, every leaf must have a credible defense — an undefended leaf is your breach.

---

## Further Reading

| Topic | Link |
|---|---|
| Threat model | [Wikipedia — Threat model](https://en.wikipedia.org/wiki/Threat_model) |
| STRIDE | [Wikipedia — STRIDE model](https://en.wikipedia.org/wiki/STRIDE_model) |
| Attack tree | [Wikipedia — Attack tree](https://en.wikipedia.org/wiki/Attack_tree) |
| OWASP Top 10 | [owasp.org/Top10](https://owasp.org/www-project-top-ten/) |

---

## Checkpoint

**Q1. What three (or four) questions does threat modeling fundamentally ask?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The core questions (from the widely-used framing): (1) <strong>What are we building?</strong> — model the
system: actors, data/credential flows, and trust boundaries. (2) <strong>What can go wrong?</strong> —
systematically enumerate threats (e.g. via STRIDE or attack trees). (3) <strong>What are we going to do about
it?</strong> — decide mitigations for each identified threat. (4) <strong>Did we do a good job?</strong> —
validate that the mitigations actually address the threats and revisit as the design changes. The discipline
is doing this <em>methodically</em> rather than relying on intuition, so threats aren't missed.
</details>

---

**Q2. What does STRIDE provide, and give an auth-relevant example for two of its categories.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>STRIDE</strong> provides a <strong>checklist of six threat categories</strong> to systematically ask
"what can go wrong" — Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, and
Elevation of privilege — so you don't overlook a class of attack. Auth-relevant examples (any two): <strong>Spoofing</strong>
— an attacker impersonates a user by phishing credentials or replaying a stolen token, defended by strong/
phishing-resistant authentication (MFA, passkeys, mTLS). <strong>Elevation of privilege</strong> — an
authenticated low-privilege user accesses admin functions due to a missing authorization check (broken access
control), defended by per-request authorization and least privilege. (Others: <strong>Tampering</strong> — a
forged JWT via <code>alg:none</code>, defended by signature verification + algorithm pinning;
<strong>Information disclosure</strong> — secrets leaked because passwords were stored with a fast hash,
defended by salted slow hashing; <strong>DoS</strong> — credential-stuffing floods, defended by rate limiting;
<strong>Repudiation</strong> — no record of who did what, defended by audit logging/signatures.)
</details>

---

**Q3. Why is identifying the "weakest link" the central goal, and what does that imply about strong controls like MFA?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because an attacker doesn't fight your strongest defense — they take the <strong>easiest path in</strong>, so
the system's real security equals the strength of its <em>weakest</em> reachable bypass, not its best control.
This implies that a strong control like <strong>MFA can be rendered moot by a weak link elsewhere</strong>: if
the password-reset flow emails a link to an inbox the attacker controls, or a forgeable JWT (<code>alg:none</code>)
lets them mint a session without logging in, or an XSS steals an active session token, then the attacker simply
<em>never faces</em> the MFA prompt. Threat modeling's job is to enumerate <strong>every</strong> path to the
goal (an attack tree) and ensure each leaf has a credible defense — because hardening one control while leaving
another open just moves the attack, it doesn't stop it. Security is holistic: close the weakest link, since
that's the one that gets exploited.
</details>

---

## Homework

Build a short threat model for a username + password + TOTP login (use the lab's attack tree as a start). For
each branch, state the defense and which lesson it came from, then identify the single weakest link in a
*typical* deployment and justify why — and what you'd change to fix it.

<details>
<summary>Show Answer</summary>
<br>
A solid model maps each path to a defense: <strong>password compromise</strong> → strong salted hashing +
rate limiting/lockout + breached-password screening (L24); <strong>credential stuffing</strong> → MFA +
breach screening (L24–25); <strong>session theft</strong> → HttpOnly/Secure/SameSite cookies, CSP for XSS,
SameSite/CSRF tokens for CSRF (L23); <strong>token forgery</strong> → JWT signature verification + algorithm
pinning + aud/iss/exp checks (L31); <strong>second-factor bypass</strong> → avoid SMS (SIM-swap), use app-
based/number-matching, ideally passkeys (L25–26); <strong>recovery abuse</strong> → hardened password-reset
identity verification (L24). The <strong>single weakest link in a typical deployment is usually phishing /
real-time MITM of the password + TOTP</strong> (or an insecure account-recovery flow): TOTP is
<em>phishable</em> — a real-time proxy captures the password and current code and replays them (L25) — so the
much-touted "MFA" doesn't actually resist a determined phish, and account recovery often bypasses MFA entirely.
<strong>Fix:</strong> replace phishable TOTP with <strong>phishing-resistant WebAuthn/passkeys</strong> (origin
binding defeats real-time phishing, L26), and harden the recovery path to the same assurance level as login
(so it can't be the soft underbelly). Justification: an attacker takes the easiest route, and in a
password+TOTP system that route is phishing/recovery, not cracking — so upgrading the factor and the recovery
flow removes the weakest link rather than over-investing in already-adequate controls.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Back to the learning plan →](../learning-plan.html){: .btn .btn-primary }

*🎉 You've reached the end of this track.*
