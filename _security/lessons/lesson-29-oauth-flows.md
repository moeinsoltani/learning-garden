---
title: "Lesson 29 — OAuth 2.0 Flows & PKCE"
nav_order: 29
parent: "Phase 5: Federated Identity & Authorization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 29: OAuth 2.0 Flows & PKCE

## Concept

OAuth defines several **grant types** (flows) for different client situations. Modern practice has narrowed
to essentially two: **Authorization Code + PKCE** for anything a user logs into, and **Client Credentials**
for machine-to-machine. The older Implicit and Password grants are **deprecated** — knowing *why* is the
lesson.

```
   User-facing app  →  Authorization Code + PKCE   (the default for web, mobile, SPA)
   Service-to-service →  Client Credentials         (no user involved)
   Implicit / Password  →  deprecated (insecure)
```

---

## How it works

**Authorization Code flow.** The client redirects the user to the authorization server; the user
authenticates and consents; the server redirects back to the client with a short-lived **authorization
code**; the client then exchanges that code (at the token endpoint, over a back channel) for tokens. The key
property: tokens come back over a **direct server-to-server request**, not through the browser URL.

**PKCE — why it's now required.** [PKCE](https://en.wikipedia.org/wiki/OAuth#PKCE) (Proof Key for Code
Exchange) protects the code from interception. The client generates a random **code_verifier**, sends its
hash (**code_challenge**) when starting the flow, and must present the original verifier when exchanging the
code. So even if an attacker **steals the authorization code** (e.g. on a mobile device via a malicious app
intercepting the redirect), they can't exchange it without the verifier they never saw. PKCE is mandatory for
**public clients** (SPAs, mobile apps that can't keep a secret) and recommended for all.

**Client Credentials.** For **machine-to-machine** (no user): the client authenticates with its own ID +
secret (or mTLS) directly to the token endpoint and gets an access token for its own privileges. Used for
backend services calling APIs (Lesson 37).

**Why Implicit and Password grants died.**
- **Implicit** returned the access token *directly in the browser URL fragment* — exposed in history, logs,
  referrers, and to scripts. Authorization Code + PKCE replaces it (tokens never ride the front channel).
- **Resource Owner Password** had the client collect the user's actual password — defeating OAuth's whole
  point (the client should never see the password) and incompatible with MFA/SSO. Deprecated.

**Redirect URIs are security-critical.** The authorization server must **exactly match** registered redirect
URIs; loose matching (wildcards, open redirects) lets attackers steal codes/tokens by redirecting to their
own site. This is a top OAuth misconfiguration.

{: .note }
> **The modern default in one line**
> For anything a human logs into — web app, SPA, mobile — use <strong>Authorization Code flow with
> PKCE</strong>, exact redirect-URI matching, and tokens exchanged over the back channel. That single recipe
> avoids the entire class of front-channel token leaks and code-interception attacks the deprecated flows
> suffered. (OAuth 2.1 essentially codifies this.)

---

## Lab

```bash
# 1. Generate PKCE values (what a client does at the start of the flow)
$ verifier=$(head -c32 /dev/urandom | basenc --base64url | tr -d '=')
$ challenge=$(printf '%s' "$verifier" | openssl dgst -sha256 -binary | basenc --base64url | tr -d '=')
$ echo "verifier=$verifier"; echo "challenge=$challenge"

# 2. The authorization request includes the challenge (browser step):
#    https://AUTH/authorize?response_type=code&client_id=...&redirect_uri=...&scope=openid
#       &code_challenge=$challenge&code_challenge_method=S256
# 3. After consent you receive ?code=AUTH_CODE at your redirect_uri.
# 4. Exchange the code for tokens, proving possession of the verifier:
$ curl -s -X POST https://AUTH/token \
    -d grant_type=authorization_code -d code=$AUTH_CODE -d redirect_uri=$REDIRECT \
    -d client_id=$CLIENT_ID -d code_verifier=$verifier
# Without the correct code_verifier, the token endpoint rejects the exchange.
```

---

## Further Reading

| Topic | Link |
|---|---|
| Authorization Code flow | [oauth.net — auth code](https://oauth.net/2/grant-types/authorization-code/) |
| PKCE | [oauth.net — PKCE](https://oauth.net/2/pkce/) |
| Client credentials | [oauth.net — client credentials](https://oauth.net/2/grant-types/client-credentials/) |
| OAuth 2.1 | [oauth.net/2.1](https://oauth.net/2.1/) |

---

## Checkpoint

**Q1. What attack does PKCE prevent, and why is it required for public clients?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
PKCE prevents <strong>authorization-code interception</strong>. In the Authorization Code flow the code is
returned via a redirect, and on platforms like mobile a malicious app (or any party that can observe the
redirect) might <strong>steal the code</strong>. With PKCE, the client first generates a secret
<strong>code_verifier</strong> and sends only its hash (<strong>code_challenge</strong>) when starting the
flow; to exchange the code for tokens it must present the original verifier. An attacker who grabs the code
<strong>can't redeem it without the verifier</strong>, which never left the legitimate client. It's required
for <strong>public clients</strong> (SPAs, mobile apps) because they <em>can't keep a client secret</em> —
their code is shipped to users — so without PKCE a stolen code could be freely exchanged; PKCE supplies a
per-request secret to bind the code to the client that initiated it.
</details>

---

**Q2. Why were the Implicit and Resource Owner Password grants deprecated?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Implicit</strong> returned the access token <em>directly in the browser's URL fragment</em> (front
channel), where it could leak via browser history, server logs, the Referer header, or malicious scripts —
and it predated PKCE, so it was inherently exposed. It's replaced by <strong>Authorization Code + PKCE</strong>,
where tokens are exchanged over a back-channel server-to-server request and never appear in the URL.
<strong>Resource Owner Password</strong> had the client <em>collect the user's actual username and password</em>
and send them to the authorization server — which defeats OAuth's core purpose (the third-party client should
never see your password), trains users to enter credentials into apps, and is incompatible with MFA, SSO, and
federated login. Both are deprecated because they expose credentials/tokens that the modern Authorization Code
+ PKCE flow keeps protected.
</details>

---

**Q3. Why is exact redirect-URI matching security-critical in OAuth?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because the redirect URI is where the authorization server sends the <strong>authorization code (or token)</strong>
back to the client. If the server allows <strong>loose matching</strong> — wildcards, partial matches, or
combined with an open redirect on the client's domain — an attacker can craft an authorization request that
causes the code/token to be delivered to <strong>a URL they control</strong>, stealing it and then accessing
the user's account/data. Requiring the redirect URI to <strong>exactly match a pre-registered value</strong>
ensures the response can only go to the legitimate client endpoint, closing this code/token-exfiltration
vector. It's one of the most common and damaging OAuth misconfigurations, which is why providers enforce exact
matching and why you must register precise URIs (and avoid open redirects on those endpoints).
</details>

---

## Homework

Walk an Authorization Code + PKCE flow end to end against a real provider (or a local Keycloak from Lesson 34)
using the PKCE values from the lab. Capture the authorization request, the returned code, and the token
exchange. Then deliberately try the exchange with a wrong/missing code_verifier and document the failure,
explaining what that proves about PKCE's protection.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A complete answer shows: the <strong>authorization request</strong> carrying <code>code_challenge</code> +
<code>code_challenge_method=S256</code>, the redirect back with <code>?code=...</code>, and the
<strong>token exchange</strong> POST including the matching <code>code_verifier</code> succeeding (returning
access/ID tokens). Then, repeating the exchange with a <strong>wrong or missing code_verifier</strong>, the
token endpoint <strong>rejects it</strong> (typically <code>invalid_grant</code>), even though the code itself
is valid and unused. What this proves: possession of the authorization code alone is <strong>not
sufficient</strong> — the token endpoint also requires proof that the requester is the same client that began
the flow, namely knowledge of the secret verifier whose hash was registered as the challenge. So an attacker
who intercepts the code but never saw the verifier cannot complete the exchange, which is exactly PKCE's
guarantee: it binds the authorization code to the originating client, neutralizing code interception for
public clients that can't hold a long-term secret.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 30 — OpenID Connect (OIDC) →](lesson-30-oidc){: .btn .btn-primary }
