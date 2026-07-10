---
title: "Lesson 21 — Swap and Memory Reclaim"
nav_order: 6
parent: "Phase 4: Memory"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 21: Swap and Memory Reclaim

## Concept

Lesson 20 ended with cache pages "evaporating" under pressure. This lesson is
about the machinery that chose them — **memory reclaim** — and what happens
when dropping clean cache isn't enough.

The kernel's problem: RAM is full (by design! — idle RAM is wasted RAM), and
someone needs a frame *now*. Every frame in use is one of two kinds, and they
have very different exit costs:

```
                     RAM is full. Need a frame. Evict what?

  FILE-BACKED pages (page cache)          ANONYMOUS pages (heaps, stacks)
  ──────────────────────────────          ──────────────────────────────
  clean: disk already has it       ◀───   no file behind them: nowhere
   → just drop it. FREE!          easy    to put them down…
  dirty: write back first,               → unless SWAP exists: a disk
   then drop. One write.                   area that acts as their "file"
                                           → write page out, mark PTE
                                             not-present (L17!), reuse frame
                                           faulting it back = MAJOR fault (L19)
```

**Swap is not the emergency overflow tank** — that's the great misconception.
Swap is what makes *anonymous* memory evictable at all. Without it, every idle
daemon's startup-only pages, that leaked-but-never-touched heap, are **pinned
in RAM forever**, while your hot page cache — actually earning its keep — gets
evicted around them. A little swap traffic on a healthy box is the kernel
tidying: cold anonymous pages out, useful cache in.

The pathology is different: **thrashing** — when the *working set* (pages
needed continuously) exceeds RAM, and the kernel evicts pages that are
immediately needed back. Majors flood, disk saturates, every process spends
its time waiting on faults. The distinction between tidying and thrashing is
*rate and re-fault*, and PSI memory pressure (Lesson 15) finally measures it
directly.

---

## How It Works

### The LRU machinery

Reclaim approximates "evict the least recently used": pages sit on
active/inactive lists (per type, per cgroup, per NUMA node); the accessed bit
(Lesson 17 — hardware sets it, kernel clears and re-checks) demotes untouched
pages toward the inactive tail, where reclaim harvests. Kernel 6.1+'s
**MGLRU** (multi-generational LRU) refines this into age generations —
better cold-page detection, same idea. Reclaim runs two ways: **kswapd**
wakes in the background as free memory dips below watermarks (keeping a float
of free frames); if allocation outruns it, the allocating process itself does
**direct reclaim** — allocation latency spikes are often exactly this.

### swappiness

`vm.swappiness` (0–200, default 60) biases *what* to reclaim: low = prefer
dropping file cache, high = prefer swapping anonymous pages. `0` ≈ "swap only
to avoid OOM" — common on databases (their cache IS the workload); `100+`
makes sense with fast compressed swap. It's a preference, not a switch:
pressure can override any setting.

### Modern swap: zram and zswap

Swapping to spinning disk earned swap its bad name. Today:
**zram** — a compressed RAM disk as swap device: "swapped" pages are
compressed in RAM (~3:1), reclaim without disk I/O at all (default on Fedora,
Android, ChromeOS); **zswap** — a compressed RAM *cache in front of* disk swap,
absorbing the traffic and spilling only overflow. Both turn the major-fault
cliff into a ~µs decompression — changing the calculus of how much swapping is
acceptable.

{: .note }
> **Reading swap health**
> <code>vmstat 1</code>: <code>si/so</code> (swap-in/out per second) —
> sustained <em>si</em> is the thrash signal (pages coming back = they were
> needed). <code>free -h</code>: swap <em>used</em> is fine; swap
> <em>churning</em> is not. Per-process: <code>grep VmSwap
> /proc/PID/status</code>. The verdict metric: <code>/proc/pressure/memory</code>
> — some/full stall time is the cost, directly.

---

## Lab

