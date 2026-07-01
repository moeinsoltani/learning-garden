---
title: "Lesson 06 — Memory Virtualization: EPT / NPT (SLAT)"
nav_order: 6
parent: "Phase 2: CPU & Hardware Virtualization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 06: Memory Virtualization — EPT / NPT (SLAT)

## Concept

The CPU virtualization of Lesson 5 runs guest *instructions* natively. But a guest
also has *memory*, and here lies a deep problem: the guest OS builds its own page
tables believing it owns physical RAM starting at address 0 — but it doesn't. Its
"physical" addresses are themselves virtual from the host's point of view.

So there are now **three** address spaces, not two:

```
   guest virtual addr   guest physical addr      host physical addr
   (what guest apps     (what the guest THINKS    (real RAM the host
    see)                 is physical RAM)          actually gave it)
        │                      │                         │
        │  guest page tables   │   ??? who maps this ??? │
        └─────────►────────────┴──────────►──────────────┘
            GVA → GPA              GPA → HPA
        (guest controls this)   (host/hypervisor must control this)
```

The guest fully controls **GVA → GPA** (its own page tables). The hypervisor must
supply the second translation, **GPA → HPA**, because guest "physical" address 0
might really be host physical address 0x1_4000_0000. The hardware feature that does
this is **SLAT** — Second Level Address Translation — called **EPT** on Intel and
**NPT** on AMD.

---

## How It Works

### The old, slow way: shadow page tables

Before SLAT, the hypervisor maintained **shadow page tables**: a single merged set
of page tables mapping GVA directly to HPA, kept in sync with the guest's own page
tables. The catch: the hypervisor had to **trap every guest page-table edit** to
update the shadows. A guest doing lots of process creation / memory mapping caused
a storm of VM exits. Correct, but expensive and complex.

### SLAT: let the hardware walk both levels

EPT/NPT add a *second* set of page tables, managed by the hypervisor, that the MMU
consults automatically. On a memory access the CPU does a **two-dimensional page
walk**:

```
   guest app accesses GVA
        │
        ▼  (1) walk GUEST page tables  → GPA          ← guest-controlled
        │
        ▼  (2) walk EPT/NPT tables     → HPA          ← hypervisor-controlled
        │
        ▼  access real RAM at HPA
```

Now the guest can edit *its own* page tables freely — **no VM exit needed** —
because the guest tables only produce GPAs, and the EPT independently maps GPA→HPA.
The hypervisor only gets involved (an **EPT violation** exit) when the guest
touches a GPA that isn't yet backed by host memory, so KVM can allocate/map it.

### Why two walks is still a win

A two-dimensional walk is *more* steps per TLB miss than a normal one-dimensional
walk — yet it's dramatically faster overall than shadow page tables, because it
**eliminates the constant VM exits** on every guest page-table modification. The
guest's frequent memory-management activity now runs entirely natively. The extra
walk cost is paid only on TLB misses and is mitigated by hardware:

- **TLB** caches completed GVA→HPA translations, so the expensive two-dimensional
  walk happens only on a miss.
- **VPID** (Intel) / **ASID** (AMD) tag TLB entries with a VM identifier, so the
  TLB does **not** have to be flushed on every world switch — entries from
  different VMs (and the host) coexist.

{: .note }
> **The analogy to device DMA (preview of Phase 9)**
> EPT/NPT translate *CPU* accesses from guest-physical to host-physical. A device
> doing DMA also issues "physical" addresses — and a passed-through device would
> issue *guest*-physical addresses that mean nothing to real RAM. The **IOMMU**
> (Intel VT-d / AMD-Vi) does the analogous GPA→HPA translation for device DMA, which
> is why safe device passthrough needs it. Same idea, different initiator.

---

## Lab

