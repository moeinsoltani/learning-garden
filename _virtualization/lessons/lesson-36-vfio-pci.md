---
title: "Lesson 36 — VFIO PCI Passthrough"
nav_order: 36
parent: "Phase 9: Device Passthrough"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 36: VFIO PCI Passthrough

## Concept

The IOMMU (Lesson 35) makes passthrough *safe*. **VFIO** (Virtual Function I/O) is the
kernel framework that makes it *happen*: it lets userspace (QEMU) securely own a physical
device and map its registers and DMA into a guest. The key act is **unbinding** the
device from its normal host driver and **binding** it to **`vfio-pci`**.

```
   BEFORE                               AFTER (bound to vfio-pci)
   ──────                               ─────────────────────────
   PCI device ◄── host driver           PCI device ◄── vfio-pci
   (e.g. e1000e drives the NIC;          (host driver detached; the device
    the HOST uses it)                     is now "claimed" for a guest)
                                         QEMU -device vfio-pci,host=... gives
                                         it to the guest, which drives it directly
```

Once a device is on `vfio-pci`, the **host no longer uses it** — it's reserved for the
guest, which drives the real hardware itself (with the IOMMU confining its DMA).

---

## How It Works

### The vfio-pci driver

`vfio-pci` is a stub driver: instead of operating the device, it exposes it through the
VFIO API so QEMU can map the device's BARs (register regions) into the guest and program
the IOMMU for the device's DMA. Binding works on whole IOMMU groups (Lesson 35).

### The binding dance

To hand device `0000:03:00.0` to a guest:

```
   1. Find its vendor:device ID:   lspci -nns 0000:03:00.0   → e.g. 8086:1533
   2. Unbind from the host driver:
        echo 0000:03:00.0 > /sys/bus/pci/devices/0000:03:00.0/driver/unbind
   3. Bind to vfio-pci:
        echo 8086 1533 > /sys/bus/pci/drivers/vfio-pci/new_id
      (or use driverctl / the modern bind-by-address sysfs)
   4. Launch QEMU:
        -device vfio-pci,host=0000:03:00.0
```

(All devices in the same IOMMU group must be bound to vfio-pci too.)

### Persisting the binding

Doing this by hand each boot is fragile — and worse, the host's normal driver may grab
the device first. To make it stick, you claim the device for vfio-pci **early in boot**,
before the regular driver loads:

- **`vfio-pci.ids=8086:1533`** on the kernel cmdline, or a modprobe option
  (`options vfio-pci ids=8086:1533`) loaded from the **initramfs** so vfio-pci binds
  first.
- **`driverctl set-override 0000:03:00.0 vfio-pci`** — a clean, persistent per-device
  override that survives reboots.
- Blacklisting the host driver (e.g. for GPUs) so it never claims the device.

Binding by **vendor:device ID** can be too broad (it captures *all* identical cards); for
that case, bind by PCI address or use `driverctl` per device.

### Device reset and the reset bug class

When a VM stops, the device must be **reset** so the next user (host or another VM) gets
it in a clean state. The standard mechanism is **FLR** (Function Level Reset). The
problem: **many devices implement reset poorly or not at all** — especially consumer
GPUs. A device that can't be properly reset may hang, fail to reinitialize after the VM
stops, or require a host reboot to recover (the infamous "reset bug," big in GPU
passthrough — Lesson 37). Quirks/workarounds (e.g. `vendor-reset`) exist for some
hardware.

{: .note }
> **What happens to the host's use of the device**
> The moment a device is bound to vfio-pci (and certainly once handed to a guest), the
> host's normal driver is detached and the host stops using it — the NIC disappears from
> the host's <code>ip link</code>, the GPU is no longer a host display, etc. The device
> is dedicated to the guest, which drives the real hardware directly with near bare-metal
> performance and the IOMMU enforcing DMA isolation. When the VM stops, the device is
> reset (FLR) and can be returned to the host or another VM — assuming it resets
> cleanly.

---

## Lab

