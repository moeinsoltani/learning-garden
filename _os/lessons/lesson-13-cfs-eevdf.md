---
title: "Lesson 13 — The Scheduler: CFS to EEVDF"
nav_order: 2
parent: "Phase 3: Scheduling"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 13: The Scheduler — CFS to EEVDF

## Concept

Ten runnable tasks, one CPU. Who runs next, and for how long? Linux's answer
for nearly two decades was the **Completely Fair Scheduler (CFS)**, replaced in
kernel 6.6 by its refinement **EEVDF** — both built on the same beautiful idea:

> Imagine an ideal CPU that could run all N tasks *simultaneously*, each at 1/N
> speed. Real hardware can't — so track how far each task has fallen behind its
> ideal share, and always run the one that's furthest behind.

The bookkeeping variable is **vruntime**: the CPU time each task has consumed
(weighted by priority — Lesson 14). Low vruntime = "owed CPU time" = runs next.

```
      per-CPU runqueue (sorted by vruntime)

   vruntime:   980ms      995ms      1010ms     1400ms
              ┌──────┐   ┌──────┐   ┌──────┐   ┌──────┐
              │ task │   │ task │   │ task │   │ task │
              │  A   │   │  B   │   │  C   │   │  D   │
              └──┬───┘   └──────┘   └──────┘   └──────┘
                 │ lowest → runs now; its vruntime grows as it runs;
                 ▼ when it passes B's, B takes over.  Self-balancing!
```

Watch what this does *without any special-casing*: a task that sleeps a lot (an
editor, a server waiting for requests) accumulates little vruntime, so whenever
it wakes it's "behind" and runs almost immediately — interactive
responsiveness falls out of pure fairness. A CPU hog racks up vruntime and
politely takes turns. Nobody starves: everyone's deficit eventually makes them
the neediest.

**EEVDF** ("Earliest Eligible Virtual Deadline First") keeps the vruntime
fairness and adds a **virtual deadline** per task, computed from its allotted
slice — fixing CFS's weak spot: predictable *latency*. Tasks that need short,
frequent turns get them on time, not just "eventually the same total".

---

## How It Works

### There is no global queue

