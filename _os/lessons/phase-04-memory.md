---
title: "Phase 4: Memory"
nav_order: 6
has_children: true
---

# Phase 4: Memory

Every address your programs have ever printed was fake. This phase builds the
whole memory story from that fact: virtual addressing and page tables, what a
process's address space really contains, how memory is allocated by *faulting*
rather than by malloc, the page cache that makes files fast, what happens under
pressure (swap, reclaim, the OOM killer), and the hardware-topology tuning
(hugepages, NUMA) that the virtualization track already used in production form.

| # | Lesson | Status |
|---|--------|--------|
| 16 | [Virtual memory — the grand illusion](lesson-16-virtual-memory) | Ready |
| 17 | [Page tables and the TLB](lesson-17-page-tables-tlb) | Ready |
| 18 | [A process's memory layout](lesson-18-process-memory-layout) | Ready |
| 19 | [Page faults and demand paging](lesson-19-page-faults) | Ready |
| 20 | [The page cache](lesson-20-page-cache) | Ready |
| 21 | [Swap and memory reclaim](lesson-21-swap-reclaim) | Ready |
| 22 | [Overcommit and the OOM killer](lesson-22-oom-overcommit) | Ready |
| 23 | [Hugepages and NUMA](lesson-23-hugepages-numa) | Ready |
