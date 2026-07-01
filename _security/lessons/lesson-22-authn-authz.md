---
title: "Lesson 22 — Authentication vs Authorization"
nav_order: 22
parent: "Phase 4: Authentication Fundamentals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 22: Authentication vs Authorization

## Concept

Two words, constantly confused, that the rest of this track depends on:
- **Authentication (AuthN)** — *who are you?* Proving identity.
- **Authorization (AuthZ)** — *what are you allowed to do?* Deciding permissions.

```
   Request arrives
      │
   AuthN: "who is this?"      → identity established (or rejected)
      │
   AuthZ: "may they do X?"    → permission granted or denied
```

You authenticate **once**; you authorize **every action**.

---

## How it works

**AuthN establishes identity.** The user/service presents **credentials** — something they know
(password), have (a key, a token, a device), or are (biometric) — and the system verifies them, producing
an authenticated **principal** (the identity for this session/request).

**AuthZ uses that identity to decide access.** Given "this is Alice," authorization answers "may Alice read
this file / call this API / delete this record?" using a policy model — roles (RBAC), attributes (ABAC),
ownership, ACLs, etc. AuthZ happens on *every* protected action, not just at login.

**Claims and principals.** After authentication, the identity is carried as a **principal** with
**claims** (assertions like "user=alice", "role=admin", "org=acme"). Authorization reads those claims to
make decisions. In federated systems (Phase 5) these claims travel in tokens (JWTs, SAML assertions).

**Where each lives in a request.** A typical web request: authenticate via a session cookie or `Authorization`
header (AuthN) → derive the principal/claims → check policy for the specific resource/action (AuthZ) →
allow or return 401 (unauthenticated) / 403 (authenticated but forbidden). The 401-vs-403 distinction *is*
the AuthN/AuthZ distinction.

{: .note }
> **401 vs 403 — the distinction in one place**
> <strong>401 Unauthorized</strong> actually means "un<em>authenticated</em>" — we don't know who you are
> (fix: log in). <strong>403 Forbidden</strong> means "authenticated, but not <em>authorized</em>" — we
> know who you are and you're not allowed (logging in again won't help). Mixing these up is the classic
> sign of conflating AuthN and AuthZ.

---

## Lab

```bash
# Observe the distinction against any API. No credentials → 401 (who are you?)
$ curl -s -o /dev/null -w "%{http_code}\n" https://api.github.com/user
401

# Authenticated but requesting something you can't access → 403 (you're known, but forbidden)
$ curl -s -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer $TOKEN" \
    https://api.github.com/repos/some/private-repo-you-cannot-see
403      # AuthN succeeded (token valid), AuthZ failed (no permission)
```

The HTTP status codes make the two concepts concrete: 401 is an authentication failure, 403 an
authorization failure.

---

## Further Reading

| Topic | Link |
|---|---|
| Authentication | [Wikipedia — Authentication](https://en.wikipedia.org/wiki/Authentication) |
| Authorization | [Wikipedia — Authorization](https://en.wikipedia.org/wiki/Authorization) |
| Access control | [Wikipedia — Access control](https://en.wikipedia.org/wiki/Access_control) |
| HTTP 401/403 | [Wikipedia — List of HTTP status codes](https://en.wikipedia.org/wiki/List_of_HTTP_status_codes) |

---

## Checkpoint

**Q1. Define authentication and authorization, and explain how often each happens in a session.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Authentication (AuthN)</strong> is proving <em>who you are</em> — verifying credentials to establish
an identity (principal). <strong>Authorization (AuthZ)</strong> is deciding <em>what that identity is allowed
to do</em> — checking permissions against a policy for a specific resource/action. In a session, you
typically <strong>authenticate once</strong> (at login, establishing the principal for the session), but you
<strong>authorize on every protected action</strong> — each request to a resource is checked against policy
using the established identity. So AuthN is the gate at the door; AuthZ is the check at every room.
</details>

---

**Q2. What do 401 and 403 HTTP status codes correspond to in AuthN/AuthZ terms?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>401 Unauthorized</strong> corresponds to an <strong>authentication</strong> failure — despite its
name it means "un<em>authenticated</em>": the server doesn't know who you are (no/invalid credentials), and
the fix is to authenticate (log in). <strong>403 Forbidden</strong> corresponds to an <strong>authorization</strong>
failure: you <em>are</em> authenticated (the server knows who you are) but you lack permission for this
resource/action — logging in again won't help because the problem is your privileges, not your identity. So
401 = "who are you?" unresolved; 403 = "we know you, and you're not allowed."
</details>

---

**Q3. After authentication, how is identity represented and used for authorization?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
After authentication, the system represents the identity as a <strong>principal</strong> carrying
<strong>claims</strong> — assertions about the identity such as user ID, roles, group memberships, or
attributes (e.g. <code>user=alice</code>, <code>role=admin</code>, <code>org=acme</code>). Authorization
then <strong>reads those claims and applies a policy</strong> (RBAC by role, ABAC by attribute, ownership/
ACL checks, etc.) to decide whether the requested action on the requested resource is permitted. In
federated systems (Phase 5) these claims are packaged into tokens — JWTs or SAML assertions — that travel
with requests so downstream services can authorize without re-authenticating. So identity established at
AuthN becomes the input (claims on a principal) that every AuthZ decision consumes.
</details>

---

## Homework

Take a system you use (e.g. a cloud console or a SaaS app) and list three operations, classifying for each
whether the relevant control is authentication or authorization, and what evidence the system uses. Then
describe a bug that would result from implementing an authorization check as if it were authentication (or
vice versa).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Example classifications: (1) "Logging in with username+password+MFA" → <strong>authentication</strong>
(evidence: credentials + second factor establishing identity). (2) "An admin can delete a user, a regular
member cannot" → <strong>authorization</strong> (evidence: the authenticated principal's role/claims checked
against policy). (3) "Accessing only your own billing data, not another tenant's" → <strong>authorization</strong>
(evidence: ownership/tenant claim compared to the requested resource). A bug from conflating them:
<strong>treating authorization as mere authentication</strong> — e.g. assuming "if the user is logged in,
they may access any record" and only checking that a valid session/token exists without checking <em>whose</em>
record it is. That yields a classic <strong>broken-access-control / IDOR</strong> vulnerability: any
authenticated user can read or modify other users' data by changing an ID, because the system authenticated
them but never authorized the specific action. Conversely, treating authentication as authorization (e.g.
granting admin powers just because someone presented <em>some</em> valid token without verifying identity/
privileges) over-grants access. The lesson: verifying <em>who</em> someone is never substitutes for checking
<em>what they're allowed to do</em> on the specific resource.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 23 — Sessions, Cookies & Tokens →](lesson-23-sessions-tokens){: .btn .btn-primary }
