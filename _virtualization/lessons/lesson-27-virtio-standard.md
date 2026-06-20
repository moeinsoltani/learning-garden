---
title: "Lesson 27 — The virtio Standard"
nav_order: 27
parent: "Phase 7: VirtIO & Paravirtualization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 27: The virtio Standard

## Concept

Emulating real hardware is slow: the guest pokes registers, each poke is a VM exit
(Lesson 10). **virtio** is a standardized *paravirtual* device framework — the guest
knows it's a VM and uses a cooperative, shared-memory interface instead of pretending
a fake NIC is real. The core mechanism is the **virtqueue**: a ring buffer in memory
shared between guest and host.

```
   EMULATED NIC                       VIRTIO-NET
   ───────────                        ──────────
   guest driver writes registers      guest driver puts a buffer descriptor
   one at a time → each = VM exit      into a shared ring → ONE notification
   (tens of exits per packet)          for a whole batch of packets
                                       (≈ exits amortized to near zero)
```

Instead of trapping on every register access, the guest places work in a shared ring
and rings a doorbell *once* for a batch. Far fewer exits, far higher throughput.

---

## How It Works

### Virtqueues and vrings

A virtio device has one or more **virtqueues**. Each is a **vring** with three parts
living in guest memory the host can also read:

```
   Descriptor table  — array of {addr, len, flags, next}: points at buffers
   Available ring    — guest → host: "these descriptors are ready to process"
   Used ring         — host → guest: "these are done, here's the result"
```

The flow: the guest fills buffers, links them via descriptors, adds their indices to
the **available** ring, and notifies the host (a doorbell write — one exit, or even
zero if suppressed). The host processes the buffers and posts indices to the **used**
ring, then interrupts the guest. Both sides advance through the rings without copying
the ring itself — only the data buffers are exchanged, by reference.

### Split vs packed virtqueues

- **Split virtqueue** (classic): the three rings are *separate* structures
  (descriptor table + available + used). Simple and universal.
- **Packed virtqueue** (newer): collapses them into a *single* ring with in-band flags
  marking availability/used. Better cache behavior and fewer memory accesses per
  descriptor — more efficient on modern CPUs. Negotiated as a feature if both sides
  support it.

### Feature negotiation — the handshake

virtio devices and drivers negotiate capabilities at init via a **feature bits**
handshake, mediated by a status/feature register set:

```
   1. device exposes the features it offers (e.g. checksum offload, mergeable bufs,
      packed vq, multiqueue, indirect descriptors)
   2. driver reads them, ACKs the subset it also supports
   3. both proceed using only the agreed-upon features
   4. driver sets DRIVER_OK; device is live
```

This is why one virtio-net device works across guest kernels of different ages — each
side enables only what both understand.

### Transports: virtio-pci vs virtio-mmio

The virtqueue mechanics are transport-independent; how the device is *discovered and
its registers reached* differs:

- **virtio-pci** — the device appears as a PCI(e) device. Standard for full machines
  (q35/pc); supports MSI-X interrupts, hotplug, etc. The default you'll use.
- **virtio-mmio** — a bare memory-mapped register block with no PCI bus. Used by
  minimal machines like `microvm` (Phase 15) and some embedded/ARM targets, where
  dropping PCI shrinks the device model and speeds boot.

{: .note }
> **Why fewer exits = faster (the whole point)**
> An emulated e1000 forces the guest to interact with fake hardware registers, and
> each interaction traps to QEMU (PIO/MMIO exit) — a world switch costing thousands of
> cycles, many times per packet. virtio-net replaces that with a shared ring: the
> guest batches many packets and notifies once, and features like notification
> suppression can drive exits toward zero under load. Fewer world switches means more
> CPU spent moving data and less spent context-switching — the core reason virtio
> dramatically outperforms emulated devices.

---

## Lab

