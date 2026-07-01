---
title: "Lesson 26 — WebAuthn, FIDO2 & Passkeys"
nav_order: 26
parent: "Phase 4: Authentication Fundamentals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 26: WebAuthn, FIDO2 & Passkeys

## Concept

[WebAuthn](https://en.wikipedia.org/wiki/WebAuthn) is **passwordless, phishing-resistant** authentication
built on public-key crypto (Phases 1, 4). Instead of a shared secret, your device holds a **private key**
and the site holds your **public key** — you authenticate by *signing a challenge*. Because the signature is
**bound to the site's origin**, it can't be phished or replayed elsewhere.

```
   Register: device creates a keypair for THIS site → site stores the public key
   Login:    site sends a challenge → device signs it (origin-bound) → site verifies with public key
             (no shared secret ever transmitted; nothing to steal or relay)
```

A **passkey** is the user-friendly name for a WebAuthn credential.

---

## How it works

**The ceremony (registration + assertion).** On **registration**, the authenticator (phone, laptop TPM,
security key) generates a **new key pair scoped to that site** and sends the **public key** to the server
(plus optional *attestation* about the authenticator's make/model). On **login (assertion)**, the server
sends a random **challenge**; the authenticator signs it with the private key (after a local user gesture —
biometric/PIN) and returns the signature; the server verifies it with the stored public key.

**Origin binding = phishing resistance.** This is the crucial property. The signed data **includes the
origin** (the real domain), and the browser/authenticator only releases a credential to the *exact* origin
it was created for. So a phishing site (`bank-login.evil.com`) can't get a valid signature for `bank.com` —
there's no shared code for a human to be tricked into typing, and the cryptographic response is scoped to
the legitimate site. This is what TOTP (Lesson 25) couldn't do.

**FIDO2 = WebAuthn + CTAP.** [FIDO2](https://en.wikipedia.org/wiki/FIDO2_Project) is the umbrella: WebAuthn
(the browser/server API) plus CTAP (the protocol to external authenticators like USB/NFC security keys).

**Passkeys: synced vs device-bound.** A **passkey** is a WebAuthn credential, now often **synced** across
your devices via a platform keychain (Apple/Google/Microsoft) so you don't lose it with one device —
convenient, with the trade-off that the sync provider is in the trust path. **Device-bound** credentials
(e.g. a hardware security key) never leave the device — highest assurance, but no automatic recovery.

{: .note }
> **No shared secret is the whole point**
> Passwords, TOTP secrets, and API keys are all <em>shared secrets</em> — anything the server stores can
> leak, and anything you type can be phished. WebAuthn stores only your <strong>public</strong> key (useless
> to a thief) and the private key never leaves your device, so a server breach exposes nothing usable and
> there's no secret to phish. Public-key auth removes the shared-secret problem entirely.

---

## Lab

WebAuthn needs a browser + authenticator, so this is mostly conceptual at the CLI, but you can explore it:

```bash
# List FIDO/U2F authenticators if you have a security key plugged in (libfido2 tools)
$ fido2-token -L
# /dev/hidrawN: vendor=0x..., product=0x... (FIDO2)

# Try a passkey demo in a browser: https://webauthn.io
#  - Register: the browser asks your laptop/phone/key to create a credential (biometric/PIN gesture)
#  - Authenticate: it signs a challenge; the site verifies — no password involved
# Inspect in devtools: the credential is tied to the origin (webauthn.io); it won't work on another origin.
```

The key observation: there is **no password field and no code to type** — and the credential simply does not
exist for any origin other than the one it was registered on.

---

## Further Reading

| Topic | Link |
|---|---|
| WebAuthn | [Wikipedia — WebAuthn](https://en.wikipedia.org/wiki/WebAuthn) |
| FIDO2 | [Wikipedia — FIDO2 Project](https://en.wikipedia.org/wiki/FIDO2_Project) |
| Passkeys | [passkeys.dev](https://passkeys.dev/) |
| Public-key crypto | [Lesson 3 — Asymmetric crypto](lesson-03-asymmetric-dh) |

---

## Checkpoint

**Q1. How does WebAuthn authenticate a user without any shared secret?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It uses <strong>public-key cryptography</strong> instead of a shared secret. At registration the user's
authenticator generates a <strong>key pair specific to that site</strong> and sends only the
<strong>public key</strong> to the server (the private key never leaves the device). To log in, the server
sends a random <strong>challenge</strong>, the authenticator <strong>signs it with the private key</strong>
(gated by a local biometric/PIN gesture), and the server <strong>verifies the signature with the stored
public key</strong>. Nothing secret is ever transmitted or stored on the server, so there's no password/
secret to leak in a breach or to phish — possession of the private key (proven by the signature) is the
authentication.
</details>

---

**Q2. What makes a passkey phishing-resistant where TOTP is not?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Origin binding.</strong> A WebAuthn credential is created for a specific origin (e.g.
<code>bank.com</code>), and the browser/authenticator will only use it for that <em>exact</em> origin, signing
data that <strong>includes the origin</strong>. So on a phishing site (<code>evil.com</code>) the credential
either isn't offered or produces a signature scoped to <code>evil.com</code> that the real bank rejects — and
there's <strong>no human-typed code</strong> for an attacker to capture and relay. TOTP, by contrast,
produces an <strong>origin-agnostic code</strong> the user manually enters, which a real-time phishing proxy
can capture and immediately replay to the legitimate site. Because the passkey's response is cryptographically
tied to the genuine site and never leaves the device in a relayable form, real-time phishing fails — the gap
TOTP leaves open.
</details>

---

**Q3. What is the trade-off between synced passkeys and device-bound credentials?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Synced passkeys</strong> are copied across your devices through a platform keychain (Apple/Google/
Microsoft), so you don't lose access if one device breaks and onboarding new devices is easy — the trade-off
is that the <strong>sync provider is in the trust path</strong> (your credentials' security now also depends
on that account/keychain), and the private key exists on multiple devices/cloud. <strong>Device-bound
credentials</strong> (e.g. a hardware security key or a non-synced platform credential) <strong>never leave
the device</strong>, giving the highest assurance and smallest attack surface — but there's <strong>no
automatic recovery</strong>: lose the device and that credential is gone, so you must enroll backups. So it's
<strong>convenience/recoverability vs. maximal assurance</strong>: synced for usability at consumer scale,
device-bound for high-security contexts where you accept manual backup/recovery procedures.
</details>

---

## Homework

Register a passkey on a site that supports them (or webauthn.io) using two different authenticators (e.g. your
laptop and a phone or security key). Then attempt to "use" the credential on a different origin (or reason
about why you can't). Explain how origin binding and the public/private split together eliminate both server-
breach credential theft and phishing.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Registering on two authenticators shows each creates its own site-scoped key pair (you can have multiple
passkeys per account, which also aids recovery). When you try to use the credential on a different origin, it
simply <strong>isn't available</strong> — the browser won't offer a credential created for origin A to origin
B, and any assertion is bound to the requesting origin, so it can't be transplanted. How the two properties
combine: <strong>the public/private split</strong> means the server stores only your <em>public</em> key, so a
<strong>server breach exposes nothing usable</strong> (a public key can't authenticate anyone) and there's no
shared secret to steal — defeating credential-database theft. <strong>Origin binding</strong> means the
private key only ever signs challenges for the legitimate origin and never produces a transferable code, so a
<strong>phishing site can't obtain or relay a valid response</strong> — defeating real-time phishing.
Together they remove the two great weaknesses of passwords/TOTP at once: there's nothing reusable for an
attacker to steal from the server, and nothing an attacker can trick the user into handing to the wrong site.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 27 — Kerberos →](lesson-27-kerberos){: .btn .btn-primary }
