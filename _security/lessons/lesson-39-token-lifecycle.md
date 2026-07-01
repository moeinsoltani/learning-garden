---
title: "Lesson 39 — Token Lifecycle & Revocation"
nav_order: 39
parent: "Phase 6: Applied Security & Identity"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 39: Token Lifecycle & Revocation

## Concept

Tokens (Phase 5) aren't issued and forgotten — they have a **lifecycle**: issued, used, renewed, and
eventually invalidated. The central tension is **access-token lifetime**: short tokens are safer (a leak
expires fast) but require frequent renewal; long tokens are convenient but dangerous and **hard to revoke**.
Managing this well is what keeps a token-based system both usable and secure.

```
   issue access token (short)  +  refresh token (longer)
        │ access token expires → use refresh token to get a new one
        │ refresh token rotated each use; revoke on logout/compromise
   revocation: easy for refresh tokens (server state) — hard for stateless JWT access tokens
```

---

## How it works

**Access vs refresh tokens.** The **access token** is short-lived (minutes) and used on every API call —
short so a leak is quickly useless. The **refresh token** is longer-lived and used only to obtain new access
tokens, so the user isn't re-prompted; it's higher-value and more carefully protected.

**Refresh token rotation.** Best practice: each time a refresh token is used, issue a **new** one and
invalidate the old. If a stolen refresh token is then reused, the server sees an **already-used (replayed)
token** and can revoke the whole chain — turning theft into a detectable event.

**The stateless-JWT revocation problem.** A self-contained signed **JWT access token** is valid until `exp`
purely by its signature — there's **no server record to delete**, so you can't easily revoke it early. This is
the cost of stateless tokens (Lesson 23/31). Workarounds:
- **Short lifetimes** — the primary answer; a 5-minute token barely needs revocation.
- **Denylist** — keep a (small, short-TTL) list of revoked token IDs the API checks — reintroduces some
  state.
- **Introspection / reference tokens** — make the access token an opaque handle the API validates against the
  authorization server each time (or cached briefly), so revocation is immediate but requires a lookup.

**Detecting and responding to theft.** Signals like refresh-token reuse, impossible-travel, or anomalous usage
should trigger **revocation** (kill the session/refresh chain) and re-authentication. The combination of short
access tokens + rotating refresh tokens makes both the damage window and the detection story manageable.

{: .note }
> **The trade-off restated**
> Stateless JWTs scale beautifully (no lookup) but can't be revoked before <code>exp</code>; stateful/reference
> tokens revoke instantly but need a lookup. The pragmatic middle ground almost everyone uses:
> <strong>short-lived stateless access tokens + revocable rotating refresh tokens</strong> — you get scale on
> the hot path and control where it matters.

---

## Lab

```bash
# Decode an access token's exp to see its (short) lifetime
$ echo "$ACCESS_TOKEN" | cut -d. -f2 | basenc -d --base64url 2>/dev/null | python3 -c \
  'import sys,json,time; d=json.load(sys.stdin); print("expires in", d["exp"]-int(time.time()), "s")'

# Use a refresh token to obtain a NEW access token (and, with rotation, a new refresh token)
$ curl -s -X POST https://AUTH/token \
    -d grant_type=refresh_token -d refresh_token=$REFRESH -d client_id=$CLIENT_ID \
    | python3 -m json.tool | grep -E 'access_token|refresh_token|expires_in'

# Revoke a token (RFC 7009) — works for the refresh token / session
$ curl -s -X POST https://AUTH/revoke -d token=$REFRESH -d client_id=$CLIENT_ID
# After revocation, reusing $REFRESH fails — but an already-issued stateless access token
# remains valid until its (short) exp, illustrating the revocation gap.
```

---

## Further Reading

