---
title: Home
nav_order: 1
---

# Linux Systems — Learning Journal

A hands-on, bottom-up journal for learning Linux systems internals. Pick a track —
each is an independent curriculum with a phased learning plan and self-contained
lessons (mental model, mechanics, labs with expected output, and checkpoints).

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
> Use the sidebar to jump between the two tracks. Within a track, start at its
> learning plan, then work through the phases in order — each lesson builds on the
> previous ones. The two tracks interlink where topics overlap (e.g. VM networking
> reuses the bridge/TAP lessons from the networking track).
