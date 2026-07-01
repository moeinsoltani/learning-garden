---
title: "Lesson 10 — The VM-Exit Loop in Depth"
nav_order: 10
parent: "Phase 3: KVM Internals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 10: The VM-Exit Loop in Depth

## Concept

The **VM-exit loop** is the single most important control flow in the entire stack.
Performance, in virtualization, is almost entirely the story of *how often you exit
the guest* and *who handles the exit*. This lesson dissects it.

A VM exit is a hardware "world switch" from guest (non-root) mode back to the
hypervisor (root) mode. It is **expensive** — saving/restoring CPU state plus
cache/TLB effects cost thousands of cycles. Every exit is a tax. The art is making
the guest run as long as possible *between* exits, and handling unavoidable exits
as close to the hardware as possible.

```
        guest running natively (NO exits = best)
                  │
        ┌─────────▼──────────────────────────────────┐
        │  something forces a VM EXIT                 │
        │  KVM reads the exit reason...               │
        └─────────┬──────────────────────────┬───────┘
                  │ handled IN KERNEL          │ bounced to USERSPACE
                  ▼ (cheap, fast)              ▼ (expensive: extra switch)
        EPT fixup, in-kernel irqchip,    KVM_EXIT_IO / KVM_EXIT_MMIO
        HLT with halt-polling            → return to QEMU device model
                  │                              │
                  └──────────► resume guest ◄────┘
```

---

## How It Works

### Exit reasons and where they go

| Exit reason | Typically handled | Notes |
|---|---|---|
| **EPT/NPT violation** | KVM (kernel) | Back guest page with host memory, update SLAT, resume. No userspace. |
| **HLT** | KVM (kernel) | Guest idled. With halt-polling, KVM may briefly spin before sleeping. |
| **External interrupt** | KVM (kernel) | A host IRQ arrived; KVM handles and may re-enter quickly. |
| **Interrupt window / irqchip** | KVM (kernel) | In-kernel interrupt controller injects guest IRQs without userspace. |
| **I/O port (PIO)** | **QEMU (userspace)** | Guest `in`/`out` to an emulated device register → KVM_EXIT_IO. |
| **MMIO** | **QEMU (userspace)** | Guest read/write to a memory-mapped device region → KVM_EXIT_MMIO. |
| **Hypercall** | KVM or QEMU | Paravirtual call; some handled in-kernel. |
| **CPUID / MSR** | KVM (kernel) | KVM filters/serves feature queries. |

The key dividing line: **memory and CPU-state exits stay in the kernel; device I/O
to QEMU-modeled hardware goes up to userspace.** A userspace exit costs *more* — it
adds a kernel→userspace→kernel round trip on top of the world switch.

### In-kernel acceleration: irqchip and PIT

Early KVM bounced even interrupt-controller and timer accesses to QEMU, which was
slow (interrupts happen constantly). The fix: KVM gained an **in-kernel irqchip**
(emulated local APIC / IOAPIC) and **in-kernel PIT** (timer). Now the most
frequent, latency-critical device interactions never leave the kernel. This is why
modern KVM is fast where naïve full emulation is not — and it foreshadows virtio
and vhost (Phase 7), which push *more* of the datapath into the kernel.

### Why reducing exits is the whole game

Consider a guest sending 1 Gbit/s over an emulated e1000 NIC: each packet involves
many register pokes, each a PIO/MMIO exit to QEMU — tens of thousands of expensive
userspace round trips per second. Replace it with **virtio-net + vhost** and the
packets move through a shared ring handled in the kernel: orders of magnitude fewer
exits. Every performance technique in this track — virtio, vhost, in-kernel
irqchip, halt-polling, hugepages — is ultimately *exit reduction*.

{: .note }
> **Why a tight CPU loop is "good"**
> A guest doing pure computation (no I/O, working set already mapped) generates
> almost no exits — it runs natively at full speed, and KVM_RUN simply doesn't
> return. That's the ideal. What *forces* exits is interaction with the outside
> world: device I/O, new memory pages, idling (HLT), interrupts, timer ticks. So a
> "chatty" guest (lots of small I/O) is inherently more expensive to virtualize than
> a compute-bound one.

---

## Lab

