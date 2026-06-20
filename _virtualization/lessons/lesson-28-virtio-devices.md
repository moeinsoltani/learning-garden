---
title: "Lesson 28 — The Core virtio Devices"
nav_order: 28
parent: "Phase 7: VirtIO & Paravirtualization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 28: The Core virtio Devices

## Concept

virtio is a *framework* (Lesson 27); on top of it sits a **catalog** of device types,
each replacing a class of emulated hardware with a fast paravirtual equivalent. Know
the catalog and what each is for.

```
   storage     virtio-blk, virtio-scsi        (Lesson 24)
   network     virtio-net                     (Phase 8)
   memory      virtio-balloon                 (Lesson 20)
   entropy     virtio-rng
   console     virtio-console / virtio-serial
   graphics    virtio-gpu
   filesharing virtio-fs   (and legacy 9p)
   host comms  virtio-vsock
```

Linux has built-in virtio drivers, so Linux guests "just work." **Windows does not**
— it needs the separate **virtio-win** driver package, which creates the classic
chicken-and-egg problem of installing a guest onto a virtio disk it can't yet see.

---

## How It Works

### The catalog

| Device | Purpose |
|---|---|
| **virtio-net** | Paravirtual NIC — the fast alternative to e1000/rtl8139 (Phase 8). |
| **virtio-blk** | Simple paravirtual block device (Lesson 24). |
| **virtio-scsi** | Paravirtual SCSI HBA: many disks, passthrough, TRIM (Lesson 24). |
| **virtio-rng** | Feeds host entropy to the guest so its `/dev/random` doesn't starve. |
| **virtio-balloon** | Cooperative memory reclamation (Lesson 20). |
| **virtio-console / virtio-serial** | Paravirtual serial ports/channels (e.g. the guest agent talks over a virtio-serial port). |
| **virtio-gpu** | Paravirtual GPU; 2D, and 3D (virgl/venus) with the right stack. |
| **virtio-fs** | Share a *host directory* into the guest with near-native, shared-memory performance (DAX). |
| **virtio-vsock** | Host↔guest socket communication with no networking at all (Lesson 30). |

### Why virtio-rng matters more than it looks

A freshly booted guest has little entropy; services needing randomness (TLS, SSH host
keys, `getrandom`) can **block** waiting for the entropy pool to fill. **virtio-rng**
pipes entropy from the host's pool into the guest, eliminating boot-time stalls and
slow key generation. Cheap to add, often forgotten.

### virtio-fs vs 9p for file sharing

To share a host directory into a guest you have two options:

- **9p** (Plan 9 filesystem protocol over virtio) — older, works, but slower and with
  protocol overhead.
- **virtio-fs** — purpose-built: uses a shared-memory region (DAX) so the guest can
  access host file data with near-local performance, with proper POSIX semantics. The
  modern choice (used by Kata Containers, Lesson 60).

### The Windows driver problem (and the fix)

Linux ships virtio drivers in-kernel; a Linux guest sees virtio-net/blk immediately.
**Windows** has no built-in virtio drivers, so a fresh Windows guest shows "unknown
device" for its virtio NIC and **can't even see its virtio boot disk** — you can't
install onto a disk Windows can't detect.

The fix (installation order matters):

1. Attach the **virtio-win ISO** as a second CD-ROM alongside the Windows installer.
2. During Windows setup, when no disk is found, **"Load driver"** from the virtio-win
   ISO (the `viostor`/`vioscsi` storage driver) so Windows can see the virtio disk.
3. After install, install the rest of the virtio-win package (NetKVM for networking,
   balloon, etc.) and the **guest agent**.

Alternatively, install Windows onto an emulated SATA disk first, then add virtio
drivers and switch the disk to virtio on the next boot.

{: .note }
> **The chicken-and-egg, stated plainly**
> To boot from a virtio disk you need the virtio storage driver; but to install the
> OS that contains the driver, you need to already see the disk. Linux dodges this
> (driver is in the kernel/initramfs). Windows resolves it by side-loading the storage
> driver from the virtio-win ISO during setup, or by installing on emulated storage
> first and migrating to virtio afterward. Networking (NetKVM) and the balloon driver
> can be added post-install, but the *boot/storage* driver must be present at install
> time.

---

## Lab

