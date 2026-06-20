---
title: "Lesson 01 — What Virtualization Actually Is"
nav_order: 1
parent: "Phase 1: Virtualization Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 01: What Virtualization Actually Is

## Concept

When a program runs on your computer, its instructions execute directly on the
physical CPU. **Virtualization** is the art of convincing an entire operating
system that it owns a computer, when in fact it is a guest sharing real hardware
with the host and possibly many other guests.

The single question that separates the three core techniques is:
**who actually executes the guest's instructions?**

```
 EMULATION                  VIRTUALIZATION              PARAVIRTUALIZATION
 (e.g. QEMU TCG)            (e.g. KVM)                  (e.g. virtio, Xen PV)

 guest instr               guest instr                 guest instr
     │                         │                            │
 [ translate each ]        [ run NATIVELY on the ]      [ run natively, BUT the ]
 [ guest instr into ]      [ real CPU using HW   ]      [ guest KNOWS it is a   ]
 [ host instrs in SW ]     [ virt extensions     ]      [ VM and asks the host  ]
     │                         │                        [ politely for I/O      ]
 host CPU                  host CPU                          │
                                                        host CPU

 SLOW, but can run         FAST, same CPU arch         FAST I/O, needs a
 a DIFFERENT CPU arch      only                        cooperative guest
```

- **Emulation** — every guest instruction is *interpreted/translated* in software.
  This lets you run an ARM or RISC-V binary on an x86 host, because nothing assumes
  the host and guest share an instruction set. The price is speed.
- **(Full) virtualization** — the guest's instructions run *directly* on the
  physical CPU. This only works when guest and host share the same architecture
  (x86 guest on x86 host). Hardware extensions make it safe.
- **Paravirtualization** — the guest is *modified* to be virtualization-aware.
  Instead of pretending a fake disk is real hardware, the guest uses a special
  driver that talks to the host efficiently. This is what **virtio** is (Phase 7).

---

## How It Works

### Full virtualization vs paravirtualization

In **full virtualization** the guest OS is *unmodified* — it boots exactly as it
would on bare metal and has no idea it is virtualized. The hypervisor must present
convincing fake hardware (a fake disk controller, a fake NIC) and intercept the
guest's attempts to touch it.

In **paravirtualization** the guest is *aware* and runs special drivers. There is
no pretending: the guest says "please send this network buffer" through an
efficient shared-memory channel instead of poking emulated NIC registers one byte
at a time. Modern KVM guests are a hybrid: full virtualization of the CPU
(unmodified kernel) + paravirtualized devices (virtio) for speed.

### Why x86 was historically hard to virtualize

In 1974 **Popek and Goldberg** formalized when an architecture is "virtualizable":
every *sensitive* instruction (one that touches privileged machine state) must
also be *privileged* (must trap when run outside the most-privileged mode). If
that holds, you can run the guest deprivileged and simply trap-and-emulate every
sensitive instruction.

Classic x86 *violated* this rule: it had ~17 instructions that were sensitive but
**did not trap** when run in user mode — they just silently did the wrong thing.
That made naïve trap-and-emulate impossible, and forced early VMware to use
elaborate **binary translation**. Hardware extensions (VT-x/AMD-V, Lesson 5)
finally fixed this in 2005–2006, which is what made KVM possible.

{: .note }
> **Trap-and-emulate**
> Run the guest in a low-privilege mode. When it executes a privileged
> instruction (like "disable interrupts"), the CPU *traps* — control jumps to the
> hypervisor, which emulates the intended effect on the guest's *virtual* state,
> then resumes the guest. The guest never knows. This is the foundation of
> classical virtualization, covered in depth in Lesson 4.

---

## Lab

This lesson is conceptual, but you can *see* the emulation-vs-virtualization split
with QEMU itself. (Install QEMU first; on Debian/Ubuntu: `sudo apt install qemu-system qemu-utils`.)

