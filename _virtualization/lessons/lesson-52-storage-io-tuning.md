---
title: "Lesson 52 — Storage and I/O Tuning"
nav_order: 52
parent: "Phase 13: Performance Tuning"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 52: Storage and I/O Tuning

## Concept

A VM's disk performance is gated by where I/O is processed and how parallel it is. The
levers: dedicate **iothreads** so disk work doesn't compete with the QEMU main loop, use
**multiqueue** so I/O scales across vCPUs, pick the right **cache/aio** (Lesson 25), and
**measure** with `fio`.

```
   DEFAULT (main loop)                  TUNED (dedicated iothread + multiqueue)
   ────────────────────                 ───────────────────────────────────────
   guest I/O ─► QEMU main loop          guest I/O ─► dedicated iothread(s)
                (shared with device       (disk has its own thread, off the
                 emulation, timers)        main loop) + N queues across vCPUs
   → I/O serialized behind other work   → parallel, isolated, higher IOPS
```

The big idea mirrors networking's vhost: get the disk datapath its own execution context
so it isn't stuck behind unrelated QEMU work.

---

## How It Works

### iothreads — dedicated disk threads

By default, block I/O completions are handled by QEMU's **main loop**, which also runs
device emulation and timers — so a busy disk competes with everything else and can be
serialized behind it. An **iothread** is a separate QEMU thread dedicated to processing a
disk's I/O. Assign a busy disk to its own iothread and that disk's I/O runs in parallel
with the main loop and other disks:

```xml
   <iothreads>2</iothreads>
   <disk ...>
     <driver name='qemu' type='qcow2' cache='none' io='native' queues='4'/>
     <target dev='vda' bus='virtio'/>
     <iothread>1</iothread>          <!-- this disk handled by iothread 1 -->
   </disk>
```

Pin iothreads off the vCPU cores (Lesson 50) for latency-sensitive guests.

### virtio-blk multiqueue and queue depth

Like networking (Lesson 33), a single virtqueue is serviced by one context. **Multiqueue**
(`queues=N` on the disk, or `num-queues` on virtio-blk) gives the disk multiple queues so
I/O parallelizes across vCPUs — important for high-IOPS NVMe-backed storage. **Queue
depth** (how many in-flight requests) also matters: deeper queues let fast storage stay
busy; benchmark to find the sweet spot.

### cache and aio revisited (for throughput)

From Lesson 25: for performance with correctness, **`cache=none`** (O_DIRECT, bypasses the
host page cache, avoids double-caching) paired with **`aio=native`** or **`aio=io_uring`**
(low-overhead async I/O). `io_uring` is the modern best on recent kernels. This combination
is the throughput default; revisit it here because it's part of the tuning toolkit.

### Discard/TRIM, write-back vs guarantees

- **Discard/TRIM** (`discard=unmap`, Lesson 24) keeps thin images compact and lets fast
  SSDs reclaim space — good for sustained performance on thin storage.
- **Write-back vs guarantees:** faster cache modes that defer flushes risk data on crash
  (Lesson 25) — tune deliberately, not accidentally.

### Benchmarking with fio

`fio` is the standard tool. Always test **inside the guest** with realistic patterns
(random vs sequential, block size, queue depth, `--direct=1`), and change *one* variable at
a time:

```
   fio --name=randread --rw=randread --bs=4k --iodepth=32 \
       --ioengine=libaio --direct=1 --size=2G --runtime=30 --time_based
```

Compare default vs iothread vs multiqueue vs cache modes and read the IOPS/latency.

{: .note }
> **What dedicating an iothread to a busy disk accomplishes**
> By default a disk's I/O completions are processed by QEMU's main loop, which is shared
> with device emulation and timers — so heavy disk I/O can be serialized behind, or contend
> with, that other work, capping IOPS and adding latency. A dedicated iothread gives that
> disk its own thread to process its virtqueue/completions, so its I/O runs in parallel with
> the main loop and other disks (and, when multiqueue is enabled, across several queues).
> The result is higher throughput, lower latency, and isolation from unrelated QEMU work —
> the storage analog of moving the network datapath off the main loop with vhost.

---

## Lab

