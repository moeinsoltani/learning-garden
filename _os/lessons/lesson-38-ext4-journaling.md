---
title: "Lesson 38 — ext4 and Journaling"
nav_order: 3
parent: "Phase 7: Files & Filesystems"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 38: ext4 and Journaling

## Concept

The VFS gave every filesystem one interface; now descend into the most common
implementation. **ext4** is what your VM's root almost certainly runs, and its
on-disk layout answers the question Lesson 37 left open: where do inodes and
data actually *live*?

```
  the block device, carved up by mkfs.ext4:

  ┌────────────┬──────────────────────────────────────────────────┐
  │ superblock │  block group 0   block group 1   block group 2 … │
  │ (fs-wide:  │ ┌──────────────┐┌──────────────┐                 │
  │  size,     │ │ block bitmap ││ …            │  each group:    │
  │  features, │ │ inode bitmap ││              │  its own inodes │
  │  state)    │ │ INODE TABLE  ││              │  + data blocks, │
  │            │ │ DATA BLOCKS  ││              │  kept together  │
  └────────────┴─┴──────────────┴┴──────────────┴─────────────────┘
                 └── locality: a file's inode and data in one
                     group when possible (seek economy — a
                     spinning-disk legacy that still helps SSDs)
```

Modern ext4 tracks a file's blocks as **extents** — `(start block, length)`
runs — instead of ext2/3's per-block pointer lists: one extent can describe
128 MiB, so a 10 GB file needs dozens of entries, not millions. Directories
are the name→inode tables from Lesson 37, hash-indexed for large counts.

And the headline feature — the **journal** — answers the crash question.
A single logical operation (append to a file) touches multiple blocks: the
data, the inode (size, mtime), the block bitmap, maybe the directory. A crash
between those writes leaves them *inconsistent* — the 1990s world of long
`fsck` runs reconstructing sanity. Journaling makes multi-block updates
atomic: **write the changes to a log first, then to their real places.**
Crash anywhere → replay or discard the log; the filesystem is *always*
consistent within seconds of mounting.

---

## How It Works

### The journal protocol

A transaction: (1) write the intended changes into the journal (a reserved
on-disk ring); (2) write a **commit record** — the transaction is now
official; (3) later, **checkpoint**: write the changes to their home
locations; (4) reclaim the journal space. Crash before commit → the partial
transaction is ignored (old state intact). Crash after commit, before/during
checkpoint → replay the journal on mount (new state completed). The commit
record is the atomic pointer-flip — the same shape as Lesson 37's rename
pattern and every database's WAL. (Databases *are* this design; ext4's jbd2
is a small WAL engine.)

### The three modes — and what's actually protected

- `data=ordered` (**the default**): only *metadata* goes through the journal,
  but data blocks are forced to disk *before* the metadata that references
  them commits. Guarantee: the fs structure is always consistent, and
  metadata never points at garbage — but a file being overwritten in place
  can still end up half-old/half-new after a crash.
- `data=journal`: data too goes through the journal — everything written
  twice: safest, slowest, rare.
- `data=writeback`: metadata journaled, data whenever — after a crash, files
  can contain stale garbage blocks (metadata pointed before data landed).
  Fast; only for data you can regenerate.

The sentence that prevents years of confusion: **journaling protects the
filesystem, not your file contents.** A crash mid-`write()` still loses/tears
application data (Lesson 20's dirty pages, Lesson 41's fsync are the
application-side story). What you're spared: the *filesystem itself*
corrupting, and hour-long fscks.

### fsck and the superblock

