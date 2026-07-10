---
title: "Lesson 23 — Hugepages and NUMA"
nav_order: 8
parent: "Phase 4: Memory"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 23: Hugepages and NUMA

## Concept

The memory phase closes where software meets hardware topology — two topics
the virtualization track already *used* (Lessons 19 and 21 there); now you own
the *why*.

**Hugepages** attack the arithmetic from Lesson 17: 4 KiB pages give ~1500 TLB
entries a reach of ~6 MB — pathetic against multi-GiB working sets. x86-64
page tables natively support **2 MiB** pages (stop the walk one level early)
and **1 GiB** pages (two levels early):

```
   4 KiB page:  PGD → PUD → PMD → PTE → frame      reach ≈ 6 MB
   2 MiB page:  PGD → PUD → PMD ───────► frame      reach ≈ 3 GB    (×512)
   1 GiB page:  PGD → PUD ─────────────► frame      reach ≈ 1.5 TB  (×512²)
   fewer levels = faster walks; fewer entries = fewer misses;
   512× fewer faults to materialize (L19); 512× smaller page tables (L17)
```

**NUMA** (Non-Uniform Memory Access) is the other physical truth: on
multi-socket (and chiplet) machines, RAM is attached *to* CPUs. A core reading
its own node's RAM: fast; reading the other socket's: ~1.5–2× slower, over a
contended interconnect:

```
  ┌───────────── node 0 ─────────────┐   ┌───────────── node 1 ─────────────┐
  │  CPUs 0-15        64GB RAM       │   │  CPUs 16-31       64GB RAM       │
  │        └── local: ~90ns ─┘       │◀─▶│        └── local: ~90ns ─┘       │
  └──────────────────────────────────┘ interconnect: remote ~150-180ns      
```

One machine, secretly two computers glued together — and the kernel spends
real effort hiding the seam: allocating memory on the toucher's node
("first-touch" policy), scheduling tasks near their memory (Lesson 13's
balancer is NUMA-aware), even migrating pages toward their users (AutoNUMA).
When the hiding fails — a task allocates on node 0, gets migrated to node 1,
and spends its life on remote reads — you get the classic "same binary, 40%
slower on the big server" mystery.

---

## How It Works

### Two ways to get hugepages

- **Transparent hugepages (THP)** — the kernel's automatic attempt: anonymous
  regions get 2 MiB pages opportunistically (at fault or via the
  `khugepaged` background collapser). Free wins for big heaps; historical
  bruises for databases — collapse/compaction stalls and memory bloat with
  sparse access — hence the old "disable THP" folklore (modern default
  `madvise`: THP only where the app asked via `madvise(MADV_HUGEPAGE)`).
  Check `/sys/kernel/mm/transparent_hugepage/enabled`, observe in
  `AnonHugePages` (meminfo, smaps).
- **Explicit hugepages (hugetlbfs)** — a reserved pool (`vm.nr_hugepages`),
  allocated at boot/runtime, immune to reclaim/swap, used via mount or
  `MAP_HUGETLB`. Deterministic — no collapse, no stalls — the choice of
  databases (PostgreSQL `huge_pages=on`), DPDK (networking Lesson 64), and
  QEMU guest RAM (virt Lesson 19). 1 GiB pages usually must be reserved at
  boot (`hugepagesz=1G hugepages=8` on the kernel cmdline — Lesson 50).

### NUMA policy tools

`numactl` sets placement per command: `--cpunodebind=0 --membind=0` (hard
pin), `--interleave=all` (spread pages round-robin — for shared structures
accessed by all nodes), `--preferred=0` (soft). `numastat` shows the
scoreboard: `numa_hit` vs `numa_miss` system-wide, and `numastat -p PID` the
per-process node split. The kernel's **AutoNUMA** samples access patterns (via
deliberate minor faults!) and migrates pages/tasks together — good general
default, worth disabling only for hand-placed workloads (virt Lesson 51 pins
guest nodes explicitly for exactly this reason).

