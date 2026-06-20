---
title: "Lesson 04 — Rings, Privileged Instructions, and Trap-and-Emulate"
nav_order: 4
parent: "Phase 2: CPU & Hardware Virtualization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 04: Rings, Privileged Instructions, and Trap-and-Emulate

## Concept

The x86 CPU enforces privilege using **protection rings** — concentric levels of
trust. There are four (0–3), but in practice only two are used:

```
        ┌───────────────────────────────────┐
        │  Ring 3  — user mode              │   ← your apps (browser, bash)
        │   ┌─────────────────────────────┐ │
        │   │ Ring 2  (unused)            │ │
        │   │  ┌───────────────────────┐  │ │
        │   │  │ Ring 1 (unused)       │  │ │
        │   │  │   ┌─────────────────┐ │  │ │
        │   │  │   │ Ring 0 — kernel │ │  │ │   ← the OS kernel
        │   │  │   │  (full power)   │ │  │ │
        │   │  │   └─────────────────┘ │  │ │
        │   │  └───────────────────────┘  │ │
        │   └─────────────────────────────┘ │
        └───────────────────────────────────┘
```

Ring 0 can do anything: change page tables, disable interrupts, talk to hardware.
Ring 3 cannot — if user code tries a privileged instruction, the CPU **traps**
(raises a fault) into ring 0.

The whole problem of CPU virtualization: a **guest OS kernel expects to be in
ring 0**, but you cannot actually give it ring 0 — that's where the *host* kernel
(and the hypervisor) live. Two ring-0 kernels would fight over the real hardware.

---

## How It Works

### Trap-and-emulate, the classical technique

The elegant 1970s solution: run the guest kernel **deprivileged** (in a less
trusted ring), but make it *believe* it's in ring 0. Whenever it executes a
privileged instruction, the CPU traps into the hypervisor, which **emulates** the
instruction's effect on the guest's *virtual* hardware state, then resumes the
guest.

```
   guest kernel (deprivileged) executes  CLI  ("disable interrupts")
        │
        ▼  CPU traps — guest isn't really allowed to do this
   hypervisor: "guest wants interrupts off → set a flag in the guest's
                virtual CPU state, but keep REAL interrupts as I please"
        │
        ▼  resume guest; it thinks interrupts are off
```

For this to work, **every sensitive instruction must be privileged** — it must
trap when run outside ring 0. Popek & Goldberg (Lesson 1) proved this is the
requirement for a strictly virtualizable architecture.

### Why classic x86 broke it

x86 had ~**17 problematic instructions** that were *sensitive* (they read or
modified privileged state) but **did not trap** in user mode — they silently
succeeded with wrong results. The textbook example is `POPF`: it can change the
interrupt-enable flag (IF), but executed in ring 3 it just *ignores* the IF bit
instead of trapping. The hypervisor never gets a chance to emulate it, so the
guest's virtual interrupt state silently desynchronizes from reality.

Because trap-and-emulate was impossible, early hypervisors used heavy workarounds:

- **Binary translation** (VMware): scan guest kernel code at runtime and rewrite
  the dangerous instructions into safe sequences that *do* trap. Clever, complex,
  and with overhead.
- **Ring deprivileging / paravirtualization** (Xen): modify the guest kernel to
  run in ring 1 and replace privileged operations with explicit *hypercalls* into
  the hypervisor. Fast, but needs a modified guest.

Hardware extensions (Lesson 5) made both workarounds unnecessary by adding a *new*
mode that gives the guest a real, safe ring 0.

{: .note }
> **Why not just put the guest kernel in ring 0 alongside the host?**
> Because there is only one set of real ring-0 hardware state — one real page-table
> register (CR3), one real interrupt flag, one real hardware. If the guest kernel
> ran in ring 0 it could reprogram the MMU, mask interrupts, or touch devices and
> instantly corrupt or hang the host and every other VM. Isolation requires that the
> guest *never* truly holds ring-0 power over the physical machine.

---

## Lab

