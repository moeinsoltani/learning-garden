---
title: "Lesson 27 — Condition Variables and Semaphores"
nav_order: 4
parent: "Phase 5: Concurrency"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 27: Condition Variables and Semaphores

## Concept

Mutexes solve "don't touch this together." The next problem is different:
**"wake me when something becomes true."** A consumer thread wants the next
item from a queue that's currently empty. Its options:

```
  poll:  while (queue_empty()) { unlock; lock; }   ← burns CPU, adds latency
  sleep: while (queue_empty()) { unlock; nap(10ms); lock; }  ← adds latency
  RIGHT: sleep until a producer TELLS me            ← condition variable
```

A **condition variable** (condvar) is a waiting room attached to a predicate
you define ("queue not empty"). The magic is in one atomic move:
`pthread_cond_wait(&cv, &m)` **releases the mutex and goes to sleep as one
indivisible step**, then re-acquires the mutex before returning. Producers
change the state under the mutex, then `signal` (wake one) or `broadcast`
(wake all).

Why must release-and-sleep be atomic? Picture the naive version: consumer
unlocks, *then* sleeps. In the gap, the producer takes the lock, adds an item,
signals — into an empty waiting room. The consumer then sleeps forever, one
item waiting. That's the **lost wakeup** — the bug the condvar's contract
exists to kill (the same re-check idea as FUTEX_WAIT's `expected` value —
Lesson 26; condvars are built on futexes too).

The **semaphore** is the other classic: a counter with atomic
`wait` (decrement-or-sleep-if-zero) and `post` (increment, wake a sleeper).
Where a mutex answers "may I be *the one*?", a semaphore answers "may I be
*one of N*?" — connection-pool slots, bounded-queue capacity, parking spaces.
A mutex has an owner (the locker must unlock); a semaphore doesn't (anyone may
post) — which makes it more flexible and easier to misuse.

---

## How It Works

### The wait must be a `while`, never an `if`

The condvar contract permits **spurious wakeups** (returning with no signal —
an artifact of efficient implementation on all platforms) and, more deeply,
**the predicate may already be false again**: with `signal`, between the wakeup
and your re-acquisition of the mutex, *another* consumer may have taken the
item (wake order ≠ lock-acquisition order). The bulletproof idiom is
mechanical:

```c
pthread_mutex_lock(&m);
while (!predicate)              /* WHILE: re-check on every wake */
    pthread_cond_wait(&cv, &m); /* atomically: unlock, sleep, relock */
/* predicate true AND mutex held — act */
pthread_mutex_unlock(&m);
```

And the producer side: change state *under the mutex*, then signal.
Signaling without the state change (or vice versa) recreates lost wakeups.

### signal vs broadcast

`signal` wakes one waiter — right when any single waiter can consume the event
(one item, one worker). `broadcast` wakes all — required when waiters wait for
*different* predicates on the same condvar, or a state change satisfies
everyone ("shutdown started"). Broadcast with N waiters causes the **thundering
herd**: N wake, one wins, N−1 re-sleep — correctness safe (the `while`!),
performance sad. Design so signal suffices when you can.

### Semaphores in practice

POSIX: `sem_init/sem_wait/sem_post` (thread-shared or process-shared — the
latter lives happily in shared memory, Lesson 33, coordinating *processes*).
A semaphore initialized to 1 imitates a mutex (worse: no owner, no
priority-inheritance — Lesson 14); initialized to N it's a resource meter; to
0 it's a signaling device ("post when ready" — pairs done/start between
threads). The bounded producer/consumer queue classically uses *two*
semaphores (slots-free, items-available) plus a mutex — the lab builds the
condvar version, the homework contrasts.

{: .note }
> **Which primitive when?**
> Protect state: <strong>mutex</strong>. Wait for a state-dependent condition:
> <strong>condvar</strong> (+ its mutex). Count resource slots:
> <strong>semaphore</strong>. Hand one value/event across:
> <strong>eventfd/pipe</strong> (Lesson 35) if an fd suits the design (event
> loops!). Most "which do I use" confusion dissolves by naming what you're
> waiting <em>for</em>.

---

## Lab

