---
title: "Lesson 12 — Context Switches"
nav_order: 1
parent: "Phase 3: Scheduling"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 12: Context Switches

## Concept

Your VM has 2 CPUs and ~200 processes. The illusion that all 200 are "running"
is manufactured by one operation repeated hundreds of times per second: the
**context switch** — freezing one task mid-instruction and thawing another
exactly where it left off.

```
  CPU running A                                CPU running B
  ─────────────                                ─────────────
  A's registers, A's stack,     SWITCH         B's registers, B's stack,
  A's address space    ──────────────────▶     B's address space
                        1. save A's registers
                           into A's task_struct
                        2. switch kernel stack
                        3. switch page tables (CR3)
                           ← the expensive part:
                             TLB flush! (Lesson 17)
                        4. restore B's registers
                        5. return... as B
```

The switch itself is fast (~1–2 µs). The real cost is **what it ruins**: the
CPU caches are full of A's data, the TLB is full of A's translations — B starts
"cold" and pays thousands of cache misses to warm up. This indirect cost is why
too many context switches show up as *the whole machine feels slow* while no
single tool says why.

There are two very different reasons a task leaves the CPU, and telling them
apart is diagnostic gold (you saw the counters in Lesson 05's lab):

- **Voluntary** — the task *blocked*: it asked for something not ready (read
  from an idle socket, a lock, a sleep). Normal and healthy for I/O-bound work.
- **Nonvoluntary (preemption)** — the tick or a woken higher-priority task
  yanked the CPU away mid-computation. A few are normal; floods mean CPU
  contention.

---

## How It Works

### What must be saved

Per-task: the CPU registers (including instruction pointer — the "where was I"),
the kernel stack it was using, and FPU/SIMD state (saved lazily — big registers,
often untouched). Per-*process* (only when switching between different
processes): the address space — reloading the page-table base register flushes
the TLB. Switching between two *threads* of the same process skips that part —
one concrete reason thread switches are cheaper than process switches
(Lesson 24 pays this off).

### A syscall is NOT a context switch

Crossing into the kernel (Lesson 02) keeps running *the same task* — same
address space, just privileged mode. Cost ~100–200 ns. A context switch changes
*which task exists on the CPU* — ~1–2 µs direct plus cache/TLB damage. Order of
magnitude apart, constantly confused. An interrupt (Lesson 05) is also not a
switch — unless the handler wakes someone who then preempts you.

### Where switches come from

Blocking syscalls (read/poll/futex — voluntary), the scheduler tick ending a
timeslice (nonvoluntary), and **wakeup preemption**: when an event makes a
"more deserving" task runnable (Lesson 13 defines deserving), the kernel may
preempt the current task immediately — that's how your keystroke gets handled
within microseconds even on a busy box.

{: .note }
> **Reading the numbers**
> Rules of thumb: an I/O-heavy service showing 10k–100k <em>voluntary</em>
> switches/s per busy thread is normal life. <em>Nonvoluntary</em> counts
> rivaling voluntary ones on a latency-sensitive service mean CPU contention —
> someone's stealing your timeslices (fix: fewer runnable threads, priorities,
> or affinity — Lessons 14–15). System-wide totals live in
> <code>vmstat</code>'s <code>cs</code> column; per-process in
> <code>pidstat -w</code> and <code>/proc/PID/status</code>.

---

## Lab

```bash
# ---- 1. System-wide switch rate ----
$ vmstat 1 5
#  r  b ... in    cs     ← cs = context switches/s; in = interrupts/s
#  0  0 ... 150   300      idle VM: a few hundred
# leave this running in a second terminal for the rest of the lab

# ---- 2. Voluntary: an I/O-bound task ----
$ ping -i 0.01 -q localhost > /dev/null &
$ sleep 3; pidstat -w -p $! 1 3
#   cswch/s  nvcswch/s
#   ~200        ~0        ← blocks on the timer/socket 100x/s: all voluntary
$ kill %1

# ---- 3. Nonvoluntary: pure CPU hogs fighting ----
$ NCPU=$(nproc)
$ for i in $(seq $((NCPU + 2))); do (while :; do :; done) & done   # CPUs+2 spinners
$ sleep 3; pidstat -w 1 3 | grep -E 'while|bash' | head -8
#   cswch/s ~0, nvcswch/s ~10-100 each  ← never block; constantly PREEMPTED
$ grep -E 'ctxt' /proc/$(jobs -p | head -1)/status 2>/dev/null
$ kill $(jobs -p)

# ---- 4. Measure the raw switch cost with a ping-pong pipe ----
# two processes alternately read/write 1 byte: every hop = 2 switches
$ cat > /tmp/pingpong.c << 'EOF'
#include <unistd.h>
#include <stdio.h>
#include <sys/wait.h>
#include <time.h>
#define N 100000
int main(void){
    int a[2], b[2]; pipe(a); pipe(b); char c = 'x';
    struct timespec t0, t1;
    if (fork() == 0) {                    /* child: echo forever */
        for (int i = 0; i < N; i++) { read(a[0], &c, 1); write(b[1], &c, 1); }
        _exit(0);
    }
    clock_gettime(CLOCK_MONOTONIC, &t0);
    for (int i = 0; i < N; i++) { write(a[1], &c, 1); read(b[0], &c, 1); }
    clock_gettime(CLOCK_MONOTONIC, &t1);
    wait(NULL);
    double ns = (t1.tv_sec - t0.tv_sec) * 1e9 + (t1.tv_nsec - t0.tv_nsec);
    printf("%.0f ns per round-trip (≈2 switches + 2 syscall pairs)\n", ns / N);
    return 0;
}
EOF
$ gcc -O2 -o /tmp/pingpong /tmp/pingpong.c && /tmp/pingpong
# ~4000-10000 ns per round-trip on a typical VM
# → a couple of µs per switch, just as promised. Watch cs explode in vmstat!

# ---- 5. Pin both to ONE cpu — more switches, but no cache bouncing ----
$ taskset -c 0 /tmp/pingpong
# often FASTER per round-trip on one core (shared L1/L2!) — preview of Lesson 15

$ rm /tmp/pingpong /tmp/pingpong.c
```