```bash
# Pre-reqs: IOMMU enabled (Lesson 35), vfio-pci module available.
$ lsmod | grep vfio || sudo modprobe vfio-pci

# 1. Pick a SPARE device (don't grab your only NIC!). Identify it:
$ lspci -nn | grep -i ethernet
03:00.0 Ethernet controller [0200]: Intel I210 Gigabit [8086:1533]
$ dev=0000:03:00.0 ; ids=8086:1533

# 2. Confirm its IOMMU group is clean (only this device) — assign mates too if not:
$ ls /sys/bus/pci/devices/$dev/iommu_group/devices/
0000:03:00.0

# 3. Unbind from the host driver, bind to vfio-pci:
$ echo "$dev" | sudo tee /sys/bus/pci/devices/$dev/driver/unbind
$ echo "${ids/:/ }" | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id
# Verify it's now on vfio-pci:
$ lspci -nnk -s 03:00.0
03:00.0 Ethernet controller [8086:1533]
        Kernel driver in use: vfio-pci          ← claimed!
# And it's GONE from the host's interfaces:
$ ip link | grep -c enp3s0    # 0 — host no longer sees the NIC

# 4. Pass it to a guest:
$ sudo qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -device vfio-pci,host=$dev \
    -drive file=disk.qcow2,if=virtio -nographic &
#   (guest)$ lspci | grep -i ethernet     # the REAL Intel I210 appears in the guest
#   (guest)$ ip link                       # guest drives the physical NIC directly

# 5. Persist the binding across reboots (cleanest method):
$ sudo driverctl set-override 0000:03:00.0 vfio-pci
# (or kernel cmdline: vfio-pci.ids=8086:1533)

# 6. Stop the VM and observe the device reset / return to host:
$ sudo kill %1
$ lspci -nnk -s 03:00.0    # still vfio-pci (override) or rebindable to host driver
```

**Expected result:** After binding, `lspci -nnk` shows `Kernel driver in use:
vfio-pci` and the device vanishes from the host's interface list. Inside the guest, the
*real* physical device appears and the guest drives it directly. `driverctl` makes the
binding survive reboots.

---

## Further Reading

| Topic | Link |
|---|---|
| VFIO | [kernel.org — VFIO](https://docs.kernel.org/driver-api/vfio.html) |
| PCI passthrough via VFIO | [PCI passthrough — Arch Wiki](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF) |
| `driverctl` | [man7.org — driverctl(8)](https://man7.org/linux/man-pages/man8/driverctl.8.html) |
| Function Level Reset (FLR) | [Wikipedia — Function Level Reset](https://en.wikipedia.org/wiki/PCI_configuration_space) |
| `lspci` | [man7.org — lspci(8)](https://man7.org/linux/man-pages/man8/lspci.8.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What happens to the host's use of a device once it's bound to vfio-pci and handed to a guest?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The host's normal driver is unbound and detached, so the host stops using the device entirely — a passed-through NIC disappears from the host's <code>ip link</code>, a GPU stops being a host display, etc. The device is dedicated to vfio-pci/the guest, which then drives the real hardware directly (near bare-metal), with the IOMMU confining its DMA to the guest's memory. When the VM stops, the device is reset (FLR) and can be returned to the host or another VM, provided it resets cleanly.
</details>

---

**Q2. Outline the steps to bind a PCI device to vfio-pci for passthrough.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) Identify the device's PCI address and vendor:device ID (lspci -nn). (2) Ensure its whole IOMMU group will be assigned (bind all group-mates too). (3) Unbind it from its current host driver (echo the address to the driver's unbind). (4) Bind it to vfio-pci (echo the vendor/device IDs to vfio-pci/new_id, or use driverctl/bind-by-address). (5) Launch QEMU with <code>-device vfio-pci,host=&lt;addr&gt;</code>. For persistence, claim it early via vfio-pci.ids on the kernel cmdline / initramfs or a driverctl override so the host driver never grabs it first.
</details>

---

**Q3. What is the "reset bug class," and why does it matter when a VM using a passed-through device shuts down?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
When a VM stops, its passed-through device must be reset (typically via Function Level Reset) so the next user gets it in a clean state. Many devices — notably consumer GPUs — implement reset poorly or not at all. Such a device may hang, fail to reinitialize, or be unusable until a host reboot after the VM that used it shuts down. This matters because it can prevent reusing the device (returning it to the host or starting another VM with it) and can destabilize the host. Workarounds like vendor-reset quirks exist for some hardware, but a reliable FLR is what makes clean device reuse possible.
</details>

---

## Homework

Take a *spare* NIC (not your primary), bind it to vfio-pci, and confirm with `lspci -nnk -s <addr>` that the kernel driver in use is vfio-pci and that the NIC no longer appears in the host's `ip link`. Boot a guest with `-device vfio-pci,host=<addr>` and confirm the real NIC appears inside the guest. Then explain why binding by `vfio-pci.ids=VENDOR:DEVICE` could be problematic if you have two identical cards.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
After binding, <code>lspci -nnk</code> shows "Kernel driver in use: vfio-pci" and the NIC is gone from the host's ip link; inside the guest the physical NIC appears and the guest drives it. Binding by <code>vfio-pci.ids=VENDOR:DEVICE</code> is problematic with two identical cards because the ID match is by model, not by slot — it would claim *both* cards for vfio-pci, including the one you wanted to keep for the host. To assign only one of identical devices you must bind by PCI address instead (e.g. driverctl set-override on the specific 0000:03:00.0, or a script that binds the exact address), so the other identical card stays on its host driver.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 37 — GPU Passthrough →](lesson-37-gpu-passthrough){: .btn .btn-primary }
