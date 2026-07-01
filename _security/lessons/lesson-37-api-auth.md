---
title: "Lesson 37 — API Authentication Patterns"
nav_order: 37
parent: "Phase 6: Applied Security & Identity"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 37: API Authentication Patterns

## Concept

APIs need to authenticate their callers, and there are several patterns with different security/operational
trade-offs. Choosing the right one — **API keys**, **bearer tokens (OAuth/JWT)**, or **mTLS** — depends on who
the caller is (user-driven app vs backend service) and your threat model.

```
   API key       — simple shared secret, identifies an app           (easy, weak)
   Bearer token  — OAuth/OIDC access token, scoped + expiring          (standard for user-delegated)
   mTLS          — client certificate, key never sent                  (strongest for service-to-service)
```

---

## How it works

**API keys.** A static string the client sends (header/query) identifying the calling application. Simple and
ubiquitous, but it's a **long-lived bearer secret**: anyone who obtains it can use it, it's easy to leak (URLs,
logs, repos), and it usually carries coarse permissions. Acceptable for low-risk/public-data APIs or as a
rate-limiting identifier, but weak as real auth. If used, they must be **scoped, rotatable, and never in
URLs**.

**Bearer tokens (OAuth/JWT).** The standard for **user-delegated** access (Lessons 28–31): the client presents
an OAuth **access token** in the `Authorization: Bearer` header; the API validates it (signature/introspection)
and checks **scopes**. Better than API keys because tokens are **scoped and short-lived**, but they're still
**bearer** credentials — a stolen token works until it expires, so transport security (TLS) and short
lifetimes matter.

**mTLS.** For **service-to-service** APIs, mutual TLS (Lessons 18, 35) authenticates the client by a
**certificate** whose private key never leaves it — no replayable secret on the wire, identity bound to the
key. Strongest option, but needs certificate infrastructure (a CA / workload identity).

**Cross-cutting concerns — first-class, not afterthoughts.** Whatever the mechanism: enforce **least-privilege
scoping**, **rate limiting** (per key/token to limit abuse and brute force), **rotation** (keys/tokens must be
rotatable, ideally automatically), TLS everywhere, and **audit logging**. The mechanism authenticates; these
controls contain damage.

{: .note }
> **Match the credential to the caller**
> User-facing/delegated access → <strong>OAuth bearer tokens</strong> (scoped, short-lived, OIDC for
> identity). Internal service-to-service → <strong>mTLS / workload identity</strong> (non-copyable, auto-
> rotated). Plain <strong>API keys</strong> → only for low-risk cases or as identifiers, always scoped and
> rotatable. Picking by caller type is most of getting API auth right.

---

## Lab

```bash
# API key in a header (NOT a URL — URLs leak via logs/referrers/history)
$ curl -s -H "X-API-Key: $API_KEY" https://api.example.com/v1/data | head

# Bearer token (OAuth/OIDC access token), validated + scope-checked by the API
$ curl -s -H "Authorization: Bearer $ACCESS_TOKEN" https://api.example.com/v1/me | head
$ curl -s -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer $LOW_SCOPE_TOKEN" \
    https://api.example.com/v1/admin       # 403 if the token's scopes don't permit it

# mTLS to a service API: authenticate with a client certificate (no bearer secret sent)
$ curl -s --cert client.crt --key client.key --cacert ca.crt https://svc.internal/v1/op | head
```

---

## Further Reading

