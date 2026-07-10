---
title: "Lesson 28 — Deadlock"
nav_order: 5
parent: "Phase 5: Concurrency"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 28: Deadlock

## Concept

Locks fixed races. Now meet the failure mode the fix invented: two threads,
two locks, forever.

```
   thread A                          thread B
   ────────                          ────────
   lock(accounts)      ✓             lock(audit_log)     ✓
   …                                 …
   lock(audit_log)     zzz waits     lock(accounts)      zzz waits
        for B…                            for A…

               A → waits-for → B
               ▲               │        a CYCLE in the waits-for graph
               └── waits-for ──┘        = DEADLOCK. No one EVER proceeds.
```

Neither thread is buggy in isolation — each takes locks it legitimately needs.
The bug is *global*: their acquisition orders form a cycle. And nothing
rescues you: mutex sleeps have no timeout by default, the kernel doesn't
detect userspace deadlocks (each thread is just "sleeping in futex_wait" —
a perfectly normal state from Lesson 26's lab!), and CPU usage drops to
exactly zero on the deadlocked threads — the system looks *calm*.

The classical analysis (Coffman, 1971): deadlock requires **all four**
conditions simultaneously — **mutual exclusion** (resources are exclusive),
**hold and wait** (holding one while requesting another), **no preemption**
(locks can't be forcibly taken away), **circular wait** (the cycle). Kill any
one condition, deadlock is impossible. Practical prevention almost always
targets the last: impose a **global lock ordering** — all threads acquire
locks in one agreed order, and cycles can't form.

Two cousins complete the taxonomy: **livelock** — threads actively run but
mutually undo each other's progress (both back off, both retry, both collide,
forever: CPU 100%, progress 0) — and **starvation** — the system progresses
but some thread never wins (an unfair lock under heavy contention). Same
symptom to users ("it's stuck"), three different diagnoses and cures — the lab
teaches telling them apart from the outside.

---

## How It Works

### Diagnosing a deadlock in the wild

The fingerprint: threads in S state (sleeping — Lesson 08), zero CPU, wchan =
`futex_wait_queue` (Lesson 26), and *no progress ever*. The tool is gdb:

```
gdb -p PID
(gdb) thread apply all bt        ← backtrace of EVERY thread
```

Read the traces: each deadlocked thread shows
`futex_wait → pthread_mutex_lock → your_function(file:line)`. From the source
lines you reconstruct who holds what and wants what — the cycle falls out on
paper. (With `pthread_mutex_t` internals, gdb can even print the owner TID of
each mutex: `p mutex.__data.__owner` — matching them to thread TIDs closes the
proof.) Python: `py-spy dump`; Java: `jstack` finds cycles *automatically*;
Go: deadlock of all goroutines panics the runtime.

### Prevention toolbox, in order of preference

1. **Fewer locks**: one lock protecting both structures can't deadlock with
   itself; coarse-then-refine beats premature fine-graining.
2. **Lock ordering**: document a global order (by address, by ID, by layer —
   any total order); enforce in review; the bank-transfer classic locks
   `min(a,b)` then `max(a,b)`.
3. **Try-and-back-off**: `pthread_mutex_trylock` on the second lock; on
   failure release *everything* and retry — kills hold-and-wait (but invites
   livelock: add randomized backoff).
4. **Timeouts as tripwires**: `pthread_mutex_timedlock` converting eternal
   silence into a loud error — not prevention, but detection in production.
5. **Runtime/static detection**: TSan detects lock-order inversions
   (`-fsanitize=thread` reports "lock-order-inversion" even when the deadlock
   didn't fire this run!); the kernel's own lockdep does this for kernel code.

{: .note }
> **Deadlocks don't need two mutexes**
> Self-deadlock: taking a non-recursive mutex you already hold (often via a
> callback — Lesson 26's "never call unknown code under a lock"). Lock +
> condvar misuse, lock + fork (the child inherits a locked mutex whose owner
> thread doesn't exist in it — the classic fork-in-threaded-program trap,
> Lesson 07 meets 24), reader-writer upgrade attempts, and two <em>processes</em>
> crossing file locks (Lesson 36) all deadlock with the same waits-for-cycle
> anatomy.

---

## Lab

```bash
# ---- 1. Manufacture the classic AB-BA deadlock ----
$ cat > /tmp/dead.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
pthread_mutex_t A = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t B = PTHREAD_MUTEX_INITIALIZER;
void *t1(void *x) {
    pthread_mutex_lock(&A); printf("t1: holds A\n");
    usleep(100000);                       /* widen the window */
    printf("t1: wants B...\n"); pthread_mutex_lock(&B);
    printf("t1: NEVER PRINTED\n");
    return NULL;
}
void *t2(void *x) {
    pthread_mutex_lock(&B); printf("t2: holds B\n");
    usleep(100000);
    printf("t2: wants A...\n"); pthread_mutex_lock(&A);
    printf("t2: NEVER PRINTED\n");
    return NULL;
}
int main(void) {
    pthread_t a, b;
    pthread_create(&a, 0, t1, 0); pthread_create(&b, 0, t2, 0);
    pthread_join(a, 0); pthread_join(b, 0);
    printf("main: never reached either\n");
    return 0;
}
EOF
$ gcc -pthread -g -o /tmp/dead /tmp/dead.c && /tmp/dead &
# t1: holds A / t2: holds B / t1: wants B... / t2: wants A...   then SILENCE.

# ---- 2. Diagnose from outside: the calm-looking corpse ----
$ PID=$(pgrep -f /tmp/dead)
$ ps -o pid,state,pcpu -Lp $PID
# every thread: S state, 0.0 %CPU — indistinguishable from healthy idle!
$ for t in $(ls /proc/$PID/task); do
    echo "tid $t: $(cat /proc/$PID/task/$t/wchan)"; done
# tid ...: futex_wait_queue   ×2 — both workers parked forever

# ---- 3. The confession: gdb thread apply all bt ----
$ sudo gdb -p $PID -batch -ex 'thread apply all bt' 2>/dev/null | grep -E 'Thread|lock|t1|t2|/tmp'
# Thread 3: ... futex_wait ... pthread_mutex_lock ... t1 () at /tmp/dead.c:10
# Thread 2: ... futex_wait ... pthread_mutex_lock ... t2 () at /tmp/dead.c:17
#   line 10 = t1 wants B; line 17 = t2 wants A. Cycle, on paper. Case closed.
$ kill -9 $PID     # no gentler cure exists (SIGTERM works too — L09 default)

# ---- 4. TSan predicts it WITHOUT triggering it ----
$ gcc -pthread -fsanitize=thread -g -o /tmp/dead_tsan /tmp/dead.c
$ timeout 5 /tmp/dead_tsan 2>&1 | grep -A3 'lock-order-inversion' | head -6
# WARNING: ThreadSanitizer: lock-order-inversion (potential deadlock)
#   ← flagged even on runs where timing let it survive. Order violations
#     are detectable statically-ish; actual deadlocks are a timing lottery.

# ---- 5. Fix #1 — global lock ordering (always A before B) ----
$ sed 's/pthread_mutex_lock(&B); printf("t2: holds B/pthread_mutex_lock(\&A); printf("t2: holds A/; s/printf("t2: wants A...\\n"); pthread_mutex_lock(&A)/printf("t2: wants B...\\n"); pthread_mutex_lock(\&B)/' /tmp/dead.c > /tmp/ordered.c
$ gcc -pthread -o /tmp/ordered /tmp/ordered.c && /tmp/ordered
# both NEVER PRINTED lines print; main reached. Same work, one agreed order.

# ---- 6. Fix #2 — trylock + backoff (and its livelock shadow) ----
$ cat > /tmp/backoff.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
pthread_mutex_t A = PTHREAD_MUTEX_INITIALIZER, B = PTHREAD_MUTEX_INITIALIZER;
void *worker(void *arg) {
    pthread_mutex_t *first = arg ? &B : &A, *second = arg ? &A : &B;
    for (;;) {
        pthread_mutex_lock(first);
        if (pthread_mutex_trylock(second) == 0) {         /* got both! */
            printf("thread %ld: acquired both — done\n", (long)arg);
            pthread_mutex_unlock(second); pthread_mutex_unlock(first);
            return NULL;
        }
        pthread_mutex_unlock(first);                       /* release ALL */
        usleep(rand() % 1000);                             /* RANDOM backoff */
    }
}
int main(void) {
    pthread_t a, b;
    pthread_create(&a, 0, worker, (void*)0);
    pthread_create(&b, 0, worker, (void*)1);
    pthread_join(a, 0); pthread_join(b, 0);
    return 0;
}
EOF
$ gcc -pthread -o /tmp/backoff /tmp/backoff.c && /tmp/backoff
# both acquire — hold-and-wait broken. Remove the usleep and both can
# livelock: grab-fail-release in lockstep, CPUs blazing, forever. The
# RANDOMNESS is what breaks the symmetry (same trick as Ethernet backoff!).

$ rm /tmp/dead* /tmp/ordered* /tmp/backoff*
```

---

## Further Reading

| Topic | Link |
|---|---|
| Deadlock | <https://en.wikipedia.org/wiki/Deadlock_(computer_science)> |
| Coffman conditions | <https://en.wikipedia.org/wiki/Deadlock_(computer_science)#Necessary_conditions> |
| Livelock & starvation | <https://en.wikipedia.org/wiki/Deadlock_(computer_science)#Livelock> |
| `pthread_mutex_trylock(3)` | <https://man7.org/linux/man-pages/man3/pthread_mutex_trylock.3p.html> |
| Dining philosophers problem | <https://en.wikipedia.org/wiki/Dining_philosophers_problem> |
| Kernel lockdep | <https://www.kernel.org/doc/html/latest/locking/lockdep-design.html> |

---

## Checkpoint

**Q1.** Your program hangs. Using gdb and top, how do you distinguish
deadlock from livelock from starvation?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
First, top/ps: <strong>deadlock</strong> shows the stuck threads at 0% CPU in
S state (sleeping in futex_wait — parked, nothing to burn);
<strong>livelock</strong> shows them at ~100% CPU (actively retrying and
failing); <strong>starvation</strong> shows the <em>system</em> busy and
progressing while one victim thread gets ~0% — but unlike deadlock, the locks
it wants keep changing hands. Then gdb <code>thread apply all bt</code>
repeatedly (two snapshots!): deadlocked threads show <em>identical</em>
backtraces both times, each blocked in mutex_lock, and the file:lines close a
waits-for cycle; livelocked threads show <em>changing</em> backtraces cycling
through the same retry loop; a starved thread sits at the same lock while
other threads' traces show them repeatedly acquiring it. The two-snapshot
trick is the key: frozen = deadlock, spinning = livelock, moving-except-one =
starvation.
</details>

**Q2.** Break each of the four Coffman conditions with one concrete
engineering technique, and note the technique's cost.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Mutual exclusion</strong> → make the resource shareable: lock-free
structures/atomics (Lesson 29) or immutable snapshots (RCU-style: readers
never lock); cost: severe design constraints, only fits some structures.
<strong>Hold and wait</strong> → acquire all-or-nothing: trylock + release-all
+ backoff (lab fix #2), or take every needed lock up front before any work;
cost: retry storms/livelock risk, and knowing your full lock set in advance.
<strong>No preemption</strong> → make locks stealable: timeouts
(timedlock) with rollback, or transactional designs (databases abort victim
transactions — deadlock <em>detection + preemption</em> is exactly what a DB's
deadlock detector does); cost: writing rollback/compensation logic.
<strong>Circular wait</strong> → total lock ordering (lab fix #1): by
address, ID, or architectural layer; cost: discipline and documentation —
the order must survive every refactor and new hire, which is why it pairs
with TSan/lockdep as the enforcement mechanism. Industry default: ordering
for correctness + timeouts as tripwires.
</details>

**Q3.** A threaded program deadlocks *only* in the child after `fork()`. Using
Lessons 07/24 plus this one, explain the mechanism and the two standard
escapes.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
fork clones only the <em>calling thread</em> into the child — but it clones
<em>all memory</em>, including mutexes in whatever state they were at that
instant (Lesson 07's CoW copies the lock bytes too). If any <em>other</em>
thread held a lock (malloc's arena lock is the classic — some thread
mid-allocation) at fork time, the child's copy shows "locked" with an owner
that <em>doesn't exist in the child</em> — no one will ever unlock it. The
child's first malloc/printf then sleeps on a corpse's lock: guaranteed
deadlock, timing-dependent (only when fork raced an allocation). Escapes: (1)
the POSIX rule — in a threaded process, the child may only call
<strong>async-signal-safe</strong> functions between fork and exec
(fork+exec immediately is always fine — the exec wipes the poisoned memory,
Lesson 07); (2) <code>pthread_atfork</code> handlers that acquire the key
locks before fork and release them in both parent and child — fragile
(every library's locks!), which is why the real-world advice is: don't fork
threaded processes except to exec, or fork early before threads exist
(posix_spawn encapsulates the safe pattern).
</details>

---

## Homework

The dining philosophers: five philosophers around a table, one fork between
each pair; eating needs both adjacent forks; every philosopher runs
`lock(left); lock(right); eat; unlock both`. (a) Prove this deadlocks (which
cycle?). (b) Fix it with lock ordering — what's the minimal change, and why
does it work when four philosophers pick "left first" and one picks "right
first"? (c) Name the starvation issue that remains even in the fixed version,
and one mitigation.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) All five sit simultaneously and each grabs their left fork: now every fork
is held, and every philosopher waits for their right neighbor's fork — the
waits-for graph is the 5-cycle P0→P1→P2→P3→P4→P0. All four Coffman conditions
hold (forks exclusive, each holds-one-wants-one, no stealing, cycle) —
deadlock with probability rising toward certainty under repetition. (b)
Number the forks 0–4 and require every philosopher to lock the
<em>lower-numbered</em> fork first. For four philosophers that happens to be
"left first"; for the one between fork 4 and fork 0 it means "right first" —
and that asymmetry is the proof: a cycle would need someone holding a
higher-numbered fork while waiting for a lower one, which the rule forbids;
with a total order on acquisitions, the waits-for graph is a DAG — no cycle,
no deadlock, guaranteed (not probabilistically). (c) Starvation: nothing
stops a philosopher's two neighbors from alternating so the middle one always
finds a fork taken (unfair mutex acquisition makes it worse — Lesson 14's
fairness themes). Mitigations: a waiter/arbitrator semaphore admitting at most
4 to the table at once (breaks the pattern and bounds waiting), fair/FIFO
mutexes, or the Chandy–Misra token protocol for the full formal fix.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 29 — Atomics and Memory Ordering →](lesson-29-atomics-ordering){: .btn .btn-primary }
