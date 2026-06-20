---
title: "Lesson 25 — Caching Modes and AIO"
nav_order: 25
parent: "Phase 6: Storage"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 25: Caching Modes and AIO

## Concept

Between the guest's disk writes and the physical platter sit several caches. The
QEMU **cache mode** decides whether to use the *host's page cache* and how to honor
the guest's flush requests. Get it wrong and you either lose data on a host crash or
leave performance on the table.

```
   guest write
       │
       ▼
   [ QEMU ]
       │   ── does it go through the HOST PAGE CACHE? ──
       ▼
   host page cache  (RAM)   ←── 'writeback' uses it; 'none' bypasses (O_DIRECT)
       │
       ▼
   physical disk  (the only place data SURVIVES a host power loss)
```

The key question for safety: **on a host power failure, is the guest's "completed"
write actually on the disk, or still in volatile host RAM?**

---

## How It Works

### The cache modes

| `cache=` | Host page cache | Honors guest flush? | Safety on host crash | Use |
|---|---|---|---|---|
| **writeback** | yes (write-back) | yes | Safe *if guest flushes*; data may sit in host cache between flushes | Fast general default for many setups |
| **none** | bypassed (O_DIRECT) | yes | Safe; writes go past host cache, guest flush hits disk | Common **production default** |
| **writethrough** | yes (read cache) | every write flushed | Very safe, slow (each write to disk) | Correctness over speed |
| **directsync** | bypassed (O_DIRECT) | every write synced | Safest, slowest | Strict durability |
| **unsafe** | yes | **ignores** flushes | **Data loss on crash** | Throwaway/transient only (fast installs) |

The two axes: (1) does it use the host page cache, and (2) does it respect the
guest's flush/FUA requests. `unsafe` is the dangerous one — it *drops* the guest's
flushes, so the guest believes data is durable when it isn't.

### Host page cache vs O_DIRECT

- Using the **host page cache** (writeback/writethrough) means data is cached in host
  RAM, which can speed reads (double-caching with the guest) but adds a volatile layer
  and uses host memory.
- **O_DIRECT** (`cache=none`/`directsync`) **bypasses** the host page cache — I/O goes
  more directly to storage. This avoids double-caching (the guest already caches), uses
  less host RAM, and gives more predictable behavior. That's why `cache=none` is a
  frequent production choice: good performance *and* correctness (it still honors guest
  flushes).

### Guest flush / FUA semantics

A correct guest issues a **flush** (e.g. `fsync`, or FUA writes) when it needs data
durable — exactly as it would on real hardware. Cache modes that honor flushes
(everything except `unsafe`) translate that guest flush into a real flush to disk, so
the guest's durability contract holds. `unsafe` breaks the contract by discarding the
flush — hence "unsafe."

### AIO backends

Separately from caching, **`aio=`** picks how QEMU issues asynchronous I/O to the
host:

- **`threads`** — a pool of host threads doing blocking I/O. Universal, the default.
- **`native`** — Linux native AIO (`libaio`). Lower overhead, best paired with
  `cache=none` (O_DIRECT). Good for throughput.
- **`io_uring`** — the modern Linux async interface; lowest overhead and best
  scalability on recent kernels.

A common high-performance combo: `cache=none,aio=native` (or `aio=io_uring`).

