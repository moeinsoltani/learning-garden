---
title: "Lesson 14 — Priorities: nice and real-time"
nav_order: 3
parent: "Phase 3: Scheduling"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 14: Priorities — nice and real-time

## Concept

Fair isn't always what you want. The backup job and the video call are not
equally important, and Linux gives you two very different levers to say so:

```
                        the priority landscape

  SCHED_FIFO/RR  prio 1..99   ═══ REAL-TIME: absolute rule ═══
                              │ any RT task preempts ALL of the
                              │ fair world, instantly, always
  ──────────────────────────────────────────────────────────────
  SCHED_OTHER    nice -20..19 ─── the fair world (Lesson 13) ───
                              │ nice tilts vruntime bookkeeping:
                              │ lower nice = heavier weight =
                              │ bigger share UNDER CONTENTION
  SCHED_BATCH                 │ like nice+, never preempts on wake
  SCHED_IDLE                  │ runs only when nothing else wants to
```

- **nice** (−20 loudest … +19 humblest) stays *inside* the fairness game: it
  reweights shares. A nice +19 task still runs — it just yields when contested.
- The **real-time classes** leave the game entirely: a SCHED_FIFO task runs
  *until it blocks or yields*, full stop. Nothing fair-world ever preempts it.
  Priority 50 beats 49, always and immediately.

The design consequence: nice is safe to sprinkle around; RT is a loaded weapon —
one spinning SCHED_FIFO task can freeze every shell, logger, and SSH session on
that CPU forever.

---

## How It Works

### nice = weights on vruntime

Each nice level multiplies the task's scheduling weight by ~1.25 (the magic
table: nice 0 = weight 1024). vruntime advances *inversely to weight*: a
nice −5 task's clock ticks slower, so it stays "behind" longer and runs more.
Two hogs at nice 0 vs nice 5 split the CPU roughly 75/25; 0 vs 19 is ~95/5.
Crucially, **an uncontended CPU makes nice invisible** — the humble task alone
on a core still gets 100%.

### The RT classes

- **SCHED_FIFO**: run until you block/yield; equal-priority peers queue in
  arrival order.
- **SCHED_RR**: FIFO plus a timeslice *among equals* (default 100 ms —
  `/proc/sys/kernel/sched_rr_timeslice_ms`).

