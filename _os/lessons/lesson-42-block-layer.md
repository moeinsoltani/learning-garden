---
title: "Lesson 42 — The Block Layer and I/O Schedulers"
nav_order: 7
parent: "Phase 7: Files & Filesystems"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 42: The Block Layer and I/O Schedulers

## Concept

Below every filesystem sits the last kernel layer before hardware: the
**block layer** — the queueing, merging, and scheduling machinery that turns
"write these pages" into device commands.

```
   filesystem (ext4/xfs)  submits BIOs ("write blocks 100-107 of vda")
        │
   ┌────▼─────────────────────────────────────────┐
   │              BLOCK LAYER (blk-mq)            │
   │  per-CPU software queues → merge adjacent    │
   │  requests, apply the I/O SCHEDULER policy    │
   │  (who goes first? fairness? deadlines?)      │
   │        │                                     │
   │  hardware queues (1 on SATA … 64+ on NVMe)   │
   └────────┼─────────────────────────────────────┘
        │ driver (virtio-blk here! — virt L24)
   ┌────▼──────────┐
   │    device      │  its OWN queue + cache + reordering
   └───────────────┘
```

Why this layer earns its keep — and why it's shrinking:

- **Merging**: adjacent 4 KB writes coalesce into one large command —
  writeback's scattered pages become sequential streams (huge on HDD,
  still real on SSD).
- **Scheduling**: on a spinning disk, request *order* was everything (seek
  time!); schedulers reordered for locality and fairness. On NVMe with
  million-IOPS parallelism, the best policy is usually **`none`** — get out
  of the way. The middle ground survives: `mq-deadline` (bound latency,
  cheap) and `bfq` (full fairness — desktops, slow media).
- **The killer metrics**: `iostat -x` — and knowing which columns lie.
  `%util` means "the queue was non-empty" — a device that can do 64 things
  at once shows 100% while nine-tenths idle: **on NVMe, %util is nearly
  meaningless; `await` (per-I/O latency) and `aqu-sz` (queue depth) are the
  truth.**

You've been above this layer all phase (page cache → writeback → "disk") and
you met its consumers in virtualization (virtio-blk, `cache=none`,
io-threads). This lesson is the missing floor.

---

## How It Works

### blk-mq: the multi-queue redesign

