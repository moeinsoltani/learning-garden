---
title: "Lesson 08 — Process Lifecycle: Zombies, Orphans, wait"
nav_order: 3
parent: "Phase 2: Processes"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 08: Process Lifecycle — Zombies, Orphans, wait

## Concept

A process isn't just "running or not". It moves through a small set of states,
and the strangest one — the **zombie** — exists because of a simple rule:
*death is a message, and someone must read it*.

```
                 fork()
                   │
                   ▼
      ┌─────── R runnable/running ◀────────┐
      │            │                        │ wakeup
      │            │ waits for something    │ (data arrived,
      ▼            ▼                        │  lock freed…)
  T stopped    S sleeping ──────────────────┘
 (SIGSTOP,     D uninterruptible sleep (disk/NFS I/O — can't be signaled!)
  debugger)        │
                   │ exit() / fatal signal
                   ▼
              Z zombie  ← dead, but exit status not yet collected
                   │
                   │ parent calls wait*()   ("reaping")
                   ▼
                 gone
```

When a process exits, the kernel frees its memory and files — but keeps a tiny
corpse: the PID, the exit status, resource usage. Why? Because the parent has
the *right to know how its child died* (`waitpid` in your minishell printed
exactly that). Until the parent collects the status, the corpse — the zombie —
occupies a PID slot. A zombie consumes almost nothing, but a *leak* of zombies
(a parent that never waits) can exhaust PIDs.

Two special cases complete the picture:

- **Orphan**: the *parent* dies first. The child is re-parented to PID 1 (or a
  nearer "subreaper"), which dutifully waits on everything — so orphans get
  reaped and never linger.
