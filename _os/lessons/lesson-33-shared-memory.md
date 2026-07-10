---
title: "Lesson 33 — Shared Memory"
nav_order: 3
parent: "Phase 6: Inter-Process Communication"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 33: Shared Memory

## Concept

Pipes and sockets share a tax: every message is copied twice (sender's buffer
→ kernel → receiver's buffer), and every send/receive is a syscall (Lesson
02's price list). For a chatty pair of processes moving serious data, the tax
dominates. The escape is to stop *sending* altogether:

**Shared memory** maps the *same physical frames* into both processes'
address spaces (Lesson 16's fiction, deliberately overlapped):

```
  process A's fiction              physical RAM             process B's fiction
  ─────────────────                ────────────             ─────────────────
  0x7f10... ──────────┐          ┌──────────────┐         ┌────────── 0x7f88...
                      └────────▶ │ SHARED frames │ ◀───────┘
                                 └──────────────┘
  A writes a byte  →  B reads it: NO syscall, NO copy, NO kernel involvement.
  It's just... memory. At memory speed.

  which is also the problem: two processes, one mutable memory —
  you are BACK IN PHASE 5. Every race, every lock, every ordering rule.
```

The modern POSIX recipe: `shm_open("/name", ...)` creates a named object
(visible as a file in `/dev/shm` — a tmpfs, so it *is* page cache, Lesson
39's preview), `ftruncate` sizes it, and both processes `mmap` it
`MAP_SHARED`. From then on, loads and stores — that's the entire API.

Who uses it: PostgreSQL's buffer pool (all backends share gigabytes of
cache), Python's `multiprocessing.shared_memory`, browsers (compositor ↔
renderer frames), games/audio (ring buffers to the sound server), science
(NumPy arrays across workers) — anywhere the data is big and the peers are
chatty. And you've met it from below twice: `MAP_SHARED` file mappings are
"shared memory with a disk identity" (Lesson 20 — the page cache is the
sharing), and vhost-user/virtiofs (virt Lesson 30) hand *guest RAM* to
another process by exactly this mechanism.

---

## How It Works

### Synchronization comes back — process-grade

Loads and stores need coordinating: Phase 5's toolbox works across processes
with one adjustment each — mutexes/condvars need
`pthread_mutexattr_setpshared(PTHREAD_PROCESS_SHARED)` and must *live in the
shared region* (a lock both sides can see!); POSIX semaphores go in shared
memory as unnamed (`sem_init(pshared=1)`) or standalone as named
(`sem_open("/name")`); atomics (Lesson 29) work unchanged — cache coherence
doesn't care about process boundaries, only about physical addresses. The
futex (Lesson 26) was *designed* for this: its address-based wait queues
work on shared pages (that's the "shared" futex flavor).

One process-world hazard threads don't have: **death while holding the
lock**. A thread dying takes the process; a *process* dying leaves the shared
mutex locked forever for the survivors. The cure:
`pthread_mutexattr_setrobust` — the next locker gets `EOWNERDEAD`, repairs
the data, and calls `pthread_mutex_consistent`. Robustness is why
production shared-memory systems (Postgres!) design crash-recovery into
their shared state from day one.

### Lifecycle and accounting

The object outlives its creators (like a file): `shm_unlink` removes the
name; the memory frees when the last mapping goes. Forgotten objects in
`/dev/shm` are a classic leak (they're RAM!). Sizing: `/dev/shm` defaults to
50% of RAM; shared pages count once physically but appear in every mapper's
RSS (PSS — Lesson 18 — tells the truth). And `memfd_create` is the anonymous
modern cousin: a shm object with *no name at all*, shared by passing its fd
over a Unix socket (Lesson 32's SCM_RIGHTS!) — no /dev/shm janitorial duty,
capability-style access, sealable against resizing (F_SEAL) — the mechanism
under Wayland buffers and modern browser IPC.

{: .note }
> **SysV shm: the ancestor you'll still meet**
> <code>shmget/shmat</code> + <code>ipcs -m</code> is the 1980s API: numeric
> keys instead of names, no fd, quirky permissions. PostgreSQL used it for
> decades (now mostly mmap). If <code>ipcs</code> shows segments on a box,
> that's what you're looking at; new code has no reason to choose it.

---

## Lab

```bash
# ---- 1. Two processes, one memory: the core demo ----
$ python3 - << 'EOF'
from multiprocessing import shared_memory
import os, time

shm = shared_memory.SharedMemory(create=True, size=4096, name="lab")
if os.fork() == 0:                       # child (could be ANY process!)
    peer = shared_memory.SharedMemory(name="lab")   # attach by NAME
    peer.buf[0:5] = b"HELLO"             # a plain memory write
    peer.close()
    os._exit(0)
os.wait()
print("parent reads:", bytes(shm.buf[0:5]))   # no recv(), no read(): just RAM
shm.close(); shm.unlink()
EOF
# parent reads: b'HELLO'

# ---- 2. It's a file in /dev/shm (which is a tmpfs = page cache) ----
$ python3 -c "
from multiprocessing import shared_memory
s = shared_memory.SharedMemory(create=True, size=100*1024*1024, name='peek')
input('created — inspect from another view: ')" &
$ sleep 0.5; ls -lh /dev/shm/
# -rw------- ... 100M peek        ← named, permissioned, mmap-able: a file
$ df -h /dev/shm | tail -1        # the tmpfs budget it lives in
$ kill %1; rm -f /dev/shm/peek    # cleanup habit! forgotten objects = leaked RAM

# ---- 3. The speed argument: shm vs pipe, one million messages ----
$ python3 - << 'EOF'
import os, time, mmap

N = 1_000_000
# pipe version: 2 syscalls + 2 copies per message
r, w = os.pipe()
if os.fork() == 0:
    for i in range(N): os.write(w, b"12345678")
    os._exit(0)
t = time.time()
got = 0
while got < N * 8: got += len(os.read(r, 65536))
pipe_s = time.time() - t
os.wait()

# shm version: plain stores + one atomic flag (no kernel in the loop)
buf = mmap.mmap(-1, 16, flags=mmap.MAP_SHARED | mmap.MAP_ANONYMOUS)
if os.fork() == 0:
    for i in range(N):
        buf[0:8] = (i).to_bytes(8, 'little')
    buf[8] = 1                            # done flag
    os._exit(0)
t = time.time()
while buf[8] == 0: pass                   # spin (lab-only!)
shm_s = time.time() - t
os.wait()
print(f"pipe: {pipe_s:.2f}s   shm: {shm_s:.2f}s   → {pipe_s/shm_s:.0f}x")
EOF
# shm typically 10-50x faster — the copies and syscalls were the cost.

# ---- 4. And the danger: Lesson 25's race, now BETWEEN processes ----
$ python3 - << 'EOF'
import os, mmap, struct
buf = mmap.mmap(-1, 8, flags=mmap.MAP_SHARED | mmap.MAP_ANONYMOUS)
buf[0:8] = (0).to_bytes(8, 'little')
def inc_loop():
    for _ in range(200_000):
        v = int.from_bytes(buf[0:8], 'little')      # load
        buf[0:8] = (v + 1).to_bytes(8, 'little')    # store — NOT atomic!
pids = [os.fork() for _ in range(2)]
if 0 in pids:
    inc_loop(); os._exit(0)
for _ in range(2): os.wait()
got = int.from_bytes(buf[0:8], 'little')
print(f"expected 400000, got {got} — processes race EXACTLY like threads")
EOF
# lost updates across process boundaries. Phase 5 is not optional here.

# ---- 5. memfd: anonymous shm handed over as a capability ----
$ python3 - << 'EOF'
import os, socket, array, mmap
fd = os.memfd_create("secret")            # no name anywhere in the filesystem
os.ftruncate(fd, 4096)
m = mmap.mmap(fd, 4096); m[0:4] = b"gift"
a, b = socket.socketpair()
if os.fork() == 0:                        # receiver gets it ONLY via the socket
    a.close()
    _, anc, _, _ = b.recvmsg(10, socket.CMSG_SPACE(4))
    rfd = array.array("i", anc[0][2])[0]
    rm = mmap.mmap(rfd, 4096)
    print("receiver sees:", bytes(rm[0:4]), "— shared, yet unnameable")
    os._exit(0)
b.close()
a.sendmsg([b"x"], [(socket.SOL_SOCKET, socket.SCM_RIGHTS, array.array("i", [fd]))])
os.wait()
EOF
# Lesson 32's fd-passing + this lesson's mapping = capability-based shared
# memory: only processes GIVEN the fd can ever attach. (Wayland does this.)
```

---

## Further Reading

| Topic | Link |
|---|---|
| `shm_open(3)` man page | <https://man7.org/linux/man-pages/man3/shm_open.3.html> |
| `shm_overview(7)` | <https://man7.org/linux/man-pages/man7/shm_overview.7.html> |
| Shared memory (Wikipedia) | <https://en.wikipedia.org/wiki/Shared_memory> |
| `memfd_create(2)` man page | <https://man7.org/linux/man-pages/man2/memfd_create.2.html> |
| `pthread_mutexattr_setpshared(3)` | <https://man7.org/linux/man-pages/man3/pthread_mutexattr_setpshared.3.html> |
| `pthread_mutexattr_setrobust(3)` | <https://man7.org/linux/man-pages/man3/pthread_mutexattr_setrobust.3.html> |

---

## Checkpoint

**Q1.** Shared memory is the fastest IPC. What exactly did you give up
compared to a pipe? List the pipe features you must now rebuild by hand.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The pipe bundled four services you now owe yourself: (1)
<strong>synchronization</strong> — pipes serialize access in the kernel; shm
resurrects all of Phase 5 (locks/atomics in the shared region, process-shared
flavors, robustness against holder death); (2) <strong>flow control</strong> —
a full pipe blocks the writer (built-in backpressure, Lesson 31); shm has no
"full": you build ring buffers with explicit head/tail and waiting; (3)
<strong>lifecycle signaling</strong> — EOF and SIGPIPE tore pipelines down
automatically; shm gives no notice when a peer dies mid-update (robust
mutexes, heartbeats, or a side-channel socket restore it); (4) <strong>copy
semantics</strong> — a pipe hands the receiver a private copy; shm peers see
each other's <em>in-progress</em> writes (torn reads without ordering
discipline — Lesson 29). The honest summary: shm is not a channel, it's a
<em>place</em>; everything channel-like about it is your code.
</details>

**Q2.** Why must a mutex protecting shared-memory data live *inside* the
shared region, and what extra attribute does it need beyond
PTHREAD_PROCESS_SHARED for production use? Explain the failure it prevents.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A mutex is state (Lesson 26: an integer the futex protocol operates on) —
for two processes to contend on <em>one lock</em>, both must address the
<em>same memory</em>: a mutex in either process's private memory is invisible
to the other (each would lock its own copy — no exclusion at all). So the
lock goes into the shared mapping, initialized once with the
PROCESS_SHARED attribute (which also makes the futex use shared addressing).
The production attribute: <strong>robustness</strong>
(<code>pthread_mutexattr_setrobust</code>). Failure prevented: process A
crashes (or is OOM-killed — Lesson 22!) while holding the lock; without
robustness the mutex stays locked forever — every other process queues behind
a corpse (Lesson 28's deadlock, with no cycle needed). With it, the next
locker gets EOWNERDEAD — "you have the lock, but the protected data may be
half-written" — repairs invariants, marks consistent, proceeds. That
repair-path requirement is the real cost of shm: your data structures need
crash-consistent design, not just locking.
</details>

**Q3.** PostgreSQL keeps its multi-gigabyte buffer pool in shared memory
accessed by dozens of backend processes, rather than using threads in one
process (where sharing is automatic). Reconstruct the reasoning — what does
the process model buy that's worth re-plumbing "sharing" manually?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Fault isolation with <em>chosen</em> sharing. In a threaded design (Lesson
24), every byte is implicitly shared: one backend's wild pointer, allocator
corruption, or extension bug can silently trash any other session's memory —
and one segfault kills every connection (Lesson 24's lab 5). With processes +
explicit shm, the blast radius is inverted: private memory (each session's
parse trees, sort buffers, extension code) is isolated by the MMU (Lesson
16), and only the deliberately shared, carefully disciplined structures —
buffer pool, lock tables, WAL buffers — are exposed, each guarded by robust
locks with crash-recovery paths (Q2). A backend crash costs one connection;
the postmaster detects it, and because shared state was designed for
consistency, decides between per-backend cleanup and (conservatively)
restarting the shm world. Extra dividends: per-session credentials/cgroup
control (Lessons 11/60) and debuggability (which backend = which PID). Cost:
connections are heavy (hence poolers) and everything shared is hand-built —
the exact trade Lesson 24's homework mapped, chosen deliberately by a system
whose top requirement is not corrupting your data.
</details>

---

## Homework

Design (and optionally build in Python) a **single-producer single-consumer
ring buffer** in shared memory: a 4 KiB data area, head and tail indexes, the
producer writes variable-length records, the consumer reads them. Specify:
which index each side may write, why SPSC needs *no lock* if head/tail
updates are atomic with the right ordering (Lesson 29!), how "full" and
"empty" are distinguished, and what breaks the moment you allow a second
producer.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Layout: <code>[head][tail][4KB data]</code> — <strong>only the consumer
writes head</strong> (where it has read up to), <strong>only the producer
writes tail</strong> (where it has written up to); each side merely
<em>reads</em> the other's index. That single-writer-per-variable property is
why SPSC is lock-free with just ordering: the producer writes the record
bytes first, then does a <em>release</em> store of the new tail; the consumer
does an <em>acquire</em> load of tail — the release/acquire pair (Lesson 29's
publication idiom) guarantees the consumer never sees the tail advance before
the record bytes are visible. Empty = head == tail; full = (tail + 1) % size
== head — sacrificing one slot resolves the ambiguity (the alternative:
separate count, which would need shared writing — worse). Blocking (instead
of spinning) when empty/full: add an eventfd or futex wait on the indexes —
next lesson's primitives slot straight in. A second producer breaks the
foundation: two writers of tail race (lost updates — Lesson 25), and even
with CAS on tail, the <em>record area</em> between old and new tail must be
claimed atomically with it — you now need either a lock or a genuinely
subtle MPSC design (claim-then-publish with per-slot sequence numbers — what
Disruptor/io_uring's rings do; io_uring, Lesson 44, is literally two SPSC
rings between you and the kernel).
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 34 — Message Queues and D-Bus →](lesson-34-message-queues){: .btn .btn-primary }
