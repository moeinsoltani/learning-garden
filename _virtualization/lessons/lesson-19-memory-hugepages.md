---
title: "Lesson 19 — Guest Memory Backends and Hugepages"
nav_order: 19
parent: "Phase 5: CPU & Memory Configuration"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 19: Guest Memory Backends and Hugepages

## Concept

Guest RAM is just host memory (Lesson 8). But *how* that host memory is provided —
its **backend** — affects performance and features. And one knob dominates large-VM
performance: the **page size** used to back guest RAM.

```
   normal 4 KiB pages                hugepages (2 MiB or 1 GiB)
   ───────────────────              ───────────────────────────
   a 16 GiB guest =                 a 16 GiB guest =
   ~4 million page-table entries    ~8 thousand (2M) or 16 (1G)
        │                                │
   TLB constantly misses,           far fewer entries → TLB hits,
   2-D page walks (Lesson 6)        cheaper translation, better perf
   are frequent and costly
```

Fewer, bigger pages → fewer TLB entries needed to cover the guest → fewer expensive
two-dimensional page walks → faster memory access, especially for large,
memory-intensive guests.

---

## How It Works

### Memory backend objects

The modern way to define guest RAM is a **memory-backend object**:

- **`memory-backend-ram`** — anonymous host RAM (the default kind of backing).
- **`memory-backend-file`** — back guest RAM by a file mapping, e.g. a **hugetlbfs**
  mount (for explicit hugepages) or `/dev/shm` (for sharing with other processes,
  needed by **vhost-user**, Lesson 30).

Key options on these objects:

- **`mem-path=`** — the file/hugetlbfs path to back memory from.
- **`prealloc=on`** — touch/allocate all pages up front at boot (vs lazy, on-demand
  via EPT violations). Prealloc avoids latency spikes later and guarantees the
  memory exists, at the cost of slower start and no overcommit benefit.
- **`share=on`** — make the mapping shared so other processes (vhost-user, virtio-fs
  DAX) can map the same memory.

### Transparent vs explicit hugepages

- **Transparent Huge Pages (THP):** the kernel *automatically* tries to back memory
  with 2 MiB pages and merge small pages into huge ones in the background (`khugepaged`).
  Zero configuration, but best-effort — under fragmentation you may not actually get
  huge pages, and the background merging adds some jitter.
- **Explicit hugepages:** you *reserve* a pool of huge pages on the host (via
  `vm.nr_hugepages` or boot cmdline `hugepagesz=`/`hugepages=`), mount **hugetlbfs**,
  and back the guest from it. Guaranteed huge pages, no fragmentation surprises, but
  the memory is reserved up front and unavailable to normal processes.

Sizes: **2 MiB** (standard hugepage) and **1 GiB** (gigantic page — needs CPU
`pdpe1gb` support and boot-time reservation). 1 GiB pages give the fewest TLB
entries and are ideal for very large guests.

### Why hugepages reduce TLB pressure

The TLB caches virtual→physical translations and has a *fixed, small* number of
entries. Each entry covers *one page*. With 4 KiB pages, a 16 GiB guest needs
millions of translations but the TLB holds only thousands — so it thrashes, forcing
constant page walks (which in a VM are *two-dimensional*, Lesson 6 — even more
expensive). With 2 MiB pages each TLB entry covers 512× more memory; with 1 GiB,
262,144× more. The guest's working set fits in far fewer TLB entries, so hits go up
and costly walks plummet.

### Overcommit and its risk

If you *don't* prealloc, guest memory is allocated lazily, so you can define guests
totaling more RAM than the host has (**memory overcommit**), relying on guests not
all using their full RAM at once. The risk: if they do, the host runs out of memory
and either swaps heavily (terrible VM performance) or invokes the **OOM killer**,
which can kill a QEMU process — taking down a whole VM. Overcommit is a calculated
bet; hugepages with prealloc are the opposite (guaranteed, reserved).

{: .note }
> **Hugepages and overcommit don't mix**
> Explicit hugepages are *reserved* and *prealloc'd* — the memory is committed up
> front and can't be overcommitted or swapped (hugetlbfs pages aren't swappable). So
> hugepages give you guaranteed performance at the cost of flexibility: you trade the
> ability to overcommit for predictable, TLB-friendly memory. Choose hugepages for
> large, performance-critical, statically-sized guests; choose plain/THP memory when
> you want density and overcommit.

---

## Lab

