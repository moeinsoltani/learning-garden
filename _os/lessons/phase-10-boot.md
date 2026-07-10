---
title: "Phase 10: Boot & Init"
nav_order: 12
has_children: true
---

# Phase 10: Boot & Init

How does a machine go from power-on to a login prompt? This phase traces the
whole chain: firmware handing off to the kernel, the initramfs that mounts
your real root, systemd as PID 1 running the userspace, and the kernel-module
+ udev machinery that makes hardware appear. It pairs with virtualization
Lesson 14 (firmware/boot in a VM) and completes the process story from
Phase 2 — you'll finally meet PID 1's real job.

| # | Lesson | Status |
|---|--------|--------|
| 50 | [From firmware to kernel](lesson-50-boot-process) | Ready |
| 51 | [initramfs](lesson-51-initramfs) | Ready |
| 52 | [systemd — PID 1](lesson-52-systemd) | Ready |
| 53 | [Kernel modules and udev](lesson-53-modules-udev) | Ready |