```bash
# 1. Confirm your CPU supports SLAT. Intel calls it 'ept', AMD calls it 'npt':
$ grep -m1 -oE 'ept|npt' /proc/cpuinfo
ept

# 2. Confirm VPID (Intel) for tagged TLB entries — avoids full TLB flush per exit:
$ grep -m1 -o 'vpid' /proc/cpuinfo
vpid

# 3. EPT is controlled by a kvm_intel module parameter. 'Y' means enabled:
$ cat /sys/module/kvm_intel/parameters/ept
Y
# (on AMD: cat /sys/module/kvm_amd/parameters/npt)

# 4. Run a guest and watch which exits relate to memory. EPT violations show up
#    as a kvm_stat counter. Start a VM, then in another terminal:
$ sudo kvm_stat -1 2>/dev/null | grep -iE 'ept|nested|tlb' || \
    echo "run a VM, then re-run; look for 'ept_violation' style counters"
```

**Expected result:** `ept` (or `npt`) and `vpid` are present, and EPT is enabled in
the module. Confirming this is what guarantees your VMs use hardware second-level
translation instead of falling back to slow shadow page tables.

---

## Further Reading

| Topic | Link |
|---|---|
| Second Level Address Translation (SLAT) | [Wikipedia — SLAT](https://en.wikipedia.org/wiki/Second_Level_Address_Translation) |
| Memory management unit | [Wikipedia — Memory management unit](https://en.wikipedia.org/wiki/Memory_management_unit) |
| Translation lookaside buffer | [Wikipedia — Translation lookaside buffer](https://en.wikipedia.org/wiki/Translation_lookaside_buffer) |
| Shadow page tables | [Wikipedia — Shadow page tables](https://en.wikipedia.org/wiki/Shadow_page_tables) |
| IOMMU (the device analog) | [Wikipedia — IOMMU](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Name the three address spaces involved in guest memory, and say which translation the guest controls and which the hypervisor controls.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The three are guest virtual (GVA), guest physical (GPA), and host physical (HPA). The guest controls GVA→GPA via its own page tables. The hypervisor controls GPA→HPA, because the guest's "physical" memory is itself just host memory the hypervisor allocated; with hardware SLAT this mapping lives in the EPT/NPT tables managed by KVM.
</details>

---

**Q2. Why is hardware SLAT (EPT/NPT) faster than shadow page tables, even though it adds an extra translation step on each TLB miss?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Shadow page tables forced a VM exit on every guest page-table modification so the hypervisor could keep the merged shadow tables in sync — a constant stream of expensive world switches for memory-heavy guests. SLAT lets the guest edit its own page tables freely with no exits, because those tables only produce GPAs and the EPT independently maps GPA→HPA in hardware. The hypervisor is only invoked on an EPT violation (unbacked page). The extra walk step on a TLB miss is far cheaper than the eliminated exit storm, and the TLB plus VPID/ASID tagging keep the walk cost rare.
</details>

---

**Q3. What is an "EPT violation," and what does KVM typically do in response?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An EPT violation is a VM exit that occurs when the guest accesses a guest-physical address that the EPT doesn't currently map to host physical memory. KVM handles it by allocating/backing the corresponding host page and inserting the GPA→HPA mapping into the EPT, then resuming the guest. This is how guest RAM is lazily/demand-allocated on the host (it ties into memory backends and overcommit in Phase 5).
</details>

---

## Homework

VPID/ASID tag TLB entries with a per-VM identifier. Explain what would happen to performance *without* this tagging: specifically, what would the CPU be forced to do on every VM exit and VM entry, and why would that hurt a workload that switches frequently between guest and host (lots of I/O)?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Without VPID/ASID, TLB entries aren't tagged by VM, so on every world switch the CPU couldn't tell guest translations apart from host (or other VMs'). To stay correct it would have to flush the entire TLB on each VM exit and entry. A workload with frequent guest↔host switches (heavy I/O causing many exits) would then suffer a cold TLB after every switch, forcing expensive page-table walks to refill it — a large, repeated penalty. Tagging lets entries from the guest, host, and other VMs coexist in the TLB across switches, so no flush is needed and translations survive the round trip.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 07 — Checking and Enabling Virtualization on the Host →](lesson-07-host-readiness){: .btn .btn-primary }