```bash
# QEMU ships a separate binary PER target architecture.
# This one emulates a full ARM 64-bit machine:
$ which qemu-system-aarch64
/usr/bin/qemu-system-aarch64

# This one is for x86-64 guests:
$ which qemu-system-x86_64
/usr/bin/qemu-system-x86_64

# On an x86 host, qemu-system-aarch64 MUST emulate (TCG) — there is no ARM CPU
# to run guest code on. qemu-system-x86_64 CAN use KVM to run guest code natively.

# Check whether your CPU has the hardware virtualization extensions that make
# native (KVM) execution possible:
$ grep -oE 'vmx|svm' /proc/cpuinfo | sort -u
vmx        # Intel VT-x present  (you may see 'svm' on AMD)

# If that printed vmx or svm, an x86 guest can run NATIVELY via KVM.
# An ARM guest on this same host would still have to be EMULATED.
```

**Expected result:** Your host has *per-architecture* QEMU binaries, and (on most
modern CPUs) the `vmx` or `svm` flag — meaning same-arch guests get native speed
while different-arch guests fall back to slow emulation.

---

## Further Reading

| Topic | Link |
|---|---|
| Virtualization (overview) | [Wikipedia — Virtualization](https://en.wikipedia.org/wiki/Virtualization) |
| Full virtualization | [Wikipedia — Full virtualization](https://en.wikipedia.org/wiki/Full_virtualization) |
| Paravirtualization | [Wikipedia — Paravirtualization](https://en.wikipedia.org/wiki/Paravirtualization) |
| Popek & Goldberg requirements | [Wikipedia — Popek and Goldberg virtualization requirements](https://en.wikipedia.org/wiki/Popek_and_Goldberg_virtualization_requirements) |
| Emulator vs hypervisor | [Wikipedia — Emulator](https://en.wikipedia.org/wiki/Emulator) |
| QEMU project | [Wikipedia — QEMU](https://en.wikipedia.org/wiki/QEMU) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Explain the difference between QEMU emulating an ARM CPU on an x86 host and KVM running an x86 guest on an x86 host. Which one actually runs guest instructions natively on the physical CPU?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
KVM runs guest instructions natively. When KVM runs an x86 guest on an x86 host, the guest's machine code executes directly on the physical CPU using the hardware virtualization extensions — no translation. When QEMU emulates an ARM CPU on x86, there is no ARM hardware available, so QEMU's TCG engine must translate every ARM instruction into equivalent x86 instructions in software. That translation is why emulation is far slower than native virtualization.
</details>

---

**Q2. Why is paravirtualization (e.g. virtio) faster for I/O than fully emulating a real network card?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A fully emulated NIC forces the guest to poke fake hardware registers one access at a time, and each access traps into the hypervisor to be emulated — many expensive context switches per packet. Paravirtualization replaces that with a cooperative driver that batches work through a shared-memory ring, dramatically reducing the number of traps/exits and copies. The guest gives up "I don't know I'm virtualized" in exchange for speed.
</details>

---

**Q3. Classic x86 had instructions that were "sensitive but not privileged." Why did that make simple trap-and-emulate impossible?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Trap-and-emulate depends on every sensitive instruction trapping when executed in a deprivileged mode, so the hypervisor can intercept and emulate it. On classic x86, certain sensitive instructions (e.g. POPF) silently did nothing or behaved differently in user mode instead of trapping. The hypervisor never got control, so it couldn't correct the guest's view of machine state. This violated the Popek & Goldberg requirement and forced workarounds like binary translation until hardware extensions added a proper trap mechanism.
</details>

---

## Homework

Run `qemu-system-aarch64 -M virt -cpu cortex-a57 -nographic -kernel /dev/null 2>&1 | head` on your x86 host (it will fail to boot a real kernel, but it *will* start). Then run the same idea with `qemu-system-x86_64 -accel kvm -nographic 2>&1 | head`. Note which one needed `-accel kvm` to be meaningful and explain, in one sentence, why the aarch64 invocation can never use `-accel kvm` on this host.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The aarch64 invocation can never use KVM on an x86 host because KVM only accelerates guests whose architecture matches the host CPU — there is no ARM silicon present to execute ARM guest instructions, so QEMU must fall back to TCG software emulation. The x86_64 invocation can use `-accel kvm` because the guest and host share the x86 instruction set, so guest code can run directly on the physical CPU via VT-x/AMD-V.
</details>
