---
title: "Lesson 03 — The QEMU + KVM Division of Labor"
nav_order: 3
parent: "Phase 1: Virtualization Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 03: The QEMU + KVM Division of Labor

## Concept

This is **the single most important relationship in the entire track.** Almost
every later topic — virtio, vhost, passthrough, performance tuning — is about
*moving work across the line* drawn in this lesson.

The line is between:

- **KVM** — the in-kernel module that uses the CPU's virtualization extensions to
  *run guest code* and isolate guest memory.
- **QEMU** — the userspace process that *emulates the machine*: devices (disk,
  NIC, BIOS, interrupt controller), the memory layout, and that *drives* KVM.

```
   ┌──────────────────── USERSPACE ────────────────────┐
   │  QEMU process (the VMM / "device model")           │
   │    • emulates disk, NIC, VGA, RTC, BIOS            │
   │    • allocates guest RAM (mmap)                    │
   │    • runs one thread per vCPU                      │
   │                                                    │
   │        ioctl(vcpu_fd, KVM_RUN)  ──────────┐        │
   └───────────────────────────────────────────┼───────┘
                                                │  ▲
   ┌──────────────── KERNEL ────────────────────┼──┼───┐
   │  /dev/kvm  (KVM module)                     ▼  │   │
   │    • enters VMX/SVM guest mode                 │   │
   │    • guest vCPU runs on the REAL CPU           │   │
   │    • on I/O or fault → VM EXIT ────────────────┘   │
   └────────────────────────────────────────────────────┘
```

---

## How It Works

### The KVM_RUN loop — the heartbeat of every VM

For each virtual CPU, QEMU runs a dedicated thread that spins in this loop:

```
   loop forever:
       ioctl(vcpu_fd, KVM_RUN)        # ── hand the CPU to the guest
       # ... guest code runs NATIVELY on the physical core ...
       # ... until something happens that the guest can't be allowed
       #     to do directly: an I/O access, an unmapped page, HLT, etc.
       # ── that triggers a VM EXIT; KVM_RUN returns to QEMU
       switch (kvm_run->exit_reason):
           case KVM_EXIT_IO:    emulate the device register access
           case KVM_EXIT_MMIO:  emulate memory-mapped I/O
           case KVM_EXIT_HLT:   guest idled
           ...
       # loop back: KVM_RUN again to resume the guest
```

While the guest runs, **QEMU is blocked inside the `ioctl`** — the physical CPU is
executing guest instructions, not QEMU. QEMU only wakes up when the guest does
something that requires host intervention (a **VM exit**). It then services that
event and re-enters the guest. Reducing the number of these exits is the entire
game of performance tuning (Phases 7 and 13).

### Who handles what — the key examples

This is the question the checkpoint asks, so internalize it:

| Guest action | Handled by | Why |
|---|---|---|
| Arithmetic, tight CPU loop | **Hardware** (no exit) | Pure computation needs no host help — runs natively. |
| Page fault on guest RAM that is *backed* | **KVM (kernel)** | KVM fixes up the second-level page tables (EPT/NPT) without leaving the kernel. |
| Write to an emulated NIC register | **QEMU (userspace)** | The NIC is a software device modeled in QEMU; the write becomes a VM exit handed up to QEMU. |
| Timer/interrupt injection (in-kernel irqchip) | **KVM (kernel)** | KVM has an in-kernel interrupt controller to avoid bouncing to userspace. |
| Disk read from a virtio-blk device | **QEMU (or vhost)** | The block backend lives in userspace QEMU (Phase 6), unless offloaded. |

### Why the split exists at all

Running *everything* in the kernel would bloat the kernel with a huge, complex,
attack-prone device model. Running *everything* in userspace would be too slow
(software CPU emulation). The split puts the **performance-critical, security-
sensitive CPU/memory execution in the kernel** and the **large, flexible, evolving
device model in userspace**, where a bug can be sandboxed (Phase 14) and crashes
only kill one VM's process — not the host kernel.

### Other VMMs that drive KVM

QEMU is the most common userspace driver of `/dev/kvm`, but not the only one. The
same kernel API is used by **crosvm** (ChromeOS), **Cloud Hypervisor**, and
**Firecracker** (AWS Lambda/Fargate) — minimalist Rust VMMs with tiny device
models for fast, secure microVMs. You meet them in Phase 15.

---

## Lab