{: .note }
> **The one to never use in production**
> <code>cache=unsafe</code> ignores the guest's flush requests, so on a host power
> loss any data the guest "committed" but that's still in host RAM is gone — and the
> guest's filesystem can be left inconsistent because its barriers were silently
> dropped. It's only for disposable workloads where speed matters and the data doesn't
> (e.g. a one-shot OS install you'll snapshot afterward). For correctness +
> performance, the usual production default is <code>cache=none</code> (often with
> <code>aio=native</code> or <code>io_uring</code>).

---

## Lab

```bash
# 1. Boot with the typical production default: O_DIRECT + native AIO.
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -drive file=disk.qcow2,if=virtio,cache=none,aio=native -nographic &

# 2. Compare cache modes explicitly on the -drive:
$ qemu-system-x86_64 -accel kvm -m 2G \
    -drive file=disk.qcow2,if=virtio,cache=writeback -nographic &   # fast, cached
$ qemu-system-x86_64 -accel kvm -m 2G \
    -drive file=disk.qcow2,if=virtio,cache=unsafe -nographic &      # FAST, UNSAFE

# 3. Modern -blockdev form (cache/aio expressed as properties):
$ qemu-system-x86_64 -accel kvm -m 2G \
    -blockdev driver=qcow2,node-name=hd0,cache.direct=on,aio=native,\
file.driver=file,file.filename=disk.qcow2,file.aio=native \
    -device virtio-blk-pci,drive=hd0 -nographic &

# 4. Benchmark inside the guest with fio to FEEL the difference (Lesson 52 detail):
#   (guest)$ fio --name=w --rw=randwrite --bs=4k --size=512M \
#                --ioengine=libaio --direct=1 --runtime=20 --time_based
#   Run it under cache=none vs cache=writeback vs cache=unsafe and compare IOPS.

# 5. See O_DIRECT in action on the HOST: with cache=none the host page cache
#    for this file stays small; with writeback it grows. Watch buff/cache:
$ free -h    # observe 'buff/cache' before/after heavy guest I/O per mode
$ kill %1 %2 %3 %4 2>/dev/null
```

**Expected result:** `cache=unsafe` shows the highest IOPS but discards flushes (data
at risk); `cache=none,aio=native` gives strong, safe performance with little host
page-cache growth; `cache=writeback` grows the host page cache. The benchmark makes
the safety-vs-speed trade tangible.

---

## Further Reading

| Topic | Link |
|---|---|
| QEMU disk cache modes | [qemu.org — Disk I/O & cache](https://www.qemu.org/docs/master/system/qemu-block-drivers.html) |
| O_DIRECT | [man7.org — open(2) (O_DIRECT)](https://man7.org/linux/man-pages/man2/open.2.html) |
| Linux AIO / io_uring | [Wikipedia — io_uring](https://en.wikipedia.org/wiki/Io_uring) |
| Page cache | [Wikipedia — Page cache](https://en.wikipedia.org/wiki/Page_cache) |
| `fsync` / durability | [man7.org — fsync(2)](https://man7.org/linux/man-pages/man2/fsync.2.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Which cache mode risks data loss on host power failure, and which is the usual production default for correctness + performance?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<code>cache=unsafe</code> risks data loss: it ignores the guest's flush/FUA requests, so writes the guest believes are durable may still be in volatile host RAM at power loss, and the guest filesystem can be left inconsistent because its barriers were dropped. The usual production default is <code>cache=none</code> (O_DIRECT) — it bypasses the host page cache yet still honors guest flushes, giving good performance and correctness; it's often paired with aio=native or aio=io_uring.
</details>

---

**Q2. What does O_DIRECT (cache=none) change versus using the host page cache, and why is avoiding double-caching beneficial?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
O_DIRECT bypasses the host page cache, so guest I/O goes more directly to storage instead of being cached in host RAM. Avoiding double-caching is beneficial because the guest already maintains its own page cache; caching the same data again in the host wastes host memory, adds an extra volatile layer, and can hurt predictability. cache=none frees that host RAM, gives more deterministic I/O behavior, and still respects guest flushes — which is why it's a common production choice, especially with native/io_uring AIO.
</details>

---

**Q3. What does the `aio=` option control, and what's a good pairing with `cache=none`?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
aio= selects how QEMU issues asynchronous I/O to the host: <code>threads</code> (a worker-thread pool doing blocking I/O, universal default), <code>native</code> (Linux libaio, lower overhead), or <code>io_uring</code> (modern, lowest overhead and best scalability on recent kernels). A good pairing with cache=none is <code>aio=native</code> (or <code>aio=io_uring</code>), because native/io_uring AIO works best with O_DIRECT and minimizes overhead for high-throughput, low-latency disk workloads.
</details>

---

## Homework

Run a 4k random-write `fio` benchmark inside a guest three times, changing only the drive's cache mode: `cache=none`, `cache=writeback`, and `cache=unsafe`. Record the IOPS for each. Explain the ranking you observe and why you would still never ship `cache=unsafe` to production despite its numbers.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Typically cache=unsafe shows the highest IOPS, writeback next, and cache=none somewhat lower (it bypasses the host cache and honors every flush). unsafe wins because it ignores the guest's flushes — it never waits for data to reach disk, so the benchmark isn't paying for durability. That's exactly why you never use it in production: on a host crash/power loss, "committed" writes still in host RAM are lost and the guest filesystem's barriers were dropped, risking corruption. The benchmark's speed is borrowed against data safety. cache=none gives honest performance while keeping the guest's durability contract intact, which is the right production trade-off.
</details>
