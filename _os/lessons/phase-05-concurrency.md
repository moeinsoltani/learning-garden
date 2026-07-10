---
title: "Phase 5: Concurrency"
nav_order: 7
has_children: true
---

# Phase 5: Concurrency

Threads made your programs faster and your bugs nondeterministic. This phase
does both halves properly: what threads actually are (clone with everything
shared), building a real race condition and watching it corrupt data, the lock
that fixes it (and the futex underneath), coordinating waiters (condition
variables, semaphores), the classic failure modes (deadlock), the lock-free
alternative (atomics and memory ordering), and how high-level async patterns
map onto all of it.

| # | Lesson | Status |
|---|--------|--------|
| 24 | [Threads vs processes](lesson-24-threads) | Ready |
| 25 | [Race conditions — build one, watch it fail](lesson-25-race-conditions) | Ready |
| 26 | [Mutexes and futexes](lesson-26-mutex-futex) | Ready |
| 27 | [Condition variables and semaphores](lesson-27-condvars-semaphores) | Ready |
| 28 | [Deadlock](lesson-28-deadlock) | Ready |
| 29 | [Atomics and memory ordering](lesson-29-atomics-ordering) | Ready |
| 30 | [Async and thread-pool patterns](lesson-30-async-patterns) | Ready |
