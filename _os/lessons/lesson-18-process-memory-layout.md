---
title: "Lesson 18 — A Process's Memory Layout"
nav_order: 3
parent: "Phase 4: Memory"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 18: A Process's Memory Layout

## Concept

Time to tour the islands in the 128 TiB ocean. Every process's address space
has the same cast of regions — and `/proc/PID/maps`, which you've been glancing
at since Lesson 01, lists them precisely. Reading it fluently is the skill of
this lesson; debugging leaks, crashes, and "where did my RAM go" all start here.

```
   high ┌───────────────────────┐
        │        [stack]        │ ← grows DOWN; locals, call frames
        │           ↓           │
        │      (guard gap)      │
        │   mmap region:        │ ← libraries (libc.so), big mallocs,
        │   libs, anon maps,    │   file mappings, thread stacks
        │   [vdso] (Lesson 03)  │
        │           ↕           │
        │           ↑           │
        │        [heap]         │ ← grows UP; malloc's small-object arena
        │  .bss  (zeroed data)  │ ← globals without initializers
        │  .data (init'd data)  │ ← globals with initializers
        │  .text (code, r-x)    │ ← the program's instructions
    low └───────────────────────┘   (segments straight from the ELF — Lesson 46)
```

Each `maps` line: `address-range perms offset dev inode pathname` — and the
perms column (`r`/`w`/`x`, `p`rivate-CoW vs `s`hared) is Lesson 16's
page-granular protection made visible: code is `r-xp` (no writing code!), data
`rw-p` (no executing data — NX), and a write to an `r--p` region is your
segfault's biography.

---

## How It Works

### Two allocation channels: brk and mmap

