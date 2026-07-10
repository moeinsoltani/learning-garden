---
title: "Lesson 35 — eventfd, signalfd, timerfd"
nav_order: 5
parent: "Phase 6: Inter-Process Communication"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 35: eventfd, signalfd, timerfd

## Concept

Across this phase a pattern kept surfacing: pipes are fds, sockets are fds,
mqueues are (secretly) fds, shm arrives via an fd. This closing lesson makes
the pattern explicit — because it's one of Linux's central design ideas:

**If everything a program waits for is a file descriptor, then one wait can
cover everything.**

The event loop (Lesson 30, built in Lesson 43) blocks in a single
`epoll_wait` on a set of fds. But real programs wait on more than I/O: on
*signals* (Lesson 09 — asynchronous, handler-hostile), on *timers* ("do X in
30 s"), on *other threads* ("wake up, work arrived"). Historically each had
its own alien mechanism that fought the loop. Linux's answer: wrap each one
as an fd.

```
              the unified wait:  epoll_wait(...) on

  socket fd     "client sent data"        (classic I/O)
  timerfd       "your 30s timer expired"  (timers → readable fd)
  signalfd      "SIGTERM arrived"         (signals → readable fd)
  eventfd       "worker thread finished"  (in-process doorbell)
  pidfd         "child process exited"    (L06's process handle)
  mqueue fd     "message queued"          (L34)
  inotify fd    "config file changed"     ...one loop rules them all
```

- **eventfd** — the minimal doorbell: an 8-byte kernel counter; writes add,
  reads drain (and block/EAGAIN at zero). One fd where a pipe-as-wakeup used
  two, with counting semantics (N rings = N recorded). It's how the main
  loop and worker pools poke each other in nginx, glib, and — wearing its
  vhost hat — how QEMU's guests kick their I/O threads (virt Lesson 29's
  ioeventfd/irqfd are *exactly this object*).
