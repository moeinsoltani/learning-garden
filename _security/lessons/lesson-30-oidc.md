---
title: "Lesson 30 — OpenID Connect (OIDC)"
nav_order: 30
parent: "Phase 5: Federated Identity & Authorization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 30: OpenID Connect (OIDC)

## Concept

OAuth (Lesson 28) is about *authorization* — it doesn't reliably tell an app *who* the user is. **OpenID
Connect (OIDC)** is a thin **identity layer on top of OAuth 2.0** that adds proper authentication: alongside
the access token, the app gets an **ID token** — a signed statement about who the user is. This is what
modern "Log in with Google/GitHub/etc." actually uses.

```
   OAuth 2.0  →  access token  (what you may DO)
   + OIDC     →  ID token      (WHO you are)   ← signed, verifiable, audience-bound
```

---

## How it works

**The ID token.** OIDC reuses the Authorization Code + PKCE flow (Lesson 29) but adds the `openid` scope; the
authorization server then also returns an **ID token** — a **JWT** (Lesson 31) signed by the provider —
containing identity **claims**: `sub` (the stable user identifier), `iss` (issuer), `aud` (the client it was
issued for), `exp` (expiry), and often `email`, `name`, etc. The app **verifies the signature and claims** to
trust "this is user `sub`, authenticated by this provider, for me."

**ID token vs access token — the key distinction.** The **ID token** is for the *client* to learn who the
user is (it's about authentication; you validate it locally). The **access token** is for calling the
*resource server's* APIs (authorization; the client shouldn't even introspect it). Sending an ID token to an
API, or using an access token as identity, are both mistakes.

**Discovery and JWKS.** OIDC standardizes **discovery**: a provider publishes its configuration at
`/.well-known/openid-configuration` (endpoints, supported scopes), and its signing keys at a **JWKS** URL.
Clients fetch the provider's **public keys** from JWKS to verify ID-token signatures — no shared secret
needed, and key rotation is automatic.

**Standard scopes/claims.** `openid` (required, signals OIDC), `profile` (name, picture…), `email`. The
**`userinfo` endpoint** returns additional claims when called with the access token.

{: .note }
> **OIDC is what makes "Log in with X" actually authentication**
> Plain OAuth gives you a token to call APIs; OIDC adds a <strong>signed, audience-bound ID token</strong>
> that's safe to treat as "who is logged in." If you're implementing login, you want OIDC — the
> <code>openid</code> scope and ID-token validation are precisely the fix for the "OAuth-as-login"
> anti-pattern from Lesson 28.

---

## Lab

```bash
# Discovery: see a provider's OIDC configuration and its JWKS (public keys) URL
$ curl -s https://accounts.google.com/.well-known/openid-configuration \
    | python3 -c 'import sys,json; d=json.load(sys.stdin); print(d["issuer"]); print(d["jwks_uri"]); print(d["userinfo_endpoint"])'

# The signing keys used to verify ID tokens (public keys, rotated by the provider)
$ curl -s "$(curl -s https://accounts.google.com/.well-known/openid-configuration | python3 -c 'import sys,json;print(json.load(sys.stdin)["jwks_uri"])')" | python3 -m json.tool | head

# After a login flow you receive an id_token (a JWT). Decode its claims (Lesson 31):
$ echo "$ID_TOKEN" | cut -d. -f2 | basenc -d --base64url 2>/dev/null | python3 -m json.tool
# Look for: iss, aud (must be YOUR client_id), sub, exp, email
```

---

## Further Reading

