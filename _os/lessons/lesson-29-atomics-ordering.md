---
title: "Lesson 29 — Atomics and Memory Ordering"
nav_order: 6
parent: "Phase 5: Concurrency"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 29: Atomics and Memory Ordering

## Concept

Locks work by *excluding* concurrency. This lesson is about the primitives
that embrace it: **atomic operations** — single instructions the hardware
guarantees are indivisible. `atomic_fetch_add(&counter, 1)` is Lesson 25's
load-add-store fused into one uninterleavable step: no lock, no sleep, no
futex — the fix for the racy counter in exactly one line.

The star primitive is **compare-and-swap (CAS)**: *"if the value still equals
`expected`, replace it with `desired` — atomically — and tell me whether you
won."* CAS is check-then-act (Lesson 25's nemesis) collapsed into hardware:

```
  retry loop — the universal lock-free pattern:
  do {
      old = load(x);
      new = f(old);                    /* compute from what you saw */
  } while (!CAS(&x, old, new));        /* commit ONLY if unchanged  */
                                       /* lost the race? loop again */
```

This is optimistic concurrency: no one blocks; a loser retries against fresh
state. It's the engine inside the futex fast path (Lesson 26), every lock-free
queue, reference counting, and — writ large — database optimistic locking and
`git push` (reject if the remote moved).

