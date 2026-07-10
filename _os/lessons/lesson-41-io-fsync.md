---
title: "Lesson 41 — Buffered vs Direct I/O, and fsync"
nav_order: 6
parent: "Phase 7: Files & Filesystems"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 41: Buffered vs Direct I/O, and fsync

## Concept

This is the durability lesson — the one that answers, precisely and finally,
the question Lesson 20 opened: **when is your data actually on disk?**

Follow one `write()` all the way down:

```
  app buffer ──write()──▶ PAGE CACHE ──writeback──▶ DISK WRITE ──flush──▶ PLATTER/
  (maybe stdio            (dirty page,  (flusher     CACHE       (FUA/     FLASH
   buffer before          "success"!)    threads,    (device's   flush     (durable
   it — L02!)                            ~5-30s)     RAM!)       cmd)      at last)

  write() returning = your data reached layer 2 of 5.
  power loss eats layers 2, 3, AND 4. Only the last one survives.
```

The tools that force the journey to complete:

- **`fsync(fd)`** — block until this file's dirty data *and* metadata are
  durable (including the device-cache flush). The real thing.
- **`fdatasync(fd)`** — same, minus non-essential metadata (skip an mtime
  journal commit — cheaper, same data guarantee).
- **`O_SYNC`** — every write behaves as write+fdatasync (no batching choice
  left).
- **`sync()` / `syncfs()`** — everything system/fs-wide (the shutdown
  hammer).
- **fsync the *directory*** — a new file's *name* lives in the directory
  (Lesson 37!); rename-based commits need the directory durable too.

And the second axis — **who caches?**: `O_DIRECT` bypasses the page cache
entirely: DMA straight between *your* buffer and the device. Not "faster" —
it *removes services* (caching, readahead, write batching) for applications
that provide their own: databases with buffer pools (Postgres being the
famous holdout that stays buffered; the debate is instructive), and QEMU
with `cache=none` — virt Lesson 25's option, now fully decoded: that lesson's
whole cache-mode matrix is this lesson's layers applied to a VM's virtual
disk.

---

## How It Works

### What fsync really costs

An fsync pays for: pending data writeback (however much you dirtied), a
journal commit (Lesson 38 — the metadata transaction), and a **device cache
flush** — telling the disk "empty your RAM onto media, now." On a SATA SSD
that flush is the dominant cost: ~0.5–5 ms; on NVMe with power-loss
protection, ~50 µs (the device's cache is battery/capacitor-safe, so flush is
nearly free — this hardware property, not the interface, is the real price
divider). Hence transactional systems' arithmetic: max transactions/s per
writer ≈ 1/fsync-latency — and their tricks: **group commit** (many
transactions, one fsync — amortize the flush) and separating the WAL onto
the lowest-fsync-latency device you own.

### The crash-consistency contract, assembled

Everything from three lessons composes into two canonical recipes:

**Atomic replace** (config files, Lesson 37): write tmp → `fsync(tmp)` →
`rename(tmp, real)` → `fsync(dir)`. Guarantees: readers always see a
complete old or complete new file, even across power loss.

**Append log / WAL** (databases, Lesson 38): append record → `fdatasync` →
*then* acknowledge/act. Guarantees: every acknowledged record survives;
recovery truncates any torn tail (records carry checksums for exactly that).

And the ugly footnote every serious engineer eventually learns —
**fsyncgate**: if fsync fails (EIO — a dying disk), the kernel historically
*cleared* the dirty flags anyway; retrying fsync could return success while
data was lost. Postgres's response (2018): treat any fsync failure as fatal
— crash and recover from WAL, never retry-and-hope. The durable lesson:
**fsync errors are data loss already in progress, not a retryable hiccup.**

### O_DIRECT's fine print

Requires aligned I/O (buffer address, offset, length — typically 512 B/4 KB
multiples: `posix_memalign`); no readahead, no write-behind — your thread
waits for the device unless you bring your own async engine (io_uring,
Lesson 44 — the pairing O_DIRECT was waiting for); and mixing direct +
buffered access to one file invites stale-cache incoherence. It's a
professional's tool with a professional's manual.

{: .note }
> **stdio's extra layer**
> <code>printf/fwrite</code> buffer in userspace (Lesson 02/03) —
> <code>fflush()</code> moves data to the kernel (layer 2), and only
> <code>fsync(fileno(f))</code> makes it durable. The full incantation for
> stdio users is fflush-then-fsync; forgetting the first makes the second
> sync nothing.

---

## Lab