- **signalfd** — signals as a readable queue: block the signals (Lesson 24
  Q3's mask trick), open a signalfd for them, and SIGTERM becomes a struct
  you `read()` at a moment of your choosing — handler async-safety rules
  (Lesson 09) simply evaporate.
- **timerfd** — timers as fds: arm once or periodic; expiration makes the fd
  readable (the read returns the number of expirations you missed — no
  drift-hiding). The event loop's entire timer wheel can be one fd.

---

## How It Works

### eventfd semantics, precisely

`eventfd(initval, flags)`: `write(fd, &n)` adds n to the counter (blocks if
it would overflow); `read(fd)` returns the counter and zeroes it —
**coalescing**: ring it 5 times between reads, one read says "5". With
`EFD_SEMAPHORE`, reads decrement by 1 instead — each read consumes one ring
(semaphore semantics — Lesson 27, as an fd!). Poll-readable when counter > 0.
The classic upgrade path: the old *self-pipe trick* (signal handler writes a
byte to a pipe the loop watches) becomes handler-writes-eventfd, or better,
no handler at all (signalfd).

### signalfd's contract

Signals must be **blocked** (sigprocmask) or they'll be delivered the old way
instead of queueing for the fd — the #1 usage bug. Reads return
`signalfd_siginfo` structs (signo, sender PID/UID — Lesson 09's missing
metadata!). Caveat worth knowing: signal *dispositions* still apply for
unblockable SIGKILL/SIGSTOP, and blocked-but-fatal signals like SIGSEGV can't
be fd-ified meaningfully. For child reaping, pidfds (`pidfd_open`, poll for
exit, `waitid(P_PIDFD)`) are the even cleaner modern road (Lesson 06's
recycling fix, completing its arc).

### timerfd's three clocks

`CLOCK_MONOTONIC` (intervals immune to wall-clock changes — the default
choice), `CLOCK_REALTIME` (calendar deadlines — fires early/late if the
clock is stepped: NTP, Lesson 72 of networking!), `CLOCK_BOOTTIME` (counts
suspend time too). `TFD_TIMER_ABSTIME` for absolute deadlines. The read's
"missed expirations" count is your built-in overrun detector — a periodic
task that reads 3 knows it fell behind, no extra bookkeeping.

{: .note }
> **Why this beats callbacks-from-anywhere**
> Every mechanism here converts an <em>interruption</em> into <em>data you
> read when ready</em>. That single transformation eliminates reentrancy
> (Lesson 09), lost-wakeup races (the fd holds state — a ring before you
> sleep is still there when you check), and priority chaos (you process
> events in the order your loop chooses). It's Lesson 27's "turn signals into
> queue items" philosophy, blessed by the kernel.

---

## Lab

```bash
# ---- 1. eventfd: the counting doorbell ----
$ python3 - << 'EOF'
import os, struct
efd = os.eventfd(0)                       # Python 3.10+
os.eventfd_write(efd, 1)                  # ring
os.eventfd_write(efd, 1)                  # ring
os.eventfd_write(efd, 5)                  # ring x5
print("one read collects all:", os.eventfd_read(efd))   # 7 — coalesced!
# now EFD_SEMAPHORE: one ring per read
sfd = os.eventfd(0, os.EFD_SEMAPHORE | os.EFD_NONBLOCK)
os.eventfd_write(sfd, 3)
print("semaphore reads:", os.eventfd_read(sfd), os.eventfd_read(sfd),
      os.eventfd_read(sfd))               # 1 1 1
try: os.eventfd_read(sfd)
except BlockingIOError: print("4th read: EAGAIN — counter empty")
EOF

# ---- 2. signalfd: SIGTERM as data (no handler anywhere) ----
$ python3 - << 'EOF'
import signal, os, ctypes, struct
libc = ctypes.CDLL("libc.so.6", use_errno=True)
# 1: BLOCK the signal (mandatory — else classic delivery wins)
signal.pthread_sigmask(signal.SIG_BLOCK, {signal.SIGTERM, signal.SIGINT})
# 2: sigset for signalfd via libc
class SigSet(ctypes.Structure): _fields_ = [("v", ctypes.c_ulong * 16)]
ss = SigSet(); libc.sigemptyset(ctypes.byref(ss))
libc.sigaddset(ctypes.byref(ss), signal.SIGTERM)
libc.sigaddset(ctypes.byref(ss), signal.SIGINT)
sfd = libc.signalfd(-1, ctypes.byref(ss), 0)
print(f"pid {os.getpid()}: signals are now a readable fd. self-sending TERM…")
os.kill(os.getpid(), signal.SIGTERM)
data = os.read(sfd, 128)                       # signalfd_siginfo struct
signo, _, _, pid_field = struct.unpack_from("IiiI", data)
print(f"read from fd: signal {signo} (SIGTERM={signal.SIGTERM}) "
      f"sent by pid {pid_field} — metadata a handler never showed you")
EOF

# ---- 3. timerfd: a timer you can epoll — with overrun detection ----
$ python3 - << 'EOF'
import os, struct, time, select
tfd = os.timerfd_create(os.CLOCK_MONOTONIC)          # Python 3.13+; else ctypes
os.timerfd_settime(tfd, initial=0.3, interval=0.3)   # periodic 300ms
p = select.poll(); p.register(tfd, select.POLLIN)
for i in range(3):
    p.poll()                                          # wait — as an fd event!
    n = struct.unpack("Q", os.read(tfd, 8))[0]
    print(f"tick {i}: {n} expiration(s)")
print("now sleep past 3 periods without reading…")
time.sleep(1.0)
n = struct.unpack("Q", os.read(tfd, 8))[0]
print(f"read says {n} — the fd COUNTED what you missed. no silent drift.")
EOF
# (older python: the same via ctypes CDLL timerfd_create/settime — 10 lines)

# ---- 4. The payoff: ONE loop waiting on all three kinds at once ----
$ python3 - << 'EOF'
import os, select, signal, struct, threading, time
efd = os.eventfd(0)
tfd = os.timerfd_create(os.CLOCK_MONOTONIC)
os.timerfd_settime(tfd, initial=0.5, interval=0)
r, w = os.pipe()                                   # stand-in for "a socket"
def worker():
    time.sleep(0.2); os.eventfd_write(efd, 1)      # thread rings doorbell
    time.sleep(0.5); os.write(w, b"payload")       # "network data"
threading.Thread(target=worker).start()
ep = select.epoll()
for fd in (efd, tfd, r): ep.register(fd, select.EPOLLIN)
names = {efd: "eventfd (worker done)", tfd: "timerfd (timeout)", r: "pipe (I/O)"}
got = 0
while got < 3:
    for fd, _ in ep.poll():
        print("event:", names[fd])
        os.read(fd, 64); got += 1
EOF
# event: eventfd (worker done)
# event: timerfd (timeout)
# event: pipe (I/O)
# Three different worlds — threads, time, I/O — one epoll_wait. This IS the
# architecture of nginx/systemd/every event loop, and Lesson 43 builds on it.

# ---- 5. Spot them in the wild ----
$ sudo ls -l /proc/1/fd | grep -Eo 'anon_inode:\[?[a-z_]+' | sort | uniq -c | sort -rn | head
# anon_inode:[timerfd] ×N, [eventfd] ×N, [signalfd], inotify…
# PID 1's entire nervous system, visible as fd types. (Try a browser's PID!)
```

---

## Further Reading

| Topic | Link |
|---|---|
| `eventfd(2)` man page | <https://man7.org/linux/man-pages/man2/eventfd.2.html> |
| `signalfd(2)` man page | <https://man7.org/linux/man-pages/man2/signalfd.2.html> |
| `timerfd_create(2)` man page | <https://man7.org/linux/man-pages/man2/timerfd_create.2.html> |
| `pidfd_open(2)` man page | <https://man7.org/linux/man-pages/man2/pidfd_open.2.html> |
| `epoll(7)` overview | <https://man7.org/linux/man-pages/man7/epoll.7.html> |
| `inotify(7)` — files as events too | <https://man7.org/linux/man-pages/man7/inotify.7.html> |

---

## Checkpoint

**Q1.** Why would a program prefer signalfd over a signal handler for
SIGCHLD? Give the three concrete problems of the handler approach that the
fd approach dissolves.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <strong>Async-safety jail</strong> (Lesson 09): a handler interrupts any
instruction — it may only call async-signal-safe functions, so real reaping
logic (logging, data structures, restarting the child — Lesson 08's
supervisor) can't live there; with signalfd the "signal" is processed as
ordinary code in your loop, full language available. (2)
<strong>Coalescing losses</strong>: standard signals don't queue — three
children dying fast may deliver ONE SIGCHLD; handlers must loop
waitpid(WNOHANG) defensively. signalfd doesn't fix coalescing (same signal
semantics) but makes the drain-loop natural and race-free at a chosen point.
(3) <strong>Event-loop integration</strong>: a handler fires on whatever
thread at whatever moment — synchronizing with loop state means self-pipe
gymnastics anyway; signalfd IS the self-pipe, kernel-made, with siginfo
metadata (sender pid/uid) the handler path loses. (Modern endgame for
children specifically: pidfd per child — poll for exit, no signals at all.)
</details>

