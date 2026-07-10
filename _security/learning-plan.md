---
title: Learning Plan
nav_order: 2
nav_exclude: false
---

# Security & Identity Learning Plan

A bottom-up curriculum: the cryptographic primitives first, then the trust
infrastructure (PKI), then the protocols that use it (TLS), then how systems prove
*who* a user or service is (authentication), then how that identity is federated and
delegated (OAuth/OIDC/SAML), and finally how it all comes together in real systems.

Labs run in your Linux VM / WSL2 using `openssl`, `curl`, a local CA (`step-ca`), and
a containerized identity provider (Keycloak).

{: .note }
> Before Lesson 1, prepare your machine with the [Lab Setup]({{ '/lab-setup.html' | relative_url }})
> page. Work the lessons in order — each builds on the ones before it.

---

## [Phase 1: Cryptography Foundations](lessons/phase-01-crypto-foundations.html)

### [Lesson 1: Symmetric encryption & AEAD](lessons/lesson-01-symmetric-aead.html)

**Goal:** Understand how a shared secret protects data, and why modern crypto is "authenticated."

**Topics:**
- Block vs stream ciphers; AES and ChaCha20
- Modes of operation; why ECB is broken; the role of the IV/nonce
- AEAD (AES-GCM, ChaCha20-Poly1305): confidentiality *and* integrity together
- Key sizes and what "128-bit security" means

**Lab:** Encrypt/decrypt a file with `openssl enc` (AES-GCM); show that a flipped byte is detected.

**Checkpoint:** Why is encryption without authentication dangerous, and how does AEAD fix it?

---

### [Lesson 2: Hashing, MACs & password storage](lessons/lesson-02-hashing-macs.html)

**Goal:** Tell apart the three things people lump together as "hashing."

**Topics:**
- Cryptographic hashes (SHA-256/512, SHA-3); properties: preimage, collision resistance
- MACs and HMAC: integrity *with* a key
- Why fast hashes are wrong for passwords; bcrypt, scrypt, argon2; salts & peppers
- Length-extension attacks (why HMAC exists)

**Lab:** Compute SHA-256 and HMAC with `openssl`; hash a password with `argon2`.

**Checkpoint:** Why is SHA-256 a bad choice for storing passwords, and what should you use instead?

---

### [Lesson 3: Asymmetric crypto & Diffie-Hellman](lessons/lesson-03-asymmetric-dh.html)

**Goal:** Understand public/private keys and how two strangers agree on a secret.

**Topics:**
- RSA vs elliptic-curve (ECDSA/Ed25519, X25519); key-size equivalences
- Public-key encryption vs key exchange
- Diffie-Hellman and ECDHE; the discrete-log problem
- Why hybrid encryption (asymmetric to exchange a symmetric key) is universal

**Lab:** Generate an RSA and an Ed25519 keypair with `openssl`; inspect them.

**Checkpoint:** How can two parties derive a shared secret over a wire an eavesdropper is reading?

---

### [Lesson 4: Digital signatures](lessons/lesson-04-signatures.html)

**Goal:** Understand how a private key proves authorship and integrity.

**Topics:**
- Sign-with-private, verify-with-public; signing a hash, not the data
- RSA-PSS, ECDSA, Ed25519
- What a signature proves (authenticity + integrity + non-repudiation) and what it doesn't (secrecy)
- The role of signatures in certificates, JWTs, software updates

**Lab:** Sign a file and verify it; tamper with it and watch verification fail.

**Checkpoint:** Why do you sign a hash of a message rather than the message itself?

---

### [Lesson 5: Randomness & entropy](lessons/lesson-05-randomness.html)

**Goal:** See why bad randomness silently breaks otherwise-perfect crypto.

**Topics:**
- CSPRNGs vs ordinary PRNGs; `/dev/urandom`, `getrandom(2)`
- Entropy at boot; the embedded-device problem
- Real-world failures (repeated nonces, predictable keys)
- Where keys, IVs, and nonces must come from

**Checkpoint:** What happens to ECDSA security if the same nonce is reused for two signatures?

---

## [Phase 2: PKI & Certificates](lessons/phase-02-pki-certificates.html)

### [Lesson 6: X.509 certificates anatomy](lessons/lesson-06-x509.html)

**Goal:** Read a certificate field by field.

