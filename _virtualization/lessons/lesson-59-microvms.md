---
title: "Lesson 59 — microVMs"
nav_order: 59
parent: "Phase 15: Lightweight VMs & Ecosystem"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 59: microVMs

## Concept

A full QEMU VM emulates a complete PC — BIOS, PCI bus, dozens of devices — which takes time
to boot and memory to run. **microVMs** throw almost all of that away: a minimal device
model, no PCI, no firmware, just enough to boot a Linux kernel directly. The result:
**millisecond boots** and tiny memory overhead — perfect for serverless and sandboxing,
where you launch and discard VMs constantly.

```
   FULL q35 VM                          microVM (QEMU microvm / Firecracker)
   ──────────                          ─────────────────────────────────────
   SeaBIOS/OVMF firmware                no firmware — kernel booted DIRECTLY
   PCI(e) bus + ACPI                    no PCI — virtio-MMIO devices
   dozens of emulated devices           a handful: virtio-net, virtio-blk, serial
   boots in seconds                     boots in ~tens of milliseconds
   ~100+ MiB overhead                   ~few MiB overhead
```

Less machine = faster boot, smaller footprint, smaller attack surface — at the cost of
features (no passthrough, limited devices).

---

## How It Works

### QEMU's `microvm` machine type

`-machine microvm` is a minimal machine: **no PCI bus, no BIOS/ACPI** (or minimal), using
**virtio-MMIO** devices (Lesson 27) instead of virtio-PCI. You boot the guest **kernel
directly** with `-kernel` (and `-initrd`/`-append`), skipping firmware and bootloader
entirely. Only the devices you explicitly add exist. This strips boot to the essentials.

### Firecracker and Cloud Hypervisor

These are **purpose-built Rust VMMs** that also drive `/dev/kvm` (Lesson 3) but replace QEMU
with a tiny, security-focused codebase:

- **Firecracker** (AWS) — powers AWS Lambda and Fargate. It supports *only* a minimal device
  set (virtio-net, virtio-blk, serial, a vsock, a one-button keyboard for reset), no PCI, no
  BIOS. Boots a microVM in ~125 ms, with a few MiB of overhead, and exposes a small REST API.
  Its tiny codebase = tiny attack surface, which is the whole point for multi-tenant
  serverless.
- **Cloud Hypervisor** — a similar modern Rust VMM, slightly more featureful (some hotplug,
  more devices), aimed at cloud workloads.

They share KVM with QEMU (same kernel acceleration) but trade QEMU's vast device model for
speed, density, and security.

### Why a tiny device model = millisecond boots

Most of a normal VM's boot time goes into **firmware initialization** (BIOS/UEFI probing
hardware), **PCI enumeration**, **ACPI**, and bringing up many devices. A microVM removes
all of that: there's no firmware (the kernel is loaded and jumped to directly), no PCI to
enumerate, and only a couple of simple virtio-MMIO devices to initialize. With almost nothing
to set up, the kernel reaches userspace in tens of milliseconds. The smaller device model
also means **less memory overhead per VM** (fewer device structures) and a **smaller attack
surface** (fewer emulated parsers, Lesson 57) — enabling thousands of microVMs per host.

### Trade-offs vs full QEMU

- **No device passthrough** (no PCI → no VFIO), limited device variety, often no graphics.
- **No (or limited) ACPI/firmware features**, fewer machine-management niceties.
- Best for **short-lived, homogeneous, programmatic** workloads (functions, sandboxes), not
  general-purpose desktops/servers needing rich hardware.

{: .note }
> **Two things Firecracker removes to boot in tens of milliseconds**
> Firecracker drops (1) the <strong>firmware/BIOS</strong> stage — instead of SeaBIOS/OVMF
> probing hardware, it loads and jumps straight into the guest kernel — and (2) the
> <strong>PCI bus and most emulated devices</strong> — it uses a tiny fixed set of
> virtio(-MMIO) devices with no PCI enumeration or ACPI. Removing firmware init and PCI
> enumeration eliminates the bulk of normal boot time, so the kernel reaches userspace in
> ~tens of milliseconds, while the minimal device set also cuts memory overhead and attack
> surface — exactly what serverless/multi-tenant needs.

---

## Lab

