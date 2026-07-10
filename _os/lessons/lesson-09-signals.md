---
title: "Lesson 09 — Signals"
nav_order: 4
parent: "Phase 2: Processes"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 09: Signals

## Concept

You've been *sending* signals for lessons (`kill`, Ctrl-C, the SIGCHLD story).
Time to understand what a signal actually is: the kernel's way of tapping a
process on the shoulder, **asynchronously** — at any moment, between any two
instructions.

```
  sender                    kernel                       target process
  ──────                    ──────                       ──────────────
  kill(1234, SIGTERM) ──▶ set bit 15 in target's          ...running...
                          PENDING mask                        │
                                                              │ next return to
                                                              │ user space
                                                   ┌──────────▼──────────┐
                                                   │ pending & ~blocked? │
                                                   └──────────┬──────────┘
                                              per-signal disposition:
                                              ├─ default action  (die, stop, ignore…)
                                              ├─ SIG_IGN         (drop it)
                                              └─ handler         (interrupt the program,
                                                                  run handler, resume)
```

A signal is *not* a message with content — it's a single bit: "signal N
happened". No payload, no queueing of duplicates (a second SIGTERM while one is
pending is simply lost). It's the oldest IPC on Unix, and it's everywhere:
Ctrl-C is the terminal sending SIGINT (Lesson 10 shows by what right), `kill`
is a thin wrapper over the [`kill(2)`](https://man7.org/linux/man-pages/man2/kill.2.html)
syscall, timers fire SIGALRM, dying children fire SIGCHLD (Lesson 08), and
touching an unmapped address gets you SIGSEGV (Lesson 05's exceptions, delivered
as a signal).

---

## How It Works

Each signal has a **disposition** in the target: the *default action* (terminate
for most; stop for SIGSTOP/SIGTSTP; ignore for SIGCHLD; terminate+core for
SIGSEGV/SIGABRT), or `SIG_IGN`, or a **handler** — a function of yours the
kernel forcibly calls in the middle of whatever the program was doing.

Processes can also **block** signals (a mask): blocked signals stay *pending*
until unblocked — the tool for protecting critical sections from asynchronous
interruption. The four masks you saw in `/proc/PID/status` (Lesson 04) are
exactly: `SigPnd` (pending), `SigBlk` (blocked), `SigIgn` (ignored), `SigCgt`
(caught = has a handler).

Two signals accept **no** handler, no ignoring, no blocking: **SIGKILL** (9)
and **SIGSTOP** (19). They are the kernel's guaranteed last words — if
processes could catch SIGKILL, a malicious or buggy process could refuse to
die (D-state, Lesson 08, remains the only real exception).

### The classic pitfalls (interview favorites, real-outage favorites)

1. **Handlers run at any instruction.** If the main program was halfway through
   `malloc` and the handler also calls `malloc` → corrupted heap, rare and
   unreproducible. Only [async-signal-safe](https://man7.org/linux/man-pages/man7/signal-safety.7.html)
   functions (write, _exit, kill — roughly: raw syscalls) are legal in handlers.
   The professional pattern: handler sets a `volatile sig_atomic_t` flag (or
   writes a byte to a pipe — "self-pipe trick"), main loop checks it. This is
   what signalfd (Lesson 35) institutionalizes.
2. **Syscalls get interrupted.** A signal during a blocking `read` makes it
   return `-1 EINTR`. Code that doesn't expect that breaks mysteriously —
   `SA_RESTART` mitigates, but robust code handles EINTR.
3. **SIGTERM vs SIGKILL etiquette**: TERM first (catchable — lets the process
   flush, clean up, deregister), wait, KILL as last resort. `docker stop` and
   systemd both encode exactly this: TERM, grace period, KILL.

{: .note }
> **kill 0 and negative PIDs**
> <code>kill(pid, 0)</code> sends nothing — it just checks "may I signal this
> process / does it exist?" (a liveness probe). <code>kill(-PGID, sig)</code>
> signals a whole process group — how the shell nukes an entire pipeline at
> once; Lesson 10 builds on this.

---

## Lab

```bash
# ---- 1. Dispositions: default vs handler vs ignore ----
$ sleep 100 &
$ kill -TERM %1; wait %1 2>/dev/null; echo "status $?"
# status 143            ← 128+15: died by SIGTERM (default action)

# a handler changes the story — bash 'trap' installs one:
$ bash -c 'trap "echo caught TERM, cleaning up; exit 0" TERM; sleep 100' &
$ sleep 0.5; kill -TERM %1; wait %1; echo "status $?"
# caught TERM, cleaning up
# status 0              ← graceful. THIS is why TERM-then-KILL matters

# ignoring — and the unignorable:
$ bash -c 'trap "" TERM; echo try to TERM me: $$; sleep 30' &
$ sleep 0.5; kill -TERM %1; sleep 1; jobs %1
# [1]+ Running           ← TERM ignored!
$ kill -KILL %1; sleep 0.5; jobs %1 2>/dev/null || echo "KILL cannot be refused"

# ---- 2. See the masks in /proc (Lesson 04 pays off) ----
$ bash -c 'trap "" TERM; trap "echo hi" INT; grep -E "SigIgn|SigCgt" /proc/$$/status'
# SigIgn: ...4000        bit 15 set (SIGTERM ignored)
# SigCgt: ...0002        bit 2 set  (SIGINT caught)

# ---- 3. Pending + blocked: signals wait in line (no queue!) ----
$ python3 - << 'EOF'
import signal, os, time
signal.pthread_sigmask(signal.SIG_BLOCK, {signal.SIGUSR1})   # block it
os.kill(os.getpid(), signal.SIGUSR1)                          # send TWICE
os.kill(os.getpid(), signal.SIGUSR1)
hits = []
signal.signal(signal.SIGUSR1, lambda s, f: hits.append(1))
signal.pthread_sigmask(signal.SIG_UNBLOCK, {signal.SIGUSR1})  # release
time.sleep(0.1)
print(f"sent 2, delivered {len(hits)}")     # → 1. Signals are bits, not queues!
EOF

# ---- 4. EINTR: a signal interrupts a blocking syscall ----
$ python3 - << 'EOF'
import signal, os, time
signal.signal(signal.SIGALRM, lambda s, f: None)   # handler: do nothing
signal.alarm(2)                                    # SIGALRM in 2s
r, w = os.pipe()
t = time.time()
try:
    os.read(r, 100)                # blocks forever... until the signal
except InterruptedError as e:
    print(f"read interrupted after {time.time()-t:.1f}s: EINTR")
EOF

# ---- 5. Core dumps are a signal disposition too ----
$ ulimit -c unlimited
$ python3 -c 'import os,signal; os.kill(os.getpid(), signal.SIGSEGV)'
# Segmentation fault (core dumped)      ← "terminate + core" default action
# (where the core went: Lesson 65. coredumpctl may have caught it.)
```

---

## Further Reading

| Topic | Link |
|---|---|
| `signal(7)` — overview & full table | <https://man7.org/linux/man-pages/man7/signal.7.html> |
| `sigaction(2)` man page | <https://man7.org/linux/man-pages/man2/sigaction.2.html> |
| `kill(2)` man page | <https://man7.org/linux/man-pages/man2/kill.2.html> |
| `signal-safety(7)` — async-signal-safe list | <https://man7.org/linux/man-pages/man7/signal-safety.7.html> |
| Unix signal (Wikipedia) | <https://en.wikipedia.org/wiki/Signal_(IPC)> |

---

## Checkpoint

**Q1.** A process ignores SIGTERM. What are your escalation options, and what
does each cost?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
First check <em>why</em> it ignores TERM — maybe it handles a different
shutdown signal (SIGINT, SIGQUIT) or a control channel (HTTP endpoint, unix
socket). If not: SIGKILL always works (unless D-state) but costs you every
cleanup the process would have done — unflushed buffers lost (Lesson 02!),
temp files and lock files left behind, connections dropped mid-transaction; for
a database that can mean recovery on next start. In between: SIGHUP sometimes
means "reload/exit" by convention. The etiquette that tools encode (systemd,
docker stop): TERM → grace period → KILL, giving cooperation a chance before
force.
</details>

**Q2.** Why can't SIGKILL and SIGSTOP be caught, blocked, or ignored — what
property of the system would break?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
They are the administrator's (and kernel's) guaranteed controls. If SIGKILL
were catchable, a buggy or malicious process could install a handler (or just
SIG_IGN) and become unkillable — resource exhaustion with no remedy short of
reboot; the kernel's OOM killer (Lesson 22) also relies on SIGKILL actually
killing. If SIGSTOP were catchable, nothing could reliably freeze a runaway
process (debuggers, job control, container pause all depend on it). The pair
is the system's ultima ratio: <em>policy over any process's objection</em> —
which is precisely why well-behaved software listens for the polite, catchable
SIGTERM instead.
</details>

**Q3.** The lab sent SIGUSR1 twice while blocked, and only one delivery
happened. A teammate's daemon uses SIGUSR1 as "reload config now" and
occasionally misses reloads under load. Connect the two, and propose two fixes.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Standard signals don't queue: pending is one bit per signal, so N sends while
busy/blocked collapse into one delivery — the daemon reloads once and misses
the rest. Under load (long reload, or signals arriving during the handler while
the same signal is auto-blocked) that's exactly "occasionally misses reloads".
Fixes: (1) treat the signal as level-triggered, not edge-triggered — the handler
sets a flag and the main loop keeps reloading <em>while the flag is set</em>,
re-checking after each pass (coalescing is then harmless); (2) use a channel
with real semantics: signalfd/self-pipe into the event loop, a unix socket
command, or inotify on the config file. (Real-time signals SIGRTMIN+ do queue,
but a proper channel is the cleaner fix.)
</details>

---

## Homework

Implement a graceful-shutdown skeleton in Python or C: a "server" loops doing
1-second work chunks; SIGTERM must let the current chunk finish, run cleanup,
and exit 0 — while a second SIGTERM during shutdown forces immediate exit 1.
Use only the flag-from-handler pattern (no work in the handler). Test with
`kill`; verify statuses with `echo $?`.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<pre>
import signal, sys, time

stop_requested = 0
def on_term(sig, frame):
    global stop_requested
    stop_requested += 1
    if stop_requested >= 2:
        sys.exit(1)               # second TERM: give up immediately
signal.signal(signal.SIGTERM, on_term)

while not stop_requested:
    time.sleep(1)                 # one "work chunk" (finishes even if TERM arrives)
    print("chunk done")
print("cleaning up…")             # flush, close, deregister
time.sleep(0.5)
sys.exit(0)
</pre>
The handler only bumps a counter (async-signal-safe in spirit; in C use
<code>volatile sig_atomic_t</code> and check it in the loop). First TERM: loop
notices the flag after the current chunk → cleanup → exit 0 (verify: kill PID;
echo $? → 0). Second TERM during cleanup: handler exits 1 immediately. This
two-stage pattern is exactly what systemd's TERM→KILL window expects from a
well-behaved service, implemented from the process's side.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 10 — Process Groups, Sessions, and Job Control →](lesson-10-sessions-job-control){: .btn .btn-primary }