`e2fsck` still exists for real damage (hardware lies, bugs): it walks bitmaps
vs reality, reconnects orphans into `lost+found` (inodes with no name —
Lesson 37's ghosts, found homeless after a crash). The superblock is
precious enough to be replicated across block groups (`mkfs` prints where;
`e2fsck -b 32768` recovers from a dead primary).

{: .note }
> **tune2fs and dumpe2fs: the fs's control panel**
> <code>dumpe2fs -h /dev/X</code> — every superblock fact (features, journal
> size, mount count); <code>tune2fs</code> — adjust reserved blocks (the 5%
> root reserve on / that "disk 100% full but root can still write" comes
> from!), fsck intervals, labels. Worth a tour on any real system.

---

## Lab

```bash
# ---- 0. Build a sacrificial ext4 on a loop device (no risk to your VM) ----
$ dd if=/dev/zero of=/tmp/fs.img bs=1M count=256 2>/dev/null
$ mkfs.ext4 -q /tmp/fs.img && mkdir -p /tmp/mnt && sudo mount -o loop /tmp/fs.img /tmp/mnt
$ sudo chown $USER /tmp/mnt

# ---- 1. Tour the superblock ----
$ dumpe2fs -h /tmp/fs.img 2>/dev/null | grep -E 'Filesystem features|Block size|Inode count|Journal|state'
# features: has_journal ext_attr ... extent 64bit    ← the feature flags
# Block size: 1024/4096, Inode count: 65536 (df -i's budget, fixed at mkfs!)

# ---- 2. Watch extents describe a file ----
$ dd if=/dev/urandom of=/tmp/mnt/big bs=1M count=64 2>/dev/null
$ filefrag -v /tmp/mnt/big | head -8
#  ext:  logical_offset:  physical_offset: length:
#    0:      0..   16383:  34816..  51199:  16384:      ← ONE extent, 64MB
# a few extents for 64MB — vs 16384 block pointers in ext2's scheme

# ---- 3. debugfs: read the filesystem like the kernel does ----
$ echo "journaled world" > /tmp/mnt/hello.txt && sync
$ sudo debugfs -R "stat /hello.txt" /tmp/fs.img 2>/dev/null | head -12
# Inode: 12  Type: regular  Mode: 0644  Links: 1
# EXTENTS: (0): 8465                      ← the actual on-disk block!
$ sudo debugfs -R "ls -l /" /tmp/fs.img 2>/dev/null
# the ROOT DIRECTORY as a table: name → inode, exactly Lesson 37's picture
$ sudo debugfs -R "cat /hello.txt" /tmp/fs.img 2>/dev/null
# journaled world      ← read via fs structures, no mount involved

# ---- 4. The journal, caught in the act ----
$ sudo debugfs -R "stat <8>" /tmp/fs.img 2>/dev/null | head -4
# inode 8 IS the journal on ext4 — a hidden file holding the transaction ring
$ for i in $(seq 50); do echo "txn $i" > /tmp/mnt/f$i; done; sync
$ sudo debugfs -R "logdump -S" /tmp/fs.img 2>/dev/null | tail -8
# Found sequence N: commit block / descriptor block …  ← real transactions!

# ---- 5. Crash simulation: journaling earns its keep ----
$ for i in $(seq 20); do echo "precious $i" > /tmp/mnt/keep$i; done; sync
$ echo "doomed data" > /tmp/mnt/doomed        # NO sync — dirty in RAM only
$ sudo umount -l /tmp/mnt 2>/dev/null; sudo dmsetup remove_all 2>/dev/null
# harsher: yank via forced detach (simulates power loss for the loop dev)
$ sudo mount -o loop /tmp/fs.img /tmp/mnt
$ dmesg | tail -2 | grep -i ext4
# "recovery complete" or clean mount — NO fsck ordeal, seconds not hours
$ ls /tmp/mnt | tail -3; cat /tmp/mnt/doomed 2>/dev/null || echo "doomed: lost (never synced)"
# keep1..keep20 intact (they were synced); doomed lost or empty —
# THE lesson: fs consistent, unfsynced DATA not guaranteed. (L41 finishes this.)

# ---- 6. fsck knows the truth ----
$ sudo umount /tmp/mnt
$ sudo e2fsck -f /tmp/fs.img
# Pass 1..5: clean. Try it after 'dd if=/dev/urandom of=/tmp/fs.img seek=100
# count=4 conv=notrunc' someday — watch it reconstruct and fill lost+found.
$ rm /tmp/fs.img
```

---

## Further Reading

| Topic | Link |
|---|---|
| ext4 (Wikipedia) | <https://en.wikipedia.org/wiki/Ext4> |
| Journaling file system | <https://en.wikipedia.org/wiki/Journaling_file_system> |
| ext4 — kernel documentation | <https://www.kernel.org/doc/html/latest/admin-guide/ext4.html> |
| `debugfs(8)` man page | <https://man7.org/linux/man-pages/man8/debugfs.8.html> |
| `dumpe2fs(8)` / `tune2fs(8)` | <https://man7.org/linux/man-pages/man8/tune2fs.8.html> |
| `e2fsck(8)` man page | <https://man7.org/linux/man-pages/man8/e2fsck.8.html> |

---

## Checkpoint

**Q1.** Journaling protects filesystem metadata. Does it protect the contents
of your half-written file? What *does*, and what exact guarantee does the
default `data=ordered` add about data?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
No. The journal makes multi-block <em>metadata</em> updates atomic — inode,
bitmaps, directories always consistent after a crash — but your file's
contents get no transactional treatment: a crash mid-overwrite leaves
whatever mix of old and new bytes had reached disk (dirty pages that never
flushed are simply gone — the lab's "doomed" file). Application data
durability is the application's job: fsync at chosen points (Lesson 41), or
the write-fsync-rename pattern (Lesson 37) for whole-file atomicity — i.e.,
you build your own transaction on top. What data=ordered does promise: data
blocks referenced by a committing transaction are flushed <em>before</em> the
metadata commit — so you'll never see a file whose metadata says "these
blocks are yours" while the blocks still hold someone's deleted data
(a security hole, not just corruption) or unwritten garbage. Ordered mode's
slogan: metadata never lies about data — but data itself is only as durable
as your last fsync.
</details>

**Q2.** Explain why the journal makes crash *recovery* take seconds instead
of the hours a full fsck needs — what does each approach have to examine?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
fsck's problem is that without a journal, a crash leaves no record of
<em>what was in flight</em> — any block anywhere might be inconsistent, so
correctness requires examining <em>everything</em>: walk every inode, verify
every bitmap bit against actual usage, count every directory entry's link,
reconnect orphans — time proportional to filesystem <em>size</em> (terabytes
= hours, while the service is down). The journal inverts the knowledge: all
in-flight changes are, by protocol, recorded in one small ring (typically
128 MB) before touching their home locations. Recovery reads just the
journal: transactions with commit records → replay (idempotent block
rewrites); without → discard. Time proportional to <em>journal size</em> —
seconds, regardless of fs size. The general principle (same as database WAL
recovery vs rebuilding from backups): bounded uncertainty. Design systems so
that after any crash, the set of things possibly-wrong is small and known —
then recovery is reading that set, not auditing the world.
</details>

**Q3.** `df -h` says 2 GB free but a non-root service gets ENOSPC on the root
filesystem. Two distinct explanations from this phase — and the command that
distinguishes them.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <strong>Reserved blocks</strong>: ext4 reserves ~5% for root (tune2fs -l
shows "Reserved block count") so the system stays operable — root can write
logs, admins can log in and clean up — even when "full". df's "free" on some
systems includes reservation the service can't touch: at 2 GB free on a
~40 GB fs, ordinary users are already at their wall. Check: <code>tune2fs
-l</code> reserved count vs df numbers (df's "Avail" column actually shows
non-root availability — compare Avail vs Free). (2) <strong>Inode
exhaustion</strong>: blocks free but the fixed inode table is spent (millions
of small files); every create fails ENOSPC while df -h looks healthy. Check:
<code>df -i</code> — IUse% at 100%. Distinguisher in one line: <code>df -h;
df -i</code> side by side — space-shaped ENOSPC vs identity-shaped ENOSPC.
(Bonus third from Lesson 36/37: the space is "free" but deleted-open files
hold it — df vs du disagreement, lsof +L1 to convict.)
</details>

---

## Homework

Databases famously recommend turning *off* the filesystem's journal for
their data files (or using data=writeback), while keeping full journaling
for the system. Using the WAL parallel from this lesson, argue their case:
what double-work happens when a journaling database runs on a journaling
filesystem, which of the two journals is redundant *for the data files*, and
why must the argument NOT be extended to the database's own binaries and
config?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The database already implements the full protocol on its data files: every
change goes to its WAL first (with its own commit records and fsync
discipline — Lesson 41), then to the data pages; crash recovery replays the
WAL. Running that on data=journal ext4 writes everything <em>four</em> times
(DB WAL, DB pages, fs journal copy of both) — and even data=ordered adds
ordering constraints and journal commits the DB never asked for, costing
IOPS and fsync latency on the hottest path. For the data files, the
filesystem journal is redundant <em>because the database's crash-consistency
contract is stronger and self-contained</em>: it only requires that fsync be
honest and that the fs not corrupt its own structure — metadata journaling
(which writeback keeps!) provides the latter. So: data=writeback (or
separate mount options for the data volume) removes the duplicate layer
while metadata stays protected. The argument dies outside the WAL's reach:
binaries, configs, log directories have no application-level journal — for
them a crash without fs data protection means the stale-garbage-in-files
failure mode, and no recovery protocol exists to fix it. Hence the actual
deployment pattern: dedicated mount (own options, own device) for DB data;
default-journaled fs for the OS — policy per data class, mechanism per
mount, which is exactly what the next two lessons build toward.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 39 — The Filesystem Zoo →](lesson-39-filesystem-zoo){: .btn .btn-primary }
