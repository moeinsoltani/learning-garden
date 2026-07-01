---
title: "Lesson 12 — Anatomy of a QEMU Command Line"
nav_order: 12
parent: "Phase 4: QEMU Fundamentals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 12: Anatomy of a QEMU Command Line

## Concept

A QEMU command line looks intimidating, but it's really just a description of a
**virtual computer**, part by part: how much RAM, how many CPUs, which disk, which
network card, how you see the screen. Think of it as assembling a PC from a parts
list, expressed as flags.

```
   qemu-system-x86_64 \      ← which CPU architecture to build
     -accel kvm \            ← how to execute guest code (HW vs emulation)
     -m 2G \                 ← RAM
     -smp 2 \                ← CPUs (cores)
     -drive file=disk.qcow2,if=virtio \   ← storage
     -nographic             ← I/O: serial console instead of a window
```

Two flags do the heavy lifting once you go beyond the basics: **`-drive`/`-blockdev`**
define a *backend* (the actual bytes — a file), and **`-device`** plugs a *frontend*
(the virtual hardware the guest sees) onto a bus. Understanding the
backend/frontend split is the key that unlocks the whole QEMU model.

---

## How It Works

### Per-architecture binaries

`qemu-system-x86_64` builds an x86-64 PC. `qemu-system-aarch64` builds an ARM64
machine, `qemu-system-riscv64` a RISC-V one, and so on. Each is a separate binary
because each emulates a different CPU and machine. On a matching host,
`-accel kvm` runs guest code natively (Lessons 1, 13).

### The essential flags

| Flag | Meaning |
|---|---|
| `-m 2G` | Guest RAM size. |
| `-smp 2` | Number of vCPUs (expandable to sockets/cores/threads, Lesson 18). |
| `-drive file=…` / `-blockdev` | A storage **backend** — the file/device holding bytes. |
| `-cdrom file.iso` | Shortcut for an IDE CD-ROM drive. |
| `-netdev` | A network **backend** (user/tap/etc., Phase 8). |
| `-device` | A virtual **device** (frontend) plugged onto a bus. |
| `-object` | A backend object (memory backend, rng source, tls creds, …). |
| `-machine` | The virtual motherboard/chipset (Lesson 13). |
| `-cpu` | The CPU model exposed to the guest (Lesson 17). |
| `-accel` | The execution engine: `kvm`, `tcg`, … (Lesson 13). |
| `-nographic` | No graphical window; multiplex serial + monitor on the terminal. |

### The crucial distinction: `-drive` vs `-device`

This trips up everyone at first. They describe **different halves** of the same
disk:

- **`-drive`** (or the modern **`-blockdev`**) declares the **backend**: *where the
  bytes live* — `file=disk.qcow2`, the format, caching, etc.
- **`-device`** declares the **frontend**: *what hardware the guest sees* —
  `virtio-blk-pci`, `ide-hd`, `scsi-hd` — and which backend it's attached to.

The convenient `-drive file=disk.qcow2,if=virtio` is **shorthand** that creates
*both* at once (a backend plus an implied virtio-blk frontend). The explicit,
modern form separates them:

```
   -blockdev driver=qcow2,node-name=hd0,file.driver=file,file.filename=disk.qcow2 \
   -device   virtio-blk-pci,drive=hd0
   #          ^frontend (what guest sees)   ^references the backend by name
```

The same pattern repeats for networking (`-netdev` backend + `-device` NIC
frontend) and is exactly how libvirt's XML is structured. Learn it once, apply it
everywhere.

{: .note }
> **Backend vs frontend — the mental model**
> Frontend = the virtual *hardware* the guest's driver binds to (a virtio-blk PCI
> device). Backend = the host-side *source of data* (a qcow2 file, a TAP interface,
> a host directory). QEMU's job is to wire a frontend to a backend. Decoupling them
> lets you, e.g., attach the same "virtio-blk disk" frontend to a file today and an
> iSCSI LUN tomorrow without the guest noticing.

---

## Lab

