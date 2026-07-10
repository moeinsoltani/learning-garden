---
title: "Lesson 16 — Virtual Memory: the Grand Illusion"
nav_order: 1
parent: "Phase 4: Memory"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 16: Virtual Memory — the Grand Illusion

## Concept

Run any program and print the address of a variable — you'll get something like
`0x7ffd8a2b3c40`. Here is the uncomfortable truth this phase is built on: **that
number is not a location in RAM.** It's a *virtual* address — a name in a
private, per-process fiction that the kernel and CPU conspire to maintain.

```
  process A's fiction              physical RAM              process B's fiction
  ─────────────────               ────────────              ─────────────────
  0x1000 ──┐                      frame 0x8000  ◀── 0x1000 in B maps here
  0x2000 ──┼── page tables ──▶    frame 0x3000
  0x3000 ──┘   (per process!)     frame 0xC000
                                  frame 0x5000  ◀── two processes may even
  same NUMBERS, different              ▲             map the SAME frame
  memory — or different            A's 0x2000        (shared libraries!)
  numbers, same memory
```

Memory is managed in **pages** — 4 KiB units. Each process has its own mapping
from virtual pages to physical *frames*, and the CPU's
[MMU](https://en.wikipedia.org/wiki/Memory_management_unit) translates every
single memory access through it. Three enormous wins fall out:

1. **Isolation** (the answer promised in Lesson 01's checkpoint): other
   processes' memory has no name in your fiction — you can't even *express* an
   address that reaches it.
2. **Sharing without trust**: the same physical frame holding libc's code
   appears in a thousand fictions at once, read-only. CoW fork (Lesson 07) is
   this trick with a twist.
3. **Lying profitably**: the fiction can be bigger than RAM (swap, Lesson 21),
   promised but not delivered (demand paging, Lesson 19), or backed by a file
   (mmap, page cache, Lesson 20). Every "advanced" memory feature is one more
   way the mapping lies.

Virtualization students: this is the same construction you saw in EPT/NPT
(virt Lesson 06) — there, the *guest's physical* memory turned out to be one
more fiction. It's translations all the way down.

---

## How It Works

### The address space is enormous and empty

On x86-64, user space gets 128 TiB of addressable range. A process uses a few
scattered islands of it (Lesson 18 maps them); everything else is *unmapped* —
touching it means the MMU finds no translation, raises a **page fault**
(Lesson 05's exceptions!), and the kernel, finding no mapping to honor, delivers
SIGSEGV (Lesson 09). A segfault is not the hardware catching you — it's the
kernel declining to extend the fiction.

### Pages and frames

Both sides are cut into 4 KiB units: virtual **pages**, physical **frames**.
The mapping is page-granular — that's the resolution at which memory can be
protected (read-only, no-execute), shared, swapped, or counted. 4 KiB is a
1980s compromise still with us; its cost at modern scale (and the 2 MiB/1 GiB
remedies) is Lesson 23's topic.

### Who does what

The **CPU/MMU** performs every translation (with the TLB cache — Lesson 17
makes translation affordable); the **kernel** owns the map: it builds page
tables, decides what each region may do, and handles every fault. The division
matters: translation is too frequent for software, policy too subtle for
hardware. `VmRSS` vs `VmSize` in `/proc/PID/status` (Lesson 06's table) are the
two sides counted: RSS = frames actually held; Size = extent of the fiction.

{: .note }
> **ASLR — randomizing the fiction**
> Since every process gets its own address space, the kernel can shuffle where
> the islands land at every exec: stack, heap, libraries each start at
> randomized offsets (<a href="https://en.wikipedia.org/wiki/Address_space_layout_randomization">ASLR</a>).
> Attackers who smuggle in an address ("jump to my shellcode at X") find X
> means nothing stable. That's why the lab's addresses change run to run —
> and why security exploits begin with an "info leak" to de-randomize.

---

## Lab

```bash
# ---- 1. Two processes, same address, different contents ----
$ cat > /tmp/vm1.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
int value;                                  /* global: lives in data segment */
int main(int argc, char **argv) {
    value = atoi(argv[1]);
    printf("pid %d: &value = %p, value = %d\n", getpid(), (void*)&value, value);
    sleep(30);
    printf("pid %d: value is STILL %d\n", getpid(), value);
    return 0;
}
EOF
$ gcc -no-pie -o /tmp/vm1 /tmp/vm1.c        # -no-pie: disable ASLR for clarity
$ /tmp/vm1 111 & /tmp/vm1 222 &
# pid 4301: &value = 0x404040, value = 111
# pid 4302: &value = 0x404040, value = 222   ← SAME address!
$ wait
# pid 4301: value is STILL 111               ← ...and neither ever saw the
# pid 4302: value is STILL 222                  other's write. Separate fictions.

# ---- 2. ASLR: watch the fiction get shuffled ----
$ for i in 1 2 3; do bash -c 'grep stack /proc/$$/maps'; done
# [stack] at a different 0x7ffc... each run
$ cat /proc/sys/kernel/randomize_va_space
# 2   (full ASLR — the default)

# ---- 3. Same physical frame in many fictions: shared libraries ----
$ grep 'libc.*r-xp' /proc/self/maps
# every process maps libc's code — read-only, execute — one physical copy
# count how many processes share it:
$ grep -l 'libc.so.6' /proc/[0-9]*/maps 2>/dev/null | wc -l
# dozens — imagine RAM usage if each had a private copy

# ---- 4. Unmapped = unnameable: make a segfault knowingly ----
$ python3 -c "
import ctypes
ctypes.string_at(0x10)      # read address 16 — nothing is mapped there
" 2>&1 | tail -1
# Segmentation fault (or SIGSEGV traceback) — the kernel refused the fiction

# ---- 5. VmSize vs VmRSS: the fiction vs the delivered ----
$ python3 - << 'EOF'
import os
big = bytearray(300_000_000)        # "give me 300 MB" (mostly a promise...)
big[0] = 1; big[-1] = 1             # ...touch just the edges
os.system(f"grep -E 'VmSize|VmRSS' /proc/{os.getpid()}/status")
EOF
# VmSize:  ~500000 kB     ← the fiction: python + 300MB promised
# VmRSS:   much smaller   ← frames actually delivered (Lesson 19 explains!)

$ rm /tmp/vm1 /tmp/vm1.c
```

---

## Further Reading

| Topic | Link |
|---|---|
| Virtual memory | <https://en.wikipedia.org/wiki/Virtual_memory> |
| Memory management unit | <https://en.wikipedia.org/wiki/Memory_management_unit> |
| Page (computer memory) | <https://en.wikipedia.org/wiki/Page_(computer_memory)> |
| ASLR | <https://en.wikipedia.org/wiki/Address_space_layout_randomization> |
| `mmap(2)` man page | <https://man7.org/linux/man-pages/man2/mmap.2.html> |

---

## Checkpoint

**Q1.** Two processes both print a pointer value `0x404040`. Same memory? How
do you know, and under what circumstance *could* two processes' identical
virtual addresses refer to the same physical memory?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Not the same memory by default: each process's page tables translate
independently, so <code>0x404040</code> lands in different physical frames —
the lab proved it by writing different values that never interfered. But the
mapping is per-process <em>policy</em>, so identical virtual addresses
<em>can</em> point at one frame when the kernel arranges sharing: the same
shared library's code, a MAP_SHARED mmap of the same file (Lesson 33's shared
memory), or parent/child right after fork, where all pages are shared
copy-on-write until someone writes. Virtual address equality tells you nothing;
only the mapping does.
</details>

**Q2.** A segfault and a "normal" memory access differ at exactly one decision
point. Walk the path of a load instruction and identify where SIGSEGV is born.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
CPU executes the load → MMU looks up the virtual page (TLB first, then page
tables). If a valid translation exists with permission for the access, the load
proceeds to cache/RAM — done, no kernel involved. If not, the MMU raises a page
fault — a synchronous exception (door #2 from Lesson 05) — and the kernel's
fault handler decides. That decision is the birthplace: if the address belongs
to a legitimate region that just isn't materialized yet (demand paging, CoW,
swapped page — Lesson 19), the kernel fixes the mapping and restarts the
instruction — invisible. Only if the address matches <em>no</em> region (or
violates permissions, e.g. writing read-only) does the kernel deliver SIGSEGV.
So: hardware detects, kernel judges. A segfault is a verdict, not an accident.
</details>

**Q3.** Explain why shared libraries would be catastrophically expensive
without virtual memory, using the numbers from the lab.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
libc's code is ~2 MB, and the lab found dozens of processes mapping it —
without sharing, each would need a private copy in RAM: dozens × 2 MB for libc
alone, times every other common library (X/GTK/Qt easily reach hundreds of MB
each). Virtual memory makes sharing free and safe: one physical copy of the
read-only code pages, mapped into every process's fiction (possibly at
different virtual addresses — the dynamic loader handles that, Lesson 47),
enforced read-only so no process can corrupt it for others. RSS accounting even
has to be careful not to double-count these shared pages (the honest metric,
PSS, appears in Lesson 18). Without VM you'd get either enormous duplication or
shared mutable code — i.e., no isolation at all.
</details>

---

## Homework

Prove page-granularity protection exists: write a C program that `mmap`s one
anonymous page `PROT_READ|PROT_WRITE`, writes to it, then uses `mprotect` to
make it read-only, installs nothing, and writes again — observing the SIGSEGV.
Then explain: at what granularity did the permission change apply, and why
couldn't you protect just one *byte*?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<pre>
#include &lt;sys/mman.h&gt;
#include &lt;stdio.h&gt;
int main(void){
    char *p = mmap(NULL, 4096, PROT_READ|PROT_WRITE,
                   MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
    p[0] = 'A';
    printf("wrote before mprotect: %c\n", p[0]);
    mprotect(p, 4096, PROT_READ);
    p[1] = 'B';                       /* ← SIGSEGV here */
    printf("never reached\n");
    return 0;
}
</pre>
Output: the first write succeeds, then "Segmentation fault" on the second. The
protection changed for the whole 4096-byte page — necessarily: permissions live
in the page-table entry, and the PTE is the unit of translation; the MMU checks
one entry per access and knows nothing smaller. One byte of a page can't be
read-only because there is no hardware structure to record it. This
page-granularity is why allocators put guard pages around stacks (Lesson 18),
why mprotect-based debugging tools (e.g. electric fence) round to pages, and
why "byte-precise" memory safety needs compiler tooling (ASan) rather than the
MMU.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 17 — Page Tables and the TLB →](lesson-17-page-tables-tlb){: .btn .btn-primary }