```bash
# 1. Add several virtio devices at once — note virtio-rng for entropy:
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -drive file=disk.qcow2,if=virtio \
    -netdev user,id=n0 -device virtio-net-pci,netdev=n0 \
    -object rng-random,filename=/dev/urandom,id=rng0 \
    -device virtio-rng-pci,rng=rng0 \
    -device virtio-balloon-pci \
    -nographic &

# 2. In a Linux guest, confirm the drivers loaded:
#   (guest)$ lspci | grep -i virtio
#   (guest)$ lsmod | grep -E 'virtio_net|virtio_blk|virtio_rng|virtio_balloon'
#   (guest)$ cat /sys/devices/virtual/misc/hw_random/rng_current   # virtio_rng.0

# 3. Share a HOST directory into the guest with virtio-fs:
#    (run the vhost-user-fs daemon, then attach; simplified):
$ qemu-system-x86_64 -accel kvm -m 2G \
    -object memory-backend-file,id=mem,size=2G,mem-path=/dev/shm,share=on \
    -machine q35,memory-backend=mem \
    -chardev socket,id=cf,path=/tmp/vfs.sock \
    -device vhost-user-fs-pci,chardev=cf,tag=hostshare \
    -drive file=disk.qcow2,if=virtio -nographic &
#   (guest)$ mount -t virtiofs hostshare /mnt    # host dir now visible at /mnt

# 4. For a Windows install, attach the virtio-win ISO as a SECOND cdrom:
$ qemu-system-x86_64 -accel kvm -m 4G -smp 4 \
    -drive file=win.qcow2,if=virtio \
    -cdrom Windows.iso \
    -drive file=virtio-win.iso,media=cdrom \
    -nographic
# During setup → "Load driver" → pick viostor from the virtio-win CD.
$ kill %1 %2 2>/dev/null
```

**Expected result:** A Linux guest auto-loads all the virtio drivers (including
`virtio_rng` feeding `hw_random`); virtio-fs mounts a host directory inside the guest;
and a Windows install can side-load the `viostor` driver from the virtio-win ISO to
see its virtio boot disk.

---

## Further Reading

| Topic | Link |
|---|---|
| virtio | [Wikipedia — Virtio](https://en.wikipedia.org/wiki/Virtio) |
| virtio-fs | [virtio-fs.gitlab.io](https://virtio-fs.gitlab.io/) |
| virtio-win drivers | [fedoraproject — Windows virtio drivers](https://docs.fedoraproject.org/en-US/quick-docs/creating-windows-virtual-machines-using-virtio-drivers/) |
| Hardware random number generator | [Wikipedia — Hardware RNG](https://en.wikipedia.org/wiki/Hardware_random_number_generator) |
| 9P (protocol) | [Wikipedia — 9P (protocol)](https://en.wikipedia.org/wiki/9P_(protocol)) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. A Windows guest sees "unknown device" for its virtio NIC and disk. What's missing, and how do you fix the installation order so it can boot from a virtio disk?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Windows lacks built-in virtio drivers (unlike Linux), so it can't recognize the virtio NIC or — more critically — the virtio boot disk. The fix is the virtio-win driver package. For installation: attach the virtio-win ISO as a second CD-ROM during Windows setup, and when the installer finds no disk, use "Load driver" to side-load the storage driver (viostor for virtio-blk / vioscsi for virtio-scsi) so Windows can see the virtio disk and install onto it. After install, add the rest (NetKVM networking, balloon, guest agent). Alternatively install onto an emulated SATA disk first, then add virtio drivers and switch to virtio. The storage driver must be present at install time because you can't install onto a disk Windows can't see.
</details>

---

**Q2. What does virtio-rng do, and what problem does it solve at guest boot?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
virtio-rng feeds entropy from the host's random pool into the guest's RNG. It solves entropy starvation: a freshly booted guest has little entropy, so operations needing randomness (generating SSH host keys, TLS handshakes, getrandom) can block until the pool fills, stalling boot and slowing services. With virtio-rng the guest gets a steady entropy supply, eliminating those stalls and slow key generation.
</details>

---

**Q3. Why is virtio-fs generally preferred over 9p for sharing a host directory into a guest?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
virtio-fs is purpose-built for VM file sharing: it uses a shared-memory region (DAX) so the guest can access host file data with near-local performance and proper POSIX semantics. 9p is the older Plan 9 filesystem protocol tunneled over virtio — it works but has more protocol overhead and lower performance, and weaker semantics in places. So for performance and correctness (and use cases like Kata Containers) virtio-fs is the modern choice.
</details>

---

## Homework

Boot a Linux guest with `virtio-rng-pci` attached. Inside the guest, check `cat /sys/devices/virtual/misc/hw_random/rng_current` (should mention virtio) and time how long `ssh-keygen -t ed25519 -f /tmp/k -N ""` takes. Then boot the same guest *without* virtio-rng and compare. Explain the difference in terms of the entropy pool.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With virtio-rng attached, rng_current reports the virtio RNG and key generation completes essentially instantly, because the guest's entropy pool is continuously fed from the host, so getrandom never blocks. Without virtio-rng (especially on a freshly booted, idle guest with little disk/network activity), the entropy pool may be low, so ssh-keygen / getrandom can block waiting for enough randomness, making key generation noticeably slower or stalling until entropy accumulates. virtio-rng removes that dependency on the guest slowly gathering its own entropy by supplying it from the host pool. (Modern kernels mitigate blocking via the CRNG, but virtio-rng still helps fill the pool faster, especially early in boot.)
</details>
