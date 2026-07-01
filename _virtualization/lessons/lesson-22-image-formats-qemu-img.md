---
title: "Lesson 22 — Disk Image Formats and qemu-img"
nav_order: 22
parent: "Phase 6: Storage"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 22: Disk Image Formats and qemu-img

## Concept

A virtual disk is just a file on the host that QEMU presents to the guest as a block
device. But that file has a **format**, and the format determines whether you get
raw speed or rich features. After the emulator itself, **`qemu-img`** is the tool
you'll use most.

```
   raw                              qcow2
   ─────                            ─────
   bytes map 1:1 to disk            metadata header + L1/L2 tables
   offset N in file = LBA N         + clusters of guest data
   ┌──────────────────────┐        ┌────┬──────────────────────────┐
   │  guest blocks, plain  │        │meta│ allocated clusters only   │
   └──────────────────────┘        └────┴──────────────────────────┘
   fastest, no features             snapshots, backing files,
                                    compression, encryption, sparse
```

**raw** = maximum performance, zero features. **qcow2** (QEMU Copy-On-Write v2) =
slightly more overhead, but snapshots, backing files (templates/clones),
compression, and built-in sparseness. Choosing between them is a recurring
decision.

---

## How It Works

### The formats

| Format | Notes |
|---|---|
| **raw** | The file *is* the disk image, byte-for-byte. No metadata. Fastest, simplest, works with any tool. Sparse if the underlying filesystem supports holes. No snapshots/backing/compression. |
| **qcow2** | QEMU's native format: lazy allocation (grows as data is written), internal snapshots, backing files (copy-on-write overlays), optional compression and encryption (LUKS). The default for flexibility. |
| **vmdk / vdi / vhdx** | VMware / VirtualBox / Hyper-V formats. `qemu-img` can read/convert them — useful for importing VMs from other hypervisors. |

### Thin vs thick provisioning, and sparseness

- **Thin / sparse:** the image only consumes host space for blocks actually written.
  A "100 GiB" qcow2 with 3 GiB of data uses ~3 GiB on disk. qcow2 is thin by design;
  raw is thin only on filesystems that support sparse files (holes).
- **Thick / preallocated:** all space is reserved up front. More predictable
  performance (no allocation on first write), no risk of the host filling up under a
  growing image, but uses full space immediately. `qemu-img create -o preallocation=`
  controls this (`off`/`metadata`/`falloc`/`full`).

### qemu-img — the essential subcommands

| Command | Purpose |
|---|---|
| `qemu-img create -f qcow2 d.qcow2 20G` | Create a new (thin) image. |
| `qemu-img info d.qcow2` | Show format, virtual size, actual size, backing file, snapshots. |
| `qemu-img convert -O raw in.qcow2 out.raw` | Convert between formats (also compacts/flattens). |
| `qemu-img resize d.qcow2 +10G` | Grow (or shrink, with `--shrink`) the virtual disk. |
| `qemu-img check d.qcow2` | Verify integrity of the metadata/refcounts. |
| `qemu-img snapshot -l d.qcow2` | List internal snapshots. |

`qemu-img info` is the one you'll run constantly — it tells you virtual size vs real
consumption, the backing chain, and embedded snapshots.

{: .note }
> **Why convert can shrink an image**
> A qcow2 that has had data written and deleted can stay large because freed clusters
> aren't always returned. <code>qemu-img convert</code> rewrites only the live data
> into a fresh image, dropping unreferenced clusters — effectively compacting it. It
> also flattens a backing-file chain (Lesson 23) into a single standalone image, which
> is handy before exporting a VM.

---

## Lab

