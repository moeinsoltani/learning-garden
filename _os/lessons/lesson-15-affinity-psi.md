---
title: "Lesson 15 — Affinity, Load, and Pressure"
nav_order: 4
parent: "Phase 3: Scheduling"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 15: Affinity, Load, and Pressure

## Concept

Three questions close out the scheduling phase, and they're the three you'll
actually face in production:

1. **Can I choose *which* CPU a task runs on?** Yes — **affinity** (`taskset`,
   already sneaking into our labs). You've seen why it matters: cache warmth
   (Lesson 12), and fencing off RT tasks (Lesson 14). Virtualization Lesson 50
   used the same tool to pin vCPUs.
2. **Is this machine overloaded?** The traditional answer — **load average** —
   is one of the most misread numbers in Linux, because it counts more than CPU.
3. **What is the load actually *costing* my tasks?** The modern answer —
   **PSI** (Pressure Stall Information): for CPU, memory, and I/O, the percent
   of time tasks stood waiting for the resource. Load says "how long is the
   queue"; PSI says "how much did the queueing hurt".

```
   load average               vs                PSI
   ─────────────                                ───
   count of tasks either                        % of wall time in which
   RUNNABLE (want CPU)                          tasks were STALLED waiting:
   or in D-STATE (Lesson 08 —                     cpu:    want CPU, none free
   waiting on disk/NFS!)                          memory: waiting on reclaim/swap
   averaged over 1/5/15 min                       io:     waiting on disk
   → an ambiguous queue length                  → an unambiguous cost, per resource
```

The infamous trap: load average 8 on a 4-core box *might* mean CPU contention —
or eight processes stuck in D-state on a dead NFS mount while CPUs sit idle.
PSI splits the ambiguity: `cpu` pressure high → real CPU contention;
`io`/`memory` pressure high → the wait is elsewhere.

---

## How It Works

### Affinity

Each task carries a CPU bitmask (`Cpus_allowed` in `/proc/PID/status`); the
scheduler and load balancer (Lesson 13) will only place it on set bits. Set it
with `taskset -c 1,3 cmd` (launch) or `taskset -cp 1,3 PID` (running; threads
individually via `-a`). Children inherit the mask. `PSR` in `ps -o psr` shows
where a task last ran. The systemd/cgroup version — `cpuset` (Lesson 60,
`AllowedCPUs=`) — is affinity applied to whole service trees, and `isolcpus`/
`nohz_full` (Lesson 05) removes cores from general scheduling entirely for the
DPDK/RT crowd.