**Q2.** eventfd reads *coalesce* (5 rings → one read of 5). When is that
exactly right, when is it wrong, and what flag changes it — connect each case
to a Phase 5/6 concept.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Right when the doorbell means "state changed, go look" — level-triggered
condition-checking (Lesson 27's while-loop philosophy): the loop wakes once,
drains the <em>actual queue</em> it guards, and 5 rings vs 1 is irrelevant —
coalescing is pure efficiency (fewer wakeups, Lesson 12's switch economy).
This is the producer/consumer doorbell and QEMU's "guest kicked the queue"
ioeventfd. Wrong when each ring IS one token of work/permission — counting
semantics where consuming 5 as 1 loses 4: worker-slot semaphores, one-read-
per-job dispatch. <code>EFD_SEMAPHORE</code> switches reads to decrement-by-1
— literally Lesson 27's semaphore as an fd (sem_wait ≈ read, sem_post ≈
write). The design question is always: does the fd carry <em>information</em>
("something happened") or <em>quantity</em> ("N things to consume")? —
mislabeling that is how wakeups get lost or duplicated.
</details>

**Q3.** A daemon needs to: serve a Unix socket, reload config on SIGHUP,
flush stats every 10 s, and shut down cleanly on SIGTERM — the second SIGTERM
forcing exit (Lesson 09's homework!). Sketch the fd-based architecture and
why it has no races by construction.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
One epoll loop over four fds: the listening Unix socket (accept → per-client
fds join the set — Lesson 32); a <strong>signalfd</strong> for {SIGHUP,
SIGTERM} with both blocked process-wide from before any thread starts
(Lesson 24 Q3); a <strong>timerfd</strong>, monotonic, 10 s periodic, reading
the expiration count so a stalled loop knows it owes multiple flushes. Loop
body: dispatch on which fd woke — client data → handle; timerfd → flush;
signalfd → read the siginfo struct and switch on signo: SIGHUP → reload
config <em>right here, between events</em> (no handler ever runs, so reload
touches shared state with zero locking against "interrupted self");
first SIGTERM → set draining flag, stop accepting, arm a deadline timerfd;
second SIGTERM (just another readable event!) → exit(1). Race-free by
construction because <em>everything is serialized through one wait point</em>:
signals can't interrupt handlers mid-datastructure (none exist), timer vs
signal vs I/O ordering is just event order in one thread, and nothing needs
Phase 5 locks until you add worker threads — whose completion notices arrive,
naturally, on an eventfd in the same set.
</details>

---

## Homework

`ls -l /proc/PID/fd` on a busy daemon shows `anon_inode:[eventfd]`,
`[timerfd]`, `[signalfd]`, `pipe:`, and `socket:` entries. Pick a real daemon
on your VM (systemd itself, or sshd/journald), inventory its fd types (lab
step 5's technique), and for each type found, write one sentence hypothesizing
what that daemon uses it for — then verify one hypothesis with
`ss -xp`/`lsof`/documentation. What does the fd inventory tell you about a
process's architecture without reading any code?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Example for PID 1 (systemd): a dozen <code>[timerfd]</code> — one per
timer-unit + internal watchdogs (verify: <code>systemctl list-timers</code>
count correlates); <code>[signalfd]</code> — PID 1 receives SIGCHLD for the
whole world's reaping (Lesson 08) plus control signals, all fd-ified for its
single event loop; many <code>[eventfd]</code> — per-unit notify/wakeup
doorbells and sd_event internals; <code>socket:</code> Unix listeners —
/run/systemd/private (systemctl's channel — verify with <code>ss -xp |
grep systemd</code>) and the notify socket where services report readiness;
<code>inotify</code> — watching unit-file directories for changes;
<code>pipe:</code> — captured stdout/stderr of services flowing to the
journal. The meta-lesson: the fd table is an architecture diagram — "one
process, event-driven, N subsystems as fd classes" reads directly from
<code>ls -l /proc/1/fd</code>; a thread-pool design would instead show few
special fds and many <code>/proc/PID/task</code> entries (Lesson 24). You
can now classify any Linux service's concurrency model in thirty seconds,
from the outside — which is the whole phase, compressed into one trick.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 7 — Files & Filesystems (Lesson 36: File Descriptors) →](lesson-36-file-descriptors){: .btn .btn-primary }
