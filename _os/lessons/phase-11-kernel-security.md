---
title: "Phase 11: Kernel Interfaces & Security"
nav_order: 13
has_children: true
---

# Phase 11: Kernel Interfaces & Security

Time to cross the boundary you've studied from outside for ten phases. This
phase has you write code that runs *in* the kernel — a hello-world module, a
character device implementing the other end of `/dev` — then covers the two
mechanisms that shrink what code can do: seccomp (restrict syscalls) and LSM
(mandatory access control, completing virt Lesson 56's sVirt story). The
capstone builds and boots your own kernel in QEMU.

| # | Lesson | Status |
|---|--------|--------|
| 54 | [Write a kernel module](lesson-54-kernel-module) | Ready |
| 55 | [A character device](lesson-55-char-device) | Ready |
| 56 | [seccomp](lesson-56-seccomp) | Ready |
| 57 | [LSM — AppArmor and SELinux](lesson-57-lsm) | Ready |
| 58 | [Build and boot your own kernel](lesson-58-build-kernel) | Ready |
