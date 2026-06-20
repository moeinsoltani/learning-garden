---
title: "Phase 7: VirtIO & Paravirtualization"
nav_order: 9
has_children: true
---

# Phase 7: VirtIO & Paravirtualization

You've seen "use virtio" repeated as advice since Phase 1. Now you learn *why* it
works and *how far* the idea goes. This phase covers the virtio standard
(virtqueues, feature negotiation), the catalog of virtio devices, and the
performance offloads that push the datapath progressively closer to the
metal — vhost (into the kernel) and vhost-user (into another userspace process) —
plus host/guest shortcuts like vsock and virtio-fs.

| # | Lesson | Status |
|---|--------|--------|
| 27 | [The virtio standard](lesson-27-virtio-standard) | In progress |
| 28 | [The core virtio devices](lesson-28-virtio-devices) | In progress |
| 29 | [vhost — moving the datapath into the kernel](lesson-29-vhost) | In progress |
| 30 | [vhost-user, vsock, and shared-memory datapaths](lesson-30-vhost-user-vsock) | In progress |
