---
title: "Lesson 53 — Observability: Measuring a Running VM"
nav_order: 53
parent: "Phase 13: Performance Tuning"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 53: Observability — Measuring a Running VM

## Concept

"The VM feels slow" is not a diagnosis. This lesson is the virtualization analog of
**Networking Lesson 47**: a disciplined, *bottom-up* method to find *which layer* is the
bottleneck, with the right tool for each. Don't guess — measure each layer in order.

```
   THE LAYERED METHOD (check in this order)
   ┌────────────────────────────────────────────────────────────┐
   │ 1. CPU / VM exits   kvm_stat, perf kvm    → too many exits?  │
   │ 2. CPU contention   virt-top, host top    → steal time?      │
   │ 3. Memory / NUMA    numastat, /proc        → remote/swap?    │
   │ 4. Disk I/O         iostat, fio, iotop     → disk-bound?     │
   │ 5. Network          iperf3, ethtool        → net-bound?      │
   └────────────────────────────────────────────────────────────┘
   Start at the bottom (CPU/exits); only move up when a layer is clean.
```

---

## How It Works

### Layer 1 — CPU and VM exits

Start where virtualization overhead lives (Lessons 10–11). `kvm_stat` shows exits by
reason; `perf kvm stat` adds timing; `perf kvm top` is a live view. A guest with excessive
MMIO/PIO exits points to emulated devices that should be virtio; excessive HLT is just
idle. This is the layer unique to virtualization — check it first because it's invisible to
in-guest tools.

### Layer 2 — CPU contention (host vs guest view)

Two viewpoints matter:

- **Host view:** `virt-top` (a `top` for VMs) shows per-VM CPU/mem/IO; `top -H`/`htop` show
  per-*thread* CPU so you can see a vCPU or iothread pegged. Overcommit shows here.
- **Guest view:** inside the guest, **steal time** (`st` in `top`/`vmstat`) reveals the
  guest waiting for a physical CPU it can't get — a host-contention symptom invisible from
  guest CPU% alone.

