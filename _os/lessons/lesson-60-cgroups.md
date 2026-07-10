---
title: "Lesson 60 — cgroups v2"
nav_order: 2
parent: "Phase 12: Containers & Resource Control"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 60: cgroups v2

## Concept

Namespaces (Lesson 59) control what a process can *see*; **cgroups** (control
groups) control what it can *use*. They're the second pillar of containers,
and the answer to the gap the last lesson ended on: a namespaced process can
still exhaust the host's CPU, memory, and I/O — cgroups bound it.

A cgroup is a group of processes with resource limits and accounting attached,
organized as a tree in a virtual filesystem:

```
   /sys/fs/cgroup/                     ← the unified v2 hierarchy (a filesystem!)
   ├── system.slice/                   ← systemd's services (Lesson 52)
   │   ├── nginx.service/
   │   │   ├── cgroup.procs      ← which PIDs are in this group
   │   │   ├── cpu.max           ← "50000 100000" = 0.5 CPU
   │   │   ├── memory.max        ← "512M" — hit it → OOM within the group (L22)
   │   │   ├── io.max            ← disk bandwidth cap (L42)
   │   │   └── memory.current    ← live usage accounting
   │   └── postgres.service/ ...
   └── user.slice/ ...
```

You've met cgroups repeatedly: Lesson 22's *safe* OOM lab ran inside one
(the bulkhead that killed only the cgroup); Lesson 15's PSI is per-cgroup;
Lesson 52's systemd `MemoryMax=`/`CPUWeight=` set cgroup files; virt Lesson 60
(Kata) and every container runtime use them. This lesson makes the mechanism
direct — you'll create cgroups by hand, move processes in, set limits, and
watch throttling happen.

The controllers (v2's unified set): **cpu** (weight for sharing + max for
hard caps), **memory** (max/high/min/low — the OOM and protection knobs),
**io** (bandwidth/IOPS limits + weights), **pids** (max process count —
fork-bomb defense), and more (hugetlb, cpuset for pinning — Lesson 15).

---

## How It Works

### The unified hierarchy (v2 vs the v1 mess)

cgroups v1 had a *separate tree per controller* (a process could be in
different cgroups for cpu vs memory — confusing, buggy interactions). **v2**
(the default on modern systems) has **one unified tree**: a process is in
exactly one cgroup, and all controllers apply there. You manage it by
reading/writing files under `/sys/fs/cgroup/` — echo a PID into
`cgroup.procs` to move it, write limits into the controller files. systemd
owns this tree (Lesson 52), which is why you set limits via unit properties
rather than poking files in production — but poking files by hand is how you
learn what those properties do.

### The key controller knobs

