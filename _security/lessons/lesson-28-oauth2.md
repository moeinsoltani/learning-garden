---
title: "Lesson 28 — OAuth 2.0 Fundamentals"
nav_order: 28
parent: "Phase 5: Federated Identity & Authorization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 28: OAuth 2.0 Fundamentals

## Concept

[OAuth 2.0](https://en.wikipedia.org/wiki/OAuth) solves **delegated authorization**: letting one app access
your data in *another* service **without giving it your password**. When "Log in with Google" lets an app
read your calendar, that's OAuth granting the app a *limited, revocable* **access token** — not your Google
password.

```
   You ──authorize──► (Google) ──issues──► access token ──► App uses token to call Calendar API
   The app NEVER sees your Google password; the token is scoped and expirable.
```

The cardinal rule: **OAuth is authorization, not authentication** (use OIDC for identity — Lesson 30).

---

## How it works

**The four roles:**
- **Resource Owner** — *you*, the user who owns the data.
- **Client** — the application wanting access (the third-party app).
- **Authorization Server** — issues tokens after you consent (e.g. Google's OAuth server).
- **Resource Server** — holds the data and accepts tokens (e.g. the Calendar API).

**Access tokens, refresh tokens, scopes.**
- An **access token** is a short-lived credential the client sends to the resource server to make API calls.
- **Scopes** limit what the token can do (`calendar.read`, not full account access) — least privilege.
- A **refresh token** is a longer-lived credential used to get new access tokens without bothering the user
  again (Lesson 39).

**The delegation, step by step (concept).** The client redirects you to the authorization server; you
authenticate *there* (the client never sees those credentials) and consent to the requested scopes; the
authorization server issues a token bound to those scopes; the client uses it against the resource server.
You can revoke the grant later without changing your password.

**The cardinal rule — and the common mistake.** OAuth answers "*may this app do X with your data?*" not
"*who are you?*" Apps that misuse a bare OAuth access token as proof of identity ("the user has a valid
Google token, so they must be this person") create security bugs — a token leaked from another app could be
replayed. For *authentication*, use **OpenID Connect** (Lesson 30), which adds a proper identity token on
top of OAuth.

{: .note }
> **"OAuth is authorization, not authentication"**
> An access token proves the bearer was <em>authorized to access some resource</em>, not <em>who the user
> is</em>. Treating "has a valid access token" as "is logged in as user X" is the classic OAuth security
> mistake — it's why OIDC exists. Keep delegation (OAuth) and identity (OIDC) mentally separate.

---

## Lab

```bash
# Inspect a provider's OAuth/OIDC capabilities (discovery document)
$ curl -s https://accounts.google.com/.well-known/openid-configuration | python3 -m json.tool \
    | grep -E 'authorization_endpoint|token_endpoint|scopes_supported' 

# Use a token against a resource server (client-credentials example with your own provider/token)
$ curl -s -H "Authorization: Bearer $ACCESS_TOKEN" https://api.example.com/v1/me | head
# The resource server validates the token's scopes before returning data.
```

The flow itself is browser-driven (Lesson 29); here you can see the endpoints and how a token is presented.

---

## Further Reading

| Topic | Link |
|---|---|
| OAuth | [Wikipedia — OAuth](https://en.wikipedia.org/wiki/OAuth) |
| OAuth 2.0 (RFC 6749) | [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) |
| Scopes | [oauth.net — scopes](https://oauth.net/2/scope/) |
| OpenID Connect | [Lesson 30 — OIDC](lesson-30-oidc) |

---

## Checkpoint

**Q1. Who are the four parties in an OAuth exchange and what does each do?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <strong>Resource Owner</strong> — the user who owns the data and grants access. (2)
<strong>Client</strong> — the application requesting access to that data on the user's behalf. (3)
<strong>Authorization Server</strong> — the server that authenticates the user, obtains their consent, and
issues <strong>tokens</strong> (e.g. the provider's OAuth server). (4) <strong>Resource Server</strong> —
the API/service that holds the protected data and accepts valid access tokens to serve requests. The flow:
the resource owner authorizes the client at the authorization server, which issues a scoped access token
that the client presents to the resource server.
</details>

---

**Q2. What are access tokens, refresh tokens, and scopes, and how do they relate?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An <strong>access token</strong> is a short-lived credential the client presents to the resource server to
make API calls. <strong>Scopes</strong> define and limit what that access token is allowed to do (e.g.
<code>calendar.read</code> only), enforcing least privilege — the token can only perform the actions its
scopes permit. A <strong>refresh token</strong> is a longer-lived credential the client uses to obtain new
access tokens from the authorization server when the old one expires, <em>without</em> prompting the user
again. They relate as: the user consents to certain <strong>scopes</strong> → the authorization server issues
a scoped <strong>access token</strong> (short-lived) plus optionally a <strong>refresh token</strong> (to
renew access silently). Short access tokens limit damage if leaked; refresh tokens preserve usability;
scopes bound the blast radius.
</details>

---

**Q3. Why is it a security mistake to use an OAuth access token as proof of identity?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because an OAuth access token proves <strong>authorization to access a resource</strong>, not <strong>who the
user is</strong>. It's a bearer credential scoped to API access, with no guarantee about <em>which</em> user
it represents to <em>your</em> app or that it was issued <em>for</em> your app. The classic exploit: an
access token leaked from (or issued to) a <em>different</em> application could be replayed to yours, and if
you treat "has a valid token" as "is logged in as user X," an attacker impersonates that user (the "confused
deputy" / token-substitution problem). Identity requires verifying an assertion that's specifically about the
user and audience — which is what <strong>OpenID Connect</strong> adds via a signed <strong>ID token</strong>
with <code>aud</code>/<code>iss</code>/<code>sub</code> claims (Lesson 30). So use OAuth for delegated access,
and OIDC for authentication.
</details>

---

## Homework

Pick a real "Log in with X" / "Connect your account" feature and map it to the four OAuth roles: identify the
resource owner, client, authorization server, and resource server, and the scopes it requests. Then determine
whether it's using OAuth for authorization, OIDC for authentication, or (mis)using OAuth as login — and
explain how you can tell.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A good answer concretely maps the roles, e.g. for "Connect your Google Calendar to App Foo": <strong>resource
owner</strong> = you; <strong>client</strong> = App Foo; <strong>authorization server</strong> = Google's
OAuth/Accounts server (where you log in and consent); <strong>resource server</strong> = the Google Calendar
API; <strong>scopes</strong> = e.g. <code>calendar.readonly</code>. To tell whether it's authorization vs
authentication: if the app is requesting <strong>resource scopes</strong> (calendar, drive, contacts) and
then <em>calling APIs</em> with the token, it's OAuth <strong>authorization/delegation</strong>. If the
consent screen requests the <strong><code>openid</code> scope</strong> (often with <code>profile</code>/
<code>email</code>) and the app uses the result to <em>establish who you are and log you in</em>, it's
<strong>OIDC authentication</strong> — you can confirm by checking whether an <strong>ID token</strong> is
issued (a signed JWT with <code>iss</code>/<code>aud</code>/<code>sub</code>). It would be a <strong>misuse</strong>
if the app takes a bare OAuth <em>access</em> token (no <code>openid</code>/ID token) and treats possession of
it as "logged in" — recognizable by the absence of the <code>openid</code> scope/ID token while still using
the token as a login. The tell is the <code>openid</code> scope and the presence of a verifiable ID token:
present → proper authentication; absent but used as login → the classic anti-pattern.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 29 — OAuth 2.0 Flows & PKCE →](lesson-29-oauth-flows){: .btn .btn-primary }