```bash
# ---- 0. What swap do you have? ----
$ swapon --show; free -h | tail -1
# NAME      TYPE SIZE USED  (maybe /swap.img, maybe zram0, maybe nothing)
# no swap? make a lab file (remove after):
#   sudo fallocate -l 2G /swaplab && sudo chmod 600 /swaplab
#   sudo mkswap /swaplab && sudo swapon /swaplab

# ---- 1. Watch reclaim choose: cache first, then swap ----
# terminal 2:  vmstat 1    (watch: free, cache, si, so)
$ dd if=/dev/urandom of=/tmp/cachefill bs=1M count=800 2>/dev/null && cat /tmp/cachefill >/dev/null
# cache is now fat. Allocate hard:
$ python3 -c "
b = bytearray(int(2.5e9)); b[::4096] = b'x'*(len(b)//4096); input('hold...')" &
$ sleep 5; free -h; grep -E 'pgsteal_direct|pgscan' /proc/vmstat | head -4
# watch in vmstat: cache SHRANK first (cheap evictions); only then so>0 (swap)

# ---- 2. Who got swapped? ----
$ for p in $(ls /proc | grep -E '^[0-9]+$' | head -80); do
    s=$(grep -s VmSwap /proc/$p/status | awk '{print $2}')
    [ -n "$s" ] && [ "$s" -gt 1000 ] && echo "$s kB  $(cat /proc/$p/comm)"
  done | sort -rn | head
# idle daemons' cold pages — exactly the right victims. Your hot python: little.
$ kill %1

# ---- 3. Feel a major-fault storm (mini-thrash, contained) ----
$ python3 - << 'EOF'
import resource, time
big = bytearray(int(1.5e9))
big[::4096] = b'x' * (len(big)//4096)          # materialize 1.5GB
# now allocate MORE to force some of 'big' out, then walk big again:
squeeze = bytearray(int(1.5e9)); squeeze[::4096] = b'y' * (len(squeeze)//4096)
r0 = resource.getrusage(resource.RUSAGE_SELF).ru_majflt; t = time.time()
s = 0
for i in range(0, len(big), 4096): s += big[i]  # every page may be a MAJOR fault
r1 = resource.getrusage(resource.RUSAGE_SELF).ru_majflt
print(f"walk took {time.time()-t:.1f}s with {r1-r0} major faults")
EOF
# hundreds-thousands of majors, seconds of stall — thrashing in miniature
# (numbers vary with your VM's RAM; shrink/grow the arrays to fit)

# ---- 4. The verdict metrics ----
$ cat /proc/pressure/memory
# some avg10=... ← nonzero after step 3: tasks stalled waiting for memory
$ grep -E '^(pswpin|pswpout|pgmajfault)' /proc/vmstat

# ---- 5. swappiness experiment ----
$ cat /proc/sys/vm/swappiness
$ sudo sysctl vm.swappiness=10     # cache-hostile … rerun step 1 and compare
$ sudo sysctl vm.swappiness=60     # restore. (persistence: networking L35!)
$ rm /tmp/cachefill                # (and swapoff/rm /swaplab if you made one)
```

---

## Further Reading

| Topic | Link |
|---|---|
| Paging / swap | <https://en.wikipedia.org/wiki/Memory_paging> |
| Thrashing | <https://en.wikipedia.org/wiki/Thrashing_(computer_science)> |
| zram | <https://en.wikipedia.org/wiki/Zram> |
| vm sysctls (swappiness etc.) | <https://www.kernel.org/doc/html/latest/admin-guide/sysctl/vm.html> |
| MGLRU — kernel docs | <https://www.kernel.org/doc/html/latest/admin-guide/mm/multigen_lru.html> |
| `swapon(8)` man page | <https://man7.org/linux/man-pages/man8/swapon.8.html> |

---

## Checkpoint

**Q1.** Is "the box is swapping" always a problem? Which numbers distinguish
healthy tidying from thrashing, and what makes the difference conceptually?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Not always — often the opposite. Healthy: swap <em>used</em> is substantial
but stable, <code>so</code> occasional, <code>si</code> ≈ 0, PSI memory ~0 —
the kernel moved genuinely cold anonymous pages (idle daemons' startup code,
untouched heap) to swap and reused the RAM for page cache that's earning hits.
Thrashing: sustained <code>si</code> <em>and</em> <code>so</code> together,
pgmajfault climbing, PSI memory some/full rising, latency visible — pages are
being evicted <em>and immediately demanded back</em>. Conceptually: tidying
evicts outside the working set; thrashing means the working set itself exceeds
RAM, so every eviction is a future major fault. Swap usage is a stock; swap
<em>traffic with re-faults</em> is the flow that hurts.
</details>

