---
title: "Lesson 51 — initramfs"
nav_order: 2
parent: "Phase 10: Boot & Init"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 51: initramfs

## Concept

Lesson 50 left a puzzle: the kernel needs to mount your root filesystem, but
the *driver* to reach that filesystem might live *on* that filesystem. Your
root is on an NVMe disk (needs `nvme` module), inside LVM (needs `dm-mod`),
on top of LUKS encryption (needs `dm-crypt` + your passphrase) — none
reachable until root is mounted, which needs all of them. Chicken, meet egg.

The **initramfs** (initial RAM filesystem) breaks it:

```
  bootloader loads TWO things (Lesson 50):
     vmlinuz  ─────────▶  the kernel
     initrd.img ────────▶ a compressed cpio archive → unpacked into a
                          tmpfs (Lesson 39!) that becomes the FIRST root
        │
        ▼  a tiny, self-contained userland in RAM:
        /init  (a script), busybox, JUST the modules + tools needed
        to FIND and mount the real root:
           load nvme, dm-mod, dm-crypt →
           prompt for LUKS passphrase →
           activate LVM →
           mount the real root →
           switch_root to it, exec the real /sbin/init (Lesson 52)
```

It's a disposable bootstrap root: everything needed to reach the real root,
and nothing else, living in RAM where no driver is required to read it (the
kernel unpacks a cpio archive with built-in code). Once its job is done it
hands off and vanishes. You've unknowingly seen its cousin: it's a tmpfs
(Lesson 39), populated by a cpio archive (the `tar`-like format), running a
minimal userspace — and the QEMU boot in Lesson 50 dropped you *into* one.

---

## How It Works

### What's inside, and who builds it

`initramfs-tools` (Debian/Ubuntu) or `dracut` (Fedora/RHEL) generate it at
kernel-install time by scanning *your* hardware and root setup: they include
only the kernel modules and userspace tools needed for *this* machine's boot
path (the `nvme`/`dm-crypt`/`raid` modules you actually use, `busybox` or a
small util set, the `/init` script). That's why it's regenerated
(`update-initramfs -u`, `dracut -f`) when you change root storage, add
encryption, or install a kernel — and why a mismatched initramfs
(post-restore, post-`dd`) causes "cannot find root" boot failures.

### The handoff: switch_root

