---
title: "Lesson 37 — GPU Passthrough"
nav_order: 37
parent: "Phase 9: Device Passthrough"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 37: GPU Passthrough

## Concept

GPU passthrough — giving a VM a real graphics card for gaming, CAD, or ML — is the
**most popular and most painful** application of VFIO (Lesson 36). The mechanism is the
same (bind to vfio-pci, assign to the guest), but GPUs pile on obstacles that NICs don't:
grouped functions, driver hostility to virtualization, broken resets, and the fact that a
single-GPU host needs *its* display too.

```
   a "GPU" is really several functions in ONE IOMMU group:
   ┌──────── IOMMU group 1 ────────┐
   │ 01:00.0  VGA  (the GPU)        │  ← all of these must go to the guest
   │ 01:00.1  HDMI/DP Audio        │
   │ 01:00.2  USB-C controller     │  (on some cards)
   │ 01:00.3  UCSI serial          │
   └────────────────────────────────┘
```

---

## How It Works

### The GPU + audio (and friends) grouping

A GPU is not one PCI function — it's a VGA function plus an **HDMI/DisplayPort audio**
function (and sometimes USB-C/UCSI). They sit in the **same IOMMU group**, so by the
Lesson 35 rule you must pass **all** of them to the guest. Forget the audio function and
you'll get errors. The full set is what makes a working GPU in the guest.

### NVIDIA "Code 43"

For years NVIDIA's **consumer** GPU drivers deliberately **refused to run in a VM**: if
the driver detected a hypervisor, it failed with **"Code 43"** in the guest. The
workaround was to **hide the hypervisor** from the guest so the driver couldn't tell it
was virtualized:

```
   -cpu host,kvm=off,hv_vendor_id=whatever       # mask KVM/hypervisor signatures
   # plus Hyper-V enlightenments to look like a "normal" Windows machine
```

NVIDIA has since **officially enabled** consumer-GPU passthrough in newer drivers, so
Code 43 is largely historical — but you'll still encounter the hypervisor-hiding tricks
in older guides, and they illustrate the general theme: GPUs fight virtualization.

### ROM / vBIOS issues

A GPU needs its **video BIOS (vBIOS)** to initialize. In a VM, the card's ROM may not be
shadowed/readable the way it is at host boot, especially for the *primary* GPU (whose ROM
the host already consumed). The fix is often to supply a **dumped vBIOS ROM file** to
QEMU (`-device vfio-pci,...,romfile=gpu.rom`), particularly for the boot GPU. Getting a
correct, clean ROM dump is a common stumbling block.

### Single-GPU vs dual-GPU hosts

- **Dual-GPU host:** one GPU for the host (display/console), one bound to vfio-pci for the
  guest. The clean setup — the host keeps working while the guest has its dedicated card.
- **Single-GPU host:** you only have one GPU, so passing it to the guest means **the host
  loses its display**. This requires "single-GPU passthrough" scripts that, on VM start,
  unbind the GPU from the host (kill the display manager, unload host GPU drivers, bind to
  vfio-pci) and reverse it on VM stop. Fragile, and reset bugs (below) bite hardest here.

### The reset bug (again)

GPUs are the worst offenders of the **reset bug class** (Lesson 36): many consumer cards
can't be cleanly reset after the VM stops, so starting the VM a second time fails or the
host hangs — often needing a reboot. Projects like **vendor-reset** add reset quirks for
specific AMD cards. This is why GPU passthrough is "harder than NIC passthrough."

### Looking Glass and VFIO workstations

For single-GPU-feel without two monitors, **Looking Glass** copies the guest GPU's
framebuffer into host memory (via a shared IVSHMEM region) so you view the passed-through
GPU's output in a *host* window with near-zero latency — popular for VFIO gaming
workstations.

{: .note }
> **Why GPU passthrough is harder than NIC passthrough**
> A NIC is usually alone in its IOMMU group, resets cleanly via FLR, has no driver
> hostility to VMs, and the host doesn't need it to show a desktop. A GPU is the opposite:
> it shares a group with audio/USB functions (all must be passed), historically had
> drivers that refused to run in a VM (Code 43), often needs a vBIOS ROM file to
> initialize, frequently can't reset cleanly (host hangs on second VM start), and on a
> single-GPU host conflicts with the host's own need for a display. Each is a separate
> obstacle stacked on the same VFIO mechanism.

---

## Lab