When to pin: keeping a latency-critical task's cache warm; separating noisy
neighbors; IRQ/app colocations (networking Lesson 63's RSS steering). When not
to: by default. The balancer is good; a stale mask after a resize is a classic
self-inflicted wound.

### Load average, precisely

The three numbers are exponentially-damped averages (1/5/15 min) of the count
of tasks in state R (runnable — running *or* queued) **plus state D**
(uninterruptible — Lesson 08). Linux included D deliberately ("load" = demand
on the *system*, not just CPU), and that decision is exactly what makes the
number ambiguous alone. Rules: compare against `nproc`; the 1-vs-15-min trend
tells you arriving vs draining; and always confirm *composition* — `vmstat`'s
`r` and `b` columns split R from D on the spot.

### PSI

`/proc/pressure/{cpu,memory,io}` each show:
`some avg10=X avg60=Y avg300=Z total=N` — "some" = at least one task stalled
(`full` = *all* non-idle tasks stalled — the whole machine waiting). avg10=25
for io means: over the last 10 s, 25% of the time some task sat blocked on
disk. Per-cgroup PSI files give the same per service (Lesson 60), and PSI
supports *triggers* — poll an fd, get woken when pressure crosses a threshold —
which is how systemd-oomd and Android's memory killer act *before* the OOM
disaster (Lesson 22).

{: .note }
> **PSI needs a modern kernel**
> PSI exists since 4.20 and is enabled on all mainstream distros. If
> <code>/proc/pressure</code> is missing, the kernel booted with
> <code>psi=0</code> — check <code>/proc/cmdline</code> (Lesson 50).

---

## Lab

```bash
# ---- 1. Affinity basics ----
$ nproc                                   # know your ceiling (assume 2+)
$ taskset -c 1 bash -c 'while :; do :; done' &
$ sleep 1; ps -o pid,psr,pcpu,comm -p $!
#  PSR 1  %CPU ~100        ← lives on CPU 1, as ordered
$ grep Cpus_allowed_list /proc/$!/status
# Cpus_allowed_list: 1
$ taskset -cp 0 $!                        # migrate it live
$ sleep 1; ps -o pid,psr -p $!            # PSR: 0 now
$ kill $!

# ---- 2. Load average vs reality, scenario A: pure CPU ----
$ for i in $(seq $(nproc)); do (while :; do :; done) & done
$ sleep 65; uptime; vmstat 1 3
# load 1-min ≈ nproc; vmstat: r ≈ nproc, b = 0, CPU us ≈ 100%
$ cat /proc/pressure/cpu
# some avg10=~XX — CPU pressure real and honest
$ kill $(jobs -p)

# ---- 3. Scenario B: same load number, ZERO cpu use (D/blocked tasks) ----
# simulate I/O-stuck tasks: heavy sync writers
$ for i in 1 2 3 4; do (dd if=/dev/zero of=/tmp/l$i bs=64K count=4000 oflag=dsync 2>/dev/null; rm /tmp/l$i) & done
$ sleep 5; uptime; vmstat 1 3
# load climbing toward 4 — but CPU idle high, b column > 0!
$ cat /proc/pressure/io /proc/pressure/cpu
# io: some avg10 HIGH; cpu: some avg10 ~0   ← PSI names the true bottleneck
$ wait

# ---- 4. Read all three pressures at a glance ----
$ tail -n +1 /proc/pressure/*
# ==> /proc/pressure/cpu <==     some avg10=0.00 ...
# ==> /proc/pressure/io <==      some avg10=... full avg10=...
# ==> /proc/pressure/memory <==  some avg10=0.00 (memory pressure: Lesson 21!)

# ---- 5. The composition check you'll use forever ----
$ vmstat 1 5
# r = runnable (CPU demand)   b = blocked/D (I/O demand)
# load ≈ r + b, but ONLY this split (+ PSI) tells you which knob to turn

# ---- 6. Pin + isolate pattern (RT lesson's fence, done properly) ----
$ taskset -c 0 bash -c 'while :; do :; done' &   # noisy neighbor on CPU 0
$ taskset -c 1 python3 - << 'EOF'                # sensitive task on CPU 1
import time
worst = 0
for _ in range(30):
    t = time.time(); time.sleep(0.05)
    worst = max(worst, (time.time()-t-0.05)*1000)
print(f"worst lateness on private CPU: {worst:.2f} ms")
EOF
$ kill $(jobs -p)
# sub-millisecond — the neighbor never touches CPU 1
```

---

## Further Reading

| Topic | Link |
|---|---|
| Load (computing) | <https://en.wikipedia.org/wiki/Load_(computing)> |
| PSI — kernel documentation | <https://www.kernel.org/doc/html/latest/accounting/psi.html> |
| `taskset(1)` man page | <https://man7.org/linux/man-pages/man1/taskset.1.html> |
| `sched_setaffinity(2)` | <https://man7.org/linux/man-pages/man2/sched_setaffinity.2.html> |
| `uptime(1)` man page | <https://man7.org/linux/man-pages/man1/uptime.1.html> |
| Processor affinity | <https://en.wikipedia.org/wiki/Processor_affinity> |

---

## Checkpoint

**Q1.** Load average is 8 on a 4-core box but CPU usage is 5%. What's going on,
and which two files/commands confirm it in under a minute?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The load isn't CPU demand: Linux's load average counts D-state
(uninterruptible) tasks alongside runnable ones, so ~8 tasks are almost
certainly blocked on I/O or a dead remote filesystem while the CPUs idle.
Confirm: <code>vmstat 1</code> — r ≈ 0, b ≈ 8 (blocked, not runnable); and
<code>/proc/pressure/io</code> vs <code>/proc/pressure/cpu</code> — io "some"
high, cpu ~0. Then find the victims: <code>ps -eo state,wchan,comm | grep
^D</code> and read their wchan/stack (Lesson 08's diagnosis) — classically an
NFS or dying-disk function. The fix is storage-side; no scheduler knob helps.
</details>

**Q2.** When is pinning a task to specific CPUs the right call, and what's the
classic way affinity masks go wrong operationally?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Right calls: latency-critical tasks that profit from a permanently warm cache
(trading systems, packet processing); fencing noisy neighbors or RT tasks onto
private cores; aligning a task with the CPU handling its NIC IRQs, or with the
NUMA node holding its memory (Lesson 23, virt Lesson 50). Wrong by default
otherwise: the load balancer handles placement well, and a mask is a hard
constraint the scheduler cannot override. Classic failure: masks outlive their
context — the box gains CPUs (or the VM is resized) and pinned services keep
crowding the original two cores; or two teams pin to the same "obviously free"
core. Affinity is configuration debt: document it, and prefer cgroup cpusets
that are owned by one authority (systemd) over scattered tasksets.
</details>

**Q3.** Explain the difference between PSI's `some` and `full` lines, and why
`full` for CPU is (almost) always zero.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<code>some</code> = fraction of time at least one task was stalled on the
resource (somebody queued — partial slowdown). <code>full</code> = fraction of
time <em>every</em> non-idle task was simultaneously stalled — the machine as a
whole made no progress, pure waste (severe memory thrash or a storage stall
freezing everything). For CPU, <code>full</code> can't meaningfully happen
system-wide: if all tasks were waiting for CPU, the CPUs would be… running one
of them — some task is always executing, so CPU stall is inherently partial.
(Per-cgroup CPU <code>full</code> does exist: an entire <em>cgroup</em> can be
starved while other groups run.) Practical reading: alert on memory/io
<code>full</code> at all, and on sustained <code>some</code> growth for
capacity planning.
</details>

---

## Homework

Build a one-screen "is this box in trouble?" triage script: it should print
nproc, the three load averages, vmstat's r/b split (one sample), and the avg10
`some` value from each of the three PSI files — followed by a verdict line per
resource ("CPU: contended/ok", "IO: …", "MEM: …") using thresholds you choose
and justify. Run it during lab scenarios A and B and check it names the right
culprit both times.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<pre>
#!/bin/bash
n=$(nproc)
read l1 l5 l15 _ < /proc/loadavg
rb=$(vmstat 1 2 | tail -1); r=$(awk '{print $1}' <<<"$rb"); b=$(awk '{print $2}' <<<"$rb")
psi() { awk -v r="$1" '/^some/{gsub("avg10=","",$2); print $2}' /proc/pressure/$1; }
cpu=$(psi cpu); io=$(psi io); mem=$(psi memory)
echo "cores=$n load=$l1/$l5/$l15 runnable=$r blocked=$b psi10 cpu=$cpu io=$io mem=$mem"
awk -v v="$cpu" 'BEGIN{print "CPU:", (v>20 ? "contended" : "ok")}'
awk -v v="$io"  'BEGIN{print "IO: ", (v>15 ? "stalling"  : "ok")}'
awk -v v="$mem" 'BEGIN{print "MEM:", (v>5  ? "pressure"  : "ok")}'
</pre>
Threshold logic: cpu some&gt;20% ≈ tasks losing a fifth of their time to
queueing (users feel it); io&gt;15% because I/O stalls compound into
application timeouts; mem&gt;5% because <em>any</em> sustained memory pressure
precedes reclaim spirals (Lesson 21) and deserves early attention. In scenario
A the script shows r≈n, cpu high, io~0 → "CPU: contended"; in scenario B, b&gt;0,
io high, cpu ok → "IO: stalling" — the verdicts diverge exactly where load
average alone stayed ambiguous. Keep the script; Phase 13's capstone extends it.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 4 — Memory (Lesson 16: Virtual Memory) →](lesson-16-virtual-memory){: .btn .btn-primary }
