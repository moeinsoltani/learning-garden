---
title: "Lesson 50 — From Firmware to Kernel"
nav_order: 1
parent: "Phase 10: Boot & Init"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 50: From Firmware to Kernel

## Concept

Everything in this course has assumed the kernel is *running*. This phase
asks: how did it get there? The boot chain is a relay race where each runner
knows just enough to find and start the next:

```
  power on
     │
  FIRMWARE (UEFI, or legacy BIOS)   ← in ROM on the motherboard
     │  initializes CPU/RAM, finds a bootable device,
     │  reads the EFI System Partition
     ▼
  BOOTLOADER (GRUB / systemd-boot)  ← a small program on disk
     │  shows a menu, knows filesystems, loads two files into RAM:
     ▼         the KERNEL and the INITRAMFS (Lesson 51)
  KERNEL       ← decompresses itself, initializes, mounts root
     │  runs everything you've studied, then starts...
     ▼
  PID 1 (systemd)  ← first userspace process (Lessons 06/52)
     │
  your login prompt
```

Each handoff is a trust and capability boundary: firmware knows hardware but
not filesystems; the bootloader knows filesystems but not your OS; the kernel
knows everything but needs to be *told where root is* — via the **kernel
command line**, the string you've read from `/proc/cmdline` since Lesson 04
and set parameters in throughout (virt Lessons 07/09's `kvm.` params,
Lesson 15's `psi=`, Lesson 23's `hugepagesz=`). Now you meet where it comes
from and how to change it.

For a self-study lab this is normally unobservable — it happens before
anything you can run. The trick this lesson uses: **watch a whole boot inside
QEMU** (virtualization Lesson 12/14), where the firmware, kernel, and serial
console are all yours to inspect.

---

## How It Works

### UEFI vs legacy BIOS

