---
title: "Lesson 30 — Async and Thread-Pool Patterns"
nav_order: 7
parent: "Phase 5: Concurrency"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 30: Async and Thread-Pool Patterns

## Concept

You now own every low-level piece: threads (24), the races they cause (25),
locks (26), coordination (27), the failure modes (28), and lock-free
primitives (29). This capstone assembles them into the handful of *patterns*
that real systems actually use — and gives you the decision framework for
choosing.

The fundamental split is what your workload waits on:

```
  CPU-BOUND (compute limits you)          I/O-BOUND (waiting limits you)
  ────────────────────────────           ─────────────────────────────
  more threads than cores = waste        threads that WAIT are ~free but
  (context switches, no new cycles)      cost stacks & switches (L24 Q2)
        │                                       │
        ▼                                       ▼
  THREAD POOL, size ≈ nproc               EVENT LOOP (epoll — L43) on 1..few
  work queue feeds workers                threads; state machines per task
  (L27's producer/consumer               async/await = compiler writes the
   queue IS the work queue!)              state machines for you
```

**Thread pool**: N long-lived workers (N ≈ cores for compute) pull tasks from
a shared bounded queue — *exactly* Lesson 27's producer/consumer, promoted to
architecture. Amortizes thread creation, caps concurrency (no fork-bomb under
load), and the bounded queue provides **backpressure**: when workers drown,
submitters block or shed instead of silently building an infinite backlog —
the single most underrated property in systems design.

**Event loop**: one thread, thousands of in-flight operations as *state*, not
*stacks*. It blocks in one place only (epoll: "wake me when any fd is
ready" — Lesson 43 builds it), then dispatches nonblocking work. Concurrency
without parallelism: no races on loop-owned state (single thread!), no locks —
but one slow computation freezes *everyone* (the "don't block the loop"
commandment of Node/asyncio/Redis).

**async/await** is the event loop with humane syntax: each `await` marks a
point where the function's locals get packed into a heap object (the state
machine the compiler generates) and the thread moves on. Same machinery,
readable code. Goroutines/virtual threads flip it again: runtime-scheduled
stacks so cheap you write blocking code and get event-loop economics — the
runtime multiplexes M green threads onto N kernel tasks.

---

## How It Works

### Sizing the pool

CPU-bound: `nproc` (more adds switches, not cycles — Lesson 12). Mixed: the
classic estimate `threads ≈ cores × (1 + wait_time/compute_time)` — a worker
that waits 90% of its life can share its core with ~9 siblings. I/O-only at
scale: don't pool threads at all; use the event loop. And separate pools for
separate concerns (the bulkhead pattern): the request pool must not starve
because the logging pool blocked on a full disk.

### The blocking-call problem — both directions

