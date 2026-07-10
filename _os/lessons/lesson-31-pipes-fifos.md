---
title: "Lesson 31 — Pipes and FIFOs"
nav_order: 1
parent: "Phase 6: Inter-Process Communication"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 31: Pipes and FIFOs

## Concept

You have typed `|` thousands of times. Today you learn what it is: the
**pipe** — the oldest, simplest IPC on Unix, and still the daily workhorse.

A pipe is a small in-kernel byte buffer with two file descriptors: a write
end and a read end. Bytes go in one end, come out the other, in order, once.

```
   pipe(fd)  →  fd[1] ══▶ ┌─────────────────┐ ══▶ fd[0]
                (write)   │  kernel buffer   │    (read)
                          │  (64 KiB default)│
                          └─────────────────┘
   full?  writer BLOCKS  ← backpressure, built into the kernel!
   empty? reader BLOCKS
   all write ends closed? reader gets EOF (read returns 0 — L02!)
   all read ends closed?  writer gets SIGPIPE / EPIPE
```

How does `ls | wc -l` use this? Pure Lesson 07 choreography: the shell calls
`pipe()`, forks twice; in the first child it `dup2`s the pipe's write end onto
fd 1 and execs `ls`; in the second, the read end onto fd 0 and execs `wc`.
Neither program knows the pipe exists — they read stdin and write stdout as
always. The pipe rides fd inheritance across fork (the deep reason fds
survive exec — the whole shell composition model depends on it).

Two properties do the heavy lifting:

