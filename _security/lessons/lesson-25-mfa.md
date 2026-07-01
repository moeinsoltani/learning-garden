---
title: "Lesson 25 — Multi-Factor Authentication (TOTP/HOTP)"
nav_order: 25
parent: "Phase 4: Authentication Fundamentals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 25: Multi-Factor Authentication (TOTP/HOTP)

## Concept

A password is a single factor — *something you know* — and it can be guessed, phished, or leaked.
**Multi-factor authentication (MFA)** requires a second, *different kind* of factor, so stealing one isn't
enough. The classic three factor categories:

```
   Something you KNOW   — password, PIN
   Something you HAVE   — a phone/app generating codes, a hardware key
   Something you ARE    — fingerprint, face (biometric)
```

True MFA combines factors from **different categories**.

---

## How it works

**HOTP and TOTP — how authenticator apps work.** [HOTP](https://en.wikipedia.org/wiki/HMAC-based_one-time_password)
(RFC 4226) generates a one-time code from a **shared secret** + a **counter**, via HMAC. [TOTP](https://en.wikipedia.org/wiki/Time-based_one-time_password)
(RFC 6238) replaces the counter with the **current time** (in 30-second steps), so codes change every 30s.
At setup, the server and your authenticator app share a secret (the QR code you scan). Both compute
`HMAC(secret, current_time_step)` → the same 6-digit code, with no network call — the proof of "something
you have" (the secret in your app).

**Why it's a strong second factor.** An attacker who phishes or cracks your password still can't log in
without the time-based code, which requires the shared secret on your device. It defeats password-only
breaches and credential stuffing.

**The weaknesses.**
- **SMS OTP** is weak: SIM-swapping, SS7 interception, and phishing can capture texted codes — avoid SMS
  where possible; app-based TOTP is better.
- **TOTP is phishable.** A real-time phishing site can ask for your password *and* current TOTP code and
  relay both to the real site immediately. TOTP raises the bar but doesn't stop a live man-in-the-middle.
- **Push fatigue.** "Approve this login?" push prompts can be spammed until a tired user taps approve
  (MFA-bombing). Number-matching prompts mitigate this.

**The phishing-resistant answer is WebAuthn/passkeys** (Lesson 26), which bind the credential to the site's
origin so it can't be relayed.

{: .note }
> **TOTP is "better," not "phishing-proof"**
> TOTP defeats <em>offline</em> credential theft (a stolen password DB is useless without the code) and
> casual reuse, but a <em>real-time</em> phishing proxy can still capture and replay both your password and
> your current code. That residual gap is exactly what origin-bound passkeys (Lesson 26) close — keep the
> distinction clear.

---

## Lab

```bash
# Generate a TOTP secret and compute codes with oathtool (same algorithm your app uses)
$ SECRET=$(head -c20 /dev/urandom | base32)     # the shared secret (normally shown as a QR)
$ oathtool --totp -b "$SECRET"                  # current 6-digit code
$ oathtool --totp -b "$SECRET"                  # within the same 30s window → same code

# Wait past a 30s boundary and it changes:
$ sleep 31 && oathtool --totp -b "$SECRET"      # a different code (time advanced)

# HOTP (counter-based) for comparison
$ oathtool --hotp -c 1 -b "$SECRET"
$ oathtool --hotp -c 2 -b "$SECRET"             # different counter → different code
```

---

## Further Reading

| Topic | Link |
|---|---|
| Multi-factor authentication | [Wikipedia — Multi-factor authentication](https://en.wikipedia.org/wiki/Multi-factor_authentication) |
| TOTP | [Wikipedia — Time-based one-time password](https://en.wikipedia.org/wiki/Time-based_one-time_password) |
| HOTP | [Wikipedia — HMAC-based one-time password](https://en.wikipedia.org/wiki/HMAC-based_one-time_password) |
| SIM swap scam | [Wikipedia — SIM swap scam](https://en.wikipedia.org/wiki/SIM_swap_scam) |

---

## Checkpoint

**Q1. What makes something "multi-factor," and why isn't a password plus a security question MFA?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
MFA requires factors from <strong>different categories</strong>: something you <em>know</em> (password,
PIN), something you <em>have</em> (a device/app/hardware key), and something you <em>are</em> (biometric).
The point is that the factors fail independently — compromising one kind doesn't compromise another. A
<strong>password plus a security question is not MFA</strong> because both are "something you know" — they're
the <em>same</em> category, and both are guessable, phishable, or discoverable (security-question answers are
often public/researchable). Real MFA pairs categories, e.g. password (know) + TOTP code from your phone
(have), so an attacker needs to defeat two fundamentally different things.
</details>

---

**Q2. How does a TOTP authenticator app produce the same code as the server without any network connection?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
At setup, the server and the app <strong>share a secret</strong> (delivered via the QR code you scan). To
generate a code, both independently compute <strong>HMAC(shared secret, current 30-second time step)</strong>
and truncate it to a 6-digit number (TOTP, RFC 6238). Because both sides have the same secret and use the
same current time (divided into 30s windows), they arrive at the <strong>same code at the same moment</strong>
with no communication — the time is the synchronized "counter." That's why it works offline: the only shared
state is the secret plus the clock. (HOTP is the same idea but uses an incrementing counter instead of time.)
Possession of the secret on your device is the "something you have" being proven.
</details>

---

**Q3. Why is TOTP phishable, and what kind of authentication resists phishing?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
TOTP is phishable because the user manually enters a code that is <strong>not bound to the site they're
actually talking to</strong>. A real-time phishing site can present a convincing login, capture your
<em>password</em> and your <em>current TOTP code</em>, and immediately <strong>relay both to the real
service</strong> before the 30-second window expires — logging the attacker in. The code proves you have the
secret, but not that you're authenticating to the legitimate site. <strong>Phishing-resistant</strong>
authentication — <strong>WebAuthn/FIDO2 passkeys</strong> (Lesson 26) — fixes this by <strong>binding the
credential to the site's origin</strong>: the authenticator signs a challenge tied to the real domain, so a
credential created for <code>bank.com</code> simply won't produce a valid response for the attacker's
<code>bank-login.evil.com</code>, and there's no human-typed code to relay. Origin binding is what defeats
real-time phishing that TOTP can't.
</details>

---

## Homework

Set up TOTP on a real account (or simulate with `oathtool`), then walk through a real-time phishing scenario
step by step: how would an attacker defeat password+TOTP, and at exactly which step would a passkey (Lesson
26) break the attack? Explain why the difference comes down to origin binding.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Phishing password+TOTP:</strong> (1) the attacker lures the victim to <code>bank-login.evil.com</code>,
a proxy that looks like the real bank; (2) the victim enters their password — captured; (3) the site prompts
for the TOTP code, the victim reads it from their app and enters it — captured; (4) the attacker's proxy
<em>immediately relays</em> the password and still-valid code to the real <code>bank.com</code> and is logged
in (and may capture the session cookie too). TOTP didn't help because the code isn't tied to <em>which</em>
site requested it. <strong>Where a passkey breaks this:</strong> at the authentication step. A WebAuthn
passkey created for <code>bank.com</code> signs a challenge that <strong>includes the origin</strong>, and the
browser/authenticator will only use the credential for the <em>actual</em> origin in the address bar
(<code>evil.com</code>). So on the phishing site either no matching credential exists or the signed assertion
is bound to <code>evil.com</code> — which the real <code>bank.com</code> rejects. There's also no typed code
to relay. The whole difference is <strong>origin binding</strong>: TOTP produces an origin-agnostic secret a
human can be tricked into handing over, while a passkey's response is cryptographically scoped to the
legitimate site and never leaves the device in a relayable form.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 26 — WebAuthn, FIDO2 & Passkeys →](lesson-26-webauthn){: .btn .btn-primary }