```bash
# 1. See virtio devices QEMU offers (frontends), across transports:
$ qemu-system-x86_64 -device help 2>&1 | grep -iE 'virtio.*pci|virtio.*mmio' | head
name "virtio-net-pci", ...
name "virtio-blk-pci", ...
name "virtio-scsi-pci", ...
name "virtio-gpu-pci", ...

# 2. Boot a guest with a virtio-net NIC over PCI:
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -netdev user,id=n0 -device virtio-net-pci,netdev=n0 \
    -drive file=disk.qcow2,if=virtio -nographic &

# 3. INSIDE the guest, confirm virtio drivers are bound and see negotiated features:
#   (guest)$ lspci | grep -i virtio
#   00:03.0 Ethernet controller: Red Hat, Inc. Virtio network device
#   (guest)$ lsmod | grep virtio
#   virtio_net, virtio_pci, virtio_ring, virtio ...
#   (guest)$ ls /sys/bus/virtio/devices/        # each virtX is a virtio device
#   (guest)$ cat /sys/bus/virtio/devices/virtio0/features   # negotiated feature bits

# 4. Inspect the virtqueues of a running device (guest side):
#   (guest)$ ethtool -l eth0     # shows combined/queue counts (multiqueue, Lesson 33)

# 5. Contrast transport: microvm uses virtio-MMIO instead of PCI (preview Ph.15):
$ qemu-system-x86_64 -M microvm -accel kvm -m 512 -nographic \
    -device virtio-blk-device,drive=hd0 \
    -blockdev driver=raw,node-name=hd0,file.driver=file,file.filename=disk.raw 2>&1 | head
$ kill %1 2>/dev/null
```

**Expected result:** The guest binds `virtio_net`/`virtio_pci`/`virtio_ring` modules,
lists the device under `/sys/bus/virtio/devices/`, and exposes the negotiated feature
bits — confirming the device/driver handshake and the shared-ring model rather than
emulated-register I/O.

---

## Further Reading

| Topic | Link |
|---|---|
| virtio | [Wikipedia — Virtio](https://en.wikipedia.org/wiki/Virtio) |
| virtio specification (OASIS) | [oasis-open.org — virtio spec](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html) |
| Paravirtualization | [Wikipedia — Paravirtualization](https://en.wikipedia.org/wiki/Paravirtualization) |
| Ring buffer | [Wikipedia — Circular buffer](https://en.wikipedia.org/wiki/Circular_buffer) |
| QEMU virtio docs | [qemu.org — virtio](https://www.qemu.org/docs/master/system/devices/virtio-pmem.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why does a virtio-net device cause far fewer VM exits than an emulated e1000 NIC?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An emulated e1000 makes the guest interact with fake hardware registers; each register access is a PIO/MMIO VM exit to QEMU — many expensive world switches per packet. virtio-net instead uses a shared-memory virtqueue: the guest places buffer descriptors into a ring and notifies the host once for a whole batch (and notifications can be suppressed under load). So a batch of packets costs roughly one exit instead of dozens of per-register exits, dramatically reducing world switches and raising throughput.
</details>

---

**Q2. Name the three parts of a (split) virtqueue and what each is for.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) The descriptor table — an array of entries describing buffers ({address, length, flags, next}), used to chain buffers. (2) The available ring — guest→host: the guest posts the indices of descriptors it has made ready for the host to process. (3) The used ring — host→guest: the host posts the indices of descriptors it has finished, returning results. The guest fills buffers and advertises them via available; the host consumes them and acknowledges via used. (A packed virtqueue collapses these into one ring with in-band flags.)
</details>

---

**Q3. What is virtio feature negotiation, and why does it let one device work across guests of different ages?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Feature negotiation is the init-time handshake where the device advertises a set of feature bits (checksum offload, mergeable buffers, packed virtqueue, multiqueue, indirect descriptors, etc.), the driver acknowledges the subset it also supports, and both then operate using only the agreed features before the driver signals DRIVER_OK. It lets one device serve guests of different ages because each guest enables only what both it and the device understand — a newer guest gets the advanced features, an older guest falls back to the common baseline, all from the same device definition.
</details>

---

## Homework

Boot a guest with `-device virtio-net-pci`, then inside the guest read `/sys/bus/virtio/devices/virtio0/features` (or the relevant virtioN) and `ethtool -i eth0`. Identify that the driver is virtio_net and that feature negotiation occurred. Then explain, in terms of virtqueues, what happens when the guest sends a burst of 100 packets — roughly how many notifications/exits that takes versus an emulated NIC.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
ethtool -i reports driver: virtio_net, and the features file shows the negotiated bits — proof the device/driver handshake ran. When the guest sends 100 packets, virtio-net places ~100 buffer descriptors into the TX virtqueue's available ring and rings the doorbell — often just one notification for the whole batch (and with notification suppression / busy host, possibly fewer), so on the order of a single exit rather than per-packet. An emulated e1000 would require many register accesses per packet (descriptor setup, tail pointer writes), each trapping to QEMU, so ~hundreds to thousands of exits for the same 100 packets. That gap is why virtio's shared-ring batching is so much faster.
</details>