**Topics:**
- Subject, issuer, validity, public key, serial
- Extensions: SAN, key usage, extended key usage, basic constraints
- DER vs PEM encoding
- Why the CN is dead and SAN is everything now

**Lab:** `openssl x509 -text` on a real site's certificate; identify every field.

**Checkpoint:** Which field does a browser actually use to match a certificate to a hostname?

---

### [Lesson 7: Certificate authorities & chains of trust](lessons/lesson-07-ca-trust.html)

**Goal:** Understand how trust is delegated from a root to a leaf.

**Topics:**
- Root vs intermediate vs leaf; why intermediates exist
- Trust stores (OS/browser); how a root gets trusted
- Path building and chain validation
- Cross-signing and the "which chain did I get served" problem

**Checkpoint:** Why do servers send intermediate certificates but not the root?

---

### [Lesson 8: CSRs & the openssl toolkit](lessons/lesson-08-csr-openssl.html)

**Goal:** Generate keys and request certificates the way real deployments do.

**Topics:**
- The CSR: what's in it, what the CA adds
- `openssl req`, `genpkey`, `x509`, `ca`
- Subject Alternative Names in a CSR config
- Keeping the private key private

**Lab:** Generate a key + CSR with SANs; inspect the CSR.

**Checkpoint:** What part of a certificate comes from the CSR and what part does the CA decide?

---

### [Lesson 9: Certificate validation](lessons/lesson-09-validation.html)

**Goal:** Know every check a client makes before trusting a certificate.

**Topics:**
- Signature chain, validity dates, hostname matching
- Key usage / EKU enforcement; basic-constraints `CA:TRUE`
- Name constraints; the classic validation bugs
- Why "it's encrypted" ≠ "it's the right server"

**Checkpoint:** List the checks a client must pass before trusting a server certificate.

---

### [Lesson 10: Revocation — CRL, OCSP, stapling](lessons/lesson-10-revocation.html)

**Goal:** Understand how a certificate gets untrusted before it expires — and why it's hard.

**Topics:**
- Why revocation is needed (key compromise)
- CRLs; OCSP; the privacy and reliability problems
- OCSP stapling and must-staple
- Short-lived certificates as the modern alternative

**Checkpoint:** Why has the industry largely moved from CRLs/OCSP toward short-lived certs?

---

### [Lesson 11: Certificate Transparency](lessons/lesson-11-ct.html)

**Goal:** Understand the public audit log that keeps CAs honest.

**Topics:**
- The misissuance problem CT solves
- Append-only Merkle-tree logs; SCTs
- How browsers enforce CT
- Monitoring CT logs for your own domains

**Checkpoint:** How does Certificate Transparency let you detect a certificate mis-issued for your domain?

---

### [Lesson 12: Running a private CA](lessons/lesson-12-private-ca.html)

**Goal:** Stand up your own CA for internal services.

**Topics:**
- Root + intermediate hierarchy; offline root
- Issuing leaf certs; distributing the root to clients
- `step-ca` / `openssl ca`
- When a private CA is the right tool (internal mTLS, dev)

**Lab:** Create a root + intermediate; issue and verify a leaf certificate.

**Checkpoint:** Why keep the root CA key offline and issue from an intermediate?

---

### [Lesson 13: ACME & Let's Encrypt](lessons/lesson-13-acme.html)

**Goal:** Automate certificate issuance end to end.

**Topics:**
- The ACME protocol; HTTP-01 and DNS-01 challenges
- Domain control validation
- `certbot` / `lego` / `step`; renewal automation
- Rate limits and staging environments

**Checkpoint:** How does an ACME challenge prove you control a domain without any human in the loop?

---

## [Phase 3: TLS & SSL](lessons/phase-03-tls-ssl.html)

### [Lesson 14: SSL → TLS history & versions](lessons/lesson-14-tls-history.html)

**Goal:** Understand how we got here and why old versions must die.

**Topics:**
- SSL 2/3 → TLS 1.0/1.1/1.2/1.3 timeline
- What each version fixed; why 1.0/1.1 are deprecated
- Where TLS sits in the stack (callback to networking Lesson 70)

**Checkpoint:** Why are TLS 1.0 and 1.1 no longer acceptable?

---

### [Lesson 15: The TLS 1.2 handshake](lessons/lesson-15-tls12-handshake.html)

