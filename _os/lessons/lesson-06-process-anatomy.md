---
title: "Lesson 06 — Anatomy of a Process"
nav_order: 1
parent: "Phase 2: Processes"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 06: Anatomy of a Process

## Concept

You've been saying "process" for five lessons. Time to define it precisely.

A **process** is the kernel's bundle of everything one running program needs:

```
 ┌───────────────────────── process (task_struct) ─────────────────────────┐
 │                                                                          │
 │  identity        PID, PPID, UIDs/GIDs, cgroup, namespaces                │
 │  address space   the virtual memory map (maps — Lesson 18)     ┐ shared  │
 │  open files      the fd table (Lesson 36)                      │ by      │
 │  signal state    handlers, masks (Lesson 09)                   ┘ threads │
 │  ...                                                                     │
 │  thread 1: registers + stack + scheduling state  ← the part that RUNS    │
 │  thread 2: registers + stack + scheduling state                          │
 └──────────────────────────────────────────────────────────────────────────┘
```

The kernel's name for the structure holding all this is the
`task_struct` — you met its rendered form already: `/proc/PID/status` *is* a
task_struct pretty-printed (Lesson 04).

One subtlety will pay off across the whole track: the kernel actually schedules
**threads** (it calls them *tasks*), not processes. A "process" is a group of
threads sharing the resources above. That's why two numbers exist:

- **TGID** (thread group ID) — what you normally call "the PID"; shared by all
  threads of a process. `getpid()` returns this.
- **PID** in kernel terms — unique per *thread*. `gettid()` returns this.

For a single-threaded program the two are equal, and `ps` shows TGIDs. This
becomes essential in Lesson 24 (threads) and explains oddities like a "process"
using 400% CPU.

---

## How It Works

### The process tree

Every process is created by another (Lesson 07), so all of user space forms a
tree rooted at PID 1 (`init`/systemd — the one the kernel starts directly at
boot). Each process records its parent's PID (**PPID**). The tree isn't
decoration: parents get first claim on their children's exit status (Lesson
08), signals travel along terminal/session lines (Lesson 10), and children
*inherit* most of the parent's setup — environment, working directory, open
fds, credentials — which is why "set an env var and run the command" works.

### Reading /proc/PID/status like a pro

The fields you'll use constantly (many will get their own lesson):

| Field | Meaning | Deep dive |
|---|---|---|
| `State` | R running, S sleeping, D uninterruptible, T stopped, Z zombie | Lesson 08 |
| `Tgid` / `Pid` | process ID / this thread's ID | Lesson 24 |
| `PPid` | parent | Lesson 07 |
| `Uid`/`Gid` (4 values each) | real, effective, saved, fs | Lesson 11 |
| `VmSize` / `VmRSS` | virtual / resident memory | Lesson 18 |
| `Threads` | thread count | Lesson 24 |
| `SigPnd`/`SigBlk`/`SigIgn`/`SigCgt` | signal masks | Lesson 09 |
| `voluntary/nonvoluntary_ctxt_switches` | how it gives up the CPU | Lesson 12 |
| `NSpid`, cgroup file nearby | namespace/cgroup membership | Phase 12 |

