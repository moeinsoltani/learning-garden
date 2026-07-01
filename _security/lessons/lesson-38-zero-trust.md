---
title: "Lesson 38 — Zero-Trust Architecture"
nav_order: 38
parent: "Phase 6: Applied Security & Identity"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 38: Zero-Trust Architecture

## Concept

The old security model was a **perimeter**: a hard firewall around the network, and everything *inside*
trusted. **Zero trust** rejects that — "**never trust, always verify**." There is no trusted inside; *every*
request must prove its identity and authorization regardless of where it comes from. This ties together the
whole track: identity (Phase 4–5), mTLS/workload identity (Phase 6), and the mesh-VPN model (networking
Phase 16).

```
   Perimeter model:  [ firewall ]  inside = trusted  → one breach = total access (lateral movement)
   Zero trust:       no trusted zone; every request authenticated + authorized per-access
```

---

## How it works

**Why the perimeter failed.** Once an attacker got *inside* (phished employee, compromised VPN, a vulnerable
internal service), the flat "trusted" network let them move **laterally** to everything. Cloud, remote work,
and microservices dissolved the perimeter entirely — there's no single "inside" anymore.

**Zero-trust principles.**
- **Verify explicitly, every time** — authenticate and authorize *each* request based on identity, not
  network location.
- **Least privilege** — grant the minimum access needed, scoped and time-bound.
- **Assume breach** — design as if attackers are already inside; segment everything to limit blast radius.

