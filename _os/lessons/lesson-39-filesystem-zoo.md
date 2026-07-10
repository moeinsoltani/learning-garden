---
title: "Lesson 39 — The Filesystem Zoo: xfs, btrfs, tmpfs, overlayfs"
nav_order: 4
parent: "Phase 7: Files & Filesystems"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 39: The Filesystem Zoo — xfs, btrfs, tmpfs, overlayfs

## Concept

The VFS (Lesson 37) makes filesystems pluggable; ext4 (Lesson 38) is one
plug. This lesson tours the four others you'll actually meet — each embodying
one big idea:

```
  xfs        "ext4's philosophy, engineered for PARALLEL SCALE"
             RHEL's default; allocation groups work independently —
             big files, many CPUs, huge volumes

  btrfs      "COPY-ON-WRITE changes everything"
             never overwrite in place → snapshots are free,
             checksums everywhere, send/receive — qcow2's ideas
             (virt L23!) built INTO the filesystem

  tmpfs      "the page cache wearing a filesystem costume"
             no disk at all: files ARE page-cache pages —
             /dev/shm (L33!), /run, /tmp on many distros

  overlayfs  "filesystems as LAYERS: read through, write on top"
             lower (read-only) + upper (writes) = merged view —
             THE container image mechanism (L61 builds on this)
```

The unifying question for any filesystem: *what happens on write?* ext4/xfs
overwrite in place (journal for consistency); btrfs writes new blocks and
flips pointers (CoW — old versions persist for free); tmpfs dirties RAM
pages that were never destined for disk; overlayfs copies the file up from
the read-only layer, then writes the copy. Every capability and every
gotcha of each fs follows from its answer.

---

## How It Works

### xfs in three sentences

Multiple **allocation groups**, each with its own free-space management and
locks — parallel writers scale where ext4's single structures contend.
Delayed allocation, extent-based, metadata journaling — the same
crash-consistency contract as ext4-ordered. Quirks to know: it can grow but
**never shrink**, and its inode budget grows dynamically (no df -i wall).

### btrfs: what CoW buys and costs