```bash
# 1. Add iothreads and assign a busy disk to one (XML):
$ virsh edit dbvm
#   <iothreads>2</iothreads>
#   <disk type='file' device='disk'>
#     <driver name='qemu' type='qcow2' cache='none' io='native' queues='4'/>
#     <source file='/var/lib/libvirt/images/db.qcow2'/>
#     <target dev='vda' bus='virtio'/>
#     <iothread>1</iothread>
#   </disk>

# 2. Pin iothreads off the vCPU cores (Lesson 50):
#   <cputune><iothreadpin iothread='1' cpuset='0-1'/></cputune>

# 3. Verify the iothread exists and where it runs:
$ ps -To pid,tid,comm,psr -p $(pgrep -f 'guest=dbvm') | grep -i iothread
12345 12350 iou1/IO            0     ← dedicated iothread on housekeeping core

# 4. Benchmark inside the guest, one variable at a time:
#   (guest)$ fio --name=rr --rw=randread --bs=4k --iodepth=32 \
#                --ioengine=libaio --direct=1 --size=2G --runtime=30 --time_based
#   read: IOPS=85.0k ...        ← record baseline, then change config & re-run

# 5. Compare configurations (re-run fio after each single change):
#   a) main loop (no iothread)   vs  dedicated iothread
#   b) queues=1                  vs  queues=4 (multiqueue)
#   c) cache=writeback           vs  cache=none,aio=native  vs  aio=io_uring

# 6. Confirm discard keeps the thin image compact under churn:
#   (guest)$ fstrim -v /
$ du -h /var/lib/libvirt/images/db.qcow2
```

**Expected result:** A dedicated iothread thread appears (and runs on the core you pinned
it to). `fio` shows higher IOPS / lower latency with the iothread + multiqueue +
`cache=none,aio=native/io_uring` combination than with the default main-loop, single-queue
setup.

---

## Further Reading

| Topic | Link |
|---|---|
| libvirt iothreads | [libvirt.org — IOThreads](https://libvirt.org/formatdomain.html#iothreads-allocation) |
| QEMU block performance / iothreads | [qemu.org — Block device docs](https://www.qemu.org/docs/master/system/qemu-block-drivers.html) |
| `fio` | [fio.readthedocs.io](https://fio.readthedocs.io/en/latest/fio_doc.html) |
| io_uring | [Wikipedia — io_uring](https://en.wikipedia.org/wiki/Io_uring) |
| Caching modes (Lesson 25) | [Caching modes and AIO]({{ '/virtualization/lessons/lesson-25-caching-aio.html' | relative_url }}) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What does dedicating an iothread to a busy disk accomplish that the default (main loop) handling does not?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
By default a disk's I/O completions are processed by QEMU's main loop, which is shared with device emulation and timers — so heavy disk I/O competes with and can be serialized behind that other work, limiting IOPS and adding latency. A dedicated iothread gives the disk its own thread to process its virtqueue and completions, so its I/O runs in parallel with the main loop and other disks (and across multiple queues if multiqueue is enabled). The result is higher throughput, lower latency, and isolation from unrelated QEMU activity — analogous to moving the network datapath off the main loop with vhost.
</details>

---

**Q2. Which cache/aio combination is the usual throughput-with-correctness choice, and why?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
cache=none (O_DIRECT) paired with aio=native (or aio=io_uring on recent kernels). cache=none bypasses the host page cache, avoiding double-caching (the guest already caches) — saving host RAM and giving predictable I/O — while still honoring the guest's flush requests for correctness. native/io_uring provide low-overhead asynchronous I/O that works best with O_DIRECT, maximizing IOPS and minimizing CPU per operation. Together they give strong throughput without sacrificing crash correctness (unlike cache=unsafe).
</details>

---

**Q3. Why must you benchmark storage *inside the guest* with `fio`, changing one variable at a time?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You benchmark inside the guest because that's where the application actually experiences I/O — it includes the full virtio/iothread/cache path the guest sees, not just raw host disk speed. You use realistic fio patterns (random vs sequential, block size, queue depth, --direct) because performance depends heavily on the workload shape. And you change one variable at a time (iothread on/off, queues=1 vs N, cache/aio mode) so you can attribute any IOPS/latency change to that specific knob; changing several at once makes results uninterpretable. Controlled, in-guest measurement is the only reliable way to know which tuning actually helped your workload.
</details>

---

## Homework

Benchmark a guest disk with `fio` (4k random read, iodepth 32) in four configurations, changing one thing each time: (1) default main loop, single queue; (2) + dedicated iothread; (3) + multiqueue (queues=4); (4) + `cache=none,aio=io_uring`. Record IOPS for each and identify which change gave the biggest gain for your storage. Explain why that change mattered most given your backing device (HDD vs SSD vs NVMe).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You'll see IOPS rise as you add an iothread, then multiqueue, then the cache=none/io_uring combo, though the biggest gain depends on the backing device. On fast NVMe, multiqueue + io_uring usually dominate, because the device can serve far more parallel IOPS than a single queue/main-loop context can submit — the bottleneck is submission parallelism, so adding queues and a low-overhead async engine unlocks the device. On a single HDD, the device itself caps IOPS (seek-bound), so iothread/multiqueue help little and the differences are small — you're storage-limited, not software-limited. On a SATA SSD it's in between. The lesson: tuning the I/O path helps most when the backing device has more performance to give than the default single-threaded, single-queue path can extract; on slow media the device is the limit and software tuning yields little.
</details>