| Topic | Link |
|---|---|
| API key | [Wikipedia — Application programming interface key](https://en.wikipedia.org/wiki/API_key) |
| OAuth bearer tokens | [Lesson 28 — OAuth 2.0](lesson-28-oauth2) |
| mTLS | [Lesson 18 — Mutual TLS](lesson-18-mtls) |
| Rate limiting | [Wikipedia — Rate limiting](https://en.wikipedia.org/wiki/Rate_limiting) |

---

## Checkpoint

**Q1. When would you choose mTLS over a bearer token for an API?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
For <strong>service-to-service / internal APIs</strong> where you control both ends and want the strongest,
non-replayable authentication. mTLS authenticates the client by a <strong>certificate whose private key never
leaves it</strong>, so there's no bearer secret transmitted that could be captured and replayed — unlike a
bearer token, which anyone who steals it can use until it expires. mTLS is the right choice when you have (or
can run) certificate infrastructure / workload identity (Lesson 35), need mutual authentication and channel
encryption together, and want identity bound to a key rather than a copyable string. Bearer tokens remain
better for <strong>user-delegated</strong> access (browsers/mobile, where OAuth/OIDC scoping and the
authorization-code flow fit) and where deploying client certs to every caller isn't practical. Rule of thumb:
machines you control → mTLS; user-driven/third-party access → bearer tokens.
</details>

---

**Q2. Why are plain API keys considered a weak authentication mechanism, and how should they be used if at all?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because an API key is a <strong>long-lived static bearer secret</strong>: whoever possesses it can use it,
it's easily leaked (in URLs, logs, referrer headers, committed code), it's often broadly scoped, and it's
rarely rotated — so a single exposure grants lasting access and is hard to detect or contain. It also typically
identifies an <em>application</em>, not a specific authenticated user/action. If used at all, they should be
limited to <strong>low-risk or public-data APIs</strong> or as <strong>rate-limiting/identification</strong>
tokens, and always: <strong>scoped to least privilege, rotatable (ideally automatically), sent in headers not
URLs, transmitted only over TLS, and audited</strong>. For anything sensitive, prefer scoped short-lived OAuth
bearer tokens or mTLS instead.
</details>

---

**Q3. Name three cross-cutting controls that should accompany any API auth mechanism, and why.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <strong>Least-privilege scoping</strong> — each credential should grant only the specific actions/resources
needed, so a leaked or misused credential has minimal blast radius. (2) <strong>Rate limiting</strong> (per
key/token/client) — caps abuse and slows brute-force/credential-stuffing and resource-exhaustion attacks,
regardless of the auth type. (3) <strong>Rotation</strong> — credentials (keys/tokens/certs) must be
rotatable, ideally automatically and short-lived, so exposure is time-bounded and recovery doesn't require
emergency manual changes. (Also valid: <strong>TLS everywhere</strong> to protect bearer credentials in
transit, and <strong>audit logging</strong> to detect and investigate misuse.) The point: the auth mechanism
proves <em>who's calling</em>, but these controls <strong>contain the damage</strong> when something is leaked
or abused — they're essential complements, not optional extras.
</details>

---

## Homework

Design the API authentication for a system with three caller types: a public read-only API, a first-party web
app acting on behalf of users, and an internal microservice-to-microservice call. Choose a mechanism for each,
justify it, and specify the cross-cutting controls (scoping, rate limits, rotation) you'd apply.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Public read-only API:</strong> use <strong>API keys</strong> (or even anonymous + rate limiting),
since the data is low-risk and the key mainly serves identification and quota — but scope keys to read-only,
require them in headers over TLS, make them rotatable, and apply <strong>strict per-key rate limits</strong> to
prevent abuse. <strong>First-party web app on behalf of users:</strong> use <strong>OAuth/OIDC bearer
tokens</strong> via Authorization Code + PKCE (Lessons 29–30) — the app obtains scoped, short-lived
<em>access tokens</em> tied to the authenticated user, and the API validates the token and enforces
<strong>per-user scopes</strong>; apply short lifetimes + refresh, per-user/-token rate limits, and TLS.
<strong>Internal service-to-service:</strong> use <strong>mTLS with workload identity</strong> (Lessons 18,
35) — each service authenticates with an auto-rotated certificate (no copyable secret), and authorization is
based on the caller's SPIFFE ID; apply least-privilege per-service authorization and rely on automatic
rotation. <strong>Across all three:</strong> least-privilege scoping, rate limiting appropriate to each
caller, automatic/short rotation, TLS everywhere, and audit logging. The justification throughout is matching
the credential to the caller (public→key, user-delegated→bearer token, internal→mTLS) and containing damage
with the cross-cutting controls so any single leak is scoped, rate-limited, and short-lived.
</details>
