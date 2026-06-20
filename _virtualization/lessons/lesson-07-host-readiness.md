---
title: "Lesson 07 — Checking and Enabling Virtualization on the Host"
nav_order: 7
parent: "Phase 2: CPU & Hardware Virtualization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 07: Checking and Enabling Virtualization on the Host

## Concept

You now know *why* KVM needs VT-x/AMD-V (Lesson 5) and EPT/NPT (Lesson 6). This
lesson is the hands-on checklist that turns those facts into a working host. There
are four gates a VM must pass through, top to bottom:

```
   ┌─────────────────────────────────────────────┐
   │ 1. CPU supports vmx/svm        (silicon)     │  grep /proc/cpuinfo
   ├─────────────────────────────────────────────┤
   │ 2. Enabled in BIOS/UEFI        (firmware)    │  reboot → setup
   ├─────────────────────────────────────────────┤
   │ 3. KVM modules loaded          (kernel)      │  lsmod | grep kvm
   ├─────────────────────────────────────────────┤
   │ 4. /dev/kvm permissions        (user)        │  kvm group
   └─────────────────────────────────────────────┘
           all four → fast KVM ;  any fail → slow TCG
```

If *any* gate fails, QEMU silently falls back to **TCG** software emulation and
your VM crawls. The skill here is diagnosing *which* gate failed.

---

## How It Works

### Gate 1 — CPU support

The `vmx` (Intel) or `svm` (AMD) flag in `/proc/cpuinfo` means the silicon can do
hardware virtualization. No flag at all (on bare metal) means an old/disabled CPU;
no flag *inside a VM* usually means nested virtualization isn't enabled on the
outer host (Lesson 55).

### Gate 2 — BIOS/UEFI

Even a capable CPU ships with virtualization **disabled** in firmware on many
machines. You enable it in BIOS/UEFI setup — look for "Intel VT-x" /
"Virtualization Technology" / "SVM Mode" / "AMD-V". For *passthrough* (Phase 9)
you also enable **VT-d / AMD-Vi** (the IOMMU). After toggling, a full power cycle
is sometimes required for the flag to appear.

### Gate 3 — Kernel modules

The kernel needs `kvm` plus the vendor module loaded:

- `kvm` — architecture-independent core
- `kvm_intel` or `kvm_amd` — vendor-specific VMX/SVM glue

These usually autoload. If not, `modprobe kvm_intel` (or `kvm_amd`). If `modprobe`
fails with "Operation not supported," that almost always means Gate 2 (BIOS) is
off.

### Gate 4 — Permissions

`/dev/kvm` is owned by the `kvm` group. A normal user must be a member to use KVM
without root:

```
$ sudo usermod -aG kvm $USER   # then log out/in
```

libvirt later runs QEMU as a dedicated `qemu`/`libvirt-qemu` user that is already
in the right group.

### The one-shot check: `virt-host-validate`

`virt-host-validate` (from libvirt) runs *all* the relevant checks at once and
prints PASS/WARN/FAIL with hints. It's the fastest way to audit a host.

{: .note }
> **Why a working /dev/kvm can still run slow (TCG)**
> `/dev/kvm` existing only means the device node is there. A VM can still fall back
> to TCG if: (a) QEMU wasn't told to use it (missing `-accel kvm` / `-enable-kvm`),
> (b) your user lacks permission to open `/dev/kvm`, or (c) the modules are loaded
> but the *vendor* module failed to enable VMX (BIOS off), leaving only the stub.
> Always pass `-accel kvm` and check `kvm_stat` shows activity to confirm.

---

## Lab