- **D state**: a process stuck in *uninterruptible* sleep — usually mid-I/O on
  a slow or dead device (the hung-NFS classic). It ignores even SIGKILL, and it
  still counts toward load average (the mystery from Lesson 15's preview!).

---

## How It Works

### The wait family

[`waitpid(2)`](https://man7.org/linux/man-pages/man2/waitpid.2.html) blocks
until a child changes state and returns its PID + status (exit code, or which
signal killed it — the shell's `$?` of 139 = 128 + SIGSEGV(11) comes from
here). Variants: `wait` (any child), `WNOHANG` (poll, don't block), and the
elegant modern option — `SIGCHLD` delivery or a `pidfd` so you can wait via
poll/epoll along with everything else.

A parent that will never care can declare it: setting `SIGCHLD` to `SIG_IGN`
(or the `SA_NOCLDWAIT` flag) tells the kernel "auto-reap my children" — no
zombies. Shells and supervisors instead wait carefully, because they *do* care.

### Re-parenting and subreapers

When a parent dies, its children move to the nearest ancestor marked as a
**child subreaper** (`prctl(PR_SET_CHILD_SUBREAPER)`) or to PID 1. This is why
daemons double-fork (Lesson 10), why systemd can track services' strays, and
why in containers *your* entrypoint is PID 1 — and inherits the reaping duty,
a classic container bug when it doesn't (the famous "PID 1 zombie problem").

{: .note }
> **Why D state exists at all**
> Interruptible sleep (S) can be woken by a signal — the syscall returns
> <code>EINTR</code> and userspace retries. But some kernel paths hold hardware
> or filesystem state that must not be abandoned halfway; being killed there
> could corrupt data. So those sleeps are uninterruptible. Modern kernels added
> <code>TASK_KILLABLE</code> (D that accepts <em>fatal</em> signals) for many
> paths, but true D remains — and a process stuck there is unkillable until the
> I/O completes or the machine reboots. Diagnose with
> <code>/proc/PID/stack</code> (Lesson 04).

---

## Lab

```bash
# ---- 1. Make a zombie on purpose ----
$ python3 -c "
import os, time
pid = os.fork()
if pid == 0:
    os._exit(42)          # child dies immediately...
time.sleep(60)            # ...parent ignores it for 60s (no wait!)
" &
$ sleep 1; ps -o pid,ppid,state,comm | grep -E 'STATE|Z|python'
#   PID  PPID S COMM
#  4321  4320 Z python3 <defunct>     ← the corpse. 'defunct' = zombie

# it holds no memory — only the status. And you cannot kill the dead:
$ kill -9 4321; sleep 1; ps -p 4321 -o state,comm
# Z python3 <defunct>                 ← still there! only reaping removes it

# reap it: kill the PARENT (its exit re-parents + PID 1 reaps the zombie)
$ kill %1; sleep 1; ps -p 4321
#   gone.

# ---- 2. Orphans get adopted ----
$ python3 -c "
import os, time
pid = os.fork()
if pid == 0:
    time.sleep(5)                       # child outlives parent
    with open('/tmp/orphan.txt','w') as f:
        f.write(f'my new parent: {os.getppid()}\n')
    os._exit(0)
os._exit(0)                             # parent dies at once
"
$ sleep 6; cat /tmp/orphan.txt
# my new parent: 1        (or a user-session subreaper PID — check ps!)

# ---- 3. Exit status semantics: normal vs killed ----
$ bash -c 'exit 3'; echo $?
# 3                        ← normal exit, code 3
$ bash -c 'kill -SEGV $$'; echo $?
# 139                      ← 128 + 11: killed by signal 11 (SIGSEGV)

# ---- 4. See a D state (safe, brief) ----
# vmstat column 'b' counts D-state tasks; generate sync I/O pressure:
$ dd if=/dev/zero of=/tmp/big bs=1M count=800 oflag=sync & vmstat 1 4
#  b column briefly ≥1 while dd sits in uninterruptible writeback sleeps
$ rm -f /tmp/big /tmp/orphan.txt

# ---- 5. Who reaps in a shell? Watch bash wait: ----
$ strace -f -e trace=wait4 bash -c 'ls >/dev/null' 2>&1 | grep wait4
# wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], ...) = <child pid>
#   every command you've ever run ended in someone's wait4
```

---

## Further Reading

| Topic | Link |
|---|---|
| Zombie process | <https://en.wikipedia.org/wiki/Zombie_process> |
| Orphan process | <https://en.wikipedia.org/wiki/Orphan_process> |
| `wait(2)` / `waitpid(2)` | <https://man7.org/linux/man-pages/man2/wait.2.html> |
| `prctl(2)` — subreaper | <https://man7.org/linux/man-pages/man2/prctl.2.html> |
| Uninterruptible sleep | <https://en.wikipedia.org/wiki/Uninterruptible_sleep> |
| `ps(1)` — process state codes | <https://man7.org/linux/man-pages/man1/ps.1.html> |

---

## Checkpoint

**Q1.** Can you kill a zombie with `kill -9`? Why or why not, and what *does*
remove it?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
No. Signals — including SIGKILL — act on a running (or sleeping) process by
interrupting its execution; a zombie has no execution left to interrupt. It
already exited: its memory, files, and threads are gone. What remains is only a
kernel record holding the PID and exit status, waiting to be read. The only
things that remove it: its parent calling <code>wait*()</code> (reaping), or
the parent dying — re-parenting the zombie to PID 1/subreaper, which reaps it
immediately. Persistent zombies are therefore always a bug <em>in the
parent</em>, and the fix targets the parent, not the corpse.
</details>

**Q2.** Your container's app spawns helper processes, and `ps` inside the
container slowly fills with `<defunct>` entries. The same app on a normal host
shows none. Explain what's happening.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
In the container, the app itself runs as PID 1 of the PID namespace. When its
helpers' children are orphaned, they re-parent to <em>it</em> — and unlike real
init, the app never calls <code>wait()</code> for strangers, so their corpses
accumulate. On a normal host the same orphans re-parent to systemd, which reaps
everything. This is the classic "PID 1 zombie problem": fix it by running a
minimal init as PID 1 (<code>docker run --init</code>, tini) or by making the
app reap (handle SIGCHLD with a wait loop, or set SIGCHLD to SIG_IGN).
</details>

**Q3.** How do you distinguish these three "stuck process" situations from
`ps -o state` output, and which one can't SIGKILL fix: (a) waiting forever for
input that never comes, (b) stopped by a debugger, (c) blocked on a dead NFS
server?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) State <strong>S</strong> — interruptible sleep in read/poll; SIGKILL works
instantly. (b) State <strong>T</strong> — stopped; it isn't waiting for
anything, it's frozen by SIGSTOP/ptrace; SIGCONT (or detaching the debugger)
resumes it, and SIGKILL also works. (c) State <strong>D</strong> —
uninterruptible sleep inside a filesystem/driver path; SIGKILL is queued but
can't be delivered until the I/O path lets go, which on a dead NFS mount may be
never (mount with <code>intr</code>/timeouts, fix the server, or reboot).
Confirm with <code>/proc/PID/wchan</code> or <code>stack</code> — D on NFS
shows an nfs/rpc function.
</details>

---

## Homework

Write (or reason through) a tiny supervisor in Python or C: fork a child that
runs a command; if the child exits with non-zero status or dies from a signal,
log *how* it died (distinguish the two!) and restart it, up to 3 times. Use
`waitpid` + the `WIFEXITED`/`WIFSIGNALED` macros (`os.WIFEXITED` etc. in
Python). What does your supervisor share with systemd's `Restart=on-failure`?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<pre>
import os, sys, time
for attempt in range(3):
    pid = os.fork()
    if pid == 0:
        os.execvp(sys.argv[1], sys.argv[1:])
    _, status = os.waitpid(pid, 0)
    if os.WIFEXITED(status):
        code = os.WEXITSTATUS(status)
        if code == 0: sys.exit(0)
        print(f"exited with code {code}; restarting")
    elif os.WIFSIGNALED(status):
        print(f"killed by signal {os.WTERMSIG(status)}; restarting")
    time.sleep(1)
</pre>
This <em>is</em> the heart of every supervisor: systemd's Restart=on-failure is
the same fork/exec + waitpid loop with more policy (backoff via RestartSec,
rate limiting via StartLimitBurst, distinguishing exit codes via
SuccessExitStatus). The deep point: supervision requires being the
<em>parent</em> — only the parent gets the death notification and status —
which is why systemd insists on spawning services itself rather than adopting
already-running daemons.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 9 — Signals →](lesson-09-signals){: .btn .btn-primary }