```bash
# ---- 1. The five layers, made visible with strace ----
$ strace -e trace=write,fsync,fdatasync,openat python3 -c "
f = open('/tmp/d1', 'w'); f.write('data'); f.flush()
import os; os.fsync(f.fileno()); f.close()" 2>&1 | grep -E 'write|fsync|/tmp'
# openat(... "/tmp/d1" ...) — write(3, "data", 4) — fsync(3)
# without .flush(): NO write syscall until close (stdio buffering, caught!)

# ---- 2. The price of durability, measured honestly ----
$ python3 - << 'EOF'
import os, time
def bench(label, fsync_each, n=300):
    fd = os.open("/tmp/bench", os.O_WRONLY | os.O_CREAT | os.O_TRUNC)
    t = time.time()
    for i in range(n):
        os.write(fd, b"x" * 4096)
        if fsync_each: os.fdatasync(fd)
    if not fsync_each: os.fdatasync(fd)          # one sync at the end
    dt = time.time() - t
    os.close(fd)
    print(f"{label}: {dt*1000:7.1f} ms  ({dt/n*1e6:6.0f} µs/write)")
bench("buffered + 1 final sync", False)
bench("fdatasync every write   ", True)
EOF
# typical VM: ~2ms total vs ~1500ms — 3 orders of magnitude. THAT is why
# group commit exists, and why "just fsync everything" is not a strategy.

# ---- 3. fsync vs fdatasync: metadata's share of the bill ----
$ python3 - << 'EOF'
import os, time
for name, fn in [("fsync    ", os.fsync), ("fdatasync", os.fdatasync)]:
    fd = os.open("/tmp/meta", os.O_WRONLY | os.O_CREAT | os.O_TRUNC)
    t = time.time()
    for i in range(200):
        os.write(fd, b"y" * 100)                 # size changes → metadata!
        fn(fd)
    print(f"{name}: {(time.time()-t)*1000:.0f} ms")
    os.close(fd)
EOF
# fdatasync usually wins measurably when metadata (size/mtime) churns —
# it may skip the journal commit for timestamps. (Appends that grow the
# file still force essential metadata — size — through.)

# ---- 4. The atomic-replace recipe, complete and correct ----
$ python3 - << 'EOF'
import os
def atomic_write(path, data):
    d = os.path.dirname(path) or "."
    tmp = path + ".tmp"
    fd = os.open(tmp, os.O_WRONLY | os.O_CREAT | os.O_TRUNC)
    os.write(fd, data)
    os.fsync(fd)                                  # 1. data durable
    os.close(fd)
    os.rename(tmp, path)                          # 2. atomic pointer flip
    dfd = os.open(d, os.O_RDONLY)                 # 3. the NAME durable —
    os.fsync(dfd)                                 #    fsync the DIRECTORY
    os.close(dfd)
atomic_write("/tmp/conf", b"version 2\n")
print(open("/tmp/conf").read().strip(), "— crash-safe at every instant")
EOF

# ---- 5. O_DIRECT: bypassing the cache, feeling the alignment rules ----
$ python3 - << 'EOF'
import os, ctypes, time
# aligned buffer via mmap (page-aligned by construction)
import mmap
buf = mmap.mmap(-1, 4096)
buf[:] = b"D" * 4096
fd = os.open("/tmp/direct", os.O_WRONLY | os.O_CREAT | os.O_DIRECT, 0o644)
t = time.time()
for i in range(100):
    os.write(fd, buf)                             # straight to device, no cache
print(f"O_DIRECT 100x4K: {(time.time()-t)*1000:.0f} ms (each waits for the DEVICE)")
os.close(fd)
# misaligned attempt → EINVAL, the famous O_DIRECT greeting:
fd = os.open("/tmp/direct", os.O_WRONLY | os.O_DIRECT)
try: os.write(fd, b"tiny")
except OSError as e: print("misaligned write:", e)
EOF

# ---- 6. Watch dirty pages drain (L20's counters, closing the loop) ----
$ dd if=/dev/zero of=/tmp/dirtyload bs=1M count=200 2>/dev/null
$ grep -E '^(Dirty|Writeback):' /proc/meminfo      # backlog exists
$ time sync                                         # the hammer — and its real cost
$ grep ^Dirty /proc/meminfo
$ rm -f /tmp/d1 /tmp/bench /tmp/meta /tmp/conf* /tmp/direct /tmp/dirtyload
```

---

## Further Reading

| Topic | Link |
|---|---|
| `fsync(2)` man page | <https://man7.org/linux/man-pages/man2/fsync.2.html> |
| `open(2)` — O_DIRECT, O_SYNC | <https://man7.org/linux/man-pages/man2/open.2.html> |
| `sync(2)` / `syncfs(2)` | <https://man7.org/linux/man-pages/man2/sync.2.html> |
| Write barrier / disk cache | <https://en.wikipedia.org/wiki/Disk_buffer> |
| PostgreSQL fsync issue ("fsyncgate") — LWN | <https://lwn.net/Articles/752063/> |
| Write-ahead logging | <https://en.wikipedia.org/wiki/Write-ahead_logging> |

---

## Checkpoint