```bash
# Start a tiny VM in the background so we can inspect the process structure.
# (No disk needed; it will just sit at the BIOS. Ctrl-A X or kill it after.)
$ qemu-system-x86_64 -accel kvm -m 512 -smp 2 -nographic -name lab03 &

# QEMU is a NORMAL userspace process. Find it:
$ pgrep -a qemu-system-x86
12345 qemu-system-x86_64 -accel kvm -m 512 -smp 2 -nographic -name lab03

# Each vCPU is a THREAD inside that one process. With -smp 2 you should see
# vCPU threads (named CPU 0/KVM, CPU 1/KVM) plus the main/IO thread:
$ ps -L -p 12345 -o pid,tid,comm
  PID   TID COMMAND
12345 12345 qemu-system-x86
12345 12346 qemu-system-x86      # main loop / IO thread
12345 12347 CPU 0/KVM            # vCPU 0  -> spins in ioctl(KVM_RUN)
12345 12348 CPU 1/KVM            # vCPU 1

# Prove the guest is using the kernel KVM device — QEMU has /dev/kvm open:
$ sudo ls -l /proc/12345/fd | grep kvm
lrwx------ 1 root root 64 Jun 20 09:10 8 -> /dev/kvm

# Clean up
$ kill 12345
```

**Expected result:** A single userspace QEMU process, with one thread per vCPU,
holding an open file descriptor to `/dev/kvm`. That is the entire architecture in
miniature: userspace device model (QEMU) + kernel execution engine (KVM).

---

## Further Reading

| Topic | Link |
|---|---|
| KVM API (KVM_RUN, ioctls) | [kernel.org — Virtual Machine API](https://docs.kernel.org/virt/kvm/api.html) |
| QEMU project | [Wikipedia — QEMU](https://en.wikipedia.org/wiki/QEMU) |
| Firecracker microVM | [Wikipedia — Firecracker (software)](https://en.wikipedia.org/wiki/Firecracker_(software)) |
| `ioctl` system call | [man7.org — ioctl(2)](https://man7.org/linux/man-pages/man2/ioctl.2.html) |
| KVM host documentation | [kernel.org — KVM](https://docs.kernel.org/virt/kvm/index.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. When a guest writes to an emulated NIC register, where does that write get handled — kernel or userspace? What about a guest page fault on RAM that is backed by host memory?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The NIC register write is handled in <strong>userspace by QEMU</strong>: the emulated NIC is a software device QEMU models, so the access causes a VM exit (KVM_EXIT_IO or KVM_EXIT_MMIO) that KVM hands up to QEMU to emulate. The page fault on backed RAM is handled <strong>in the kernel by KVM</strong>, which fixes up the second-level page tables (EPT/NPT) and resumes the guest without ever returning to userspace — far cheaper.
</details>

---

**Q2. Describe the KVM_RUN loop in your own words. What is QEMU doing while the guest is executing?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A per-vCPU QEMU thread calls ioctl(KVM_RUN), which transfers the physical CPU into guest mode; the guest's instructions then run natively on the core. While that happens, the QEMU thread is blocked inside the ioctl — it is not running. When the guest does something needing host help (I/O, MMIO, HLT, unmapped page), a VM exit occurs and KVM_RUN returns; QEMU inspects exit_reason, services it, and calls KVM_RUN again to resume. It repeats forever.
</details>

---

**Q3. Why is the work split between kernel (KVM) and userspace (QEMU) instead of putting everything in one place?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Putting everything in the kernel would mean a huge, complex device model in kernel space — hard to maintain and a large attack surface where one bug compromises the whole host. Putting everything in userspace would require software CPU emulation, which is slow. The split keeps the performance-critical, security-sensitive CPU/memory execution in the kernel (fast, hardware-accelerated) and the large, evolving device model in a sandboxable userspace process, so a device bug or crash is contained to one VM.
</details>

---

## Homework

Start a guest with `-smp 4` and use `ps -L` (as in the lab) to count the threads. Then identify which single thread would be blocked inside `ioctl(KVM_RUN)` the *most* if the guest were running a pure CPU benchmark with no I/O, and which thread would be busiest if the guest were hammering an emulated (non-virtio) NIC. Explain.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With -smp 4 you'd see four "CPU N/KVM" vCPU threads plus a main/IO thread. During a pure CPU benchmark, all four vCPU threads stay inside ioctl(KVM_RUN) almost continuously (the guest runs natively with almost no exits) — that's ideal. When the guest hammers an emulated NIC, each register access forces a VM exit back to QEMU, and the QEMU main/IO thread (the device-model thread) becomes busy emulating the NIC, while the vCPU threads keep exiting and re-entering — lots of context switching, which is exactly why virtio/vhost exist to cut those exits.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 04 — Rings, Privileged Instructions, and Trap-and-Emulate →](lesson-04-rings-trap-emulate){: .btn .btn-primary }
