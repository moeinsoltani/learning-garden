---
title: "Phase 3: Scheduling"
nav_order: 5
has_children: true
---

# Phase 3: Scheduling

Lesson 05 showed *how* the kernel takes the CPU back (the timer interrupt).
This phase covers what it does with that power: what a context switch costs,
how the fair scheduler decides who runs next, what the priority knobs (nice,
real-time classes) actually do, and how to read system load honestly — including
the modern pressure metrics (PSI) that finally answer "is this box overloaded?"

| # | Lesson | Status |
|---|--------|--------|
| 12 | [Context switches](lesson-12-context-switches) | Ready |
| 13 | [The scheduler — CFS to EEVDF](lesson-13-cfs-eevdf) | Ready |
| 14 | [Priorities — nice and real-time](lesson-14-priorities-realtime) | Ready |
| 15 | [Affinity, load, and pressure](lesson-15-affinity-psi) | Ready |
