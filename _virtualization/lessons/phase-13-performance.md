---
title: "Phase 13: Performance Tuning"
nav_order: 15
has_children: true
---

# Phase 13: Performance Tuning & Observability

This phase turns the conceptual knobs from earlier phases into a practical tuning
discipline: pinning vCPUs and placing threads to kill jitter, applying NUMA/memory tuning
for real workloads, getting the disk subsystem out of the way (iothreads, multiqueue),
measuring a running VM the way the networking track measures packets, and using `tuned`
profiles for sane defaults. The methodology mirrors **Networking Lesson 47**: work
bottom-up, layer by layer.

| # | Lesson | Status |
|---|--------|--------|
| 50 | [CPU pinning and thread placement](lesson-50-cpu-pinning) | In progress |
| 51 | [Memory and NUMA tuning in production](lesson-51-memory-numa-tuning) | In progress |
| 52 | [Storage and I/O tuning](lesson-52-storage-io-tuning) | In progress |
| 53 | [Observability — measuring a running VM](lesson-53-observability) | In progress |
| 54 | [tuned and host profiles](lesson-54-tuned-profiles) | In progress |