Each CPU has its own runqueue — else every scheduling decision on a 128-core
box would fight over one lock. Fairness *across* CPUs comes from periodic
**load balancing**: idle CPUs pull work ("work stealing"), and the balancer
respects cache/NUMA topology (don't casually move a task away from its warm
cache — Lesson 15's affinity is you overriding this by hand).

### Timeslices are dynamic

There's no fixed quantum. The scheduler targets a *latency window* (a handful
of milliseconds) in which every runnable task should get a turn; the slice is
roughly window/N with a floor. Two runnable tasks → generous slices; twenty →
slivers (and correspondingly more context switches — Lesson 12's cost appears
exactly when you're most loaded).

### Sleepers, wakers, and preemption

When a sleeping task wakes (data arrived — Lesson 05's interrupt path), the
scheduler compares it with the running task; if the waker is sufficiently
behind (CFS) or has an earlier virtual deadline (EEVDF), it **preempts
immediately** — the nonvoluntary switch from Lesson 12, and the reason your
keystroke echoes during a kernel compile. Waking tasks get placed *near* the
current minimum vruntime — they must not bank unlimited credit while sleeping,
or a task that slept an hour would monopolize the CPU on waking.

### Observing it

`/proc/PID/sched` shows a task's vruntime (`se.vruntime`), slice counts, and
wait time; `/proc/sched_debug` (or `/sys/kernel/debug/sched/debug`) dumps every
runqueue. The `r` column of vmstat is the runnable count — the demand side of
the whole story.

{: .note }
> **Nothing here applies to the RT classes**
> This lesson describes SCHED_OTHER (a.k.a. SCHED_NORMAL) — the default class
> holding ~everything. SCHED_FIFO/SCHED_RR real-time tasks live outside the
> fairness world entirely and simply preempt all of it, which is precisely why
> they're dangerous (Lesson 14).

---

## Lab

```bash
# ---- 1. Two hogs share one CPU 50/50 — no configuration, pure fairness ----
$ taskset -c 0 bash -c 'while :; do :; done' &
$ taskset -c 0 bash -c 'while :; do :; done' &
$ sleep 3; ps -o pid,psr,pcpu,comm -p $(jobs -p)
#  PID PSR %CPU COMM
#  ...   0 49.9 bash      ← ~50% each, converging fast
#  ...   0 49.8 bash

# ---- 2. Add a third: 33/33/33 within seconds ----
$ taskset -c 0 bash -c 'while :; do :; done' &
$ sleep 3; ps -o pid,pcpu,comm -p $(jobs -p)
# ~33.3 each. The "ideal CPU at 1/N speed", made real by bookkeeping.

# ---- 3. Watch vruntime race upward ----
$ PID=$(jobs -p | head -1)
$ grep -E 'se.vruntime|nr_switches' /proc/$PID/sched
$ sleep 2
$ grep -E 'se.vruntime|nr_switches' /proc/$PID/sched
# vruntime climbed by ~2/3 of wall time (3 tasks sharing!); switches climbed too

# ---- 4. A sleeper stays responsive among hogs ----
# this loop sleeps 90% of the time — measure its wakeup latency while hogs run:
$ python3 - << 'EOF' &
import time
worst = 0
for _ in range(50):
    t = time.time(); time.sleep(0.1)
    late = (time.time() - t - 0.1) * 1000
    worst = max(worst, late)
print(f"worst wakeup lateness among 3 CPU hogs: {worst:.1f} ms")
EOF
$ taskset -c 0 python3 -c "print('hogs still on cpu0')" # (sleeper floats free)
$ wait %4 2>/dev/null
# worst lateness: typically ~1-2 ms — the sleeper's low vruntime wins instantly
# (pin the python to CPU 0 too with taskset if you want the harder test!)

# ---- 5. The demand side: runnable queue length ----
$ vmstat 1 3
#  r ... ← with 3 spinners: r ≈ 3-4. r persistently > nproc = CPU contention
$ kill $(jobs -p)

# ---- 6. Your kernel's scheduler generation ----
$ uname -r                    # ≥ 6.6 → EEVDF; older → CFS
$ ls /sys/kernel/debug/sched/ 2>/dev/null || sudo ls /sys/kernel/debug/sched/
# base_slice_ns (EEVDF's slice knob) — or latency_ns etc. under older CFS
```

---

## Further Reading

| Topic | Link |
|---|---|
| Completely Fair Scheduler | <https://en.wikipedia.org/wiki/Completely_Fair_Scheduler> |
| EEVDF scheduler | <https://en.wikipedia.org/wiki/Earliest_eligible_virtual_deadline_first_scheduling> |
| `sched(7)` — scheduling overview | <https://man7.org/linux/man-pages/man7/sched.7.html> |
| Scheduling (computing) | <https://en.wikipedia.org/wiki/Scheduling_(computing)> |
| `chrt(1)` man page | <https://man7.org/linux/man-pages/man1/chrt.1.html> |

---

## Checkpoint

**Q1.** Two CPU-bound tasks share one core — each gets 50%. Now one of them
starts sleeping half of every second. What share does each get, and what
happens to the sleeper's *latency* when it wakes?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The sleeper uses ~50% of the CPU only while awake — so it consumes up to ~50%
of total time (its awake half), and the hog absorbs everything the sleeper
doesn't use, ending near ~75% (its guaranteed fair half plus the sleeper's idle
half). Fairness is about <em>entitlement under contention</em>, not forced
equality — unused share flows to whoever wants it. Latency: excellent. While
sleeping, the sleeper's vruntime stands still while the hog's climbs, so at
every wakeup the sleeper is the furthest behind (EEVDF: earliest deadline) and
preempts the hog within microseconds–milliseconds. Sleep is automatically
rewarded with responsiveness — no interactivity heuristic needed.
</details>

**Q2.** Why does each CPU keep its own runqueue, and what problem does that
choice create that the kernel must then solve?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A single global queue would need a global lock touched on every scheduling
decision by every core — on many-core machines that lock becomes the system's
hottest cache line and scalability dies. Per-CPU runqueues make the common case
(pick next task on this core) lock-local and cache-friendly. The created
problem is <em>imbalance</em>: one CPU with five runnable tasks, another idle —
per-queue fairness is no longer global fairness. Hence the load balancer: idle
CPUs steal work, periodic rebalancing migrates tasks — but conservatively,
because migration destroys cache warmth (Lesson 12) and must respect NUMA
(Lesson 23). Every scheduler feature after the basic algorithm is managing this
tension between locality and balance.
</details>

**Q3.** A task that slept for an hour wakes up. Naively its vruntime is an hour
behind everyone's — it should own the CPU for a very long time to "catch up".
Why would that be wrong, and what does the scheduler actually do?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Fairness is meant to divide the CPU among tasks <em>competing now</em> — not to
pay reparations for demand that didn't exist. Letting the waker cash an hour of
credit would starve every current task (a cron job waking each hour would own
the machine), and sleep would become an exploit: sleep long, then burst with
impunity. So on wakeup the scheduler clamps the task's vruntime to
approximately the runqueue's current minimum (slightly behind it): enough to
win the next scheduling decision — preserving the snappy-wakeup property from
Q1 — but only a normal slice's worth of advantage, not an hour's. Old credit
expires; only <em>recent</em> behavior counts.
</details>

---

## Homework

Design the experiment before running it: you have `stress-ng --cpu N` (or your
spinner loops), `taskset`, and `/proc/PID/sched`. Predict, then verify: pin 4
spinners to one core, wait 10 seconds, and read all four `se.vruntime` values.
(a) How closely do they match? (b) Kill two; do the survivors' vruntimes
converge with each other or keep their gap? Explain what the answers
demonstrate about how fairness is enforced (hint: is vruntime *equalized* or is
its *growth rate* equalized?).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) The four vruntimes stay within a few milliseconds of each other — the
scheduler always runs the minimum, so no one can fall far behind or pull far
ahead: values are continuously herded together while competing. (b) The gap
between survivors <em>persists</em>: each now gets 50% and their vruntimes grow
at the same rate, but nothing ever subtracts from a vruntime or transfers
between tasks — there's no force pulling absolute values together, only the
"run the minimum" rule that stops divergence while they compete (any small gap
means the lower one runs a bit more until it passes the other, so they leapfrog
within a slice's width — but a pre-existing common offset vs the runqueue
simply remains). Demonstrated: fairness is enforced on <em>rates</em> (equal
progress per wall-clock second under contention), with vruntime as a relative
ordering device, not a bank balance to be settled.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 14 — Priorities: nice and real-time →](lesson-14-priorities-realtime){: .btn .btn-primary }
