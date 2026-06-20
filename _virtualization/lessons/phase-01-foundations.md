---
title: "Phase 1: Virtualization Foundations"
nav_order: 3
has_children: true
---

# Phase 1: Virtualization Foundations

Before touching a single tool, you need the mental model: what virtualization
*is*, how it differs from emulation and paravirtualization, the family tree of
hypervisors, and the single most important relationship in this whole track —
the division of labor between **QEMU** (userspace) and **KVM** (the kernel).

Everything in later phases (CPU extensions, memory, storage, virtio, libvirt) is
a refinement of the ideas introduced here.

| # | Lesson | Status |
|---|--------|--------|
| 01 | [What virtualization actually is](lesson-01-what-virtualization-is) | In progress |
| 02 | [Hypervisor types and where KVM fits](lesson-02-hypervisor-types) | In progress |
| 03 | [The QEMU + KVM division of labor](lesson-03-qemu-kvm-division) | In progress |