- **cpu.max** = `"quota period"` (µs): `50000 100000` = 50 ms of CPU per
  100 ms = half a core. Hard ceiling — throttled when exceeded (visible in
  `cpu.stat`'s `nr_throttled`). **cpu.weight** (1–10000, default 100) is the
  *soft* share under contention — the cgroup version of nice (Lesson 14).
- **memory.max** — hard limit; exceeding triggers reclaim then OOM *within the
  cgroup* (Lesson 22's bulkhead). **memory.high** — soft limit: throttle +
  aggressive reclaim, but no kill (graceful pressure — the Lesson 22 homework
  answer). **memory.low/min** — *protection*: reclaim avoids/never touches
  this cgroup's memory (protect the database from noisy neighbors).
- **io.max** — `"MAJ:MIN rbps=... wbps=... riops=... wiops="` per device
  (Lesson 42's device numbers) — bound the batch job's disk (Lesson 42's
  noisy-neighbor fix, now the real tool). **io.weight** — proportional share
  (what `ionice` wished it could do, Lesson 14).
- **pids.max** — cap the process/thread count: the fork-bomb defense every
  container needs.

### Accounting is half the value

Even without limits, cgroups *account*: `memory.current`, `cpu.stat`,
`io.stat`, and per-cgroup PSI (`cpu.pressure` etc. — Lesson 15) give precise
per-service resource attribution — "which service used what" — that's
otherwise hard to get (summing per-process stats misses shared pages,
kernel memory, etc.). This is why `systemctl status` shows a service's
resource use: it reads the cgroup.

{: .note }
> **cgroup namespace: the view, virtualized**
> The <em>cgroup namespace</em> (Lesson 59's eighth) virtualizes the
> <em>view</em> of the tree — inside a container, its own cgroup looks like
> the root, so it can't see (or escape toward) the host's hierarchy. Limits
> (this lesson) + isolation of the view (namespace) compose: the container is
> bounded <em>and</em> can't tell it's in a sub-tree.

---

## Lab

```bash
# ---- 1. Explore the hierarchy that's already running your system ----
$ mount | grep cgroup2                        # cgroup2 on /sys/fs/cgroup type cgroup2
$ ls /sys/fs/cgroup/                          # the tree root: slices + control files
$ cat /sys/fs/cgroup/cgroup.controllers       # available: cpu memory io pids ...
$ systemd-cgls | head -15                      # systemd's view of the tree (L52)
$ cat /proc/self/cgroup                        # which cgroup is YOUR shell in?

# ---- 2. Create a cgroup and put a process under a CPU cap ----
$ sudo mkdir /sys/fs/cgroup/lab
$ echo "+cpu +memory +pids" | sudo tee /sys/fs/cgroup/lab/../cgroup.subtree_control >/dev/null 2>&1 || \
    echo "+cpu +memory +pids" | sudo tee /sys/fs/cgroup/cgroup.subtree_control >/dev/null
$ echo "50000 100000" | sudo tee /sys/fs/cgroup/lab/cpu.max >/dev/null   # 0.5 CPU
# start a CPU hog and move it in:
$ bash -c 'while :; do :; done' & HOG=$!
$ echo $HOG | sudo tee /sys/fs/cgroup/lab/cgroup.procs >/dev/null
$ sleep 3; ps -o pid,pcpu,comm -p $HOG
#  %CPU ~50.0        ← capped at half a core despite an infinite loop!
$ cat /sys/fs/cgroup/lab/cpu.stat | grep -E 'nr_throttled|throttled_usec'
# nr_throttled: growing — the kernel is actively throttling it (the mechanism)
$ kill $HOG

# ---- 3. Memory limit → contained OOM (Lesson 22's bulkhead, direct) ----
$ echo "100M" | sudo tee /sys/fs/cgroup/lab/memory.max >/dev/null
$ echo 0 | sudo tee /sys/fs/cgroup/lab/memory.swap.max >/dev/null 2>&1
$ sudo bash -c 'echo $$ > /sys/fs/cgroup/lab/cgroup.procs && \
    python3 -c "b=bytearray(); 
while True: b += bytes(5_000_000)"'
# Killed              ← OOM-killed WITHIN the cgroup; host untouched (L22)
$ cat /sys/fs/cgroup/lab/memory.events | grep oom_kill    # oom_kill 1
$ cat /sys/fs/cgroup/lab/memory.current                   # usage, live

# ---- 4. pids.max: fork-bomb defense in one line ----
$ echo 20 | sudo tee /sys/fs/cgroup/lab/pids.max >/dev/null
$ sudo bash -c 'echo $$ > /sys/fs/cgroup/lab/cgroup.procs
    for i in $(seq 30); do sleep 60 & done 2>&1 | tail -1'
# fork: retry: Resource temporarily unavailable   ← capped at 20 procs.
# a fork bomb here exhausts ITS budget, not the host PID space (L06)
$ sudo pkill -f 'sleep 60' 2>/dev/null

# ---- 5. Per-cgroup accounting & PSI (Lesson 15, scoped) ----
$ cat /sys/fs/cgroup/lab/cpu.stat | head -3        # usage_usec, throttling
$ cat /sys/fs/cgroup/lab/memory.stat | head -5     # anon, file, kernel breakdown
$ cat /sys/fs/cgroup/lab/cpu.pressure 2>/dev/null  # PSI for THIS group only

# ---- 6. See systemd's real limits, then clean up ----
$ systemctl show systemd-journald -p MemoryMax -p CPUWeight -p TasksMax
# the same files, set declaratively via the unit (L52)
$ sudo rmdir /sys/fs/cgroup/lab                    # remove our lab cgroup
```

---

## Further Reading

| Topic | Link |
|---|---|
| cgroups v2 — kernel docs | <https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html> |
| cgroups (Wikipedia) | <https://en.wikipedia.org/wiki/Cgroups> |
| `systemd.resource-control(5)` | <https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html> |
| `cgroup_namespaces(7)` | <https://man7.org/linux/man-pages/man7/cgroup_namespaces.7.html> |
| Memory controller — kernel docs | <https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#memory> |
| `systemd-cgls(1)` / `systemd-cgtop(1)` | <https://www.freedesktop.org/software/systemd/man/latest/systemd-cgls.html> |

---

## Checkpoint

**Q1.** `memory.max` vs `memory.high` — one OOM-kills, one throttles. Which is
which, and when do you want each?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>memory.max</strong> is the hard limit: when the cgroup can't stay
under it even after reclaim (dropping cache, swapping — Lessons 20/21), the
kernel invokes the OOM killer <em>within the cgroup</em> (Lesson 22's
bulkhead) — a process dies (exit 137). <strong>memory.high</strong> is a soft
limit: crossing it triggers aggressive reclaim and <em>throttles</em> the
cgroup's processes (they're slowed, allocations stall) but nothing is killed —
usage is pushed back toward high by pressure, not force. When to use each:
<code>memory.high</code> for graceful degradation — a service that should slow
down and shed load under memory pressure rather than crash (and it gives the
app time to notice PSI/pressure and react — Lesson 22's homework answer),
often set as the primary limit with <code>memory.max</code> as a higher
backstop. <code>memory.max</code> alone for hard multi-tenant isolation where
a runaway tenant <em>must</em> be stopped and killing it is acceptable (the
container default — a container exceeding its memory is a bug, kill and
restart). Best practice combines them: high somewhat below max, so the group
first gets throttled+reclaimed (a soft warning the app can respond to) and only
a genuine runaway that blows past high hits max and dies. Plus
<code>memory.low</code>/<code>min</code> on the <em>other</em> direction —
protecting an important group's memory from reclaim when neighbors are
pressured (protect the database — Lesson 22 Q1's "make the right process die"
answered structurally: bound the offender with max, protect the victim with
low).
</details>

**Q2.** A namespaced process (Lesson 59) with no cgroup can still take down the
host. Give three concrete ways, and the specific cgroup controller that stops
each — tying the two pillars together.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Namespaces isolate <em>visibility</em>, not <em>consumption</em> — the process
sees its own world but draws from the host's real, shared resources
(Lesson 59's framing). Three attacks and their controllers: (1)
<strong>Memory exhaustion</strong> — allocate until the <em>host</em> OOM
killer fires, potentially killing critical host services or other containers
(Lesson 22's global-OOM lottery); stopped by <strong>memory.max</strong>,
which confines the OOM to the offending cgroup (the bulkhead). (2)
<strong>CPU monopolization</strong> — spin every core (an infinite loop per
CPU, or a crypto-miner), starving every other workload; stopped by
<strong>cpu.max</strong> (hard ceiling — the process is throttled to its quota,
lab 2) and/or <strong>cpu.weight</strong> (fair share under contention). (3)
<strong>Fork bomb / PID exhaustion</strong> — spawn processes until the host's
pid_max (Lesson 06) is consumed and nothing anywhere can fork; stopped by
<strong>pids.max</strong> (the fork bomb hits its own budget's wall, lab 4).
Bonus: (4) <strong>I/O saturation</strong> — flood the shared disk so every
other workload's latency explodes (Lesson 42's noisy neighbor); stopped by
<strong>io.max</strong>/<strong>io.weight</strong>. This is exactly why a
container is <em>namespaces + cgroups</em> and neither alone: namespaces make
the process think it's on its own machine, cgroups make sure it can't consume
more than its share of the real one. Docker/Kubernetes always set both — a
"container" with namespaces but no cgroup limits (or --memory unset) is a
loaded gun, which is why Kubernetes requests/limits map directly to cgroup
knobs and why unbounded pods are the classic cluster-outage cause.
</details>

**Q3.** Both cgroups (this lesson) and nice/ionice (Lesson 14) control
resource sharing. Explain what cgroups do that the process-level priority
tools fundamentally cannot — and why that difference matters for containers
and multi-tenancy.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
nice/ionice operate on <em>individual processes</em> and only express
<em>relative priority</em> (Lesson 14: nice tilts CPU shares under contention,
with no absolute guarantee — a nice-19 process still gets 100% of an idle
core). cgroups do three things process priorities structurally can't:
(1) <strong>Group semantics</strong> — limits apply to a <em>whole tree of
processes together</em>: a container spawning 50 workers is bounded as a unit
(its total CPU/memory/pids), whereas nice on 50 processes gives each its own
priority with no collective cap — you can't say "these 50 processes together
get at most 2 cores" with nice, but cpu.max on their cgroup does exactly that.
(2) <strong>Hard limits, not just shares</strong> — cpu.max/memory.max/io.max
are absolute ceilings enforced by throttling/OOM (lab 2–4), giving
<em>guarantees</em> and <em>isolation</em>; nice only reshuffles shares under
contention and vanishes when uncontended (Lesson 14 Q1), so it can't promise a
tenant "you get no more than X" or "you're protected to at least Y"
(memory.low). (3) <strong>Unified multi-resource + accounting</strong> — one
cgroup bounds CPU, memory, I/O, and pids coherently, with precise per-group
usage accounting (memory.current, cpu.stat, per-cgroup PSI), which
process-level tools scatter across separate mechanisms (nice, ionice, ulimit,
OOM adj) that don't compose or account as a unit. Why it matters for
containers/multi-tenancy: isolation between tenants requires <em>enforceable
group-level guarantees and limits</em> — tenant A must not be able to starve
tenant B regardless of how many processes A forks or how it's niced, and each
tenant needs both a ceiling (fairness/protection from noisy neighbors) and a
floor (guaranteed minimum). That's a group-level, absolute, multi-resource,
accountable contract — precisely cgroups' design and precisely what the older
per-process priority model was never meant to provide. It's why cgroups (2007,
from Google's container needs) were <em>the</em> enabling kernel feature for
containers: namespaces gave isolation of view, cgroups gave isolation of
resources — and modern cloud computing's whole "pack many tenants on one host
safely" premise rests on this controller.
</details>

---

## Homework

Reproduce Kubernetes' "requests and limits" model by hand with cgroups. Design
a two-tier setup: a `guaranteed.slice` for a latency-critical service and a
`besteffort.slice` for batch jobs, on a machine where they compete for CPU,
memory, and disk. Specify the exact cgroup knobs (weights, max, low) for each
tier and justify: how do you ensure the critical service always gets what it
needs while letting batch jobs use spare capacity — and what happens to each
tier under contention? Connect each choice to the controller's semantics.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Design — two sibling cgroups under root, using <em>shares for spare capacity</em>
+ <em>protection/limits for guarantees</em>: <br>
<strong>guaranteed.slice</strong> (the critical service):
<code>cpu.weight=10000</code> (max — wins CPU decisively under contention,
Lesson 14's weight as the cgroup version), <code>memory.min=&lt;working set&gt;</code>
(the strong guarantee — reclaim will <em>never</em> take this memory even under
host pressure, so the service never swaps/thrashes — Lesson 21), optionally
<code>memory.low</code> as a softer protected floor, and <code>io.weight=1000</code>
(prioritized disk). Deliberately <em>no</em> tight cpu.max (let it burst to
whatever it needs) — its guarantee comes from weight + memory protection, not
a cap. <br>
<strong>besteffort.slice</strong> (batch): <code>cpu.weight=10</code> (tiny
share — yields to guaranteed under contention but consumes ALL idle CPU when
guaranteed is quiet: "spare capacity" achieved, because weights only bind
under contention — Lesson 14 Q1), <code>memory.high=&lt;modest&gt;</code>
(throttle+reclaim it first when memory tightens — it degrades gracefully
rather than pressuring the critical tier), <code>memory.max</code> as a hard
backstop, <code>io.weight=10</code> + possibly <code>io.max</code> (cap its
disk so a batch scan can't saturate the device and inflate the critical
service's I/O latency — Lesson 42's noisy-neighbor fix). <br>
Behavior under contention: CPU — guaranteed's 1000:1 weight ratio means it
gets essentially all the CPU it asks for while batch fills only the remainder;
when guaranteed is idle, batch runs at full speed (weights don't waste idle
capacity — the key property vs hard caps). Memory — as the host pressures,
reclaim hits besteffort first (memory.high throttles it, its cache is
evicted), and guaranteed's memory.min is protected from reclaim entirely, so
the critical service's working set stays resident (no Lesson 21 thrashing);
if besteffort blows past memory.max it's OOM-killed <em>within its own slice</em>
(Lesson 22 bulkhead — the critical tier never sees the OOM). I/O — io.weight
prioritizes guaranteed's requests and io.max caps besteffort's bandwidth so
batch can't queue-flood the shared device. This IS Kubernetes' QoS model:
"Guaranteed" pods (requests=limits) map to strong cpu.weight + memory.min-style
protection, "Burstable"/"BestEffort" pods get low weights and get reclaimed/
evicted/OOM-killed first — all implemented as exactly these cgroup files the
kubelet writes. The design principle throughout: use <em>weights</em> for
sharing spare capacity efficiently (bind only under contention, never waste
idle resources), <em>protection</em> (min/low) for guarantees the critical
tier can rely on, and <em>limits</em> (max) to bulkhead the untrusted tier —
three distinct knob types for three distinct jobs, which is why cgroups v2
separates them and why the older single-priority-number model (nice) couldn't
express multi-tenant resource policy at all.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 61 — Container Images and overlayfs →](lesson-61-container-images){: .btn .btn-primary }