```bash
# kvm_stat shows exits BY REASON, live. Install it: sudo apt install kvm (or it
# ships with qemu-kvm). Start a guest first, then watch.

$ qemu-system-x86_64 -accel kvm -m 1G -smp 2 -nographic \
    -drive file=guest.qcow2,if=virtio &

# 1. Live view of exit reasons across all VMs:
$ sudo kvm_stat
kvm statistics - summary

 Event                         Total   %Total  CurAvg/s
 kvm_exit                      54213    100.0      1832
 kvm_entry                     54190     ...        1831
 kvm_userspace_exit            1204      ...          12   ← went up to QEMU
 kvm_mmio                       402      ...           4
 kvm_pio                        110      ...           1
 kvm_halt_poll_*                ...
 ... (press q to quit)

# 2. Snapshot once and exit (good for scripting/diffing):
$ sudo kvm_stat -1 | head -20

# 3. Now generate I/O inside the guest (e.g. run `dd` or `ping` in the guest)
#    and re-watch: kvm_mmio / kvm_pio / userspace_exit climb. Compare an
#    idle guest (mostly HLT exits) to a busy one (I/O exits dominate).
```

**Expected result:** `kvm_stat` shows the exit-reason breakdown. An idle guest is
dominated by HLT/halt-poll; a guest doing I/O shows climbing MMIO/PIO and
userspace exits — making the kernel-vs-userspace split from this lesson visible.

---

## Further Reading

| Topic | Link |
|---|---|
| KVM API (exit reasons, kvm_run) | [kernel.org — KVM API](https://docs.kernel.org/virt/kvm/api.html) |
| In-kernel irqchip / APIC | [kernel.org — KVM locking & devices](https://docs.kernel.org/virt/kvm/index.html) |
| `kvm_stat` | [linux-kvm.org — kvm_stat](https://www.linux-kvm.org/page/Tuning_KVM) |
| World switch / context switch cost | [Wikipedia — Context switch](https://en.wikipedia.org/wiki/Context_switch) |
| virtio (exit reduction) | [Wikipedia — Virtio](https://en.wikipedia.org/wiki/Virtio) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. A guest executing a tight CPU loop with no I/O causes almost no VM exits. Why is that good, and what kinds of guest behavior *force* exits?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It's good because each VM exit is an expensive world switch (thousands of cycles plus cache/TLB effects); with no exits, the guest runs natively at near bare-metal speed and KVM_RUN never returns. Exits are forced by interaction with the outside world: device I/O (PIO/MMIO to emulated hardware), EPT violations when touching unmapped guest-physical pages, HLT when idling, external interrupts, timer ticks, and CPUID/MSR accesses. A compute-bound guest exits rarely; a chatty I/O-bound guest exits constantly.
</details>

---

**Q2. Which is more expensive — an exit KVM handles in-kernel (e.g. EPT violation) or one it bounces to QEMU (e.g. KVM_EXIT_IO)? Why?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The userspace exit (KVM_EXIT_IO/MMIO to QEMU) is more expensive. Both pay the hardware world-switch cost from guest to hypervisor, but the in-kernel exit is then resolved within the kernel and the guest resumed. The userspace exit additionally requires returning from KVM_RUN out to the QEMU process, running the device-model code there, and re-entering the kernel and the guest — extra context switches and cache effects on top of the world switch.
</details>

---

**Q3. You see a huge count of MMIO exits. What does that suggest about the guest's device configuration, and what would you change?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A flood of MMIO exits suggests the guest is hammering an emulated hardware device whose registers are memory-mapped — typically a fully emulated NIC (e1000) or disk controller — so every register access traps to QEMU. The fix is to switch the guest to paravirtualized virtio devices (virtio-net, virtio-blk/scsi), which use shared-memory rings instead of register pokes, and ideally enable vhost so the datapath stays in the kernel. That replaces tens of thousands of MMIO userspace exits with far fewer, cheaper notifications.
</details>

---

## Homework

Run an idle guest and capture `kvm_stat -1`. Then, inside the guest, run a network ping flood or `dd if=/dev/zero of=/tmp/f bs=1M count=500` and capture `kvm_stat -1` again. Compare the two snapshots and identify which exit-reason counters changed the most. Explain what each rising counter tells you about what the guest was doing.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
On the idle snapshot, HLT/halt-poll-related counters dominate (the guest mostly sleeps). After the disk/network load, expect kvm_mmio and/or kvm_pio and kvm_userspace_exit to climb sharply, along with more kvm_exit/kvm_entry overall — the guest is doing device I/O that traps to QEMU's device model. If you used virtio devices the rise is smaller (shared-ring notifications) than with emulated devices (per-register exits). Rising MMIO/PIO/userspace_exit counters indicate device interaction; rising EPT-violation counters would indicate the workload touching lots of newly-allocated guest memory (e.g. the dd buffer growing the working set).
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 11 — Observing KVM at Runtime →](lesson-11-observing-kvm){: .btn .btn-primary }
