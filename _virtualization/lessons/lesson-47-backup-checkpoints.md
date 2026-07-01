---
title: "Lesson 47 — Domain Snapshots, Checkpoints, and Incremental Backup"
nav_order: 47
parent: "Phase 12: Snapshots, Backup & Migration"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 47: Domain Snapshots, Checkpoints, and Incremental Backup

## Concept

Phase 6 covered snapshots at the *QEMU/qcow2* level. libvirt adds management-layer
operations: **domain snapshots**, **checkpoints** (which track *what changed* via dirty
bitmaps), and a **backup API**. The crucial new idea is the **dirty bitmap** — a record
of which disk blocks changed since a point in time, enabling **incremental backup**.

```
   FULL BACKUP every night            INCREMENTAL via dirty bitmap
   ────────────────────────           ────────────────────────────
   copy ALL 100 GiB nightly            copy only the blocks that
   (slow, huge, wasteful)              CHANGED since last backup
                                       (e.g. 2 GiB) — fast, small
   bitmap tracks: ▓ = changed,  ░ = unchanged since checkpoint
   [░░░▓░░▓▓░░░░▓░░] → back up only the ▓ blocks
```

A dirty bitmap lets a backup tool ask "what changed since yesterday?" and copy only that
— the difference between re-copying everything and copying a delta.

---

## How It Works

### Domain snapshots (libvirt layer)

`virsh snapshot-create-as` creates a snapshot of a domain — **disk-only** or **full
system** (disk + RAM), internal or external (the Phase 6 distinction, now driven through
libvirt):

```
   virsh snapshot-create-as web01 snap1 --disk-only --atomic
   virsh snapshot-list web01
   virsh snapshot-revert web01 snap1
   virsh snapshot-delete web01 snap1
```

These are for quick rollback points. They are **not** backups (Lesson 26) — they live on
the same host/storage.

### Checkpoints and dirty bitmaps

A **checkpoint** marks a point in time and starts a **dirty bitmap** that tracks every
block written *after* it. QEMU maintains the bitmap as the guest runs. Later you can ask
for "all blocks dirtied since checkpoint X" — exactly the data an incremental backup
needs. Checkpoints can be chained (checkpoint per backup), so each incremental captures
the delta since the previous one.

### The backup API: full and incremental

libvirt's **`virsh backup-begin`** (using QEMU's backup/`blockdev-backup` machinery)
performs a backup of a *running* VM:

- **Full backup:** copy the entire disk (optionally creating a checkpoint to anchor future
  incrementals).
- **Incremental backup:** using the dirty bitmap from the last checkpoint, copy **only the
  changed blocks** since then. Fast and small.

```
   virsh backup-begin web01 backup.xml      # backup.xml specifies full or incremental
   virsh domjobinfo web01                    # watch progress
```

The backup runs **online** (no downtime) and produces a consistent point-in-time image,
because QEMU coordinates the copy with the dirty-bitmap state.

### Snapshot vs backup vs replication — different jobs

| Job | What it is | Survives host loss? |
|---|---|---|
| **Snapshot** | Fast local rollback point (same storage) | No |
| **Backup** | Copy of data to **independent** storage, often incremental | Yes |
| **Replication** | Continuous copy to another host/site for failover | Yes (near-real-time) |

They're complementary: snapshots for quick "undo," backups for disaster recovery,
replication for high availability. A snapshot is *not* a backup; a backup is *not* HA.

{: .note }
> **What a dirty bitmap buys you**
> Without one, the only way to know what changed is to compare or re-copy the whole disk —
> so "incremental" backup degenerates into a full copy every time (slow, huge, heavy I/O).
> A dirty bitmap records, live and cheaply, exactly which blocks were written since a
> checkpoint, so a backup tool copies only the delta — turning a nightly 100 GiB full copy
> into, say, a 2 GiB incremental. It's the mechanism that makes efficient, frequent
> backups of large running VMs practical.

---

## Lab