**Goal:** Trace a full TLS 1.2 handshake message by message.

**Topics:**
- ClientHello/ServerHello; certificate; key exchange; Finished
- RSA key transport vs ECDHE
- Round trips and latency cost

**Lab:** `openssl s_client -tls1_2` to a server; map the messages to the model.

**Checkpoint:** What's exchanged in each round trip of a TLS 1.2 handshake?

---

### [Lesson 16: The TLS 1.3 handshake](lessons/lesson-16-tls13-handshake.html)

**Goal:** See what 1.3 removed and why it's faster and safer.

**Topics:**
- 1-RTT handshake; encrypted handshake messages
- Mandatory forward secrecy; removed legacy crypto
- HelloRetryRequest; key schedule overview

**Lab:** `openssl s_client -tls1_3`; compare with the 1.2 capture.

**Checkpoint:** How does TLS 1.3 cut the handshake to one round trip?

---

### [Lesson 17: Cipher suites & forward secrecy](lessons/lesson-17-cipher-suites.html)

**Goal:** Read a cipher suite and understand what each part guarantees.

**Topics:**
- Anatomy of a suite (KX, auth, cipher, MAC) in 1.2 vs the simplified 1.3 set
- Forward secrecy via ephemeral key exchange
- Choosing safe suites; ordering

**Checkpoint:** What does "forward secrecy" protect against, and which suites provide it?

---

### [Lesson 18: Mutual TLS (mTLS)](lessons/lesson-18-mtls.html)

**Goal:** Authenticate the *client* with a certificate too.

**Topics:**
- CertificateRequest; client cert + verify
- Use cases: service-to-service, device identity
- Trust configuration on both ends

**Lab:** Configure a server to require client certs; connect with and without one.

**Checkpoint:** What does mTLS prove that ordinary server-only TLS does not?

---

### [Lesson 19: SNI, ALPN, resumption & 0-RTT](lessons/lesson-19-sni-alpn-resumption.html)

**Goal:** Understand the extensions that make the modern web work.

**Topics:**
- SNI (and ECH); virtual hosting many sites on one IP
- ALPN (how HTTP/2 and HTTP/3 are negotiated)
- Session resumption (tickets); 0-RTT and its replay risk

**Checkpoint:** Why is 0-RTT data subject to replay, and how is that mitigated?

---

### [Lesson 20: Inspecting & hardening TLS](lessons/lesson-20-tls-hardening.html)

**Goal:** Audit and tighten a real TLS deployment.

**Topics:**
- `openssl s_client`, `testssl.sh`, SSL Labs methodology
- Disabling weak protocols/suites; HSTS
- Cert/key file hygiene; OCSP stapling config

**Lab:** Scan a server; identify and fix one weakness.

**Checkpoint:** Name three things you'd check to call a TLS deployment "hardened."

---

### [Lesson 21: TLS attacks & the lessons they taught](lessons/lesson-21-tls-attacks.html)

**Goal:** Learn defense by studying historic breaks.

**Topics:**
- Downgrade attacks; BEAST, CRIME, POODLE, Heartbleed (a memory bug, not crypto)
- What each one changed in defaults
- Why "use the library, don't roll your own" is the meta-lesson

**Checkpoint:** Pick one historic TLS attack and explain the defense that resulted from it.

---

## [Phase 4: Authentication Fundamentals](lessons/phase-04-authentication.html)

### [Lesson 22: Authentication vs authorization](lessons/lesson-22-authn-authz.html)

**Goal:** Nail the distinction the rest of the track depends on.

**Topics:**
- AuthN (who are you) vs AuthZ (what may you do)
- Identity, credentials, claims, principals
- Where each lives in a request

**Checkpoint:** Classify a list of operations as authentication or authorization.

---

### [Lesson 23: Sessions, cookies & tokens](lessons/lesson-23-sessions-tokens.html)

**Goal:** Understand how a web app remembers you after login.

**Topics:**
- Server-side sessions vs stateless tokens
- Cookies: `Secure`, `HttpOnly`, `SameSite`; CSRF
- Bearer tokens and where to store them (XSS risk)

**Checkpoint:** What attack does `SameSite` mitigate, and how?

---

### [Lesson 24: Passwords & credential storage](lessons/lesson-24-passwords.html)

