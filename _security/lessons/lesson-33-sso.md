---
title: "Lesson 33 — SSO & Session Management"
nav_order: 33
parent: "Phase 5: Federated Identity & Authorization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 33: SSO & Session Management

## Concept

**Single Sign-On (SSO)** lets a user authenticate once at an **identity provider** and access many
applications without logging in again. The subtlety is **session management**: there are now *multiple*
sessions in play — one at the IdP and one at each app — and keeping them coherent (especially **logout**) is
surprisingly hard.

```
   Login once at IdP  →  app A session, app B session, app C session
   The IdP session is the "master"; each app has its own local session.
   Logout: tearing all of them down (single logout) is the hard part.
```

---

## How it works

**Two layers of session.** SSO creates an **IdP session** (you're logged in at Google/Okta/Keycloak) and,
for each app you visit, a **local app session** (established when the app consumed your token/assertion). When
you open a new SSO app, it bounces you to the IdP, which — seeing your existing IdP session — issues a token
*without* re-prompting. That silent re-auth is what makes SSO feel seamless.

**Token lifetimes & silent renewal.** App sessions and tokens are short-lived for safety; to avoid constant
prompts, apps **silently renew** (refresh tokens, or a hidden redirect to the IdP that succeeds because the
IdP session is still valid). Balancing lifetime (security) against renewal (usability) is core session design
(Lesson 39).

**Single Logout (SLO) — the hard problem.** Logging *in* once is easy; logging *out* everywhere is not.
Killing the IdP session doesn't automatically end every app's local session — each app must be told. SLO
protocols try to propagate logout (front-channel: the browser hits each app's logout URL; back-channel: the
IdP notifies apps server-to-server), but it's fragile: an app might be missed, a browser tab closed,
back-channel calls fail silently. So a user can "log out" yet remain logged into some app — a real risk on
shared machines.

**Why SLO is harder than SSO.** SSO only needs the IdP to *vouch* once and each app to *trust* that. SLO needs
**every** app's independent session to be *actively terminated* and confirmed — a distributed teardown with no
single owner, where any failure leaves a lingering session. It's the classic "easy to fan out, hard to fan
in."

{: .note }
> **"Logout" is often weaker than users think**
> Because each app holds its own session, clicking "log out" at the IdP may not end them all. Sensitive apps
> should use <strong>short app-session lifetimes</strong> and <strong>back-channel logout</strong>, and not
> rely on SLO alone — especially on shared devices. Treat single logout as best-effort, and design sessions
> so a missed logout expires quickly.

---

## Lab

```bash
# Watch SSO's silent re-auth: log into app A via the IdP, then visit app B.
# In browser devtools (Network), app B's redirect to the IdP returns a token WITHOUT a login prompt,
# because the IdP session cookie is still valid. That's SSO.

# Inspect IdP session vs app session cookies (two different cookies/domains)
$ curl -sI https://your-idp.example/realms/main/ | grep -i set-cookie   # IdP session cookie
# vs the app's own session cookie on the app's domain after login.

# Trigger logout and observe: does hitting the IdP logout end the app session too?
# (Front-channel SLO will redirect the browser through each app's logout endpoint.)
```

---

## Further Reading

| Topic | Link |
|---|---|
| Single sign-on | [Wikipedia — Single sign-on](https://en.wikipedia.org/wiki/Single_sign-on) |
| Single logout | [Wikipedia — Single logout](https://en.wikipedia.org/wiki/Single_logout) |
| OIDC session management | [openid.net — session](https://openid.net/specs/openid-connect-session-1_0.html) |
| Token lifecycle | [Lesson 39 — Token lifecycle & revocation](lesson-39-token-lifecycle) |

---

## Checkpoint

**Q1. In SSO, what are the two layers of session, and how does visiting a second app avoid a new login?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
There's the <strong>IdP session</strong> (established when you first authenticate at the identity provider)
and a separate <strong>local app session</strong> at each application (established when that app consumes your
token/assertion). When you visit a <strong>second app</strong>, it redirects you to the IdP to authenticate;
the IdP sees your <strong>existing IdP session</strong> (via its session cookie) and therefore issues a token/
assertion <strong>without prompting you to log in again</strong>. The second app consumes that and creates its
own local session. So the IdP session acts as the "master" that lets the IdP silently vouch for you to each
new app — which is what makes SSO feel like a single login across many apps.
</details>

---

**Q2. Why is single logout (SLO) harder than single sign-on?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because logging in only requires the IdP to <em>vouch once</em> and each app to <em>trust</em> that assertion
— a simple fan-out. Logging out requires <strong>every app's independent local session to be actively
terminated</strong> and ideally confirmed — a distributed teardown with no single owner. Killing the IdP
session doesn't automatically end the apps' sessions; each app must be notified and must actually destroy its
own session, via fragile mechanisms (front-channel: the browser is bounced through each app's logout URL;
back-channel: the IdP calls each app server-to-server). Any of these can fail silently — an app is missed, a
back-channel call errors, the browser is closed mid-flow — leaving the user still logged into some apps. It's
"easy to fan out, hard to fan in": SSO needs one success, SLO needs <em>every</em> termination to succeed.
</details>

---

**Q3. Given SLO's fragility, how should a sensitive application manage sessions?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It should not rely on single logout alone. Practical measures: use <strong>short app-session and token
lifetimes</strong> so that even if a logout is missed, the session expires quickly on its own; implement
<strong>back-channel logout</strong> (server-to-server notifications from the IdP) which is more reliable than
front-channel browser redirects; <strong>re-validate the IdP session</strong> periodically (or require
step-up/re-authentication for sensitive actions); and provide a clear, working local logout that destroys the
app's own session immediately. On <strong>shared devices</strong> especially, assume "log out" may not reach
every app, so short lifetimes are the safety net. In short: treat SLO as best-effort and design sessions
(short-lived, back-channel-aware, re-validated) so a lingering session is bounded and low-risk.
</details>

---

## Homework

With two SSO apps backed by one IdP (e.g. local Keycloak from Lesson 34), log into both, then log out via the
IdP and check whether each app's session actually ends (try accessing a protected page in each afterward).
Document what happened and explain, in terms of front-channel vs back-channel logout, why one app might remain
logged in.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A typical observation: after IdP logout, one or both apps may <strong>still serve protected pages</strong>
until their local sessions expire — demonstrating that ending the IdP session doesn't automatically end app
sessions. The explanation in terms of logout channels: <strong>front-channel</strong> SLO works by redirecting
the <em>browser</em> through each app's logout endpoint, so it only succeeds if the browser actually visits
every app's logout URL in sequence — if the chain is interrupted (a tab closed, a redirect blocked, an app not
registered for SLO, or pop-up/redirect issues), some apps never get the logout signal and keep their session.
<strong>Back-channel</strong> logout instead has the IdP notify each app <em>server-to-server</em>, which
doesn't depend on the browser but can fail silently if an app isn't configured for it or the call errors. So
an app remains logged in when its logout notification (front- or back-channel) didn't reach or wasn't acted on
— exactly why SLO is best-effort and why short app-session lifetimes are the reliable backstop.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 34 — Identity Providers in Practice (Keycloak) →](lesson-34-idp-keycloak){: .btn .btn-primary }