Old **BIOS**: firmware reads the first 512 bytes of the disk (the MBR boot
sector), runs that tiny stub, which chain-loads the real bootloader — a
cramped, filesystem-blind design. Modern **UEFI**: firmware understands
FAT (the EFI System Partition, `/boot/efi`) and runs `.efi` executables
directly — bootloaders are just files (`grubx64.efi`,
`systemd-bootx64.efi`), boot entries live in NVRAM (`efibootmgr`), and
**Secure Boot** can require signatures on what runs (the chain of trust that
extends to virt Lesson 58's confidential computing). `ls /sys/firmware/efi`
existing = you booted UEFI.

### The bootloader's job

Present boot options, then load the kernel image (`vmlinuz` — a
self-decompressing bzImage) and the initramfs into memory, assemble the
kernel command line, and jump to the kernel's entry point. GRUB reads its
config (`/boot/grub/grub.cfg`, generated — never hand-edit; change
`/etc/default/grub` + `update-grub`); systemd-boot reads simple text entries.
The command line is where root=, quiet, and every `param=value` you've set
gets assembled and handed forward.

### The kernel's early life

The compressed kernel decompresses itself, sets up the very things this
course studied — page tables (Lesson 17), the scheduler (Lesson 13),
interrupts (Lesson 05) — detects hardware, mounts the **initramfs** as a
temporary root (Lesson 51), and eventually mounts the *real* root filesystem
and executes `/sbin/init` (→ systemd) as PID 1. Early messages go to the
kernel ring buffer (`dmesg` — the `printk` output) and, if `console=ttyS0`
is set, out a serial port — which is exactly how the lab watches it.

{: .note }
> **Editing the command line — the recovery skill**
> At the GRUB menu, press <code>e</code> to edit the selected entry's kernel
> line for one boot: add <code>single</code> or <code>init=/bin/bash</code>
> to get a root shell (password recovery), <code>nomodeset</code> for broken
> graphics, or remove <code>quiet splash</code> to watch the boot. Permanent
> changes go through <code>/etc/default/grub</code>. This is the "my machine
> won't boot" escape hatch — and it's why physical/console access is
> effectively root (Lesson 11's theme, at the boot layer).

---

## Lab

```bash
# ---- 1. Reconstruct YOUR boot from the evidence ----
$ cat /proc/cmdline
# BOOT_IMAGE=/vmlinuz-6.8.0-x root=UUID=... ro quiet splash
#   root= told the kernel where to mount /; ro = mount read-only first
$ ls /sys/firmware/efi >/dev/null 2>&1 && echo "UEFI boot" || echo "legacy BIOS boot"
$ [ -d /sys/firmware/efi ] && efibootmgr 2>/dev/null | head -5   # NVRAM entries

# ---- 2. The bootloader's files ----
$ ls /boot/
# vmlinuz-*  initrd.img-*  grub/ ...   ← the two files the loader loads, on disk
$ ls -la /boot/vmlinuz* | tail -1       # the kernel image itself
$ file /boot/vmlinuz-$(uname -r) 2>/dev/null || echo "(compressed kernel image)"

# ---- 3. The kernel's own boot narration ----
$ sudo dmesg | head -15
# [0.000000] Linux version 6.8.0 ... ← the very first printk
# [0.000000] Command line: BOOT_IMAGE=...  ← the kernel echoing what it got
# [0.0x] Memory: ... / SLUB: ... / smpboot: bringing up CPUs
$ sudo dmesg | grep -iE 'command line|memory:|smpboot|mounted root' | head
# the phases: parse cmdline → set up memory → bring up CPUs → mount root

# ---- 4. THE MAIN EVENT: watch a full boot in QEMU (virt L12/14) ----
# download a tiny cloud image or use one you have; -nographic pipes the
# serial console to your terminal so you SEE firmware→kernel→init:
$ command -v qemu-system-x86_64 >/dev/null && cat << 'NOTE'
  qemu-system-x86_64 -m 512 -nographic \
    -kernel /boot/vmlinuz-$(uname -r) \
    -initrd /boot/initrd.img-$(uname -r) \
    -append "console=ttyS0 root=/dev/ram0 rdinit=/bin/sh" 2>/dev/null

  ↑ boots YOUR kernel + initramfs in a VM, serial console to your terminal.
    You'll see the exact printk sequence from step 3 scroll past live, then
    drop into the initramfs shell (Lesson 51's world). Ctrl-A X to quit QEMU.
NOTE
# (Full disk boot needs a bootable image + OVMF firmware — virt Lesson 14.
#  The -kernel shortcut lets QEMU act as the bootloader, skipping firmware.)

# ---- 5. See a kernel parameter you set take effect ----
$ cat /proc/cmdline | tr ' ' '\n' | grep -E 'psi|hugepages|quiet|root' 
# every token here was assembled by the bootloader and consumed by the kernel
$ sysctl kernel.core_pattern      # some settings' defaults trace to cmdline
$ ls /sys/module/ | head -5       # modules whose params CAME from cmdline

# ---- 6. The chain of PIDs it produced (Lesson 06, from the top) ----
$ ps -p 1 -o pid,comm             # PID 1 = systemd = the boot's final handoff
$ systemd-analyze 2>/dev/null      # "Startup finished in 3.2s (kernel) + 8s (userspace)"
$ systemd-analyze blame 2>/dev/null | head -5   # what took the longest
```

---

## Further Reading

| Topic | Link |
|---|---|
| Booting (Linux) | <https://en.wikipedia.org/wiki/Linux_startup_process> |
| UEFI | <https://en.wikipedia.org/wiki/UEFI> |
| GRUB | <https://en.wikipedia.org/wiki/GNU_GRUB> |
| `bootparam(7)` — kernel command line | <https://man7.org/linux/man-pages/man7/bootparam.7.html> |
| kernel-parameters — full list | <https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html> |
| `dmesg(1)` man page | <https://man7.org/linux/man-pages/man1/dmesg.1.html> |

---

## Checkpoint

**Q1.** Where does the kernel command line come from, and name two parameters
you've already used in other tracks or phases (say what each does).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It's assembled by the <strong>bootloader</strong> (GRUB/systemd-boot) from
its config — for GRUB, generated from <code>/etc/default/grub</code>'s
GRUB_CMDLINE_LINUX plus the detected root device — and handed to the kernel
at the moment the bootloader jumps into it; the kernel echoes it (dmesg
"Command line:") and exposes it forever at <code>/proc/cmdline</code>.
Parameters seen earlier: <code>root=UUID=...</code> (which device to mount as
/ — the single most essential one, without which the kernel panics
"unable to mount root fs"); <code>psi=1</code> (enable pressure-stall
accounting — Lesson 15); <code>hugepagesz=1G hugepages=8</code> (reserve
1 GiB hugepages at boot — Lesson 23, which must be boot-time because
contiguous physical memory is only guaranteed early); virt track's
<code>kvm.ignore_msrs</code> / module params (Lesson 09);
<code>console=ttyS0</code> (route kernel messages to a serial port — the
lab's whole trick). The theme: the command line is the one configuration
channel available <em>before</em> any filesystem or userspace exists, so
anything the kernel must know at the very start lives here.
</details>

**Q2.** Explain the capability handoffs: why can't the firmware just load
Linux directly — what does each stage (firmware → bootloader → kernel) know
that the previous one doesn't?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Each stage is deliberately minimal, knowing only enough to start the next —
a bootstrapping ladder. <strong>Firmware</strong> lives in ROM with tight
size limits; it knows how to initialize the CPU and RAM and read a
<em>simple</em> medium (a raw sector on BIOS, or FAT on UEFI) — it does
<em>not</em> understand ext4/xfs/btrfs (Lessons 38–39) or Linux's kernel
format, and embedding drivers for every filesystem and OS in motherboard ROM
would be unmaintainable. <strong>Bootloader</strong> is a real program on
disk, so it can be large enough to understand filesystems (find /boot),
present menus, and know how to load a Linux kernel + initramfs and build the
command line — but it's not an OS: it can't run programs, schedule, or drive
most hardware. <strong>Kernel</strong> knows everything (this whole course)
but was just a passive file until loaded, and even it can't reach the real
root filesystem yet without drivers that may live <em>in</em> that
filesystem — the chicken-and-egg the initramfs (Lesson 51) resolves. Each
boundary is also a <em>trust</em> boundary (Secure Boot verifies the next
stage's signature), which is why the chain matters for security, not just
capability. The design principle: bootstrap from tiny-and-fixed to
large-and-flexible, one verified handoff at a time.
</details>

**Q3.** Your VM won't boot after a bad update — it hangs before login. Using
this lesson, describe the GRUB-menu recovery path to a root shell, and why
console access this powerful means physical/console access is effectively
root (tie to Lesson 11).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
At the GRUB menu (hold Shift/Esc during boot if hidden), press <code>e</code>
to edit the selected entry, find the <code>linux</code> line, and either
append <code>single</code> (boot to a minimal single-user root maintenance
shell) or, more forcefully, <code>init=/bin/bash</code> — which replaces PID
1 itself with a bash shell (Lesson 06: the kernel execs whatever
<code>init=</code> names), landing you at a root prompt with the real root
filesystem mounted (remount rw: <code>mount -o remount,rw /</code>), where
you can revert the bad update, reset a forgotten password (<code>passwd</code>),
or fix the config, then reboot. Why this equals root: you've bypassed every
authentication and authorization mechanism — no login, no password, no PAM,
no sudo policy (Lesson 11's entire credential system) — because all of that
is <em>userspace</em> enforced by processes that a normal boot starts, and
you rewrote what the boot starts <em>before</em> any of it runs. The kernel
grants init whatever you asked for. Therefore anyone who can reach the
bootloader menu (physical access, or console/serial in a VM/cloud) can become
root regardless of OS-level security — which is why real hardening requires
disk encryption (so the attacker can't mount root — the initramfs prompts for
a passphrase, Lesson 51), a GRUB password, and Secure Boot: the OS's
credential model has no authority over the layer beneath it, and console
access lives beneath it. "Physical access is root" is this fact.
</details>

---

## Homework

Run the QEMU boot from lab 4 (or study its output if you can't). Then map
each thing you see scroll past to a lesson in this course: find in the boot
log (a) memory initialization, (b) CPU bring-up, (c) the scheduler starting,
(d) devices/drivers probing, (e) the root filesystem being mounted, (f) the
handoff to init. For each, name the phase/lesson that explained the machinery
and one command that inspects that subsystem on the running system.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) <strong>Memory init</strong> — dmesg "Memory: ... available", SLUB/slab
setup, "Zone ranges" → Phase 4 (paging, page allocator, Lessons 16–23);
inspect: <code>/proc/meminfo</code>, <code>free -h</code>. (b) <strong>CPU
bring-up</strong> — "smpboot: Booting Node 0 ... CPU1" (each core started in
turn), "smpboot: Total of N processors activated" → Phase 3 scheduling +
Lesson 05 interrupts (the APIC/timer setup) + Lesson 23 NUMA (Node lines);
inspect: <code>lscpu</code>, <code>/proc/interrupts</code>. (c)
<strong>Scheduler</strong> — "sched_clock", "clocksource" selection, "rcu"
init (the RCU threads from Lesson 01's kthreadd children) → Lesson 13
(CFS/EEVDF), Lesson 05 (timer/tick); inspect: <code>/proc/sched_debug</code>,
<code>chrt -p 1</code>. (d) <strong>Device/driver probe</strong> — "virtio_blk
virtio1: [vda]", "e1000/virtio_net", PCI enumeration → Lesson 53 (modules &
udev) + Phase 7 (block layer, Lesson 42) + networking track; inspect:
<code>lspci</code>, <code>lsmod</code>, <code>ls /sys/class</code>. (e)
<strong>Root mount</strong> — "EXT4-fs (vda1): mounted filesystem", "VFS:
Mounted root" (preceded by initramfs "Loading, please wait" and possibly a
LUKS/LVM prompt) → Lessons 37–40 (VFS, ext4, mounts) + Lesson 51
(initramfs); inspect: <code>findmnt /</code>, <code>dmesg | grep EXT4</code>.
(f) <strong>Handoff to init</strong> — "Run /sbin/init as init process",
then systemd's "Welcome to..." and unit startup → Lessons 06/52 (PID 1,
systemd); inspect: <code>systemd-analyze</code>, <code>ps -p 1</code>. The
exercise's point: a boot log is this entire course executing in sequence —
every subsystem you studied in isolation, announcing itself in the order
dependencies demand.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 51 — initramfs →](lesson-51-initramfs){: .btn .btn-primary }