- **Backpressure for free** (Lesson 30's treasured queue property!): a fast
  producer piping into a slow consumer simply blocks when the buffer fills.
  `yes | head` doesn't melt your machine *because* of this.
- **Lifecycle signaling built in**: EOF when writers vanish; SIGPIPE when
  readers vanish — the pipeline tears itself down without coordination.

A **FIFO** (named pipe, `mkfifo`) is the same kernel object given a filesystem
name, so *unrelated* processes (no common ancestor to inherit fds from) can
find it. Same semantics, different rendezvous.

---

## How It Works

### The rules that explain every pipe mystery

1. **Capacity**: 64 KiB by default (`fcntl(F_SETPIPE_SZ)` adjusts). Writes up
   to `PIPE_BUF` (4 KiB) are **atomic** — never interleaved with other
   writers' data: why multiple processes can append lines to one logger pipe
   safely, and why bigger writes can shear.
2. **Blocking dance**: write blocks when full, read blocks when empty (unless
   O_NONBLOCK — then EAGAIN, and you're in Lesson 43's event-loop world).
3. **EOF requires ALL write ends closed**: the #1 practical bug — a forgotten
   duplicate write fd (often in the reader itself, inherited at fork!) keeps
   the pipe "open" and the reader waits forever. Close what you don't use,
   immediately after fork.
4. **SIGPIPE kills by default** (Lesson 09): writing to a reader-less pipe
   delivers it — which is exactly how `head` stops `yes`: head exits, yes's
   next write dies. Servers ignore SIGPIPE and handle EPIPE instead.
5. **Unidirectional**: two-way needs two pipes (or a socketpair — next
   lesson).

### FIFOs and their rendezvous semantics

`mkfifo /tmp/f` creates the name; no data exists until both sides open it —
`open` for reading *blocks until a writer arrives* (and vice versa), making
open itself a synchronization point. Multiple writers, one reader is the
classic log-collector shape (PIPE_BUF atomicity!). FIFOs show as type `p` in
`ls -l`, size always 0 — the data lives in kernel memory, never on disk
(compare /proc, Lesson 04: names in the filesystem, contents elsewhere).

{: .note }
> **Pipes in your toolbox beyond `|`**
> Process substitution <code>diff <(sort a) <(sort b)</code> is pipes +
> /dev/fd names; the self-pipe trick turns signals into event-loop input
> (Lesson 09 → 35); <code>splice(2)</code> moves data pipe↔file with zero
> copies (Lesson 45); and pagers, <code>tee</code>, coprocesses — all this
> one object. Few kernel objects earn their bytes like the pipe.

---

## Lab

```bash
# ---- 1. Watch the shell build a pipeline (Lesson 07's dance, live) ----
$ strace -f -e trace=pipe2,dup2,execve bash -c 'ls | wc -l' 2>&1 | grep -E 'pipe2|dup2|execve' | head -8
# pipe2([3, 4], 0)                     ← the buffer + two fds
# [pid A] dup2(4, 1)  … execve("/usr/bin/ls")   ← ls's stdout = write end
# [pid B] dup2(3, 0)  … execve("/usr/bin/wc")   ← wc's stdin  = read end

# ---- 2. Backpressure: measure the buffer, feel the block ----
$ mkfifo /tmp/fifo
$ dd if=/dev/zero of=/tmp/fifo bs=1024 count=1000 &   # writer, no reader yet
$ sleep 1; jobs -l                                     # still Running: blocked
$ cat /proc/$(jobs -p)/wchan; echo                     # pipe_write — named!
$ head -c 999999 /tmp/fifo > /dev/null                 # a reader drains it
$ wait; echo "writer finished only when read"

# ---- 3. EOF semantics: the forgotten-fd bug, reproduced ----
$ python3 - << 'EOF'
import os
r, w = os.pipe()
pid = os.fork()
if pid == 0:                       # child = writer
    os.close(r)                    # closes ITS copy of the read end (good habit)
    os.write(w, b"hello")
    os.close(w)
    os._exit(0)
# parent = reader. BUG: parent still holds w!
os.close(w)                        # ← comment this line out and read() below
                                   #   hangs forever after "hello": the pipe
                                   #   still has a write end open — OURS.
os.close # (no-op line for clarity)
data = b""
while chunk := os.read(r, 4096):
    data += chunk
print("got:", data, "then clean EOF")
os.waitpid(pid, 0)
EOF

# ---- 4. SIGPIPE: how `head` stops `yes` ----
$ yes | head -1
# y                        ← instant exit. Where did yes go?
$ yes > >(head -1 >/dev/null) & sleep 0.2; wait $! 2>/dev/null; echo "yes exit: $?"
# 141 = 128 + 13 (SIGPIPE) — Lesson 08's arithmetic names the killer
$ trap '' PIPE; yes 2>/dev/null | head -1 >/dev/null; trap - PIPE
# with SIGPIPE ignored, yes would get EPIPE errors instead (server pattern)

# ---- 5. PIPE_BUF atomicity: parallel writers, unsheared lines ----
$ for i in 1 2 3 4; do
    ( for j in $(seq 200); do echo "writer$i line$j padded-to-be-a-full-line"; done > /tmp/fifo ) &
  done
$ cat /tmp/fifo | awk '{print $1}' | sort | uniq -c
# 200 writer1 / 200 writer2 / ... — every line intact: writes < 4KiB are atomic.
$ rm /tmp/fifo

# ---- 6. Pipes are fds are inheritance: see one in /proc ----
$ sleep 100 | sleep 100 &
$ ls -l /proc/$(jobs -p | tail -1)/fd | grep pipe
# 0 -> pipe:[181542]      ← the kernel object's inode number;
$ ss -x 2>/dev/null | head -1; lsof -p $(jobs -p | tail -1) 2>/dev/null | grep FIFO
$ kill $(jobs -p)
```

---

## Further Reading

| Topic | Link |
|---|---|
| `pipe(2)` man page | <https://man7.org/linux/man-pages/man2/pipe.2.html> |
| `pipe(7)` — semantics & PIPE_BUF | <https://man7.org/linux/man-pages/man7/pipe.7.html> |
| `mkfifo(1)` / FIFO(7) | <https://man7.org/linux/man-pages/man7/fifo.7.html> |
| Pipeline (Unix) | <https://en.wikipedia.org/wiki/Pipeline_(Unix)> |
| `splice(2)` man page | <https://man7.org/linux/man-pages/man2/splice.2.html> |

---

## Checkpoint

**Q1.** `yes | head -1` exits immediately although `yes` loops forever. Trace
the exact mechanism that stops `yes`.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
head reads one line, prints it, and exits — closing the pipe's only read end.
yes, meanwhile, keeps writing; its next <code>write()</code> to a pipe with no
readers triggers the kernel's rule: deliver <strong>SIGPIPE</strong> to the
writer. yes doesn't handle it, and SIGPIPE's default action is terminate
(Lesson 09) — exit status 141 = 128+13. Nothing polled, nothing was signaled
explicitly by head, no coordination existed: pipeline teardown is an emergent
property of fd lifecycle + one signal. (The design's why: a filter's purpose
vanishes when its consumer leaves — killing it by default is the right
economy; programs that outlive readers, like servers writing to clients,
ignore SIGPIPE and handle the EPIPE error instead.)
</details>

**Q2.** In lab 3, forgetting `os.close(w)` in the *reader* makes it hang after
receiving all data. Explain precisely why, and state the general fd-hygiene
rule for fork-based IPC.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
EOF on a pipe is defined as "read returns 0 <em>when the buffer is empty and
no write ends exist</em>." fork duplicated both fds into the child, so the
system has two write-end descriptors: the child's (closed when it finished)
and the parent's own inherited copy — which the parent never closed. From the
kernel's view a writer still exists (it can't know the only remaining writer
is the reader itself), so read() blocks awaiting data that can never come:
deadlock by reference counting. Rule: <strong>immediately after fork, every
process closes the pipe ends it will not use</strong> — writer closes r,
reader closes w — before any I/O. The same leak wears other disguises:
inherited fds in daemons holding devices open, a child inheriting the
listening socket keeping a "dead" server's port busy — fd inheritance is
powerful (lab 1) and must be groomed (O_CLOEXEC, Lesson 36, is the systematic
fix).
</details>

