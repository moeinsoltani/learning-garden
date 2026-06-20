---
title: "Lesson 38 — SR-IOV and Mediated Devices"
nav_order: 38
parent: "Phase 9: Device Passthrough"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 38: SR-IOV and Mediated Devices

## Concept

Full passthrough (Lessons 36–37) gives one device to one guest. But what if you want to
share *one* physical device among *many* guests, still with hardware-level performance?
Two answers, at different points on the isolation↔sharing spectrum:

```
   FULL PASSTHROUGH        SR-IOV VF              MEDIATED DEVICE (mdev)
   ────────────────        ─────────              ──────────────────────
   1 device → 1 guest      1 device → N guests    1 device → N guests
   (whole card)            via HARDWARE VFs       via SOFTWARE time-slicing
   max isolation,          near-HW perf, the      driver mediates; e.g.
   no sharing              card splits itself      NVIDIA vGPU, Intel GVT-g
```

**SR-IOV** splits a device in *hardware* into Virtual Functions, each assignable like a
mini-passthrough. **Mediated devices (mdev)** split a device in *software*: the host
driver time-slices/partitions it and presents virtual instances to guests.

---

## How It Works

### SR-IOV: PF and VFs (recap from Lesson 34)

An SR-IOV-capable device exposes one **Physical Function (PF)** (the full device, host-
managed) and many **Virtual Functions (VFs)** — lightweight hardware instances with their
own queues/resources. You create VFs by writing to `sriov_numvfs`, then assign a VF to a
guest via VFIO exactly like full passthrough:

```
   echo 4 > /sys/class/net/eth0/device/sriov_numvfs   # make 4 VFs
   # each VF is its own PCI function: 01:10.0, 01:10.1, ...
   -device vfio-pci,host=0000:01:10.0                 # give VF0 to a guest
```

Each guest's VF DMAs directly to its own memory (IOMMU-confined), and the device's
embedded switch moves data — **near bare-metal performance**, shared across guests. SR-IOV
is most common for NICs (and some NVMe/accelerators).

### Mediated devices (mdev): software-shared

Some devices can't make hardware VFs but *can* be shared by a smart host driver. **mdev**
is a kernel framework where the host driver creates **virtual device instances** (each a
UUID) carved from one physical device, and assigns them to guests via VFIO. The host
driver **mediates** access — time-slicing the engine and partitioning resources.

Classic examples:

- **NVIDIA vGPU** — one datacenter GPU presented as several vGPUs to different VMs, with
  the host driver scheduling GPU time among them.
- **Intel GVT-g** — mediated passthrough of Intel integrated graphics to multiple guests.

You create an mdev by writing a UUID into the device's `mdev_supported_types/.../create`
and then pass `-device vfio-pci,sysfsdev=/sys/.../<uuid>`.

### The spectrum: isolation vs sharing

| Mechanism | Sharing | Isolation | Performance | Live migration |
|---|---|---|---|---|
| **Full passthrough** | none (1:1) | highest (whole device) | bare metal | hardest |
| **SR-IOV VF** | hardware (1:N) | high (HW-partitioned) | near bare metal | hard |
| **mdev** | software (1:N) | medium (driver-mediated) | good, with overhead | varies (some vGPU supports it) |

Order by **isolation** (high→low): full passthrough → SR-IOV VF → mdev. Order by
**sharing** (more→less): mdev / SR-IOV (both 1:N) → full passthrough (1:1). The more you
share one device, the more you rely on hardware or software partitioning rather than
giving a guest the whole thing.

### Live-migration limitation

All three bind a guest to *physical* device state, so migration is harder than with
virtio: full passthrough and SR-IOV VFs are very hard to migrate (device state is in
hardware); some mdev implementations (notably certain NVIDIA vGPU setups) add migration
support because the *software* mediator can serialize state. The general rule stands:
the closer to raw hardware, the worse the migration story (the virtio trade-off in
reverse).

