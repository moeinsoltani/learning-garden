---
title: "Lesson 23 — Sessions, Cookies & Tokens"
nav_order: 23
parent: "Phase 4: Authentication Fundamentals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 23: Sessions, Cookies & Tokens

## Concept

HTTP is stateless — each request is independent — yet apps must "remember you're logged in" after
authentication (Lesson 22). The two dominant approaches are **server-side sessions** (the server keeps the
state, the client holds a reference) and **stateless tokens** (the client holds the signed state itself).

```
   Server-side session:  client holds SESSION_ID cookie → server looks up session store
   Stateless token:      client holds a signed TOKEN (e.g. JWT) → server verifies signature, no lookup
```

---

## How it works

**Server-side sessions.** At login the server creates a session record (in memory/DB/Redis) and sends the
client a **session ID** in a cookie. Each request includes the cookie; the server looks up the session to
recover the identity. Easy to **revoke** (delete the record), but requires a shared session store
(stateful).

**Stateless tokens.** The server issues a **signed token** (e.g. a JWT, Lesson 31) containing the claims;
the client sends it on each request and the server **verifies the signature** — no server-side lookup
needed. Scales well and is great across services, but **revocation is hard** (the token is valid until it
expires; Lesson 39).

**Cookies and their security flags.** When state rides in a cookie, the flags matter enormously:
- **`Secure`** — only sent over HTTPS (never leaks over plaintext).
- **`HttpOnly`** — not readable by JavaScript (mitigates token theft via **XSS**).
- **`SameSite`** — restricts cross-site sending (mitigates **CSRF**).

**CSRF and XSS — the two storage threats.**
- **CSRF (Cross-Site Request Forgery):** a malicious site tricks the browser into sending a request to your
  app *with the user's cookies attached* (cookies auto-send). Defended by **SameSite** cookies and CSRF
  tokens.
- **XSS (Cross-Site Scripting):** attacker-injected JavaScript runs in your page and can **steal** tokens
  it can read. `HttpOnly` keeps cookies out of JS's reach; storing tokens in `localStorage` (readable by
  JS) is more XSS-exposed.

{: .note }
> **The storage trade-off**
> Cookies with <code>HttpOnly</code>+<code>Secure</code>+<code>SameSite</code> resist XSS theft and CSRF but
> auto-send (so CSRF must be handled). Bearer tokens in <code>localStorage</code> avoid CSRF (not auto-sent)
> but are readable by JS, so XSS can steal them. There's no free lunch — pick the model and apply its
> specific defenses.

---

## Lab

```bash
# Inspect cookie security flags a site sets at login
$ curl -sI https://github.com | grep -i set-cookie
set-cookie: _gh_sess=...; Path=/; HttpOnly; Secure; SameSite=Lax
# HttpOnly (no JS access), Secure (HTTPS only), SameSite (CSRF mitigation)

# A bearer token is sent explicitly in a header (not auto-sent like a cookie → not CSRF-prone)
$ curl -s -H "Authorization: Bearer $TOKEN" https://api.github.com/user | head
```

---

## Further Reading