```bash
# 1. Boot a guest with QEMU's microvm machine type — kernel booted DIRECTLY (no BIOS).
#    You need a kernel image (vmlinux) and a rootfs (initrd or virtio-blk image).
$ time qemu-system-x86_64 -M microvm,acpi=off -accel kvm -m 512 -smp 1 \
    -kernel ./vmlinux \
    -append "console=ttyS0 root=/dev/vda rw" \
    -drive id=root,file=rootfs.ext4,format=raw,if=none \
    -device virtio-blk-device,drive=root \
    -netdev user,id=n0 -device virtio-net-device,netdev=n0 \
    -nodefaults -no-reboot -serial stdio
# Note: virtio-blk-DEVICE / virtio-net-DEVICE = the MMIO variants (no PCI).

# 2. Compare boot time against a full q35 VM of the same image:
$ time qemu-system-x86_64 -M q35 -accel kvm -m 512 \
    -drive file=disk.qcow2,if=virtio -nographic -no-reboot
# The microvm reaches userspace dramatically faster (no firmware/PCI/ACPI).

# 3. Firecracker (if installed) — config-driven, REST API, microVM in ~125 ms:
$ cat > vm.json <<'EOF'
{ "boot-source": { "kernel_image_path": "vmlinux",
                   "boot_args": "console=ttyS0 reboot=k panic=1 root=/dev/vda" },
  "drives": [ { "drive_id":"rootfs","path_on_host":"rootfs.ext4",
                "is_root_device":true,"is_read_only":false } ],
  "machine-config": { "vcpu_count":1,"mem_size_mib":128 } }
EOF
$ time firecracker --no-api --config-file vm.json
# Boots in tens of milliseconds; tiny memory footprint.

# 4. Observe the difference in device count / overhead:
#    microvm/Firecracker: only virtio-blk, virtio-net, serial, (vsock). No PCI tree.
#    Full VM: VGA, USB, ACPI, PCI bridges, RTC, etc.
```

**Expected result:** The microVM (QEMU `microvm` or Firecracker) boots to userspace in a
fraction of the time of a full q35 VM and with far less memory overhead, because it skips
firmware, PCI enumeration, and ACPI and presents only a couple of virtio-MMIO devices.

---

## Further Reading

| Topic | Link |
|---|---|
| QEMU microvm machine | [qemu.org — microvm](https://www.qemu.org/docs/master/system/i386/microvm.html) |
| Firecracker | [Wikipedia — Firecracker (software)](https://en.wikipedia.org/wiki/Firecracker_(software)) |
| Firecracker project | [firecracker-microvm.github.io](https://firecracker-microvm.github.io/) |
| Cloud Hypervisor | [cloudhypervisor.org](https://www.cloudhypervisor.org/) |
| virtio-mmio (Lesson 27) | [The virtio standard]({{ '/virtualization/lessons/lesson-27-virtio-standard.html' | relative_url }}) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Firecracker boots in tens of milliseconds. Name two things it removes from the classic QEMU machine to get there.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) The firmware/BIOS stage — instead of SeaBIOS/OVMF probing hardware, it loads and jumps directly into the guest kernel. (2) The PCI bus and most emulated devices — it uses a tiny fixed set of virtio-MMIO devices (virtio-net, virtio-blk, serial, vsock) with no PCI enumeration and no/minimal ACPI. Removing firmware initialization and PCI enumeration eliminates most of normal boot time, so the kernel reaches userspace in tens of milliseconds; the minimal device set also reduces memory overhead and attack surface.
</details>

---

**Q2. Why does a smaller device model lead to both faster boots and lower memory overhead?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Faster boots: most of a normal VM's boot time is firmware init, PCI enumeration, ACPI parsing, and initializing many devices. With only a couple of simple virtio-MMIO devices and no firmware/PCI/ACPI, there's almost nothing to set up, so the kernel reaches userspace quickly. Lower memory overhead: each emulated device and the firmware/PCI/ACPI machinery consume host memory for their data structures and state; fewer devices means fewer such structures per VM, so each microVM's baseline footprint is just a few MiB. Together this lets a host pack thousands of microVMs.
</details>

---

**Q3. What capability do microVMs/Firecracker give up that a full q35 QEMU VM has, and why?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
They give up device passthrough (VFIO) and rich device variety — and often graphics and many machine-management features. The main reason is the absence of a PCI bus: VFIO passthrough assigns PCI devices, so with no PCI there's no passthrough; and the deliberately minimal device model omits the broad set of emulated/PCI devices a full q35 machine offers. This is an intentional trade: dropping those features is what enables the millisecond boots, tiny footprint, and small attack surface that serverless/sandbox workloads need, at the cost of being unsuitable for general-purpose VMs that require diverse or passed-through hardware.
</details>

---

## Homework

Boot the same kernel/rootfs once with QEMU `-M microvm` and once with `-M q35`, and `time` each to userspace (login prompt). Record both times and the rough memory each VM uses. Then explain which removed components account for most of the boot-time difference, and name one workload for which the microVM's trade-offs are ideal and one for which they're unacceptable.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The microvm boot is dramatically faster (often sub-100 ms to userspace vs seconds for q35) and uses noticeably less memory. Most of the difference comes from the components microvm removes: firmware/BIOS initialization (no SeaBIOS/OVMF hardware probing), PCI bus enumeration, and ACPI parsing/device bring-up — these dominate a normal VM's early boot, and microvm skips them by booting the kernel directly with only a few virtio-MMIO devices. Ideal workload: serverless functions / ephemeral sandboxes (e.g. AWS Lambda) where you launch and tear down many short-lived, homogeneous VMs and value fast boot, density, and a small attack surface. Unacceptable workload: a general-purpose desktop or a VM needing GPU passthrough — no PCI means no VFIO passthrough and no graphics, so the microVM's stripped-down model can't support it; you'd use a full q35 VM there.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 60 — VM-Isolated Containers and the VM-vs-Container Question →](lesson-60-kata-containers){: .btn .btn-primary }
