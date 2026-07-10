---
title: "Lesson 44 — io_uring"
nav_order: 2
parent: "Phase 8: Event-Driven & Async I/O"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 44: io_uring

## Concept

Two problems survived Lesson 43. **Files**: epoll refuses regular files —
their latency hides inside "always-ready" reads, so event loops smuggle
file I/O onto thread pools. **Syscalls**: even a perfect epoll loop pays one
syscall per operation (Lesson 02's toll), and post-Meltdown mitigations
(Lesson 17) made every crossing pricier.

**io_uring** (2019, Jens Axboe) attacks both with a change of *model* — from
readiness to **completion**:

```
  READINESS (epoll):  "tell me WHEN I CAN do the I/O"
                      → you still do it: one syscall per op, and it
                        must be nonblocking-able (bye, files)

  COMPLETION (io_uring): "DO this I/O; tell me when it's DONE"
                      → the kernel does it — any I/O, files included;
                        you collect results.

  the mechanism: TWO RINGS in SHARED MEMORY (L33's SPSC rings, verbatim!)

   your process                          kernel
   ┌──────────────────────────┐
   │  SQ (submission queue)   │ ──────▶  workers/async engine
   │  "read fd 7, 4KB, off 0" │          execute operations
   │  "write fd 3 …"          │
   │  ...batch of entries...  │
   │  CQ (completion queue)   │ ◀──────  "op #1: 4096 bytes, ok"
   └──────────────────────────┘
   submit N ops: ONE io_uring_enter() … or ZERO (SQPOLL mode:
   a kernel thread polls the ring — syscall-free I/O!)
```

Both rings live in memory shared between you and the kernel (mmap'd at
setup): producing an entry is writing a struct and bumping a tail index —
Lesson 33's homework ring buffer, deployed at the kernel boundary, with
Lesson 29's ordering rules doing the synchronization. One
`io_uring_enter()` submits a *batch* and can wait for completions in the
same call: the syscall's fixed cost amortized (Lesson 41 Q2's group-commit
economics, applied to syscalls themselves).

---

## How It Works

### The primitives

Setup: `io_uring_setup(entries, params)` → an fd; mmap the SQ, CQ, and SQE
array. Each **SQE** (submission entry) is one operation: opcode
(READ/WRITE/ACCEPT/SEND/RECV/OPENAT/FSYNC/TIMEOUT/... — 40+ and growing),
fd, buffer, offset, flags, and a `user_data` cookie you choose. Each **CQE**
(completion) returns `user_data` + result (bytes or -errno). The cookie is
how you match completions to your state machines — the event loop's
dispatch, relocated.

liburing (`io_uring_get_sqe`, `io_uring_submit`, `io_uring_wait_cqe`) wraps
the ring arithmetic; nobody writes raw ring code twice.

### The feature ladder (each rung deletes a cost)

- **Batching** — N ops, 1 syscall (the default win).
- **Chaining** (`IOSQE_IO_LINK`) — ops that run in sequence *in the kernel*:
  write→fsync, accept→read; a failure cancels the chain's remainder.
- **Fixed files/buffers** — pre-register fds and buffers: skips per-op
  refcounting/pinning (the hidden per-syscall work).
