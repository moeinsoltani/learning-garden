---
title: "Lesson 26 — Snapshots"
nav_order: 26
parent: "Phase 6: Storage"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 26: Snapshots

## Concept

A **snapshot** captures a VM's state so you can roll back to it — before a risky
upgrade, for testing, for quick recovery. There are two orthogonal distinctions:

```
   WHERE the snapshot lives                 WHAT it captures
   ─────────────────────────                ──────────────────
   INTERNAL  — inside one qcow2 file        DISK-ONLY  — just the disk
              (savevm / qemu-img snapshot)  FULL       — disk + RAM + CPU
   EXTERNAL  — a new overlay file                       (a complete moment,
              (backing chain, Lesson 23)                 resumes mid-run)
```

Internal snapshots tuck multiple point-in-time states *inside* a single qcow2.
External snapshots freeze the current file read-only and redirect new writes to a
fresh overlay (the copy-on-write chain from Lesson 23). Production tooling generally
prefers **external**.

---

## How It Works

### Internal snapshots

Stored *within* a qcow2 using its refcount/snapshot tables. Managed with
`qemu-img snapshot` (offline) or the monitor `savevm`/`loadvm` (live, can include
RAM). A single file can hold many named snapshots.

- Pros: one file to manage, simple to create.
- Cons: all snapshots share one file (can't easily move/back-up individual ones);
  performance degrades with many snapshots; deleting/merging is slower; only works
  with qcow2.

```
   qemu-img snapshot -c before-upgrade disk.qcow2   # create
   qemu-img snapshot -l disk.qcow2                  # list
   qemu-img snapshot -a before-upgrade disk.qcow2   # revert (apply)
   qemu-img snapshot -d before-upgrade disk.qcow2   # delete
```

### External snapshots

Take the **live** approach: the current disk becomes a read-only **backing file** and
QEMU starts writing to a *new overlay*. The snapshot is "the state captured in the
now-frozen base." This is just the backing-chain mechanism from Lesson 23, applied at
a moment in time. Done live via QMP `blockdev-snapshot` / `snapshot-create`.

```
   before:  disk.qcow2  (active, being written)
   snapshot:
       disk.qcow2  (frozen, read-only)  ◄─backing─  overlay.qcow2 (now active)
```

- Pros: each piece is a separate file (easy to copy/back-up/move); the frozen base is
  a clean restore point; the active overlay can later be **merged back** with
  `blockdev-commit`. The standard for backups and production.
- Cons: chains can grow long (sprawl), and reads traverse the chain (slight cost).

### Disk-only vs full (disk + RAM)

- **Disk-only:** captures just the block device contents. On revert you get the disk
  at that moment, but the VM is *off/reset* (like a power-cut restore). Fast, small.
  For crash-consistency, quiesce the guest filesystem first (e.g. via the guest agent
  freezing/fsyncing).
- **Full (system) snapshot:** captures disk **plus** RAM and CPU state, so you can
  resume *exactly* where you were, mid-application, as if time paused. Larger
  (includes memory) and requires saving the VM state.

### Live snapshots via QMP

For running VMs, you don't stop the world: `blockdev-snapshot-sync` /
`blockdev-snapshot` create an external overlay on the fly, and `blockdev-commit`
merges an overlay back into its base while the VM runs (online block-commit). This is
how backup tools snapshot, copy the frozen base, then commit — with negligible
downtime.

### Pitfalls

- **Snapshot sprawl:** dozens of forgotten snapshots/overlays bloat storage and slow
  I/O. Prune them.
- **Performance:** long internal-snapshot histories or deep external chains add
  overhead; flatten/commit periodically.
- **Snapshot ≠ backup:** a snapshot on the same disk dies with the disk. Real backups
  copy data off-host (Lesson 47).

{: .note }
> **Why production prefers external snapshots**
> External snapshots produce discrete files: the frozen backing file is an immutable
> restore point you can copy or ship off-host, and the active overlay is mergeable
> online with blockdev-commit. That maps cleanly onto backup workflows (snapshot →
> copy the frozen base → commit) and onto live operations with minimal downtime.
> Internal snapshots bury everything in one qcow2 — convenient for a quick local
> rollback, but harder to back up granularly and slower to manage at scale.

---

## Lab

```bash
# ---- INTERNAL snapshot (offline, with qemu-img) ----
$ qemu-img snapshot -c clean-install disk.qcow2     # create named snapshot
$ qemu-img snapshot -l disk.qcow2                    # list
Snapshot list:
ID   TAG            VM-SIZE   DATE         VM-CLOCK
1    clean-install      0 B   2026-06-20   00:00:00.000
# ...make changes (boot VM, break something)...
$ qemu-img snapshot -a clean-install disk.qcow2      # revert to it
$ qemu-img snapshot -d clean-install disk.qcow2      # delete it

# ---- EXTERNAL live snapshot via QMP ----
# Start a VM exposing QMP, with the active disk as node 'hd0':
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -blockdev driver=qcow2,node-name=hd0,file.driver=file,file.filename=disk.qcow2 \
    -device virtio-blk-pci,drive=hd0 -nographic \
    -qmp unix:/tmp/qmp.sock,server,nowait &

# Pre-create the overlay, then snapshot live (freezes disk.qcow2, writes go to overlay):
$ qemu-img create -f qcow2 -b disk.qcow2 -F qcow2 overlay.qcow2
$ qmp-shell /tmp/qmp.sock
(QEMU) blockdev-add driver=qcow2 node-name=ov \
        file={"driver":"file","filename":"overlay.qcow2"} backing=hd0
(QEMU) blockdev-snapshot node=hd0 overlay=ov
# now the VM writes to 'ov'; disk.qcow2 is the frozen restore point.

# ---- Merge the overlay back into the base (online block-commit) ----
(QEMU) block-commit device=ov            # streams overlay -> backing, then pivots
# (watch for BLOCK_JOB_READY, then:)
(QEMU) block-job-complete device=ov
(QEMU) quit

# ---- FULL snapshot (disk + RAM) via the monitor ----
# (HMP) savevm mysnap     # saves CPU+RAM+disk state into the qcow2
# (HMP) loadvm mysnap     # resume exactly there
```

**Expected result:** `qemu-img snapshot -l` lists internal snapshots inside one file;
the QMP `blockdev-snapshot` flow freezes the base and redirects writes to a new
overlay live; `block-commit` merges it back — all without stopping the VM.

---

## Further Reading

| Topic | Link |
|---|---|
| QEMU live block operations / snapshots | [qemu.org — Live block operations](https://www.qemu.org/docs/master/interop/live-block-operations.html) |
| `qemu-img snapshot` | [qemu.org — qemu-img](https://www.qemu.org/docs/master/tools/qemu-img.html) |
| Snapshot (computer storage) | [Wikipedia — Snapshot (computer storage)](https://en.wikipedia.org/wiki/Snapshot_(computer_storage)) |
| QMP blockdev-snapshot | [qemu.org — QMP reference](https://www.qemu.org/docs/master/interop/qemu-qmp-ref.html) |
| Copy-on-write (chains) | [Wikipedia — Copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What is the difference between an internal and an external snapshot, and why do production tools usually prefer external?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An internal snapshot stores point-in-time state *inside* a single qcow2 file (via its snapshot/refcount tables). An external snapshot freezes the current file read-only as a backing file and redirects new writes to a separate overlay file (the copy-on-write chain). Production prefers external because each state is a discrete file: the frozen base is an immutable restore point you can copy/ship off-host for backup, the overlay can be merged back online with block-commit, and it integrates cleanly with backup workflows and live operations. Internal snapshots bury everything in one file — fine for quick local rollback but harder to back up granularly and slower to manage at scale.
</details>

---

**Q2. What extra state does a "full" (system) snapshot capture compared to a disk-only snapshot, and what does that let you do on revert?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A full/system snapshot captures the VM's RAM and CPU state in addition to the disk; a disk-only snapshot captures just the block device contents. On revert, a full snapshot lets you resume the VM exactly where it was — mid-application, as if time had paused — because memory and CPU registers are restored. Reverting a disk-only snapshot gives you the disk at that moment but the VM is off/reset (like a power-cut restore), so you boot fresh from that disk state. Full snapshots are larger because they include memory.
</details>

---

**Q3. Why is a snapshot not the same as a backup?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A snapshot is a point-in-time view that typically lives on (or depends on) the same storage as the live disk — an internal snapshot is inside the same qcow2, and an external overlay's restore point is a file alongside it. If that disk/host fails or is corrupted/deleted, the snapshot is lost too. A backup copies data to independent, off-host storage so it survives failure of the original. Snapshots are great for fast local rollback and as a *source* for backups, but you must copy the data elsewhere to have a real backup (Lesson 47).
</details>

---

## Homework

On a running VM, take an external live snapshot via QMP `blockdev-snapshot` so writes go to `overlay.qcow2`. Make a change in the guest, then use `block-commit` to merge the overlay back into the base. Verify with `qemu-img info --backing-chain` before and after. Explain what the chain looked like at each step and what `block-commit` accomplished without downtime.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Before the snapshot, the chain is just <code>disk.qcow2</code> (active, no backing). After blockdev-snapshot, <code>--backing-chain</code> shows two layers: <code>overlay.qcow2</code> (active) → backing <code>disk.qcow2</code> (frozen restore point); the guest's new change lands only in overlay.qcow2. <code>block-commit</code> then streams the overlay's clusters down into disk.qcow2 and pivots the active layer back to the base, so afterward the chain is again just <code>disk.qcow2</code> — now containing the merged change — and the overlay can be removed. It did this online: the VM kept running throughout, with QEMU performing the merge in the background and pivoting atomically, so there was negligible/zero downtime.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 27 — The virtio Standard →](lesson-27-virtio-standard){: .btn .btn-primary }