The 2010s rebuild for flash: per-CPU **software queues** (no shared-lock
bottleneck — Lesson 26's contention economics, applied to I/O submission)
feeding the device's **hardware queues** (NVMe: up to 64K queues × 64K
entries; virtio-blk: configurable — virt Lesson 33's multiqueue). Requests
merge and sort in the software queues; the scheduler is a pluggable policy
between software and hardware queues. This is why modern Linux scales to
millions of IOPS — and why scheduler choice matters *less* every year.

### The scheduler menu (per device!)

- **`none`** — FIFO into the hardware queues. NVMe default and correct
  there: the device parallelizes internally; software reordering just adds
  latency.
- **`mq-deadline`** — reads get a 500 ms deadline, writes 5 s (reads block
  someone *now* — Lesson 19's major faults!; writes are usually async
  writeback). Cheap insurance on SATA SSDs and mixed loads; a common
  default.
- **`bfq`** — budget fair queueing: per-process budgets, weight support
  (`ionice` finally *does* something real, and cgroup `io.weight` —
  Lesson 60), designed for desktops/HDDs where one background `cp` can
  starve the UI.

`cat /sys/block/vda/queue/scheduler` shows (bracketed = active); echo to
change, udev rules to persist.

### Reading iostat -x like an engineer

- `r/s, w/s, rkB/s, wkB/s` — the demand.
- `await` (r_await/w_await) — average ms per completed I/O *including queue
  time*: the user-experienced number. SSD sane: <1–2 ms; NVMe: <0.5 ms;
  climbing await with flat throughput = queueing (overload or a dying
  device).
- `aqu-sz` — average requests in flight: compare to what the device can
  absorb (SATA ~32; NVMe: hundreds+).
- `%util` — fraction of time the device had ≥1 request. Only meaningful as
  "busy" on serial devices (HDD). On NVMe treat as a hint, never an alarm.

The one-liner triage: **high await + low throughput = latency problem
(queueing/device); high await + high aqu-sz + high throughput = you're
simply asking for more than it has.**

{: .note }
> **Watching requests individually**
> <code>blktrace -d /dev/vda -o - | blkparse -i -</code> streams every
> request's lifecycle (queued → merged → dispatched → completed, with
> sector and latency); <code>biolatency</code>/<code>biosnoop</code> from
> the BCC/eBPF toolkit (networking Phase 11's tooling!) give histograms and
> per-process attribution with less ceremony. When iostat says "slow" and
> nobody confesses, these name the process and the pattern.

---

## Lab

```bash
# ---- 1. Your device's queue anatomy ----
$ lsblk -o NAME,SIZE,TYPE,ROTA                  # ROTA 0 = SSD/flash-backed
$ cat /sys/block/vda/queue/scheduler 2>/dev/null || cat /sys/block/sda/queue/scheduler
# [none] mq-deadline    or    none [mq-deadline] bfq
$ D=$(lsblk -dno NAME | head -1)
$ grep . /sys/block/$D/queue/{nr_requests,max_sectors_kb,rotational} 2>/dev/null
$ ls /sys/block/$D/mq/ 2>/dev/null | head -4     # the hardware queues (blk-mq)

# ---- 2. iostat -x: create load, read the truth ----
# terminal 2: iostat -x 1   (watch await, aqu-sz, %util, w/s)
$ fio --name=seqwrite --filename=/tmp/fio.dat --size=256M --bs=1M \
      --rw=write --direct=1 --numjobs=1 --runtime=10 --time_based 2>/dev/null | tail -3 \
  || dd if=/dev/zero of=/tmp/fio.dat bs=1M count=256 oflag=direct
# sequential: high wkB/s, modest w/s, await low — the device's happy path

$ fio --name=randwrite --filename=/tmp/fio.dat --size=256M --bs=4k \
      --rw=randwrite --direct=1 --iodepth=32 --runtime=10 --time_based 2>/dev/null | tail -3 \
  || echo "(apt install fio for the random-I/O leg)"
# random 4k: w/s way up, wkB/s way down, await/aqu-sz reveal queueing —
# same device, 100x different character. PATTERN is everything in storage.

# ---- 3. Merging: watch the block layer consolidate ----
$ grep . /sys/block/$D/queue/nomerges            # 0 = merging enabled
$ iostat -x 1 2 | tail -3                        # note wrqm/s (write requests
# merged per second) during buffered writes — writeback's scattered pages
# arriving as merged, larger requests. Free efficiency.
$ dd if=/dev/zero of=/tmp/merge.dat bs=4k count=20000 2>/dev/null & iostat -x 1 3 | grep -A1 Device | tail -4
$ wait

# ---- 4. Scheduler swap: measure, don't believe ----
$ cat /sys/block/$D/queue/scheduler
$ command -v fio >/dev/null && for s in none mq-deadline; do
    echo $s | sudo tee /sys/block/$D/queue/scheduler >/dev/null 2>&1 || continue
    echo "=== $s ==="
    fio --name=mix --filename=/tmp/fio.dat --size=256M --bs=4k --rw=randrw \
        --direct=1 --iodepth=16 --runtime=8 --time_based 2>/dev/null \
        | grep -E 'read:|write:' | head -2
  done
# on a VM's virtual disk the difference is usually small — which IS the
# lesson: measure on YOUR stack before cargo-culting scheduler advice.

# ---- 5. Per-process I/O attribution ----
$ sudo iotop -bon 2 2>/dev/null | head -8 || pidstat -d 1 3
# who is actually writing? (pidstat -d: kB_wr/s per process — always available)

# ---- 6. The (safe) dying-disk drill: what latency pathology looks like ----
$ command -v fio >/dev/null && fio --name=lat --filename=/tmp/fio.dat --size=64M \
      --bs=4k --rw=randread --direct=1 --iodepth=1 --runtime=5 --time_based 2>/dev/null \
      | grep -E 'lat.*avg|clat percentiles' | head -3
# note p99/p99.9 vs avg — a healthy device is TIGHT; a dying one grows a tail
# (avg 0.3ms, p99.9 800ms = the "it's slow sometimes" ghost). Percentiles
# catch what averages bury — in storage as in services.
$ rm -f /tmp/fio.dat /tmp/merge.dat
```

---

## Further Reading

| Topic | Link |
|---|---|
| blk-mq — kernel docs | <https://www.kernel.org/doc/html/latest/block/blk-mq.html> |
| I/O scheduling (Wikipedia) | <https://en.wikipedia.org/wiki/I/O_scheduling> |
| BFQ scheduler — kernel docs | <https://www.kernel.org/doc/html/latest/block/bfq-iosched.html> |
| `iostat(1)` man page | <https://man7.org/linux/man-pages/man1/iostat.1.html> |
| `blktrace(8)` man page | <https://man7.org/linux/man-pages/man8/blktrace.8.html> |
| `fio(1)` — flexible I/O tester | <https://fio.readthedocs.io/en/latest/fio_doc.html> |

---

## Checkpoint

**Q1.** `%util` is 100% but the NVMe disk isn't saturated. Why is this metric
misleading on modern SSDs, and which two iostat columns answer "is it
actually overloaded?"

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
%util measures the fraction of time the device had <em>at least one</em>
request outstanding — a utilization concept from serial devices (HDDs) that
process one request at a time, where non-empty-queue really meant busy. An
NVMe device services dozens-to-thousands of requests concurrently across its
hardware queues: one long-running request per second keeps the queue
technically non-empty (%util → 100%) while 99% of the device's parallelism
sits idle. The honest columns: <strong>await</strong> — if per-I/O latency
is normal for the device class (say &lt;0.5 ms), it's coping regardless of
%util; rising await means requests are queueing beyond capacity. And
<strong>aqu-sz</strong> — in-flight depth vs what the device absorbs
(hundreds for NVMe): aqu-sz of 2 with %util 100% is a nearly idle NVMe;
aqu-sz of 500 with climbing await is real saturation. Rule: %util is binary
evidence ("was there work"), await/aqu-sz are the load gauges.
</details>

**Q2.** Why does mq-deadline give reads a 500 ms deadline but writes 5 s —
justify the asymmetry from earlier phases, and name the workload where it's
the wrong bias.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Reads are almost always <em>synchronous</em> — a process is blocked waiting:
a major page fault (Lesson 19), an application read miss, a binary's code
page (Lesson 20) — every ms of read latency is a ms of a human or request
waiting, and D-state processes pile into load (Lesson 15). Writes are almost
always <em>asynchronous</em>: writeback flushing dirty pages (Lesson 20)
minutes after the application "wrote" — nobody is waiting on any individual
write, so they can yield to reads almost indefinitely; the 5 s bound merely
prevents infinite starvation (dirty-page backlog eventually throttles
writers — Lesson 20's other mechanism). Wrong-bias workload: fsync-heavy
transactional writes — a database's WAL commits (Lesson 41) ARE synchronous
writes with a client blocked on each; a read-flood starving them inverts the
intended fairness (your commits crawl so someone's table scan flies). Fixes
there: FUA/flush requests get priority handling anyway, separate the WAL
device, or cgroup io.weight to protect the writer explicitly (Lesson 60).
</details>

**Q3.** A batch job doing sequential 1 MB reads and an interactive service
doing 4 KB random reads share one SATA SSD. Describe what each experiences
under `none` vs `bfq`, and the non-scheduler fix that beats both.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Under <code>none</code>: FIFO mixing — the batch job keeps the queue full of
large requests; each of the service's 4 KB reads waits behind megabytes of
queued batch data: service p99 latency balloons (tens of ms) while the batch
job gets full bandwidth. Interactivity loses exactly proportional to how
well the batch job pipelines. Under <code>bfq</code>: per-process budgets —
the service's small infrequent reads get scheduled promptly within its
budget (latency protected, near-idle-disk numbers), the batch job pays with
somewhat lower throughput (fairness bookkeeping + lost merging
opportunities: bfq's CPU/throughput tax, the reason it's not default for
fast devices). The better fix: <strong>don't share the contention
point</strong> — cgroup io.max/io.weight on the batch job (policy targeted
at the offender, not global scheduling), or put the interactive service's
data on a different device/class entirely (NVMe for latency-sensitive, SATA
for batch — Lesson 38/41's per-data-class theme reaching hardware). Bulkhead
beats arbitration whenever you can afford it — the recurring systems lesson.
</details>

---

## Homework

Your VM's disk is `virtio-blk` (virt Lessons 24/33): the guest's block layer
feeds a virtqueue to QEMU, which submits to the *host's* block layer, its
scheduler, and the real device. Answer: (a) which scheduler should the
*guest* run and why; (b) where does a guest `fsync` land in this stack
(trace it through Lesson 41's layers, both kernels); (c) why can guest
iostat show await=8ms while the host's NVMe shows await=0.2ms — name two
places the missing 7.8 ms can hide.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) <code>none</code>. The guest's "device" is a software queue to QEMU —
seek-locality reordering optimizes an illusion, and the <em>host</em> will
re-queue/merge/schedule everything again anyway: guest-side scheduling is
pure double work (the same reasoning as NVMe: parallelism behind the
interface makes reordering someone else's job). Distro images ship this
default for virtio disks. (b) guest app fsync → guest kernel: flush guest
page cache pages to the virtual disk + a virtio-blk FLUSH request through
the virtqueue → QEMU: with cache=none/O_DIRECT (Lesson 41's homework),
data went straight to host block layer, and the FLUSH becomes
fdatasync/device-flush on the image fd; with cache=writeback, QEMU's fsync
must first drain <em>host</em> page cache. Then host block layer →
NVMe flush → device's power-safe media. Two full kernels' worth of Lesson 41
layers, chained — and any layer configured to lie (cache=unsafe!) breaks the
whole contract. (c) The gap hides in the middle hops: (1) <strong>QEMU/
virtqueue processing</strong> — the I/O thread wakeups (ioeventfd — Lesson
35!), request copying, and host syscall submission add per-request latency
invisible to both block layers — especially if the io-thread is CPU-starved
on a busy host (virt Lesson 50's pinning exists for this); (2) <strong>host-
side queueing above the device</strong> — the host block layer/cgroup
throttles (io.max!), a deep host queue shared with other VMs, or host page-
cache writeback interference: host iostat measures only after dispatch to
the NVMe, so time spent queued in software (or in QEMU) appears in neither
device's await. Moral: in virtualized storage, end-to-end latency must be
measured end-to-end — per-layer tools each see only their floor.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 8 — Event-Driven & Async I/O (Lesson 43: epoll) →](lesson-43-epoll){: .btn .btn-primary }
