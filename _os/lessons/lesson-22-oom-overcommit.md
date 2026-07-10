---
title: "Lesson 22 — Overcommit and the OOM Killer"
nav_order: 7
parent: "Phase 4: Memory"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 22: Overcommit and the OOM Killer

## Concept

This phase has been a chronicle of promises: malloc promises (Lesson 18),
faults deliver (Lesson 19), reclaim and swap stretch the truth further
(Lessons 20–21). This lesson is about the day the promises exceed all
stretching — and the kernel must either refuse new promises or break old ones.

**Overcommit** is the policy of promising more virtual memory than
RAM + swap could ever back. It sounds reckless; it's essential. Recall the
evidence you've already gathered: your 300 MB bytearray that used 3 MB
(Lesson 16), fork's CoW sharing where a 2 GiB process "duplicates" for free
(Lesson 07), the zero page (Lesson 19). Programs *systematically* ask for far
more than they touch — refusing those asks would waste most of RAM and break
fork entirely.

```
  vm.overcommit_memory:
    0 (default)  heuristic: refuse only the absurd; promise the plausible
    1            always promise everything (some allocators/redis like this)
    2            never overcommit: promises capped at swap + RAM×ratio —
                 malloc actually FAILS again. Predictable; wasteful; rare.

  But promises are broken at TOUCH time, not ask time:
  a write faults (L19) → no frame free → reclaim fights (L21) → nothing left?
                        ┌──────────────────────────────┐
                        │       OOM KILLER             │
                        │ pick the process whose death │
                        │ frees the most, hurts least: │
                        │ SIGKILL. No appeal. (L09!)   │
                        └──────────────────────────────┘
```

The genuinely uncomfortable part: with overcommit, **a correct program can be
killed for another program's appetite** — your innocent write to memory you
"successfully allocated" can arrive at the moment the system is bankrupt. The
OOM killer is the kernel choosing *which* process dies so the kernel itself
(and hopefully the important work) survives.

---

## How It Works

### Who dies: oom_score

The victim is chosen by score, visible per process in `/proc/PID/oom_score`:
essentially *fraction of usable memory this process consumes* (RSS + swap +
page tables), adjusted by `/proc/PID/oom_score_adj` (−1000 … +1000): −1000 =
never kill ("OOM immune" — sshd on many distros!), +1000 = kill me first.
systemd exposes it as `OOMScoreAdjust=`; Kubernetes sets it per-QoS-class.
The kill is logged loudly in `dmesg` — a full table of candidates, the chosen
victim, and why. Reading that report is a core production skill (the lab
practices it).

### cgroups change the blast radius

With cgroup memory limits (Lesson 60), OOM happens *per group*: a container
hitting its `memory.max` triggers an OOM kill **inside that group only** —
the host is fine; the noisy tenant eats its own children. This is the single
biggest practical improvement: global OOM is a lottery; cgroup OOM is a
bulkhead. `memory.events` (oom, oom_kill counters) is where container
platforms watch. Userspace killers (systemd-oomd, earlyoom) go one step
earlier: they act on PSI pressure (Lesson 15) *before* the kernel's last-resort
moment, killing chosen victims while the system is still responsive.

### Why not just fail malloc?

