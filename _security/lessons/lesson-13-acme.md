---
title: "Lesson 13 — ACME & Let's Encrypt"
nav_order: 13
parent: "Phase 2: PKI & Certificates"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 13: ACME & Let's Encrypt

## Concept

Getting a public certificate used to be a manual, paid, error-prone chore. **ACME** (Automatic
Certificate Management Environment) — the protocol behind **Let's Encrypt** — automates the whole thing:
a client proves it controls a domain, and the CA issues a certificate, with **no human in the loop**.
This automation is what makes short-lived certificates (Lesson 10) practical and HTTPS ubiquitous.

```
   client: "I want a cert for example.com"
   CA:     "Prove you control it: serve this token / publish this DNS record"
   client:  completes the challenge
   CA:     validates → issues the certificate → client auto-renews before expiry
```

---

## How it works

**Domain control validation (the core idea).** The CA needs proof you actually control the domain — not
your identity, just *control*. ACME does this with **challenges**:
- **HTTP-01** — the CA tells you to serve a specific token at
  `http://yourdomain/.well-known/acme-challenge/<token>`; it fetches it to confirm you control the web
  server for that name.
- **DNS-01** — the CA tells you to publish a specific `TXT` record under `_acme-challenge.yourdomain`; it
  queries DNS to confirm you control the domain's DNS. (DNS-01 supports **wildcards** and works without a
  public web server.)

**The flow.** Your ACME client generates a key, creates an account with the CA, requests a cert for your
domains (submitting a CSR), completes the challenge(s), and the CA issues the cert. The client stores it
and **automatically renews** well before expiry — so you never touch it again.

**Clients.** `certbot` (the classic), `lego`, `acme.sh`, `step`, and built-in ACME in many web servers
(Caddy does it transparently). For internal use, `step-ca` (Lesson 12) speaks ACME too.

**Rate limits & staging.** Public CAs impose **rate limits** (e.g. certs per domain per week) to prevent
abuse, so you develop against a **staging environment** (which issues untrusted certs) before hitting
production limits.

{: .note }
> **Why this changed the web**
> Free + automated certificate issuance removed both the cost and the operational pain of HTTPS, which is
> why the web went from mostly-HTTP to mostly-HTTPS in a few years. It also enabled the shift to
> <strong>short-lived certs</strong>: 90-day lifetimes are only tolerable because renewal is automatic —
> ACME and short lifetimes reinforce each other (Lesson 10).

---

## Lab

```bash
# Dry-run against Let's Encrypt STAGING (won't hit prod rate limits; issues untrusted certs)
# HTTP-01 (needs port 80 reachable for the domain):
$ sudo certbot certonly --standalone --staging -d example.test --dry-run

# DNS-01 (works without a web server; supports wildcards) — manual mode shows the TXT record to add:
$ sudo certbot certonly --manual --preferred-challenges dns --staging -d '*.example.test'
# certbot prints: add TXT record _acme-challenge.example.test = <value>, then it validates.

# Inspect what an ACME client set up for renewal
$ sudo certbot certificates           # lists managed certs, expiry, and renewal config
$ systemctl list-timers | grep certbot   # the timer that auto-renews
```

---

## Further Reading

| Topic | Link |
|---|---|
| ACME | [Wikipedia — Automatic Certificate Management Environment](https://en.wikipedia.org/wiki/Automatic_Certificate_Management_Environment) |
| Let's Encrypt | [Wikipedia — Let's Encrypt](https://en.wikipedia.org/wiki/Let%27s_Encrypt) |
| certbot | [certbot.eff.org](https://certbot.eff.org/) |
| Challenge types | [letsencrypt.org — challenge types](https://letsencrypt.org/docs/challenge-types/) |

---

## Checkpoint

**Q1. How does an ACME challenge prove you control a domain without any human in the loop?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
ACME asks the requester to perform an action that <em>only someone in control of the domain could do</em>,
then the CA verifies it automatically. With <strong>HTTP-01</strong>, the CA issues a random token and the
client must serve it at a specific URL (<code>/.well-known/acme-challenge/&lt;token&gt;</code>) on the
domain; the CA fetches that URL and, if the right token is there, concludes the requester controls the web
server for that name. With <strong>DNS-01</strong>, the client must publish a specific <code>TXT</code>
record under <code>_acme-challenge.&lt;domain&gt;</code>; the CA queries DNS and, if the record matches,
concludes the requester controls the domain's DNS. Either way the proof is <strong>possession-based and
machine-checkable</strong> — no human reviews documents — which is what allows fully automated, instant,
free issuance.
</details>

---

**Q2. When would you use DNS-01 instead of HTTP-01?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Use <strong>DNS-01</strong> when: (1) you need a <strong>wildcard certificate</strong>
(<code>*.example.com</code>) — HTTP-01 can't issue wildcards, only DNS-01 can; (2) the host has <strong>no
reachable public web server / port 80</strong> (e.g. an internal service, a mail server, a box behind a
firewall) — DNS-01 only requires control of DNS, not an HTTP endpoint; (3) you want to <strong>issue
certificates centrally</strong> for many hosts without each needing to serve challenge files; or (4) port
80 is blocked. HTTP-01 is simpler when you <em>do</em> have a public web server on the exact name and don't
need wildcards. The trade-off: DNS-01 needs programmatic access to your DNS provider (an API) to publish
the TXT record automatically.
</details>

---

**Q3. Why do public ACME CAs have rate limits and staging environments, and how does that affect your workflow?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Rate limits</strong> (e.g. a cap on certificates per domain per week) exist to prevent abuse and
protect the CA's infrastructure — without them, a bug or attacker could request unlimited certificates.
The <strong>staging environment</strong> is a parallel ACME endpoint that issues <em>untrusted</em>
certificates with much looser limits, meant for testing. This affects your workflow by pushing you to
<strong>develop and debug against staging first</strong> — getting your challenge setup, automation, and
renewal working there — and only switch to production once it's reliable, so you don't exhaust the
production rate limit with failed attempts (which could leave you unable to issue a real cert for days).
In short: test on staging, go to production once it works, and design renewal to run well before expiry so
you're never racing a limit.
</details>

---

## Homework

Set up automated issuance + renewal for a domain you control (or use staging + a DNS provider you can
script). Confirm the renewal timer/cron is in place and force a renewal dry-run. Then explain how ACME
automation and the move to short-lived certificates (Lesson 10) depend on each other.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
After issuing with an ACME client you'll find a <strong>renewal mechanism already configured</strong> —
e.g. a <code>certbot</code> systemd timer (or cron) that periodically attempts renewal — and
<code>certbot renew --dry-run</code> exercises it without consuming rate limits, proving renewal works
end-to-end (re-completing the challenge and fetching a fresh cert). The dependency between ACME and
short-lived certs is mutual: <strong>short lifetimes are only tolerable because ACME makes renewal
automatic</strong> — no human could manually reissue 90-day (or shorter) certificates across a fleet, so
without automation short lifetimes would be operationally impossible. Conversely, <strong>automation makes
short lifetimes desirable</strong> because frequent, hands-off renewal means you can shrink the validity
window to limit the damage of a compromised key (Lesson 10) without any operational cost. So ACME enables
short certs, and short certs are the payoff that justifies ACME — together they replace the fragile
revocation model with "expire fast, renew automatically."
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 14 — SSL → TLS History & Versions →](lesson-14-tls-history){: .btn .btn-primary }
