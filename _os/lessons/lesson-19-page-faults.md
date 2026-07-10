---
title: "Lesson 19 — Page Faults and Demand Paging"
nav_order: 4
parent: "Phase 4: Memory"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 19: Page Faults and Demand Paging

## Concept

Three lessons have now promised that memory is "materialized on first touch."
This is the lesson where that machinery gets a name, a price list, and a lab.

The kernel's memory system is built on a productive lie: when you ask for
memory (malloc → mmap, Lesson 18), it says *yes* and does *nothing*. The truth
is enforced lazily by the **page fault** — the MMU trap on a missing
translation (Lesson 17's present bit) — and the fault handler's judgment:

```
                     page fault on address X
                              │
              does a VMA cover X? (maps regions)
               │                            │
              yes                           no → SIGSEGV (the verdict, L16)
               │
     what does the region need?
      ├─ anonymous, first touch  → grab a zero frame, map it      ─┐
      ├─ file-backed, not read   → read block into page cache,     │ MINOR
      │                            map that frame (L20!)           │ (no disk*)
      ├─ CoW write (fork, L07)   → copy the frame, remap writable ─┘
      └─ swapped out (L21)       → read from swap device…          ─ MAJOR
                                                                     (disk I/O!)
```

**Minor faults** are bookkeeping: microseconds, no disk. **Major faults** wait
for storage: *milliseconds* — a 1000× cliff. The distinction is in every
process's accounting (`ps -o min_flt,maj_flt`), and it's the difference between
"program warming up" and "system thrashing" (Lesson 21).

Demand paging is why: huge executables start instantly (only the pages actually
executed ever load), your 300 MB bytearray from Lesson 16 cost almost nothing
(untouched pages), and the first pass over any big buffer is always
mysteriously slower than the second — you're paying one minor fault per page,
exactly measurable, exactly predictable.

---

## How It Works

### The fault path, precisely

1. MMU raises the exception; CPU enters the kernel with the faulting address
   (in the CR2 register) and cause bits (read/write? user/kernel? present?).
2. Handler finds the covering **VMA** — the kernel's record of each maps-line
   region: its range, permissions, and *what backs it* (nothing/file/…).
3. Legitimate? Fix the PTE per the table above; return; the CPU **re-executes
   the faulting instruction**, which now succeeds. The program never knows.
4. Not legitimate? SIGSEGV (or SIGBUS for "valid mapping, dead backing" — e.g.
   a truncated mapped file: the region exists but the bytes don't).

That re-execution detail is load-bearing: faults are transparent *because* the
instruction restarts. It's the same trap-fix-retry pattern as the whole
virtualization track (VM exits) — the kernel is a hypervisor for processes.

### Fault-driven features you now fully own

- **Zero page**: reads of untouched anonymous memory all map one shared
  physical page of zeros (read-only); the private frame arrives only on first
  *write* (a CoW fault off the zero page!). Calloc of gigabytes: nearly free
  until written.
- **CoW fork** (Lesson 07): every post-fork first-write is a minor fault +
  4 KiB copy. A fork-heavy workload's cost hides in its children's
  `min_flt`.
- **File mapping**: executables and libraries load by fault — `execve` maps
  regions and jumps; each first-executed page faults in through the page cache
  (Lesson 20 completes this story).

`MAP_POPULATE`, `mlock(2)`, and plain "touch every page first" are the three
standard ways to pre-pay faults when latency spikes at first-access are
unacceptable (trading systems, audio, VMs — virt Lesson 19's `prealloc`).

{: .note }
> **Watching faults live**
> Per process: <code>ps -o min_flt,maj_flt PID</code>, or the 10th/12th fields
> of <code>/proc/PID/stat</code>. System-wide: <code>vmstat 1</code>'s
> <code>si/so</code> hint at majors via swap; <code>sar -B 1</code> gives
> faults/s directly; and <code>/proc/vmstat</code>'s <code>pgfault</code> /
> <code>pgmajfault</code> counters are the raw truth (you met them in
> Lesson 05's homework).

---

## Lab

```bash
# ---- 1. Count the faults you cause: touch N pages, get N faults ----
$ python3 - << 'EOF'
import os, resource
def faults():
    r = resource.getrusage(resource.RUSAGE_SELF)
    return r.ru_minflt, r.ru_majflt
m0, M0 = faults()
buf = bytearray(100 * 1024 * 1024)          # 100 MB promised (25600 pages)
m1, M1 = faults()
print(f"after alloc:  +{m1-m0} minor, +{M1-M0} major")   # ~0: nothing touched!
for i in range(0, len(buf), 4096):
    buf[i] = 1                               # first touch, page by page
m2, M2 = faults()
print(f"after touch:  +{m2-m1} minor, +{M2-M1} major")   # ≈ 25600 minor, 0 major
for i in range(0, len(buf), 4096):
    buf[i] = 2                               # SECOND pass
m3, M3 = faults()
print(f"second pass:  +{m3-m2} minor")                    # ≈ 0: already real
EOF

# ---- 2. Time the fault tax: first pass vs second pass ----
$ python3 - << 'EOF'
import time
buf = bytearray(400 * 1024 * 1024)
t = time.time()
for i in range(0, len(buf), 4096): buf[i] = 1
print(f"first pass:  {time.time()-t:.3f}s  (faulting)")
t = time.time()
for i in range(0, len(buf), 4096): buf[i] = 2
print(f"second pass: {time.time()-t:.3f}s  (mapped)")
EOF
# first pass typically 2-4x slower — that gap is ~100k minor faults

# ---- 3. Major faults: evict a file, then map-read it ----
$ dd if=/dev/urandom of=/tmp/blob bs=1M count=200 2>/dev/null
$ sync && echo 3 | sudo tee /proc/sys/vm/drop_caches   # cold cache (lab only!)
$ python3 - << 'EOF'
import mmap, resource
def majors(): return resource.getrusage(resource.RUSAGE_SELF).ru_majflt
with open('/tmp/blob', 'rb') as f:
    m = mmap.mmap(f.fileno(), 0, prot=mmap.PROT_READ)
    before = majors()
    s = 0
    for i in range(0, len(m), 4096): s += m[i]   # every touch may hit DISK
    print(f"major faults reading cold mapped file: {majors()-before}")
EOF
# hundreds-thousands of majors. Re-run WITHOUT drop_caches: ~0 majors —
# the pages are in the page cache now (Lesson 20 takes it from here)

# ---- 4. CoW faults after fork, measured (Lesson 07's promise) ----
$ python3 - << 'EOF'
import os, resource
buf = bytearray(50 * 1024 * 1024)
for i in range(0, len(buf), 4096): buf[i] = 1     # fully materialize FIRST
pid = os.fork()
if pid == 0:
    r0 = resource.getrusage(resource.RUSAGE_SELF).ru_minflt
    for i in range(0, len(buf), 4096): buf[i] = 2 # child writes → CoW copies
    print(f"child CoW faults: {resource.getrusage(resource.RUSAGE_SELF).ru_minflt - r0}")
    os._exit(0)
os.waitpid(pid, 0)
EOF
# ≈ 12800 faults = 50MB/4K — each one copied a page. CoW's bill, itemized.

# ---- 5. ps view, for any process ----
$ ps -o pid,min_flt,maj_flt,comm -p $$
$ rm /tmp/blob
```

---

## Further Reading

| Topic | Link |
|---|---|
| Page fault | <https://en.wikipedia.org/wiki/Page_fault> |
| Demand paging | <https://en.wikipedia.org/wiki/Demand_paging> |
| `getrusage(2)` — fault counters | <https://man7.org/linux/man-pages/man2/getrusage.2.html> |
| `mlock(2)` man page | <https://man7.org/linux/man-pages/man2/mlock.2.html> |
| `mmap(2)` — MAP_POPULATE | <https://man7.org/linux/man-pages/man2/mmap.2.html> |

---

## Checkpoint

**Q1.** Which causes a major fault: first touch of malloc'd memory, or first
read of an mmap'd file after `drop_caches`? Explain the asymmetry.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The file read. Anonymous memory's first touch needs only a zeroed frame — RAM
bookkeeping, no device: minor. The cold file mapping's content exists only on
disk; the fault handler must issue real I/O and put the process to sleep until
the block arrives (through the page cache): major. The asymmetry is simply
<em>where the truth lives</em>: anonymous pages have no truth yet (zeros will
do), file pages have truth elsewhere and fetching it costs milliseconds. Same
trap, thousand-fold price difference — and it's why "maj_flt climbing" is an
alarm (disk in the memory path) while min_flt climbing is usually just
workload.
</details>

**Q2.** Why must the CPU re-execute the faulting instruction after the handler
fixes the PTE — what would break with "just continue at the next instruction"?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The faulting instruction never completed: its load didn't produce a value, its
store never wrote. Skipping it would corrupt the program — a
<code>mov (%rax), %rbx</code> would leave garbage in rbx and execution would
continue on false state. Restart semantics make the fault a pure hidden delay:
fix the mapping, re-run the instruction, it succeeds as if memory had always
been there. This requires instructions to be restartable (no partial side
effects before the fault is resolved) — a real constraint CPU architects carry
so kernels can build everything in this phase: demand paging, CoW, swap, and
mapped files are all "fault, fix, retry" — invisible precisely because of the
retry.
</details>

**Q3.** A latency-critical service shows p999 spikes traced to first-access
faults on a 2 GiB in-memory table it builds at startup. Give three distinct
pre-payment strategies and their trade-offs.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <strong>Touch it all at startup</strong> — walk every page (or build with
writes that touch everything): simplest, no special APIs; costs startup time
and pays the fault path per page. (2) <strong>MAP_POPULATE</strong> at mmap
time: the kernel pre-faults the region in one batch — cheaper than a userspace
walk (no per-page trap), but startup blocks for the whole population. (3)
<strong>mlock/mlockall</strong>: populates AND pins — pages can never be
reclaimed or swapped later (protects against Lesson 21 evictions too!), the
strongest guarantee; costs RLIMIT_MEMLOCK privilege and permanently removes the
memory from the kernel's flexibility (and hugepages, Lesson 23, would shrink
the fault count 512× as a complementary move). Production latency systems
typically combine: hugepages + mlock + prefault — exactly what the virt track's
guest-memory prealloc options do for VMs.
</details>

---

## Homework

Executables load by demand paging: only executed pages ever fault in. Verify
it: `ps -o min_flt,maj_flt` on a freshly-started `sleep 300 &` (tiny binary),
then on a python that only prints, then design and run the comparison that
shows a *cold* start (after drop_caches) of python paying major faults while a
warm restart pays almost none. Report the three numbers and what each proves.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Typical results: (1) <code>sleep</code>: ~100 minor, 0 major — a few dozen
pages of code/libc actually executed, everything already cache-hot. (2) warm
<code>python3 -c pass</code>: ~1000–3000 minor, ~0 major — bigger runtime, many
more pages touched, but the interpreter and libraries sit in the page cache so
every fault is a cheap remap. (3) after <code>sync; echo 3 &gt;
drop_caches</code>, the same python shows the <em>same order</em> of minor
faults <em>plus</em> tens-to-hundreds of majors — each one a disk read of a
code page being executed for the first time since eviction; wall-clock startup
visibly jumps. Proves: process startup cost is dominated by the page cache's
state, not by execve; "the second run is always faster" is demand paging +
page cache, quantified — and it's why container cold-starts and just-booted
systems feel slow: everyone's majors, everywhere, at once (Lesson 20 next).
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 20 — The Page Cache →](lesson-20-page-cache){: .btn .btn-primary }
