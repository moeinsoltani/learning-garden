---
title: "Lesson 63 — perf"
nav_order: 1
parent: "Phase 13: Tracing & Debugging"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 63: perf

## Concept

Something is slow. The first question is always "where is the time going?" —
and `perf`, the kernel's built-in profiler, answers it with evidence instead
of guesses.

perf works two complementary ways:

```
   COUNTING (perf stat):  read hardware/software counters over a run.
                          "how many cache misses? branch mispredicts?
                           context switches? instructions per cycle?"
                          → the WHAT: is this CPU-bound? cache-bound?

   SAMPLING (perf record/top): interrupt the CPU N times/sec, record WHERE
                          it was (the instruction + call stack) each time.
                          "which functions is the CPU actually in?"
                          → the WHERE: which code to fix.
```

The magic is the **PMU** (Performance Monitoring Unit) — dedicated hardware
counters in every CPU that tally low-level events (cycles, instructions
retired, cache references/misses, TLB misses — Lesson 17!, branch
mispredictions) with near-zero overhead. perf reads these plus software events
(context switches — Lesson 12, page faults — Lesson 19) and can sample both
user and kernel stacks, attributing time across the whole boundary you've
studied.

You've reached for perf already: Lesson 17 counted TLB misses, Lesson 45
compared cycles for zero-copy. This lesson makes it a fluent tool — because
"the system is slow" is the most common real problem, and perf is how you stop
guessing.

---

## How It Works

### perf stat: the vital signs

`perf stat CMD` runs CMD and prints counters. The ones that matter:

- **IPC** (instructions per cycle) — the headline efficiency number: >2 is
  good, <1 means the CPU is *stalled* (waiting on memory, mispredicts) more
  than computing — a signal to look at cache/memory, not algorithm.