```bash
# Prepare a disk and grab a tiny bootable image. (qemu-img covered in Phase 6.)
$ qemu-img create -f qcow2 disk.qcow2 10G
Formatting 'disk.qcow2', fmt=qcow2 ... size=10737418240

# 1. Minimal boot from an install ISO over a serial console:
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -drive file=disk.qcow2,if=virtio \
    -cdrom alpine-virt.iso \
    -boot d \
    -nographic
# You'll see firmware messages then the guest boot, all on your terminal.
# Press Ctrl-A then X to quit QEMU; Ctrl-A then C toggles the QEMU monitor.

# 2. The SAME disk, expressed with explicit backend/frontend separation:
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -blockdev driver=qcow2,node-name=hd0,file.driver=file,file.filename=disk.qcow2 \
    -device virtio-blk-pci,drive=hd0 \
    -nographic

# 3. List the device types QEMU can plug in (the frontends):
$ qemu-system-x86_64 -device help 2>&1 | grep -iE 'virtio-blk|virtio-net|e1000|nvme' | head
name "virtio-blk-pci", ...
name "virtio-net-pci", ...
name "e1000", ...
name "nvme", ...

# 4. List machine types and accelerators (preview Lesson 13):
$ qemu-system-x86_64 -machine help | head
$ qemu-system-x86_64 -accel help
```

**Expected result:** The shorthand and explicit forms boot the *same* VM. You can
enumerate available device frontends with `-device help`, confirming `-device` and
`-drive`/`-blockdev` are two halves of one disk.

---

## Further Reading

| Topic | Link |
|---|---|
| QEMU invocation (full flag reference) | [qemu.org — Invocation](https://www.qemu.org/docs/master/system/invocation.html) |
| QEMU block devices (`-blockdev`) | [qemu.org — Block device docs](https://www.qemu.org/docs/master/system/qemu-block-drivers.html) |
| QEMU user docs | [qemu.org — System Emulation](https://www.qemu.org/docs/master/system/index.html) |
| QEMU (overview) | [Wikipedia — QEMU](https://en.wikipedia.org/wiki/QEMU) |
| qcow2 / qemu-img | [qemu.org — qemu-img](https://www.qemu.org/docs/master/tools/qemu-img.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What is the difference between `-drive` and `-device`, and how do they relate?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`-drive` (or the modern `-blockdev`) defines the storage *backend* — where the bytes actually live (a qcow2 file, its format and caching). `-device` defines the *frontend* — the virtual hardware the guest sees (e.g. virtio-blk-pci, ide-hd) plugged onto a bus, referencing a backend by name. They're two halves of one disk: the device is what the guest's driver talks to; the drive is the host-side data source it's wired to. `-drive file=...,if=virtio` is shorthand that creates both at once.
</details>

---

**Q2. Why does QEMU separate "backend" from "frontend" at all? Give a practical benefit.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Decoupling lets the same virtual hardware frontend be wired to different host data sources without the guest noticing. For example a virtio-blk disk the guest sees can be backed by a local qcow2 file today and an iSCSI LUN or RBD volume tomorrow — only the backend changes. It also lets one backend be shared/referenced cleanly and keeps device modeling independent of storage/network plumbing. The same pattern (netdev backend + device NIC frontend) applies to networking and maps directly onto libvirt's XML.
</details>

---

**Q3. What does `-nographic` do, and how do you quit QEMU / reach the monitor when using it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`-nographic` disables the graphical window and multiplexes the guest's serial console (and the QEMU monitor) onto your terminal, which is ideal for headless/SSH use. With it, Ctrl-A then X quits QEMU, and Ctrl-A then C toggles between the guest serial console and the QEMU (HMP) monitor. (Ctrl-A then H shows the help for these escape sequences.)
</details>

---

## Homework

Take the shorthand command `-drive file=disk.qcow2,if=virtio` and rewrite it using the explicit `-blockdev` + `-device virtio-blk-pci` form (as in the lab). Then add a *second* disk as a separate `-blockdev`/`-device` pair. Explain why the explicit form is what libvirt and automation prefer over the shorthand.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Explicit form for two disks:<br>
<code>-blockdev driver=qcow2,node-name=hd0,file.driver=file,file.filename=disk.qcow2 -device virtio-blk-pci,drive=hd0</code><br>
<code>-blockdev driver=qcow2,node-name=hd1,file.driver=file,file.filename=disk2.qcow2 -device virtio-blk-pci,drive=hd1</code><br>
Automation prefers the explicit form because each node and device is independently named and addressable (node-name=hd0), which is required for unambiguous hotplug, snapshots, live block operations (block-commit/mirror via QMP), and for libvirt to map each element onto a distinct XML node. The shorthand bundles backend+frontend opaquely, giving less control and no stable handle to reference later.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 13 — Machine Types and Accelerators →](lesson-13-machine-types-accel){: .btn .btn-primary }
