---
title: "Lesson 05 — Hardware Virtualization Extensions (VT-x / AMD-V)"
nav_order: 5
parent: "Phase 2: CPU & Hardware Virtualization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 05: Hardware Virtualization Extensions (VT-x / AMD-V)

## Concept

Lesson 4 ended with a problem: x86 couldn't be cleanly virtualized. The fix
(2005–2006) was to add a whole new *axis* of CPU operation. Instead of squeezing
the guest into a ring it doesn't fit, the CPU gained two **modes**:

```
                    ┌──────────────────────────────────────┐
   ROOT mode        │  hypervisor (KVM)                     │
   (VMX root /      │   rings 0-3 all available             │
    host)           └──────────────────┬───────────────────┘
                          VMLAUNCH /    │   ▲   VM EXIT
                          VMRESUME      ▼   │
                    ┌──────────────────────────────────────┐
   NON-ROOT mode    │  guest                                │
   (VMX non-root /  │   has its OWN full rings 0-3!         │
    guest)          │   guest kernel really runs in ring 0  │
                    └──────────────────────────────────────┘
```

The guest gets its **own complete set of rings** in non-root mode. Its kernel
*really does* run in ring 0 — but it's a *guest* ring 0 that cannot escape to the
physical machine. When the guest does something the hypervisor must mediate, the
CPU performs a **VM exit**, switching back to root mode where KVM runs.

This is "trap-and-emulate" done in hardware, with a much richer, configurable trap
mechanism — and it's exactly what KVM uses.

---

## How It Works

### The two vendor implementations

- **Intel VT-x** (a.k.a. VMX, "Virtual Machine Extensions") — CPU flag: `vmx`.
  Key instructions: `VMXON` (enable VMX), `VMLAUNCH`/`VMRESUME` (enter guest),
  and VM exits return to the hypervisor.
- **AMD-V** (a.k.a. SVM, "Secure Virtual Machine") — CPU flag: `svm`.
  Key instructions: `VMRUN` enters the guest.

KVM abstracts both: the generic `kvm.ko` plus a vendor module (`kvm_intel.ko` or
`kvm_amd.ko`, Lesson 9) that speaks the right dialect.

### The VMCS / VMCB — the per-vCPU control structure

The hardware needs somewhere to store "what is this guest, and what should make it
exit?" That is the **VMCS** (Virtual Machine Control Structure) on Intel, or the
**VMCB** (Virtual Machine Control Block) on AMD. There is one per virtual CPU. It
holds:

- **Guest state** — saved register values, RIP, control registers, etc., restored
  on VM entry and saved on VM exit.