```bash
# GATE 1 — CPU silicon. Count how many cores report the flag (nonzero = good):
$ grep -c -E 'vmx|svm' /proc/cpuinfo
8

# Convenience tool (Debian/Ubuntu: sudo apt install cpu-checker):
$ kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used

# GATE 3 — modules loaded?
$ lsmod | grep kvm
kvm_intel             380928  0
kvm                  1142784  1 kvm_intel

# If missing, load it (failure here usually means BIOS/Gate 2 is off):
$ sudo modprobe kvm_intel    # or kvm_amd

# GATE 4 — permissions. Are you in the kvm group?
$ id -nG | tr ' ' '\n' | grep -x kvm || echo "NOT in kvm group"
kvm

# ONE-SHOT — the full readiness audit (sudo apt install libvirt-clients):
$ virt-host-validate qemu
  QEMU: Checking for hardware virtualization                       : PASS
  QEMU: Checking if device /dev/kvm exists                         : PASS
  QEMU: Checking if device /dev/kvm is accessible                  : PASS
  QEMU: Checking for device assignment IOMMU support               : PASS
  QEMU: Checking if IOMMU is enabled by kernel                     : WARN (IOMMU appears to be disabled in kernel. Add intel_iommu=on)
  ...

# Fix a typical FAIL/WARN: enable the IOMMU on the kernel cmdline (Phase 9),
# e.g. edit /etc/default/grub: GRUB_CMDLINE_LINUX_DEFAULT="... intel_iommu=on"
# then: sudo update-grub && reboot
```

**Expected result:** Gate 1 reports a nonzero count, modules are loaded, you're in
the `kvm` group, and `virt-host-validate` shows PASS for the core virtualization
checks. Any FAIL maps directly to one of the four gates.

---

## Further Reading

| Topic | Link |
|---|---|
| `virt-host-validate` | [libvirt.org — virt-host-validate](https://libvirt.org/manpages/virt-host-validate.html) |
| KVM how-to (enabling) | [linux-kvm.org — FAQ](https://www.linux-kvm.org/page/FAQ) |
| `modprobe` | [man7.org — modprobe(8)](https://man7.org/linux/man-pages/man8/modprobe.8.html) |
| `/proc/cpuinfo` | [man7.org — proc(5)](https://man7.org/linux/man-pages/man5/proc.5.html) |
| `usermod` (group membership) | [man7.org — usermod(8)](https://man7.org/linux/man-pages/man8/usermod.8.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. `/dev/kvm` exists but your VM falls back to slow TCG emulation. List three things you'd check.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) Did you actually tell QEMU to use it? Confirm `-accel kvm` (or `-enable-kvm`) is on the command line / the libvirt domain type is "kvm" not "qemu". (2) Permissions: can your user open /dev/kvm — are you in the kvm group? (3) Is the vendor module really enabling VMX/SVM, or is virtualization disabled in BIOS/UEFI (modprobe may load a stub but VMX is off)? Also check that, if the host is itself a VM, nested virtualization is enabled so vmx/svm is exposed.
</details>

---

**Q2. What are the four "gates" a host must pass for hardware-accelerated KVM, from hardware to user?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) CPU silicon supports vmx/svm; (2) virtualization is enabled in BIOS/UEFI firmware (and VT-d/AMD-Vi too if you want passthrough); (3) the kernel has the kvm core plus the vendor module (kvm_intel/kvm_amd) loaded; (4) the user has permission to open /dev/kvm, i.e. is in the kvm group (or runs as the libvirt qemu user). Failing any one drops you to TCG.
</details>

---

**Q3. `sudo modprobe kvm_intel` fails with "Operation not supported." What is the most likely cause, and where do you fix it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The most likely cause is that Intel VT-x is disabled in the BIOS/UEFI firmware (Gate 2) — the CPU has the feature but the firmware hasn't enabled VMX, so the module can't initialize it. The fix is to reboot into BIOS/UEFI setup, enable "Intel Virtualization Technology / VT-x" (and VT-d if you'll do passthrough), save, and power-cycle. After that, modprobe succeeds and the vmx flag appears.
</details>

---

## Homework

Run `virt-host-validate qemu` on your host and read every line. Pick one WARN or FAIL it reports, explain in one or two sentences what that check verifies, and what concrete change fixes it. (If everything PASSes, explain what the "IOMMU is enabled by kernel" check verifies and which kernel cmdline parameter satisfies it.)

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Example: "Checking if IOMMU is enabled by kernel : WARN" verifies that the kernel was booted with the IOMMU active, which is required for safe VFIO device passthrough (Phase 9). The fix is to add <code>intel_iommu=on</code> (or <code>amd_iommu=on</code>) to GRUB_CMDLINE_LINUX_DEFAULT in /etc/default/grub, run <code>sudo update-grub</code>, and reboot. The check then passes because the IOMMU is now initialized at boot. (Other common ones: hardware virtualization FAIL → enable VT-x/AMD-V in BIOS; /dev/kvm not accessible → add your user to the kvm group.)
</details>
