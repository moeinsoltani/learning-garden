---
title: "Lesson 43 — epoll and Non-Blocking I/O"
nav_order: 1
parent: "Phase 8: Event-Driven & Async I/O"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 43: epoll and Non-Blocking I/O

## Concept

Lesson 24 argued you can't afford a thread per idle connection; Lesson 30
promised the event loop; Lesson 35 fed it exotic fds. Today you build the
engine itself.

Two ingredients. First, **non-blocking mode** (`O_NONBLOCK`): a read on an
empty socket returns `-1 EAGAIN` instead of sleeping — "nothing now, come
back later." Alone, that's just polite failure: spinning on EAGAIN burns a
CPU. The second ingredient makes it an architecture — **readiness
notification**: tell the kernel *all* the fds you care about, then sleep in
one call until *any* becomes ready.

```
  the evolution of "wait for any of N fds":

  select(2)   pass a bitmap of fds EVERY call; kernel scans ALL of them;
   (1983)     1024-fd limit.                    O(N) per call
  poll(2)     array instead of bitmap; no limit; still scanned fully.
   (1986)                                       O(N) per call
  epoll(7)    REGISTER interest once (epoll_ctl); kernel maintains a
   (2002)     ready-list as events arrive (from interrupts! L05);
              epoll_wait returns just the ready ones.
                                                O(ready) per call ★
```

That complexity difference — O(N) vs O(ready) — is the entire C10K story:
with 10,000 connections of which 50 are active, select/poll inspect 10,000
fds per iteration; epoll returns 50. nginx, Redis, Node.js, every asyncio
loop: `epoll_wait` is the one place they sleep (kqueue on BSD/macOS, IOCP on
Windows — same idea, different lineage).

The loop, in its eternal shape:

```
  while true:
      events = epoll_wait(epfd)              # THE only blocking call
      for fd, what in events:
          if fd == listener:  accept new client (nonblocking!), register it
          elif what & IN:     read until EAGAIN, advance state machine
          elif what & OUT:    flush pending writes until EAGAIN
```

---

## How It Works

### Level-triggered vs edge-triggered

- **Level-triggered (LT, default)**: "fd is readable" reported *as long as*
  data remains. Forgiving — read some now, epoll reminds you next round.