| Topic | Link |
|---|---|
| Session (web) | [Wikipedia — Session (computer science)](https://en.wikipedia.org/wiki/Session_(computer_science)) |
| HTTP cookie | [Wikipedia — HTTP cookie](https://en.wikipedia.org/wiki/HTTP_cookie) |
| CSRF | [Wikipedia — Cross-site request forgery](https://en.wikipedia.org/wiki/Cross-site_request_forgery) |
| XSS | [Wikipedia — Cross-site scripting](https://en.wikipedia.org/wiki/Cross-site_scripting) |

---

## Checkpoint

**Q1. What is the core difference between a server-side session and a stateless token, and the main trade-off?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With a <strong>server-side session</strong>, the server stores the session state and gives the client only a
<strong>session ID</strong> (usually in a cookie); each request triggers a lookup in the session store to
recover the identity. With a <strong>stateless token</strong> (e.g. a signed JWT), the client holds the
actual claims, and the server just <strong>verifies the signature</strong> on each request — no server-side
lookup. The main trade-off is <strong>revocation vs scalability</strong>: sessions are easy to revoke
(delete the server record) but require a shared, stateful session store; stateless tokens scale effortlessly
and work across services without a shared store, but are <strong>hard to revoke</strong> before they expire
(Lesson 39). So sessions favor control, tokens favor scale.
</details>

---

**Q2. What do the `HttpOnly`, `Secure`, and `SameSite` cookie flags each defend against?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong><code>HttpOnly</code></strong> makes the cookie inaccessible to JavaScript, so injected scripts
(<strong>XSS</strong>) can't read and steal it. <strong><code>Secure</code></strong> ensures the cookie is
only sent over HTTPS, so it never leaks over a plaintext connection (defending against network
eavesdropping/SSL-stripping). <strong><code>SameSite</code></strong> restricts whether the cookie is sent on
cross-site requests (Lax/Strict), which defends against <strong>CSRF</strong> — a malicious site can no
longer cause the browser to send your authenticated cookie along with a forged cross-site request. Together
they harden cookie-based sessions against the main browser attacks: XSS theft, plaintext leakage, and CSRF.
</details>

---

**Q3. Why does storing a bearer token in `localStorage` avoid CSRF but increase XSS risk?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>CSRF avoided:</strong> CSRF works because browsers <em>automatically attach cookies</em> to
requests to a site, so a malicious page can trigger an authenticated request without reading anything. A
token in <code>localStorage</code> is <strong>not auto-sent</strong> — the application's JavaScript must
explicitly read it and put it in an <code>Authorization</code> header — so a cross-site forged request won't
carry it, neutralizing CSRF. <strong>XSS increased:</strong> precisely because the token lives in
JavaScript-accessible storage, any <strong>XSS</strong> flaw that runs script in your origin can simply read
<code>localStorage</code> and <strong>exfiltrate the token</strong>, fully impersonating the user. A
<code>HttpOnly</code> cookie, by contrast, is invisible to JS and can't be stolen this way (though it
reintroduces CSRF, handled by SameSite/CSRF tokens). So the choice moves the risk between two attack classes
rather than eliminating it — which is why XSS prevention is critical when using token-in-JS storage.
</details>

---

## Homework

Log into a web app and inspect (browser devtools) how it maintains your session: is it a cookie or a
token-in-storage? Record the cookie flags or storage location, then identify which attacks (CSRF/XSS) the
chosen approach must specifically defend against, and what mechanism the app appears to use for that defense.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A good answer reports the concrete findings and maps them to defenses. If the app uses a <strong>cookie</strong>:
note its flags — ideally <code>HttpOnly</code> (blocks XSS token theft), <code>Secure</code> (HTTPS only),
and <code>SameSite=Lax/Strict</code> (CSRF mitigation); because cookies auto-send, the app must also defend
<strong>CSRF</strong>, which you might see via a CSRF token in forms/headers or reliance on SameSite. If the
app uses a <strong>token in <code>localStorage</code>/<code>sessionStorage</code></strong> sent via an
<code>Authorization: Bearer</code> header: it sidesteps CSRF (not auto-sent) but must rigorously prevent
<strong>XSS</strong>, since any script injection can read and steal the token — defenses to look for include
a strict <strong>Content-Security-Policy</strong>, output encoding/sanitization, and short token lifetimes.
The exercise should conclude with the trade-off: cookie-based → guard CSRF (SameSite/CSRF tokens) while
HttpOnly handles XSS theft; storage-based → guard XSS hard (CSP, sanitization) since there's no HttpOnly
protection. Either way, the chosen storage dictates which attack you must most carefully defend.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 24 — Passwords & Credential Storage →](lesson-24-passwords){: .btn .btn-primary }
