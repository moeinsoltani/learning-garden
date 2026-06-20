---
title: "Lesson 14 — Firmware and the Boot Path"
nav_order: 14
parent: "Phase 4: QEMU Fundamentals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 14: Firmware and the Boot Path

## Concept

When a real PC powers on, the CPU doesn't jump straight to your OS — it runs
**firmware** first (BIOS or UEFI), which initializes the machine and then hands off
to a **bootloader**, which loads the **kernel**. A VM is no different: QEMU must
supply firmware too.

```
   power on
      │
      ▼
   FIRMWARE          ← SeaBIOS (legacy BIOS)  OR  OVMF (UEFI)
      │                initializes virtual HW, finds a boot device
      ▼
   BOOTLOADER        ← GRUB / systemd-boot / Windows Boot Manager
      │                loaded from the disk's boot sector / EFI partition
      ▼
   GUEST KERNEL      ← Linux / Windows starts
```

The choice of firmware — **SeaBIOS vs OVMF** — determines whether your guest boots
in legacy BIOS mode or modern UEFI mode, and UEFI brings a wrinkle: it needs
**writable NVRAM** to remember boot entries and Secure Boot keys.

---

## How It Works

### SeaBIOS — the legacy default

By default QEMU x86 machines boot with **SeaBIOS**, an open-source legacy BIOS. It
emulates the old PC firmware interface: it runs, finds a bootable disk (MBR boot
sector), and jumps to it. Simple, fast, universally compatible — but it's *legacy
BIOS*, so it can't do UEFI things (GPT-only boot on some OSes, Secure Boot,
large-disk boot).

### OVMF — UEFI for VMs

**OVMF** (Open Virtual Machine Firmware) is the UEFI firmware for QEMU, built from
the same **TianoCore EDK II** codebase as real UEFI. It gives the guest a modern
UEFI environment: GPT booting, the EFI System Partition, UEFI boot menus, and
**Secure Boot**.

OVMF ships as **two files**, and understanding the split is the whole point:

- **`OVMF_CODE.fd`** — the firmware *code*, read-only, shared by all VMs.
- **`OVMF_VARS.fd`** — the **NVRAM varstore**, *writable, per-VM*. This holds UEFI
  variables: boot entries, boot order, and Secure Boot keys.

You attach them as two **pflash** drives:

```
   -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE.fd \
   -drive if=pflash,format=raw,file=my-vm_VARS.fd        ← a PER-VM writable copy
```

You copy the template `OVMF_VARS.fd` once per VM so each VM has its *own* writable
NVRAM. Without a writable varstore, the guest can't persist boot entries — every
boot the firmware forgets where the OS is, and Secure Boot key enrollment can't be
saved.

### Secure Boot in a VM

UEFI **Secure Boot** verifies that the bootloader/kernel are signed by trusted
keys. In a VM, you use an OVMF build with Secure Boot support plus a varstore
pre-enrolled with Microsoft/distro keys (`OVMF_VARS.secboot.fd`). It's required to
boot, e.g., Windows 11 guests properly.

### Boot order and PXE

`-boot` controls boot device order: `-boot d` (CD first), `-boot c` (disk),
`-boot n` (network/PXE), `-boot menu=on` (interactive menu), `-boot order=dc`.
**PXE** boot loads an OS image over the network from a TFTP/DHCP server — common
for diskless provisioning; QEMU's NICs include a PXE option ROM.

{: .note }
> **Why a UEFI guest needs a writable NVRAM/varstore**
> UEFI stores persistent state — boot entries (which EFI binary to launch), boot
> order, and Secure Boot key databases — in NVRAM variables. On real hardware that's
> a flash chip on the motherboard. In a VM, OVMF_VARS.fd *is* that flash chip. The
> code (OVMF_CODE.fd) is read-only and shared, but each VM must have its own writable
> VARS copy or it cannot remember how to boot itself (and Secure Boot enrollment
> would be lost on every reboot).

---

## Lab