- **Host state** — what to restore when control returns to the hypervisor.
- **Control fields** — *which guest activities cause a VM exit* (e.g. "exit on
  CPUID," "exit on access to CR3," "exit on this I/O port"). This is the knob that
  lets KVM choose how much to intercept.

```
   VM ENTRY (VMLAUNCH/VMRESUME)          VM EXIT
   ──────────────────────────────►   ◄──────────────────────────────
   load guest state from VMCS         save guest state to VMCS,
   run guest in non-root mode         load host state, set exit reason,
                                      hand control to KVM
```

### VM exits and exit reasons

A **VM exit** is the hardware transition from guest (non-root) back to hypervisor
(root). Each exit records an **exit reason** so KVM knows why it happened.
Common reasons:

| Exit reason | Triggered by |
|---|---|
| I/O instruction | guest `in`/`out` to a port (emulated device) |
| EPT violation | guest touched a guest-physical page not yet mapped (Lesson 6) |
| HLT | guest halted (idle) |
| CPUID | guest queried CPU features (KVM filters what it sees) |
| External interrupt | a host interrupt arrived while the guest was running |
| MSR read/write | guest touched a model-specific register KVM intercepts |

Exits are not free — each one costs thousands of cycles for the world switch.
Lesson 10 dissects the exit loop, and Lesson 11 shows you how to *count* exits.

{: .note }
> **"The guest gets its own ring 0" — why that's the breakthrough**
> Before VT-x, the hypervisor had to lie to the guest about its privilege and patch
> around non-trapping instructions. With VT-x, the guest kernel genuinely executes
> in ring 0 *of non-root mode*. Privileged instructions that the hypervisor cares
> about are configured (via the VMCS) to cause clean VM exits; everything else runs
> at full native speed. No binary translation, no guest modification.

---

## Lab

```bash
# 1. Confirm the hardware extension is present (vmx = Intel, svm = AMD):
$ grep -m1 -oE 'vmx|svm' /proc/cpuinfo
vmx

# 2. See the full set of virtualization-related CPU features the guest could use.
#    'ept' (Lesson 6) and 'vpid' show up here on Intel:
$ grep -m1 -oE 'vmx|ept|vpid|tpr_shadow|vnmi|flexpriority' /proc/cpuinfo
# (these appear in the flags line if your CPU supports them)
$ grep -o 'ept' /proc/cpuinfo | head -1
ept

# 3. The KVM module that drives the VMCS is loaded as the vendor module:
$ lsmod | grep -E 'kvm_intel|kvm_amd'
kvm_intel             380928  0

# 4. KVM exposes which exit reasons it has handled, per VM, via debugfs.
#    Mount it if needed, then watch exit-reason counters appear once a VM runs:
$ sudo mount -t debugfs none /sys/kernel/debug 2>/dev/null
$ ls /sys/kernel/debug/kvm/ 2>/dev/null || echo "(start a VM first; counters appear here)"
```

**Expected result:** The `vmx`/`svm` flag confirms VT-x/AMD-V; the `ept` flag
confirms second-level address translation (next lesson); and the vendor KVM module
is loaded, ready to manage a VMCS/VMCB per vCPU.

---

## Further Reading

| Topic | Link |
|---|---|
| x86 virtualization (VT-x / AMD-V) | [Wikipedia — x86 virtualization](https://en.wikipedia.org/wiki/X86_virtualization) |
| VT-x / VMX overview | [Wikipedia — VT-x](https://en.wikipedia.org/wiki/X86_virtualization#Intel_virtualization_(VT-x)) |
| Virtual Machine Control Structure | [Wikipedia — VMCS](https://en.wikipedia.org/wiki/Hardware-assisted_virtualization) |
| KVM API | [kernel.org — KVM API](https://docs.kernel.org/virt/kvm/api.html) |
| /proc/cpuinfo flags | [man7.org — proc(5)](https://man7.org/linux/man-pages/man5/proc.5.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What is a VMCS (or VMCB on AMD), and what three broad categories of information does it hold?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The VMCS (Intel) / VMCB (AMD) is a per-vCPU control structure the CPU uses to manage a guest. It holds (1) guest state — registers, RIP, control registers — loaded on VM entry and saved on VM exit; (2) host state — what to restore when control returns to the hypervisor; and (3) control fields that configure which guest activities cause a VM exit. The control fields are what let KVM decide how much of the guest's behavior to intercept.
</details>

---

**Q2. What kinds of guest activity cause a "VM exit" back to the hypervisor? Give three.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Examples: an I/O port access (in/out) to an emulated device; an EPT violation (touching a guest-physical page not yet backed by host memory); executing HLT to idle; a CPUID query (KVM filters reported features); accessing an intercepted MSR; or an external host interrupt arriving while the guest runs. Any of these transitions the CPU from non-root (guest) back to root (hypervisor) mode, recording an exit reason.
</details>

---

**Q3. How do VT-x/AMD-V give the guest "its own ring 0" safely, when Lesson 4 said two ring-0 kernels can't coexist?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
They add a new orthogonal mode (non-root / guest mode) on top of the ring system. The guest runs in non-root mode with its full set of rings 0–3, so its kernel really executes in ring 0 — but it's the ring 0 *of non-root mode*, which cannot touch the physical machine. The hypervisor runs in root mode. Operations the hypervisor must mediate are configured (via the VMCS) to trigger a clean VM exit back to root mode. So both kernels get a "ring 0," but only the host's ring 0 controls real hardware.
</details>

---

## Homework

Suppose a guest runs a tight loop computing prime numbers, never touching I/O. Using what you learned about VM exits and the VMCS, explain why this guest can run for long stretches with essentially zero VM exits, and why that is the best possible case for performance.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Pure arithmetic and memory access within already-mapped pages are not configured to cause VM exits — the guest's instructions run natively in non-root mode at full CPU speed, with the VMCS sitting idle. There's no I/O, no CPUID, no MSR access, no new page faults once the working set is mapped, so the CPU never needs to switch back to root mode. With near-zero exits there is no world-switch overhead, so the guest runs at close to bare-metal speed — which is exactly why minimizing exits is the core of performance tuning.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 06 — Memory Virtualization: EPT / NPT (SLAT) →](lesson-06-ept-npt-slat){: .btn .btn-primary }