```bash
# ---- A bounded producer/consumer queue: the phase's centerpiece ----
$ cat > /tmp/queue.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
#define CAP 8
int buf[CAP], head = 0, tail = 0, count = 0;
int produced = 0, consumed = 0, done = 0;
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t not_full  = PTHREAD_COND_INITIALIZER;
pthread_cond_t not_empty = PTHREAD_COND_INITIALIZER;

void *producer(void *arg) {
    for (int i = 1; i <= 100000; i++) {
        pthread_mutex_lock(&m);
        while (count == CAP)                       /* full? wait for room  */
            pthread_cond_wait(&not_full, &m);
        buf[tail] = i; tail = (tail + 1) % CAP; count++; produced++;
        pthread_cond_signal(&not_empty);           /* one item → one waker */
        pthread_mutex_unlock(&m);
    }
    pthread_mutex_lock(&m);
    done = 1;
    pthread_cond_broadcast(&not_empty);            /* EVERYONE re-checks   */
    pthread_mutex_unlock(&m);
    return NULL;
}
void *consumer(void *arg) {
    long my = 0;
    for (;;) {
        pthread_mutex_lock(&m);
        while (count == 0 && !done)                /* WHILE, never if!     */
            pthread_cond_wait(&not_empty, &m);
        if (count == 0 && done) {                  /* drained + finished   */
            pthread_mutex_unlock(&m);
            break;
        }
        head = (head + 1) % CAP; count--; consumed++; my++;
        pthread_cond_signal(&not_full);
        pthread_mutex_unlock(&m);
    }
    printf("consumer %ld ate %ld items\n", (long)arg, my);
    return NULL;
}
int main(void) {
    pthread_t p, c[3];
    pthread_create(&p, NULL, producer, NULL);
    for (long i = 0; i < 3; i++) pthread_create(&c[i], NULL, consumer, (void*)i);
    pthread_join(p, NULL);
    for (int i = 0; i < 3; i++) pthread_join(c[i], NULL);
    printf("produced %d, consumed %d %s\n", produced, consumed,
           produced == consumed ? "— NOTHING LOST" : "— BUG!");
    return 0;
}
EOF
$ gcc -pthread -O2 -o /tmp/queue /tmp/queue.c && /tmp/queue
# consumer 0 ate 33420 items      ← split varies run to run (scheduling!)
# consumer 1 ate 33108 items
# consumer 2 ate 33472 items
# produced 100000, consumed 100000 — NOTHING LOST
# 100k items through an 8-slot buffer, 3 competing consumers, zero loss.

# ---- Break it on purpose #1: if instead of while ----
$ sed 's/while (count == 0 \&\& !done)/if (count == 0 \&\& !done)/' /tmp/queue.c > /tmp/broken.c
$ gcc -pthread -O2 -o /tmp/broken /tmp/broken.c
$ for i in 1 2 3; do timeout 10 /tmp/broken || echo "HUNG or crashed"; done
# with 3 consumers racing to re-acquire, an 'if' acts on a stale predicate:
# consuming from an empty queue (count goes negative / garbage values) or hang.
# The while costs one re-check; the if costs your weekend.

# ---- Break it on purpose #2: signal without state change is a lie ----
# (thought experiment — trace it: producer signals BEFORE incrementing count;
#  consumer wakes, while-check sees count==0, sleeps again: wakeup wasted.
#  Now imagine the reverse order under 'if': consumes garbage. The pairing
#  state-change-then-signal UNDER THE MUTEX is the whole contract.)

# ---- Semaphores: same queue, different accounting ----
$ python3 - << 'EOF'
import threading, queue, time
# Python's queue.Queue IS this lesson (condvars inside). Use it, know it:
q = queue.Queue(maxsize=8)
done = object()
def consumer(n):
    ate = 0
    while (item := q.get()) is not done:
        ate += 1
    print(f"consumer {n} ate {ate}")
t = [threading.Thread(target=consumer, args=(i,)) for i in range(3)]
[x.start() for x in t]
for i in range(100000): q.put(i)
for _ in t: q.put(done)                    # one poison pill per consumer
[x.join() for x in t]
print("clean shutdown via sentinel values")
EOF

# ---- Watch the waiting rooms from outside (Lesson 26's trick) ----
$ /tmp/queue & sleep 0.05
$ for tid in $(ls /proc/$(pgrep -f /tmp/queue)/task 2>/dev/null); do
    cat /proc/$(pgrep -f /tmp/queue)/task/$tid/wchan 2>/dev/null; echo; done | sort | uniq -c
# futex_wait_queue ×N — condvar waiters and mutex waiters, same futex bones
$ wait; rm /tmp/queue* /tmp/broken*
```

---

## Further Reading

| Topic | Link |
|---|---|
| `pthread_cond_wait(3)` | <https://man7.org/linux/man-pages/man3/pthread_cond_wait.3.html> |
| Monitor / condition variable | <https://en.wikipedia.org/wiki/Monitor_(synchronization)> |
| `sem_wait(3)` man page | <https://man7.org/linux/man-pages/man3/sem_wait.3.html> |
| Semaphore (programming) | <https://en.wikipedia.org/wiki/Semaphore_(programming)> |
| Producer–consumer problem | <https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem> |
| Thundering herd problem | <https://en.wikipedia.org/wiki/Thundering_herd_problem> |

