---
title: "Lesson 02 — Hypervisor Types and Where KVM Fits"
nav_order: 2
parent: "Phase 1: Virtualization Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 02: Hypervisor Types and Where KVM Fits

## Concept

A **hypervisor** (or Virtual Machine Monitor, VMM) is the software layer that
creates and runs virtual machines. The textbook taxonomy splits hypervisors into
two types based on what sits between them and the hardware.

```
        TYPE 1 (bare-metal)                 TYPE 2 (hosted)
   ┌───────────────────────────┐      ┌───────────────────────────┐
   │  VM   VM   VM              │      │  VM   VM                   │
   ├───────────────────────────┤      ├──────────┬────────────────┤
   │      HYPERVISOR            │      │  VMM app │ other host apps│
   │  (ESXi, Xen, Hyper-V)      │      ├──────────┴────────────────┤
   ├───────────────────────────┤      │      HOST OS (Windows/Mac) │
   │        HARDWARE            │      ├───────────────────────────┤
   └───────────────────────────┘      │        HARDWARE            │
                                       └───────────────────────────┘
   hypervisor runs DIRECTLY            VMM is an APPLICATION on top
   on hardware                         of a normal OS
```

- **Type 1 (bare-metal):** the hypervisor *is* the lowest software layer, running
  directly on hardware. Examples: VMware ESXi, Xen, Microsoft Hyper-V.
- **Type 2 (hosted):** the hypervisor is an ordinary application running on a
  general-purpose host OS. Examples: VirtualBox, VMware Workstation.

**KVM breaks this neat dichotomy** — and understanding *why* is the point of this
lesson.

---

## How It Works

### KVM turns Linux itself into the hypervisor

KVM (Kernel-based Virtual Machine) is not a separate product that sits under or
over Linux — it is a **kernel module** that gives the running Linux kernel the
ability to act as a hypervisor. Once `kvm.ko` and `kvm_intel.ko`/`kvm_amd.ko` are
loaded, the Linux kernel can use the CPU's virtualization extensions to run guest
code with hardware isolation.

```
   ┌─────────────────────────────────────────────┐
   │  guest VM (vCPU threads)   normal processes  │
   │       │  via QEMU                │            │
   ├───────┼─────────────────────────┼────────────┤
   │   ┌───▼──── KVM module ────┐     │            │   ← Linux KERNEL
   │   │ uses VT-x/AMD-V        │  scheduler, fs…  │
   ├───┴───────────────────────┴──────────────────┤
   │                 HARDWARE                       │
   └───────────────────────────────────────────────┘
```

This is why KVM is *arguable both ways*:

- It looks **Type 2**, because the Linux kernel is a full general-purpose OS that
  also runs normal applications, and each VM is backed by a QEMU *process* that
  the Linux scheduler treats like any other.
- It behaves **Type 1**, because the part that actually runs guest code — the KVM
  module — lives *inside the kernel*, the lowest software layer, with direct access
  to the hardware virtualization extensions. There is no second OS underneath it.

The most accurate description: **KVM converts the Linux kernel into a Type 1
hypervisor.** A guest's vCPU runs guest code directly via the kernel; there is no
intermediary OS layer the way a true Type 2 product runs on top of Windows.

### Where QEMU fits

KVM alone only runs *CPU* code. It does not emulate a disk, a NIC, a BIOS, or a
display. That device model lives in userspace — almost always **QEMU**. So a
running KVM VM is the pair *(QEMU userspace process + KVM kernel acceleration)*.
Lesson 3 is dedicated to this division.

### The competitive landscape (high level)

| Hypervisor | Type | Notes |
|---|---|---|
| **KVM** | Type 1-ish (in-kernel) | Linux kernel module + userspace VMM (QEMU). The Linux default. |
| **Xen** | Type 1 | A dedicated hypervisor that boots first; Linux runs as a privileged "dom0" guest. |
| **VMware ESXi** | Type 1 | Proprietary bare-metal hypervisor with its own minimal kernel (vmkernel). |
| **Microsoft Hyper-V** | Type 1 | Even when "on Windows," Windows becomes a parent partition above the hypervisor. |
| **VirtualBox / VMware Workstation** | Type 2 | Classic hosted apps on a general-purpose desktop OS. |