**Q1.** `write()` returned success and the machine lost power. List every
layer that could still be holding the data (there are four), and the API/
configuration that closes each gap.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <strong>stdio/userspace buffer</strong> — printf/fwrite data that never
even became a syscall (Lesson 02); closed by <code>fflush()</code> (or
unbuffered/line-buffered modes). (2) <strong>page cache</strong> — the dirty
page write() created, awaiting writeback for up to ~30 s (Lesson 20); closed
by <code>fsync/fdatasync</code> (or O_SYNC per write). (3) <strong>device
write cache</strong> — the disk acknowledged into its onboard RAM; closed by
the flush/FUA the kernel's fsync issues — <em>if</em> the device honors it
(consumer drives mostly do; the config answer is power-loss-protected
[enterprise] drives or battery-backed controllers where flush is safely a
no-op). (4) <strong>the namespace</strong> — for a new/renamed file, the
directory entry itself may not be durable: the data survives but no name
points to it; closed by fsync on the <em>directory</em> fd. Only after all
four is "success" true across power loss — which is why the atomic-replace
recipe has exactly those steps.
</details>

**Q2.** A database does 400 small transactions/s and each fsync costs 2.5 ms
— the math says 400/s is the hard ceiling for one writer. Explain group
commit: how does it break that ceiling, what does it trade, and why does the
same idea appear in the kernel's own journal?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The ceiling comes from serializing one flush per transaction: 2.5 ms × 400 =
100% of a second. Group commit decouples transactions from flushes: incoming
transactions append their WAL records and <em>wait</em>; one fsync covers
everything appended since the last flush; all waiters are then acknowledged
together. Throughput becomes (records per flush) × (flushes/s) — hundreds of
transactions can share one 2.5 ms flush, lifting the ceiling 10–100× under
concurrency. The trade: <em>latency for throughput</em> — an individual
transaction now waits for the group window (often bounded:
"flush every N ms or M records"); a lone transaction on an idle system
gains nothing. The kernel's jbd2 (Lesson 38) does the identical trick:
multiple syscalls' metadata changes batch into one journal transaction with
one commit block — and even the device is playing the same game (write cache
batching flushes). It's the universal answer to per-operation fixed costs:
amortize across a batch — the same economics as syscall batching
(Lesson 02's dd experiment) and io_uring (Lesson 44).
</details>

**Q3.** QEMU's `cache=none` (virt Lesson 25) opens the disk image with
O_DIRECT. Using this lesson, explain what problem that solves for a VM host,
and why `cache=writeback` + a guest that fsyncs correctly is *also* a
defensible configuration.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Problem solved: <strong>double caching</strong>. The guest kernel already
runs a full page cache (Lesson 20) for its applications; if the host also
caches the image file, every guest disk block sits in RAM twice (guest cache
+ host cache) — wasting the host's memory that could serve other VMs — and
the host's cache adds a second, invisible layer of "acknowledged but not
durable" that the guest can't reason about. O_DIRECT hands the guest's
writes straight to the device: one cache (the guest's — closest to the
application, best-informed), honest durability semantics (the guest's fsync
→ virtio flush → device flush, no host-RAM stopover). Why writeback is still
defensible: correctness is preserved <em>if</em> the stack honors flushes —
guest fsync triggers a virtio FLUSH which QEMU translates to fsync on the
image — so an fsync-disciplined guest loses nothing it was promised; and the
host cache genuinely helps (shared read caching across VMs booting the same
base image, write batching for fsync-light workloads). The choice is Lesson
38's homework generalized: who should own the cache and the durability
contract — the layer with the information (guest/database), or the layer
with the shared view (host/kernel)? Dedicated I/O-heavy VMs: cache=none +
io_uring/native AIO; dense general hosting: writeback earns its RAM.
</details>

---

## Homework

SQLite's documentation describes its commit as: write rollback-journal →
fsync journal → write database pages → fsync database → delete journal →
(optionally) fsync directory. Walk each step and answer: what crash-window
does each fsync close, why must the journal be synced *before* touching the
database, and how does WAL mode (append log + checkpoint) change the fsync
count per transaction — connecting it to lab 2's measurement.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Rollback-journal commit: the journal holds the <em>old</em> page images.
fsync #1 (journal, before any database write) guarantees: if we crash while
scribbling on the database, a complete undo record exists — recovery rolls
back to the pre-transaction state. Without that ordering, a crash could find
the database half-modified AND the journal half-written: no consistent state
recoverable — the ordering IS the atomicity (Lesson 38's protocol, verbatim:
log first, commit record, then home locations). fsync #2 (database) closes
the window where the journal gets deleted while database pages still float
in cache — deletion is the "transaction complete" marker, so everything it
implies must be durable first. Directory fsync closes the deleted-journal
name's durability (a resurrected journal after crash would trigger a bogus
rollback of a committed transaction!). Net: ≥2 fsyncs per transaction, both
on the critical path — at lab 2's ~5 ms/fsync, a ~100 tx/s ceiling. WAL
mode: transactions <em>append</em> to a write-ahead log with ONE fsync (the
database files aren't touched at commit at all); a periodic
<em>checkpoint</em> batch-applies the WAL to the database with its own
fsyncs, amortized over hundreds of transactions (group commit's cousin —
Q2). One fsync instead of two, sequential appends instead of scattered page
writes, readers proceed during commits: the 2-5× write speedup SQLite
documents is exactly this arithmetic — and it's the same reason every modern
database is WAL-shaped.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 42 — The Block Layer and I/O Schedulers →](lesson-42-block-layer){: .btn .btn-primary }
