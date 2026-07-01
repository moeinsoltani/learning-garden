---
title: "Lesson 31 — JWT Deep Dive"
nav_order: 31
parent: "Phase 5: Federated Identity & Authorization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 31: JWT Deep Dive

## Concept

A [JWT](https://en.wikipedia.org/wiki/JSON_Web_Token) (JSON Web Token) is a compact, **signed** bundle of
claims — the format behind OIDC ID tokens (Lesson 30) and many access tokens. Its appeal is that it's
**self-contained and verifiable**: a service can check it with the issuer's key alone, no database lookup. Its
danger is a set of **classic validation pitfalls** that have caused real breaches.

```
   header . payload . signature        (three base64url parts, dot-separated)
   {alg,kid}.{claims: sub,iss,aud,exp}.{signature over header+payload}
```

---

## How it works

**Structure.** Three base64url-encoded parts joined by dots: **header** (algorithm `alg`, key id `kid`),
**payload** (the claims: `sub`, `iss`, `aud`, `exp`, `iat`, custom data), and **signature**. Note: base64url
is **encoding, not encryption** — the payload is *readable by anyone*. A JWT protects *integrity* (via the
signature), not *confidentiality*.

**JWS vs JWE.** The common JWT is a **JWS** (signed, contents visible). A **JWE** is *encrypted* (contents
hidden). If you put secrets in a plain JWT, remember they're visible — use JWE or just don't.

**Validation — the part everyone gets wrong.** To trust a JWT you must:
1. **Verify the signature** with the issuer's key (asymmetric: public key from JWKS; symmetric: shared
   secret).
2. **Pin the algorithm** — only accept the `alg` *you* expect; never trust the token's own `alg`.
3. Check **`iss`** (right issuer), **`aud`** (intended for you), **`exp`/`nbf`** (time validity).

**The classic vulnerabilities:**
- **`alg: none`** — some libraries accepted a token claiming "no signature," so an attacker forges any
  payload. *Always reject `none`.*
- **RS256 → HS256 confusion** — a token signed with `alg:HS256` where the verifier mistakenly uses the
  issuer's **public key as the HMAC secret**. Since the public key is, well, public, the attacker can forge a
  validly-"signed" token. *Fix: pin the expected algorithm; don't let the token choose.*
- **Missing `aud`/`iss` checks** — accepting a token meant for another service/audience (token redirection).
- **No `exp` check** — replaying expired tokens.

**Why revocation is hard (preview of Lesson 39).** A JWT is valid until `exp` purely by its signature — there's
no server state to delete. So you can't easily revoke one early without extra machinery (short lifetimes,
denylists, token introspection).

{: .note }
> **Pin the algorithm; trust nothing the token says about how to verify it**
> The root of the worst JWT bugs is letting the <em>token</em> dictate verification (its <code>alg</code>).
> The verifier must decide the expected algorithm and key out-of-band and apply <em>only</em> that. "Verify
> with what I expect, not what the attacker-supplied header claims" is the one rule that kills <code>alg:none</code>
> and RS256→HS256 confusion at once.

---

## Lab

```bash
# Decode (NOT verify) a JWT's header and payload — note they're just base64url, readable
$ echo "$JWT" | cut -d. -f1 | basenc -d --base64url 2>/dev/null; echo
{"alg":"RS256","kid":"abc","typ":"JWT"}
$ echo "$JWT" | cut -d. -f2 | basenc -d --base64url 2>/dev/null | python3 -m json.tool
{ "iss": "...", "aud": "your-client", "sub": "...", "exp": 1750000000 }

# Demonstrate the alg:none danger conceptually: a token with header {"alg":"none"} and no signature
$ printf '%s' '{"alg":"none"}' | basenc --base64url | tr -d '='
# A naive verifier that honors alg:none would accept ANY payload appended with an empty signature — reject it.

# Proper verification requires the issuer's key + pinned alg (use a real library; pyjwt example):
$ python3 -c 'import jwt,sys; print(jwt.decode(sys.argv[1], key=open("issuer.pub").read(), algorithms=["RS256"], audience="your-client"))' "$JWT"
# Note algorithms=["RS256"] (pinned) and audience set — these enforce the checks.
```

---

## Further Reading