But atomics come with the price of admission to the second topic — the one
that breaks intuition: **memory ordering**. Compilers and CPUs reorder your
memory operations (store buffers, speculative loads, optimizer motion —
Lesson 25's saboteurs, now formalized). Within one thread you never notice
(the illusion of sequential execution is preserved *for you*); other threads,
watching your stores land in memory, can see them **out of order**:

```
  thread 1:  data = 42;        thread 2:  while (!ready);
             ready = true;                use(data);    ← may see data == 0 !!
  
  t1's stores can become visible in either order.  "ready" arriving first
  publishes garbage.  The fix is an ordering CONTRACT on the flag:
             data = 42;                   while (!load_acquire(&ready));
             store_release(&ready, true); use(data);    ← 42 guaranteed
```

**Release** ("everything I wrote before this store is visible to whoever...")
pairs with **acquire** ("...loads this value and reads after"). This
release/acquire handshake is *the* publication idiom — and it's what mutexes
were secretly doing all along: unlock = release, lock = acquire (Lesson 26's
"memory visibility" guarantee, now explained).

---

## How It Works

### The C11/C++11 menu (all languages copy it)

- `memory_order_relaxed` — atomic, but no ordering: right for pure counters
  (statistics) where only the total matters, never for publication.
- `release` (stores) / `acquire` (loads) — the publication pair above; the
  default mental model for flags, pointers handed between threads,
  producer/consumer indexes.
- `memory_order_seq_cst` — the strongest: all seq_cst operations appear in
  one global order everyone agrees on. The safe default (`_Atomic` operations
  without qualifiers) — measurably slower on write-heavy hot paths, boringly
  correct everywhere else.

x86 hardware is strongly ordered (only store→load reorders visibly), which
means acquire/release compile to plain instructions there — *and* means racy
code often "works on x86" then corrupts on ARM (phones, Macs, AWS Graviton):
the classic portability landmine of the 2010s.

### What atomics cannot do

One word, atomically — that's the whole offer. Two variables? No single
atomic covers them: the bank transfer (`debit(a); credit(b);`) needs both to
move together, and CAS on each separately leaves a window where money is in
neither account (checkpoint Q). Invariants spanning multiple locations are
lock territory (or transactional designs). The craft is recognizing which
side of that line a problem sits on: counters, flags, once-init pointers,
sequence numbers → atomics; multi-field consistency → locks.

### Costs: contention doesn't disappear

Lock-free ≠ contention-free. Every CAS on a shared variable still needs the
cache line exclusively (Lesson 26's ping-pong): 16 cores hammering one atomic
counter serialize on the *hardware* lock of the cache line, and failed CAS
loops burn cycles (progress guaranteed for *someone* — that's the "lock-free"
guarantee — not for *you*). Scaling patterns: per-CPU/per-thread counters
summed on read (what the kernel does for statistics everywhere), sharding,
and padding to a cache line to stop *false sharing* — the homework's subject.

{: .note }
> **`volatile` is not a synchronization keyword**
> C's <code>volatile</code> stops compiler caching of a variable, nothing
> more: no atomicity, no CPU-level ordering, no visibility contract. Its
> legitimate uses are hardware registers (Lesson 55) and
> <code>sig_atomic_t</code> flags for signal handlers (Lesson 09 — same
> thread, different rules). Every "volatile as thread flag" program is a
> latent data race — Java's <code>volatile</code> (which IS acquire/release)
> taught a generation the wrong instinct for C.

---

## Lab

```bash
# ---- 1. The one-line fix for Lesson 25 ----
$ cat > /tmp/atomic.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdatomic.h>
atomic_long counter = 0;                 /* the entire fix */
long iters;
void *inc(void *a) {
    for (long i = 0; i < iters; i++)
        atomic_fetch_add_explicit(&counter, 1, memory_order_relaxed);
    return NULL;
}
int main(int argc, char **argv) {
    iters = atol(argv[1]);
    pthread_t a, b;
    pthread_create(&a, 0, inc, 0); pthread_create(&b, 0, inc, 0);
    pthread_join(a, 0); pthread_join(b, 0);
    printf("expected %ld, got %ld\n", 2 * iters, counter);
    return 0;
}
EOF
$ gcc -pthread -O2 -o /tmp/atomic /tmp/atomic.c
$ for i in 1 2 3; do /tmp/atomic 1000000; done
# expected 2000000, got 2000000 — every time. relaxed suffices: only the
# TOTAL matters, no publication. Compare speed vs Lesson 26's mutex version!

# ---- 2. See the hardware do it: one instruction ----
$ gcc -O2 -S -o - /tmp/atomic.c 2>/dev/null | grep -A1 'lock'
#   lock addq $1, counter(%rip)        ← the LOCK prefix: bus-level atomicity.
# The mutex version compiles to dozens of instructions + potential syscalls.

# ---- 3. CAS: the retry loop, spelled out ----
$ cat > /tmp/cas.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
#include <stdatomic.h>
atomic_long max_seen = 0;
void *report(void *arg) {                /* lock-free "update max" */
    long mine = (long)arg;
    long cur = atomic_load(&max_seen);
    while (mine > cur &&
           !atomic_compare_exchange_weak(&max_seen, &cur, mine))
        ;   /* CAS failed: cur now holds the fresh value — loop re-decides */
    return NULL;
}
int main(void) {
    pthread_t t[8];
    long vals[8] = {17, 99, 3, 42, 99, 7, 88, 51};
    for (int i = 0; i < 8; i++) pthread_create(&t[i], 0, report, (void*)vals[i]);
    for (int i = 0; i < 8; i++) pthread_join(t[i], 0);
    printf("max = %ld\n", max_seen);     /* always 99, any interleaving */
    return 0;
}
EOF
$ gcc -pthread -O2 -o /tmp/cas /tmp/cas.c && for i in 1 2 3; do /tmp/cas; done
# max = 99 — check-then-act made safe WITHOUT a lock: the check is IN the act.

# ---- 4. Publication: release/acquire vs broken plain stores ----
$ cat > /tmp/pub.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
#include <stdatomic.h>
int data = 0;                            /* plain — protected by the pact */
atomic_int ready = 0;                    /* the flag carries the ordering */
void *producer(void *a) {
    data = 42;                                            /* 1: write     */
    atomic_store_explicit(&ready, 1, memory_order_release); /* 2: publish */
    return NULL;
}
void *consumer(void *a) {
    while (!atomic_load_explicit(&ready, memory_order_acquire))
        ;                                                 /* spin on flag */
    printf("data = %d %s\n", data, data == 42 ? "(guaranteed)" : "(BROKEN!)");
    return NULL;
}
int main(void) {
    pthread_t p, c;
    pthread_create(&c, 0, consumer, 0);
    pthread_create(&p, 0, producer, 0);
    pthread_join(p, 0); pthread_join(c, 0);
    return 0;
}
EOF
$ gcc -pthread -O2 -o /tmp/pub /tmp/pub.c && /tmp/pub
# data = 42 (guaranteed) — and on x86 you could weaken it and STILL see 42
# every time you test… which is exactly how the ARM-only bug gets shipped.
# TSan knows better than the hardware you happen to have:
$ sed 's/memory_order_release/memory_order_relaxed/; s/memory_order_acquire/memory_order_relaxed/' /tmp/pub.c > /tmp/pub_broken.c
$ gcc -pthread -fsanitize=thread -O1 -o /tmp/pub_tsan /tmp/pub_broken.c && /tmp/pub_tsan 2>&1 | grep -m1 'data race'
# WARNING: ThreadSanitizer: data race    ← on 'data': relaxed doesn't publish.

# ---- 5. Contention is conserved: atomic ≠ free at scale ----
$ cat > /tmp/scale.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdatomic.h>
atomic_long shared_ctr = 0;
long per_thread[8][16];                  /* [16]: pad rows to a cache line */
int mode, nthreads;
void *work(void *arg) {
    long id = (long)arg;
    for (int i = 0; i < 10000000; i++)
        if (mode) per_thread[id][0]++;                       /* private   */
        else atomic_fetch_add_explicit(&shared_ctr, 1,
                                       memory_order_relaxed); /* shared   */
    return NULL;
}
int main(int argc, char **argv) {
    mode = atoi(argv[1]); nthreads = atoi(argv[2]);
    pthread_t t[8];
    for (long i = 0; i < nthreads; i++) pthread_create(&t[i], 0, work, (void*)i);
    long total = 0;
    for (int i = 0; i < nthreads; i++) pthread_join(t[i], 0);
    for (int i = 0; i < nthreads; i++) total += per_thread[i][0];
    printf("total %ld\n", mode ? total : shared_ctr);
    return 0;
}
EOF
$ gcc -pthread -O2 -o /tmp/scale /tmp/scale.c
$ time /tmp/scale 0 4       # 4 threads, ONE shared atomic: cache-line war
$ time /tmp/scale 1 4       # 4 threads, private counters: embarrassingly fast
# often 5-20x apart. Lock-free code still queues at the cache line —
# the kernel's per-CPU counters exist precisely because of this measurement.

$ rm /tmp/atomic* /tmp/cas* /tmp/pub* /tmp/scale*
```

---

## Further Reading

| Topic | Link |
|---|---|
| Compare-and-swap | <https://en.wikipedia.org/wiki/Compare-and-swap> |
| Memory ordering | <https://en.wikipedia.org/wiki/Memory_ordering> |
| C11 memory model (cppreference) | <https://en.cppreference.com/w/c/atomic/memory_order> |
| Non-blocking algorithm | <https://en.wikipedia.org/wiki/Non-blocking_algorithm> |
| False sharing | <https://en.wikipedia.org/wiki/False_sharing> |
| MESI cache coherence | <https://en.wikipedia.org/wiki/MESI_protocol> |

---

## Checkpoint

**Q1.** An atomic counter fixed the race in one line. Why can't you build a
bank transfer (debit account A, credit account B) from two separate atomics?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Each atomic makes <em>one word's</em> update indivisible; the transfer's
correctness is an invariant across <em>two</em> words ("total money
constant", "never observe the in-between"). With
<code>atomic_sub(&a, x); atomic_add(&b, x);</code> there is an interval where
the money has left A and not reached B: a concurrent auditor summing accounts
sees money destroyed; a crash there loses it permanently; and interleaved
transfers can weave through each other's windows. No ordering annotation fixes
this — ordering controls <em>visibility sequence</em>, not <em>joint
atomicity</em>. Multi-location invariants need a mechanism that makes a
<em>group</em> of writes one event: a lock over both accounts (with ordering
discipline — Lesson 28!), a transaction (databases), or redesigning so the
invariant lives in one word (e.g., packing both balances into one CAS-able
struct — only works for tiny cases). The skill is classifying invariants by
their <em>span</em>.
</details>

**Q2.** Explain the release/acquire handshake in the producer/consumer lab:
what exactly does each side promise, and why is the *pairing* essential —
what breaks if only one side is annotated?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The release store on <code>ready</code> promises: "all my memory writes
before this store (data=42) are visible to any thread that observes this
store's value." The acquire load promises: "if I read the released value, all
my reads after this load see everything the releaser wrote before it." The
contract is a <em>pair</em> bound through the same atomic variable — a
synchronizes-with edge. One-sided annotation breaks it: release-only means the
producer pushed its writes out in order, but the consumer's CPU/compiler may
still hoist the <code>data</code> read <em>before</em> its flag loop
(speculative/reordered loads) — reading stale data before permission; while
acquire-only means the consumer would honor an ordering the producer never
established (data may reach memory after ready). Both sides, same variable, or
no guarantee — and x86's strong hardware ordering hides one-sided bugs that
ARM then exposes, which is why the discipline is "annotate the contract, not
what your desktop needs."
</details>

**Q3.** Lab step 5 showed one shared atomic being 5–20× slower than private
counters. Explain the mechanism ("lock-free has no lock — what's
serializing?"), and describe the read-side cost the per-thread design accepts.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The serializer is the cache-coherence protocol: an atomic RMW needs its cache
line in exclusive (Modified) state, so the line migrates core→core→core on
every increment — each transfer tens of ns of interconnect traffic, during
which other cores' operations on that line stall. The LOCK prefix is atomic
<em>because</em> it owns the line; 4 cores incrementing = a hardware queue for
line ownership: physically serialized, just without sleeping. (Same ping-pong
as Lesson 26's contended mutex — atomics shrink the critical section to one
instruction but can't repeal coherence.) Per-thread counters give each core
its own line (the [16] padding prevents false sharing — neighbors on one
line would reintroduce the war invisibly): writes become local cache hits.
The accepted cost: reads must <em>aggregate</em> — summing N counters that
are concurrently moving yields a slightly stale, non-atomic snapshot (fine
for statistics, wrong for exact gates), and the memory footprint × threads.
This trade is everywhere: kernel per-CPU stats, JVM LongAdder,
sharded rate limiters.
</details>

---

## Homework

**False sharing safari.** Take lab step 5's per-thread version and delete the
`[16]` padding (make `per_thread[8]` a plain `long` array so the 8 counters
share one cache line). Predict, then measure with `time` (and, if available,
`perf stat -e cache-misses`). Explain why the private-data version can be
nearly as slow as the shared-atomic version despite zero logical sharing —
and state the two standard fixes.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Measured: the unpadded version collapses toward shared-atomic speed —
typically only 1.5–3× better than the single shared counter and 5–10× worse
than padded, with cache-misses exploding. Mechanism: coherence operates on
64-byte <em>lines</em>, not variables — eight adjacent longs occupy one line,
so each thread's write invalidates every other core's copy exactly as if they
shared one variable: <strong>false sharing</strong> — all of the coherence
war, none of the actual sharing. The compiler can't save you; the data layout
did the damage. Fixes: (1) pad/align each thread's hot data to its own cache
line (<code>alignas(64)</code>, the lab's [16] longs, or allocator APIs); (2)
restructure so per-thread data is truly per-thread — thread-local storage
(Lesson 24's TLS) or per-thread heap blocks, aggregating on demand. The
general habit: in concurrent hot paths, think in cache lines — "who else is
on my 64 bytes?" belongs in code review right next to "who else takes this
lock?"
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 30 — Async and Thread-Pool Patterns →](lesson-30-async-patterns){: .btn .btn-primary }
