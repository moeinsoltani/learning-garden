---
title: "Lesson 24 — Block Device Models"
nav_order: 24
parent: "Phase 6: Storage"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 24: Block Device Models

## Concept

The image format (Lesson 22) is *where the bytes live*. The **block device model** is
*how the guest sees the disk* — what kind of controller and disk the guest's driver
binds to. The same qcow2 file can appear to the guest as an ancient IDE drive or a
modern virtio disk; the choice dominates performance.

```
   emulated real hardware                paravirtual (virtio)
   ──────────────────────                ─────────────────────
   IDE  → SATA/AHCI                       virtio-blk    (simple block dev)
   slow: every register access            virtio-scsi   (full SCSI stack)
   traps to QEMU (MMIO exits)             fast: shared-ring, few exits
   (use only for compatibility)          (use for everything modern)
```

For anything modern you want **virtio-blk** or **virtio-scsi**. The emulated
controllers (IDE, AHCI) exist for compatibility — e.g. an installer that lacks
virtio drivers, or a guest OS too old to know virtio.

---

## How It Works

### The emulated controllers

- **IDE** — ancient PATA emulation. Maximally compatible (every OS has drivers), very
  slow, limited to a few drives. Use only when nothing else works (old installers).
- **AHCI / SATA** — emulates a SATA controller. Better than IDE, still fully emulated
  hardware (register pokes → MMIO exits, Lesson 10). A reasonable fallback when
  virtio drivers aren't yet available (e.g. boot CD of an old OS).

### The paravirtual controllers (virtio)

- **virtio-blk** — a *simple, fast* paravirtual block device. The guest's virtio-blk
  driver talks to QEMU through a shared virtqueue ring, so a disk op is a ring entry +
  a notification, not a storm of register accesses. Each virtio-blk device is its own
  PCI device. Great for the common case of a handful of disks.
- **virtio-scsi** — a paravirtual **SCSI HBA**. One controller can host *many* disks
  (LUNs) behind it, supports the full SCSI command set, multiqueue, and crucially
  **SCSI passthrough** (sending real SCSI commands to a backing device, needed for
  things like persistent reservations, tape, or exact device features). Also handles
  **TRIM/UNMAP** (discard) cleanly.

### virtio-blk vs virtio-scsi — when each wins

| Need | Pick | Why |
|---|---|---|
| A few disks, lowest latency | **virtio-blk** | Minimal layering; each disk a direct PCI device. |
| Many disks on one guest | **virtio-scsi** | One HBA hosts many LUNs; you don't burn a PCI slot per disk. |
| SCSI passthrough / real device features | **virtio-scsi** | Only it forwards SCSI commands (reservations, UNMAP, etc.). |
| TRIM/discard, thin reclamation | **virtio-scsi** (or modern virtio-blk) | Propagates UNMAP so freed guest space is returned. |

### Modern attachment: -blockdev + -device

Recall Lesson 12's backend/frontend split. A virtio-scsi disk needs *three* pieces:
the backend, the SCSI **controller**, and the **disk** on it:

```
   -blockdev driver=qcow2,node-name=hd0,file.driver=file,file.filename=disk.qcow2 \
   -device virtio-scsi-pci,id=scsi0 \
   -device scsi-hd,drive=hd0,bus=scsi0.0
```

virtio-blk is simpler (no separate controller): `-device virtio-blk-pci,drive=hd0`.

### Discard / TRIM and thin reclamation

When a guest deletes files, the freed blocks aren't automatically returned to the
host's thin image — the host doesn't know they're free. **Discard/TRIM** propagates
"these blocks are now unused" down to QEMU, which can punch holes in the image
(reclaiming space) and pass it to the underlying storage. Enable with
`discard=unmap` on the device; the guest must also issue TRIM (`fstrim`, or mount
with `discard`).

{: .note }
> **"30 disks + SCSI passthrough" → virtio-scsi**
> If a guest needs many disks, attaching 30 individual virtio-blk PCI devices wastes
> PCI slots and is unwieldy; one virtio-scsi HBA hosts all 30 as LUNs behind a single
> controller. And if you need SCSI passthrough (forwarding real SCSI commands —
> persistent reservations for clustering, exact device geometry, UNMAP), only
> virtio-scsi can do it. virtio-blk is leaner for a couple of disks but lacks the SCSI
> command path and scales poorly to dozens of devices.

---

## Lab

