---
title: "Phase 12: Snapshots, Backup & Migration"
nav_order: 14
has_children: true
---

# Phase 12: Snapshots, Backup & Live Migration

Three different jobs that people often conflate: capturing state at the management layer
(domain snapshots and incremental backup with dirty bitmaps), and the headline trick of
virtualization — moving a *running* VM between hosts with minimal downtime (live
migration). This phase covers the mechanics (pre-copy vs post-copy) and the practice
(`virsh migrate`, shared storage, CPU compatibility).

| # | Lesson | Status |
|---|--------|--------|
| 47 | [Domain snapshots, checkpoints, and incremental backup](lesson-47-backup-checkpoints) | In progress |
| 48 | [Live migration mechanics](lesson-48-migration-mechanics) | In progress |
| 49 | [Live migration in practice](lesson-49-migration-practice) | In progress |