`/init` does its work, then **switch_root** (a cousin of `pivot_root` —
Lesson 40): it makes the newly-mounted real root the `/`, moves essential
mounts across, frees the initramfs's memory, and `exec`s the real
`/sbin/init` — which becomes the ongoing PID 1 (Lesson 06: exec keeps the
PID, so systemd inherits PID 1 from the initramfs's init). One process
lineage from the kernel's first userspace jump to your running system.

### When there's no initramfs

If root needs *no* special drivers — everything built into the kernel
(`=y` not `=m`), plain partition, no LVM/LUKS/RAID — you can boot without an
initramfs (`root=/dev/sda1` directly). Embedded systems and custom kernels
(Lesson 58!) often do exactly this. But general-purpose distros ship
modular kernels (small `vmlinuz` + loadable modules) precisely so one kernel
boots any hardware — which *requires* the initramfs to load the right
modules for each machine. The trade is Lesson 46's static-vs-dynamic debate,
at the kernel level.

{: .note }
> **Peeking inside — and building your own**
> <code>lsinitramfs /boot/initrd.img-$(uname -r)</code> lists its contents;
> <code>unmkinitramfs</code> (or <code>cpio -id</code> after decompression)
> extracts it. The lab builds a one-file initramfs from scratch and boots it
> in QEMU — the fastest way to truly understand "userspace is just files the
> kernel runs," and the foundation for Lesson 58's custom-kernel boot and
> Lesson 62's build-a-container.

---

## Lab

```bash
# ---- 1. Look inside your real initramfs ----
$ lsinitramfs /boot/initrd.img-$(uname -r) 2>/dev/null | head -20 || \
    lsinitrd /boot/initramfs-$(uname -r).img 2>/dev/null | head -20
# init, bin/busybox, the module tree (kernel/drivers/...), scripts/
$ lsinitramfs /boot/initrd.img-$(uname -r) 2>/dev/null | grep -E '\.ko' | head -8
# ONLY the modules your boot path needs: nvme, virtio_blk, dm-mod, ext4...
$ lsinitramfs /boot/initrd.img-$(uname -r) 2>/dev/null | grep -c '\.ko'
# a few dozen, not the thousands a full kernel has — curated for THIS machine

# ---- 2. It's just a compressed cpio archive ----
$ file /boot/initrd.img-$(uname -r)
# ASCII cpio archive / gzip / zstd compressed data — (may be concatenated:
#   an uncompressed microcode blob + the compressed main archive)
$ mkdir -p /tmp/initx && cd /tmp/initx
$ sudo unmkinitramfs /boot/initrd.img-$(uname -r) /tmp/initx 2>/dev/null && ls
$ cat /tmp/initx/init 2>/dev/null | head -20 || find /tmp/initx -name init | head -1
# a SHELL SCRIPT — the whole "initial userspace" is readable text + busybox

# ---- 3. BUILD your own minimal initramfs and boot it ----
$ mkdir -p /tmp/myinit && cd /tmp/myinit
$ mkdir -p bin dev proc sys
# static busybox gives us a shell + coreutils in one file:
$ cp $(which busybox) bin/ 2>/dev/null || sudo apt install -y busybox-static && cp /bin/busybox bin/
$ cat > init << 'EOF'
#!/bin/busybox sh
/bin/busybox mkdir -p /proc /sys /dev
/bin/busybox mount -t proc none /proc
/bin/busybox mount -t sysfs none /sys
echo ""
echo "=== Hello from a hand-built initramfs! ==="
echo "kernel cmdline was: $(cat /proc/cmdline)"
echo "I am PID $$ — the first userspace process (L06)"
/bin/busybox sh          # drop to an interactive shell
EOF
$ chmod +x init
# for busybox applets to work as commands:
$ for cmd in sh mount mkdir cat echo ls; do ln -sf busybox bin/$cmd; done
# pack it into a cpio.gz — the initramfs format:
$ find . | cpio -o -H newc 2>/dev/null | gzip > /tmp/myinitramfs.gz
$ ls -lh /tmp/myinitramfs.gz              # your entire "boot userland", ~1MB

# ---- 4. Boot it in QEMU — YOUR init runs as PID 1 (virt L12) ----
$ command -v qemu-system-x86_64 >/dev/null && cat << 'NOTE'
  qemu-system-x86_64 -m 256 -nographic \
    -kernel /boot/vmlinuz-$(uname -r) \
    -initrd /tmp/myinitramfs.gz \
    -append "console=ttyS0"

  → the kernel boots, unpacks YOUR cpio into a tmpfs, runs /init as PID 1,
    and drops you into your busybox shell. Run `ps`, `cat /proc/1/comm`
    (= init — that's YOU), `ls /`. Ctrl-A X to exit QEMU.
    You just built userspace from one script + one binary. That's all an
    initramfs IS.
NOTE

# ---- 5. Why encryption forces an initramfs (the chicken-and-egg, concrete) ----
$ lsblk -o NAME,TYPE,FSTYPE,MOUNTPOINT | grep -iE 'crypt|lvm|raid' | head
# if you see crypt/lvm here, your root NEEDS the initramfs to unlock/activate
$ lsinitramfs /boot/initrd.img-$(uname -r) 2>/dev/null | grep -iE 'cryptsetup|lvm|dmsetup' | head
# the tools baked in to do exactly that, before the real root exists
$ cd / && rm -rf /tmp/initx /tmp/myinit /tmp/myinitramfs.gz
```

---

## Further Reading

| Topic | Link |
|---|---|
| initramfs — kernel docs | <https://www.kernel.org/doc/html/latest/filesystems/ramfs-rootfs-initramfs.html> |
| initrd / initramfs (Wikipedia) | <https://en.wikipedia.org/wiki/Initial_ramdisk> |
| `dracut(8)` man page | <https://man7.org/linux/man-pages/man8/dracut.8.html> |
| `switch_root(8)` man page | <https://man7.org/linux/man-pages/man8/switch_root.8.html> |
| `cpio(1)` — the archive format | <https://man7.org/linux/man-pages/man1/cpio.1.html> |
| `update-initramfs(8)` | <https://manpages.debian.org/update-initramfs> |

---

## Checkpoint

**Q1.** Your root is on LVM-on-LUKS. Why is booting without an initramfs
essentially impossible?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The kernel must mount root to reach userspace, but reaching this root
requires steps that themselves need drivers and userspace tools not
available before root is mounted — a strict circular dependency. Concretely:
the physical disk needs its controller driver (nvme/ahci — a module);
the LUKS layer needs the <code>dm-crypt</code> module <em>and</em> a
userspace prompt to collect your passphrase and <code>cryptsetup</code> to
open the encrypted volume (the kernel can't ask you for a password on its
own); the LVM layer needs <code>dm-mod</code> and <code>lvm</code> tools to
scan and activate the logical volume; only then does the ext4/xfs root
appear to be mounted. Every one of those modules and tools would have to
come <em>from</em> the encrypted, LVM'd root — which isn't accessible yet.
The initramfs breaks the loop by carrying exactly those modules and tools in
a RAM filesystem the kernel can unpack with no drivers at all, running
<code>/init</code> to unlock → activate → mount → switch_root. Without it
the kernel decompresses, looks for root, finds an encrypted blob it can't
read, and panics "unable to mount root fs." (A fully built-in kernel could
handle the <em>drivers</em>, but not the interactive passphrase prompt —
so encrypted root effectively mandates an initramfs regardless.)
</details>

**Q2.** Trace the PID-1 lineage from the kernel's first userspace instruction
to your running systemd. What role does switch_root play, and why is it
`switch_root` and not just "start systemd"?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The kernel, after unpacking the initramfs into a tmpfs root, execs its
<code>/init</code> as the very first userspace process — PID 1. That init
does the bootstrap work (load modules, unlock/activate/mount the real root
under a mountpoint like /root). Then <code>switch_root</code>: it makes the
real root become <code>/</code> (moving essential mounts — /proc, /sys, /dev
— across, then freeing the initramfs's RAM since it's disposable) and, using
<code>exec</code>, replaces itself with the real <code>/sbin/init</code>
(systemd). Because exec preserves the PID (Lesson 07 — same process shell,
new program), systemd <em>becomes</em> PID 1: one continuous process
identity from the kernel's first jump to your running system. Why switch_root
rather than "just start systemd as a new process": PID 1 is special and
singular — the kernel only ever creates one first process, and there must
always be exactly one PID 1 (its death panics the kernel — Lesson 52). You
can't spawn "a second PID 1"; you must <em>transform</em> the existing one in
place, and simultaneously pivot the entire filesystem view from the throwaway
RAM root to the real one (switch_root also prevents the initramfs from
lingering as a hidden mount holding the PID-1 role). It's pivot_root
(Lesson 40) specialized for the boot's one-time root handoff — same reason
containers use pivot_root to install their root without spawning a fake init.
</details>

**Q3.** After restoring a system from backup (or `dd`ing a disk image to
different hardware), it fails to boot with "cannot find root device" even
though the files are all present. Explain why, and the fix — connecting to
how the initramfs is generated.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The initramfs is <em>machine-specific</em>: it was generated (by
initramfs-tools/dracut) by scanning the <em>original</em> hardware and root
layout, so it contains only the modules and root-identification (a specific
UUID, or the specific storage-controller driver) for <em>that</em> machine.
On different hardware, the boot path changed — the new disk controller
(different NVMe/SATA/virtio driver not included), a different root UUID than
the one baked into the initramfs's config, or a RAID/LVM setup the old
initramfs doesn't know how to assemble — so /init loads the wrong (or no)
driver, never finds the device matching the expected root, times out, and
drops to an emergency "cannot find root" shell. The files being present is
irrelevant: the failure is in the <em>bootstrap</em>, before the root
filesystem is reachable. Fix: regenerate the initramfs against the new
reality — boot a rescue/live environment, chroot into the restored system
(mounting /proc, /sys, /dev — the bind mounts of Lesson 40), then
<code>update-initramfs -u -k all</code> (or <code>dracut -f</code>) to
rescan hardware and rebuild with the correct modules, and update
<code>/etc/fstab</code> + the bootloader's <code>root=</code> to the new
UUID (<code>update-grub</code>). It's the same lesson as static-vs-dynamic
(Lesson 46): the modular kernel's flexibility is paid for by a
machine-tailored initramfs that must be regenerated whenever the machine
changes — which is precisely what image-based deployment and "generic
cloud" images (with broad module sets and cloud-init, virt Lesson 44)
engineer around.
</details>

---

## Homework

Your hand-built initramfs (lab 3) ran a shell as PID 1 but couldn't reach any
real disk. Extend the design (on paper or in practice): what would you need
to add to make it mount a real ext4 partition (`/dev/vdb1`) and switch_root
into it — list the modules, tools, device nodes, and the /init logic. Then
explain how this exercise is a stripped-down version of both a real distro's
boot AND a container runtime's startup (Lesson 62).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
To mount and pivot into a real root, the initramfs needs: <strong>modules</strong>
— the storage controller driver (<code>virtio_blk.ko</code> for QEMU's disk,
or nvme/ahci on metal) and the filesystem driver (<code>ext4.ko</code>) if
not built-in, inserted with <code>modprobe</code>/<code>insmod</code>
(Lesson 53); <strong>device nodes</strong> — /dev populated so /dev/vdb1
exists (mount devtmpfs, or run a mini-udev/mdev — Lesson 53's territory);
<strong>tools</strong> — busybox provides mount, switch_root, modprobe;
<strong>/init logic</strong>:
<pre>
mount -t devtmpfs none /dev        # so block device nodes appear
modprobe virtio_blk; modprobe ext4 # (or rely on built-ins)
mkdir /newroot
mount -t ext4 /dev/vdb1 /newroot   # the real root
mount --move /proc /newroot/proc   # carry essential mounts across
mount --move /sys  /newroot/sys
mount --move /dev  /newroot/dev
exec switch_root /newroot /sbin/init   # pivot + hand off PID 1
</pre>
This is the whole distro boot in miniature: a real initramfs does exactly
this with more robustness (wait for devices to appear, LUKS prompts, LVM
scan, root= parsing, fsck, fallback shells) but the same skeleton —
detect, mount, switch_root, exec init. And it's a container runtime's
startup with two swaps: the container replaces <code>switch_root</code> with
<code>pivot_root</code> into an <em>overlayfs</em> root (Lesson 39) instead
of a real partition, and wraps the whole thing in fresh <em>namespaces</em>
(Lesson 59) so the new root, PID space, and mounts are private — then execs
the container's entrypoint as its PID 1 (inheriting the reaping duty,
Lesson 08's container-zombie problem). Boot and containerization are the
same move — "assemble a root filesystem in a throwaway environment, then
pivot into it and exec the real first process" — which is why Lesson 62
builds a container by literally performing these steps by hand, and why
understanding initramfs is understanding containers from the other side.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 52 — systemd: PID 1 →](lesson-52-systemd){: .btn .btn-primary }