```bash
# You can SEE the current privilege model of your CPU. x86_64 long mode still
# exposes the classic ring concept. Inspect CPU capabilities:
$ lscpu | grep -iE 'virtualization|byte order|architecture'
Architecture:            x86_64
Byte Order:              Little Endian
Virtualization:          VT-x

# The 'Virtualization: VT-x' line is the hardware feature (Lesson 5) that
# REPLACED the trap-and-emulate workarounds described above.

# Look at the raw CPU flags. The classic technique needed no flag; the modern
# hardware-assisted technique shows up as vmx (Intel) or svm (AMD):
$ grep -m1 -oE 'vmx|svm' /proc/cpuinfo
vmx

# Historical note you can observe: QEMU's pure-software engine (TCG) is the
# spiritual successor to binary translation — it translates guest code in SW.
# Force it and you are essentially back in the pre-hardware world (and slow):
$ qemu-system-x86_64 -accel tcg -m 256 -nographic -name no_hw_assist &
$ kill %1   # it works without vmx/svm, but every privileged op is software-handled
```

**Expected result:** Your CPU reports `VT-x` (or `AMD-V`), the hardware feature
that obsoleted binary translation and ring-deprivileging by giving guests a safe
ring 0. The TCG path shows virtualization is *possible* without it — just slow.

---

## Further Reading

| Topic | Link |
|---|---|
| Protection ring | [Wikipedia — Protection ring](https://en.wikipedia.org/wiki/Protection_ring) |
| x86 virtualization (the 17 instructions, binary translation) | [Wikipedia — x86 virtualization](https://en.wikipedia.org/wiki/X86_virtualization) |
| Popek & Goldberg requirements | [Wikipedia — Popek and Goldberg virtualization requirements](https://en.wikipedia.org/wiki/Popek_and_Goldberg_virtualization_requirements) |
| Hypercall / paravirtualization | [Wikipedia — Paravirtualization](https://en.wikipedia.org/wiki/Paravirtualization) |
| `lscpu` | [man7.org — lscpu(1)](https://man7.org/linux/man-pages/man1/lscpu.1.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why can't you simply run a guest OS kernel directly in ring 0 alongside the host kernel?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because ring 0 controls the single set of real hardware state — the MMU/page-table register, the interrupt flag, device access. There is only one physical machine. If a guest kernel ran in true ring 0 it could reprogram the MMU, disable real interrupts, or drive devices directly, corrupting or hanging the host and all other VMs. Isolation requires the guest never holds genuine ring-0 power over the physical hardware; it must be deprivileged and have its privileged operations mediated.
</details>

---

**Q2. What is "trap-and-emulate," and what architectural property must hold for it to work?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Trap-and-emulate runs the guest kernel deprivileged; when it executes a privileged instruction the CPU traps into the hypervisor, which emulates the instruction's effect on the guest's virtual state and then resumes the guest. For it to work, every sensitive instruction (one touching privileged state) must also be privileged — i.e. it must trap when executed outside ring 0. That is exactly the Popek & Goldberg condition.
</details>

---

**Q3. Classic x86 had instructions like POPF that were sensitive but didn't trap in user mode. Why was that fatal for trap-and-emulate, and name the two historical workarounds.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
If a sensitive instruction doesn't trap, the hypervisor never gets control to emulate it; the instruction silently does the wrong thing (e.g. POPF ignoring the interrupt flag in ring 3), desynchronizing the guest's virtual state from what the hypervisor thinks. Trap-and-emulate relies on the trap, so it simply can't be used. The two workarounds were (1) binary translation — rewriting dangerous guest code at runtime (VMware), and (2) paravirtualization / ring deprivileging — modifying the guest kernel to use hypercalls (Xen).
</details>

---

## Homework

Explain in a short paragraph why paravirtualization (modifying the guest to use hypercalls) was *faster* than binary translation, yet both were ultimately replaced by hardware virtualization. What does the hardware add that neither software approach had?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Paravirtualization is faster because the guest cooperatively calls the hypervisor exactly where needed (explicit hypercalls), avoiding the cost of scanning and rewriting code and avoiding traps on instructions that don't matter — but it requires a modified guest kernel. Binary translation runs unmodified guests but pays runtime overhead to detect and rewrite dangerous instructions. Hardware virtualization (VT-x/AMD-V) adds a brand-new CPU mode that gives the guest a genuine, safe ring 0 with automatic, low-overhead VM exits on the operations the hypervisor must see — so it runs unmodified guests AND avoids both software techniques' overhead.
</details>