```bash
# 1. Identify the GPU and its FULL IOMMU group (all functions must be passed):
$ lspci -nn | grep -iE 'vga|audio|nvidia|amd/ati' | head
01:00.0 VGA compatible controller [NVIDIA GeForce ...] [10de:2484]
01:00.1 Audio device [NVIDIA HDMI Audio] [10de:228b]
$ ls /sys/bus/pci/devices/0000:01:00.0/iommu_group/devices/
0000:01:00.0  0000:01:00.1        ← BOTH go to the guest

# 2. Bind ALL group functions to vfio-pci (claim early via cmdline for GPUs):
#   kernel cmdline: vfio-pci.ids=10de:2484,10de:228b   (plus blacklist nouveau/nvidia)
$ sudo driverctl set-override 0000:01:00.0 vfio-pci
$ sudo driverctl set-override 0000:01:00.1 vfio-pci
$ lspci -nnk -s 01:00.0 | grep 'in use'
        Kernel driver in use: vfio-pci

# 3. Pass the GPU + its audio to a (UEFI/OVMF) guest; supply a vBIOS if needed:
$ cp /usr/share/OVMF/OVMF_VARS.fd ./vm_VARS.fd
$ sudo qemu-system-x86_64 -accel kvm -m 8G -smp 6 -cpu host \
    -machine q35 \
    -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE.fd \
    -drive if=pflash,format=raw,file=./vm_VARS.fd \
    -device vfio-pci,host=0000:01:00.0,multifunction=on,romfile=gpu.rom \
    -device vfio-pci,host=0000:01:00.1 \
    -drive file=win.qcow2,if=virtio -nographic &
#   In a Windows guest, Device Manager shows the real GPU; install vendor drivers.

# 4. (Historical) hide the hypervisor to dodge NVIDIA Code 43 on old drivers:
#    -cpu host,kvm=off,hv_vendor_id=null  (+ hv_* enlightenments)

# 5. Observe the reset behavior: stop the VM, then try to start it again.
$ sudo kill %1
$ dmesg | tail   # watch for reset failures / "Unable to reset device" on bad cards
```

**Expected result:** All functions of the GPU's IOMMU group bind to vfio-pci; passing
the VGA + audio functions (with a vBIOS ROM if required) makes the real GPU appear in the
guest. On cards with reset problems you'll observe failures when restarting the VM — the
hallmark difficulty of GPU passthrough.

---

## Further Reading

| Topic | Link |
|---|---|
| PCI passthrough via OVMF (GPU) | [Arch Wiki — PCI passthrough via OVMF](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF) |
| NVIDIA Code 43 / VM detection | [Arch Wiki — passthrough (Code 43)](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#%22Error_43:_Driver_failed_to_load%22_on_Nvidia_GPUs) |
| vendor-reset | [github.com/gnif/vendor-reset](https://github.com/gnif/vendor-reset) |
| Looking Glass | [looking-glass.io](https://looking-glass.io/) |
| VFIO | [kernel.org — VFIO](https://docs.kernel.org/driver-api/vfio.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why is GPU passthrough notably harder than NIC passthrough? Name two GPU-specific obstacles.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A NIC is typically alone in its IOMMU group, resets cleanly, and the host doesn't need it for a desktop. GPUs add obstacles such as: (1) the GPU shares an IOMMU group with its HDMI/DP audio (and sometimes USB-C) functions, so all must be passed together; (2) historically NVIDIA consumer drivers refused to run in a VM (Code 43), needing the hypervisor to be hidden; (3) the card often needs a vBIOS ROM file to initialize, especially the primary GPU; (4) many GPUs can't reset cleanly after the VM stops (reset bug), hanging the host on a second start; and (5) on a single-GPU host, passing it through means the host loses its own display. Any two of these illustrate why it's harder.
</details>

---

**Q2. What was NVIDIA "Code 43," and how was it worked around?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Code 43 was an error NVIDIA's consumer GPU drivers raised inside a guest when they detected they were running under a hypervisor — the driver deliberately refused to initialize in a VM. The workaround was to hide the hypervisor from the guest so the driver couldn't tell it was virtualized: e.g. <code>-cpu host,kvm=off,hv_vendor_id=...</code> to mask KVM/hypervisor signatures (often combined with Hyper-V enlightenments to look like a normal Windows machine). NVIDIA has since officially enabled consumer passthrough in newer drivers, so it's largely historical now.
</details>

---

**Q3. On a single-GPU host, what extra complication does GPU passthrough create, and how is it handled?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With only one GPU, passing it to the guest means the host loses its display, since the same card can't serve both. This is handled with "single-GPU passthrough" scripts (libvirt hooks) that, on VM start, tear down the host's use of the GPU — stop the display manager, unbind the host GPU driver, unload modules, bind the card (and its group functions) to vfio-pci — then reverse all of it on VM stop to give the host its display back. It's fragile and especially vulnerable to reset bugs, since the GPU must cleanly return to the host afterward.
</details>

---

## Homework

Identify your GPU's complete IOMMU group with `ls /sys/bus/pci/devices/<gpu-addr>/iommu_group/devices/`. List every function it contains and state which you'd need to pass to a guest. Then explain whether your host is a single-GPU or dual-GPU setup and what that means for whether you could realistically do GPU passthrough without losing your host display.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The group listing typically shows the VGA function plus an HDMI/DP audio function (and possibly USB-C/UCSI functions) — all of which must be passed to the guest together, because the IOMMU can't split a group. If your host has a second GPU (e.g. integrated graphics + a discrete card), it's a dual-GPU setup: you can bind the discrete GPU's whole group to vfio-pci and pass it to a guest while the host keeps its display on the other GPU — the clean case. If you have only one GPU, it's single-GPU: passing it through means the host loses its display, requiring fragile start/stop scripts to detach and reattach the GPU around the VM's lifetime, so realistic passthrough without losing the host display isn't possible without those hooks (or a second GPU).
</details>
