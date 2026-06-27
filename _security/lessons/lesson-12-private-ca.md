---
title: "Lesson 12 — Running a Private CA"
nav_order: 12
parent: "Phase 2: PKI & Certificates"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 12: Running a Private CA

## Concept

Public CAs (Lesson 7) are for internet-facing names. For **internal** services — service-to-service
mTLS, dev environments, internal tools — you run your **own** CA. You create a root, distribute it to
your machines as trusted, and issue certificates for your internal names. It's the same chain-of-trust
model, but *you* are the trust anchor.

```
   Your offline ROOT  →  signs  →  Intermediate  →  signs  →  internal service certs
        │
   distribute root to all your hosts' trust stores → they trust everything you issue
```

---

## How it works

**Build a hierarchy, not a single CA.** Best practice mirrors public PKI: a **root** that's created
once and kept **offline**, used only to sign one or more **intermediates**, which do the actual issuing.
If an intermediate's key is exposed, you revoke it and issue a new one without rebuilding trust on every
device (which would mean re-distributing a new root everywhere).

**Distribute the root.** Internal clients trust your CA only if its **root certificate is installed in
their trust stores** (system CA bundle, container images, language runtimes). That distribution step is
what makes your private PKI work — and is the operational burden of running one.

**Tools.** `openssl ca` can do it, but it's fiddly; **[step-ca](https://smallstep.com/docs/step-ca/)**
(and `cfssl`) are purpose-built private-CA tools that handle issuance, renewal, and even ACME for
internal automation.

**When a private CA is right.** Internal mTLS (Phase 6), dev/test certs, IoT/device fleets, anything not
exposed to the public internet where you control all the clients. **When it's wrong:** public-facing
sites — external users won't have your root, so they'd get trust errors; use a public CA (Lesson 13) there.

{: .note }
> **Keep the root offline; issue from the intermediate**
> The root key is your crown jewel — if it leaks, an attacker can mint trusted certs for your entire
> internal world, and recovery means re-distributing a new root to every host. So generate it, sign one
> intermediate, then take the root <em>offline</em> (encrypted, air-gapped). Day-to-day issuance uses the
> intermediate, which is replaceable. This is the same reasoning public CAs use, scaled down.

---

## Lab

```bash
# 1. Create a self-signed ROOT CA
$ openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out root.key
$ openssl req -x509 -new -key root.key -days 3650 -subj "/CN=My Internal Root" \
    -addext "basicConstraints=critical,CA:TRUE" -addext "keyUsage=critical,keyCertSign,cRLSign" \
    -out root.crt

# 2. Create an INTERMEDIATE, signed by the root
$ openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out int.key
$ openssl req -new -key int.key -subj "/CN=My Internal Intermediate" -out int.csr
$ openssl x509 -req -in int.csr -CA root.crt -CAkey root.key -CAcreateserial -days 1825 \
    -copy_extensions copyall -extfile <(printf "basicConstraints=critical,CA:TRUE,pathlen:0\nkeyUsage=critical,keyCertSign,cRLSign") \
    -out int.crt

# 3. Issue a LEAF for an internal service
$ openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out svc.key
$ openssl req -new -key svc.key -subj "/CN=svc.internal" \
    -addext "subjectAltName=DNS:svc.internal" -out svc.csr
$ openssl x509 -req -in svc.csr -CA int.crt -CAkey int.key -CAcreateserial -days 90 \
    -copy_extensions copyall -out svc.crt

# 4. Verify the full chain
$ cat int.crt root.crt > chain.crt
$ openssl verify -CAfile root.crt -untrusted int.crt svc.crt
svc.crt: OK
```

To actually *use* this internally, install `root.crt` into clients' trust stores.

---

## Further Reading

| Topic | Link |
|---|---|
| Public key infrastructure | [Wikipedia — Public key infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) |
| step-ca | [smallstep.com/docs/step-ca](https://smallstep.com/docs/step-ca/) |
| `openssl ca` | [openssl ca](https://docs.openssl.org/master/man1/openssl-ca/) |
| Chain of trust | [Lesson 7 — CAs & chains of trust](lesson-07-ca-trust) |

---

## Checkpoint

**Q1. Why keep the root CA key offline and issue from an intermediate, even for a private/internal CA?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because the root is the trust anchor that you've installed in every internal client's trust store. If the
<strong>root key</strong> were compromised, an attacker could issue trusted certificates for your entire
internal environment, and recovering means generating a new root and <strong>re-distributing it to every
host</strong> — a painful, error-prone operation. By keeping the root <strong>offline</strong> (air-
gapped, used only to sign intermediates) and doing all day-to-day issuance from a replaceable
<strong>intermediate</strong>, a key compromise is contained: you revoke the intermediate and sign a new
one with the still-safe offline root, <em>without</em> touching the trust stores. It minimizes exposure of
the most valuable key, exactly as public CAs do — the logic scales down to private PKI unchanged.
</details>

---

**Q2. What makes a private CA actually trusted by your internal clients, and what's the operational burden?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A private CA is trusted only if its <strong>root certificate is installed in each client's trust
store</strong> (the OS CA bundle, container base images, and any language/runtime-specific stores). Once a
client trusts your root, it trusts every certificate that chains up to it. The <strong>operational
burden</strong> is exactly that distribution and lifecycle management: you must reliably deploy the root to
every host/container/device that needs it, update those stores if you rotate the root, and run issuance/
renewal/revocation yourself (no public CA does it for you). For ephemeral infrastructure (containers,
autoscaling) this means baking the root into images or provisioning it automatically — which is why tools
like step-ca exist to automate issuance and trust distribution.
</details>

---

**Q3. When is a private CA the right tool, and when should you use a public CA instead?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A <strong>private CA</strong> is right when <em>you control all the clients</em> and the names aren't
public: internal service-to-service mTLS, dev/test certificates, internal admin tools, device/IoT fleets,
and anything inside your own trust boundary. There, issuing your own certs is cheaper, faster, and lets you
do things public CAs won't (arbitrary internal names, very short lifetimes, custom policies). A
<strong>public CA</strong> (Lesson 13) is required for anything <em>public-facing</em> — websites and APIs
reached by external users/browsers — because those clients won't have your private root in their trust
store and would get certificate errors; only a CA already in the public trust stores will be accepted.
Rule of thumb: public internet audience → public CA; internal, controlled clients → private CA.
</details>

---

## Homework

Build the root→intermediate→leaf hierarchy from the lab, then install the root into your local trust store
(e.g. `update-ca-certificates` after copying to `/usr/local/share/ca-certificates/`) and serve the leaf
with a simple TLS server. Confirm `curl` trusts it *with* the root installed and fails *without* it.
Explain what this proves about where trust actually comes from.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
After building the hierarchy and serving the leaf, <code>curl https://svc.internal</code> <strong>fails</strong>
with a trust error <em>before</em> you install the root ("self-signed certificate in chain" / "unable to
get local issuer certificate"), because your private root isn't in the system trust store. Once you copy
<code>root.crt</code> into the trust store and run <code>update-ca-certificates</code>, the same
<code>curl</code> <strong>succeeds</strong> — nothing about the certificate changed, only the client's
trust configuration. This proves that <strong>trust comes from the client's trust store, not from the
certificate itself</strong>: a certificate is just a signed binding, and whether it's accepted depends
entirely on whether the verifier already trusts the root that anchors its chain. It's the same mechanism
public CAs rely on — they're simply pre-installed in everyone's trust store — and it's why running a
private CA is fundamentally about <em>distributing your root</em> to the clients you control.
</details>