{: .note }
> **Your VM is probably NUMA-blind — and that's a lesson too**
> A small VM usually shows one node (<code>numactl --hardware</code>) — the
> hypervisor hides host topology unless configured to expose it (virt
> Lesson 21's vNUMA!). A guest bigger than one host node <em>without</em>
> vNUMA gets silently remote memory: the "same binary slower in the big VM"
> mystery, one level down. The hugepage labs below work everywhere; the NUMA
> labs degrade gracefully to observation on one node.

---

## Lab

```bash
# ---- 1. THP: watch the kernel upgrade your pages ----
$ cat /sys/kernel/mm/transparent_hugepage/enabled
# always [madvise] never          ← bracketed = active mode
$ grep AnonHugePages /proc/meminfo
$ python3 - << 'EOF'
import ctypes, mmap, os
libc = ctypes.CDLL("libc.so.6", use_errno=True)
SZ = 512 * 1024 * 1024
m = mmap.mmap(-1, SZ)
addr = ctypes.addressof(ctypes.c_char.from_buffer(m))
libc.madvise(ctypes.c_void_p(addr), ctypes.c_size_t(SZ), 14)   # MADV_HUGEPAGE
m[::4096] = b'x' * (SZ // 4096)                                # touch it all
os.system(f"grep -E 'AnonHugePages' /proc/{os.getpid()}/status")
os.system("grep AnonHugePages /proc/meminfo")
input("compare with system value before — Enter to exit")
EOF
# AnonHugePages: ~524288 kB in the process — 2MiB pages, granted on request

# ---- 2. The fault-count dividend (Lesson 19 × 512) ----
$ python3 - << 'EOF'
import ctypes, mmap, resource
libc = ctypes.CDLL("libc.so.6")
def run(advise):
    m = mmap.mmap(-1, 512*1024*1024)
    a = ctypes.addressof(ctypes.c_char.from_buffer(m))
    libc.madvise(ctypes.c_void_p(a), 512*1024*1024, 14 if advise else 4)
    f0 = resource.getrusage(resource.RUSAGE_SELF).ru_minflt
    m[::4096] = b'x' * (len(m)//4096)
    print(("huge" if advise else "4k  "),
          resource.getrusage(resource.RUSAGE_SELF).ru_minflt - f0, "minor faults")
    m.close()
run(False); run(True)
EOF
# 4k: ~131072 faults; huge: ~256 — one fault per 2MiB. 512:1, as advertised.

# ---- 3. Explicit hugepages: reserve a pool ----
$ sudo sysctl vm.nr_hugepages=64        # 64 × 2MiB = 128MB pool
$ grep -E 'HugePages_(Total|Free)|Hugepagesize' /proc/meminfo
# HugePages_Total: 64  Free: 64  Hugepagesize: 2048 kB
# these pages are NOW carved out: unswappable, invisible to 'available'!
$ sudo sysctl vm.nr_hugepages=0         # return them when done

# ---- 4. NUMA: discover your topology ----
$ numactl --hardware
# available: 1 nodes (0) ...            ← typical small VM: one node
#   (2+ nodes? run the real experiment below!)
$ numastat | head -4
# numa_hit / numa_miss counters — misses = allocations that went remote

# ---- 5. (Multi-node machines) local vs remote, measured ----
# pin CPU to node 0 but memory to node 1 — the anti-pattern, on purpose:
$ numactl --cpunodebind=0 --membind=0 python3 -c "
import time; a=bytearray(1_000_000_000); t=time.time()
for i in range(0,len(a),64): a[i]=1
print(f'local:  {time.time()-t:.2f}s')"
$ numactl --cpunodebind=0 --membind=1 python3 -c "
import time; a=bytearray(1_000_000_000); t=time.time()
for i in range(0,len(a),64): a[i]=1
print(f'remote: {time.time()-t:.2f}s')"
# remote typically 1.3-2x slower — the seam, measured
# one-node VMs: both run identically; the lesson is knowing to CHECK first
```

---

## Further Reading

| Topic | Link |
|---|---|
| Huge pages — kernel docs (hugetlbpage) | <https://www.kernel.org/doc/html/latest/admin-guide/mm/hugetlbpage.html> |
| Transparent hugepages — kernel docs | <https://www.kernel.org/doc/html/latest/admin-guide/mm/transhuge.html> |
| Non-uniform memory access | <https://en.wikipedia.org/wiki/Non-uniform_memory_access> |
| `numactl(8)` man page | <https://man7.org/linux/man-pages/man8/numactl.8.html> |
| `numastat(8)` man page | <https://man7.org/linux/man-pages/man8/numastat.8.html> |
| `madvise(2)` man page | <https://man7.org/linux/man-pages/man2/madvise.2.html> |

---

## Checkpoint

**Q1.** Which benefits more from hugepages: random access over a 40 GiB
working set, or sequential streaming over the same data? Why?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Random access, dramatically. Its pain <em>is</em> TLB misses: every access
lands on an unpredictable page, and 40 GiB at 4 KiB is ~10M pages vs ~1500 TLB
entries — a miss (and 4-level walk) per access; 2 MiB pages cut entries needed
512× and shorten each walk, often double-digit percent wins (Lesson 17's Q3
database). Sequential streaming already amortizes: 512 consecutive accesses
share one 4 KiB page's TLB entry, the prefetcher hides walk latency, and the
bottleneck is memory bandwidth — which hugepages don't increase. Benefits
sequential mildly (fewer faults to materialize, smaller tables); transforms
random. Rule: hugepages fix <em>translation</em> costs, so they help exactly
where translation dominates.
</details>

**Q2.** Explicit hugepages (hugetlbfs) vs THP: a database vendor recommends
the former and warns against the latter. Reconstruct their reasoning.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
THP is opportunistic and ongoing: khugepaged collapses 4 KiB runs into 2 MiB
pages at runtime (taking page-table locks, requiring contiguous physical
memory — triggering <em>compaction</em>, which migrates pages), and
historically <code>always</code>-mode THP caused latency spikes mid-query and
memory bloat (a sparse 2 MiB region held fully resident for a few bytes —
painful for allocators with scattered access). The database wants
<em>deterministic</em> latency: hugetlbfs pages are reserved up front, never
collapsed, never split, never swapped (they're outside reclaim entirely) —
all the TLB benefit, none of the background machinery. Costs the vendor
accepts: explicit capacity planning (the pool is carved out of RAM whether
used or not) and configuration burden. Modern middle ground:
<code>madvise</code>-mode THP — quiet system-wide, huge only where the app
opts in — which is exactly today's default.
</details>

**Q3.** A service allocates its cache at startup (single-threaded init), then
serves from 32 threads across both sockets. `numastat -p` shows nearly all its
memory on node 0 and heavy remote traffic. Explain how first-touch created
this, and two fixes with their trade-offs.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
First-touch places each page on the node of the CPU that <em>first writes</em>
it: the single-threaded init ran on some node-0 core and touched the entire
cache, so every page landed on node 0. At serve time, half the threads run on
node 1, and every one of their cache reads crosses the interconnect — the
measured remote penalty on ~50% of all accesses, plus contention on the link.
Fixes: (1) <code>numactl --interleave=all</code> (or set interleave policy
before the init allocation) — pages spread round-robin, every thread sees
~50/50 local/remote: predictable, mediocre-but-flat, right for uniformly
shared data; (2) restructure init: have per-node worker threads touch (or
copy) the partitions they'll serve — true locality, best performance, but
requires partitionable data and real code changes (this is what NUMA-aware
databases do). (AutoNUMA may eventually migrate hot pages toward users, but
migrating a huge shared cache is slow and it can't split what all nodes read.)
Same math scaled down explains virt Lesson 51's vNUMA pinning.
</details>

---

## Homework

QEMU guest RAM with hugepages (virt Lesson 19) claims double benefits: the
*guest's* page walks and the *host's* EPT walks both shorten. Using Lesson
17's EPT note (2-D walks: up to 4×4=24 steps) plus this lesson, write the
short explanation you'd give a colleague for why hugepages help VMs *more*
than bare metal — then verify what your VM's QEMU/libvirt would need
(`memoryBacking` XML or `-mem-path`) and what the host must have prepared.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
On bare metal a TLB miss costs one 4-level walk (≤4 memory reads). In a VM,
every step of the <em>guest's</em> walk yields guest-physical addresses that
must themselves be translated through EPT — the two-dimensional walk: up to
~24 reads per miss. Hugepages shrink both dimensions: 2 MiB guest pages cut
guest levels, and 2 MiB host (EPT) pages cut each translation of those steps —
compounding to roughly (3×3+3+3+3)-ish ≈ 2–3× fewer steps <em>per miss</em>,
on top of far fewer misses from 512× TLB reach. Since VMs pay more per miss,
the same reduction buys more — hence "hugepages matter more under
virtualization." Setup: host reserves a pool (<code>vm.nr_hugepages</code>, or
boot-time for 1 GiB pages) and mounts hugetlbfs; libvirt domain gets
<code>&lt;memoryBacking&gt;&lt;hugepages/&gt;&lt;/memoryBacking&gt;</code>
(QEMU: <code>-object memory-backend-file,mem-path=/dev/hugepages,…</code>);
verify with HugePages_Free dropping by guest-RAM/2 MiB at VM start and
<code>numastat -p qemu</code> for node placement — combining both halves of
this lesson.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 5 — Concurrency (Lesson 24: Threads vs Processes) →](lesson-24-threads){: .btn .btn-primary }