---

## Checkpoint

**Q1.** Why must `pthread_cond_wait` be called in a `while` loop, not an `if`?
Give both independent reasons.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Reason 1 — <strong>spurious wakeups</strong>: the contract explicitly permits
wait to return with no signal (an implementation-efficiency allowance on every
platform); an <code>if</code> would proceed on a predicate nobody made true.
Reason 2 — <strong>the predicate can be re-falsified before you run</strong>:
signal wakes you, but you must re-acquire the mutex to return, and another
thread (a competing consumer who was <em>not</em> asleep) can take the lock
first and consume the item; by your turn, the queue is empty again. The while
makes wakeup mean only "re-check now", never "condition true" — turning both
hazards into a harmless extra loop iteration. It also makes the code robust to
broadcast, sentinel shutdowns, and future refactors: the predicate is the
truth; the signal is just a hint.
</details>

**Q2.** In the lab's shutdown, the producer sets `done` and uses `broadcast`,
while normal production uses `signal`. Justify both choices — what breaks with
signal-on-shutdown or broadcast-on-produce?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Produce → <code>signal</code>: one new item can satisfy exactly one consumer;
waking all three (broadcast) makes two lose the race and re-sleep — the
thundering herd: correct but wasteful (3× wakeups, context switches, mutex
contention — Lessons 12/26 — for nothing). Shutdown → <code>broadcast</code>:
"done" is news <em>every</em> waiter must act on (each must wake, see the
drained+done state, and exit); a single signal wakes one consumer — the other
two sleep forever on a condvar nobody will signal again: the program never
joins, a shutdown hang. Rule of thumb: signal when any one waiter can fully
consume the event; broadcast when the event changes the world for all waiters
(shutdown, config flip, "queue was rebuilt"). When in doubt, broadcast — the
while-loop makes it merely slower, never wrong.
</details>

**Q3.** A mutex initialized-to-1 semaphore can replace a mutex — but list two
concrete protections you lose, and one legitimate pattern only the semaphore
enables *because* it lacks an owner.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Lost protections: (1) <strong>ownership errors go undetected</strong> — a
mutex (especially PTHREAD_MUTEX_ERRORCHECK) can reject unlock-by-non-owner
and recursive locking; a semaphore happily accepts <code>post</code> from any
thread, including double-posts that turn "mutual exclusion" into two
concurrent holders — silent corruption (Lesson 25's world again); (2)
<strong>priority inheritance</strong> (Lesson 14) — PI needs to know who holds
the lock to boost them; ownerless semaphores can't, so an RT waiter can
invert on a preempted low-priority poster. The ownerless pattern that's
legitimately unique: <strong>cross-context signaling</strong> — thread A
acquires (waits) and a <em>different</em> context releases: an interrupt
handler posting "data ready" to a driver thread, a signal handler posting to
the main loop (async-signal-safe <code>sem_post</code> is explicitly
sanctioned! — Lesson 09), or pipeline stages where stage N posts what stage
N+1 waits on. The asymmetry that's a bug for locking is the feature for
signaling.
</details>

---

## Homework

Rebuild the lab's bounded queue using two semaphores (`slots = CAP`,
`items = 0`) plus one mutex, in C or pseudocode. Then answer: where did the
`while` loop go — what happened to the spurious-wakeup/re-falsification
hazards, and what new failure mode did you accept that the condvar version
didn't have?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<pre>
producer: sem_wait(&slots);           consumer: sem_wait(&items);
          mutex_lock(&m);                       mutex_lock(&m);
          buf[tail]=x; tail=(tail+1)%CAP;       x=buf[head]; head=(head+1)%CAP;
          mutex_unlock(&m);                     mutex_unlock(&m);
          sem_post(&items);                     sem_post(&slots);
</pre>
The while vanished because the semaphore <em>fuses predicate and wait</em>:
its counter IS "items available"/"slots free", and sem_wait atomically
decrements-or-sleeps — you can't wake without having already claimed one unit,
so re-falsification is impossible by construction and POSIX semaphores don't
have spurious returns (and even under EINTR you'd just retry the wait). The
accepted new failure mode: <strong>the accounting is frozen into the
counters</strong> — shutdown/poison-pills get awkward (you must post fake
"items" to release blocked consumers), you can't wait on richer predicates
("queue has a HIGH-priority item", "either new work OR shutdown"), and a
mismatched post (bug or double-post) silently corrupts capacity with no
error-checking analogue. Condvars trade mechanical safety for expressive
predicates; semaphores trade expressiveness for built-in counting — which is
why real queue implementations (like glibc's own, or Python's
<code>queue.Queue</code>) are condvar-based.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 28 — Deadlock →](lesson-28-deadlock){: .btn .btn-primary }
