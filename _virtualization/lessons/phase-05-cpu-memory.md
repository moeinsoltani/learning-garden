---
title: "Phase 5: CPU & Memory Configuration"
nav_order: 7
has_children: true
---

# Phase 5: CPU & Memory Configuration

Now that you can boot a VM, you tune what it's made of. This phase covers what CPU
the guest believes it has (models and feature flags), how to present a sensible
sockets/cores/threads topology, how guest RAM is backed on the host (memory
backends and hugepages), reclaiming and deduplicating memory across guests
(ballooning and KSM), and avoiding the NUMA performance cliff.

| # | Lesson | Status |
|---|--------|--------|
| 17 | [vCPU models and feature flags](lesson-17-vcpu-models) | In progress |
| 18 | [CPU topology](lesson-18-cpu-topology) | In progress |
| 19 | [Guest memory backends and hugepages](lesson-19-memory-hugepages) | In progress |
| 20 | [Ballooning and KSM](lesson-20-balloon-ksm) | In progress |
| 21 | [NUMA awareness](lesson-21-numa) | In progress |