- **cache-misses / cache-references** — memory-bound workloads (Lesson 12's
  cache damage, Lesson 17's TLB reach) show high miss ratios.
- **context-switches, cpu-migrations** — Lesson 12/15's scheduling costs.
- **page-faults** — Lesson 19's memory materialization.

Low IPC + high cache-misses = memory-bound (fix data layout — Lesson 29's
false sharing, Lesson 23's hugepages); high IPC but slow = genuinely
compute-bound (fix the algorithm); high context-switches = contention
(Lessons 12/26).

### perf record → report → flame graphs

`perf record -g CMD` samples with call stacks; `perf report` shows a ranked
tree of where time went; **flame graphs** (Brendan Gregg's visualization)
render the stacks as a picture where width = time — the single most effective
way to see "this function and its callees ate 40% of the CPU." `perf top` is
the live version (like `top`, but for functions). The `-g` (call graph) is
essential — without it you learn *which function* but not *who called it*
(the difference between "memcpy is hot" and "memcpy is hot because of *this*
code path").

### Attribution across the boundary

perf samples both user and kernel stacks, so a flame graph can show your code
calling into libc calling into a syscall calling into kernel functions — the
whole path from Lesson 46's binary through Lesson 02's syscall into the kernel
subsystems. Time in `_raw_spin_lock` or `futex` = lock contention (Lesson 26);
time in kernel I/O paths = you're I/O-bound (Lesson 42); time in
`copy_to_user`/page-fault handlers = memory movement (Lessons 19/45). perf is
where the whole course becomes a diagnostic vocabulary.

{: .note }
> **perf needs permission (and sometimes symbols)</br>**
> <code>perf</code> often needs <code>sudo</code> or a relaxed
> <code>kernel.perf_event_paranoid</code> sysctl; kernel symbols need
> <code>/proc/kallsyms</code> readable; your program's symbols need it built
> with frame pointers (<code>-fno-omit-frame-pointer</code>) or DWARF
> (<code>perf record --call-graph dwarf</code>) — otherwise stacks show
> <code>[unknown]</code>. In VMs the PMU may be limited (some counters
> unavailable) — the software events (context-switches, faults) always work.

---

## Lab

```bash
$ sudo apt install -y linux-tools-common linux-tools-generic 2>/dev/null | tail -1
$ sudo sysctl kernel.perf_event_paranoid=1 2>/dev/null   # allow user profiling

# ---- 1. perf stat: the vital signs of a workload ----
$ perf stat dd if=/dev/zero of=/dev/null bs=1M count=5000 2>&1 | grep -E 'instructions|IPC|cache|context|seconds'
#    ...  instructions  #  X.XX  insn per cycle   ← IPC: the efficiency headline
#    ...  cache-misses  #  Y % of all cache refs
#    ...  context-switches

# ---- 2. Memory-bound vs compute-bound, told apart by IPC + cache ----
$ cat > /tmp/bound.c << 'EOF'
#include <stdlib.h>
#include <string.h>
#define N (128*1024*1024)
int main(int c, char **v) {
    char *a = malloc(N);
    long s = 0;
    if (v[1][0] == 'm')                    /* MEMORY-bound: random 64B strides */
        for (int r = 0; r < 40; r++)
            for (long i = 0; i < N; i += 4099) s += a[i];
    else                                    /* COMPUTE-bound: tight arithmetic */
        for (long i = 0; i < 3000000000L; i++) s += i * 3 % 7;
    return s & 1;
}
EOF
$ gcc -O2 -o /tmp/bound /tmp/bound.c
$ echo "=== memory-bound ==="; perf stat /tmp/bound m 2>&1 | grep -E 'insn per|cache-misses'
$ echo "=== compute-bound ==="; perf stat /tmp/bound c 2>&1 | grep -E 'insn per|cache-misses'
# memory: LOW IPC (<1), HIGH cache-miss ratio → fix data layout/access (L23/29)
# compute: HIGH IPC (>2), low cache misses    → fix the algorithm
# SAME "it's slow", OPPOSITE remedies — IPC + cache-misses told you which.

# ---- 3. perf record + report: WHERE the time goes ----
$ perf record -g -o /tmp/perf.data /tmp/bound m 2>/dev/null
$ perf report -i /tmp/perf.data --stdio 2>/dev/null | grep -A8 'Overhead' | head -10
#  Overhead  Command  Symbol
#   98.xx%   bound    [.] main          ← the hot function, ranked by samples
# with real programs you'd see the call TREE (-g) — who called the hot code

# ---- 4. perf top: the live function-level 'top' ----
$ /tmp/bound m & 
$ sudo timeout 3 perf top -p $! --stdio 2>/dev/null | head -8 || echo "(perf top: live hottest functions)"
$ wait 2>/dev/null

# ---- 5. See lock contention as a perf signature (L26, from the profile) ----
$ cat > /tmp/contend.c << 'EOF'
#include <pthread.h>
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
long ctr;
void *w(void *a){ for(int i=0;i<2000000;i++){pthread_mutex_lock(&m);ctr++;pthread_mutex_unlock(&m);} return 0;}
int main(){ pthread_t t[8]; for(int i=0;i<8;i++)pthread_create(&t[i],0,w,0); for(int i=0;i<8;i++)pthread_join(t[i],0); return 0;}
EOF
$ gcc -pthread -O2 -o /tmp/contend /tmp/contend.c
$ perf record -g -o /tmp/c.data /tmp/contend 2>/dev/null
$ perf report -i /tmp/c.data --stdio 2>/dev/null | grep -iE 'lock|futex' | head -3
# time in __lll_lock_wait / futex / spin ← contention, visible in the profile
# (8 threads fighting one lock — L26's cache-line war, now measured by perf)

# ---- 6. Flame graph data (the picture) ----
$ perf record -g -o /tmp/fg.data /tmp/bound m 2>/dev/null
$ perf script -i /tmp/fg.data 2>/dev/null | head -5
# raw stack samples — pipe through Brendan Gregg's stackcollapse + flamegraph.pl
# (or `perf report --stdio`) to see width=time. THE tool for "what's hot".
$ rm -f /tmp/bound* /tmp/contend* /tmp/perf.data /tmp/c.data /tmp/fg.data
```

---

## Further Reading

| Topic | Link |
|---|---|
| `perf(1)` — the profiler | <https://perfwiki.github.io/main/> |
| `perf stat(1)` man page | <https://man7.org/linux/man-pages/man1/perf-stat.1.html> |
| Brendan Gregg — perf examples | <https://www.brendangregg.com/perf.html> |
| Flame graphs | <https://www.brendangregg.com/flamegraphs.html> |
| Perf Monitoring Unit / hardware counters | <https://en.wikipedia.org/wiki/Hardware_performance_counter> |
| Instructions per cycle | <https://en.wikipedia.org/wiki/Instructions_per_cycle> |

---

## Checkpoint

**Q1.** perf shows 40% of time in `_raw_spin_lock`. What class of problem is
this, and which earlier lesson's tools confirm it?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Lock contention (Lesson 26). Time in <code>_raw_spin_lock</code> (or
<code>__lll_lock_wait</code>/<code>futex</code> in userspace) means threads are
spending their CPU <em>waiting to acquire a lock</em> rather than doing work —
a serialization bottleneck where a hot lock's critical section can't keep up
with the threads contending for it (Lesson 26's cache-line ping-pong: every
CAS steals the lock's cache line between cores — Lesson 29). It's a scalability
problem: adding cores makes it <em>worse</em>, not better, because more threads
pile onto the same lock. Confirming tools from earlier lessons: <code>pidstat
-w</code> and <code>/proc/PID/status</code> (Lesson 12) show high
<em>voluntary</em> context switches (threads sleeping in futex_wait —
Lesson 26's lab wchan of <code>futex_wait_queue</code>) with low CPU
efficiency; <code>perf stat</code> (this lesson) shows low IPC despite high CPU
usage (stalled on the lock, not computing); and reading
<code>/proc/PID/task/*/wchan</code> shows threads parked in futex. The fixes are
Lesson 26's: shrink the critical section (do work outside the lock — the
narrow-vs-wide experiment), shard the hot lock into many (per-bucket locks),
switch to per-CPU/per-thread state summed on read (Lesson 29's scaling
pattern), or use lock-free atomics where the invariant allows (Lesson 29). perf
told you <em>where</em> (which lock, via -g's call graph — who's contending),
and the confirming tools quantify <em>how bad</em> — together turning "it's
slow and doesn't scale" into a specific, fixable contention point.
</details>

**Q2.** Two programs both run slowly. perf stat shows program A at IPC 0.4
with 30% cache-miss ratio, program B at IPC 2.8 with 1% cache-miss ratio. What
does each number tell you, and how do the *fixes* differ?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Program A (IPC 0.4, 30% cache misses) is <strong>memory-bound</strong>: IPC
below 1 means the CPU is stalled most cycles — it issues an instruction, then
waits — and the 30% cache-miss ratio names why: it's constantly fetching data
from RAM (hundreds of cycles each) instead of cache (a few), so the processor
spins idle waiting on memory. The bottleneck is <em>data movement and access
patterns</em>, not computation. Fixes target locality: improve data layout
(struct-of-arrays vs array-of-structs, pack hot fields together — Lesson 29's
false-sharing lesson), access memory sequentially to help prefetch and TLB
(Lesson 17's sequential-vs-random), use hugepages if TLB-bound (Lesson 23),
shrink the working set, or add a cache-friendly algorithm — <em>making the same
computation touch memory better</em>. Program B (IPC 2.8, 1% cache misses) is
<strong>compute-bound</strong>: it's using the CPU efficiently — nearly 3
instructions retired per cycle, memory well-cached — it's simply doing a lot of
work. The bottleneck is the <em>amount of computation</em>. Fixes target the
algorithm: a better big-O, SIMD/vectorization to do more per instruction,
parallelism across cores (Lesson 24 — if B is single-threaded, this is the big
win), precomputation/caching of results, or removing redundant work — <em>doing
less computation</em>, since each unit is already cheap. The lesson: identical
symptoms ("slow"), opposite remedies, and IPC + cache-miss ratio distinguish
them in one <code>perf stat</code> — which is why profiling before optimizing
matters. Optimizing A's algorithm or B's data layout would waste effort on the
non-bottleneck; the numbers point you at the right half of the machine.
</details>

**Q3.** Why is `perf record -g` (with call graphs) so much more useful than
without, and what does perf's ability to sample *both* user and kernel stacks
let you diagnose that userspace-only tools can't?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Without <code>-g</code>, perf tells you <em>which function</em> is hot but not
<em>why it's being called</em> — you learn "<code>memcpy</code> is 40% of time"
but not which of the twenty call sites drives it, so you can't fix the cause,
only observe the symptom. With <code>-g</code>, perf captures the full call
stack at each sample, so <code>perf report</code> shows the <em>tree</em>:
memcpy is hot <em>because</em> function X calls Y calls memcpy in a loop —
pointing at the actual code to change (and flame graphs make this visual, width
= time down each call path). For hot leaf functions shared across many callers
(memcpy, malloc, lock primitives), the call graph is the difference between a
dead end and a fix. The cross-boundary sampling is perf's unique power: it
captures user stacks <em>and</em> kernel stacks in one sample, so a single flame
graph spans your application code → libc → the syscall (Lesson 02) → the kernel
subsystem handling it. This diagnoses things userspace-only tools structurally
can't see: whether your "slow" program is actually spending its time in
<em>kernel</em> code — blocked in the block-layer I/O path (Lesson 42, you're
I/O-bound not CPU-bound), in <code>futex</code>/spinlocks (Lesson 26,
contention), in page-fault handlers (Lesson 19, memory pressure), in
<code>copy_to_user</code> (Lesson 45, data movement), or in network stack
functions (the networking track). A userspace profiler sees your code make a
syscall and go quiet — perf sees <em>into</em> the syscall and attributes the
time to the specific kernel function, turning "the program is slow but the CPU
looks idle in my profiler" into "it's spending 60% in the NFS client waiting on
the server." That whole-stack attribution is why perf is the tool that makes
this entire course's subsystems into a single diagnostic picture: the boundary
you studied from both sides, profiled as one.
</details>

---

## Homework

Take a real program you have (a build, a script, a service) and profile it
end to end: `perf stat` it first to classify (CPU-bound? memory-bound?
I/O-bound? — use IPC, cache-misses, and context-switches), then `perf record
-g` and `perf report` to find the hottest path. Write down: your hypothesis
before profiling, what the numbers actually showed, and whether they matched.
Then explain why "profile before optimizing" is the discipline this lesson
teaches — connecting to a specific way you could have wasted effort optimizing
the wrong thing.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise's value is in the gap between hypothesis and evidence — which is
almost always surprising. A representative result: you profile a data-processing
script you assumed was compute-bound (it does "a lot of math"), and
<code>perf stat</code> shows IPC 0.5 with high cache-misses and a large
context-switch count — it's actually memory-bound <em>and</em> spending time
blocked (I/O or lock waits), not compute-bound at all; <code>perf record -g</code>
reveals the hot path isn't your math kernel but JSON parsing allocating and
copying strings (memory churn — Lesson 18/29), or time in kernel read paths
(I/O-bound — Lesson 42). The math you were about to hand-optimize was 5% of the
time. Why "profile before optimizing" is the discipline: optimization effort
spent on the non-bottleneck yields <em>zero</em> speedup no matter how clever —
Amdahl's law (Lesson 26) is brutal, improving 5% of runtime by 2× saves 2.5%.
Concretely, you could have spent a week vectorizing the math kernel (the
"obviously expensive" part) for no measurable win, while the real fix — a
better data structure to avoid the allocations, or buffering to cut the I/O
syscalls (Lesson 02) — was a smaller change to the code you weren't looking at.
Human intuition about where time goes is reliably wrong: we over-weight
algorithmically-interesting code and under-weight the "boring" glue
(serialization, allocation, syscalls, waiting) that dominates real programs.
perf replaces intuition with measurement — <code>stat</code> to classify the
bottleneck's <em>nature</em> (so you fix data layout vs algorithm vs I/O — Q2's
divergent remedies), <code>record -g</code> to locate its <em>site</em> (so you
change the right code — Q3's call graph). This is the meta-skill the whole
Phase 13 builds toward: measure, then act — the same evidence-over-guessing
principle as reading /proc before restarting (Lesson 04), checking PSI before
tuning the scheduler (Lesson 15), and strace before assuming DNS (Lesson 03).
The system will tell you what's wrong if you ask it correctly; optimizing
without profiling is guessing with extra steps.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 64 — ftrace and kprobes →](lesson-64-ftrace-kprobes){: .btn .btn-primary }