Mismatch (guest thinks it's busy but host shows the vCPU idle/waiting) localizes the
problem to scheduling/overcommit.

### Layer 3 — Memory and NUMA

`numastat -p <qemu-pid>` shows whether guest memory is local or remote (Lesson 21/51);
`/proc/<pid>/status` and `free` reveal swapping. Remote memory or any swapping of guest
pages is a red flag — fix with pinning/binding/hugepages (Lesson 51).

### Layer 4 — Disk I/O

Host `iostat -x 1` shows per-device utilization/await; `iotop` shows which process
(QEMU/iothread) drives it; inside the guest, `iostat`/`fio` confirm whether the guest is
disk-bound. High `await` and ~100% util mean the storage layer is the limit (tune per
Lesson 52, or the backing device is saturated).

### Layer 5 — Network

`iperf3` (Networking Lesson 33) measures throughput; `ethtool -S` shows per-queue stats and
drops; `kvm_stat` confirms whether networking is causing exits (emulated NIC → switch to
virtio + vhost, Lesson 33).

### The methodology (mirror of Networking L47)

Work **bottom-up**: confirm CPU/exits are healthy, then CPU contention, then memory/NUMA,
then disk, then network. Each layer has a tool that gives a yes/no answer. Stopping at the
first guess ("must be the disk") without ruling out exits/steal/NUMA is how people tune the
wrong thing for hours.

{: .note }
> **Why a bottom-up order**
> Lower layers can *masquerade* as higher-layer problems. A guest that "feels like slow
> disk" might actually be suffering vCPU steal time (CPU contention) that delays I/O
> submission, or remote NUMA memory that slows the page cache — both of which look like
> sluggish disk from inside the guest. If you start at the top you'll tune storage that was
> never the bottleneck. Checking CPU exits → contention → memory/NUMA → disk → network in
> order ensures each suspected cause is actually ruled out before you blame the next layer.

---

## Lab

```bash
# LAYER 1 — VM exits (virtualization-specific; check first):
$ sudo kvm_stat -1 | sort -k2 -nr | head
kvm_exit            54213
kvm_halt             8800        ← mostly idle? fine.
kvm_mmio             402         ← high MMIO? emulated device → use virtio
$ sudo perf kvm stat record -a -- sleep 5 && sudo perf kvm stat report | head

# LAYER 2 — CPU contention, host AND guest views:
$ virt-top                       # per-VM CPU/mem/disk/net (host view)
$ top -H -p $(pgrep -f 'guest=slowvm')   # which THREAD (vCPU/iothread) is hot
#   (guest)$ vmstat 1            # watch the 'st' (steal) column — >0 means contention

# LAYER 3 — Memory / NUMA:
$ numastat -p $(pgrep -f 'guest=slowvm')   # local vs remote guest memory
$ grep -E 'VmSwap' /proc/$(pgrep -f 'guest=slowvm')/status   # any swap of guest mem?

# LAYER 4 — Disk I/O:
$ iostat -x 1 3                  # %util ~100 and high await = disk-bound (host)
$ sudo iotop -o                  # which process drives the I/O
#   (guest)$ iostat -x 1         # confirm from the guest's perspective

# LAYER 5 — Network:
#   (host)$ iperf3 -s ; (guest)$ iperf3 -c <host>     # throughput
#   (guest)$ ethtool -S eth0 | grep -E 'drop|error|queue'   # drops / per-queue
$ sudo kvm_stat -1 | grep -i 'mmio\|pio'   # net causing exits? -> virtio+vhost
```

**Expected result:** Each layer's tool gives a clear signal — exit-reason histogram,
steal-time percentage, local-vs-remote memory, disk %util/await, network throughput/drops
— so you can point to the *specific* bottleneck instead of guessing. A healthy lower layer
lets you move up with confidence.

---

## Further Reading

| Topic | Link |
|---|---|
| Networking Lesson 47 (debugging methodology) | [Network debugging methodology]({{ '/networking/lessons/lesson-47-debugging-methodology.html' | relative_url }}) |
| `kvm_stat` / perf kvm (Lesson 11) | [Observing KVM at runtime]({{ '/virtualization/lessons/lesson-11-observing-kvm.html' | relative_url }}) |
| `virt-top` | [virt-manager.org — virt-top](https://virt-manager.org/) |
| `iostat` | [man7.org — iostat(1)](https://man7.org/linux/man-pages/man1/iostat.1.html) |
| CPU steal time | [Wikipedia — CPU time](https://en.wikipedia.org/wiki/CPU_time) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. A guest "feels slow." Outline the order of checks you'd run, from CPU exits down to I/O, and which tool answers each.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Bottom-up: (1) CPU/VM exits — kvm_stat / perf kvm stat — are there excessive MMIO/PIO/userspace exits (emulated devices) vs harmless HLT? (2) CPU contention — virt-top and host top -H for per-thread load, plus the guest's steal time (vmstat 'st') — is the vCPU waiting for a physical core (overcommit)? (3) Memory/NUMA — numastat -p <pid> for local vs remote memory and /proc status for swap — is guest RAM remote or being swapped? (4) Disk I/O — iostat -x / iotop on the host and iostat/fio in the guest — high %util/await means disk-bound. (5) Network — iperf3 for throughput, ethtool -S for drops/queues, kvm_stat for net-induced exits. Each layer has a tool giving a yes/no answer; move up only when the current layer is clean.
</details>

---

**Q2. What is steal time, and from which viewpoint (host or guest) do you observe it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Steal time is the time a guest's vCPU is runnable but can't get a physical CPU because the host is busy running other vCPUs/tasks — i.e. the hypervisor "stole" CPU time from this guest. You observe it from inside the guest, as the "st" column in top/vmstat (the guest can tell it was scheduled out involuntarily). A high steal time indicates host CPU overcommit/contention, even if the guest's own CPU usage looks moderate; from the host side you'd corroborate it with virt-top / per-thread CPU showing many vCPUs competing for cores.
</details>

---

**Q3. Why check VM exits (Layer 1) before concluding "it's a disk problem"?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because lower-layer problems can masquerade as disk slowness from inside the guest. Excessive VM exits (e.g. an emulated disk/NIC causing MMIO/PIO storms) add latency that looks like sluggish I/O, and CPU steal or remote NUMA memory can delay I/O submission and page-cache operations so the guest "feels disk-bound" when the real cause is elsewhere. VM exits are also invisible to in-guest tools — only host-side kvm_stat/perf kvm reveal them. Checking exits (and then contention/memory) first ensures you don't spend hours tuning storage that was never the bottleneck.
</details>

---

## Homework

Pick a VM and run one tool per layer (kvm_stat, virt-top + guest vmstat, numastat, iostat, iperf3). For each, write down the single number that tells you whether that layer is healthy (e.g. exit rate, steal %, remote-memory %, disk %util, throughput). Then describe a concrete scenario where the guest reports "slow disk" but the actual culprit is a *different* layer, and which number would reveal it.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Per-layer health numbers: kvm_stat → MMIO/PIO/userspace-exit rate (low = healthy; high = emulated-device overhead); guest vmstat → steal % (≈0 = healthy); numastat → fraction of guest memory on the local node (≈100% = healthy) and any swap; iostat → disk %util and await (well below 100%/low await = healthy); iperf3 → throughput vs link capacity. A concrete cross-layer scenario: a guest reports "slow disk" (high latency on reads), but the host is heavily CPU-overcommitted — the vCPU keeps getting descheduled, delaying I/O submission and completion handling. The revealing number is steal time in the guest's vmstat 'st' column (and many competing vCPU threads in host top): it's high, while iostat on the host shows the disk is actually under-utilized with low await. The fix is reducing CPU overcommit/pinning, not touching storage — which only checking the CPU-contention layer would have shown.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 54 — tuned and Host Profiles →](lesson-54-tuned-profiles){: .btn .btn-primary }