Mode 2 exists — and it resurrects the C-textbook world where malloc returns
NULL. The costs: fork breaks for large processes (the CoW promise must be
fully reserved — a 4 GiB process can't fork without 4 GiB of headroom), memory
utilization craters (all those untouched promises need reservation), and in
practice most software handles NULL malloc worse than it handles being killed
(half-completed operations, no cleanup paths tested). The industry's actual
posture: overcommit + cgroup bulkheads + pressure-based early action.

{: .note }
> **The daily-life version**
> You've almost certainly met the OOM killer as: "my training job / build /
> browser tab vanished, exit code 137." 137 = 128 + 9: killed by SIGKILL
> (Lesson 08's arithmetic) — and <code>dmesg | grep -i oom</code> tells you
> it was the kernel, not a colleague.

---

## Lab

```bash
# ---- 1. Overcommit in numbers ----
$ cat /proc/sys/vm/overcommit_memory
# 0
$ grep -E 'CommitLimit|Committed_AS' /proc/meminfo
# Committed_AS (all promises) routinely EXCEEDS CommitLimit — running on trust

# ---- 2. Watch a doomed promise get made — and kept until touched ----
$ python3 -c "
b = bytearray(int(50e9))     # 50 GB 'allocated' on a 4 GB VM. Sure, says mode 0!
print('promise accepted')
input('now touching would be... unwise. Enter to exit.')"
# promise accepted            ← VmSize +50GB, RSS ~0. The fiction at its boldest.

# ---- 3. Trigger a REAL OOM kill — safely, inside a cgroup bulkhead ----
$ sudo mkdir /sys/fs/cgroup/oomlab
$ echo 200M | sudo tee /sys/fs/cgroup/oomlab/memory.max
$ echo 0    | sudo tee /sys/fs/cgroup/oomlab/memory.swap.max   # no escape via swap
$ sudo bash -c 'echo $$ > /sys/fs/cgroup/oomlab/cgroup.procs && \
    python3 -c "
b = bytearray()
while True: b += bytes(10_000_000)   # eat until the wall
"'
# Killed                       ← SIGKILL from the kernel. Host: untouched.
$ echo $?
# 137

# ---- 4. Read the kill report like an incident responder ----
$ sudo dmesg | grep -iA 20 'memory cgroup out of memory' | tail -25
# memory: usage 204800kB, limit 204800kB ...
# oom-kill: constraint=CONSTRAINT_MEMCG ... 
# Killed process 5432 (python3) total-vm:...kB, anon-rss:...kB
#   → WHO (comm, pid), WHY (memcg limit), WHAT it held (anon-rss!)
$ cat /sys/fs/cgroup/oomlab/memory.events
# oom 1  oom_kill 1            ← the counter platforms alert on
$ sudo rmdir /sys/fs/cgroup/oomlab

# ---- 5. Steer the killer: oom_score_adj ----
$ sleep 300 & S=$!
$ cat /proc/$S/oom_score
$ sudo bash -c "echo -1000 > /proc/$S/oom_score_adj"   # immune
$ cat /proc/$S/oom_score
# 0 — off the menu. (+1000 would make it the designated sacrifice.)
$ kill $S
$ ps -o pid,comm $(pgrep sshd | head -1) && cat /proc/$(pgrep sshd | head -1)/oom_score_adj
# -1000 on many distros: your lifeline stays alive by policy, not luck

# ---- 6. Who's on the menu right now? ----
$ for p in $(ls /proc | grep -E '^[0-9]+$'); do
    s=$(cat /proc/$p/oom_score 2>/dev/null); [ "${s:-0}" -gt 50 ] && \
    echo "$s $(cat /proc/$p/comm 2>/dev/null)"; done | sort -rn | head -5
# your biggest RSS holders — exactly who dmesg would name tonight
```

---

## Further Reading

| Topic | Link |
|---|---|
| Out of memory / OOM killer | <https://en.wikipedia.org/wiki/Out_of_memory> |
| Memory overcommitment | <https://en.wikipedia.org/wiki/Memory_overcommitment> |
| Overcommit accounting — kernel docs | <https://www.kernel.org/doc/html/latest/mm/overcommit-accounting.html> |
| cgroup v2 memory controller | <https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#memory> |
| `proc(5)` — oom_score fields | <https://man7.org/linux/man-pages/man5/proc.5.html> |
| systemd-oomd | <https://www.freedesktop.org/software/systemd/man/latest/systemd-oomd.service.html> |

---

## Checkpoint

**Q1.** Your database got OOM-killed instead of the obviously-leaking log
shipper. Why might the killer choose that way, and what two knobs make the
right process die next time?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The killer scores by <em>current footprint</em>, not by blame: the database
legitimately holds the biggest anon-RSS (its buffer pool), while the leaking
shipper — though growing — may still be smaller at the fatal moment. Nothing
in oom_score knows "leak" from "working set". Knobs: (1)
<code>oom_score_adj</code> — protect the database (negative; systemd
<code>OOMScoreAdjust=-800</code>) and/or mark the shipper expendable
(positive); (2) better — <strong>cgroup bulkheads</strong>: give the shipper
its own <code>memory.max</code> so its leak OOMs <em>its own group</em> long
before global pressure (and <code>memory.low</code> protection for the
database's group). Trade-off of the adj-only route: an immune database on a
still-bankrupt host just redirects the massacre to everything else — limits
attack the cause, adjustments only re-aim the gun.
</details>

**Q2.** Explain how overcommit mode 2 ("never overcommit") breaks fork for a
4 GiB process on an 8 GiB host, and why CoW doesn't rescue the accounting.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Under mode 2 every mapping must be fully reserved against CommitLimit at
creation. fork clones the address space copy-on-write: physically ~free
(Lesson 07), but <em>in promises</em> the child now holds 4 GiB of writable
mappings that it is entitled to fully dirty — worst case, both processes write
everything and need 8 GiB total. Strict accounting must reserve for that worst
case, so the fork needs 4 GiB of unreserved headroom; with the parent + system
already holding most of the 8 GiB CommitLimit, <code>fork()</code> returns
ENOMEM — famously breaking "big process spawns a tiny helper" (Redis
BGSAVE's documented pain: it forks for snapshotting). CoW is a physical
optimization; mode 2 is a promise ledger — and the ledger must assume promises
get called. Workarounds: <code>posix_spawn</code>/vfork semantics, mode 0/1,
or accepting the reservation waste.
</details>

**Q3.** Compare the kernel OOM killer with PSI-based userspace killers
(systemd-oomd/earlyoom): what does each optimize for, and why do modern
desktops/servers increasingly run both?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Kernel OOM is the <em>correctness</em> backstop: it fires only at true
bankruptcy — an allocation cannot be satisfied after reclaim exhausts — and it
must succeed regardless of system state, so it's built to be last-resort
(SIGKILL, no negotiation). By that moment the box has often spent minutes in
reclaim hell: everything stalled, UI frozen, SSH molasses — technically alive,
practically dead. PSI killers optimize <em>experience/SLOs</em>: they watch
stall time (memory some/full — the "practically dead" metric) and kill chosen
victims (per config: browser tabs, batch cgroups) while the system is still
responsive — trading an earlier, softer casualty for never visiting the coma
state. Run both: oomd as the early, policy-rich layer; kernel OOM as the
guaranteed floor when oomd itself can't run. It's defense in depth over the
same cliff — one rail before the edge, one net below it.
</details>

---

## Homework

Write the postmortem for lab step 3 as if it were production: from the dmesg
report and `memory.events`, reconstruct the timeline (limit, usage at death,
victim, what it held), then propose and implement (commands) the fix for each
of three different "requirements": (a) this workload must never die — others
may; (b) this workload may die but must fail gracefully earlier; (c) this
workload is best-effort — die first, quietly.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Postmortem skeleton: cgroup oomlab, <code>memory.max=200M</code>,
swap.max=0; python's allocation loop dirtied anonymous pages until usage hit
the limit with nothing reclaimable (no file pages to drop, no swap to spill);
kernel memcg-OOM selected the only candidate (python3, anon-rss ≈ limit),
SIGKILL, exit 137; <code>memory.events oom=1 oom_kill=1</code>. Fixes: (a)
<em>never die</em>: give it a dedicated cgroup with
<code>memory.low</code>/<code>min</code> protection sized to its working set +
<code>OOMScoreAdjust=-900</code>, and cap the <em>neighbors</em> with
memory.max — protection through others' limits. (b) <em>fail gracefully
earlier</em>: set <code>memory.high</code> below max (throttles/reclaims the
group first — the app sees pressure as slowdown), subscribe to PSI/memory.high
events (or use an allocator hook) and shed load/flush caches before the wall;
max stays as the backstop. (c) <em>best-effort</em>: <code>memory.max</code>
small, <code>oom_score_adj=+1000</code>, <code>Restart=on-failure</code> in
its unit, and alert on memory.events.oom_kill as a counter, not a page — death
is the design, so make it cheap and observable.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 23 — Hugepages and NUMA →](lesson-23-hugepages-numa){: .btn .btn-primary }