| Topic | Link |
|---|---|
| JSON Web Token | [Wikipedia — JSON Web Token](https://en.wikipedia.org/wiki/JSON_Web_Token) |
| JWT (RFC 7519) | [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519) |
| JWT attacks | [PortSwigger — JWT attacks](https://portswigger.net/web-security/jwt) |
| Digital signatures | [Lesson 4 — Digital signatures](lesson-04-signatures) |

---

## Checkpoint

**Q1. Is the payload of a standard (JWS) JWT secret? What does the JWT actually protect?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
No — the payload is <strong>not secret</strong>. A standard JWT (a JWS) is just three <strong>base64url-
encoded</strong> parts; base64url is <em>encoding, not encryption</em>, so anyone holding the token can decode
and read the header and payload (including all claims). What a JWT protects is <strong>integrity and
authenticity</strong> via its <strong>signature</strong>: a verifier can confirm the claims were issued by the
holder of the signing key and weren't altered. It does <em>not</em> provide confidentiality. If you need the
contents hidden, you must use a <strong>JWE</strong> (encrypted JWT) or simply not put sensitive data in the
token. A common mistake is treating a signed JWT as if its contents were private.
</details>

---

**Q2. Why must a verifier pin the expected algorithm rather than trust the token's `alg` header?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because letting the <em>token</em> choose how it's verified hands control to the attacker, enabling two
classic forgeries. (1) <strong><code>alg: none</code></strong>: a token declares it has no signature, and a
verifier that honors that accepts <em>any</em> forged payload. (2) <strong>RS256→HS256 confusion</strong>: a
token is sent with <code>alg:HS256</code> (symmetric HMAC), and a naive verifier uses the issuer's RSA
<em>public</em> key as the HMAC secret — but the public key is public, so the attacker can compute a valid HMAC
and forge tokens. Both are defeated by <strong>pinning the algorithm</strong>: the verifier decides out-of-band
that it expects, say, RS256 with a specific public key, and rejects anything else — never reading the
acceptable algorithm from the attacker-controlled header. "Verify with what I expect, not what the token
claims."
</details>

---

**Q3. Name two JWT validation checks beyond the signature and the attack each prevents.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <strong><code>aud</code> (audience) check</strong> — ensures the token was issued <em>for your
service/client</em>; it prevents <strong>token redirection / confused-deputy</strong>, where a token validly
issued for a different audience is replayed against yours. (2) <strong><code>exp</code> (expiry) check</strong>
(and <code>nbf</code>) — ensures the token is currently valid; it prevents <strong>replay of expired
tokens</strong>. (Also valid: <strong><code>iss</code> check</strong> — rejects tokens from a wrong/rogue
issuer; <strong>nonce check</strong> in OIDC — prevents replay of an ID token from a different login.) The
theme: a valid signature only proves the token is authentic and untampered — you still must confirm it's from
the right issuer, intended for you, and still in its validity window, or attackers reuse legitimately-signed
tokens in the wrong context.
</details>

---

## Homework

Take a real JWT (e.g. an OIDC ID token) and (1) decode its parts to read the claims, (2) verify it properly
with a library using a pinned algorithm + the issuer's JWKS key + audience, then (3) tamper with a claim and
re-verify to watch it fail, and (4) craft an `alg:none` version and confirm your verifier rejects it. Explain
what each result demonstrates about JWT security.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Results and meaning: (1) <strong>Decoding</strong> shows the header/payload in cleartext — demonstrating a JWT
is encoded, not encrypted, so never put secrets in a JWS. (2) <strong>Proper verification</strong> (pinned
<code>algorithms=["RS256"]</code>, key from JWKS, <code>audience</code> set) succeeds for the genuine token —
demonstrating that with the issuer's public key you can validate authenticity locally, no DB lookup needed.
(3) <strong>Tampering with a claim</strong> (e.g. changing <code>sub</code> or a role) and re-verifying
<strong>fails the signature check</strong> — demonstrating integrity protection: you can't alter claims without
invalidating the signature. (4) Crafting an <strong><code>alg:none</code></strong> token (and an HS256-with-
public-key variant) and seeing the verifier <strong>reject it</strong> demonstrates the value of algorithm
pinning — the verifier refuses attacker-chosen verification methods, defeating the <code>alg:none</code> and
RS256→HS256 forgeries. Together these show JWT's security rests on (a) understanding it's signed-not-secret,
(b) verifying the signature with the correct key, (c) pinning the algorithm, and (d) checking
<code>iss</code>/<code>aud</code>/<code>exp</code> — and that skipping any of these is where real-world JWT
breaches come from.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 32 — SAML 2.0 →](lesson-32-saml){: .btn .btn-primary }
