---
title: "Lesson 10 — Process Groups, Sessions, and Job Control"
nav_order: 5
parent: "Phase 2: Processes"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 10: Process Groups, Sessions, and Job Control

## Concept

Press Ctrl-C during `cat bigfile | grep foo | sort` and *all three* processes
die — but your shell survives, though it's attached to the very same terminal.
Who decided that? By what mechanism does one keystroke find exactly those three
processes?

The machinery is two grouping layers the kernel maintains above processes:

```
 session 1210  (leader: bash — "the login")
 │  controlling terminal: /dev/pts/1
 │
 ├── process group 1210: bash                        (background: just sits)
 ├── process group 4301: cat | grep | sort  ◀━━━ FOREGROUND group
 │                                               │
 └── process group 4310: make -j4 &   (background) │
                                                   │
        Ctrl-C ──▶ terminal driver ──▶ SIGINT to every member of
                                       the FOREGROUND group only
```

- A **process group** (PGID) is a set of related processes — the shell puts
  each *pipeline* into its own group. Signals can target a whole group at once
  (`kill -- -PGID`, Lesson 09).
- A **session** (SID) is a set of process groups under one login/terminal. The
  session's *leader* (your shell) owns a **controlling terminal**, and exactly
  one process group at a time is that terminal's **foreground group**: it may
  read from the terminal, and it receives the terminal's keystroke signals
  (Ctrl-C → SIGINT, Ctrl-Z → SIGTSTP, Ctrl-\ → SIGQUIT).

"Job control" — `&`, `jobs`, `fg`, `bg`, Ctrl-Z — is just your shell moving
groups between foreground and background and SIGCONT-ing stopped ones. And a
**daemon** is a process that has carefully *left* this whole structure so no
terminal event can ever touch it.

---

## How It Works

### The keystroke path