---

## Lab

```bash
# Confirm the KVM modules that turn THIS kernel into a hypervisor are loaded:
$ lsmod | grep kvm
kvm_intel             380928  0
kvm                  1142784  1 kvm_intel
# (on AMD you'd see kvm_amd instead of kvm_intel)

# KVM exposes itself to userspace as a single character device.
# Its mere existence is what lets QEMU ask the kernel to run guest code:
$ ls -l /dev/kvm
crw-rw----+ 1 root kvm 10, 232 Jun 20 09:00 /dev/kvm

# Notice the group is 'kvm' — being in that group is how a normal user
# (and the libvirt qemu user later) gets to use hardware virtualization.
$ getent group kvm
kvm:x:36:

# There is no separate "hypervisor" process or OS — it's just the kernel.
# Compare: a true Type 2 product would show a big userspace daemon doing the work.
```

**Expected result:** The `kvm` and `kvm_intel`/`kvm_amd` modules are loaded into
the *running Linux kernel*, and `/dev/kvm` exists — proof that the kernel itself
has become the hypervisor, with no extra OS layer beneath it.

---

## Further Reading

| Topic | Link |
|---|---|
| Hypervisor (Type 1 / Type 2) | [Wikipedia — Hypervisor](https://en.wikipedia.org/wiki/Hypervisor) |
| Kernel-based Virtual Machine | [Wikipedia — Kernel-based Virtual Machine](https://en.wikipedia.org/wiki/Kernel-based_Virtual_Machine) |
| Xen | [Wikipedia — Xen](https://en.wikipedia.org/wiki/Xen) |
| Hyper-V | [Wikipedia — Hyper-V](https://en.wikipedia.org/wiki/Hyper-V) |
| KVM documentation | [linux-kvm.org](https://www.linux-kvm.org/page/Main_Page) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Is KVM a Type 1 or Type 2 hypervisor? Defend your answer — why is it arguable both ways?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It is best described as Type 1, because the component that runs guest code is a kernel module living in the lowest software layer with direct hardware access — there is no separate host OS beneath it. It is arguable as Type 2 because the Linux kernel it lives in is a full general-purpose OS that also runs ordinary applications, and each VM is driven by a userspace QEMU process scheduled like any other. The cleanest statement: KVM turns the Linux kernel into a Type 1 hypervisor.
</details>

---

**Q2. What does KVM provide, and what does it NOT provide (forcing you to also use QEMU)?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
KVM provides CPU and memory virtualization: it uses VT-x/AMD-V to run guest instructions natively and to isolate guest memory. It does NOT provide a device model — no emulated disk, NIC, BIOS, interrupt controller details, or display. Those are supplied by a userspace VMM, almost always QEMU, which is why a real KVM VM is the pair of a QEMU process plus KVM acceleration.
</details>

---

**Q3. How does Xen's design differ from KVM's in terms of what boots first?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With Xen, the Xen hypervisor itself boots first as the lowest layer; Linux then runs as a privileged guest called dom0 that handles drivers and management. With KVM, Linux boots normally as the host OS and *becomes* the hypervisor when the KVM modules are loaded — there is no separate hypervisor binary underneath Linux. Xen is a dedicated bare-metal hypervisor; KVM is Linux-with-a-module.
</details>

---

## Homework

On your host, run `systemd-detect-virt` (it reports if the machine is itself a VM and what hypervisor). Then explain: if your "host" is actually a VM, what must be enabled for KVM to still work inside it (one term), and which lesson covers that.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
If your host is itself a VM, you need <strong>nested virtualization</strong> enabled on the outer hypervisor so that the CPU's virtualization extensions (vmx/svm) are exposed to your VM. Without that, /dev/kvm won't be usable inside the VM and QEMU falls back to slow TCG emulation. Nested virtualization is covered in Lesson 55.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 03 — The QEMU + KVM Division of Labor →](lesson-03-qemu-kvm-division){: .btn .btn-primary }