| Topic | Link |
|---|---|
| OpenID Connect | [Wikipedia — OpenID Connect](https://en.wikipedia.org/wiki/OpenID_Connect) |
| OIDC spec | [openid.net/connect](https://openid.net/developers/how-connect-works/) |
| JWKS / JWT | [Lesson 31 — JWT deep dive](lesson-31-jwt) |
| OAuth fundamentals | [Lesson 28 — OAuth 2.0](lesson-28-oauth2) |

---

## Checkpoint

**Q1. What does the ID token give you that an OAuth access token does not?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The <strong>ID token</strong> gives the client a <strong>verifiable statement of who the user is</strong>: a
provider-signed JWT containing identity claims like <code>sub</code> (stable user ID), <code>iss</code>
(issuer), <code>aud</code> (the client it was issued for), <code>exp</code>, and often email/name. The client
validates its signature and claims to <em>authenticate</em> the user. An <strong>access token</strong>, by
contrast, only conveys <em>authorization</em> to call a resource server's APIs — it isn't guaranteed to be
about identity, isn't necessarily meant for the client to read, and isn't audience-bound to the client. So
the ID token answers "who is logged in (and was this authentication for me)?", which the access token cannot
reliably do — that's exactly why OIDC adds it.
</details>

---

**Q2. What is the difference in intended audience/use between the ID token and the access token?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The <strong>ID token is for the client</strong>: it's the audience (its <code>aud</code> equals the client
ID), and the client validates it locally to learn the user's identity. It should <em>not</em> be sent to APIs.
The <strong>access token is for the resource server</strong>: the client treats it as an opaque credential and
presents it to the API, which validates it and checks scopes; the client generally shouldn't introspect or
depend on the access token's contents. Mixing them up causes bugs — sending an ID token to an API (which isn't
its audience) or using an access token as proof of identity (the Lesson 28 anti-pattern). Rule: ID token →
"who you are," consumed by the client; access token → "what you may do," consumed by the resource server.
</details>

---

**Q3. How does a client verify an ID token's signature, and what role does discovery/JWKS play?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The ID token is a JWT <strong>signed by the provider's private key</strong>; the client verifies it with the
provider's <strong>public key</strong>. It obtains that key via OIDC <strong>discovery</strong>: the provider
publishes its metadata at <code>/.well-known/openid-configuration</code>, which includes a
<strong>JWKS</strong> (JSON Web Key Set) URL listing the current public signing keys (each with a key ID,
<code>kid</code>). The client fetches the JWKS, picks the key matching the token header's <code>kid</code>,
and verifies the signature — then checks claims (<code>iss</code> matches the provider, <code>aud</code>
matches its own client ID, <code>exp</code> not passed, nonce if used). JWKS makes this work without any
shared secret and supports <strong>automatic key rotation</strong>: the provider can publish new keys and
clients pick them up on the next fetch, so verification keeps working as keys roll.
</details>

---

## Homework

Complete an OIDC login against a provider (or local Keycloak, Lesson 34), capture the ID token, and validate
it properly: verify the signature against the provider's JWKS, and check `iss`, `aud`, and `exp`. Then
describe a concrete attack that each of those three checks prevents.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Validation: decode the ID token, find its <code>kid</code>, fetch the matching public key from the provider's
JWKS, and verify the signature; then assert <code>iss</code> equals the expected provider, <code>aud</code>
equals your client ID, and <code>exp</code> is in the future (plus nonce if used). What each check prevents:
(1) <strong>Signature verification</strong> prevents <strong>token forgery/tampering</strong> — without it an
attacker could mint or alter an ID token (e.g. change <code>sub</code> to impersonate someone); only the
provider's private key can produce a valid signature. (2) <strong><code>iss</code> check</strong> prevents
accepting tokens from a <strong>different/rogue issuer</strong> — a token validly signed by some <em>other</em>
provider shouldn't be honored as if it came from yours. (3) <strong><code>aud</code> check</strong> prevents
<strong>token redirection/confused-deputy</strong> — a token issued for a <em>different client</em> can't be
replayed at yours; it must be addressed to you. (4) <strong><code>exp</code> check</strong> prevents
<strong>replay of expired/stale tokens</strong>. Together they ensure the token is authentic, from the right
issuer, intended for your app, and still valid — the core of safe OIDC authentication and the antidote to the
"any valid token = logged in" mistake.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 31 — JWT Deep Dive →](lesson-31-jwt){: .btn .btn-primary }