**Q3.** Four processes write log lines to one FIFO and a single collector
reads it. What guarantees keep the lines unscrambled, what's the size limit
on that guarantee, and what happens beyond it?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
POSIX guarantees writes of ≤ <code>PIPE_BUF</code> bytes (4096 on Linux) to a
pipe/FIFO are <strong>atomic</strong>: the kernel inserts each such write as
an indivisible unit, never interleaving it byte-wise with other writers —
so as long as each log line is a single write() call of ≤4 KiB, lines arrive
intact (lab 5's proof). Requirements hidden in that sentence: <em>one write
per line</em> — line-buffered stdio or explicit single writes; two write()
calls for one line can still interleave. Beyond PIPE_BUF, atomicity ends: the
kernel may split the write into pieces placed between other writers' data —
long lines shear mid-line, corrupting the stream. Fixes at scale: keep
records ≤4 KiB, add framing (length prefixes) and reassemble, or move to
datagram sockets (next lesson) where message boundaries are first-class at
larger sizes.
</details>

---

## Homework

Implement `ls | wc -l` yourself in Python or C — no shell: create the pipe,
fork two children, wire stdin/stdout with dup2, exec both programs, close all
the right fds in all three processes, and wait for both (collecting exit
statuses). Then break it deliberately: leave the parent's write-end open and
report what happens and why.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<pre>
import os
r, w = os.pipe()
if (ls := os.fork()) == 0:
    os.close(r)
    os.dup2(w, 1); os.close(w)          # stdout → pipe
    os.execvp("ls", ["ls"])
if (wc := os.fork()) == 0:
    os.close(w)                          # ← the critical close!
    os.dup2(r, 0); os.close(r)           # stdin ← pipe
    os.execvp("wc", ["wc", "-l"])
os.close(r); os.close(w)                 # parent uses neither
print("ls:", os.waitpid(ls, 0)[1], " wc:", os.waitpid(wc, 0)[1])
</pre>
Note wc's child must close <code>w</code> <em>before</em> exec — it inherited
it! With the parent's <code>os.close(w)</code> removed: ls finishes and exits,
but wc never sees EOF (the parent's write end keeps the pipe alive — lab 3's
bug at pipeline scale) — wc counts everything, then blocks in read forever;
the parent blocks in waitpid on wc: the program hangs after doing all its
work, the most confusing kind of hang. The exercise's real lesson: the shell
does this bookkeeping perfectly, silently, for every pipeline you've ever
typed — three processes, six fd decisions, zero errors, every time.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 32 — Unix Domain Sockets →](lesson-32-unix-sockets){: .btn .btn-primary }
