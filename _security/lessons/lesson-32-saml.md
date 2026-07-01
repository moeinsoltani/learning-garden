---
title: "Lesson 32 — SAML 2.0"
nav_order: 32
parent: "Phase 5: Federated Identity & Authorization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 32: SAML 2.0

## Concept

[SAML 2.0](https://en.wikipedia.org/wiki/SAML_2.0) is the **enterprise** federated-SSO standard that
predates OIDC. It does the same job — let users authenticate once at an identity provider and access many
apps — but with **XML assertions** instead of JSON tokens, primarily for **browser-based SSO** into SaaS and
corporate apps. You'll still meet it constantly in enterprise integrations.

```
   User → App (SP) → redirected to → Identity Provider (IdP) → user authenticates →
        IdP returns a signed XML "assertion" → SP trusts it → user is logged in
```

---

## How it works

**The two parties.** The **Service Provider (SP)** is the app the user wants to use; the **Identity Provider
(IdP)** authenticates the user and vouches for them. They establish trust ahead of time by exchanging
**metadata** (entity IDs, endpoints, and the IdP's **signing certificate**).

**The assertion.** After the user authenticates at the IdP, the IdP issues a **SAML assertion** — a signed
XML document stating "this user (NameID) is authenticated, here are their attributes (email, groups), valid
until …, intended for this SP." The SP verifies the **XML signature** against the IdP's certificate and logs
the user in. (The assertion is SAML's analogue of the OIDC ID token.)

**SP-initiated vs IdP-initiated.**
- **SP-initiated:** the user starts at the app, which redirects them to the IdP to authenticate, then back
  with the assertion. (The common, recommended flow.)
- **IdP-initiated:** the user starts at the IdP's portal and clicks an app, which posts an assertion to the
  SP unsolicited. Convenient but more prone to certain attacks (no SP-side request to correlate).

**XML signing — and the wrapping attacks.** SAML's security hinges on the XML signature over the assertion.
A notorious class of bugs, **XML Signature Wrapping (XSW)**, exploited parsers that *verified* the signature
on one element but *read* a different, attacker-injected element — letting attackers forge identities. *Lesson:
use a well-tested SAML library; validate exactly what was signed.*

**SAML vs OIDC.** Same goal, different era: SAML is **XML, browser-POST, enterprise**, heavyweight; OIDC is
**JSON/JWT, REST-friendly, mobile/SPA-friendly**, lighter. New integrations favor OIDC; SAML persists across
enterprise SaaS and legacy systems, so you must know both.

{: .note }
> **Assertion ≈ ID token**
> If you understand the OIDC ID token (Lesson 30), SAML maps cleanly: the <strong>assertion</strong> is the
> signed identity statement, the <strong>IdP</strong> is the issuer, the <strong>SP</strong> is the
> audience, and <strong>metadata/certificate</strong> play the role of discovery/JWKS. Same trust model,
> XML instead of JWT.

---

## Lab

```bash
# Inspect a SAML IdP's metadata (entityID, SSO endpoint, signing certificate)
$ curl -s https://samltest.id/saml/idp | xmllint --format - 2>/dev/null \
    | grep -E 'entityID|SingleSignOnService|X509Certificate' | head

# A SAML assertion is base64'd XML in the SAMLResponse POST. Decode one you've captured:
$ echo "$SAML_RESPONSE_B64" | base64 -d | xmllint --format - 2>/dev/null | head -40
# Look for <saml:Assertion>, <saml:Subject> (NameID), <saml:Conditions> (audience/validity),
# and <ds:Signature> (the XML signature the SP must verify against the IdP cert).
```

Try the hosted test IdP/SP at `samltest.id` to walk an SP-initiated login end to end in a browser.

---

## Further Reading

| Topic | Link |
|---|---|
| SAML 2.0 | [Wikipedia — SAML 2.0](https://en.wikipedia.org/wiki/SAML_2.0) |
| SAML assertions | [Wikipedia — SAML assertions](https://en.wikipedia.org/wiki/SAML_2.0#SAML_assertions) |
| XML Signature Wrapping | [Wikipedia — XML Signature Wrapping](https://en.wikipedia.org/wiki/XML_Signature_Wrapping_attack) |
| OIDC (the modern alternative) | [Lesson 30 — OIDC](lesson-30-oidc) |

---

## Checkpoint

**Q1. Compare the SAML assertion to the OIDC ID token — what role does each play?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Both are <strong>signed statements of authenticated identity issued by the identity provider for a specific
relying party</strong>. The <strong>SAML assertion</strong> is an <strong>XML</strong> document signed (XML
Signature) by the <strong>IdP</strong>, asserting the user's NameID and attributes, with conditions
(audience, validity); the <strong>Service Provider</strong> verifies it against the IdP's certificate to log
the user in. The <strong>OIDC ID token</strong> is the same concept in <strong>JSON/JWT</strong> form, signed
by the OIDC provider, with claims (<code>sub</code>, <code>iss</code>, <code>aud</code>, <code>exp</code>),
verified by the client against the provider's JWKS key. So assertion ↔ ID token, IdP ↔ OIDC provider, SP ↔
client, SAML metadata/cert ↔ OIDC discovery/JWKS — the trust model is identical; SAML uses XML and
browser-POST, OIDC uses JWT and REST.
</details>

---

**Q2. What's the difference between SP-initiated and IdP-initiated SSO, and why is SP-initiated preferred?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
In <strong>SP-initiated</strong> SSO the user starts at the application (SP), which generates an
authentication <em>request</em> and redirects the user to the IdP; after authenticating, the IdP returns an
assertion that the SP can <strong>correlate to its original request</strong>. In <strong>IdP-initiated</strong>
SSO the user starts at the IdP portal and the IdP posts an <em>unsolicited</em> assertion to the SP with no
prior SP request. SP-initiated is <strong>preferred</strong> because the SP can tie the response to a request
it created (with state/IDs), which helps prevent <strong>replay and CSRF-style injection</strong> of
assertions; IdP-initiated has no such request to match against, making it more susceptible to assertion
injection/replay attacks. So the recommended, more secure flow is SP-initiated.
</details>

---

**Q3. What is XML Signature Wrapping, and what's the defensive lesson?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>XML Signature Wrapping (XSW)</strong> is an attack on SAML's XML signatures where the attacker
restructures the XML so that the signature still validates over the <em>original</em> (legitimate) element,
but the SP's application logic <strong>reads a different, attacker-injected element</strong> (e.g. a forged
assertion with a different identity). The mismatch between "what was signature-verified" and "what was
actually consumed" lets the attacker impersonate users despite a valid signature. The defensive lesson:
<strong>validate exactly what was signed and consume only that</strong> — use a well-tested, XSW-hardened SAML
library, ensure the signature covers the assertion you actually process, reject documents with unexpected
structure/multiple assertions, and never hand-roll XML signature verification. More broadly, it's a reason
many new integrations prefer OIDC/JWT, whose simpler signing model has fewer such parsing pitfalls.
</details>

---

## Homework

Using a test IdP/SP (e.g. samltest.id), capture a SAML assertion and dissect it: find the NameID (subject),
the audience restriction, the validity conditions, and the signature. Verify which element the signature
covers. Then explain how an XSW attack would try to subvert this and which check defeats it.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
In the decoded assertion you can identify: <code>&lt;saml:Subject&gt;&lt;saml:NameID&gt;</code> (the user
identity), <code>&lt;saml:Conditions&gt;</code> with an <code>&lt;AudienceRestriction&gt;</code> (which SP the
assertion is for) and <code>NotBefore</code>/<code>NotOnOrAfter</code> (validity window), and a
<code>&lt;ds:Signature&gt;</code> with a <code>&lt;Reference URI="#..."&gt;</code> pointing to the signed
element's ID — that URI tells you <em>exactly which element the signature covers</em>. An <strong>XSW</strong>
attack would inject a second, forged assertion (with a different NameID) while keeping the original signed
element present, arranging the document so the signature still verifies against the original but the SP's code
picks up the forged one. The check that defeats it: ensure the element the application <strong>consumes is the
very same element the signature reference covers</strong> (verify the signed reference resolves to the
assertion you actually read), reject documents containing unexpected/multiple assertions or ambiguous IDs, and
enforce the audience and validity conditions on <em>that</em> verified assertion. In practice this means
relying on a hardened SAML library that binds verification and consumption together, rather than verifying a
signature and then independently parsing the XML.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 33 — SSO & Session Management →](lesson-33-sso){: .btn .btn-primary }