Event loop calling blocking code (DNS lookup, file I/O — which epoll can't
cover, Lesson 44's motivation for io_uring!): the loop stalls; solution:
offload to a side thread-pool (`asyncio.to_thread`, Node's libuv pool — yes,
the event-loop runtimes carry a secret thread pool for exactly this).
Thread pool calling slow/blocked work: workers get eaten (all N stuck on one
dead dependency = pool-wide outage); solutions: timeouts everywhere, bounded
queues, circuit breakers — and monitoring queue depth as a first-class metric
(it's your earliest overload signal, ahead of latency).

### Choosing, honestly

- Mostly-idle connections in the thousands → **event loop** (Lesson 24 Q2's
  arithmetic).
- Parallel computation → **pool sized to cores**; shard data, avoid shared
  hot state (Lesson 29's scaling patterns).
- Both at once (typical server: async I/O + occasional CPU spikes) → event
  loop front + compute pool behind a queue — the two patterns composed, each
  doing what it's for.
- Simplicity budget matters → blocking threads remain *fine* for tens-to-
  hundreds of concurrent tasks; the exotic patterns buy scale, not
  correctness. Don't pay complexity for load you don't have.

{: .note }
> **The queue is the system**
> Every pattern here reduces to: producers → <em>bounded queue</em> →
> consumers. The queue's bound is your backpressure policy; its depth is your
> load gauge; its overflow behavior (block? drop? shed oldest?) is your
> degradation strategy. When a system melts under load, the autopsy almost
> always reads: "an unbounded queue somewhere." You built the correct bounded
> one in Lesson 27 — treasure it.

---

## Lab

```bash
# ---- 1. A real thread pool in ~40 lines, on your Lesson 27 queue ----
$ cat > /tmp/pool.py << 'EOF'
import threading, queue, time, os

class Pool:
    def __init__(self, n):
        self.q = queue.Queue(maxsize=32)          # BOUNDED: backpressure!
        self.workers = [threading.Thread(target=self._run, daemon=True)
                        for _ in range(n)]
        [w.start() for w in self.workers]
    def _run(self):
        while (job := self.q.get()) is not None:  # sentinel shutdown (L27!)
            fn, args = job
            try: fn(*args)
            except Exception as e: print("job failed:", e)
            self.q.task_done()
    def submit(self, fn, *args):
        self.q.put((fn, args))                    # BLOCKS when full ← policy
    def shutdown(self):
        self.q.join()
        for _ in self.workers: self.q.put(None)

def io_task(n):     time.sleep(0.05)              # "network call"
def cpu_task(n):    sum(i*i for i in range(200_000))

pool = Pool(os.cpu_count())
t = time.time()
for i in range(64): pool.submit(io_task, i)
pool.shutdown()
print(f"64 I/O tasks, {os.cpu_count()} workers: {time.time()-t:.2f}s")
# ~64*0.05/nproc — waiting parallelizes fine even in Python (GIL releases on I/O)
EOF
$ python3 /tmp/pool.py

# ---- 2. Pool sizing: watch the CPU-bound crossover ----
$ python3 - << 'EOF'
import concurrent.futures as cf, time, os
def cpu(n): return sum(i*i for i in range(300_000))
for workers in (1, os.cpu_count(), os.cpu_count()*4):
    t = time.time()
    with cf.ProcessPoolExecutor(workers) as ex:      # processes: real parallelism
        list(ex.map(cpu, range(32)))
    print(f"{workers:3d} workers: {time.time()-t:.2f}s")
EOF
# 1 → slow; nproc → ~nproc× faster; 4×nproc → NO faster (often slower).
# Cores are the budget; extra workers just buy context switches (L12).

# ---- 3. The event loop: 2000 concurrent "connections", one thread ----
$ python3 - << 'EOF'
import asyncio, time, threading
async def session(i):
    await asyncio.sleep(0.5)          # 2000 of these IN FLIGHT at once
    return i
async def main():
    t = time.time()
    await asyncio.gather(*[session(i) for i in range(2000)])
    print(f"2000 concurrent sessions: {time.time()-t:.2f}s "
          f"on {threading.active_count()} thread(s)")
asyncio.run(main())
EOF
# ~0.5s total, 1 thread. 2000 blocking threads would need ~16GB of stacks (L24).

# ---- 4. Block the loop, freeze the world — then fix it ----
$ python3 - << 'EOF'
import asyncio, time
async def heartbeat():
    for _ in range(6):
        print(f"  beat {time.strftime('%S')}s")
        await asyncio.sleep(0.5)
def evil_blocking_call(): time.sleep(2)          # sync sleep: HOLDS the thread
async def main():
    hb = asyncio.create_task(heartbeat())
    await asyncio.sleep(0.7)
    print("blocking call on the loop thread:")
    evil_blocking_call()                          # beats STOP for 2 full seconds
    print("fixed version (offloaded to side pool):")
    await asyncio.to_thread(evil_blocking_call)   # beats continue!
    await hb
asyncio.run(main())
EOF
# the gap in heartbeats is every user's request freezing at once —
# and to_thread is the event loop borrowing pattern #1. Composition!

# ---- 5. Backpressure vs the unbounded queue disaster ----
$ python3 - << 'EOF'
import queue, threading, time
def flood(q, label):
    produced = 0
    t = time.time()
    try:
        while time.time() - t < 1.0:
            q.put_nowait(("x" * 1000)); produced += 1
    except queue.Full: pass
    print(f"{label}: producer got {produced} in; queue holds {q.qsize()}")
bounded = queue.Queue(maxsize=100)
unbounded = queue.Queue()
threading.Thread(target=flood, args=(bounded, "bounded  ")).start()
threading.Thread(target=flood, args=(unbounded, "unbounded")).start()
time.sleep(1.2)
EOF
# bounded: producer stopped at 100 — overload is VISIBLE at the source.
# unbounded: hundreds of thousands queued — memory grows until the OOM
# killer (L22) writes the incident report. The bound IS the feature.
$ rm /tmp/pool.py
```

---

## Further Reading

| Topic | Link |
|---|---|
| Thread pool | <https://en.wikipedia.org/wiki/Thread_pool> |
| Event loop | <https://en.wikipedia.org/wiki/Event_loop> |
| Async/await | <https://en.wikipedia.org/wiki/Async/await> |
| Coroutine | <https://en.wikipedia.org/wiki/Coroutine> |
| Python asyncio docs | <https://docs.python.org/3/library/asyncio.html> |
| Back pressure (Wikipedia: flow control) | <https://en.wikipedia.org/wiki/Flow_control_(data)> |

---

## Checkpoint

**Q1.** A service handles 10k mostly-idle connections. Threads-per-connection
or event loop? Justify with numbers from this phase.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Event loop. Threads-per-connection at 10k: ~8 MiB stack each = 80 GB virtual /
several GB touched RSS (Lesson 24), 10k task_structs for the scheduler to
balance, and on every traffic burst, wakeup storms of context switches at
~1–2 µs each plus cache damage (Lesson 12) — for connections that are 99%
idle and hold no computation. The event loop stores each connection as a few
KB of state (buffers + state machine): 10k connections ≈ tens of MB, one
epoll_wait covers them all, and ~nproc threads do all execution. The
thread-per-connection model spends an execution context to represent
<em>waiting</em>; epoll represents waiting as data. Accepted costs: async
programming model, and strict discipline about blocking calls on the loop
(lab step 4) — with CPU spikes offloaded to a pool (composition).
</details>

**Q2.** Why must a thread pool's work queue be *bounded*, and what are the
three standard policies when it fills? Give a failure story for the unbounded
choice.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An unbounded queue converts overload into hidden debt: producers never feel
resistance, so arrival &gt; service just accumulates — memory grows (until
Lesson 22's OOM killer ends the story), and every queued item's latency grows
unboundedly (items served minutes late are usually worthless: timed-out
clients already retried, doubling load — the retry storm spiral). The bound
makes overload <em>immediate and local</em>: the system pushes back at the
edge while still healthy. Full-queue policies: (1) <strong>block</strong> the
submitter — backpressure propagates upstream (right for internal pipelines,
your Lesson 27 queue); (2) <strong>reject/shed</strong> — fail fast with an
error (right at service edges: a quick 503 beats a slow timeout, and clients'
backoff does the rest); (3) <strong>drop/degrade</strong> — discard oldest or
lowest-value items (right for telemetry, caches, best-effort work). Choosing
per queue — and monitoring depth — is capacity planning made concrete.
</details>

**Q3.** async/await code has no locks, yet Lesson 25-style corruption can
still occur. Where does the single-threaded event loop's atomicity guarantee
end, and what's the async-world equivalent of a critical section?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The guarantee: code between two <code>await</code>s runs without interleaving
— the loop switches tasks <em>only at await points</em>, so synchronous
sections are naturally atomic (no data races on loop-owned state). It ends at
every <code>await</code>: the moment a coroutine suspends, any other task may
run and mutate shared state — so check-then-act <em>spanning an await</em>
(read balance, <code>await</code> the DB, subtract) is exactly Lesson 25's
bank bug, cooperative edition. The async critical section: either keep the
invariant's read-modify-write purely synchronous (no await inside — often the
cleanest fix), or use <code>asyncio.Lock</code> — same acquire/release
semantics, but suspending instead of blocking — around multi-await sequences.
Same discipline as threads, coarser interleaving points: races didn't
disappear, they just got scheduled politely.
</details>

---

## Homework

Design (on paper) the concurrency architecture for a thumbnail service:
HTTP endpoint receives image URLs; service downloads each image (network,
~200 ms), resizes it (CPU, ~50 ms), uploads the result (network, ~100 ms).
Target: 500 requests/s on an 8-core box. Specify: the pattern(s), pool/loop
sizes with arithmetic, every queue and its bound + overflow policy, and the
two failure scenarios your design must survive (name the mechanism that
saves each).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Composed design: <strong>event loop</strong> (async HTTP + downloads +
uploads — it's all waiting) front, <strong>process/thread pool</strong> for
resizing behind a bounded queue. Arithmetic: CPU demand = 500/s × 50 ms =
25 core-seconds/s… which exceeds 8 cores — the honest conclusion first: as
specified it needs ~4 such boxes, or ~works at 150/s per box; size the resize
pool at 8 (= cores, CPU-bound — Lesson 12/lab 2). I/O in flight ≈ 500/s ×
0.3 s ≈ 150 concurrent downloads+uploads — trivial for one loop (lab 3).
Queues: (a) HTTP accept → in-flight cap ~200, overflow = reject 503 (edge:
shed fast); (b) downloaded-images → resize-pool queue, bound ~32
(≈ 2× workers×latency headroom), overflow = <em>block the downloader
coroutine</em> — backpressure naturally slows intake, which then trips (a)'s
shedding under sustained overload; (c) resized → upload queue, bound ~64,
block. Failure scenarios: (1) <em>slow origin server</em> — downloads take
5 s: per-download timeouts + the in-flight cap keep the loop's memory bounded;
without them, half a gigabyte of stuck buffers (Q2's debt). (2) <em>resize
pool wedged</em> (poison image, library hang): queue (b) fills → backpressure
→ 503s at the edge while healthy requests drain; watchdog timeouts on resize
jobs (worker kills + job requeue-once) recover the pool — bulkheaded so
uploads/downloads never stall for the CPU stage. Every number is monitorable:
queue depths are the dashboard.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 6 — IPC (Lesson 31: Pipes and FIFOs) →](lesson-31-pipes-fifos){: .btn .btn-primary }
