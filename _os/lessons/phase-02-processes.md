---
title: "Phase 2: Processes"
nav_order: 4
has_children: true
---

# Phase 2: Processes

The process is the kernel's unit of "a running program": an address space, a
set of open files, credentials, and at least one thread of execution. This phase
covers the full lifecycle — the two-step fork/exec birth, the states a process
moves through, death and reaping (zombies!), the signal machinery, how the
terminal's job control really works, and who a process is allowed to *be*
(credentials and capabilities).

| # | Lesson | Status |
|---|--------|--------|
| 6 | [Anatomy of a process](lesson-06-process-anatomy) | Ready |
| 7 | [fork and exec](lesson-07-fork-exec) | Ready |
| 8 | [Process lifecycle — zombies, orphans, wait](lesson-08-process-lifecycle) | Ready |
| 9 | [Signals](lesson-09-signals) | Ready |
| 10 | [Process groups, sessions, and job control](lesson-10-sessions-job-control) | Ready |
| 11 | [Credentials and capabilities](lesson-11-credentials-capabilities) | Ready |
