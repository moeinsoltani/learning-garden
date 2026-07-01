---
title: "Lesson 36 — Secret Management & Short-Lived Credentials"
nav_order: 36
parent: "Phase 6: Applied Security & Identity"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 36: Secret Management & Short-Lived Credentials

## Concept

Real systems are full of secrets — database passwords, API keys, signing keys, TLS private keys. Hardcoding
them in code, config, or images is the cause of countless breaches. **Secret management** centralizes secrets
in a protected store and, better still, replaces long-lived static secrets with **short-lived dynamic** ones
that shrink the blast radius of any leak.

```
   Bad:  secret hardcoded in repo/image/env   → leaks forever, in git history, everywhere
   Good: secret stored in a vault, fetched at runtime, access-controlled, audited
   Best: dynamic short-lived secret minted on demand, auto-expiring
```

---

## How it works

**Stop hardcoding.** Secrets in source control persist in **git history** even after deletion; secrets in
images leak when images are shared; secrets in env vars leak via crash dumps and child processes. The first
step is to get secrets *out* of code and into a dedicated, access-controlled store fetched at runtime.

**Secret stores (e.g. Vault).** A secrets manager like [Vault](https://www.vaultproject.io/) provides:
encrypted storage, fine-grained **access policies** (who/what can read which secret), and an **audit log** of
every access. Applications authenticate to it (ideally via workload identity, Lesson 35) and fetch secrets at
startup/runtime rather than embedding them.

**Dynamic, short-lived secrets — the big idea.** Beyond *storing* static secrets, a manager can **generate
them on demand**: e.g. instead of one long-lived database password shared by all app instances, Vault creates
a **unique, short-lived DB credential per request/lease**, then **revokes** it when the lease expires. So a
leaked credential is valid only briefly and is tied to one consumer — dramatically smaller blast radius, with
rotation built in.

**Leasing, rotation, revocation.** Dynamic secrets come with a **lease** (TTL); they're auto-revoked at expiry
and can be revoked early if compromise is suspected. Static secrets that must remain (third-party API keys) are
**rotated** on a schedule by the manager.

**Secret-zero / bootstrapping.** The unavoidable hard problem: an app needs *some* initial credential to
authenticate to the secret store. Solving "secret zero" with another static secret just moves the problem —
the robust answer is **attestation-based identity** (Lesson 35): the platform vouches for the workload so it
can obtain a secret-store token without a pre-shared secret.

{: .note }
> **The whole arc: from static to ephemeral**
> Lessons 10, 13, 35, and this one share one theme — <strong>don't protect a long-lived secret, eliminate it</strong>.
> Short-lived TLS certs, ACME automation, rotating workload identities, and dynamic secrets all replace
> "guard this forever and hope it doesn't leak" with "issue it briefly, expire it fast, reissue automatically."
> That's the modern credential philosophy.

---

## Lab

```bash
# Demonstrate the problem: secrets persist in git history even after deletion
$ git log -p --all -S 'AKIA' 2>/dev/null | head    # search history for a leaked AWS-key-like string
# Anything ever committed is recoverable from history — why secrets must never be committed.

# Scan a repo for secrets before they leak (gitleaks / trufflehog)
$ gitleaks detect --source . --no-banner 2>/dev/null | head

# Vault (dev mode) — store and retrieve a secret with access control + audit
$ vault server -dev &        # dev server prints a root token
$ VAULT_ADDR=http://127.0.0.1:8200 vault kv put secret/db password=s3cr3t
$ VAULT_ADDR=http://127.0.0.1:8200 vault kv get secret/db
# Dynamic secrets (concept): enable a database secrets engine and `vault read database/creds/role`
# returns a FRESH, short-lived DB username/password each call, auto-revoked at lease end.
```

---

## Further Reading

| Topic | Link |
|---|---|
| Secrets management | [Wikipedia — Secrets management](https://en.wikipedia.org/wiki/Secrets_management) |
| HashiCorp Vault | [vaultproject.io](https://www.vaultproject.io/) |
| Workload identity (secret-zero) | [Lesson 35 — Workload identity](lesson-35-workload-identity) |
| gitleaks | [github.com/gitleaks/gitleaks](https://github.com/gitleaks/gitleaks) |

---

## Checkpoint

**Q1. Why are short-lived dynamic credentials safer than a long-lived static secret?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A <strong>long-lived static secret</strong> (e.g. one shared DB password) is valid indefinitely, usually
shared across many consumers, and hard to rotate — so if it leaks, the attacker has broad access for a long
time and you may not even know which copy leaked. A <strong>short-lived dynamic credential</strong> is
<strong>generated on demand, unique to the consumer, and auto-expires (and can be revoked early)</strong> after
a short lease. This shrinks the blast radius on every axis: a leak is valid only briefly (small time window),
is scoped to one consumer (easy to attribute and contain), and rotation is automatic (no manual toil or stale
secrets). So even a successful theft yields little, and recovery is largely hands-off — versus a static secret
whose leak is high-impact and painful to remediate.
</details>

---

**Q2. What concrete risks come from putting secrets in source code or container images?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
In <strong>source code</strong>, a secret persists in <strong>git history forever</strong> — deleting it in a
later commit doesn't remove it from past commits, so anyone with repo access (or a leaked clone, or a public
fork) can recover it; it also spreads to every clone, CI system, and backup. In <strong>container images</strong>,
a baked-in secret leaks whenever the image is pushed to a registry, shared, or pulled — image layers are
inspectable, so the secret travels with the artifact. Both also tend to mean the secret is <strong>static and
widely copied</strong>, so it's hard to rotate and a single exposure is high-impact. The remedy is to keep
secrets <strong>out of code/images entirely</strong>, store them in an access-controlled secrets manager, and
fetch them at runtime — ideally as short-lived dynamic credentials.
</details>

---

**Q3. What is the "secret-zero" problem, and what's the robust way to solve it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
"Secret zero" is the <strong>bootstrapping problem</strong>: to fetch secrets from a secret manager, an
application first needs <em>some</em> credential to authenticate to the manager — so what protects that
<em>initial</em> credential? Solving it with another <strong>static secret</strong> (a hardcoded token to
reach the vault) just relocates the problem — that token can leak like any other. The robust solution is
<strong>attestation-based workload identity</strong> (Lesson 35): the platform itself vouches for the workload
(via its Kubernetes service account, instance identity, process attestation, etc.), and the secret manager
grants access based on that <em>verifiable property of what/where the workload is</em> rather than a pre-shared
secret. This breaks the chicken-and-egg cycle by grounding the first credential in non-copyable runtime
evidence, so there's no static "secret zero" to protect in the first place.
</details>

---

## Homework

Take a small app that currently has a secret in its config/env and redesign it to (1) remove the secret from
code/history, (2) fetch it from a secret store at runtime, and (3) ideally use a dynamic short-lived
credential. Describe the migration steps (including cleaning git history), how the app authenticates to the
store, and how you'd handle secret-zero.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Migration steps: (1) <strong>Remove the secret and rotate it</strong> — treat any committed secret as already
compromised: rotate/revoke it immediately, delete it from config, and <strong>purge it from git history</strong>
(e.g. <code>git filter-repo</code> / BFG) plus force-push and invalidate old clones; add a secret scanner
(gitleaks) to CI to prevent recurrence. (2) <strong>Externalize</strong> — store the secret in a secrets
manager (Vault) with a fine-grained policy granting only this app read access, and have the app fetch it at
startup/runtime instead of reading a baked-in value; add audit logging. (3) <strong>Make it dynamic</strong> —
where the backend supports it (e.g. a database), switch to a <strong>dynamic secrets engine</strong> so the app
requests a fresh, short-lived credential per lease that auto-expires, rather than sharing one static password.
<strong>Authentication to the store / secret-zero:</strong> the app authenticates to Vault using
<strong>workload identity / attestation</strong> (Lesson 35) — e.g. its Kubernetes service account or
instance identity — rather than a hardcoded Vault token, so there's no static bootstrap secret to protect. The
end state: nothing secret in code or images, runtime-fetched access controlled and audited, credentials
short-lived and auto-rotated, and the initial trust grounded in attested identity — turning a permanent-leak
risk into a contained, automated system.
</details>
