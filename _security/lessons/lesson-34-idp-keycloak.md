---
title: "Lesson 34 — Identity Providers in Practice (Keycloak)"
nav_order: 34
parent: "Phase 5: Federated Identity & Authorization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 34: Identity Providers in Practice (Keycloak)

## Concept

All of Phase 5's protocols (OAuth, OIDC, SAML) are implemented by an **Identity Provider (IdP)** — the central
service that authenticates users and issues tokens/assertions. Rather than each app building login, MFA, and
user management itself, they **delegate to the IdP**. [Keycloak](https://www.keycloak.org/) is a popular
open-source IdP you can run yourself to make all of this concrete.

```
   apps  →  delegate login to  →  IdP (Keycloak)
   IdP centralizes: users, passwords, MFA, OIDC/SAML, social login, sessions, tokens
```

---

## How it works

**Core Keycloak concepts:**
- **Realm** — an isolated tenant: its own users, clients, and config (e.g. one realm per environment/company).
- **Client** — an application registered with the IdP (your web app, SPA, API), configured with its protocol
  (OIDC/SAML), redirect URIs, and whether it's public or confidential.
- **Users** — the accounts the realm authenticates (with credentials, MFA enrollment, etc.).
- **Roles & groups** — authorization data assigned to users.
- **Mappers** — rules that put user attributes/roles into token **claims** (so apps receive the identity data
  they need in the ID token / access token).

**Connecting an app (OIDC).** You register a **client** in the realm, set its **redirect URI** (exact match,
Lesson 29), choose the **Authorization Code + PKCE** flow, and point the app at the realm's discovery URL
(`/.well-known/openid-configuration`). The app then does standard OIDC: redirect to Keycloak, user
authenticates, app gets and validates the ID token.

**What the IdP centralizes (and why that's the point).** Instead of every app reimplementing password storage
(Lesson 24), MFA (Lesson 25), passkeys (Lesson 26), session management (Lesson 33), and token issuance, the
IdP does it **once, correctly**, and every app delegates. Add MFA in the IdP → all apps get it. Disable a user
in the IdP → they lose access everywhere. **Federation/brokering** lets the IdP itself delegate to other IdPs
(social login, corporate SAML), so users bring existing identities.

{: .note }
> **The IdP is where the hard security lives — by design**
> Centralizing identity means the tricky, easy-to-get-wrong parts (credential storage, MFA, token signing,
> session/logout, key rotation) are implemented and audited in <em>one</em> place by people focused on it,
> instead of (badly) re-implemented in every app. Apps shrink to "redirect to IdP, validate the token." That
> consolidation is the core value — and why "don't build your own auth" is standard advice.

---

## Lab

```bash
# 1. Run Keycloak in a container (dev mode)
$ docker run -d --name kc -p 8080:8080 \
    -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin \
    quay.io/keycloak/keycloak:latest start-dev

# 2. In the admin console (http://localhost:8080): create a realm "demo", a user, and an
#    OIDC client "myapp" (Authorization Code flow, redirect URI http://localhost:3000/*).

# 3. Confirm the realm's OIDC discovery is live
$ curl -s http://localhost:8080/realms/demo/.well-known/openid-configuration \
    | python3 -c 'import sys,json;d=json.load(sys.stdin);print(d["issuer"]);print(d["authorization_endpoint"]);print(d["jwks_uri"])'

# 4. Drive a login (browser) to the authorization_endpoint with PKCE (Lesson 29),
#    exchange the code at token_endpoint, and decode the returned id_token (Lesson 31)
#    to see the claims the mappers produced (sub, email, realm roles).
```

---

## Further Reading

| Topic | Link |
|---|---|
| Keycloak | [keycloak.org](https://www.keycloak.org/) |
| Identity provider | [Wikipedia — Identity provider](https://en.wikipedia.org/wiki/Identity_provider) |
| OIDC | [Lesson 30 — OIDC](lesson-30-oidc) |
| Identity federation | [Wikipedia — Federated identity](https://en.wikipedia.org/wiki/Federated_identity) |

---

## Checkpoint

**Q1. What does an identity provider centralize that you'd otherwise rebuild in every app?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An IdP centralizes the entire <strong>authentication and identity stack</strong>: user accounts and
<strong>credential storage</strong> (secure password hashing, Lesson 24), <strong>MFA</strong> and passkeys
(Lessons 25–26), <strong>token/assertion issuance and signing</strong> for OIDC/SAML, <strong>session
management and logout</strong> (Lesson 33), key management/rotation, and often <strong>federation/social
login</strong>. Without it, every application would have to implement all of these itself — and get the hard,
security-critical parts right repeatedly. With an IdP, each app just <strong>delegates login</strong>
(redirect to the IdP, validate the returned token), so the difficult, easily-botched machinery is built and
audited once and reused everywhere. Centralizing also means policy changes (enable MFA, disable a user) apply
across all apps at once.
</details>

---

**Q2. In Keycloak terms, what are a realm, a client, and a mapper?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A <strong>realm</strong> is an isolated tenant — its own set of users, clients, roles, and configuration
(e.g. one realm per organization or environment); identities in one realm are separate from another. A
<strong>client</strong> is an application registered with the realm (a web app, SPA, mobile app, or API),
configured with its protocol (OIDC/SAML), allowed <strong>redirect URIs</strong>, flow type, and whether it's
public or confidential — it's how an app is known to the IdP. A <strong>mapper</strong> is a rule that
<strong>injects user data (attributes, roles, group memberships) into the token claims</strong> the client
receives, so the app gets exactly the identity/authorization information it needs in the ID/access token (e.g.
mapping a user's group to a <code>roles</code> claim). Together: realm = tenant boundary, client = a
registered app, mapper = shapes what claims that app's tokens carry.
</details>

---

**Q3. Why is "don't build your own auth — delegate to an IdP" common security advice?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because authentication is full of hard, easy-to-get-wrong, security-critical details — secure password
hashing, MFA, passkeys, correct OAuth/OIDC/SAML flows (PKCE, redirect-URI matching, token validation), session
and logout handling, token signing, and key rotation. Re-implementing all of that in <em>every</em>
application multiplies the chances of subtle, dangerous mistakes (the kinds of bugs seen throughout this
track), and each app becomes its own attack surface. Delegating to a mature, audited <strong>IdP</strong>
concentrates that complexity in <strong>one place</strong> maintained by specialists, so apps shrink to
"redirect to the IdP and validate the token." It also yields operational benefits — central MFA enforcement,
instant cross-app user disablement, consistent policy, and federation — that per-app auth can't easily match.
In short, you get better security and less duplicated risk by reusing a dedicated identity system.
</details>

---

## Homework

Run Keycloak, create a realm + an OIDC client + a user, and complete an Authorization Code + PKCE login
(Lesson 29), decoding the resulting ID token (Lesson 31). Then add a role to the user and a mapper that puts it
into the token, and confirm the new claim appears. Explain how this demonstrates the IdP centralizing both
authentication and the authorization data apps consume.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
After setting up the realm/client/user and completing the PKCE login, decoding the <code>id_token</code> shows
standard identity claims (<code>iss</code> = your realm, <code>aud</code> = your client, <code>sub</code>,
<code>email</code>, <code>exp</code>) — proving Keycloak performed the <strong>authentication</strong> and
issued a verifiable token. When you assign a <strong>role</strong> to the user and add a <strong>mapper</strong>
that emits it, re-running the login produces a token that now contains the new role claim (e.g. in
<code>realm_access.roles</code> or a custom claim) — without changing any application code. This demonstrates
the IdP's dual role: it <strong>centralizes authentication</strong> (it verified the user and signed the token)
<em>and</em> it <strong>centralizes the authorization data</strong> apps rely on (roles/attributes delivered as
claims via mappers). The app simply validates the token and reads the claims to make access decisions, so both
<em>who the user is</em> and <em>what they're entitled to</em> are managed in one place and propagated to every
client — exactly the consolidation that makes IdPs valuable.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 35 — mTLS & Workload Identity (SPIFFE/SPIRE) →](lesson-35-workload-identity){: .btn .btn-primary }