- **Edge-triggered (ET, `EPOLLET`)**: reported *once*, when the state
  changes (new data arrives). You must **read until EAGAIN** or the
  remaining data is silently never announced again — the classic ET hang.
  Why choose it? Fewer wakeups under load, and it pairs with
  one-thread-per-core designs (with `EPOLLEXCLUSIVE`/`EPOLLONESHOT` taming
  thundering herds — Lesson 27's herd, epoll edition).

The safe recipe (what production code does): ET + loop-until-EAGAIN +
nonblocking fds, or LT + careful partial handling. Mixing ET with a single
read() is the bug that works in tests (small messages) and wedges in
production (big bursts).

### Readiness ≠ success

epoll says "readable *now*" — a hint, not a contract. Between notification
and your read, another thread may have consumed the data
(accept-distribution races), or a checksum-failed packet got discarded:
robust loops treat EAGAIN after "ready" as normal weather. And **readiness
is the wrong model for regular files**: a disk file is *always* "ready"
(the data's just... slow — Lesson 19's major faults happen *inside* your
"non-blocking" read!). epoll can't help with file I/O — the gap io_uring
(next lesson) exists to close; event-loop runtimes meanwhile ship thread
pools for file work (Lesson 30's composition).

### What accept and write add

The listener socket is just another fd: readable = connection(s) waiting —
accept in a loop until EAGAIN (ET especially: multiple connections, one
edge!). Writes: a full socket buffer (slow receiver — TCP backpressure,
networking track!) makes write return a *short count* or EAGAIN; you buffer
the残り, register EPOLLOUT, flush when writable, deregister. Every real
event loop is 60% this bookkeeping — which is why frameworks exist.

{: .note }
> **timeout and the tick**
> <code>epoll_wait(…, timeout)</code> doubles as the loop's timer floor —
> but real loops pass the delay to the <em>next scheduled timer</em>
> (timerfd, Lesson 35, or a heap of deadlines) and 0/-1 for poll/forever.
> CPU profile of a healthy event loop: ~100% of wall time asleep in
> epoll_wait, microbursts of work — compare Lesson 12's voluntary-switch
> fingerprint.

---

## Lab

```bash
# ---- 1. EAGAIN: meet polite refusal ----
$ python3 - << 'EOF'
import os, socket
a, b = socket.socketpair()
b.setblocking(False)
try:
    b.recv(100)
except BlockingIOError as e:
    print("empty nonblocking read:", e)          # EAGAIN — no sleep, no data
a.send(b"now there is data")
print("after send:", b.recv(100))
EOF

# ---- 2. Build the echo server — a real epoll loop in ~45 lines ----
$ cat > /tmp/epoll_echo.py << 'EOF'
import socket, select, sys

lst = socket.socket()
lst.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
lst.bind(("127.0.0.1", 9009)); lst.listen(128); lst.setblocking(False)

ep = select.epoll()
ep.register(lst.fileno(), select.EPOLLIN)
conns, nclients = {}, 0

while True:
    for fd, ev in ep.poll():                     # THE blocking point
        if fd == lst.fileno():
            while True:                          # accept until EAGAIN
                try:
                    c, addr = lst.accept()
                except BlockingIOError:
                    break
                c.setblocking(False)
                conns[c.fileno()] = c
                ep.register(c.fileno(), select.EPOLLIN)
                nclients += 1
        elif ev & select.EPOLLIN:
            c = conns[fd]
            try:
                data = c.recv(4096)
            except BlockingIOError:
                continue                         # ready-but-EAGAIN: shrug
            if not data:                         # EOF (L02: read()==0)
                ep.unregister(fd); c.close(); del conns[fd]
                continue
            c.send(data)                         # (real code: handle short
                                                 #  writes + EPOLLOUT!)
            if data.strip() == b"stats":
                c.send(f" clients ever: {nclients}\n".encode())
EOF
$ python3 /tmp/epoll_echo.py &
$ sleep 0.5

# ---- 3. One process, many concurrent clients ----
$ for i in 1 2 3 4 5; do (echo "hello from $i"; sleep 2) | timeout 3 nc 127.0.0.1 9009 & done
$ sleep 1; ls /proc/$(pgrep -f epoll_echo)/task | wc -l
# 1        ← five simultaneous conversations, ONE thread
$ echo stats | timeout 1 nc 127.0.0.1 9009
$ wait 2>/dev/null

# ---- 4. See the loop's syscall rhythm ----
$ strace -p $(pgrep -f epoll_echo) -e trace=epoll_wait,accept4,recvfrom,sendto -f 2>&1 | head -12 &
$ sleep 0.3; echo "trace me" | timeout 1 nc 127.0.0.1 9009; sleep 0.5; kill %2 2>/dev/null
# epoll_wait(...) = 1 [{EPOLLIN, fd=lst}]     ← wake: connection
# accept4(...) / accept4(...) = EAGAIN        ← drain the accept queue
# epoll_wait(...) = 1 [{EPOLLIN, fd=7}]       ← wake: data
# recvfrom(7, "trace me\n"...) / sendto(7,...) ← handle, echo
# epoll_wait(                                  ← back to sleep. The heartbeat.

# ---- 5. The wchan fingerprint of a healthy loop (L26's trick) ----
$ cat /proc/$(pgrep -f epoll_echo)/wchan; echo
# ep_poll        ← asleep in epoll_wait: the 100%-idle-100%-responsive state
$ kill %1

# ---- 6. Readiness is a LIE for regular files ----
$ python3 - << 'EOF'
import select, os
fd = os.open("/etc/hostname", os.O_RDONLY | os.O_NONBLOCK)
ep = select.epoll()
try:
    ep.register(fd, select.EPOLLIN)
    print("registered a regular file:", ep.poll(0.1))
except PermissionError as e:
    print("epoll refuses regular files outright:", e)
# Linux: EPERM — epoll won't even pretend. Files are "always ready";
# their latency hides in the read itself. Hence: thread pools… or io_uring.
EOF
$ rm /tmp/epoll_echo.py
```

---

## Further Reading

| Topic | Link |
|---|---|
| `epoll(7)` overview | <https://man7.org/linux/man-pages/man7/epoll.7.html> |
| `epoll_ctl(2)` / `epoll_wait(2)` | <https://man7.org/linux/man-pages/man2/epoll_ctl.2.html> |
| C10k problem | <https://en.wikipedia.org/wiki/C10k_problem> |
| Edge vs level triggered — epoll(7) Q&A | <https://man7.org/linux/man-pages/man7/epoll.7.html#NOTES> |
| `select(2)` — the ancestor | <https://man7.org/linux/man-pages/man2/select.2.html> |
| Reactor pattern | <https://en.wikipedia.org/wiki/Reactor_pattern> |

---

## Checkpoint

**Q1.** epoll says a socket is readable; your read gets EAGAIN anyway. How is
that possible, and why must robust code treat it as normal?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Readiness is a snapshot, not a reservation. Between epoll_wait returning and
your read executing, the data can vanish: another thread/process sharing the
epoll set or the socket consumed it first (multi-worker accept and
SO_REUSEPORT setups do this constantly); the kernel discarded a packet after
notification (checksum failure post-wakeup is the documented select/epoll
false-ready case); or with ET + multiple events coalesced, an earlier
handler in the same loop iteration drained the fd. Since none of these are
bugs, EAGAIN-after-ready must be handled as a no-op (continue the loop) —
and this is precisely why nonblocking mode is mandatory in epoll designs:
a blocking read on a false-ready fd would hang the entire loop, converting
a benign race into a full-service outage. Treat epoll as a hint
accelerator: correctness must never depend on its word.
</details>

**Q2.** Explain the edge-triggered contract and construct the classic ET bug:
what exact sequence leaves data forever unread, and what discipline prevents
it?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
ET reports state <em>transitions</em>: an event fires when the fd goes from
not-ready to ready (new bytes arrive into an empty buffer). The bug: client
sends 10 KB; epoll fires once; your handler reads 4 KB (one read call) and
returns to epoll_wait. The remaining 6 KB sit in the buffer — but the fd
never <em>transitions</em> again (it was ready and stayed ready), so no
event ever fires… unless the client happens to send more. With a
request/response protocol, the client is waiting for your reply to data
you'll never finish reading: mutual eternal wait — a distributed deadlock
(Lesson 28's shape, no locks involved). Discipline: after any ET
notification, consume until the syscall says EAGAIN (loop reads/accepts/
writes), guaranteeing you leave the fd genuinely empty so the next arrival
is a real transition. Corollaries: fds must be nonblocking (the drain loop's
last call would otherwise block), and per-fd fairness needs care (one
fire-hose fd can monopolize a drain loop — cap per-iteration work and
re-arm with EPOLLONESHOT if needed).
</details>

**Q3.** Redis famously served hundreds of thousands of ops/s on ONE thread
for a decade. Using Lessons 12/24/26/30 + this one, explain why the
single-threaded epoll design was *fast* — and what class of Redis operation
eventually forced threads in anyway.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Single-threaded epoll deletes entire cost categories: no locks anywhere
(Lesson 26's fast path is still ~10 ns × millions — and contention would be
worse), no cache-line ping-pong between cores (Lesson 29's coherence war),
no context switches per request (Lesson 12 — the loop handles hundreds of
commands per wakeup), and no races by construction (Lesson 30's async
atomicity between awaits — Redis commands are those atomic sections, which
is also its <em>consistency model</em>: free serializability). Each command
being O(µs) of pure memory work, one core's throughput ceiling is enormous;
the bottleneck was typically the network stack, not Redis. What broke the
model: operations whose cost isn't tiny — large-value serialization, big
DEL/FLUSH frees, and above all <em>syscall-heavy I/O</em> (read/write per
client at high connection counts eats the single core). Redis's concessions
track this lesson exactly: background threads for slow frees (lazyfree, so
the loop never blocks — rule #1 of Lesson 30), and I/O threads that only do
socket read/parse/write while the command core stays single-threaded —
keeping the no-locks datapath, multiplying the syscall muscle. The design
lesson: single-threaded isn't a limitation that was tolerated; it was the
optimization — threads were added only where the loop's one commandment
("never block, never compute long") couldn't hold.
</details>

---

## Homework

Your epoll echo server (lab 2) has a known hole: `c.send(data)` ignores short
writes. Construct the scenario where it corrupts a client's stream (hint: a
slow reader and a large echo — think TCP send buffers and backpressure),
then write the fix in pseudocode: per-connection write buffer, EPOLLOUT
registration/deregistration, and the order of operations when new data
arrives while a backlog exists.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Scenario: client sends 1 MB and reads its echo slowly (or not at all). The
server's <code>send()</code> copies what fits into the socket's send buffer
— say 64 KB — and returns 64 KB (or raises EAGAIN when full). The lab code
ignores the return value: the remaining 960 KB are silently dropped; worse,
when the client's next message arrives, its echo is sent as if nothing were
missing — the client's stream now has a hole mid-payload: corruption
invisible to the server. (TCP's backpressure — receiver window → sender
buffer fills — is working perfectly; the application layer dropped the
ball.) Fix:
<pre>
on EPOLLIN data:
    if conn.outbuf:  conn.outbuf += echo(data); return   # ORDER: never
    n = send(fd, echo(data))                              # bypass the queue!
    if n < len:      conn.outbuf = echo(data)[n:]
                     epoll_ctl(MOD, fd, IN|OUT)           # arm writable
on EPOLLOUT:
    n = send(fd, conn.outbuf)      # until EAGAIN
    conn.outbuf = conn.outbuf[n:]
    if not conn.outbuf: epoll_ctl(MOD, fd, IN)            # disarm! (else
                                                          # OUT fires forever
                                                          # on an empty queue
                                                          # = busy loop)
also: cap outbuf size — beyond the cap, stop reading from this client
(deregister IN) = propagate backpressure (L30!), or drop the connection.
</pre>
The three invariants: new data goes <em>behind</em> the backlog (ordering),
EPOLLOUT is registered only while a backlog exists (else the loop spins),
and the buffer is bounded (else one slow client becomes your OOM — Lesson
22 closes the circle).
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 44 — io_uring →](lesson-44-io-uring){: .btn .btn-primary }