**Goal:** Do the thing everyone gets wrong, correctly.

**Topics:**
- Storage with argon2/bcrypt (callback to Lesson 2); salting
- Rate limiting, lockout, breach (k-anonymity / HIBP)
- Password policies that actually help (length > complexity)

**Checkpoint:** Why is per-user salting essential even with a slow hash?

---

### [Lesson 25: Multi-factor authentication (TOTP/HOTP)](lessons/lesson-25-mfa.html)

**Goal:** Add a second factor and understand its limits.

**Topics:**
- Something you know/have/are
- HOTP and TOTP (RFC 4226/6238); how authenticator apps work
- SMS OTP weaknesses; push fatigue

**Lab:** Generate a TOTP secret and validate codes with `oathtool`.

**Checkpoint:** Why is TOTP phishable, and what factor resists phishing?

---

### [Lesson 26: WebAuthn, FIDO2 & passkeys](lessons/lesson-26-webauthn.html)

**Goal:** Understand phishing-resistant, public-key authentication.

**Topics:**
- WebAuthn ceremony (registration + assertion); attestation
- Origin binding (why it resists phishing)
- Passkeys: synced vs device-bound; the UX shift

**Checkpoint:** What makes a passkey phishing-resistant where TOTP is not?

---

### [Lesson 27: Kerberos](lessons/lesson-27-kerberos.html)

**Goal:** Understand the classic enterprise ticket-based SSO.

**Topics:**
- KDC, TGT, service tickets; the three-party model
- Why tickets avoid sending passwords around
- Where it's used (Active Directory) and its constraints

**Checkpoint:** How does Kerberos let you reach many services after authenticating once?

---

## [Phase 5: Federated Identity & Authorization](lessons/phase-05-federated-identity.html)

### [Lesson 28: OAuth 2.0 fundamentals](lessons/lesson-28-oauth2.html)

**Goal:** Understand delegated authorization and its four roles.

**Topics:**
- Resource owner, client, authorization server, resource server
- Access tokens, refresh tokens, scopes
- "OAuth is authorization, not authentication" — the cardinal rule

**Checkpoint:** Who are the four parties in an OAuth exchange and what does each do?

---

### [Lesson 29: OAuth 2.0 flows & PKCE](lessons/lesson-29-oauth-flows.html)

**Goal:** Pick the right grant and understand why the others are deprecated.

**Topics:**
- Authorization code + PKCE (the default for everything now)
- Client credentials (machine-to-machine)
- Why implicit and password grants are dead
- Redirect URIs and the dangers around them

**Lab:** Walk an authorization-code + PKCE flow against a local IdP with `curl`.

**Checkpoint:** What attack does PKCE prevent, and why is it now required for public clients?

---

### [Lesson 30: OpenID Connect (OIDC)](lessons/lesson-30-oidc.html)

**Goal:** Get *authentication* by layering OIDC on OAuth.

**Topics:**
- ID token vs access token; the `userinfo` endpoint
- Discovery (`.well-known/openid-configuration`) and JWKS
- Standard claims and scopes (`openid`, `profile`, `email`)

**Checkpoint:** What does the ID token give you that an OAuth access token does not?

---

### [Lesson 31: JWT deep dive](lessons/lesson-31-jwt.html)

**Goal:** Read, validate, and *not* get burned by JWTs.

**Topics:**
- Header/payload/signature; JWS vs JWE
- Validation: signature, `alg`, `iss`, `aud`, `exp`
- The classic vulns: `alg:none`, RS256→HS256 confusion, missing `aud` check

**Lab:** Decode and verify a JWT; demonstrate why `alg:none` must be rejected.

**Checkpoint:** Why must a verifier pin the expected algorithm rather than trust the token's `alg`?

---

### [Lesson 32: SAML 2.0](lessons/lesson-32-saml.html)

**Goal:** Understand enterprise browser SSO and how it differs from OIDC.

**Topics:**
- Assertions; SP and IdP; metadata
- SP-initiated vs IdP-initiated browser flows
- XML signing and the wrapping attacks to beware
- SAML vs OIDC: when you'll still meet SAML

**Checkpoint:** Compare the SAML assertion to the OIDC ID token — what role do both play?

---

### [Lesson 33: SSO & session management](lessons/lesson-33-sso.html)

**Goal:** Manage identity *across* many applications cleanly.