**Q2.** Argue the case for running a database server *with* some swap (and
swappiness low), against the common "disable swap on databases" reflex —
including what happens under pressure in each configuration.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With no swap, anonymous memory is unevictable: under pressure the kernel's
only levers are dropping page cache (the database's file cache — its
performance!) and, when that's gone, the OOM killer — likely killing the
database itself (biggest RSS, Lesson 22). A modest swap + swappiness 1–10
changes the failure mode: truly cold anonymous pages (connection buffers from
idle sessions, allocator waste, startup-only code paths) can leave RAM, the
hot cache survives, and pressure degrades performance gradually instead of
executing your process. The reflex's real target is thrashing the DB's hot
memory — addressed by low swappiness and, properly, by mlock for critical
regions and cgroup memory.low protection (Lesson 60), not by removing the
kernel's safety valve. (Kubernetes historically demanded swap-off for
accounting reasons — an orchestration constraint, not a performance argument,
and one that's being revisited.)
</details>

**Q3.** Explain how zram changes the thrashing math: a workload whose working
set is 130% of RAM thrashes horribly on disk swap yet may run acceptably with
zram. Where does the win come from, and what workload property could ruin it?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The cliff shrinks by ~3 orders of magnitude: a disk-swap major fault costs
ms (queueing + I/O), while a zram "fault" is a decompression in RAM — a few
µs. At 130% working set, some fraction of accesses continuously hit swapped
pages; at ms each, the machine spends nearly all time waiting (thrash); at µs
each it's a modest slowdown. The win requires pages to <em>compress</em>:
zram effectively buys you ~RAM×(compression ratio − its overhead) of extra
capacity — text, JSON, zeroed and sparse heaps compress 3–4:1. Ruined by
incompressible data: media buffers, encrypted blobs, already-compressed
caches — ratio →1:1, zram becomes pure overhead (CPU spent compressing plus
RAM still full), and you're back to real thrashing, now with hot CPUs.
Check <code>/sys/block/zram0/mm_stat</code> for the achieved ratio before
trusting it.
</details>

---

## Homework

Build a "memory weather report" script: from `/proc/meminfo`, `/proc/vmstat`
(pswpin/pswpout/pgmajfault deltas over 10 s), `swapon --show`, and
`/proc/pressure/memory`, print: total/available, cache size, swap used, the
three 10-second rates, PSI avg10 — and a one-line verdict:
`COMFORTABLE / TIDYING / PRESSURED / THRASHING`. Define your thresholds and
justify each boundary with this lesson's concepts. Run it during lab steps 1
and 3 and confirm the verdicts move.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<pre>
#!/bin/bash
snap() { awk '/^(pswpin|pswpout|pgmajfault)/{print $2}' /proc/vmstat; }
a=($(snap)); sleep 10; b=($(snap))
si=$(( (${b[0]}-${a[0]}) / 10 )); so=$(( (${b[1]}-${a[1]}) / 10 ))
mf=$(( (${b[2]}-${a[2]}) / 10 ))
avail=$(awk '/MemAvailable/{print $2}' /proc/meminfo)
total=$(awk '/MemTotal/{print $2}' /proc/meminfo)
psi=$(awk '/^some/{gsub("avg10=","",$2); print $2}' /proc/pressure/memory)
echo "avail ${avail}kB/${total}kB  si=$si/s so=$so/s majflt=$mf/s  psi10=$psi"
v=COMFORTABLE
[ "$so" -gt 10 ] && v=TIDYING
awk -v p="$psi" 'BEGIN{exit !(p>5)}' && v=PRESSURED
{ [ "$si" -gt 100 ] && [ "$mf" -gt 100 ]; } && v=THRASHING
echo "verdict: $v"
</pre>
Boundaries: <em>so&gt;10/s alone</em> = TIDYING — outbound-only swap is the
kernel parking cold pages (no re-faults, no cost). <em>PSI some&gt;5%</em> =
PRESSURED — tasks measurably stalled on memory: reclaim is in the latency
path even if swap traffic looks small (direct reclaim!). <em>si&gt;100/s AND
majfault&gt;100/s</em> = THRASHING — the re-fault loop is closed: evicted
pages are being demanded back continuously; each is a disk-latency stall.
During lab 1 the script reads TIDYING (cache dropped, some so); during lab 3
it flips to PRESSURED/THRASHING as si and majflt spike together — the verdicts
track the conceptual line between eviction outside vs inside the working set.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 22 — Overcommit and the OOM Killer →](lesson-22-oom-overcommit){: .btn .btn-primary }
