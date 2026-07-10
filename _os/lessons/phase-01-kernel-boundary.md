---
title: "Phase 1: The Kernel Boundary"
nav_order: 3
has_children: true
---

# Phase 1: The Kernel Boundary

Everything in this track hangs off one idea: there are exactly two worlds on your
machine — unprivileged user space and the all-powerful kernel — and exactly one
door between them, the system call. This phase builds that mental model and hands
you the observation tools (`strace`, `ltrace`, `/proc`, `/sys`) that every later
lab depends on. By the end you can watch any program talk to the kernel and read
what you see.

| # | Lesson | Status |
|---|--------|--------|
| 1 | [What an operating system actually does](lesson-01-what-an-os-does) | Ready |
| 2 | [System calls — the only door in](lesson-02-syscalls-strace) | Ready |
| 3 | [libc, the ABI, and the vDSO](lesson-03-libc-abi-vdso) | Ready |
| 4 | [/proc and /sys — the kernel's dashboard](lesson-04-proc-sys) | Ready |
| 5 | [Interrupts and the timer tick](lesson-05-interrupts-timers) | Ready |
