---
title: "Lesson 36 — File Descriptors, dup, and Redirection"
nav_order: 1
parent: "Phase 7: Files & Filesystems"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 36: File Descriptors, dup, and Redirection

## Concept

File descriptors have appeared in every phase — inherited across fork (07),
listed in /proc (04), passed over sockets (32). Time to reveal their real
anatomy, because fds are not one layer but **three**, and every fd mystery
dissolves once you can draw this picture:

```
  process A's fd table         OPEN FILE DESCRIPTIONS          inodes
  (per process)                (system-wide table)             (per file)
  ┌──────────────┐             ┌───────────────────┐
  │ 0 → ─────────┼──────┐      │ OFD #1            │        ┌─────────┐
  │ 1 → ─────────┼──┐   └────▶ │  mode: O_RDONLY   │ ─────▶ │ inode   │
  │ 2 → ─────────┼──┤          │  OFFSET: 4096 ◀── the      │ 88231   │
  │ 3 → ─────────┼─┐└────────▶ │ OFD #2            │ shared  │ (file's │
  └──────────────┘ │           │  mode: O_WRONLY   │ cursor! │  true   │
                   │           │  OFFSET: 0        │ ──────▶ │identity)│
  process B        │           └───────────────────┘         └─────────┘
  ┌──────────────┐ │              ▲
  │ 4 → ─────────┼─┼──────────────┘   ← fork/dup2/SCM_RIGHTS make fds
  └──────────────┘ └─ two fds, ONE OFD: they share the offset!
```

- **fd** — a small integer, an index into *your process's* table (Lesson 04's
  `/proc/PID/fd`).
- **open file description (OFD)** — created by each `open()`: holds the mode,
  flags, and crucially **the file offset** (where the next read/write
  happens).
- **inode** — the file itself (next lesson's star).

The middle layer is the one nobody teaches and everybody needs: `dup2`, fork,
and SCM_RIGHTS duplicate *fd table entries pointing at the same OFD* — so
they **share one offset**. Two separate `open()`s of the same file get two
OFDs — independent offsets. This single distinction explains why shell
redirection works, why parent and child appending to one log don't collide,
and a family of classic bugs.

---

## How It Works

### dup2 and the redirection algebra

`dup2(oldfd, newfd)` makes `newfd` another name for `oldfd`'s OFD (closing
what newfd held). Shell redirection is dup2 choreography performed in the
fork/exec gap (Lesson 07):

- `cmd > file` → open(file), dup2(that, 1)
- `cmd 2>&1` → dup2(1, 2) — "make 2 point where **1 points now**"

Which is why **order matters** — the classic interview puzzle:

```
cmd > f 2>&1     dup2(f,1); dup2(1,2)  → both → f          ✓ what you meant
cmd 2>&1 > f     dup2(1,2); dup2(f,1)  → 2→terminal, 1→f   ✗ stderr escaped!
```

Read `2>&1` as *assignment of current value*, not as a symbolic link, and it
never confuses you again.

### Why shared offsets are the design, not a bug

Parent and shell-launched child both writing to the session's log: because
their fds share one OFD (inheritance!), the offset advances jointly — writes
append after each other instead of overwriting from each process's private
idea of "position". With `O_APPEND`, the kernel goes further: every write
atomically seeks-to-end-and-writes — multi-process logging without
clobbering, no locks needed. Two independent `open()`s (two OFDs) *without*
O_APPEND is precisely the recipe for interleaved corruption — the bug you can
now diagnose from `lsof`'s offset column.

### The lifecycle rules that run your system

- Limits: `ulimit -n` (soft, often 1024) — "too many open files" is this;
  raise per-service via systemd `LimitNOFILE=`.