| Topic | Link |
|---|---|
| OAuth token revocation (RFC 7009) | [RFC 7009](https://datatracker.ietf.org/doc/html/rfc7009) |
| Refresh token rotation | [oauth.net — refresh tokens](https://oauth.net/2/grant-types/refresh-token/) |
| Token introspection (RFC 7662) | [RFC 7662](https://datatracker.ietf.org/doc/html/rfc7662) |
| JWT | [Lesson 31 — JWT deep dive](lesson-31-jwt) |

---

## Checkpoint

**Q1. Why is revoking a stateless JWT access token hard, and what are the workarounds?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A stateless JWT is <strong>self-contained and validated purely by its signature and <code>exp</code></strong>
— the server keeps no record of it, so there's nothing to delete to invalidate it early; any holder can use it
until it expires. Workarounds: (1) <strong>Short lifetimes</strong> — the primary answer; if access tokens
live only a few minutes, the inability to revoke barely matters. (2) <strong>Denylist/blocklist</strong> — keep
a (small, short-TTL) list of revoked token IDs (<code>jti</code>) that APIs check, which reintroduces some
server state. (3) <strong>Introspection / reference tokens</strong> — make the access token an opaque handle
the API validates against the authorization server on each use (or with brief caching), so revocation is
immediate but requires a lookup, trading away pure statelessness. Most systems combine short-lived stateless
access tokens with revocable refresh tokens to balance scale and control.
</details>

---

**Q2. What is refresh-token rotation, and how does it help detect theft?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Refresh-token rotation</strong> means that every time a refresh token is exchanged for a new access
token, the server also issues a <strong>new refresh token and invalidates the old one</strong> — so each
refresh token is single-use. This helps detect theft because if an attacker steals a refresh token and uses
it, then the legitimate client later uses <em>its</em> copy (or vice versa), the server sees a
<strong>previously-used/replayed refresh token</strong> — an anomaly that shouldn't happen with single-use
rotation. The server can treat that as evidence of compromise and <strong>revoke the entire token chain/
session</strong>, forcing re-authentication. So rotation converts silent refresh-token theft into a detectable,
containable event, rather than letting a stolen long-lived refresh token grant indefinite access.
</details>

---

**Q3. What's the practical compromise between stateless and stateful tokens that most systems adopt?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Most systems use <strong>short-lived stateless (JWT) access tokens plus longer-lived, revocable, rotating
refresh tokens</strong>. The <strong>access token</strong> is a stateless JWT so the hot path — validating it
on every API call — needs no database lookup and scales effortlessly; making it short-lived (minutes) keeps
the un-revocability acceptable since a leak expires almost immediately. The <strong>refresh token</strong> is
<strong>stateful and revocable</strong> (the server tracks it), so logout, compromise, or detected reuse can
immediately cut off the ability to mint new access tokens, and rotation makes theft detectable. This gives the
best of both: stateless scalability where requests are frequent, and stateful control where revocation
actually matters — bounded by the short access-token lifetime so the revocation gap is small.
</details>

---

## Homework

Design the token lifecycle for a web app: choose access- and refresh-token lifetimes, decide stateless vs
reference access tokens, and specify your revocation strategy (logout, compromise, refresh-token reuse). Then
walk through what happens, step by step, when a user's refresh token is stolen — and how your design limits the
damage.

<details>
<summary>Show Answer</summary>
<br>
<strong>Design:</strong> short-lived <strong>access tokens</strong> (e.g. 5–15 min, stateless JWT for scale),
longer-lived <strong>refresh tokens</strong> (e.g. hours–days) that are <strong>rotated on every use</strong>
and stored server-side so they're revocable; optionally a short-TTL <strong>denylist</strong> (by
<code>jti</code>) for emergency access-token revocation, or reference tokens for the most sensitive endpoints.
<strong>Revocation strategy:</strong> on <strong>logout</strong>, revoke the refresh token / session; on
<strong>suspected compromise</strong>, revoke the whole session chain and force re-auth; on <strong>refresh-
token reuse</strong> (a single-use token presented twice), treat it as theft and revoke the entire chain.
<strong>Stolen refresh token, step by step:</strong> (1) the attacker uses the stolen refresh token to get an
access token and a <em>new</em> refresh token (rotation invalidates the old one). (2) When the
<strong>legitimate</strong> client next tries to refresh with its now-invalidated copy, the server detects a
<strong>reused/old refresh token</strong> — an impossible state under rotation. (3) The server
<strong>revokes the whole refresh-token chain/session</strong>, so both the attacker's and the user's refresh
tokens stop working, and the user must re-authenticate. <strong>Damage is limited</strong> because: the
attacker's access token expires within minutes (short lifetime), the reuse is <em>detected</em> the moment
either party refreshes (rotation), revocation cuts off further access immediately, and re-authentication (with
MFA, Phase 4) re-establishes legitimate control. So a stolen refresh token yields at most a brief window and a
detectable, automatically-contained incident rather than persistent silent access.
</details>
