---
title: "Lesson 20 — The Page Cache"
nav_order: 5
parent: "Phase 4: Memory"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 20: The Page Cache

## Concept

Run `free -h` on any Linux box that's been up for a day and "available" memory
looks alarming until you spot the `buff/cache` column quietly holding most of
your RAM. That's the **page cache**: every file page ever read or written
recently, kept in otherwise-idle RAM on a simple bet — *you'll want it again,
and RAM is 100,000× faster than disk*.

```
        read("file")                          write("file")
             │                                     │
             ▼                                     ▼
   ┌──────────────────┐  hit: copy from    ┌──────────────────┐
   │    PAGE CACHE    │  RAM, ~µs          │    PAGE CACHE    │ write goes to
   │  (file pages in  │                    │   page marked    │ RAM, returns
   │    idle RAM)     │  miss: read disk   │     DIRTY        │ immediately!
   └────────┬─────────┘  into cache first  └────────┬─────────┘
            │ (major fault if mmap'd — L19)          │ later: WRITEBACK
            ▼                                        ▼ flusher threads,
          disk                                     disk  every ~5-30s
```

Two consequences run the world:

1. **Reads**: the first read of anything is slow (disk), every repeat is RAM —
   the "second run is faster" effect you measured in Lesson 19, the reason
   `grep` over a warm tree feels instant.
2. **Writes**: `write()` returning does **not** mean data is on disk — it means
   it's in RAM, marked *dirty*, awaiting writeback. Power loss in that window
   loses "successfully written" data. The only fix is `fsync` — and the full
   durability story is Lesson 41's.