- **O_CLOEXEC** (Lesson 31's promised fix): mark fds close-on-exec so
  children don't inherit your database connections — modern APIs take it at
  creation (`open(..., O_CLOEXEC)`, `pipe2`, `SOCK_CLOEXEC`) precisely
  because setting it "later" races with fork in threaded programs (Lesson
  28's fork hazard, fd edition).
- An OFD (and the inode's data) lives until the *last* fd anywhere closes —
  the mechanics behind "deleted but space not freed" (Lesson 37 completes
  this).

{: .note }
> **`lsof` and fuser: the fd census**
> <code>lsof /path</code> — who has it open; <code>lsof -p PID</code> —
> everything a process holds (with offsets and modes!); <code>fuser -v
> /mnt</code> — who's blocking your unmount. Under the hood they read
> /proc/*/fd — Lesson 04's tour was the API all along.

---

## Lab

```bash
# ---- 1. Three layers made visible ----
$ exec 7</etc/hostname                 # open fd 7 in THIS shell
$ ls -l /proc/$$/fd/7
# lr-x------ ... 7 -> /etc/hostname
$ cat /proc/$$/fdinfo/7
# pos: 0        ← the OFD's offset, live!
# flags: 0100000
$ read -u 7 line && echo "read: $line"
$ cat /proc/$$/fdinfo/7 | head -1
# pos: 10       ← reading moved the OFD's cursor
$ exec 7<&-                            # close it

# ---- 2. Shared offset: fork-inherited fds are ONE cursor ----
$ python3 - << 'EOF'
import os
fd = os.open("/tmp/shared", os.O_WRONLY | os.O_CREAT | os.O_TRUNC)
if os.fork() == 0:
    os.write(fd, b"child")             # same OFD as parent's fd!
    os._exit(0)
os.wait()
os.write(fd, b"PARENT")                # continues AFTER child's write
os.close(fd)
print(open("/tmp/shared").read())      # childPARENT — one shared cursor
# now the CONTRAST: two independent open()s = two OFDs = two cursors
fd1 = os.open("/tmp/shared", os.O_WRONLY)
fd2 = os.open("/tmp/shared", os.O_WRONLY)
os.write(fd1, b"AAAA"); os.write(fd2, b"BBBB")   # both wrote at offset 0!
os.close(fd1); os.close(fd2)
print(open("/tmp/shared").read())      # BBBBPARENT — fd2 CLOBBERED fd1
EOF

# ---- 3. The redirection order puzzle, live ----
$ ls /nonexistent > /tmp/out 2>&1; wc -c < /tmp/out
# 60ish — stderr captured (dup2(f,1) THEN dup2(1,2))
$ ls /nonexistent 2>&1 > /tmp/out; wc -c < /tmp/out
# ls: cannot access...      ← appears on TERMINAL
# 0                          ← file empty: stderr had already been aimed
#                              at the terminal before 1 was redirected

# ---- 4. O_APPEND: atomic multi-writer logging ----
$ python3 - << 'EOF'
import os
# WITHOUT O_APPEND: two processes, independent offsets — corruption
plain = "/tmp/log_plain"; app = "/tmp/log_append"
for path, flags in [(plain, os.O_WRONLY|os.O_CREAT|os.O_TRUNC),
                    (app, os.O_WRONLY|os.O_CREAT|os.O_TRUNC|os.O_APPEND)]:
    pids = []
    for who in (b"AAAAAAAA\n", b"BBBBBBBB\n"):
        if (p := os.fork()) == 0:
            fd = os.open(path, flags)          # each opens SEPARATELY
            for _ in range(500): os.write(fd, who)
            os._exit(0)
        pids.append(p)
    for p in pids: os.wait()
for name, path in [("plain ", plain), ("append", app)]:
    lines = open(path, "rb").read().splitlines()
    total = len(lines)
    print(f"{name}: {total} lines survive (expected 1000)")
EOF
# plain : ~500-700 lines — the writers overwrote each other at shared offsets
# append: 1000 lines — O_APPEND's atomic seek+write saved every one

# ---- 5. Find the "deleted file still eating disk" ----
$ dd if=/dev/zero of=/tmp/big bs=1M count=200 2>/dev/null
$ python3 -c "f = open('/tmp/big'); import time; time.sleep(30)" &
$ rm /tmp/big; df -h /tmp | tail -1        # space NOT freed!
$ lsof +L1 2>/dev/null | grep deleted | head -2
# python3 ... /tmp/big (deleted)           ← the culprit, holding the inode
$ kill %1; sleep 1; df -h /tmp | tail -1   # space returns when last fd closes
$ rm -f /tmp/shared /tmp/out /tmp/log_plain /tmp/log_append
```

---

## Further Reading

| Topic | Link |
|---|---|
| `open(2)` man page | <https://man7.org/linux/man-pages/man2/open.2.html> |
| `dup2(2)` man page | <https://man7.org/linux/man-pages/man2/dup.2.html> |
| File descriptor (Wikipedia) | <https://en.wikipedia.org/wiki/File_descriptor> |
| `fcntl(2)` — OFD flags & locks | <https://man7.org/linux/man-pages/man2/fcntl.2.html> |
| `lsof(8)` man page | <https://man7.org/linux/man-pages/man8/lsof.8.html> |
| `proc(5)` — fdinfo | <https://man7.org/linux/man-pages/man5/proc.5.html> |

---

## Checkpoint

**Q1.** `cmd 2>&1 > file` and `cmd > file 2>&1` behave differently. Trace
both through the fd table, dup2 call by dup2 call.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Redirections apply left to right, each a dup2 of <em>current</em> state.
<code>&gt; file 2&gt;&amp;1</code>: (1) open file → dup2 onto fd 1 — fd 1's
OFD is now the file; (2) <code>2&gt;&amp;1</code> → dup2(1, 2) — fd 2 points
at <em>fd 1's current OFD</em> = the file. Both streams → file, sharing one
OFD (interleaving correctly via the shared offset!). Reversed,
<code>2&gt;&amp;1 &gt; file</code>: (1) dup2(1, 2) while fd 1 still holds the
terminal — fd 2 now (independently) points at the terminal's OFD; (2) fd 1 is
then redirected to the file — but fd 2's entry was already copied and doesn't
follow. Result: stdout → file, stderr → terminal. The rule that resolves it
forever: <code>2&gt;&amp;1</code> copies a table entry <em>by value at that
moment</em> — it creates no ongoing link between 2 and 1.
</details>

**Q2.** Lab 2 showed two `open()`s clobbering each other while fork-inherited
fds interleaved cleanly. State the rule, and explain how O_APPEND makes even
independent opens safe — including why it must be a kernel flag rather than
"seek to end, then write" in userspace.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Rule: the write position lives in the <em>open file description</em>, so who
shares a cursor is decided by how the fds came to be — fork/dup2/fd-passing
share one OFD (joint offset: appends chain naturally); separate open() calls
create separate OFDs (each starts at 0: last writer wins, corruption).
O_APPEND moves the append decision into the write operation itself: the
kernel atomically positions at current end-of-file <em>and</em> writes, as
one uninterruptible step against all other writers of that inode. Userspace
seek-then-write is check-then-act (Lesson 25!): between your lseek(END) and
write, another process appends — you overwrite their record. The window is
small, the corruption weekly. Same story one level up: this is why loggers
either use O_APPEND, a single writer process (Lesson 25's homework), or
line-buffered writes ≤ PIPE_BUF to a pipe (Lesson 31) — three solutions, one
race.
</details>

**Q3.** A service reports "Too many open files." Give the 4-step diagnosis
path using this lesson's tools, and the three most common root causes with
their distinct fingerprints.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Diagnosis: (1) confirm the limit vs usage — <code>ls /proc/PID/fd | wc
-l</code> against <code>cat /proc/PID/limits</code> (soft NOFILE); (2)
classify the population — <code>ls -l /proc/PID/fd | awk</code>-count by
type (socket:/pipe:/regular/anon_inode — Lesson 35's inventory trick); (3)
look for growth — sample the count over minutes; (4) match to code paths —
offsets/paths via lsof -p, socket peers via ss -p. Root causes: <strong>fd
leak</strong> — count grows monotonically, dominated by one type, often many
<code>(deleted)</code> entries (opened, never closed — each deploy's log
file, every retry's socket); <strong>legitimately undersized limit</strong> —
count stable at ~limit, healthy type mix, service simply handles that many
connections: raise LimitNOFILE (1024 is a 1990s default); <strong>inheritance
bloat</strong> — children hold copies of everything the parent had (no
O_CLOEXEC): fingerprint is the same fds appearing in whole process trees
(compare /proc/parent/fd with /proc/child/fd) — fix at creation flags, not
with a bigger limit.
</details>

---

## Homework

fcntl file locks come in two flavors with a famous trap: traditional POSIX
locks (`F_SETLK`) are owned *per process* and — bizarrely — released when the
process closes **any** fd for that file; modern **OFD locks** (`F_OFD_SETLK`)
are owned by the open file description. Construct the disaster scenario for
the traditional flavor (hint: a library function that innocently
opens-reads-closes the same file your program holds locked), and explain why
OFD ownership fixes it. Which of this lesson's three layers does each flavor
attach to?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Disaster: your program opens the database file, takes a POSIX write lock, and
begins a critical update. Mid-update it calls a helper (logging library,
config re-reader — any code) that happens to <code>open()</code> the same
file, read a header, and <code>close()</code> its fd. POSIX lock semantics:
<em>closing any fd referring to that file drops all of the process's locks on
it</em> — your lock silently evaporated mid-transaction; another process
acquires it and both write: corruption with no error anywhere. The trap is
composability: correctness now depends on every library never touching your
locked files. OFD locks attach to the open file description instead — the
helper's open created a <em>different</em> OFD, and its close releases only
locks on that OFD (none): your lock survives. Layer mapping: POSIX locks
attach (conceptually) to the (process, inode) pair — spanning layers 1 and 3
while skipping the middle, which is exactly why they interact catastrophically
with the fd table; OFD locks attach to layer 2, inheriting its sane
lifecycle (shared across dup/fork like the offset, released when the last fd
on <em>that OFD</em> closes). Modern rule: always F_OFD_SETLK (or flock,
which was OFD-scoped all along).
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 37 — VFS and Inodes →](lesson-37-vfs-inodes){: .btn .btn-primary }
