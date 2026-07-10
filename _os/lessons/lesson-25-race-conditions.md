---
title: "Lesson 25 — Race Conditions: Build One, Watch It Fail"
nav_order: 2
parent: "Phase 5: Concurrency"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 25: Race Conditions — Build One, Watch It Fail

## Concept

Lesson 24 handed every thread the same memory. Today you learn why that gift
is booby-trapped — not by reading about it, but by writing a program that is
*obviously correct* and watching it produce wrong answers.

The trap hides in plain sight. `counter++` looks atomic. It is three
operations:

```
   thread A                        thread B
   ────────                        ────────
   load  counter (=100)
                    ← preempted! (L12 — any instruction boundary)
                                   load  counter (=100)
                                   add   1        (=101)
                                   store counter  (=101)
   add   1        (=101)
   store counter  (=101)     ← B's increment is GONE. 2 increments, +1 total.
```

This is a **lost update** — the canonical **race condition**: correctness
depends on the *timing* of interleaved operations, which nothing controls.
The scheduler preempts at any instruction (Lessons 05/12); on multicore, A and
B execute *simultaneously* and the interleaving is decided by cache hardware.

What makes races uniquely evil among bugs:

- **They're probabilistic.** The lab's counter is sometimes correct. Tests
  pass; production at 3 a.m. doesn't.
- **They're silent.** No fault, no signal, no log line — just wrong data,
  discovered arbitrarily later.
- **They scale with success.** More cores, more load = more interleavings =
  more corruption. The race ships quiet and detonates at peak.
- **Observation changes them.** Add a printf, the race vanishes (timing
  shifted) — the classic *heisenbug*.

