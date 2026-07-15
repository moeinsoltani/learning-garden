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

### 🧠 Operating Systems

The kernel boundary & syscalls, processes, scheduling, memory, concurrency, IPC,
filesystems, async I/O, linking & loading, boot, kernel modules, containers from
scratch, and tracing — every lesson observed live with strace, /proc, gdb, and
small C programs.

[OS learning plan]({{ '/os/learning-plan.html' | relative_url }}){: .btn .btn-primary }

### 🧭 Engineering Leadership

The Senior → Lead/EM transition: technical leadership, communication, feedback &
difficult conversations, coaching, delegation, influence without authority,
stakeholders, business thinking, project leadership, and people management.
Labs are realistic scenario exercises with model answers.

[Leadership learning plan]({{ '/leadership/learning-plan.html' | relative_url }}){: .btn .btn-primary }

### 💬 English for Work

Professional English for software teams: accurate sentences (articles, tenses,
agreement), warm tone, Slack communication, meetings & spoken fluency, design
docs, and feedback language. Labs are rewrite drills with model rewrites and
phrase banks.

[English learning plan]({{ '/english/learning-plan.html' | relative_url }}){: .btn .btn-primary }

### 🍁 Canada: History & Civics

Canada from the ground up: geography, the First Peoples and colonial era,
Confederation and the modern nation, how government actually works, civic life,
the economy, and culture & identity. Labs are source & scenario exercises, with
citizenship-test-style checkpoints throughout.

[Canada learning plan]({{ '/canada/learning-plan.html' | relative_url }}){: .btn .btn-primary }

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
> The tracks are independent — do one at a time or interleave them. They
> interlink where topics overlap (e.g. VM networking reuses the bridge/TAP lessons
> from the networking track, the security track's TLS phase pairs with networking
> Lesson 70, and English for Work pairs with Engineering Leadership: one teaches
> the language, the other the judgment).
