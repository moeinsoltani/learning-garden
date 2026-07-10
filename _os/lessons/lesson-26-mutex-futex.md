---
title: "Lesson 26 — Mutexes and Futexes"
nav_order: 3
parent: "Phase 5: Concurrency"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 26: Mutexes and Futexes

## Concept

Lesson 25's disease has a standard cure: the **mutex** (mutual exclusion
lock). One thread at a time may hold it; everyone else queues. Wrap the
load-add-store in lock/unlock and the three steps become indivisible *as far
as other lockers are concerned*:

```
   thread A                          thread B
   ────────                          ────────
   lock(m)        ✓ got it
   load counter (100)                lock(m)   ✗ held → SLEEP (not spin!)
   add 1                                       zzz…
   store counter (101)                         zzz…
   unlock(m) ──── wakes B ──────────▶ ✓ got it
                                     load counter (101)   ← sees A's work
                                     …
```

The mutex guarantees two things at once — and you need both:
**mutual exclusion** (the critical section never interleaves) and **memory
visibility** (everything A wrote before unlock is visible to B after lock —
the cache/compiler saboteurs from Lesson 25 are ordered around the lock; this
is the acquire/release contract formalized in Lesson 29).

But here's the performance puzzle this lesson actually answers: your programs
lock and unlock mutexes *millions of times per second*. If each operation were
a syscall (Lesson 02: ~hundreds of ns just to enter the kernel), locking would
dominate every workload. Run strace on a lock-heavy program and you'll see…
almost nothing. The trick is the **futex** — *fast userspace mutex* — one of
Linux's most elegant designs: **the uncontended case never enters the kernel
at all.**

---

## How It Works

### The futex protocol