```bash
# ---- Reserve 1 GiB worth of 2 MiB hugepages on the HOST (512 × 2MiB = 1 GiB) ----
$ grep Hugepagesize /proc/meminfo
Hugepagesize:       2048 kB
$ sudo sysctl vm.nr_hugepages=512
vm.nr_hugepages = 512
$ grep -E 'HugePages_(Total|Free)' /proc/meminfo
HugePages_Total:     512
HugePages_Free:      512

# Mount hugetlbfs (often already at /dev/hugepages):
$ mount | grep hugetlbfs || sudo mount -t hugetlbfs none /dev/hugepages

# ---- Back a guest with those hugepages via a memory-backend-file ----
$ qemu-system-x86_64 -accel kvm -smp 2 \
    -object memory-backend-file,id=mem,size=1G,mem-path=/dev/hugepages,prealloc=on \
    -machine q35,memory-backend=mem \
    -cpu host -drive file=disk.qcow2,if=virtio -nographic &

# ---- Confirm the host handed out hugepages to the guest ----
$ grep -E 'HugePages_(Total|Free)' /proc/meminfo
HugePages_Total:     512
HugePages_Free:        0      ← all 512 now consumed by the VM

# ---- The guest's perception (it just sees 1 GiB RAM; huge backing is host-side) ----
#   (guest)$ grep MemTotal /proc/meminfo
#   MemTotal:  1010xxx kB

# ---- Check THP status on the host (the automatic alternative) ----
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
$ kill %1 2>/dev/null
```

**Expected result:** Reserving 512 hugepages and backing the VM from hugetlbfs
drops `HugePages_Free` to 0 while the VM runs — proof the guest's RAM is backed by
explicit 2 MiB pages. The guest itself just sees 1 GiB of normal RAM.

---

## Further Reading

| Topic | Link |
|---|---|
| HugeTLB pages (kernel) | [kernel.org — HugeTLB Pages](https://docs.kernel.org/admin-guide/mm/hugetlbpage.html) |
| Transparent Hugepage support | [kernel.org — Transparent Hugepage](https://docs.kernel.org/admin-guide/mm/transhuge.html) |
| TLB | [Wikipedia — Translation lookaside buffer](https://en.wikipedia.org/wiki/Translation_lookaside_buffer) |
| QEMU memory backends | [qemu.org — Memory backends](https://www.qemu.org/docs/master/system/devices/igd-assign.html) |
| Memory overcommitment | [Wikipedia — Memory overcommitment](https://en.wikipedia.org/wiki/Memory_overcommitment) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why do explicit hugepages reduce TLB pressure and improve performance for large guests?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The TLB has a small, fixed number of entries, and each entry covers exactly one page. With 4 KiB pages a large guest needs millions of translations but the TLB holds only thousands, so it thrashes and triggers frequent page walks — which in a VM are two-dimensional (guest tables + EPT/NPT), making them especially costly. A 2 MiB page covers 512× more memory per TLB entry (1 GiB covers 262,144×), so the guest's working set fits in far fewer entries: TLB hit rate rises and expensive walks drop sharply, speeding up memory-intensive large guests.
</details>

---

**Q2. What is the difference between transparent hugepages (THP) and explicit hugepages?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
THP is automatic and best-effort: the kernel tries to back memory with 2 MiB pages and merges small pages in the background (khugepaged) with no configuration, but under fragmentation you may not actually get huge pages, and the background work adds jitter. Explicit hugepages are manually reserved up front (vm.nr_hugepages or boot cmdline), exposed via hugetlbfs, and the guest is backed from that pool — guaranteeing huge pages with no fragmentation surprises, but the memory is reserved, non-swappable, and unavailable to normal processes.
</details>

---

**Q3. Why can't you overcommit memory that is backed by explicit hugepages?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Explicit hugepages are reserved and preallocated — the physical memory is committed to the hugepage pool up front and hugetlbfs pages are not swappable. Overcommit relies on lazy allocation and the ability to reclaim/swap memory if guests collectively demand more than exists. Since hugepages are already fully reserved and can't be paged out, there's nothing to overcommit or reclaim: the memory is guaranteed-present by definition. That's the trade — hugepages give predictable performance in exchange for losing overcommit flexibility.
</details>

---

## Homework

Reserve some 2 MiB hugepages on your host with `sysctl vm.nr_hugepages=N`, note `HugePages_Free`, then back a guest with a `memory-backend-file` on `/dev/hugepages` with `prealloc=on`. Watch `HugePages_Free` before, during, and after the VM runs. Explain what `prealloc=on` changed about *when* the hugepages were consumed, versus leaving it off.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With prealloc=on, HugePages_Free drops by the guest's full size the moment the VM starts (QEMU touches/allocates all the pages up front), stays consumed for the VM's lifetime, and is released when the VM exits. Without prealloc, the pages would be allocated lazily as the guest first touched each region (via EPT violations), so HugePages_Free would decline gradually as the guest used memory rather than all at once. prealloc trades a slower start and immediate full reservation for guaranteed availability and no later allocation-latency spikes (and it surfaces "not enough hugepages" immediately at boot rather than mid-run).
</details>