The terminal driver (the kernel's tty layer) holds two pieces of state: which
session it belongs to, and which PGID is foreground (`tcsetpgrp(3)`). On Ctrl-C
it sends SIGINT to *every process in the foreground group* — that's how one key
kills a whole pipeline while background jobs and the shell (different groups!)
are untouched. Background jobs that try to *read* the terminal get SIGTTIN and
stop — that's why a backgrounded interactive program freezes until `fg`.

Hang-ups ride the same rails: when the terminal goes away (ssh drops, terminal
window closed), the session leader gets **SIGHUP**, and the shell (or kernel)
propagates HUP to its jobs — the historical reason `nohup` exists.

### Making a daemon: leaving the session

A classic daemon must never get the terminal's signals, so at startup it:

1. `fork()` and let the parent exit — the child is *not* a group leader;
2. [`setsid()`](https://man7.org/linux/man-pages/man2/setsid.2.html) — become
   leader of a **new session** with **no controlling terminal**;
3. `fork()` *again* — now not even a session leader, so it can never acquire a
   terminal accidentally (the "double fork");
4. redirect stdin/stdout/stderr away from the tty, `chdir("/")`.

Modern practice: don't do this dance at all — run as a plain foreground process
and let **systemd** own the lifecycle (it starts services in their own session,
no terminal, with journal-connected stdio). You'll formalize that in Lesson 52.
The shell tools for ad-hoc detaching: `nohup cmd &` (ignore SIGHUP), `setsid
cmd` (new session now), `disown` (shell forgets the job — no HUP forwarding).

{: .note }
> **Reading the columns**
> <code>ps -o pid,ppid,pgid,sid,tty,stat,comm</code> shows the whole structure.
> In STAT: <code>s</code> = session leader, <code>+</code> = in the foreground
> group. A daemon shows <code>TTY ?</code>, SID = its own PID, and no
> <code>+</code> anywhere — check any systemd service and you'll see exactly
> that signature.

---

## Lab

```bash
# ---- 1. See the structure of a pipeline ----
$ sleep 300 | tr a b | cat &
$ ps -o pid,ppid,pgid,sid,stat,comm | tail -5
#   PID  PPID  PGID   SID STAT COMM
#  1210   800  1210  1210 Ss   bash        ← session leader (s)
#  4301  1210  4301  1210 S    sleep       ┐ one pipeline =
#  4302  1210  4301  1210 S    tr          │ one process group
#  4303  1210  4301  1210 S    cat         ┘ (PGID 4301 = first member)

# kill the whole pipeline with ONE group signal (note the -- and minus):
$ kill -- -4301
$ jobs
# [1]+ Terminated    sleep 300 | tr a b | cat

# ---- 2. Foreground vs background and Ctrl-C ----
$ sleep 300 &
$ sleep 400            # foreground now — press Ctrl-C
# ^C                   ← 400 dies (foreground group got SIGINT)
$ jobs
# [1]+ Running  sleep 300 &        ← 300 untouched: different group!

# ---- 3. Ctrl-Z / fg / bg: job control = group state changes ----
$ sleep 500
# press Ctrl-Z
# [2]+ Stopped   sleep 500
$ ps -o pid,stat,comm -p $(jobs -p | tail -1)
#  STAT T        ← stopped state (Lesson 08's T!)
$ bg %2          # shell sends SIGCONT; job runs in background
$ fg %2          # back to foreground... Ctrl-C to kill it

# ---- 4. Background jobs may not READ the terminal ----
$ cat &
# [3]+ Stopped   cat            ← instantly stopped: SIGTTIN
$ fg %3          # now it may read — type a line, see it echo, Ctrl-D, done

# ---- 5. Watch SIGHUP propagate — and evade it ----
$ bash                          # nested shell = new session? No! same session,
$ ps -o pid,sid,tty,comm -p $$  # new process, SAME sid/tty — check it
$ nohup sleep 600 & disown      # two independent shields
$ exit                          # leave nested shell — sleep survives; verify:
$ pgrep -a sleep
# 600-second sleep still running, now owned by... check its PPID and SID!

# ---- 6. A real daemon's signature ----
$ ps -o pid,pgid,sid,tty,stat,comm -p $(pgrep -o cron 2>/dev/null || pgrep -o sshd)
#  PID = PGID = SID, TTY = ?, STAT Ss — the setsid() fingerprint
```

---

## Further Reading

| Topic | Link |
|---|---|
| `setsid(2)` man page | <https://man7.org/linux/man-pages/man2/setsid.2.html> |
| `setpgid(2)` — process groups | <https://man7.org/linux/man-pages/man2/setpgid.2.html> |
| `credentials(7)` — PID/PGID/SID overview | <https://man7.org/linux/man-pages/man7/credentials.7.html> |
| Process group (Wikipedia) | <https://en.wikipedia.org/wiki/Process_group> |
| `daemon(7)` — old vs new style daemons | <https://man7.org/linux/man-pages/man7/daemon.7.html> |
| Job control (Wikipedia) | <https://en.wikipedia.org/wiki/Job_control_(Unix)> |

---

## Checkpoint

**Q1.** Why does Ctrl-C kill your pipeline but not your shell, even though both
are connected to the same terminal?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Ctrl-C doesn't go "to the terminal's processes" — the tty driver sends SIGINT
only to the <em>foreground process group</em>, and the shell arranged not to be
in it: it placed the pipeline into its own group and made that group foreground
(<code>tcsetpgrp</code>) before waiting. The shell sits in a different group of
the same session, so the signal never reaches it. When the pipeline dies, the
shell (notified via SIGCHLD/wait, Lesson 08) puts itself back in the
foreground and prints the next prompt. One keystroke, surgically scoped — by
group membership, not by terminal attachment.
</details>

**Q2.** Explain each step of the daemon double-fork: why isn't a single
`fork()` + `setsid()` enough, and what does the second fork add?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Fork #1 + parent exit: <code>setsid()</code> fails if the caller is already a
process-group leader (the group would be left leaderless); the forked child is
guaranteed not to lead anything, so setsid succeeds — giving it a fresh
session, own group, and <em>no controlling terminal</em>. But it is now a
<em>session leader</em>, and on some Unixes a session leader that opens a tty
device can <em>acquire it as its controlling terminal</em> accidentally. Fork
#2 and exiting the middle process leaves a worker that is in the new session
but not its leader — permanently incapable of acquiring a terminal. Belt and
suspenders; systemd makes the whole ritual obsolete but understanding it
explains a thousand old init scripts.
</details>

**Q3.** You run a long build over ssh and your connection drops. Yesterday the
same build survived a drop because you'd started it differently. List three
different ways you could have started it that survive the disconnect, and what
each one actually changes.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <code>nohup make … &</code> — the process ignores SIGHUP (disposition
change, Lesson 09) and its output goes to nohup.out, so the terminal's death
signal bounces off. (2) <code>setsid make …</code> — the build leaves your
session entirely (new SID, no controlling terminal); the HUP storm targets your
old session, and it isn't in it. (3) run inside <code>tmux</code>/<code>screen</code>
— the build's "terminal" is a pseudo-tty owned by the tmux server (a daemon in
its own session); your ssh dropping kills only the tmux <em>client</em>.
(Honorable mention: <code>disown</code> after backgrounding — the shell stops
forwarding HUP.) All four work by breaking a different link in the chain:
terminal → session leader → shell → job.
</details>

---

## Homework

Using only `ps -eo pid,ppid,pgid,sid,tty,stat,comm`, find on your VM: (a) one
session with more than one process group in it, (b) one process whose PID =
PGID = SID with TTY `?` (a daemon), and (c) the process currently holding the
foreground of your terminal. Explain how you identified each from the columns
alone.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) Filter rows sharing one SID (e.g. your login shell's) and look for
different PGID values among them — your shell plus any background job shows
this immediately; the shell's SID appears on all, PGIDs differ per pipeline.
(b) Rows with TTY <code>?</code> and PID = PGID = SID are setsid()-style
daemons — cron, sshd, systemd services all match; the equality is the
fingerprint of "became own session leader." (c) The foreground process of your
terminal is the row whose TTY matches yours and whose STAT contains
<code>+</code> — while running the ps itself, that's ps and your shell's
pipeline; <code>+</code> literally marks membership in the tty's current
foreground group, which is exactly what Ctrl-C would hit at this instant.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 11 — Credentials and Capabilities →](lesson-11-credentials-capabilities){: .btn .btn-primary }