The heap island grows/shrinks via [`brk(2)`](https://man7.org/linux/man-pages/man2/brk.2.html)
— moving the "program break" fence at its top. Anything else — big allocations,
libraries, thread stacks — arrives via `mmap` as independent regions. **malloc
uses both**: small allocations are carved from the brk-heap arena (cheap,
reused, rarely returned to the OS — why RSS doesn't drop after `free`!); big
ones (≥ ~128 KiB, glibc's `M_MMAP_THRESHOLD`) get a private mmap each (fully
returned on free). This split answers a whole family of "memory mysteries":
free() not shrinking RSS is *arena retention*, not a leak.

### The stack: growth and guards

The main stack's region auto-grows downward: touch just below it, the fault
handler recognizes "stack expansion" and maps more (up to `ulimit -s`, default
8 MiB). Beyond the limit — or hitting the **guard gap** the kernel enforces
below the stack — SIGSEGV: that's a stack overflow's true anatomy, and why
infinite recursion dies cleanly instead of corrupting the heap below. Thread
stacks (Lesson 24) are fixed-size mmaps with an explicit dead guard page — same
idea, statically built.

### Reading memory *sizes* honestly

- **VSZ/VmSize** — total mapped fiction; nearly meaningless operationally.
- **RSS/VmRSS** — frames resident; but *shared pages count fully in every
  process*: 100 nginx workers each "holding" 50 MB of mostly-shared memory do
  not use 5 GB.
- **PSS** (proportional set size, `/proc/PID/smaps_rollup`) — shared pages
  divided by their sharer count: the honest per-process figure that actually
  sums to system usage.
- **USS/Private** — pages exclusively this process's: what you'd get back by
  killing it.

`smaps` gives all of it per-region — the microscope over `maps`.

{: .note }
> **pmap**
> <code>pmap -x PID</code> is <code>maps</code>+<code>smaps</code> as a tidy
> table (address, RSS, dirty per region). Ideal for the "which region is
> eating RAM?" question that raw smaps answers verbosely.

---

## Lab

```bash
# ---- 1. The tour, region by region ----
$ cat /proc/self/maps          # cat's own space at the moment of reading
# 55..    r-xp  .../cat        ← .text (code)
# 55..    rw-p  .../cat        ← .data/.bss
# 55..    rw-p  [heap]
# 7f..    r-xp  .../libc.so.6  ← library code (shared with everyone!)
# 7f..    rw-p                 ← anonymous mmaps
# 7ffc..  rw-p  [stack]
# 7ffc..  r-xp  [vdso]         ← Lesson 03's guest appearance

# ---- 2. Watch malloc choose brk vs mmap ----
$ strace -e trace=brk,mmap python3 -c "a = bytearray(10_000)" 2>&1 | grep -c brk
# several brk calls — small allocation grew the heap
$ strace -e trace=brk,mmap python3 -c "a = bytearray(10_000_000)" 2>&1 | grep 'mmap.*10.*ANONYMOUS' 
# mmap(NULL, 10002432, ..., MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) ← big = own region

# ---- 3. free() ≠ RSS drop (arena retention) ----
$ python3 - << 'EOF'
import os
def rss(): return int(open(f'/proc/{os.getpid()}/status').read().split('VmRSS:')[1].split()[0])
print("start:", rss(), "kB")
small = [bytes(1000) for _ in range(500_000)]   # 500 MB of SMALL objects
print("alloc small:", rss(), "kB")
del small                                        # free them all
print("after del:", rss(), "kB")                 # ← RSS barely drops! arena keeps it
big = bytearray(500_000_000)                     # one BIG allocation
big[::4096] = b'x' * (len(big)//4096)            # touch it
print("alloc big:", rss(), "kB")
del big
print("after del big:", rss(), "kB")             # ← mmap'd: fully returned!
EOF

# ---- 4. Stack growth and its limit ----
$ ulimit -s
# 8192   (kB)
$ python3 - << 'EOF'
import sys
sys.setrecursionlimit(1_000_000)
def deep(n):
    return 1 if n == 0 else 1 + deep(n - 1)
try:
    deep(500_000)
except RecursionError:
    print("python guarded itself first")
EOF
# python protects itself; C wouldn't — segfault at the guard gap (try ulimit -s 512!)

# ---- 5. RSS vs PSS: the honest number ----
$ grep -E '^(Rss|Pss|Private_Dirty)' /proc/self/smaps_rollup
# Rss:  2000 kB     ← counts every shared libc page fully
# Pss:   800 kB     ← libc pages divided by their dozens of sharers
$ pmap -x $$ | tail -3        # your shell's totals, per-region table above

# ---- 6. ASLR sees regions, not bytes: two runs, whole islands moved ----
$ bash -c 'grep -E "heap|stack" /proc/$$/maps'
$ bash -c 'grep -E "heap|stack" /proc/$$/maps'
```

---

## Further Reading

| Topic | Link |
|---|---|
| `proc(5)` — maps, smaps | <https://man7.org/linux/man-pages/man5/proc.5.html> |
| `brk(2)` man page | <https://man7.org/linux/man-pages/man2/brk.2.html> |
| `mallopt(3)` — M_MMAP_THRESHOLD | <https://man7.org/linux/man-pages/man3/mallopt.3.html> |
| `pmap(1)` man page | <https://man7.org/linux/man-pages/man1/pmap.1.html> |
| Data segment (Wikipedia) | <https://en.wikipedia.org/wiki/Data_segment> |
| Proportional set size | <https://en.wikipedia.org/wiki/Proportional_set_size> |

---

## Checkpoint

**Q1.** `malloc(1 GiB)` succeeds instantly but RSS barely moves. Explain, and
predict what appears in `maps`.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
1 GiB is far past the mmap threshold, so glibc makes one
<code>mmap(MAP_PRIVATE|MAP_ANONYMOUS)</code> — which only <em>reserves</em>
address space: the kernel creates a VMA (a new <code>rw-p</code> anonymous
region visible in maps immediately) but assigns no frames; every PTE is absent.
VmSize jumps by 1 GiB; RSS waits. Frames arrive page-by-page on first touch via
demand-paging faults (Lesson 19 measures exactly this). So maps shows a fresh
1 GiB anonymous region, RSS grows only as you write into it — and if you never
touch most of it, most of it never exists. malloc sells promises; the fault
handler delivers.
</details>

**Q2.** A service's RSS climbs all day and never comes down, but heap-profiler
totals are flat. Using the brk/mmap split, give the two innocent explanations
and the test that separates them from a real leak.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Innocent 1 — <strong>arena retention</strong>: freed small objects return to
malloc's arenas, not the OS; the brk heap stays at high-water mark (plus
fragmentation pinning it). Innocent 2 — <strong>page-cache-like touches</strong>
inside the process: lazily-faulted pages of big mappings accumulating toward
their full size (RSS catching up to VmSize, no new allocation at all). Test:
compare <em>Private_Dirty/heap region growth</em> in smaps over time against
the profiler — flat profiler + stable heap VMA size + RSS plateauing = retention
(harmless; <code>malloc_trim</code> or restart reclaims); continuously growing
heap/anon VMA <em>size</em> with proportional Private_Dirty = a real leak
(something's allocating and keeping). Also glance at PSS — shared-page
double-counting can fake "growth" when worker count changes.
</details>

**Q3.** Why does infinite recursion produce a clean SIGSEGV rather than
silently corrupting other memory, and what does `ulimit -s` change about the
answer?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The stack grows into a <em>gap</em>: below the stack region the kernel enforces
an unmapped guard area, and the auto-grow logic only extends the stack up to
RLIMIT_STACK. Recursion pushes frames until the next touch lands beyond the
allowed growth — into unmapped guard territory — where the fault handler finds
no legitimate expansion possible and delivers SIGSEGV: crash at the boundary,
before reaching any other region's pages. <code>ulimit -s</code> sets where
that boundary is: smaller = earlier, cleaner failure (and it's per-process
policy — Lesson 11's rlimits family); <code>unlimited</code> pushes the limit
out but the guard gap below the stack region still separates it from the mmap
region — the design goal is that overflow is always a fault, never a quiet
walk into someone's heap. (Historical footnote: the 2017 "Stack Clash" attacks
defeated a too-small gap; the kernel's gap is now generous.)
</details>

---

## Homework

Explore your biggest real process (browser, IDE, or `python3` with your data
loaded): using `pmap -x PID | sort -k3 -n | tail -15` and
`/proc/PID/smaps_rollup`, answer: (a) which three regions hold the most RSS and
what kind is each (heap? anon mmap? a specific .so? stack?); (b) what fraction
of RSS is shared (RSS − PSS gap); (c) how much would the system *actually* get
back if this process died (which metric answers that)?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Typical findings: (a) the top consumers are one or two huge <em>anonymous</em>
regions (the language runtime's arena/GC heap — labeled <code>[heap]</code> or
blank pathname) plus, in browsers, per-tab shared-memory segments and JIT'd
code regions (<code>rw-p</code> then re-protected <code>r-xp</code> anon —
executable anonymous memory is a JIT fingerprint); .so files rank high in VSZ
but low in RSS because only touched pages count. (b) RSS − PSS is commonly
20–50% for GUI apps — dozens of shared libraries and fonts amortized across
processes. (c) The honest "reclaim if killed" figure is <strong>USS</strong>
(Private_Clean + Private_Dirty in smaps_rollup): exclusively-owned pages —
shared pages survive in their other users, so neither RSS nor even PSS is what
<code>kill</code> refunds. The general habit this builds: never quote one
number for "memory usage" without saying which question it answers.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 19 — Page Faults and Demand Paging →](lesson-19-page-faults){: .btn .btn-primary }
