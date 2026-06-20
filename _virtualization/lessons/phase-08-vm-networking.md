---
title: "Phase 8: Networking for VMs"
nav_order: 10
has_children: true
---

# Phase 8: Networking for VMs

This phase deliberately **reuses the networking curriculum**. VM networking is just
the networking primitives you already learned — TAP devices, Linux bridges, MACVLAN —
wired to a guest. We climb the ladder from the zero-config default (SLIRP) to the
production pattern (TAP + bridge), then make it fast (vhost-net, multiqueue), then
look at lower-overhead alternatives (macvtap, SR-IOV).

> Cross-references: **Networking Lesson 14** (TAP), **Lesson 11** (bridges),
> **Lesson 13** (MACVLAN), **Lesson 33** (iperf3/bandwidth).

| # | Lesson | Status |
|---|--------|--------|
| 31 | [User-mode networking (SLIRP)](lesson-31-slirp) | In progress |
| 32 | [TAP + bridge networking](lesson-32-tap-bridge) | In progress |
| 33 | [Accelerated networking — vhost-net and multiqueue](lesson-33-accelerated-net) | In progress |
| 34 | [macvtap and SR-IOV networking](lesson-34-macvtap-sriov) | In progress |