And the accounting insight that stops the classic panic: **cache memory is
free memory with a bonus**. Under pressure the kernel drops clean cache pages
instantly (they're copies; disk has the truth). `free`'s "available" column
already counts that — it's the only number worth alerting on.

---

## How It Works

### One cache to rule them all

The page cache is *the* interface between files and memory — both I/O styles
go through it: `read()/write()` copy between your buffer and cache pages;
`mmap` (Lesson 19) maps the cache pages themselves into your address space.
Same physical frames — which is why a file written via write() is instantly
visible through a neighbor's mmap: there's only one copy. Executables and
libraries live here too: "loading a program" = faulting its file pages into
the page cache (Lesson 19's homework), shared by every process running it —
the page cache *is* Lesson 16's shared-frame story for files.

### Dirty pages and writeback

Written pages are flagged dirty (the hardware PTE bit from Lesson 17 plus
cache-level flags) and flushed by kernel *flusher* threads: after age
(`vm.dirty_expire_centisecs`, ~30 s), or when dirty volume crosses
`vm.dirty_background_ratio` (~10% of RAM — async flush starts) and at
`vm.dirty_ratio` (~20% — *writers* are throttled to let disk catch up: the
mysterious stalls in big-copy workloads). `sync(1)` forces everything out;
`/proc/meminfo`'s `Dirty:` line is the live backlog.

### Readahead

Sequential reads trigger prefetch: the kernel notices the pattern and reads
ahead (128 KiB+ windows, adaptive), so streaming a file keeps the disk busy
*ahead* of the reader — a huge part of why sequential I/O so outruns random
(`posix_fadvise` lets programs hint; Lesson 42's block layer executes).

{: .note }
> **Watching the cache decide**
> Per-file cache residency: <code>fincore</code> (util-linux) or
> <code>mincore(2)</code>. System counters: <code>grep -E
> 'Cached|Dirty|Writeback' /proc/meminfo</code>. Cache hit behavior:
> the timing difference is usually proof enough — and
> <code>echo 3 &gt; /proc/sys/vm/drop_caches</code> (labs only! production
> never!) gives you a cold start on demand.

---

## Lab

```bash
# ---- 1. The headline numbers, decoded ----
$ free -h
#                total   used   free   shared  buff/cache  available
# Mem:            4.0G   600M   200M      20M        3.2G       3.1G
#   "free" is tiny and IRRELEVANT; "available" (free + droppable cache) is truth

# ---- 2. Read caching: 50-100x, measured ----
$ dd if=/dev/urandom of=/tmp/big bs=1M count=500 2>/dev/null && sync
$ echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null      # cold
$ time cat /tmp/big > /dev/null
# real ~0.5-3s          ← disk speed
$ time cat /tmp/big > /dev/null
# real ~0.05s           ← RAM speed. The bet, won.

# ---- 3. Watch the cache grow by exactly your file ----
$ echo 3 | sudo tee /proc/sys/vm/drop_caches > /dev/null
$ grep ^Cached /proc/meminfo
$ cat /tmp/big > /dev/null
$ grep ^Cached /proc/meminfo
# Cached grew ~500MB — your file, resident

# ---- 4. Dirty pages: write() lies (nobly) ----
$ grep -E '^(Dirty|Writeback):' /proc/meminfo
$ dd if=/dev/zero of=/tmp/dirty bs=1M count=300 2>/dev/null    # NO sync!
$ grep -E '^(Dirty|Writeback):' /proc/meminfo
# Dirty: ~300000 kB     ← "written" data living only in RAM right now
$ time sync                                                    # force it out
$ grep ^Dirty /proc/meminfo
# Dirty: ~0 — and sync's duration was your real disk write happening

# ---- 5. write() vs fsync latency: the durability price tag ----
$ python3 - << 'EOF'
import os, time
f = os.open('/tmp/sync_test', os.O_WRONLY | os.O_CREAT | os.O_TRUNC)
t = time.time()
for i in range(200): os.write(f, b'x' * 4096)
print(f"200 writes:           {(time.time()-t)*1000:6.1f} ms  (RAM)")
t = time.time()
for i in range(200):
    os.write(f, b'x' * 4096); os.fsync(f)
print(f"200 writes + fsync:   {(time.time()-t)*1000:6.1f} ms  (DISK, each)")
os.close(f)
EOF
# 10-1000x apart. Databases pay the right column. (Full story: Lesson 41.)

# ---- 6. Cache is instantly reclaimable — pressure test ----
$ grep -E '^(Cached|MemAvailable)' /proc/meminfo
$ python3 -c "b = bytearray(1_500_000_000); b[::4096] = b'x'*(len(b)//4096); input()" &
$ sleep 3; grep -E '^(Cached|MemAvailable)' /proc/meminfo
# Cached shrank to feed your allocation — no OOM, no swap needed: clean pages
# just evaporated (disk still has them). kill %1 when done.
$ kill %1; rm /tmp/big /tmp/dirty /tmp/sync_test
```

---

## Further Reading

| Topic | Link |
|---|---|
| Page cache | <https://en.wikipedia.org/wiki/Page_cache> |
| `free(1)` man page | <https://man7.org/linux/man-pages/man1/free.1.html> |
| `sync(2)` / `fsync(2)` | <https://man7.org/linux/man-pages/man2/fsync.2.html> |
| Dirty-page sysctls — kernel docs | <https://www.kernel.org/doc/html/latest/admin-guide/sysctl/vm.html> |
| `posix_fadvise(2)` man page | <https://man7.org/linux/man-pages/man2/posix_fadvise.2.html> |
| `mincore(2)` man page | <https://man7.org/linux/man-pages/man2/mincore.2.html> |

---

## Checkpoint

**Q1.** Reading a 1 GiB file twice: the second read is 50× faster. Where did
the data come from, and name three events that could make the third read slow
again.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
From the page cache — the first read populated RAM frames with the file's
pages; the second read copied from those frames without touching disk. Slow
again if the pages get evicted: (1) <strong>memory pressure</strong> — another
workload allocated enough that reclaim dropped these clean pages (lab step 6);
(2) <strong>competing I/O</strong> — reading other large files pushed this one
out of the LRU; (3) <strong>invalidation</strong> — the file was rewritten
(cache updated, but a changed file on a network filesystem or a
drop_caches/remount/reboot empties residency). Cache warmth is a probabilistic
asset — never a guarantee — which is why cold-start behavior deserves its own
testing.
</details>

**Q2.** `write()` returned success and the machine lost power one second
later. Is the data safe? List every layer that could still be holding it, and
the API that closes each gap. (Preview of Lesson 41 — reason from this lesson.)

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Almost certainly not safe. Holding it: (1) the <strong>page cache</strong> —
dirty pages await writeback for up to ~30 s; closed by
<code>fsync(fd)</code>/<code>fdatasync</code> (or O_SYNC per-write). (2) The
<strong>drive's volatile write cache</strong> — the disk acknowledged receipt
into its RAM buffer; closed by the kernel sending a cache-flush/FUA with fsync
(on by default; battery-backed controllers legitimately absorb it). (3)
Subtler: the file's <strong>metadata & directory entry</strong> — a brand-new
file also needs its directory durable: fsync the containing directory too.
(And libc userspace buffering — Lesson 02 — sits before all of this for stdio
users: fflush first.) "Success" from write() means one thing only: the kernel
has a copy in RAM. Durability is a separate, explicit, expensive request —
which is the entire reason databases are hard.
</details>

**Q3.** During a large `cp`, unrelated processes making small writes suddenly
stall for hundreds of ms. Connect this to the dirty-page machinery and name
the knobs that shape the behavior.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The cp generated dirty pages faster than the disk drains them; when system
dirty volume crossed <code>vm.dirty_background_ratio</code>, async flushers
started, and at <code>vm.dirty_ratio</code> the kernel began
<strong>throttling writers</strong> — any process dirtying pages (including
innocent small writers) gets paused inside write() until writeback makes
space: your stalls. Balanced-dirty-throttling scales the pause to each writer's
rate, but under a heavy backlog everyone waits. Knobs:
lower <code>dirty_background_*</code> to start flushing earlier (smoother, more
steady-state I/O), tune <code>dirty_expire_centisecs</code> for age-based
flushing, or make the bulk copy bypass/limit cache usage
(<code>dd oflag=direct</code>, rsync bandwidth limits, ionice + cgroup io.max —
Lessons 42/60). Diagnosis fingerprint: stalls correlate with
<code>Dirty:</code> near the ratio thresholds and PSI <code>io</code> pressure
rising (Lesson 15's tools).
</details>

---

## Homework

The page cache serves *shared* executables (every process running `python3`
shares its cached code pages). Design and run the measurement: how much faster
does `python3 -c pass` start warm vs cold? Use `time` for wall clock and
`ps`-style fault counts (Lesson 19's homework technique) to attribute the gap.
Then answer: on a fleet where hosts reboot for kernel updates, what *operational
pattern* does this create in the minutes after reboot, and what mitigations
exist?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Typical: cold ~300–800 ms with dozens–hundreds of major faults (python binary
+ libpython + stdlib .pyc files all read from disk); warm ~30–80 ms, ~0 majors
— a 5–10× gap owned entirely by the page cache. Fleet pattern after reboot:
the <em>thundering cold start</em> — every service starts with an empty cache,
so all binaries, libraries, config files, and data files major-fault
simultaneously: disk saturates, startups that normally take seconds take
minutes, health checks time out, orchestrators restart "slow" services and
make it worse. Mitigations: stagger service starts; pre-warm deliberately
(<code>cat</code>/vmtouch critical files in early boot, readahead profiles —
systemd's old readahead, bootchart-guided prefetch); keep hot data on NVMe so
cold reads are cheap; for VMs, restore from a snapshot with warm cache rather
than cold boot (virt track's migration lessons); and accept it in SLOs — the
first N minutes after boot are a different performance regime.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 21 — Swap and Memory Reclaim →](lesson-21-swap-reclaim){: .btn .btn-primary }