{: .note }
> **When to reach for which**
> Use <strong>full passthrough</strong> when one guest needs the whole device and maximum
> isolation/perf (a dedicated GPU, a database's NVMe). Use <strong>SR-IOV</strong> when a
> capable NIC must serve many guests at near-line-rate (cloud/NFV). Use <strong>mdev</strong>
> when the hardware can't make VFs but the vendor driver can share it — typically GPUs for
> VDI/ML where many VMs need a slice of one expensive accelerator. All three sacrifice easy
> live migration relative to virtio.

---

## Lab

```bash
# ---- SR-IOV ----
# 1. Does the device support SR-IOV, and how many VFs max?
$ cat /sys/class/net/eth0/device/sriov_totalvfs
64
# 2. Create 4 VFs:
$ echo 4 | sudo tee /sys/class/net/eth0/device/sriov_numvfs
$ lspci | grep -i 'Virtual Function'
01:10.0 Ethernet ... Virtual Function
01:10.1 Ethernet ... Virtual Function
...
# 3. (Optional) set a VF's MAC from the PF, then assign VF0 to a guest via VFIO:
$ sudo ip link set eth0 vf 0 mac 52:54:00:aa:bb:00
$ sudo driverctl set-override 0000:01:10.0 vfio-pci
$ sudo qemu-system-x86_64 -accel kvm -m 2G \
    -device vfio-pci,host=0000:01:10.0 \
    -drive file=disk.qcow2,if=virtio -nographic &
#   (guest)$ lspci | grep -i ethernet    # the VF appears as a real NIC

# ---- Mediated devices (mdev), e.g. Intel GVT-g / NVIDIA vGPU ----
# 4. List the mdev types a device supports:
$ ls /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/ 2>/dev/null
i915-GVTg_V5_4  i915-GVTg_V5_8  ...
$ cat /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/i915-GVTg_V5_4/description

# 5. Create an mdev instance with a fresh UUID, then assign it:
$ uuid=$(uuidgen)
$ echo "$uuid" | sudo tee \
    /sys/bus/pci/devices/0000:00:02.0/mdev_supported_types/i915-GVTg_V5_4/create
$ sudo qemu-system-x86_64 -accel kvm -m 4G \
    -device vfio-pci,sysfsdev=/sys/bus/pci/devices/0000:00:02.0/$uuid \
    -drive file=disk.qcow2,if=virtio -nographic &

# 6. Tear down VFs / mdev when done:
$ echo 0 | sudo tee /sys/class/net/eth0/device/sriov_numvfs
$ sudo kill %1 %2 2>/dev/null
```

**Expected result:** Writing `sriov_numvfs` creates VFs that appear as independent PCI
functions assignable to different guests; creating an mdev with a UUID yields a virtual
device instance you assign via VFIO — two ways to share one physical device among many
guests.

---

## Further Reading

| Topic | Link |
|---|---|
| SR-IOV | [Wikipedia — Single-root I/O virtualization](https://en.wikipedia.org/wiki/Single-root_input/output_virtualization) |
| Linux SR-IOV howto | [kernel.org — PCI IOV howto](https://docs.kernel.org/PCI/pci-iov-howto.html) |
| Mediated devices (mdev) | [kernel.org — VFIO Mediated devices](https://docs.kernel.org/driver-api/vfio-mediated-device.html) |
| Intel GVT-g | [Wikipedia — Intel GVT-g](https://en.wikipedia.org/wiki/Intel_GVT-g) |
| NVIDIA vGPU | [docs.nvidia.com — Virtual GPU](https://docs.nvidia.com/grid/) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What is the difference between passing through a whole device, an SR-IOV VF, and an mdev? Order them by isolation vs sharing.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Full passthrough gives one guest an entire physical device (1:1) with the highest isolation and bare-metal performance, but no sharing. An SR-IOV VF is a hardware-created virtual function of one device — the card splits itself in hardware so many guests each get a VF (1:N) with high, hardware-enforced isolation and near bare-metal performance. An mdev is a software-mediated virtual instance: the host driver time-slices/partitions one device and presents virtual devices to many guests (1:N), with medium isolation and some overhead. Ordered by isolation (high→low): full passthrough → SR-IOV VF → mdev. Ordered by sharing (more→less): mdev/SR-IOV (both 1:N) → full passthrough (1:1).
</details>

---

**Q2. How do you create SR-IOV VFs, and how is a VF then given to a guest?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You create VFs by writing the desired count to the device's <code>sriov_numvfs</code> sysfs file (bounded by sriov_totalvfs). The card then exposes that many Virtual Functions, each as its own PCI function (e.g. 01:10.0, 01:10.1, ...). You give a VF to a guest exactly like full passthrough via VFIO: bind that VF's PCI address to vfio-pci (e.g. with driverctl) and launch QEMU with <code>-device vfio-pci,host=&lt;vf-addr&gt;</code>. The guest then drives the VF as a real NIC, DMAing directly with IOMMU confinement.
</details>

---

**Q3. Why do all three sharing/passthrough mechanisms complicate live migration compared to virtio?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because they bind the guest to *physical* device state. With virtio the device is emulated/paravirtual, so its state lives in software QEMU can serialize and reconstruct on the destination. With full passthrough and SR-IOV VFs, the device's working state is in real hardware that can't be trivially captured and moved, and the destination must have an equivalent free device — so migration is very hard. mdev is in between: because a software mediator manages the virtual instance, some implementations (e.g. certain NVIDIA vGPU) can serialize state and support migration. The general rule: the closer the guest is to raw hardware, the worse the migration story.
</details>

---

## Homework

If your hardware supports it, create 2 SR-IOV VFs on a NIC and confirm they appear as separate PCI functions in `lspci`. (If not, list the mdev types under a device's `mdev_supported_types/`.) Then, for your environment, decide which of full passthrough / SR-IOV / mdev you'd use to give *eight* VMs fast network access from a single physical NIC, and justify why the other two are worse fits.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
For eight VMs sharing one fast NIC, <strong>SR-IOV</strong> is the right choice: create 8 VFs and assign one to each VM, giving each near bare-metal throughput and low CPU with hardware-enforced isolation — exactly what SR-IOV is designed for. <strong>Full passthrough</strong> is a poor fit because it's 1:1 — one NIC could serve only a single VM, so you'd need eight physical NICs. <strong>mdev</strong> is a poor fit for NICs because it's software-mediated time-slicing intended mainly for devices that can't make hardware VFs (typically GPUs); for networking it adds overhead and isn't the standard path, whereas SR-IOV NICs provide hardware partitioning natively. (If the NIC lacked SR-IOV, you'd fall back to virtio-net + vhost/multiqueue rather than mdev.)
</details>