{: .note }
> **PIDs recycle**
> PIDs are allocated from a limited range (`/proc/sys/kernel/pid_max`) and
> reused after a process dies. So a stored PID can later name a *different*
> process — the classic race behind buggy `kill $SAVED_PID` scripts. The modern
> fix is <a href="https://man7.org/linux/man-pages/man2/pidfd_open.2.html">pidfd</a>:
> a file descriptor that pins the identity of one specific process (fds again —
> Lesson 35's pattern).

---

## Lab

```bash
# 1. The whole tree, rooted at PID 1:
$ pstree -p | head -15
# systemd(1)─┬─sshd(800)───sshd(1200)───bash(1210)───pstree(1301)
#            ├─cron(750)
#            └─...            your shell's ancestry traces back to PID 1

# 2. Climb the ancestry by hand, PPID by PPID:
$ cat /proc/$$/status | grep -E '^(Name|Pid|PPid)'
$ PP=$(grep ^PPid /proc/$$/status | awk '{print $2}')
$ grep -E '^(Name|PPid)' /proc/$PP/status
# repeat until you reach PPid 0 — who are the two processes with PPid 0?

# 3. Tgid vs Pid — identical for your single-threaded shell:
$ grep -E '^(Tgid|Pid|Threads)' /proc/$$/status
# Tgid: 1210 / Pid: 1210 / Threads: 1

# now a multi-threaded process (any browser/daemon; or start one):
$ sleep 100 &   # boring, 1 thread — find something threaded instead:
$ ps -eLf | awk '$2!=$4' | head -5    # LWP column ≠ PID ⇒ extra threads
# pick a PID from there:
$ ls /proc/<PID>/task/                 # one dir per thread — each a task!
$ grep Threads /proc/<PID>/status

# 4. Inheritance: children copy the parent's environment & cwd at fork time
$ export MARKER=hello
$ bash -c 'echo $MARKER; pwd'          # child sees both
$ cd /tmp && bash -c pwd; cd - >/dev/null
# but it is a COPY — child changes don't flow back:
$ bash -c 'export MARKER=changed'; echo $MARKER   # still hello

# 5. ps is /proc, part 2 (you proved the mechanism in Lesson 04):
$ ps -o pid,ppid,state,comm -p $$
#   matches exactly what you read from status by hand

# 6. PID limits and recycling:
$ cat /proc/sys/kernel/pid_max
# 4194304 (or 32768 on older setups)
$ for i in 1 2 3; do bash -c 'echo $$'; done
# usually consecutive-ish numbers — watch the counter climb and imagine wrap-around
```

---

## Further Reading

| Topic | Link |
|---|---|
| Process (computing) | <https://en.wikipedia.org/wiki/Process_(computing)> |
| `proc(5)` — status fields | <https://man7.org/linux/man-pages/man5/proc.5.html> |
| `pstree(1)` man page | <https://man7.org/linux/man-pages/man1/pstree.1.html> |
| `getpid(2)` / `gettid(2)` | <https://man7.org/linux/man-pages/man2/getpid.2.html> |
| `pidfd_open(2)` man page | <https://man7.org/linux/man-pages/man2/pidfd_open.2.html> |
| `credentials(7)` man page | <https://man7.org/linux/man-pages/man7/credentials.7.html> |

---

## Checkpoint

**Q1.** What is the difference between a PID and a TGID, and which one does
`ps` show you?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The kernel schedules threads (tasks), and every thread has its own unique
kernel-level PID. Threads that share an address space, fd table, etc. form a
<em>thread group</em>, identified by the TGID — the PID of the group's first
thread. What userspace calls "the process's PID" (and what <code>ps</code>,
<code>kill</code>, and <code>getpid()</code> use) is the TGID. Per-thread IDs
show up as LWP in <code>ps -eLf</code>, as directories in
<code>/proc/PID/task/</code>, and from <code>gettid()</code>. Equal for
single-threaded programs, which is why the distinction hides so well.
</details>

**Q2.** In the lab you found the ancestry chain ends at two processes with
PPid 0. Which two, and why is PPid 0 special?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
PID 1 (<code>systemd</code>/init — ancestor of all of user space) and PID 2
(<code>kthreadd</code> — parent of all kernel threads, Lesson 01). PPid 0 means
"no user-space parent": these two were created directly by the kernel at boot
rather than by fork. The tree therefore really has two roots — one for user
space, one for kernel threads — and "PID 0" itself isn't a process you can see:
it's the kernel's per-CPU idle task.
</details>

**Q3.** A monitoring script stores a worker's PID, and an hour later runs
`kill -9 $PID` to clean it up. Describe the failure mode and a safer approach.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
PIDs recycle: if the worker already exited, its number may now belong to an
unrelated process — the script kills an innocent victim (classically, something
important). This is a time-of-check/time-of-use race on a recycled identifier.
Safer: verify identity before killing (compare <code>/proc/PID/exe</code>,
start time from <code>/proc/PID/stat</code>, or cmdline) — or eliminate the
race entirely with <code>pidfd_open()</code> + <code>pidfd_send_signal()</code>,
where the fd names <em>that one process</em> forever; or better, let the parent
(which can wait on its child, Lesson 08) or a supervisor like systemd own the
lifecycle.
</details>

---

## Homework

Using only /proc (no `ps`, no `pstree`), reconstruct your shell's full ancestry:
write a loop that starts at `$$` and follows PPid up to PID 1, printing
`PID  Name` per line. Then answer: which fields of `/proc/PID/status` stayed
identical down the whole chain, and why would that be?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<pre>
p=$$
while [ "$p" != 0 ]; do
  name=$(awk '/^Name:/{print $2}' /proc/$p/status)
  echo "$p  $name"
  p=$(awk '/^PPid:/{print $2}' /proc/$p/status)
done
</pre>
Typical chain: bash ← sshd ← sshd ← systemd. Fields identical down the chain
usually include the Uid/Gid lines (your login's identity was set once by
sshd/login and inherited unchanged — Lesson 11) and namespace/cgroup membership
(all in the default namespaces, same session slice — Phase 12). This is
inheritance made visible: children copy the parent's identity and context at
fork and only deliberate acts (setuid, unshare, cgroup moves) change them.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 7 — fork and exec →](lesson-07-fork-exec){: .btn .btn-primary }