```bash
# Install OVMF (Debian/Ubuntu: sudo apt install ovmf). Files land under /usr/share.
$ ls /usr/share/OVMF/
OVMF_CODE.fd  OVMF_CODE.secboot.fd  OVMF_VARS.fd  OVMF_VARS.secboot.fd

# 1. Boot the disk with the LEGACY BIOS (SeaBIOS) — the default, nothing special:
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -drive file=disk.qcow2,if=virtio -nographic
# Watch for the SeaBIOS banner at the top.

# 2. Make a PER-VM writable copy of the UEFI varstore:
$ cp /usr/share/OVMF/OVMF_VARS.fd ./my-vm_VARS.fd

# 3. Boot the SAME disk with UEFI (OVMF): note the two pflash drives.
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE.fd \
    -drive if=pflash,format=raw,file=./my-vm_VARS.fd \
    -drive file=disk.qcow2,if=virtio -nographic
# Now you'll see the TianoCore/UEFI banner instead of SeaBIOS.

# 4. Boot from CD first (e.g. to run an installer), then disk:
$ qemu-system-x86_64 -accel kvm -m 2G -cdrom installer.iso -boot d \
    -drive file=disk.qcow2,if=virtio -nographic

# 5. Prove the varstore is writable: change UEFI boot order in the firmware menu,
#    quit, reboot — the change persists because it was saved into my-vm_VARS.fd.
$ ls -l my-vm_VARS.fd   # mtime updates after the guest writes UEFI vars
```

**Expected result:** The same disk shows a SeaBIOS banner in legacy mode and a
TianoCore/UEFI banner with OVMF. The per-VM `OVMF_VARS.fd` copy is writable and its
contents persist UEFI settings across reboots.

---

## Further Reading

| Topic | Link |
|---|---|
| UEFI | [Wikipedia — UEFI](https://en.wikipedia.org/wiki/UEFI) |
| BIOS | [Wikipedia — BIOS](https://en.wikipedia.org/wiki/BIOS) |
| OVMF / TianoCore | [tianocore — OVMF](https://github.com/tianocore/tianocore.github.io/wiki/OVMF) |
| SeaBIOS | [seabios.org](https://www.seabios.org/SeaBIOS) |
| Secure Boot | [Wikipedia — UEFI Secure Boot](https://en.wikipedia.org/wiki/UEFI#Secure_Boot) |
| Preboot Execution Environment (PXE) | [Wikipedia — PXE](https://en.wikipedia.org/wiki/Preboot_Execution_Environment) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What is OVMF, and why does a UEFI guest need a writable NVRAM/varstore file?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
OVMF (Open Virtual Machine Firmware) is the UEFI firmware for QEMU VMs, built from TianoCore EDK II — it gives a guest a modern UEFI environment (GPT boot, EFI System Partition, Secure Boot). A UEFI guest needs a writable NVRAM/varstore (OVMF_VARS.fd) because UEFI stores persistent state — boot entries, boot order, Secure Boot key databases — in NVRAM variables, which on real hardware is a motherboard flash chip. The firmware code (OVMF_CODE.fd) is read-only and shared; each VM needs its own writable VARS copy, or it can't remember how to boot and loses Secure Boot enrollment on every reboot.
</details>

---

**Q2. Describe the boot sequence of a VM from power-on to the guest kernel running.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
On power-on the CPU runs firmware (SeaBIOS for legacy BIOS, or OVMF for UEFI), which initializes the virtual hardware and locates a boot device. The firmware loads and runs a bootloader (GRUB, systemd-boot, Windows Boot Manager) from the disk's boot sector (MBR) or EFI System Partition (UEFI). The bootloader then loads and starts the guest kernel (Linux/Windows). For PXE boot, the firmware instead pulls the image over the network via DHCP/TFTP before the bootloader stage.
</details>

---

**Q3. Why do you copy `OVMF_VARS.fd` once per VM instead of pointing every VM at the shared template?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because OVMF_VARS.fd is writable per-VM state: each VM stores its own boot entries, boot order, and Secure Boot keys there. If multiple VMs shared one varstore they'd overwrite each other's UEFI settings and Secure Boot enrollment, and the shared template would no longer be a clean baseline. So you copy the template to a per-VM file (e.g. my-vm_VARS.fd) and attach that writable copy, while all VMs share the read-only OVMF_CODE.fd.
</details>

---

## Homework

Create two separate writable varstores (`vm1_VARS.fd` and `vm2_VARS.fd`) from the OVMF template. Boot each VM once with OVMF, change a UEFI setting (e.g. boot order) differently in each, then reboot both. Confirm the settings persist independently. Explain what would have gone wrong if both VMs had shared a single `OVMF_VARS.fd`.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With separate varstores, each VM persists its own UEFI boot order/settings independently across reboots, because each writes to its own NVRAM file. If both shared a single OVMF_VARS.fd, they'd race to write the same NVRAM: each VM's boot entries and Secure Boot keys would clobber the other's, settings would appear to randomly change, and concurrent writes could corrupt the varstore — leaving one or both VMs unable to find their boot entry. Per-VM varstores isolate this mutable firmware state, exactly like each physical machine having its own NVRAM chip.
</details>
