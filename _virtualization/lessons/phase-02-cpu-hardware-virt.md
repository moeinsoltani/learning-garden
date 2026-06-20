---
title: "Phase 2: CPU & Hardware Virtualization"
nav_order: 4
has_children: true
---

# Phase 2: CPU & Hardware Virtualization

This phase is the *physics* of virtualization: protection rings and the classical
trap-and-emulate technique, why plain x86 broke it, and the hardware extensions
(Intel VT-x / AMD-V, EPT/NPT) that fixed it and make KVM possible. It ends with
the practical skill of getting a host ready to run KVM.

| # | Lesson | Status |
|---|--------|--------|
| 04 | [Rings, privileged instructions, and trap-and-emulate](lesson-04-rings-trap-emulate) | In progress |
| 05 | [Hardware virtualization extensions (VT-x / AMD-V)](lesson-05-vtx-amdv) | In progress |
| 06 | [Memory virtualization — EPT / NPT (SLAT)](lesson-06-ept-npt-slat) | In progress |
| 07 | [Checking and enabling virtualization on the host](lesson-07-host-readiness) | In progress |
