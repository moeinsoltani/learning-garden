---
title: Home
nav_order: 1
---

# Linux Systems — Learning Journal

A hands-on, bottom-up journal for learning Linux systems internals. Pick a track —
each is an independent curriculum with a phased learning plan and self-contained
lessons (mental model, mechanics, labs with expected output, and checkpoints).

New here? **[Set up your lab environment]({{ '/lab-setup.html' | relative_url }})**
first — it takes ten minutes and every lesson's lab depends on it.

---

## Tracks

### 🌐 Linux Networking

Network namespaces, virtual interfaces, bridges & VLANs, overlays, routing, NAT,
firewalling with nftables, traffic control, eBPF/XDP, and container/Kubernetes
networking.

[Networking learning plan]({{ '/networking/learning-plan.html' | relative_url }}){: .btn .btn-primary }

### 🖥️ Linux Virtualization (QEMU / KVM)

CPU & hardware virtualization, KVM internals, QEMU, CPU/memory tuning, storage,
virtio, device passthrough (VFIO), libvirt, cloud images, live migration,
performance, security, and microVMs.

[Virtualization learning plan]({{ '/virtualization/learning-plan.html' | relative_url }}){: .btn .btn-primary }

### 🔐 Security & Identity

Cryptography foundations, X.509 certificates & PKI, TLS/SSL, authentication
(passwords, MFA, passkeys, Kerberos), and federated identity & authorization
(OAuth 2.0, OIDC, SAML, JWT, SSO, and identity providers).

[Security learning plan]({{ '/security/learning-plan.html' | relative_url }}){: .btn .btn-primary }

---

{: .note }
> **How to use this site**
> 1. Prepare your machine once with the [Lab Setup]({{ '/lab-setup.html' | relative_url }}) page.
> 2. Pick a track and skim its learning plan to see the road ahead.
> 3. Work through the lessons **in order** — each builds on the previous ones.
>    Read the concept, type every lab command yourself, and answer each
>    checkpoint *before* revealing the model answer.
> 4. Use the **Next lesson** link at the bottom of each page to keep moving.
>
> The three tracks are independent — do one at a time or interleave them. They
> interlink where topics overlap (e.g. VM networking reuses the bridge/TAP lessons
> from the networking track, and the security track's TLS phase pairs with
> networking Lesson 70).
