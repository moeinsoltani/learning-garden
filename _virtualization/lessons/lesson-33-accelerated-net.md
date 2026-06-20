---
title: "Lesson 33 — Accelerated Networking: vhost-net and Multiqueue"
nav_order: 33
parent: "Phase 8: Networking for VMs"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 33: Accelerated Networking — vhost-net and Multiqueue

## Concept

You have a guest on the LAN via TAP + bridge (Lesson 32). Now make it *fast*. Two
levers: **vhost-net** (Lesson 29 — move the datapath into the kernel) and
**multiqueue** (scale packet processing across multiple vCPUs instead of one).

```
   SINGLE QUEUE                         MULTIQUEUE (N queues)
   ────────────                         ─────────────────────
   all packets ─► 1 virtqueue ─► 1 CPU   flow1 ─► q0 ─► CPU0
   bottleneck: one CPU's worth of        flow2 ─► q1 ─► CPU1
   packet processing                     flow3 ─► q2 ─► CPU2
                                         parallel: scales with vCPUs
```

A single virtio-net queue is processed by one CPU — so throughput caps at one core's
packet-processing capacity no matter how many vCPUs the guest has. Multiqueue spreads
flows across several queues and cores.

---

## How It Works

### vhost-net (recap + how to verify)

Enable with `vhost=on` on the TAP netdev. The datapath moves into a kernel `vhost-<pid>`
thread (Lesson 29), removing per-batch QEMU userspace round trips. Verify it's active:
the kernel thread exists, `/dev/vhost-net` is open, and under load QEMU's CPU and
`kvm_stat` userspace exits stay low. This is the single biggest networking win and
should essentially always be on for TAP-backed virtio-net.

### Multiqueue virtio-net

A single virtqueue pair (RX/TX) is serviced by one CPU, so a busy guest hits a
one-core ceiling. **Multiqueue** gives the NIC *N* queue pairs, and the host (with
vhost) can run *N* vhost threads, while the guest distributes flows across the queues —
parallelizing across vCPUs.

It takes coordination on **both sides**:

1. **QEMU/host:** request N queues on the device and the TAP.

   ```
   -netdev tap,id=n0,ifname=tap0,vhost=on,queues=4 \
   -device virtio-net-pci,netdev=n0,mq=on,vectors=10
   #   vectors ≈ 2*queues + 2  (one MSI-X vector per queue direction + config)
   ```

2. **Guest:** the queues exist but default to one active. Turn them on with ethtool:

   ```
   (guest)$ ethtool -L eth0 combined 4
   ```

Without the guest's `ethtool -L`, extra queues sit idle and you get no benefit — a
classic "I enabled multiqueue but nothing changed" gotcha.

### Offloads (checksum, TSO/GSO)

virtio-net supports **offloads** negotiated as features:

- **Checksum offload** — the guest skips computing TCP/UDP checksums; done later.
- **TSO/GSO** (TCP/Generic Segmentation Offload) — the guest hands over large segments
  and segmentation happens lower down, so fewer, bigger units traverse the stack →
  higher throughput, less CPU.

These are usually on by default and beneficial. You toggle them with `ethtool -K` for
debugging (e.g. disabling GSO/TSO when chasing a checksum/MTU bug), but normally leave
them enabled.

### Measuring

Use **iperf3** (recall **Networking Lesson 33**) to measure throughput, and watch
`kvm_stat` for exits and `ethtool -S eth0` for per-queue stats. Compare: vhost off vs
on; single queue vs multiqueue (before and after `ethtool -L`).

{: .note }
> **Why the guest "must also do something" for multiqueue**
> Requesting <code>queues=4,mq=on</code> only makes the queues *available*. The guest's
> driver starts with a single combined queue active; it must be told to use the rest via
> <code>ethtool -L eth0 combined 4</code> (and the guest needs enough vCPUs and RX flow
> steering/RSS to actually spread flows). Only then do multiple vhost threads and vCPUs
> process packets in parallel. Multiqueue is a both-ends feature: host provisions, guest
> activates.

---

## Lab