```bash
# ---- libvirt domain snapshots (quick rollback) ----
$ virsh snapshot-create-as web01 before-upgrade --disk-only --atomic
Domain snapshot before-upgrade created
$ virsh snapshot-list web01
 Name             Creation Time               State
------------------------------------------------------
 before-upgrade   2026-06-20 10:00:00 +0000   disk-snapshot
$ virsh snapshot-revert web01 before-upgrade     # roll back
$ virsh snapshot-delete web01 before-upgrade

# ---- Full backup with a checkpoint to anchor future incrementals ----
$ cat > full-backup.xml <<'EOF'
<domainbackup mode='pull'>
  <disks><disk name='vda' backup='yes'/></disks>
</domainbackup>
EOF
$ cat > checkpoint.xml <<'EOF'
<domaincheckpoint><name>cp0</name></domaincheckpoint>
EOF
$ virsh backup-begin web01 full-backup.xml checkpoint.xml
Backup started
$ virsh domjobinfo web01            # watch it
$ virsh checkpoint-list web01
 Name   Creation Time
-----------------------------------
 cp0    2026-06-20 10:05:00 +0000

# ---- Later: INCREMENTAL backup of only blocks changed since cp0 ----
$ cat > incr-backup.xml <<'EOF'
<domainbackup mode='pull'>
  <incremental>cp0</incremental>
  <disks><disk name='vda' backup='yes'/></disks>
</domainbackup>
EOF
$ virsh backup-begin web01 incr-backup.xml
# Only blocks dirtied since checkpoint cp0 are transferred — fast and small.

# ---- Inspect the dirty bitmap QEMU maintains for the disk ----
$ qemu-img info --output=json /var/lib/libvirt/images/web01.qcow2 \
    | grep -A4 bitmaps
```

**Expected result:** Domain snapshots give quick rollback points; a full `backup-begin`
with a checkpoint anchors a dirty bitmap, and a subsequent incremental backup transfers
only the blocks changed since that checkpoint — visibly far less data than a full copy.

---

## Further Reading

| Topic | Link |
|---|---|
| libvirt snapshots | [libvirt.org — Snapshots](https://libvirt.org/formatsnapshot.html) |
| libvirt checkpoints & incremental backup | [libvirt.org — Checkpoints](https://libvirt.org/formatcheckpoint.html) |
| QEMU dirty bitmaps | [qemu.org — Dirty bitmaps](https://www.qemu.org/docs/master/interop/bitmaps.html) |
| Backup vs snapshot | [Wikipedia — Backup](https://en.wikipedia.org/wiki/Backup) |
| Incremental backup | [Wikipedia — Incremental backup](https://en.wikipedia.org/wiki/Incremental_backup) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What does a dirty bitmap let a backup tool do that a naive "copy the whole disk every night" approach cannot?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A dirty bitmap records, live and cheaply, exactly which disk blocks were written since a checkpoint. That lets a backup tool perform an *incremental* backup — copying only the blocks that changed since the last checkpoint instead of the entire disk. The naive approach must re-copy all data every time (it has no way to know what changed), making each backup slow, huge, and I/O-heavy. With a dirty bitmap a nightly full copy of, say, 100 GiB becomes a small incremental of just the delta (e.g. 2 GiB), enabling frequent, efficient backups of large running VMs.
</details>

---

**Q2. Distinguish snapshot, backup, and replication — what is each for, and which survive host loss?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A snapshot is a fast local rollback point on the same storage as the VM — great for "undo" before a risky change, but it does NOT survive host/storage loss. A backup is a copy of the data to *independent* storage (often incremental via dirty bitmaps) for disaster recovery — it survives host loss. Replication is a continuous, near-real-time copy to another host/site for failover/high availability — it also survives host loss and minimizes downtime. They're complementary: snapshot for quick undo, backup for DR, replication for HA. A snapshot is not a backup, and a backup is not HA.
</details>

---

**Q3. What is a checkpoint, and how does it relate to incremental backups?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A checkpoint marks a point in time and starts (or anchors) a dirty bitmap that tracks every block written after it as the guest runs. Incremental backups use it: when you back up, you reference the previous checkpoint, and QEMU/libvirt copy only the blocks dirtied since that checkpoint. Checkpoints can be chained (one per backup), so each incremental captures just the delta since the last one. Without a checkpoint there's no recorded "since when," so you'd have to do a full backup.
</details>

---

## Homework

Create a checkpoint-anchored full backup of a running VM, then write some data inside the guest, then take an incremental backup referencing that checkpoint. Compare the size/time of the incremental versus the full. Explain why the incremental was smaller and what role the dirty bitmap played. Then state in one sentence why neither of these is a substitute for replication if you need high availability.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The full backup copies the entire disk and takes proportionally long; after writing a small amount of new data, the incremental backup (referencing the checkpoint) is much smaller and faster because it transfers only the blocks dirtied since the checkpoint. The dirty bitmap is what made this possible: it recorded exactly which blocks changed after the checkpoint, so the backup tool copied only those rather than re-scanning/re-copying the whole disk. Neither is a substitute for replication for HA, because backups are point-in-time copies that require a restore (downtime + data loss back to the last backup), whereas replication continuously mirrors state to another host so a standby can take over near-instantly on failure.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 48 — Live Migration Mechanics →](lesson-48-migration-mechanics){: .btn .btn-primary }