**Topics:**
- SSO session at the IdP vs app sessions
- Single logout and its difficulties
- Token lifetimes, silent renewal, back-channel logout

**Checkpoint:** Why is single logout harder than single sign-on?

---

### [Lesson 34: Identity providers in practice (Keycloak)](lessons/lesson-34-idp-keycloak.html)

**Goal:** Run a real IdP and connect an app to it.

**Topics:**
- Realms, clients, users, roles, mappers
- Configuring an OIDC client end to end
- Federation and social login brokering

**Lab:** Run Keycloak in a container; register an OIDC client and complete a login.

**Checkpoint:** What does an identity provider centralize that you'd otherwise rebuild per app?

---

## [Phase 6: Applied Security & Identity](lessons/phase-06-applied.html)

### [Lesson 35: mTLS & workload identity (SPIFFE/SPIRE)](lessons/lesson-35-workload-identity.html)

**Goal:** Give *services* cryptographic identities, not shared secrets.

**Topics:**
- The shared-secret sprawl problem
- SPIFFE IDs and SVIDs (X.509/JWT); SPIRE issuance & attestation
- Automatic rotation; mesh integration

**Checkpoint:** How does workload identity replace long-lived API keys between services?

---

### [Lesson 36: Secret management & short-lived credentials](lessons/lesson-36-secrets.html)

**Goal:** Stop hardcoding secrets and shrink their blast radius.

**Topics:**
- Secret stores (Vault); dynamic, short-lived secrets
- Leasing, rotation, revocation
- Secret-zero / bootstrapping trust

**Checkpoint:** Why are short-lived dynamic credentials safer than a long-lived static secret?

---

### [Lesson 37: API authentication patterns](lessons/lesson-37-api-auth.html)

**Goal:** Choose the right auth mechanism for an API.

**Topics:**
- API keys vs bearer tokens vs mTLS
- OAuth client-credentials for service APIs
- Rate limiting, scoping, and key rotation as first-class concerns

**Checkpoint:** When would you choose mTLS over a bearer token for an internal API?

---

### [Lesson 38: Zero-trust architecture](lessons/lesson-38-zero-trust.html)

**Goal:** Understand "never trust, always verify" as an architecture.

**Topics:**
- The perimeter model's failure; BeyondCorp
- Identity-aware proxies; per-request authZ; device posture
- Ties back to the mesh-VPN model (networking Phase 16)

**Checkpoint:** What replaces "inside the network = trusted" in a zero-trust design?

---

### [Lesson 39: Token lifecycle & revocation](lessons/lesson-39-token-lifecycle.html)

**Goal:** Manage tokens safely from issuance to death.

**Topics:**
- Access vs refresh token lifetimes; rotation
- Revocation and introspection; the stateless-JWT revocation problem
- Detecting and responding to token theft

**Checkpoint:** Why is revoking a stateless JWT hard, and what are the workarounds?

---

### [Lesson 40: Threat modeling authentication systems](lessons/lesson-40-threat-modeling.html)

**Goal:** Reason systematically about how an auth system fails.

**Topics:**
- STRIDE applied to auth; the OWASP Top 10 angles
- Common failure modes from this whole track, in one place
- Building an attack tree for a login flow

**Checkpoint:** Build a short threat model for a username+password+TOTP login and name its weakest link.

---

## Recommended Learning Order

Each phase depends on the previous ones — crypto underpins certificates, certificates
underpin TLS, and identity protocols assume all three.

1. Symmetric crypto & AEAD
2. Hashing, MACs, password hashing
3. Asymmetric crypto & Diffie-Hellman
4. Digital signatures
5. Randomness & entropy
6. X.509 certificates & chains of trust
7. CSRs, validation, revocation, transparency
8. Private CA & ACME automation
9. TLS history and the 1.2 / 1.3 handshakes
10. Cipher suites, forward secrecy, mTLS
11. TLS extensions, hardening, historic attacks
12. AuthN vs AuthZ; sessions, cookies, tokens
13. Passwords, MFA, passkeys, Kerberos
14. OAuth 2.0 + PKCE; OpenID Connect
15. JWT; SAML; SSO & session management
16. Identity providers (Keycloak)
17. Workload identity, secrets, API auth
18. Zero-trust, token lifecycle, threat modeling