Terminology worth precision: a **data race** is the low-level crime — two
threads access the same memory, at least one writes, no synchronization (in
C/C++ this is undefined behavior *even when it looks harmless*). A **race
condition** is the broader design flaw — timing-dependent correctness — which
survives even data-race-free code (check-then-act across two atomic steps:
next lesson's homework). Data races are mechanically detectable; race
conditions require thought.

---

## How It Works

### Why it's worse than interleaving: caches and compilers

The three-instruction story is the *optimistic* version. Reality adds two
saboteurs. **Hardware**: each core works against its own cache (Lesson 12);
without synchronization, a store by core 0 becomes visible to core 1 *later*,
possibly reordered relative to other stores (the full story is Lesson 29).
**Compiler**: absent synchronization, the optimizer may keep `counter` in a
register for the whole loop, hoist reads, or merge "redundant" writes — your
three instructions might be one, or thousands apart. The C standard blesses
all of it because data races are UB. Moral: you cannot reason about racy code
by imagining its assembly.

### The detector: ThreadSanitizer

`gcc -fsanitize=thread` instruments every memory access and lock operation; at
runtime it reports *actual data races* — with stack traces for both sides.
5–15× slowdown: a test-suite tool, not production. It finds data races, not
logical race conditions — but that's most of them, with essentially no false
positives. Non-negotiable for concurrent C/C++; Go's `-race` is the same idea
built in.

{: .note }
> **Races aren't only about threads**
> The same disease infects any concurrent access to shared mutable state: two
> processes appending to one file, check-then-create on a path (the TOCTOU
> security bug class), read-modify-write against a database without
> transactions, two `kubectl apply`s. This phase's vocabulary transfers to all
> of them; only the lock spellings differ.

---

## Lab

```bash
# ---- 1. The obviously-correct wrong program ----
$ cat > /tmp/race.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
long counter = 0;
long iters;
void *inc(void *arg) {
    for (long i = 0; i < iters; i++)
        counter++;                      /* load, add, store — three ops */
    return NULL;
}
int main(int argc, char **argv) {
    iters = atol(argv[1]);
    pthread_t a, b;
    pthread_create(&a, NULL, inc, NULL);
    pthread_create(&b, NULL, inc, NULL);
    pthread_join(a, NULL); pthread_join(b, NULL);
    printf("expected %ld, got %ld  (lost %ld)\n",
           2 * iters, counter, 2 * iters - counter);
    return 0;
}
EOF
$ gcc -pthread -O0 -o /tmp/race /tmp/race.c

# ---- 2. Run it five times. Same program, five answers. ----
$ for i in 1 2 3 4 5; do /tmp/race 1000000; done
# expected 2000000, got 1103482  (lost 896518)
# expected 2000000, got 1245901  (lost 754099)
# expected 2000000, got 2000000  (lost 0)        ← "it works on my machine"
# expected 2000000, got  998237  (lost 1001763)  ← >50% of updates GONE
# expected 2000000, got 1367112  (lost 632888)

# ---- 3. Shrink the window: small iteration counts "never" fail ----
$ for i in 1 2 3 4 5; do /tmp/race 1000; done
# likely five perfect results — this is why your unit tests pass.
# The race is still there. Only the probability moved.

# ---- 4. The optimizer makes it stranger ----
$ gcc -pthread -O2 -o /tmp/race2 /tmp/race.c && for i in 1 2 3; do /tmp/race2 1000000; done
# results change character entirely (often exactly 1000000 —
# each thread's loop collapsed to counter += 1000000: ONE racy add each).
# UB means: any of these behaviors is "correct" compilation of your bug.

# ---- 5. ThreadSanitizer: the accusation, with evidence ----
$ gcc -pthread -fsanitize=thread -O1 -o /tmp/race_tsan /tmp/race.c
$ /tmp/race_tsan 10000 2>&1 | head -15
# WARNING: ThreadSanitizer: data race
#   Write of size 8 ... by thread T2:   #0 inc /tmp/race.c:9
#   Previous write of size 8 ... by thread T1: #0 inc /tmp/race.c:9
#   ← file:line of BOTH sides. It even fires when the count comes out right:
#     tsan detects the race itself, not the corruption.

# ---- 6. Check-then-act: the race that has no data race ----
$ python3 - << 'EOF'
import threading
balance = 100
def withdraw(amount):
    global balance
    if balance >= amount:            # CHECK
        # scheduler can switch here!
        balance = balance - amount   # ACT (GIL makes each line safe — 
                                     #      the GAP is still a race!)
threads = [threading.Thread(target=withdraw, args=(100,)) for _ in range(2)]
[t.start() for t in threads]; [t.join() for t in threads]
print(f"balance: {balance}")         # run repeatedly: sometimes -100!
EOF
# two withdrawals of 100 from a balance of 100 both pass the CHECK.
# No shared-memory data race (thanks GIL) — pure logical race condition.

$ rm /tmp/race /tmp/race2 /tmp/race_tsan /tmp/race.c
```

---

## Further Reading

| Topic | Link |
|---|---|
| Race condition | <https://en.wikipedia.org/wiki/Race_condition> |
| ThreadSanitizer | <https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual> |
| Time-of-check to time-of-use (TOCTOU) | <https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use> |
| Heisenbug | <https://en.wikipedia.org/wiki/Heisenbug> |
| Linearizability | <https://en.wikipedia.org/wiki/Linearizability> |

---

## Checkpoint

**Q1.** The racy counter is *sometimes* correct. Why does that make races
worse, not better?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because correctness-by-luck defeats every quality process built on
observation: tests pass (small iteration counts and quiet CI machines shrink
the window to near zero — lab step 3), code review sees plausible logic,
staging behaves, and the bug reveals itself only under production load,
non-reproducibly, as silently wrong data rather than a crash. A deterministic
bug is found once and fixed; a probabilistic one is "fixed" repeatedly by
whoever changes the timing (adding logging!), then returns. It also breaks the
debugging feedback loop itself — attaching a debugger or adding prints alters
scheduling enough to hide it. This is why the discipline is <em>prevention
and detection by construction</em> (locks, tsan, design) rather than testing:
you cannot test your way out of a probability.
</details>

**Q2.** Step 4 showed `-O2` making each thread race just once with a whole
million pre-summed. A colleague concludes "so with -O2 the code is *nearly*
safe." Correct them precisely.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The observation is an artifact, not a guarantee. The code contains a data
race, which is undefined behavior — the compiler is entitled to produce
<em>any</em> code, and "collapsed to one add" is merely what this compiler
version chose today; a different optimization level, compiler, surrounding
code, or inlining decision produces different behavior with no warning.
"Nearly safe" also misreads the result: one racy read-modify-write of a
million-unit quantum means when it does lose, it loses a million at once —
lower probability, catastrophic magnitude. UB reasoning is the key correction:
once the program races, you don't have a slightly-wrong program, you have no
defined program at all — the fix is synchronization (next lesson), never
optimizer archaeology.
</details>

**Q3.** Step 6's bank withdrawal has no data race (the GIL serializes each
bytecode), yet it double-spends. Name the pattern, explain why "make every
operation atomic" doesn't fix it, and state the general shape of the real fix.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Check-then-act</strong> (test-and-set race, TOCTOU's sibling): the
decision (balance ≥ 100) and the action (deduct) are individually safe but
separated by a window in which another thread invalidates the decision. Making
each step atomic is exactly what the GIL already did — the bug is that the
<em>invariant spans both steps</em>: correctness requires "no one touches
balance between my check and my act". The general fix: make the check and act
one <em>indivisible critical section</em> — hold a lock across both (Lesson
26), or use a primitive that fuses them (compare-and-swap: "deduct only if
still ≥100" — Lesson 29; databases: transactions/SELECT FOR UPDATE; files:
O_EXCL create). The lesson generalizes: atomicity is a property of
<em>invariants</em>, not of instructions — always ask "what must be true
across which span?" before asking "which operation should be atomic?"
</details>

---

## Homework

Find the race in this real-world-shaped code and describe two failure modes
and the fix:

```python
# log rotation, called from multiple worker threads
def get_log_file():
    global log_file
    if log_file is None or log_file.size() > MAX:
        if log_file: log_file.close()
        log_file = open(new_name(), "w")
    return log_file
```

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It's check-then-act on shared mutable state (<code>log_file</code>), with an
extra cruelty: the act has three sub-steps (close, open, assign). Failure
mode 1 — <strong>double rotation</strong>: threads A and B both see size &gt;
MAX, both close/reopen: two new files created, one leaked immediately; log
lines split across files; the close of a file the other thread is mid-write
to. Failure mode 2 — <strong>use-after-close</strong>: thread A passes the
check, gets preempted; B rotates (closes the file A is about to return); A
returns the closed handle and every subsequent write raises (or worse, with
fd reuse — Lesson 36! — writes into an unrelated file B just opened). Fix:
one lock around the whole check-and-swap (<code>with rotate_lock:</code>
re-check size <em>inside</em> the lock — the double-checked pattern), and
callers must receive the handle from within the same protected call. Cleaner
still: a single writer thread owning the file, workers sending lines through
a queue (Lesson 27's producer/consumer — turning shared-state racing into
message passing).
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 26 — Mutexes and Futexes →](lesson-26-mutex-futex){: .btn .btn-primary }