- **Multishot** — one SQE, many CQEs: multishot accept ("keep accepting,
  keep completing") replaces the accept loop entirely.
- **SQPOLL** — a kernel thread busy-polls your SQ: submission needs *zero*
  syscalls (burning a core for latency — networking Lesson 64's
  kernel-bypass economics, without leaving the kernel's safety).

Who's on it: high-performance storage engines (RocksDB, ScyllaDB's
ecosystem), QEMU (`aio=io_uring` — virt Lesson 25's third option, decoded),
Rust runtimes (tokio-uring, glommio), nginx/Envoy experiments, and
`fio --ioengine=io_uring` for benchmarking it.

### The honest caveats

Security history is real: the fast-moving surface produced serious CVEs
(2021–2023 especially); hardened environments restrict it (Docker's default
seccomp profile blocks it wholesale, Android/ChromeOS gate it —
`io_uring_disabled` sysctl exists). Reasoning: the async-op engine runs with
kernel privileges on complex, attacker-suppliable state — a juicy target
(Lesson 56 will generalize the "syscall surface" idea). It's also *not*
automatically faster for light workloads: an epoll loop at 10K mostly-idle
connections gains little; io_uring shines when op *rate* is the bottleneck —
storage-bound and syscall-bound systems.

{: .note }
> **Three generations, one sentence each**
> <code>select/poll</code>: ask about everything, every time.
> <code>epoll</code>: the kernel remembers what you care about and tells
> you when to act. <code>io_uring</code>: stop asking — hand over the work
> itself, in bulk, through shared memory.

---

## Lab

```bash
# ---- 0. availability check ----
$ grep -r io_uring_setup /proc/kallsyms >/dev/null 2>&1 && echo "kernel: yes"
$ cat /proc/sys/kernel/io_uring_disabled 2>/dev/null
# 0 = enabled (1/2 = restricted/disabled — the hardening knob, live)
$ sudo apt install -y liburing-dev 2>/dev/null | tail -1

# ---- 1. Async FILE read — the thing epoll refused (L43 lab 6) ----
$ cat > /tmp/uring1.c << 'EOF'
#include <liburing.h>
#include <stdio.h>
#include <fcntl.h>
#include <string.h>
int main(void) {
    struct io_uring ring;
    io_uring_queue_init(8, &ring, 0);

    int fd = open("/etc/hostname", O_RDONLY);
    char buf[256];
    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fd, buf, sizeof buf, 0);
    io_uring_sqe_set_data64(sqe, 42);              /* our cookie */

    io_uring_submit(&ring);                         /* ONE syscall */
    struct io_uring_cqe *cqe;
    io_uring_wait_cqe(&ring, &cqe);
    printf("op %llu completed: %d bytes: %.*s",
           (unsigned long long)cqe->user_data, cqe->res, cqe->res, buf);
    io_uring_cqe_seen(&ring, cqe);
    io_uring_queue_exit(&ring);
    return 0;
}
EOF
$ gcc -o /tmp/uring1 /tmp/uring1.c -luring && /tmp/uring1
# op 42 completed: N bytes: yourhostname
# a REGULAR FILE read, asynchronously, matched by cookie. epoll never could.

# ---- 2. Batching: 64 reads, ONE syscall ----
$ cat > /tmp/uring2.c << 'EOF'
#include <liburing.h>
#include <stdio.h>
#include <fcntl.h>
#define N 64
int main(void) {
    struct io_uring ring;
    io_uring_queue_init(N, &ring, 0);
    int fd = open("/tmp/batch.dat", O_RDONLY);
    static char bufs[N][4096];
    for (int i = 0; i < N; i++) {
        struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
        io_uring_prep_read(sqe, fd, bufs[i], 4096, i * 4096);
        io_uring_sqe_set_data64(sqe, i);
    }
    int submitted = io_uring_submit(&ring);         /* 64 ops, 1 enter()  */
    printf("submitted %d ops in one syscall\n", submitted);
    long total = 0;
    for (int i = 0; i < N; i++) {
        struct io_uring_cqe *cqe;
        io_uring_wait_cqe(&ring, &cqe);
        total += cqe->res;
        io_uring_cqe_seen(&ring, cqe);
    }
    printf("collected %d completions, %ld bytes\n", N, total);
    return 0;
}
EOF
$ dd if=/dev/urandom of=/tmp/batch.dat bs=4096 count=64 2>/dev/null
$ gcc -o /tmp/uring2 /tmp/uring2.c -luring
$ strace -c -e trace=io_uring_enter,read /tmp/uring2 2>&1 | tail -6
# io_uring_enter: 1-2 calls. read: 0. Sixty-four I/Os, one crossing —
# Lesson 02's dd experiment, won at a new level.

# ---- 3. Chaining: write THEN fsync, ordered in-kernel (L41's recipe!) ----
$ cat > /tmp/uring3.c << 'EOF'
#include <liburing.h>
#include <stdio.h>
#include <fcntl.h>
#include <string.h>
int main(void) {
    struct io_uring ring;
    io_uring_queue_init(4, &ring, 0);
    int fd = open("/tmp/wal.log", O_WRONLY | O_CREAT | O_APPEND, 0644);
    const char *rec = "txn-commit-record\n";

    struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
    io_uring_prep_write(sqe, fd, rec, strlen(rec), -1);
    sqe->flags |= IOSQE_IO_LINK;                    /* chain to next */
    io_uring_sqe_set_data64(sqe, 1);

    sqe = io_uring_get_sqe(&ring);
    io_uring_prep_fsync(sqe, fd, IORING_FSYNC_DATASYNC);
    io_uring_sqe_set_data64(sqe, 2);

    io_uring_submit(&ring);                         /* write+fsync: 1 syscall */
    for (int i = 0; i < 2; i++) {
        struct io_uring_cqe *cqe;
        io_uring_wait_cqe(&ring, &cqe);
        printf("op %llu: res=%d\n", (unsigned long long)cqe->user_data, cqe->res);
        io_uring_cqe_seen(&ring, cqe);
    }
    return 0;
}
EOF
$ gcc -o /tmp/uring3 /tmp/uring3.c -luring && /tmp/uring3
# op 1: res=18 / op 2: res=0 — a WAL append made durable, ordered by the
# kernel, one crossing. Group commit's building block (L41 Q2), as a syscall.

# ---- 4. The benchmark: fio's three engines, same disk ----
$ command -v fio >/dev/null && for eng in sync io_uring; do
    echo "=== $eng ==="
    fio --name=t --filename=/tmp/fio.dat --size=128M --bs=4k --rw=randread \
        --direct=1 --iodepth=32 --ioengine=$eng --runtime=8 --time_based 2>/dev/null \
        | grep -E 'read: IOPS'
  done
# io_uring at iodepth=32 typically multiplies sync's IOPS — 32 in-flight ops
# per crossing vs 1. (psync/libaio comparisons are instructive too.)
$ rm -f /tmp/uring[123]* /tmp/batch.dat /tmp/wal.log /tmp/fio.dat
```

---

## Further Reading

| Topic | Link |
|---|---|
| io_uring (Wikipedia) | <https://en.wikipedia.org/wiki/Io_uring> |
| `io_uring(7)` man page | <https://man7.org/linux/man-pages/man7/io_uring.7.html> |
| liburing repository | <https://github.com/axboe/liburing> |
| Efficient I/O with io_uring — Axboe's design doc | <https://kernel.dk/io_uring.pdf> |
| `io_uring_enter(2)` man page | <https://man7.org/linux/man-pages/man2/io_uring_enter.2.html> |
| Lord of the io_uring — guide | <https://unixism.net/loti/> |

---

## Checkpoint

**Q1.** epoll tells you "you may now read without blocking"; io_uring tells
you something categorically different. What — and why does that difference
finally make *disk files* first-class async citizens?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
io_uring says "the read you asked for <em>has happened</em> — here are your
bytes (or your error)." Readiness vs completion. Files broke the readiness
model because readiness is about <em>data availability at the fd</em> — and
a regular file's data is definitionally "available" (it's on disk, no peer
needs to send it); the waiting hides inside the read itself (page-cache miss
→ block layer → device — Lessons 19/20/42), which readiness can't express:
epoll would have to say "ready" always, making it useless (hence its EPERM).
The completion model doesn't care <em>where</em> latency lives: you hand
over the operation; the kernel navigates cache hits (immediate completion),
misses (async through the block layer), whatever — and completes when done.
Any operation with a result fits, which is why the opcode list grew past
storage into accept/connect/send/recv and even openat: completion is the
universal shape; readiness was a special case that happened to fit sockets.
</details>

**Q2.** Explain how the two rings eliminate syscalls, and connect the design
to two specific things you built/learned in Phases 5–6.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The rings are single-producer/single-consumer queues in memory mmap'd into
both the process and the kernel: submitting = write an SQE at the tail
index, publish the new tail with a release store; the kernel (in
io_uring_enter, or continuously under SQPOLL) acquires-loads the tail and
consumes; completions flow back identically through the CQ with roles
reversed. Syscalls stop being per-operation because the queue <em>is</em>
the communication: one enter() hands over everything queued (or zero
enters with SQPOLL — the kernel thread polls the shared tail). The two
connections: (1) Lesson 33's homework — the SPSC ring buffer in shared
memory with head/tail single-writer discipline — this is that exact design,
with the kernel as your peer process (the homework even foreshadowed
"io_uring is two SPSC rings"); (2) Lesson 29's release/acquire publication —
the ring indexes are the atomic flags whose ordering guarantees the consumer
never reads an SQE before its fields are visible: memory-ordering rules
doing syscall work. (And the economics is Lesson 02/41's batching law:
amortize fixed per-crossing costs over N items — the same curve from dd
bs=1 to group commit.)
</details>

**Q3.** Docker's default seccomp profile blocks io_uring entirely. Steelman
that decision, then describe what a workload loses and the middle paths
available.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Steelman: seccomp's whole model (Lesson 56 ahead) is shrinking the kernel
attack surface reachable from a container — and io_uring is a
<em>surface concentrator</em>: one interface reaching 40+ operations,
executed by kernel-side async machinery whose complexity (op parsing,
buffer/file registration, worker threads acting with caller credentials)
yielded a genuine CVE streak, including sandbox escapes; worse, operations
performed by uring workers historically evaded seccomp's per-syscall
filtering itself (the filter sees io_uring_enter, not the read/openat
inside it) — so allowing it partially defeats the rest of the profile.
Blocking it trades performance for a verified-smaller surface — the right
default for untrusted multi-tenant workloads. What's lost: storage-heavy
apps fall back to thread pools + readiness (more syscalls, more threads —
the Lesson 43-era ceilings), typically a real but survivable regression for
ordinary services. Middle paths: per-workload seccomp profiles allowing the
three io_uring syscalls for trusted/storage-critical containers (with
io_uring's own restriction API and IORING_SETUP flags narrowing opcodes);
kernel-level gating via <code>io_uring_disabled=2</code>-except-privileged;
or isolating the storage-hot component into a VM-isolated runtime (Kata —
virt Lesson 60) where the guest kernel's io_uring endangers only itself.
The general pattern: performance features that concentrate privilege get
adopted first where trust is highest — same story as SQPOLL, hugepages,
and kernel-bypass networking.
</details>

---

## Homework

Sketch (pseudocode) the io_uring version of Lesson 43's echo server using
**multishot accept** and per-connection read/write SQEs. Then answer: what
replaces the epoll event loop's `for fd, ev in events` dispatch, what
replaces EPOLLOUT re-arming for short writes, and which single property of
`user_data` makes the whole state machine simpler than the epoll version?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<pre>
setup ring; sqe = multishot_accept(listener, user_data=ACCEPT)
submit; loop:
    wait_cqe(&cqe); switch(cqe->user_data.type):
      ACCEPT:   # fires per-connection, stays armed (multishot!)
          conn = new_conn(cqe->res)                  # res = client fd
          sqe = recv(conn.fd, conn.buf, user_data={READ, conn})
      READ:
          if cqe->res <= 0: close/cleanup; break
          sqe = send(conn.fd, conn.buf, cqe->res, user_data={WRITE, conn})
          # note: no re-arm of read until write completes → natural
          # per-connection pipelining; or link recv after send with IO_LINK
      WRITE:
          if cqe->res < conn.pending:               # SHORT write:
              sqe = send(conn.fd, buf+res, pending-res, {WRITE, conn})
          else:
              sqe = recv(conn.fd, conn.buf, {READ, conn})   # next request
    submit_batch()
</pre>
The dispatch: epoll's "which fd, which direction, now figure out what I was
doing" becomes a completion stream — each CQE arrives already knowing
<em>what operation finished and its result</em>; there is no readable/
writable interpretation step at all. EPOLLOUT re-arming dissolves: a short
write isn't a state to monitor but simply <em>the next SQE to submit</em>
(send the remainder; its completion tells you when) — the arm/disarm
bookkeeping and the busy-loop-on-empty-buffer bug class (Lesson 43's
homework) can't exist. The property doing the heavy lifting:
<code>user_data</code> is an opaque 64-bit cookie <em>you</em> choose —
typically a pointer to the connection's state struct plus an op tag — so
every completion carries its full context with it: the state machine's
"where was I?" is answered by the kernel handing your own pointer back,
instead of a hash lookup keyed by fd (which also kills the fd-reuse race:
a recycled fd number can't misroute a completion that carries the old
connection's pointer — you free it only after its last in-flight op
completes).
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 45 — Zero-Copy: sendfile and splice →](lesson-45-zero-copy){: .btn .btn-primary }
