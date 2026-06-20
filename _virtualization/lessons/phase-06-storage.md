---
title: "Phase 6: Storage"
nav_order: 8
has_children: true
---

# Phase 6: Storage

A VM's disk is a file (or a block device) on the host, and how you manage it
determines performance, flexibility, and safety. This phase covers `qemu-img` and
image formats, qcow2 internals (backing files and copy-on-write — the magic behind
templates and clones), how to connect disks to the guest (block device models),
caching modes and their data-safety trade-offs, and snapshots.

| # | Lesson | Status |
|---|--------|--------|
| 22 | [Disk image formats and qemu-img](lesson-22-image-formats-qemu-img) | In progress |
| 23 | [qcow2 internals — backing files and copy-on-write](lesson-23-qcow2-internals) | In progress |
| 24 | [Block device models](lesson-24-block-device-models) | In progress |
| 25 | [Caching modes and AIO](lesson-25-caching-aio) | In progress |
| 26 | [Snapshots](lesson-26-snapshots) | In progress |