---

## Further Reading

| Topic | Link |
|---|---|
| Context switch (Wikipedia) | <https://en.wikipedia.org/wiki/Context_switch> |
| `pidstat(1)` man page | <https://man7.org/linux/man-pages/man1/pidstat.1.html> |
| `vmstat(8)` man page | <https://man7.org/linux/man-pages/man8/vmstat.8.html> |
| Translation lookaside buffer | <https://en.wikipedia.org/wiki/Translation_lookaside_buffer> |
| `taskset(1)` man page | <https://man7.org/linux/man-pages/man1/taskset.1.html> |

---

## Checkpoint

**Q1.** Which workload context-switches more — a busy web server or a video
encoder — and what does the voluntary/nonvoluntary mix look like for each?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The web server, by orders of magnitude: its life is short bursts between
blocking points (accept, read request, wait on database, write response) — each
block is a <em>voluntary</em> switch, thousands per second per core. Its
nonvoluntary count stays low unless CPUs are oversubscribed. The encoder is the
opposite: it computes for its entire timeslice and gets <em>preempted</em> by
the tick — purely nonvoluntary switches at a low steady rate (about the tick
rate per CPU), voluntary near zero except when reading input frames. The mix is
a fingerprint of I/O-bound vs CPU-bound — worth checking before any tuning.
</details>

**Q2.** Rank by cost and justify: (a) a getpid() syscall, (b) a context switch
between two threads of one process, (c) a context switch between two different
processes.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) &lt; (b) &lt; (c). The syscall (~100–200 ns) changes privilege but not
identity — same task, same address space, same caches. The thread switch
(~1 µs) saves/restores registers and kernel stacks and runs the scheduler, but
keeps the address space — page tables stay, TLB survives. The process switch
adds the address-space change: new page-table base, TLB flushed/invalidated,
and the new process's working set isn't in cache — its true cost is paid over
the next milliseconds as misses. (Hardware TLB tagging (PCID) and Lesson 17
soften but don't erase the difference.)
</details>

**Q3.** In the lab, pinning the ping-pong pair to a single CPU often made
round-trips *faster*, even though the two processes must now take strict turns.
Explain the mechanism.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
On two CPUs, each wakeup crosses cores: CPU 0 writes, then pokes CPU 1 (an IPI
— inter-processor interrupt), whose task wakes with cold L1/L2 for the shared
data, and every byte handed over migrates between the cores' caches. On one
CPU, the pair shares a warm L1/L2 — the pipe buffer and both tasks' hot state
stay resident — and the switch between them is exactly the cheap same-CPU path.
When the actual work per message is tiny, cache locality beats parallelism.
The general lesson (Lesson 15, and virt CPU-pinning): placement matters, and
"more cores" is not automatically faster for chatty, fine-grained communication.
</details>

---

## Homework

Your monitoring shows a latency-sensitive service's p99 degraded after a deploy
that added a background batch job to the same box. Using only `pidstat -w`,
`vmstat`, and `/proc/PID/status`, design the 5-minute investigation that
confirms or refutes "the batch job is stealing CPU from the service" — state
the exact numbers you'd compare and the two possible verdicts.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Baseline first: <code>vmstat 1</code> — is the run queue (r) exceeding CPU
count and did cs jump vs pre-deploy? Then the key comparison:
<code>pidstat -w -p &lt;service&gt; 1 60</code> — watch
<strong>nvcswch/s</strong>. Verdict A (confirmed): service's nonvoluntary
switches rose sharply post-deploy (check
<code>nonvoluntary_ctxt_switches</code> growth in /proc/PID/status over a fixed
interval), r &gt; nproc while the batch job shows high CPU — the service is
being preempted mid-request; fix with nice/cgroup CPU limits (Lessons 14, 60)
or separate cores (Lesson 15). Verdict B (refuted): nvcswch/s unchanged and r
&le; nproc — the CPU isn't contended; suspect instead memory pressure or I/O
from the batch job (page cache eviction — Lesson 20's territory, PSI in
Lesson 15 would show it), and the context-switch data just saved you from
tuning the wrong thing.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 13 — The Scheduler: CFS to EEVDF →](lesson-13-cfs-eevdf){: .btn .btn-primary }