A mutex is just an integer in ordinary shared memory. Locking tries an atomic
compare-and-swap (Lesson 29's CAS: "if 0, set to 1") — pure userspace, a few
nanoseconds. Uncontended lock/unlock is *only that*. The kernel exists solely
for the unhappy path: when CAS fails (lock held), the thread calls
[`futex(FUTEX_WAIT, &lock, expected)`](https://man7.org/linux/man-pages/man2/futex.2.html)
— "put me to sleep *unless* the value already changed" (the re-check closes a
lost-wakeup race). Unlock stores 0 and — only if the value says someone waits
— calls `futex(FUTEX_WAKE)`. The kernel keeps a hash table: address → sleeping
threads. That's the whole design:

```
  fast path (no contention):   CAS in userspace          ~10-25 ns, 0 syscalls
  slow path (contention):      futex syscall, sleep       µs, real syscalls
```

Every pthread mutex, every language-runtime lock on Linux (Java, Go, Python,
Rust) bottoms out in this protocol.

### Spinlocks vs sleeping locks

The mutex's slow path *sleeps* (voluntary context switch, Lesson 12) — right
for critical sections of unknown length. The alternative, **spinlock** — retry
the CAS in a loop, burning CPU — wins only when the wait is reliably shorter
than a context switch (~1–2 µs): kernel internals and interrupt context (which
*cannot* sleep — Lesson 05) use them constantly; userspace almost never should
(a preempted lock-holder leaves spinners burning whole timeslices — the
priority-inversion adjacent disaster). Adaptive mutexes split the difference:
spin briefly, then sleep.

### What contention costs — and how to shrink it

A contended lock serializes: Amdahl's law with a queue. Worse, the lock's
cache line ping-pongs between cores (each CAS steals it), so even *failed*
attempts slow the holder. The craft is minimizing the critical section:
lock, mutate, unlock — never I/O, allocation, or callbacks while holding;
shard one hot lock into N (per-bucket locks in hash tables); or step off to
read-write locks / per-thread counters / atomics when the pattern fits
(Lessons 29–30).

{: .note }
> **Diagnosing lock trouble from the outside**
> High <em>voluntary</em> context switches + low CPU + slow throughput =
> threads queueing on locks (Lesson 12's tools!). <code>/proc/PID/wchan</code>
> shows <code>futex_wait</code>; <code>perf top</code> shows time in lock
> functions; <code>strace -c</code> counts futex calls — a healthy program
> shows few; a contended one shows storms. You already own every tool in that
> list.

---

## Lab

```bash
# ---- 1. Fix Lesson 25's race — completely ----
$ cat > /tmp/fixed.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
long counter = 0; long iters;
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
void *inc(void *arg) {
    for (long i = 0; i < iters; i++) {
        pthread_mutex_lock(&m);
        counter++;
        pthread_mutex_unlock(&m);
    }
    return NULL;
}
int main(int argc, char **argv) {
    iters = atol(argv[1]);
    pthread_t a, b;
    pthread_create(&a, NULL, inc, NULL); pthread_create(&b, NULL, inc, NULL);
    pthread_join(a, NULL); pthread_join(b, NULL);
    printf("expected %ld, got %ld\n", 2 * iters, counter);
    return 0;
}
EOF
$ gcc -pthread -O2 -o /tmp/fixed /tmp/fixed.c
$ for i in 1 2 3; do /tmp/fixed 1000000; done
# expected 2000000, got 2000000 — every time, any -O level, forever.

# ---- 2. The futex reveal: count the syscalls ----
# single thread = zero contention = the fast path only:
$ cat > /tmp/solo.c << 'EOF'
#include <pthread.h>
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
int main(void) {
    for (int i = 0; i < 5000000; i++) {
        pthread_mutex_lock(&m);
        pthread_mutex_unlock(&m);
    }
    return 0;
}
EOF
$ gcc -pthread -O2 -o /tmp/solo /tmp/solo.c
$ strace -c -e trace=futex /tmp/solo
# futex calls: 0 (or ~1)   ← FIVE MILLION lock/unlock pairs. Zero kernel visits.
$ time /tmp/solo
# ~0.05s → ~10ns per lock+unlock pair. Now you know why locking is affordable.

# ---- 3. Add contention: watch the slow path appear ----
$ strace -cf -e trace=futex /tmp/fixed 500000 2>&1 | tail -4
# thousands of futex calls — every one is a thread that found the lock held,
# slept, and was woken. Contention is what costs; locks are nearly free.

# ---- 4. See a waiter stuck in futex_wait, from outside ----
$ cat > /tmp/hold.c << 'EOF'
#include <pthread.h>
#include <unistd.h>
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
void *grab(void *a){ pthread_mutex_lock(&m); pause(); return 0; }
int main(void){
    pthread_t t1, t2;
    pthread_create(&t1, 0, grab, 0);    /* wins, holds forever */
    sleep(1);
    pthread_create(&t2, 0, grab, 0);    /* queues forever */
    pthread_join(t1, 0);
    return 0;
}
EOF
$ gcc -pthread -o /tmp/hold /tmp/hold.c && /tmp/hold &
$ sleep 2; for tid in $(ls /proc/$(pgrep -f /tmp/hold)/task); do
    echo "tid $tid: $(cat /proc/$(pgrep -f /tmp/hold)/task/$tid/wchan)"; done
# tid ...: do_wait / pause          ← main & holder
# tid ...: futex_wait_queue         ← the queued thread, named precisely!
$ kill %1

# ---- 5. Critical section size: the difference it makes ----
$ cat > /tmp/width.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
long counter = 0; int wide;
void *work(void *a) {
    for (int i = 0; i < 200000; i++) {
        if (wide) {                     /* BAD: work inside the lock */
            pthread_mutex_lock(&m);
            for (volatile int j = 0; j < 300; j++);   /* "processing" */
            counter++;
            pthread_mutex_unlock(&m);
        } else {                        /* GOOD: work outside */
            for (volatile int j = 0; j < 300; j++);
            pthread_mutex_lock(&m);
            counter++;
            pthread_mutex_unlock(&m);
        }
    }
    return NULL;
}
int main(int argc, char **argv) {
    wide = atoi(argv[1]);
    pthread_t t[4];
    for (int i = 0; i < 4; i++) pthread_create(&t[i], 0, work, 0);
    for (int i = 0; i < 4; i++) pthread_join(t[i], 0);
    printf("counter=%ld\n", counter);
    return 0;
}
EOF
$ gcc -pthread -O1 -o /tmp/width /tmp/width.c
$ time /tmp/width 1      # wide: "processing" serialized under the lock
$ time /tmp/width 0      # narrow: processing parallel, only ++ locked
# narrow is ~2-4x faster on 4 CPUs. Same lock, same work — different WIDTH.

$ rm /tmp/fixed* /tmp/solo* /tmp/hold* /tmp/width* 2>/dev/null
```

---

## Further Reading

| Topic | Link |
|---|---|
| `futex(2)` man page | <https://man7.org/linux/man-pages/man2/futex.2.html> |
| `pthread_mutex_lock(3)` | <https://man7.org/linux/man-pages/man3/pthread_mutex_lock.3.html> |
| Futex (Wikipedia) | <https://en.wikipedia.org/wiki/Futex> |
| Mutual exclusion | <https://en.wikipedia.org/wiki/Mutual_exclusion> |
| Spinlock | <https://en.wikipedia.org/wiki/Spinlock> |
| `futex(7)` overview | <https://man7.org/linux/man-pages/man7/futex.7.html> |

---

## Checkpoint

**Q1.** strace shows no futex calls though the code locks a mutex constantly.
Is the mutex working? Explain what you'd see if contention appeared.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It's working perfectly — that's the futex design succeeding. Uncontended
lock/unlock is a userspace atomic compare-and-swap on the mutex's integer;
the kernel is consulted only when a thread must sleep (lock found held) or
waiters must be woken. No contention → no syscalls → nothing for strace
(which watches the syscall boundary — Lesson 02) to see; mutual exclusion and
the memory-visibility guarantees are enforced by the CPU's atomic instruction,
not by the kernel. Under contention you'd see FUTEX_WAIT calls from blockers
(threads parked in <code>futex_wait_queue</code>, visible in wchan) and
FUTEX_WAKE from unlockers — the syscall count becomes a direct contention
meter, which is exactly how lab step 3 used it.
</details>

**Q2.** Why do sleeping locks (mutexes) suit userspace while the kernel is
full of spinlocks? Give both directions of the argument.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Userspace → sleep: critical-section length is unpredictable (allocation, page
faults — Lesson 19 — can strike inside), and a spinning waiter burns a full
timeslice achieving nothing whenever the holder is preempted — with the
scheduler (Lesson 13) happily running the spinner <em>instead of</em> the
holder it's waiting for. Sleeping converts waiting into scheduling: the CPU
does other work, the wake arrives when progress is possible. Kernel →
spin: interrupt context cannot sleep at all (no task context to suspend —
Lesson 05), and kernel critical sections are engineered to be a few dozen
instructions with preemption disabled on that CPU — the holder <em>cannot</em>
be descheduled, so waits are provably tiny and a context switch (~µs) would
cost 100× the wait. Each environment picks the lock whose worst case its
guarantees can bound. (Userspace adaptive mutexes spin ~microseconds first —
borrowing the kernel's bet only where it's statistically safe.)
</details>

**Q3.** The lab's "narrow vs wide" step showed 2–4× from moving work outside
the lock. State the general law at work, and list three things that should
essentially never happen while holding a mutex.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The law: a lock serializes its critical section, so system throughput is
bounded by (time inside the lock) × (times it's taken) — Amdahl's law where
the serial fraction is <em>your choice</em>. Everything inside the section
subtracts from all threads; everything outside parallelizes. Never inside a
mutex: (1) <strong>blocking I/O</strong> — disk/network waits (ms!) while N
threads queue: latency multiplies across the fleet, and interactions with
other locks invite deadlock (Lesson 28); (2) <strong>unbounded work</strong> —
allocation that may reclaim (Lesson 21), fault-able cold memory, loops over
user-sized data; (3) <strong>calls into unknown code</strong> — callbacks,
logging frameworks, signal-unsafe anything: they may block, take other locks
(ordering violations), or re-enter your lock. The discipline: compute first,
then lock-copy/swap-unlock — critical sections of a dozen lines, measured in
nanoseconds.
</details>

---

## Homework

The double-checked locking pattern initializes a singleton with a fast
lock-free read:

```c
if (instance == NULL) {              // check (no lock)
    pthread_mutex_lock(&m);
    if (instance == NULL)            // check again (with lock)
        instance = create();
    pthread_mutex_unlock(&m);
}
use(instance);
```

Explain: (a) why the *second* check is needed, (b) why the pattern as written
is *still broken* in C (hint: Lesson 25's saboteurs — what can the compiler
and CPU do to the unlocked first read and to `instance = create()`?), and (c)
what fixes it.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) Between a thread's failed first check and acquiring the mutex, another
thread may have initialized: without re-checking under the lock, both would
call <code>create()</code> — the check-then-act race (Lesson 25) reborn with
a lock that guards only the act. (b) The first read races with the writing
thread's <code>instance = create()</code> — a data race, hence UB — with two
concrete demons: the compiler/CPU may reorder so <code>instance</code> is
assigned <em>before</em> create()'s writes to the object complete (the mutex
orders memory only for threads that <em>take it</em> — the fast-path reader
never does!), so a reader can see a non-NULL pointer to uninitialized memory;
and the compiler may cache/tear the unsynchronized read. (c) Make the
fast-path read and the publishing write a synchronized pair: declare the
pointer <code>_Atomic</code> and use release store (writer) / acquire load
(reader) — Lesson 29's exact subject — or sidestep the pattern entirely with
<code>pthread_once</code> / C11 static-local initialization, which encapsulate
a correct implementation. Moral: locks synchronize only their takers; any
lock-free reader needs its own memory-ordering contract.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 27 — Condition Variables and Semaphores →](lesson-27-condvars-semaphores){: .btn .btn-primary }
