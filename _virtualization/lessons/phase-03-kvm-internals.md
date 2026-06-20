---
title: "Phase 3: KVM Internals"
nav_order: 5
has_children: true
---

# Phase 3: KVM Internals

Now we open the hood. KVM is "just" a character device with an ioctl API: you
create a VM, create vCPUs, hand it memory, and call `KVM_RUN` in a loop. This phase
walks that API, the kernel modules and their tunables, the VM-exit loop in depth,
and how to *observe* KVM at runtime so the invisible world switches become visible.

| # | Lesson | Status |
|---|--------|--------|
| 08 | [The /dev/kvm ioctl API](lesson-08-dev-kvm-ioctl) | In progress |
| 09 | [KVM kernel modules and parameters](lesson-09-kvm-modules-params) | In progress |
| 10 | [The VM-exit loop in depth](lesson-10-vm-exit-loop) | In progress |
| 11 | [Observing KVM at runtime](lesson-11-observing-kvm) | In progress |
