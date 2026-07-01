---
title: "Lesson 35 — mTLS & Workload Identity (SPIFFE/SPIRE)"
nav_order: 35
parent: "Phase 6: Applied Security & Identity"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 35: mTLS & Workload Identity (SPIFFE/SPIRE)

## Concept

Users have passwords and passkeys — but how does *service A* prove its identity to *service B*? The old
answer was **shared secrets** (API keys, passwords in config), which sprawl, leak, and rarely rotate.
**Workload identity** gives each service a strong, **automatically-issued, short-lived cryptographic
identity** (typically an mTLS certificate, Lesson 18) instead.

```
   Old:  service A ──(shared API key in config)──► service B     (leaky, static, hard to rotate)
   New:  service A ──(mTLS cert = its identity)──► service B      (issued + rotated automatically)
```

[SPIFFE/SPIRE](https://spiffe.io/) is the standard framework for this.

---

## How it works

**The shared-secret sprawl problem.** Long-lived API keys/passwords get copied into config files, env vars,
CI systems, and images; they're hard to inventory, rarely rotated, and a single leak grants access until
someone notices. They're "something you have" that's trivially copyable.

**SPIFFE IDs and SVIDs.** [SPIFFE](https://spiffe.io/) defines a **SPIFFE ID** — a URI naming a workload, like
`spiffe://example.org/service/payments`. A workload proves it via an **SVID** (SPIFFE Verifiable Identity
Document), usually an **X.509 certificate** (or a JWT) whose SAN is the SPIFFE ID. So a service's identity is a
short-lived cert it presents over **mTLS** — Lesson 18's machinery, now with automatic identity.

**SPIRE: issuance + attestation.** **SPIRE** is the runtime that issues SVIDs. The clever part is
**attestation**: before handing a workload an identity, SPIRE *verifies what the workload actually is* — using
platform evidence (the Kubernetes pod's service account, the process's properties, the node's identity) rather
than a secret the workload presents. So identity is granted based on *what you are/where you run*, not a
copyable token — solving the bootstrapping problem (how the first credential is obtained).

**Automatic rotation.** SVIDs are **short-lived** and continuously rotated by SPIRE, so a leaked cert is
useless within minutes and there's no manual key rotation. This is the workload analogue of short-lived TLS
certs (Lesson 10) + ACME automation (Lesson 13).

**Mesh integration.** Service meshes (Istio, Linkerd) build on exactly this: each pod gets a workload-identity
cert and all service-to-service traffic is mutually authenticated and encrypted with mTLS automatically.

{: .note }
> **Identity you can't copy and don't have to rotate by hand**
> The win over API keys is twofold: identity is <strong>bound to a private key that stays on the workload</strong>
> (not a copyable bearer secret), and it's <strong>issued/rotated automatically</strong> based on attestation
> (not provisioned and forgotten). That removes both the leak-impact and the rotation-toil of shared secrets.

---

## Lab

```bash
# Concept-level: SPIFFE/SPIRE needs a deployment, but you can see the model with the identity cert.
# A workload's SVID is an X.509 cert whose SAN is a spiffe:// URI. Inspect such a cert:
$ openssl x509 -in svid.pem -noout -ext subjectAltName
X509v3 Subject Alternative Name:
    URI:spiffe://example.org/service/payments       # the workload's identity, in the SAN

# Two services then authenticate to each other with mTLS (Lesson 18) using their SVIDs;
# each verifies the peer's SVID chains to the trust domain's CA AND checks the peer's SPIFFE ID
# is one it's authorized to talk to.

# In Kubernetes with SPIRE installed, a workload fetches its SVID from the SPIRE agent socket
# (no secret in the pod spec); rotation happens transparently.
```

---

## Further Reading

| Topic | Link |
|---|---|
| SPIFFE / SPIRE | [spiffe.io](https://spiffe.io/) |
| Workload identity | [Wikipedia — Workload identity](https://en.wikipedia.org/wiki/Identity_management) |
| Mutual TLS | [Lesson 18 — mTLS](lesson-18-mtls) |
| Service mesh | networking Lesson 43 (Kubernetes networking) |

---

## Checkpoint

**Q1. How does workload identity replace long-lived API keys between services?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Instead of giving each service a static, copyable <strong>shared secret</strong> (API key/password) to
present, workload identity gives each service a strong <strong>cryptographic identity</strong> — typically a
short-lived X.509 certificate (an SVID) whose name is the workload's SPIFFE ID — that it uses to authenticate
via <strong>mTLS</strong> (Lesson 18). The identity is bound to a <strong>private key that stays on the
workload</strong> (not a bearer token that can be copied and replayed), and it's <strong>issued and rotated
automatically</strong> by an infrastructure component (e.g. SPIRE). So service A proves who it is by presenting
its cert and signing the TLS handshake, and service B verifies the cert chains to the trusted CA and that the
peer's identity is authorized — no shared secret is ever distributed, stored in config, or manually rotated.
</details>

---

**Q2. What is attestation in SPIRE, and what problem does it solve?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Attestation</strong> is how SPIRE decides <em>whether a requesting workload really is who it claims to
be</em> <strong>before issuing it an identity (SVID)</strong>. Rather than the workload presenting a secret
(which would just move the shared-secret problem), SPIRE verifies <strong>platform evidence about what the
workload actually is</strong> — e.g. the Kubernetes pod's service account and namespace, the process's
attributes, or the node's identity — and only then mints a certificate for the corresponding SPIFFE ID. This
solves the <strong>bootstrapping / "secret zero" problem</strong>: how does a fresh workload obtain its first
credential without already having a credential? Attestation grounds the very first identity in
<em>verifiable facts about the runtime environment</em> instead of a pre-shared secret, so there's no initial
copyable token to leak.
</details>

---

**Q3. Why does making workload identities short-lived and auto-rotated improve security?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because it shrinks the value and lifetime of any credential an attacker might steal, and removes error-prone
manual rotation. A <strong>short-lived, auto-rotated</strong> SVID is only valid for minutes, so even if a
workload's certificate/key is somehow exfiltrated, it becomes <strong>useless almost immediately</strong> when
it expires and is replaced — the attacker's window is tiny, and there's no need to detect-and-revoke a
long-lived secret (which, as Lesson 10 showed, is unreliable). It also eliminates the operational toil and
risk of humans rotating keys: rotation is continuous and automatic, so credentials never go stale or get
"temporarily" extended forever. This is the same principle as short-lived TLS certs + ACME automation
(Lessons 10, 13) applied to service identity: don't rely on protecting a long-lived secret — expire it fast
and reissue automatically.
</details>

---

## Homework

Compare two designs for service-to-service auth: (a) a shared API key stored in each service's config, and (b)
SPIFFE/SPIRE workload identity with mTLS. For a realistic scenario (one service is compromised and its
credentials leak), trace the blast radius and remediation effort in each design, and explain why (b) limits
damage.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>(a) Shared API key:</strong> if a service is compromised and its key leaks, the attacker holds a
<strong>long-lived, copyable credential</strong> that works from anywhere until someone manually rotates it.
Blast radius: anything that key authorizes, for as long as it's valid; if the same key is reused across
services (common), it's worse. Remediation: detect the leak (hard), generate a new key, and update <em>every</em>
config/image/CI system that used it without breakage — slow, manual, error-prone, and often incomplete.
<strong>(b) SPIFFE/SPIRE + mTLS:</strong> the leaked credential is a <strong>short-lived SVID bound to a
private key</strong>; it expires within minutes and is auto-rotated, so the stolen material is useless almost
immediately. Identity is also tied to attestation (what/where the workload is), so an attacker can't easily
mint new identities from elsewhere, and authorization can be scoped to specific SPIFFE IDs. Blast radius is
bounded to a tiny time window and a single workload's scope; remediation is mostly automatic (rotation already
handles it; you can also revoke/rotate the trust domain if needed). Design (b) limits damage because it
replaces a static, broadly-valid, copyable secret with an ephemeral, narrowly-scoped, automatically-expiring,
non-copyable identity — exactly the properties that make a leak low-impact and recovery hands-off.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 36 — Secret Management & Short-Lived Credentials →](lesson-36-secrets){: .btn .btn-primary }