Set with [`chrt`](https://man7.org/linux/man-pages/man1/chrt.1.html) (needs
`CAP_SYS_NICE` — Lesson 11's world). Who legitimately uses RT: audio servers
(PipeWire), PTP/time daemons (networking Lesson 72), industrial control,
`ksoftirqd`'s helpers under RT kernels. The kernel keeps a safety net:
`sched_rt_runtime_us` (default 950000/1000000) reserves 5% of each second for
non-RT tasks — the only reason an RT runaway leaves your SSH barely alive.

### Priority inversion

The classic trap: low-prio task L holds a lock; high-prio RT task H blocks on
it; medium task M preempts L forever → H waits on M. The Mars Pathfinder bug.
Fixes: **priority inheritance** (L temporarily borrows H's priority) — built
into RT-mutexes and `pthread_mutexattr_setprotocol(PTHREAD_PRIO_INHERIT)`;
plus the blunt rule: RT code paths must not share plain locks with non-RT code
(futex details in Lesson 26).

{: .note }
> **What about ionice and autogroups?**
> Two neighbors worth knowing: <code>ionice</code> sets <em>I/O</em> priority
> (block layer, Lesson 42 — separate from CPU); and desktop kernels group
> sessions into <em>autogroups</em> so `make -j64` in one terminal doesn't
> crush another session — nice applied between groups, not tasks. Server
> distros usually disable autogrouping; cgroup CPU weights (Lesson 60) are the
> production-grade version of the same idea.

---

## Lab

```bash
# ---- 1. nice shares under contention (pin everything to CPU 0) ----
$ taskset -c 0 nice -n 0  bash -c 'while :; do :; done' &
$ taskset -c 0 nice -n 5  bash -c 'while :; do :; done' &
$ taskset -c 0 nice -n 10 bash -c 'while :; do :; done' &
$ sleep 5; ps -o pid,ni,pcpu,comm -p $(jobs -p)
#  NI %CPU
#   0  ~56     ← weights 1024 : 335 : 110 ≈ 70:23:7 of the pie
#   5  ~29       (ratios approximate; watch them stabilize)
#  10  ~13

# ---- 2. nice is invisible without contention ----
$ kill $(jobs -p); taskset -c 0 nice -n 19 bash -c 'while :; do :; done' &
$ sleep 2; ps -o ni,pcpu -p $(jobs -p)
#  19  ~99      ← humblest priority, full CPU: nobody else wanted it
$ kill $(jobs -p)

# ---- 3. renice a running process (only root may go DOWN) ----
$ sleep 300 & renice +10 $! && ps -o pid,ni -p $!
$ renice -5 $! 2>&1
# renice: failed ... Permission denied   ← lowering needs CAP_SYS_NICE
$ sudo renice -5 $! && ps -o pid,ni -p $!; kill $!

# ---- 4. Real-time: absolute preemption (CAREFUL — pinned to CPU 0 only) ----
$ taskset -c 0 bash -c 'while :; do :; done' &            # fair-world hog
$ sudo taskset -c 0 chrt -f 50 bash -c 'i=0; while [ $i -lt 100000000 ]; do i=$((i+1)); done' &
$ sleep 2; ps -o pid,cls,rtprio,pcpu,comm -p $(jobs -p)
#  CLS RTPRIO %CPU
#  TS      -   ~0-5   ← the fair hog gets ONLY the 5% RT throttle leftover!
#  FF     50  ~95     ← FIFO owns the core until its loop ends
$ wait %2; kill %1     # RT task finishes by itself; then free the hog

# ---- 5. The 5% lifeline that saved you ----
$ sysctl kernel.sched_rt_runtime_us kernel.sched_rt_period_us
# 950000 / 1000000     ← RT may consume at most 95% of each second
# (set runtime to -1 for "no limit" — never do this on a box you like)

# ---- 6. Who runs RT on your VM already? ----
$ ps -eo cls,rtprio,comm | grep -v '^ *TS' | sort -u | head
#  FF 99 migration/0     ← kernel's own per-CPU helpers
#  FF 50 ...             ← maybe irq threads, audio, PTP daemons
```

---

## Further Reading

| Topic | Link |
|---|---|
| `sched(7)` — policies & priorities | <https://man7.org/linux/man-pages/man7/sched.7.html> |
| `nice(2)` / `setpriority(2)` | <https://man7.org/linux/man-pages/man2/setpriority.2.html> |
| `chrt(1)` man page | <https://man7.org/linux/man-pages/man1/chrt.1.html> |
| Priority inversion | <https://en.wikipedia.org/wiki/Priority_inversion> |
| Priority inheritance | <https://en.wikipedia.org/wiki/Priority_inheritance> |
| `ionice(1)` man page | <https://man7.org/linux/man-pages/man1/ionice.1.html> |

---

## Checkpoint

**Q1.** When does `nice 19` actually make a visible difference — always, or
only under contention? Explain via the weight/vruntime mechanism.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Only under contention. nice changes the task's <em>weight</em>, which scales
how fast its vruntime advances per second of CPU — i.e., how big its slice of a
<em>contested</em> pie is. Alone on a CPU, the runqueue has one task: it's
always the minimum vruntime, so it runs 100% regardless of weight — there's
nobody to be humble toward. The moment a heavier task shares the core, the
nice-19 task's fast-ticking clock makes it perpetually "ahead", so it yields
almost everything (~5%). This is exactly why nice is the right tool for batch
work: full speed on idle machines, near-invisible when real work arrives.
</details>

**Q2.** Why is a spinning SCHED_FIFO task catastrophic in a way a nice −20
spinner never is, and what two built-in mechanisms limit the damage?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
nice −20 is still inside the fair world: enormous weight, but every other task
keeps a nonzero share and keeps making progress — sluggish, not frozen. A
spinning SCHED_FIFO task is outside fairness: nothing in the fair world ever
preempts it, so on its CPU every normal task — shells, sshd, log daemons,
kworkers — gets exactly zero until it blocks, which a spinner never does. The
two damage limiters: (1) RT throttling — <code>sched_rt_runtime_us</code>
reserves 5% of each period for non-RT tasks (your barely-alive SSH); (2) the
privilege gate — entering RT classes requires CAP_SYS_NICE/RLIMIT_RTPRIO, so
random processes can't do this to you. Defense in practice: pin RT tasks to
dedicated cores (Lesson 15) so a bug freezes a core, not the box.
</details>

**Q3.** Describe priority inversion with the three-task pattern, and explain
how priority inheritance unwinds it.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Low-priority L takes a mutex. High-priority (RT) H wakes, preempts L, tries the
mutex, blocks — fine so far, H waits for one short critical section. But
medium-priority M (higher than L, lower than H) now preempts L indefinitely: L
never runs, never releases, and H — nominally the most important task on the
machine — waits on M's pleasure. The priority ordering has silently
<em>inverted</em> (H behind M). With priority inheritance, the moment H blocks
on the mutex, its priority is lent to the holder: L runs at H's priority,
M cannot preempt it, L finishes the critical section and releases, priorities
snap back, H proceeds. Cost: bookkeeping on every contended RT-mutex, and
chains (L waits on another lock…) must propagate the boost — which is why
PI-futexes exist as a distinct, more expensive flavor (Lesson 26).
</details>

---

## Homework

A build server runs CI jobs (CPU-heavy, throughput-matters) alongside a
latency-sensitive artifact-serving daemon. Design the priority scheme: what do
you set the CI jobs and the daemon to, what do you explicitly *not* use, and
what single measurement from Lesson 12 verifies your scheme works under full
CI load?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
CI jobs: <code>nice 10–19</code> (or better: a cgroup with low
<code>cpu.weight</code> — Lesson 60 — which also contains their thread count),
optionally <code>SCHED_BATCH</code> to suppress their wakeup preemption, and
<code>ionice -c3</code> so their disk traffic yields too (compilation is
I/O-heavy). The daemon: <strong>nice 0 is enough</strong> — as a mostly-sleeping
task it wins every wakeup against nice-19 hogs (Lesson 13's sleeper bonus does
the real work). Explicitly not: SCHED_FIFO for the daemon — it's tempting, but
a bug (spin, busy retry loop) then freezes the box, and it gains ~nothing over
the sleeper bonus; also avoid nice −20 "to be safe" (it starves kernel threads'
fair-world helpers). Verification: under full CI load, <code>pidstat -w -p
&lt;daemon&gt;</code> — its <strong>nonvoluntary</strong> context switches
should stay near zero and its request p99 flat; if nvcswch/s climbs, the daemon
is losing its CPU mid-request and the weights need another turn.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 15 — Affinity, Load, and Pressure →](lesson-15-affinity-psi){: .btn .btn-primary }