```bash
# 1. Attach a disk as virtio-blk (simple, fast — the common case):
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -blockdev driver=qcow2,node-name=hd0,file.driver=file,file.filename=disk.qcow2 \
    -device virtio-blk-pci,drive=hd0 -nographic &
#   (guest)$ lsblk        # shows /dev/vda  (virtio-blk disks are vdX)

# 2. Attach via virtio-scsi (controller + disk), enabling discard:
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -blockdev driver=qcow2,node-name=hd0,file.driver=file,file.filename=disk.qcow2 \
    -device virtio-scsi-pci,id=scsi0 \
    -device scsi-hd,drive=hd0,bus=scsi0.0,discard_granularity=4096 -nographic &
#   (guest)$ lsblk        # shows /dev/sda  (SCSI disks are sdX)

# 3. Contrast with an EMULATED controller (slow; only for compatibility):
$ qemu-system-x86_64 -accel kvm -m 2G \
    -drive file=disk.qcow2,if=ide -nographic &
#   (guest)$ lsblk        # /dev/sda but driven by the emulated IDE/SATA path

# 4. Many disks on ONE virtio-scsi HBA (no extra PCI slots per disk):
$ qemu-system-x86_64 -accel kvm -m 4G \
    -device virtio-scsi-pci,id=scsi0 \
    -blockdev driver=qcow2,node-name=d1,file.driver=file,file.filename=disk1.qcow2 \
    -device scsi-hd,drive=d1,bus=scsi0.0 \
    -blockdev driver=qcow2,node-name=d2,file.driver=file,file.filename=disk2.qcow2 \
    -device scsi-hd,drive=d2,bus=scsi0.0 -nographic &

# 5. Demonstrate discard/TRIM reclaiming space (virtio-scsi with discard=unmap):
#   (guest)$ fstrim -v /          # tells host which blocks are free
$ du -h disk.qcow2               # image shrinks as freed clusters are punched
$ kill %1 %2 %3 %4 2>/dev/null
```

**Expected result:** virtio-blk disks appear as `/dev/vdX`, SCSI disks (emulated or
virtio-scsi) as `/dev/sdX`. One virtio-scsi controller can carry multiple disks. With
discard enabled, `fstrim` in the guest causes the host image to shrink.

---

## Further Reading

| Topic | Link |
|---|---|
| virtio | [Wikipedia — Virtio](https://en.wikipedia.org/wiki/Virtio) |
| QEMU block device docs | [qemu.org — Block devices](https://www.qemu.org/docs/master/system/qemu-block-drivers.html) |
| SCSI | [Wikipedia — SCSI](https://en.wikipedia.org/wiki/SCSI) |
| TRIM / discard | [Wikipedia — Trim (computing)](https://en.wikipedia.org/wiki/Trim_(computing)) |
| AHCI | [Wikipedia — Advanced Host Controller Interface](https://en.wikipedia.org/wiki/Advanced_Host_Controller_Interface) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. You need 30 disks and SCSI passthrough on one guest. virtio-blk or virtio-scsi? Why?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
virtio-scsi. A single virtio-scsi HBA hosts many disks as LUNs behind one controller, so 30 disks don't consume 30 PCI slots the way 30 virtio-blk devices would, and it scales cleanly. Crucially, only virtio-scsi supports SCSI passthrough — forwarding real SCSI commands to the backing device (persistent reservations for clustering, exact device features, UNMAP) — which virtio-blk cannot do. virtio-blk is leaner for one or two disks but lacks the SCSI command path and the many-LUN scaling.
</details>

---

**Q2. Why are emulated controllers like IDE/AHCI slow compared to virtio, and when would you still use them?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Emulated controllers pretend to be real hardware, so the guest driver pokes hardware registers; each access is an MMIO/PIO VM exit to QEMU's device model — many expensive exits per I/O. virtio devices instead use a shared-memory virtqueue ring with batched notifications, drastically cutting exits. You still use emulated controllers for compatibility: booting an OS/installer that lacks virtio drivers, or a guest too old to support virtio — typically as a temporary path until virtio drivers are available.
</details>

---

**Q3. What is discard/TRIM, and what must be true for deleting a file in the guest to actually shrink the host image?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Discard/TRIM is the mechanism by which the guest tells the storage stack "these blocks are no longer in use," allowing QEMU to punch holes in the thin image (reclaiming host space) and pass the hint to the underlying storage. For a guest delete to shrink the host image: the device must be configured to honor discard (e.g. discard=unmap on virtio-scsi or modern virtio-blk), AND the guest must actually issue TRIM — either by mounting with the discard option or running fstrim. A plain file delete without TRIM leaves the clusters allocated in the image.
</details>

---

## Homework

Attach the same qcow2 to a guest first as virtio-blk, then as virtio-scsi, booting each time. In each guest run `lsblk` and note the device name (vdX vs sdX). Then enable `discard=unmap` on the virtio-scsi device, write and delete a large file in the guest, run `fstrim /`, and compare `du -h` of the image before and after. Explain what you observed.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
As virtio-blk the disk appears as /dev/vda; as virtio-scsi it appears as /dev/sda (SCSI naming). After writing then deleting a large file, the qcow2's du size stays inflated because the delete alone doesn't return clusters. Once you run <code>fstrim /</code> with discard=unmap enabled, the guest issues UNMAP for the freed blocks, QEMU punches holes in the image, and <code>du -h</code> of the qcow2 drops back down — reclaiming the space. This shows that thin reclamation requires both a discard-capable device path and an explicit TRIM from the guest; the device model (virtio-scsi here) is what carries the UNMAP down to the image.
</details>
