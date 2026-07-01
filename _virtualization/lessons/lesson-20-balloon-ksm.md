---
title: "Lesson 20 — Ballooning and KSM"
nav_order: 20
parent: "Phase 5: CPU & Memory Configuration"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 20: Ballooning and KSM

## Concept

Memory is the scarcest VM resource. Two mechanisms let a host *reclaim* and
*deduplicate* RAM across guests — squeezing more VMs onto less memory:

```
   BALLOONING — give memory back, cooperatively
   ┌──────────── guest RAM (4 GiB) ────────────┐
   │ used by guest │  balloon (inflated)         │   the host "inflates" a
   │               │ ░░░░ pinned, returned ░░░░  │   driver inside the guest
   └───────────────┴─────────────────────────────┘   that pins pages and hands
                          │ host reuses these pages   them back to the host

   KSM — Kernel Samepage Merging, dedupe across VMs
   VM-A page [ZEROS]  ┐
   VM-B page [ZEROS]  ├─► one shared copy-on-write page in host RAM
   VM-C page [ZEROS]  ┘
```

**Ballooning** moves the *boundary* of how much RAM a guest holds, with the guest's
cooperation. **KSM** finds *identical pages* across guests and collapses them into
one shared page.

---

## How It Works

### virtio-balloon

The **balloon** is a paravirtual device (`virtio-balloon`) with a driver inside the
guest. To reclaim memory, the host tells the balloon to **inflate**: the guest
driver allocates pages (pinning them so the guest won't use them) and tells the host
"these are free now." The host then unmaps those pages and reuses them for other
VMs or itself. To give memory back, the host **deflates** the balloon: the driver
releases the pages back to the guest.

```
   inflate  → guest driver grabs pages → host reclaims them   (guest has less)
   deflate  → host returns pages → guest driver frees them     (guest has more)
```

The critical word is **cooperative**: the host cannot just *take* a guest's memory —
the guest is using it and the host doesn't know which pages are free. The balloon
driver *inside* the guest is what knows, so it must run and respond. A guest without
the driver (or one ignoring the request) keeps all its memory.

**Pitfalls:** inflate too aggressively and the guest is starved → it swaps or
invokes its own OOM killer. The balloon has a configured maximum; you shrink a guest
only down to what it can tolerate.

### KSM — Kernel Samepage Merging

**KSM** is a *host* kernel feature. A daemon (`ksmd`) periodically scans memory
marked mergeable and finds pages with **identical content** across processes/VMs —
e.g. zeroed pages, identical OS code pages when many guests run the same distro. It
merges them into a single **copy-on-write** page: all the duplicates point at one
physical page, freeing the rest. If any guest *writes* to a merged page, COW makes a
private copy first, so correctness is preserved.

```
   /sys/kernel/mm/ksm/run          1 = active
   /sys/kernel/mm/ksm/pages_shared    how many merged pages exist
   /sys/kernel/mm/ksm/pages_sharing   how many pages point at them (savings)
```

**Trade-offs:**

- **CPU cost:** `ksmd` constantly scans and hashes pages — it spends CPU to save
  memory. Worthwhile when memory is the bottleneck and guests are similar (same OS);
  wasteful when guests are diverse or CPU is scarce.
- **Security caveat:** because merging timing leaks whether a page already existed
  elsewhere, KSM enables **side-channel/memory-deduplication attacks** across VMs
  (an attacker can infer another guest's memory content by timing COW faults). For
  this reason KSM is often disabled in multi-tenant/untrusted environments.

{: .note }
> **Balloon vs KSM — different jobs**
> Ballooning changes <em>how much</em> memory a guest holds (a cooperative resize,
> driven by the host but executed by the guest driver). KSM doesn't change any
> guest's apparent memory size — it transparently shares <em>identical</em> physical
> pages behind the scenes. Use ballooning to rebalance memory between guests
> dynamically; use KSM to reclaim duplication when many similar guests run together.
> They compose: balloon to right-size, KSM to dedupe the rest.

---

## Lab

```bash
# ---- BALLOONING ----
# 1. Add a virtio-balloon device to a guest:
$ qemu-system-x86_64 -accel kvm -m 4G -smp 2 \
    -device virtio-balloon-pci \
    -cpu host -drive file=disk.qcow2,if=virtio -nographic \
    -qmp unix:/tmp/qmp.sock,server,nowait &

# 2. Via the QEMU monitor (HMP, Ctrl-A C) check and shrink the guest:
#    (qemu) info balloon
#    balloon: actual=4096        ← guest currently has 4096 MiB
#    (qemu) balloon 2048          ← ask the guest to give back to 2 GiB
#    (qemu) info balloon
#    balloon: actual=2048        ← guest now holds 2 GiB (driver inflated)

# 3. From inside the guest, watch MemTotal/MemAvailable shrink as it inflates:
#    (guest)$ grep MemAvailable /proc/meminfo

# ---- KSM (host-side) ----
# 4. Inspect / enable KSM:
$ cat /sys/kernel/mm/ksm/run
0
$ echo 1 | sudo tee /sys/kernel/mm/ksm/run
1
# (Often managed by ksmtuned; on Debian/Ubuntu: the ksmtuned service.)

# 5. Run two or more similar guests, then watch sharing accumulate:
$ cat /sys/kernel/mm/ksm/pages_shared      # unique pages kept
12000
$ cat /sys/kernel/mm/ksm/pages_sharing     # pages pointing at them (the savings)
58000
# Memory saved ≈ pages_sharing × page_size (here ~58000 × 4 KiB ≈ 226 MiB).

# 6. CPU cost is visible as the ksmd kernel thread:
$ ps -eo pid,comm | grep ksmd
$ kill %1 2>/dev/null
```

**Expected result:** `balloon 2048` shrinks the guest's available memory to ~2 GiB
(visible in the guest's `/proc/meminfo`), and with KSM enabled and similar guests
running, `pages_sharing` climbs — quantifying memory reclaimed by deduplication.

---

## Further Reading

| Topic | Link |
|---|---|
| Kernel Samepage Merging (KSM) | [kernel.org — KSM](https://docs.kernel.org/admin-guide/mm/ksm.html) |
| virtio-balloon / memory ballooning | [Wikipedia — Memory ballooning](https://en.wikipedia.org/wiki/Virtual_memory#Memory_ballooning) |
| KSM security (dedup side channels) | [Wikipedia — Memory deduplication](https://en.wikipedia.org/wiki/Memory_deduplication) |
| Copy-on-write | [Wikipedia — Copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write) |
| QEMU memory devices | [qemu.org — virtio-balloon](https://www.qemu.org/docs/master/system/devices/virtio-pmem.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What does the balloon driver actually do to "give memory back" to the host, and why does it require guest cooperation?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
When the host asks the balloon to inflate, the virtio-balloon driver *inside the guest* allocates and pins a set of pages so the guest's own allocator won't use them, then tells the host those pages are free; the host unmaps and reuses them. Deflating reverses this. It requires guest cooperation because only the guest knows which of its pages are actually free — the host sees the guest's RAM as opaque and can't safely take pages the guest might be using. So a driver inside the guest must run and respond; a guest without it keeps all its memory.
</details>

---

**Q2. How does KSM save memory, and what guarantees correctness when a guest writes to a merged page?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
KSM's ksmd scans mergeable memory and finds pages with identical content across guests/processes (e.g. zero pages, shared OS code), then collapses each set of duplicates into a single physical page that all the owners reference — freeing the redundant copies. Correctness is preserved by copy-on-write: the shared page is read-only, so if any guest writes to it, the kernel first makes that guest a private copy and lets the write proceed there, leaving the others' shared page intact.
</details>

---

**Q3. Name one CPU cost and one security risk of enabling KSM.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
CPU cost: ksmd continuously scans and hashes memory pages looking for duplicates, consuming host CPU — wasteful if guests are dissimilar or CPU is scarce. Security risk: KSM enables memory-deduplication side-channel attacks — because writing a merged page triggers a measurable COW fault, an attacker in one VM can craft pages and time faults to infer whether identical content exists in another VM's memory, leaking cross-tenant information. This is why KSM is often disabled in multi-tenant/untrusted environments.
</details>

---

## Homework

Enable KSM on your host and run two VMs booted from the *same* base image. After a few minutes, read `pages_sharing` and `pages_shared` and compute the approximate memory saved (pages_sharing × page size). Then explain why two VMs from *different* OSes would likely show much less sharing.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Memory saved ≈ pages_sharing × 4 KiB (the count of pages that now point at a shared copy instead of holding their own). Two VMs from the same base image share large amounts because they have identical kernel/library/code pages and many identical zeroed pages, so KSM finds lots of duplicates to merge. Two VMs running *different* OSes have far fewer byte-identical pages — different kernels, libraries, and layouts — so KSM finds fewer matches and saves much less, while still paying the same scanning CPU cost. KSM pays off best when many similar guests run together.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 21 — NUMA Awareness →](lesson-21-numa){: .btn .btn-primary }