```bash
# 1. vhost-net ON with a 4-queue multiqueue virtio-net (TAP/bridge from L32):
$ sudo qemu-system-x86_64 -accel kvm -m 4G -smp 4 \
    -netdev tap,id=n0,ifname=tap0,script=no,downscript=no,vhost=on,queues=4 \
    -device virtio-net-pci,netdev=n0,mq=on,vectors=10 \
    -drive file=disk.qcow2,if=virtio -nographic &

# 2. Confirm vhost threads (should see multiple with multiqueue):
$ ps -eLf | grep vhost | grep -v grep
root ... [vhost-12345]
root ... [vhost-12345]    ← multiple workers for multiple queues

# 3. INSIDE the guest, see available vs active queues and activate them:
#   (guest)$ ethtool -l eth0
#   Pre-set maximums:  Combined: 4
#   Current hardware settings:  Combined: 1     ← only 1 active by default!
#   (guest)$ sudo ethtool -L eth0 combined 4    ← activate all 4
#   (guest)$ ethtool -l eth0
#   Current hardware settings:  Combined: 4

# 4. Check offloads are on:
#   (guest)$ ethtool -k eth0 | grep -E 'tcp-segmentation-offload|generic-segmentation|rx-checksumming'
#   tcp-segmentation-offload: on
#   generic-segmentation-offload: on

# 5. Measure throughput with iperf3 (guest -> host or another LAN host):
#   (host)$ iperf3 -s
#   (guest)$ iperf3 -c <host-ip> -P 4     # 4 parallel streams to exercise queues
# Compare: vhost=off vs on; combined 1 vs combined 4. Watch per-queue stats:
#   (guest)$ ethtool -S eth0 | grep -E 'rx_queue_._packets|tx_queue_._packets'
$ sudo kill %1 2>/dev/null
```

**Expected result:** With vhost on you see kernel `vhost` threads; `ethtool -l` shows 4
queues available but only 1 active until `ethtool -L eth0 combined 4`. After activating
and running parallel iperf3 streams, per-queue stats show traffic spread across queues
and aggregate throughput scales beyond a single core.

---

## Further Reading

| Topic | Link |
|---|---|
| Networking Lesson 33 (bandwidth/iperf3) | [Bandwidth measurement]({{ '/networking/lessons/lesson-33-bandwidth.html' | relative_url }}) |
| vhost-net | [kernel.org — vhost](https://docs.kernel.org/driver-api/vhost.html) |
| Multiqueue virtio-net | [kernel.org — Scaling in the Linux networking stack](https://docs.kernel.org/networking/scaling.html) |
| `ethtool` | [man7.org — ethtool(8)](https://man7.org/linux/man-pages/man8/ethtool.8.html) |
| Large segmentation offload (TSO/GSO) | [Wikipedia — Large send offload](https://en.wikipedia.org/wiki/Large_send_offload) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Single-queue virtio-net caps out at one CPU's worth of packet processing. How does multiqueue fix this, and what must the guest also do?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A single virtqueue pair is serviced by one CPU (and one vhost thread), so throughput is limited to that core's packet-processing capacity. Multiqueue gives the NIC N queue pairs so multiple vhost threads and multiple vCPUs can process flows in parallel, scaling throughput with cores. The guest must also activate the extra queues: the host provisioning (queues=N, mq=on) only makes them available; the guest driver starts with one active queue and you must run <code>ethtool -L eth0 combined N</code> (and have enough vCPUs / RSS flow steering) for traffic to actually spread across queues.
</details>

---

**Q2. How do you verify vhost-net is actually active for a running guest?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Check the host for a kernel worker thread named [vhost-<pid>] (ps -eLf | grep vhost), confirm QEMU holds /dev/vhost-net open (lsof), and observe that under network load QEMU's own CPU usage stays low and kvm_stat shows few userspace exits — because the datapath is in the kernel thread, not QEMU. If you used vhost=on and these hold, vhost-net is active; if QEMU's CPU spikes and a vhost thread is absent, it fell back to userspace virtio.
</details>

---

**Q3. What do TSO/GSO offloads do, and why generally leave them on?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
TSO (TCP Segmentation Offload) / GSO (Generic Segmentation Offload) let the guest hand off large data units and defer segmentation into MTU-sized packets to a lower layer, so the network stack processes fewer, bigger units. This raises throughput and lowers CPU usage per byte. You leave them on because they're a free performance win in virtualized networking; you'd only disable them (via ethtool -K) temporarily when debugging checksum/MTU/fragmentation issues, since offloads can mask or interact with such bugs.
</details>

---

## Homework

Boot a guest with `queues=4,mq=on,vhost=on`. Run `iperf3 -c <host> -P 4` first with the guest at `ethtool -L eth0 combined 1`, then again at `combined 4`. Record throughput and check `ethtool -S eth0` per-queue counters in each case. Explain the results and confirm that simply requesting `queues=4` on the QEMU side wasn't enough.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With combined 1, all traffic funnels through one queue/one CPU, so throughput plateaus at roughly a single core's capacity and ethtool -S shows packets concentrated on queue 0. With combined 4, flows spread across all four queues (per-queue counters all increment) and multiple vhost threads/vCPUs process in parallel, so aggregate throughput rises (especially with -P 4 multiple streams). This confirms that requesting queues=4 on the QEMU side only *provisions* the queues; the guest must activate them with ethtool -L before they're used. Provisioning without guest activation leaves the extra queues idle and yields no speedup — multiqueue is a both-ends feature.
</details>