```bash
# 1. Create a thin 10 GiB qcow2 and inspect it:
$ qemu-img create -f qcow2 disk.qcow2 10G
Formatting 'disk.qcow2', fmt=qcow2 cluster_size=65536 ... size=10737418240

$ qemu-img info disk.qcow2
image: disk.qcow2
file format: qcow2
virtual size: 10 GiB (10737418240 bytes)
disk size: 196 KiB              ← thin! almost nothing used yet
cluster_size: 65536

# 2. Compare actual disk usage vs virtual size:
$ du -h disk.qcow2
196K    disk.qcow2             ← real bytes on host
$ ls -lh disk.qcow2
-rw-r--r-- 1 you you 10G ... disk.qcow2   ← apparent (sparse) size hint

# 3. Convert qcow2 -> raw, then back, and compare:
$ qemu-img convert -O raw disk.qcow2 disk.raw
$ qemu-img info disk.raw
file format: raw
virtual size: 10 GiB
disk size: 0 B                 ← raw is sparse on a hole-supporting FS

$ qemu-img convert -O qcow2 disk.raw disk2.qcow2

# 4. Resize (grow) the virtual disk by 5 GiB:
$ qemu-img resize disk.qcow2 +5G
Image resized.
$ qemu-img info disk.qcow2 | grep 'virtual size'
virtual size: 15 GiB ...

# 5. Create a THICK (fully preallocated) raw image and see the difference:
$ qemu-img create -f raw -o preallocation=full thick.raw 2G
$ du -h thick.raw
2.0G    thick.raw              ← all space reserved immediately

# 6. Check integrity:
$ qemu-img check disk.qcow2
No errors were found on the image.
```

**Expected result:** A freshly created qcow2 reports a 10 GiB *virtual* size but
only ~200 KiB *actual* usage (thin). Conversion to raw and back works; resize grows
the virtual size; a preallocated image immediately consumes its full size.

---

## Further Reading

| Topic | Link |
|---|---|
| `qemu-img` reference | [qemu.org — qemu-img](https://www.qemu.org/docs/master/tools/qemu-img.html) |
| qcow2 format | [Wikipedia — qcow](https://en.wikipedia.org/wiki/Qcow) |
| Thin provisioning | [Wikipedia — Thin provisioning](https://en.wikipedia.org/wiki/Thin_provisioning) |
| Sparse file | [Wikipedia — Sparse file](https://en.wikipedia.org/wiki/Sparse_file) |
| QEMU block drivers | [qemu.org — Disk image file formats](https://www.qemu.org/docs/master/system/qemu-block-drivers.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. When would you choose raw over qcow2 despite losing snapshots and compression?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Choose raw when you want maximum, predictable performance and minimal overhead, and you don't need qcow2's features — e.g. high-I/O databases, when snapshots/backing-files are handled at a lower layer (LVM, Ceph/RBD, ZFS) or by the storage array, or when you need a plain byte-for-byte image other tools can read directly. raw avoids qcow2's metadata/L2-table indirection and copy-on-write amplification, giving the lowest latency and simplest behavior; you give up built-in snapshots, backing chains, and compression in exchange.
</details>

---

**Q2. What does `qemu-img info` tell you, and why is "virtual size" usually different from "disk size"?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
qemu-img info reports the format, the virtual size (how big the disk appears to the guest), the actual disk size (real bytes consumed on the host), the cluster size, any backing file, and embedded snapshots. Virtual size differs from disk size because qcow2 (and sparse raw) are thin: space is allocated lazily as the guest writes, so a 100 GiB virtual disk with only 3 GiB of written data consumes ~3 GiB on the host. The gap is unwritten/unallocated space.
</details>

---

**Q3. What is the difference between thin and thick provisioning, and one trade-off of each?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Thin/sparse provisioning consumes host space only for blocks actually written, so images start tiny and grow — saving space and enabling overcommit of storage, but risking the host filling up if many thin images grow at once, plus a small first-write allocation cost. Thick/preallocated provisioning reserves all the space up front — giving predictable performance (no allocation on first write) and no surprise-full-host risk, but using the full capacity immediately even when mostly empty.
</details>

---

## Homework

Create a 20 GiB qcow2, note its `disk size` from `qemu-img info`. Then attach it to a guest, write ~1 GiB inside the guest (`dd if=/dev/zero of=/tmp/f bs=1M count=1024`), shut down, and re-run `qemu-img info`. Explain how `disk size` changed and why, then run `qemu-img convert` to a new qcow2 and compare sizes after deleting the file in the guest first.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Initially disk size is tiny (~200 KiB) because the qcow2 is thin. After writing ~1 GiB inside the guest, disk size grows to roughly 1 GiB+ because qcow2 lazily allocated clusters for the newly written data. If you then delete the file inside the guest, the qcow2 usually does NOT shrink automatically — the clusters remain allocated (unless discard/TRIM is plumbed through). Running <code>qemu-img convert</code> to a new qcow2 rewrites only the live data, dropping the now-unreferenced clusters, so the converted image is smaller — demonstrating that convert compacts/reclaims freed space that a delete alone didn't return.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 23 — qcow2 Internals: Backing Files and Copy-on-Write →](lesson-23-qcow2-internals){: .btn .btn-primary }