Never overwriting means: **snapshots are O(1)** (a snapshot is "keep the old
pointer tree" — same trick as qcow2 backing files, virt Lesson 23);
**checksums on everything** (data + metadata — bit-rot detected on read,
self-healed if RAID copies exist); **send/receive** (ship the block-level
diff between snapshots — incremental backup built in); subvolumes (cheap
fs-within-fs boundaries — what snapper/Ubuntu's zsys snapshot). Costs, from
the same mechanism: fragmentation (every rewrite lands elsewhere — bad for
databases/VM images; hence `chattr +C` to opt files out of CoW), write
amplification, and the infamous "df is complicated" (shared blocks between
snapshots make "free space" genuinely ambiguous — `btrfs filesystem usage`
tells the layered truth). ZFS is the same philosophy, out-of-tree.

### tmpfs precisely

A mount whose files live purely as page-cache pages (Lesson 20) with no
backing store: RAM-speed always, contents vanish at unmount/reboot,
`size=` caps it (default half of RAM), and under memory pressure it
**swaps** (Lesson 21 — tmpfs pages are anonymous-like; a "RAM disk" that
can quietly hit disk!). `/dev/shm` (Lesson 33's shm objects live here),
`/run` (pidfiles/sockets that must die at reboot), and — increasingly —
`/tmp` are all tmpfs: if it must not survive reboot, tmpfs is the honest home.

### overlayfs mechanics — learn these before Lesson 61

`mount -t overlay -o lowerdir=L,upperdir=U,workdir=W overlay /merged`:
reads fall through U→L (upper wins); **writes copy-up** the file from L to U
first (whole file, at open-for-write — a 4 GB file edited by one byte copies
4 GB); **deletes** of lower files create *whiteouts* in U (a marker character
device saying "pretend this doesn't exist" — the lower file is untouched!).
The lower layer is never modified: N containers share one image layer
read-only, each with a private tiny upper — Lesson 61's entire economics.

{: .note }
> **Choosing, honestly**
> Boring default: <strong>ext4</strong>. Big parallel servers / RHEL-land:
> <strong>xfs</strong>. Snapshots/checksums as a workflow (rollbackable
> updates, cheap backups) and you accept the care: <strong>btrfs</strong>.
> Ephemeral: <strong>tmpfs</strong>. Layered images: <strong>overlayfs</strong>
> (you rarely mount it by hand — your container runtime does). The realistic
> skill is recognizing which you're on (<code>findmnt -T .</code>) and what
> that implies — not filesystem partisanship.

---

## Lab

```bash
# ---- 1. What are YOU running? ----
$ findmnt -no FSTYPE / /tmp /dev/shm /run
# ext4 (or xfs/btrfs) / tmpfs×3 — count how much of your VM is RAM-backed!

# ---- 2. tmpfs: RAM speed, RAM cost, swap truth ----
$ sudo mkdir -p /mnt/ram && sudo mount -t tmpfs -o size=256M tmpfs /mnt/ram
$ dd if=/dev/zero of=/mnt/ram/speed bs=1M count=200 2>&1 | tail -1
# multi-GB/s — it's a memcpy into page cache
$ grep -E 'Shmem:' /proc/meminfo          # tmpfs usage shows as Shmem
$ dd if=/dev/zero of=/mnt/ram/full bs=1M count=100 2>&1 | tail -1
# "No space left" at the size= cap — tmpfs enforces its budget (RAM protection)
$ sudo umount /mnt/ram                     # contents GONE. That's the contract.

# ---- 3. overlayfs by hand: the container illusion, demystified ----
$ mkdir -p /tmp/ov/{lower,upper,work,merged}
$ echo "from the IMAGE"    > /tmp/ov/lower/base.txt
$ echo "shared library v1" > /tmp/ov/lower/lib.txt
$ sudo mount -t overlay overlay \
    -o lowerdir=/tmp/ov/lower,upperdir=/tmp/ov/upper,workdir=/tmp/ov/work \
    /tmp/ov/merged
$ ls /tmp/ov/merged/                       # both files: the MERGED view
$ cat /tmp/ov/merged/base.txt              # reads fall through to lower

# write → copy-up:
$ echo "container modified this" >> /tmp/ov/merged/base.txt
$ cat /tmp/ov/lower/base.txt               # "from the IMAGE" — UNTOUCHED
$ cat /tmp/ov/upper/base.txt               # full modified copy lives here
# delete → whiteout:
$ rm /tmp/ov/merged/lib.txt
$ ls /tmp/ov/merged/                       # lib.txt gone from the view...
$ ls -l /tmp/ov/upper/                     # c--------- 0,0 lib.txt ← WHITEOUT
$ ls /tmp/ov/lower/                        # ...but lib.txt SAFE in lower
$ sudo umount /tmp/ov/merged

# ---- 4. btrfs: CoW snapshots in 60 seconds (loop device) ----
$ if command -v mkfs.btrfs >/dev/null; then
    dd if=/dev/zero of=/tmp/btr.img bs=1M count=300 2>/dev/null
    mkfs.btrfs -q /tmp/btr.img && mkdir -p /tmp/btr
    sudo mount -o loop /tmp/btr.img /tmp/btr && sudo chown $USER /tmp/btr
    sudo btrfs subvolume create /tmp/btr/data
    echo "version 1 of everything" | sudo tee /tmp/btr/data/file >/dev/null
    sudo btrfs subvolume snapshot /tmp/btr/data /tmp/btr/snap-before  # O(1)!
    echo "version 2 CHANGED" | sudo tee /tmp/btr/data/file >/dev/null
    echo "--- live:";     cat /tmp/btr/data/file
    echo "--- snapshot:"; cat /tmp/btr/snap-before/file   # v1, preserved free
    sudo btrfs filesystem usage /tmp/btr 2>/dev/null | head -5
    sudo umount /tmp/btr && rm /tmp/btr.img
  else echo "no btrfs-progs — apt install btrfs-progs to run this leg"; fi
# the snapshot cost no copying and no meaningful space — CoW's magic trick,
# and exactly what qcow2 backing chains did in virt Lesson 23, now in the fs.

# ---- 5. Spot overlayfs in the wild (if docker/podman present) ----
$ docker info 2>/dev/null | grep -i storage || true
# Storage Driver: overlay2      ← lab 3 is what it does for EVERY container
$ mount | grep -m2 overlay || echo "start a container and re-check"
$ rm -rf /tmp/ov
```

---

## Further Reading

| Topic | Link |
|---|---|
| XFS (Wikipedia) | <https://en.wikipedia.org/wiki/XFS> |
| Btrfs (Wikipedia) | <https://en.wikipedia.org/wiki/Btrfs> |
| tmpfs — kernel docs | <https://www.kernel.org/doc/html/latest/filesystems/tmpfs.html> |
| Overlayfs — kernel docs | <https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html> |
| Copy-on-write | <https://en.wikipedia.org/wiki/Copy-on-write> |
| OverlayFS (Wikipedia) | <https://en.wikipedia.org/wiki/OverlayFS> |

---

## Checkpoint

**Q1.** In your hand-built overlay, you deleted a file that lives in the
lower layer. What actually happened on disk, and why is this design forced by
the container use case?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Nothing in the lower layer changed — it's mounted read-only by contract.
overlayfs recorded the deletion as a <strong>whiteout</strong> in the upper
layer: a 0/0 character-device entry with the file's name, which the merged
view interprets as "this name is deleted — do not fall through to lower."
The design is forced because the lower layer is <em>shared</em>: fifty
containers run from one image layer, and container A deleting
/usr/lib/libfoo must not affect containers B–Z (nor the image itself, which
must stay content-addressed and immutable — Lesson 37's homework logic).
Every mutation therefore must be representable purely in the private upper
layer — modifications as copied-up files, deletions as whiteouts, new
directories-atop-deleted-ones as "opaque" markers. Corollary worth knowing:
deleting files in a container <em>frees no image space</em> (the lower blocks
remain) and even <em>consumes</em> a little (the whiteout) — the root of the
"deleted 2 GB inside the container, disk usage grew" surprise, and of the
Dockerfile rule to clean up within the same layer that created the mess.
</details>

**Q2.** A database's data files fragment horribly on btrfs but not on ext4 —
while btrfs snapshots of that same database are instant. Explain both from
the one CoW mechanism, and name the two standard mitigations.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
One mechanism, both faces. btrfs never overwrites in place: each rewrite of
a database page (random 8–16 KB updates, thousands per second) allocates a
<em>new</em> block elsewhere and abandons the old — a file physically
contiguous at creation becomes scattered across the device at the rate of
its update traffic: CoW-induced fragmentation (and extra metadata churn per
write). ext4 rewrites the same blocks in place — layout stable forever. But
that same never-overwrite property is why snapshots cost nothing: the old
pointer tree simply survives (no blocks need copying — they were never going
to be overwritten anyway), sharing all unmodified data with the live volume.
Mitigations: (1) <code>chattr +C</code> (nodatacow) on the database
directory — those files overwrite in place like ext4 (surrendering their
checksumming and making their snapshots cost real copies); (2) periodic
<code>btrfs filesystem defragment</code> (with the caveat that defragging
snapshotted files un-shares blocks and can explode space usage). Deeper
lesson: CoW is a <em>global</em> design choice — you inherit the whole
bundle, so match data classes to filesystems (Lesson 38's homework
conclusion, again).
</details>

**Q3.** Why does tmpfs data end up in *swap* under memory pressure, and is
"tmpfs plus swap" still faster than just using ext4 for the same scratch
files? Reason it through the memory phase.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
tmpfs pages have no file behind them — they're effectively anonymous memory
wearing filenames (Lesson 21's taxonomy). Under pressure, reclaim can't just
drop them (they're the only copy — dropping clean page-cache pages works
because disk has the truth; here there is no disk truth), so the only
eviction path is swapping them out like any anonymous page. Is it still a
win vs ext4 scratch files? Usually yes, on three counts: (1) the happy path —
uncontended RAM — never touches storage at all, while ext4 scratch files
generate writeback I/O even when RAM was plentiful (dirty-page flushing,
journal commits, metadata — Lessons 20/38 costs for data that will be
deleted in seconds); (2) under pressure, tmpfs pages are swapped
<em>selectively and lazily</em> (only the cold ones, only when needed),
roughly matching what ext4's page cache would do anyway; (3) deletion is
free (drop pages) vs ext4's metadata work. The honest caveat: a workload
that <em>always</em> exceeds RAM makes tmpfs into swap-thrash (Lesson 21) —
worse than ext4's sequential writeback; and swapless boxes turn the excess
into OOM pressure (Lesson 22). Rule: tmpfs for scratch that <em>usually</em>
fits in RAM; real fs for scratch that usually doesn't.
</details>

---

## Homework

Docker's `overlay2` driver stacks up to 128 `lowerdir`s (one per image
layer). Using lab 3's mechanics, explain: (a) what a `Dockerfile`'s
`RUN rm -rf /var/cache/apt` in a *later* layer actually achieves (does the
image shrink?); (b) why `docker commit` of a long-running container can
produce a bloated layer; (c) why the first write to a large file in a
container sometimes shows a mysterious latency spike — and which mount
option (or file layout choice) image authors use to avoid shipping that
spike to users.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) Nothing shrinks. The rm creates whiteouts in the new layer; the cached
files still occupy their original layer in full (layers are immutable and
content-addressed — Lesson 37's homework again). The image gains whiteout
metadata and loses no bytes: cleanup must happen <em>in the same RUN</em>
that created the files (one layer = create + delete = files never enter any
shipped layer). This single fact explains most "why is my image 2 GB"
audits. (b) docker commit snapshots the container's upper layer — everything
the container ever wrote: logs, temp files, package caches, copied-up
versions of every file it touched (each copy-up duplicates a lower file into
the layer even for a one-byte change!). A long-running container's upper is
an archaeological record, and commit ships all of it — the reason images are
built from Dockerfiles (controlled, minimal layers), not committed from pets.
(c) The spike is <strong>copy-up</strong>: opening a lower-layer file for
write copies the whole file to upper before the write proceeds — a one-byte
append to a 2 GB lower file costs a 2 GB copy, once, at first-write. Image
authors avoid shipping the trap by keeping mutable-by-design files out of
image layers entirely: declare VOLUMEs or bind-mount data directories
(volumes bypass overlayfs — direct fs, no copy-up ever), or pre-create
empty/small files in the final layer so any copy-up is trivial. (Metadata-
only changes like chmod also historically triggered copy-up; the
metacopy=on option reduced that class.)
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 40 — Mounts and Bind Mounts →](lesson-40-mounts-bind){: .btn .btn-primary }
