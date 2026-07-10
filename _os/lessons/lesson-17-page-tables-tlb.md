---
title: "Lesson 17 — Page Tables and the TLB"
nav_order: 2
parent: "Phase 4: Memory"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 17: Page Tables and the TLB

## Concept

Lesson 16 waved at "the mapping". Here's what it physically is — because its
shape explains half of systems performance.

A flat table mapping every virtual page would need 256 billion entries per
process (128 TiB ÷ 4 KiB) — absurd for address spaces that are almost entirely
empty islands. So the mapping is a **radix tree**: the 48-bit virtual address
is chopped into four 9-bit indexes plus a 12-bit offset, and each index selects
an entry in one level of a 4-level table walk:

```
 virtual address:  [ 9b PGD ][ 9b PUD ][ 9b PMD ][ 9b PTE ][ 12b offset ]

  CR3 register ─▶ PGD table ─▶ PUD table ─▶ PMD table ─▶ PTE table ─▶ frame + offset
  (per process:      │
   switching CR3     └── empty regions = missing subtrees = no memory wasted
   IS switching
   address spaces — Lesson 12's step 3!)
```

Each final entry (**PTE**) holds the physical frame number plus the bits that
run the whole memory system: `present` (is it in RAM at all? — swap and demand
paging hide here), `writable`, `user`, `accessed`/`dirty` (hardware-maintained
usage tracking — reclaim reads these, Lesson 21), and `NX` (no-execute — why
stacks aren't executable).

But a 4-level walk means **four extra memory reads per memory access** — a 5×
slowdown, unacceptable. Enter the **TLB** (Translation Lookaside Buffer): a
tiny, blazing cache of recent translations inside the CPU. With ~1500 entries
covering ~6 MB (at 4 KiB pages) and >99% hit rates, translation is usually
free. *Usually* — and the exceptions are where performance goes to die.

---

## How It Works

### The walk is hardware, the tables are kernel

The kernel writes the tables (ordinary memory it allocates); the CPU's
page-walker reads them autonomously on TLB misses. `CR3` holds the root — the
per-process value the context switch loads (Lesson 12). This split is the same
mechanism/policy division as ever: hardware executes, kernel decides.

### What the TLB costs you

- **TLB miss** → hardware walk: ~tens of ns (the walker caches upper levels,
  and page-table contents compete for the normal caches too).
- **TLB reach**: 1500 entries × 4 KiB ≈ 6 MB. Touch memory beyond that with
  poor locality — every access misses. A random walk over 4 GiB pays a page
  walk *per access*: this is the hidden tax on hash tables, graph algorithms,
  and databases, and the problem hugepages attack (2 MiB pages × 1500 entries
  ≈ 3 GB reach — Lesson 23).
- **Context switches flush**: classic behavior invalidated the whole TLB on
  CR3 load (Lesson 12's "cache damage"). Modern CPUs tag entries with an
  address-space ID (**PCID**) so switches only deactivate, not destroy — one
  reason the Lesson 12 ping-pong wasn't even slower.

### Dirty and accessed bits: the kernel's spies

On every write, hardware sets the PTE's **dirty** bit; on every access, the
**accessed** bit. The kernel harvests them: accessed bits drive the LRU-ish
reclaim decisions ("which pages are cold?", Lesson 21); dirty bits tell
writeback which page-cache pages must hit disk (Lesson 20) — and KVM's dirty
bitmaps for live migration (virt Lesson 48) are the same idea one level down.

{: .note }
> **Meltdown, KPTI, and why page tables made the news**
> The kernel's own memory used to be mapped (supervisor-only) into every
> process's page tables — syscalls needed no CR3 switch. The 2018
> <a href="https://en.wikipedia.org/wiki/Meltdown_(security_vulnerability)">Meltdown</a>
> flaw let user code read those supervisor pages via speculative execution, so
> kernels now maintain <em>two</em> page-table sets per process (KPTI) and
> switch on every syscall entry — a measurable chunk of the syscall cost from
> Lesson 02, paid forever, because of PTE permission bits that one CPU
> generation didn't honor speculatively.

---

## Lab

```bash
# ---- 1. See translation work: virtual → physical via /proc/PID/pagemap ----
$ cat > /tmp/pm.py << 'EOF'
import os, struct, ctypes
buf = ctypes.create_string_buffer(4096)         # one page of ours
addr = ctypes.addressof(buf)
buf[0] = b'x'                                    # touch it (materialize!)
vpn = addr // 4096
with open(f"/proc/self/pagemap", "rb") as f:     # 8 bytes per virtual page
    f.seek(vpn * 8)
    entry = struct.unpack("<Q", f.read(8))[0]
present = entry >> 63 & 1
pfn = entry & ((1 << 55) - 1)
print(f"virtual page 0x{vpn:x} present={present} → physical frame 0x{pfn:x}")
EOF
$ sudo python3 /tmp/pm.py          # root: PFNs are hidden from users (security!)
# virtual page 0x7f9c3a2b4 present=1 → physical frame 0x1a2b3c
# THE MAPPING, made visible. Run twice — virtual differs (ASLR), physical too.

# ---- 2. Count TLB misses: sequential vs random access, same work ----
$ cat > /tmp/tlb.c << 'EOF'
#include <stdlib.h>
#include <stdio.h>
#define N (256*1024*1024)        /* 256 MB — far beyond TLB reach */
int main(int argc, char **argv) {
    char *a = malloc(N);
    long sum = 0;
    if (argv[1][0] == 's')
        for (long i = 0; i < N; i += 4096) sum += a[i];          /* sequential */
    else {
        unsigned r = 12345;
        for (long i = 0; i < N/4096; i++) {                      /* random    */
            r = r * 1103515245 + 12345;
            sum += a[(long)(r % (N/4096)) * 4096];
        }
    }
    printf("%ld\n", sum);
    return 0;
}
EOF
$ gcc -O2 -o /tmp/tlb /tmp/tlb.c
$ perf stat -e dTLB-load-misses,cycles /tmp/tlb s 2>&1 | grep -E 'dTLB|seconds'
$ perf stat -e dTLB-load-misses,cycles /tmp/tlb r 2>&1 | grep -E 'dTLB|seconds'
# random: MANY more dTLB misses and notably slower — same touches, same data,
# only the ORDER changed. That's the TLB's fingerprint.
# (If perf counters are unavailable in your VM, compare `time` runs — the
#  wall-clock gap is the same story told bluntly.)

# ---- 3. The page-table overhead of a big sparse mapping ----
$ python3 - << 'EOF'
import mmap, os
m = mmap.mmap(-1, 8 * 1024**3)                  # 8 GiB reserved, untouched
os.system(f"grep -E 'VmPTE|VmSize|VmRSS' /proc/{os.getpid()}/status")
for i in range(0, 8 * 1024**3, 4096): m[i] = 1  # touch EVERY page... (slow! ~20s)
os.system(f"grep -E 'VmPTE|VmRSS' /proc/{os.getpid()}/status")
EOF
# before: VmPTE tiny (empty subtrees cost nothing!)
# after:  VmPTE ~16 MB — the page tables THEMSELVES for 8GiB of 4K pages.
# 2M entries × 8 bytes: the radix tree is efficient when sparse, real when full.

$ rm /tmp/pm.py /tmp/tlb /tmp/tlb.c
```

---

## Further Reading

| Topic | Link |
|---|---|
| Page table | <https://en.wikipedia.org/wiki/Page_table> |
| Translation lookaside buffer | <https://en.wikipedia.org/wiki/Translation_lookaside_buffer> |
| `/proc/PID/pagemap` — kernel docs | <https://www.kernel.org/doc/html/latest/admin-guide/mm/pagemap.html> |
| Meltdown vulnerability | <https://en.wikipedia.org/wiki/Meltdown_(security_vulnerability)> |
| `perf-stat(1)` man page | <https://man7.org/linux/man-pages/man1/perf-stat.1.html> |

---

## Checkpoint

**Q1.** Why does a 4-level page walk not make every memory access 5× slower in
practice — and name two situations where the cost *does* surface.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because the TLB caches completed translations: with locality-friendly access
patterns, &gt;99% of accesses hit the TLB and pay no walk; the CPU also caches
upper table levels in dedicated paging-structure caches and pulls table entries
into the ordinary data caches, so the rare walk itself often avoids RAM.
Surfaces: (1) working sets far beyond TLB reach with poor locality — random
access over gigabytes (databases, graph traversal) misses the TLB per access,
exactly the lab's random-vs-sequential gap; (2) address-space-heavy operation
patterns — rapid context switching between many processes (without effective
PCID) or fork-storm CoW where translations are constantly invalidated. Remedies
correspond: hugepages extend reach (Lesson 23); fewer/warmer switches
(Lessons 12/15).
</details>

**Q2.** The `present` bit is a single PTE flag, yet three completely different
kernel features hang off it being 0. Name them and what the fault handler does
in each case.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Present=0 makes the MMU fault on access, handing the kernel a
policy-controlled interception point. Case 1 — <strong>demand paging / never
materialized</strong> (Lesson 19): region is valid but no frame assigned yet;
handler allocates a zeroed frame, fills the PTE, restarts — malloc's promises
become real here. Case 2 — <strong>swapped out</strong> (Lesson 21): the
non-present PTE's remaining bits encode a swap location; handler reads the page
back from swap (a major fault), maps it, restarts. Case 3 — <strong>truly
invalid</strong>: no VMA covers the address; handler delivers SIGSEGV
(Lesson 16's verdict). Same hardware event, three verdicts — present=0 is the
kernel's universal hook into the memory system, and CoW does the same trick
with the writable bit instead.
</details>

**Q3.** Your database's profiler shows heavy `dTLB-load-misses` on its 64 GiB
in-memory hash index. Walk the arithmetic that explains it, and the standard
mitigation.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Hash access is deliberately random: each lookup lands on an unpredictable page.
TLB reach at 4 KiB is roughly 1500 entries × 4 KiB ≈ 6 MB — versus a 64 GiB
structure, so the chance a lookup's page is cached in the TLB is ~6MB/64GB ≈
0.01%: essentially every lookup pays a page walk (tens of ns, potentially
several cache misses for the table levels themselves) on top of the inevitable
data-cache miss. Mitigation: 2 MiB hugepages (Lesson 23) — reach becomes ~1500
× 2 MiB ≈ 3 GB, and each walk is one level shorter; databases and JVMs support
this directly (and the kernel's transparent hugepages try to do it for you).
The virt track saw the same math doubled: EPT walks are 4×4 levels, which is
why hugepages matter even more for VMs (virt Lesson 19).
</details>

---

## Homework

`fork()` (Lesson 07) was "almost free" — but not free. Using this lesson: what
exactly must fork copy for a 2 GiB process even with full CoW, how big is that
copy roughly (use the lab's VmPTE arithmetic), and why does the first write to
each page after fork cost *two* prices? Verify the VmPTE claim: compare
`grep VmPTE /proc/self/status` in a shell against the lab's 8 GiB-touched
python.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
CoW shares the <em>data</em> pages but the child needs its own <em>page
tables</em> — the entire radix tree must be replicated so the two processes'
mappings can later diverge: for 2 GiB fully-touched at 4 KiB, that's ~512K PTEs
× 8 B ≈ 4 MB of tables (plus upper levels), all copied at fork time — the real,
unavoidable cost behind "almost free", and why fork latency grows with the
parent's mapped memory even when no data is copied. First-write double price:
(1) the CoW fault itself — trap, allocate a fresh frame, copy 4 KiB, update the
PTE; (2) the TLB/coherency cost — both processes' cached translations for that
page must be invalidated (fork marked them read-only, the fix rewrites them),
and on multicore that's a TLB shootdown IPI to other CPUs. Verification: a
shell's VmPTE is a few dozen KB; the lab's fully-touched 8 GiB python showed
~16 MB — three orders of magnitude, purely from mapping density.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 18 — A Process's Memory Layout →](lesson-18-process-memory-layout){: .btn .btn-primary }