**[BeyondCorp](https://en.wikipedia.org/wiki/BeyondCorp)** (Google's model) made this concrete: access
decisions move from "are you on the corporate network?" to "**who are you (strong identity), on what device
(posture), and are you allowed this specific resource?**" — enforced at an **identity-aware proxy** in front of
each app, so being on the network grants nothing.

**The building blocks you've already learned.** Zero trust is assembled from this track: **strong
authentication** (MFA/passkeys, Phase 4), **federated identity & tokens** (OIDC/JWT, Phase 5), **mTLS /
workload identity** for services (Lesson 35), **per-request authorization** (Lesson 22), **device posture**,
and **short-lived credentials** (Lessons 10, 36). The mesh VPN of networking Phase 16 (identity-based,
ACL-enforced connectivity) is a network-level expression of the same idea.

{: .note }
> **"Being connected ≠ being allowed"**
> The one-line shift: network location stops being a credential. A request from inside the data center gets
> exactly the same scrutiny as one from a coffee shop — its identity and authorization decide access, every
> time. That's why this lesson sits at the end: zero trust is the <em>assembly</em> of everything in the
> track into one architecture.

---

## Lab

```bash
# Conceptual: contrast the two models with a simple access check.
# Perimeter style (bad): trust by source IP / network
#   if client_ip in CORP_SUBNET: allow      ← one foothold inside = full access

# Zero-trust style: verify identity + authorize the specific action, ignore location
#   token = verify_oidc(request.authorization)        # strong identity (Phase 5)
#   require(token.aud == THIS_SERVICE and token.valid)
#   require(authorized(token.sub, action, resource))  # per-request authZ (Lesson 22)
#   require(device_posture_ok(token))                 # device check (BeyondCorp)
#   # network location is NOT part of the decision

# In practice you'd see an identity-aware proxy (e.g. oauth2-proxy / a service mesh) enforce this
# in front of every app, plus mTLS between services (Lesson 35).
```

---

## Further Reading

| Topic | Link |
|---|---|
| Zero trust security model | [Wikipedia — Zero trust security model](https://en.wikipedia.org/wiki/Zero_trust_security_model) |
| BeyondCorp | [Wikipedia — BeyondCorp](https://en.wikipedia.org/wiki/BeyondCorp) |
| Mesh VPN / zero trust networking | networking Lesson 55 (Mesh VPNs) |
| Workload identity | [Lesson 35 — Workload identity](lesson-35-workload-identity) |

---

## Checkpoint

**Q1. What replaces "inside the network = trusted" in a zero-trust design?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Per-request verification of identity and authorization</strong>, independent of network location. In
zero trust, every access — whether it originates inside the data center or from the public internet — must
<strong>prove who/what is making it (strong authentication), on what device (posture), and that it's allowed
this specific resource/action (authorization)</strong>, every time. Network position grants nothing; the
decision is based on verified identity + policy + device state, typically enforced by an identity-aware proxy
in front of each application and mTLS between services. So "trusted because you're on the LAN/VPN" is replaced
by "trusted for <em>this</em> request because your identity is verified and authorized for <em>this</em>
resource right now."
</details>

---

**Q2. Why did the perimeter security model fail, and what does "assume breach" mean in response?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The <strong>perimeter model</strong> put a hard boundary around the network and trusted everything inside, so
a single breach of that boundary — a phished employee, a compromised VPN, one vulnerable internal service —
gave an attacker a <strong>trusted foothold from which to move laterally</strong> to everything else. Modern
realities (cloud, remote work, microservices, SaaS) also dissolved any clear "inside," making the perimeter
both leaky and ill-defined. <strong>"Assume breach"</strong> is the response principle: design as though
attackers are <em>already inside</em>, so you don't rely on keeping them out. In practice that means
<strong>segmenting everything, verifying every request, granting least privilege, and using short-lived
scoped credentials</strong> — so that a compromise of one identity/service yields minimal access and can't
fan out across a flat trusted network.
</details>

---

**Q3. How is zero trust assembled from the other concepts in this track?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Zero trust is essentially the <strong>integration</strong> of the track's pieces into one architecture: it
uses <strong>strong authentication</strong> (MFA/passkeys, Phase 4) to establish user identity reliably;
<strong>federated identity and tokens</strong> (OIDC/JWT, Phase 5) to carry verifiable identity/claims to each
service; <strong>per-request authorization</strong> (Lesson 22) to decide access on every call rather than
once at a perimeter; <strong>mTLS and workload identity</strong> (Lesson 35) so services authenticate each
other cryptographically rather than trusting the network; <strong>short-lived, scoped credentials</strong>
(Lessons 10, 36, 39) to limit blast radius; and <strong>device posture</strong> checks (BeyondCorp). At the
network layer, the identity-based, ACL-enforced <strong>mesh VPN</strong> (networking Phase 16) is the same
philosophy applied to connectivity. Each component answers a piece of "verify explicitly, least privilege,
assume breach," and zero trust is the architecture that wires them together so location confers no trust.
</details>

---

## Homework

Take an existing perimeter-based system (e.g. "internal apps are open to anyone on the VPN") and redesign one
service to be zero-trust. Specify: how a request is authenticated, how it's authorized per-access, what
replaces network-location trust, and which earlier-lesson mechanisms you'd use. Then describe how lateral
movement is contained if one component is compromised.

<details>
<summary>Show Answer</summary>
<br>
<strong>Redesign:</strong> put the service behind an <strong>identity-aware proxy</strong> (or service-mesh
sidecar) so no request reaches it unauthenticated. <strong>Authentication:</strong> users authenticate via
<strong>OIDC with MFA/passkeys</strong> (Phases 4–5), and the proxy validates the resulting token
(signature, <code>iss</code>/<code>aud</code>/<code>exp</code>, Lesson 30–31); services authenticate to each
other with <strong>mTLS / workload identity</strong> (Lesson 35), not by IP. <strong>Authorization
per-access:</strong> on <em>every</em> request, check the caller's identity/claims against policy for the
specific resource and action (Lesson 22), plus <strong>device posture</strong>; deny by default.
<strong>What replaces network-location trust:</strong> nothing trusts the VPN/subnet anymore — access depends
solely on verified identity + authorization + device state, so being "on the network" grants zero access.
<strong>Mechanisms reused:</strong> OIDC/JWT, MFA/passkeys, mTLS/SPIFFE, short-lived scoped tokens/credentials
(Lessons 36, 39), least-privilege scopes, rate limiting, and audit logging. <strong>Containing lateral
movement:</strong> if one component is compromised, the attacker still can't reach other services freely —
each target independently re-verifies identity and authorization and accepts only properly scoped, short-lived
credentials, services are segmented and mutually authenticated, and the stolen credentials are narrow and
expire quickly. So a single foothold yields only that component's limited scope for a short time, rather than
the whole "trusted inside," which is exactly the "assume breach + least privilege" payoff of zero trust.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 39 — Token Lifecycle & Revocation →](lesson-39-token-lifecycle){: .btn .btn-primary }
