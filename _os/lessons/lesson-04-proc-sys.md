---
title: "Lesson 04 — /proc and /sys: the Kernel's Dashboard"
nav_order: 4
parent: "Phase 1: The Kernel Boundary"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 04: /proc and /sys — the Kernel's Dashboard

## Concept

The kernel's memory is unreachable (Lesson 01), and syscalls are a narrow door
(Lesson 02). So how did commands like `ps`, `free`, and `ip` show you all that
kernel-owned information?

Through two **virtual filesystems** — [`/proc`](https://man7.org/linux/man-pages/man5/proc.5.html)
and `/sys`. They look like directories of text files, but *nothing behind them
exists on disk*. Every "file" is a window: when you `read()` it, the kernel
generates the content on the fly from its live data structures; when you
`write()` to the writable ones, you change kernel state.

```
 $ cat /proc/uptime
      │
      ▼ read() syscall
 ┌─────────────────────────────┐
 │ kernel: "someone read       │     no disk involved —
 │  uptime → format my         │     the file IS the kernel
 │  internal counter as text"  │     data structure, rendered
 └─────────────────────────────┘     as text at read time
      │
      ▼
 "123456.78 987654.32"
```

You've used this constantly without ceremony: `sysctl` (networking Lessons
34–35) is just a `/proc/sys` editor; `/proc/PID/maps` appeared in Lesson 01;
KVM stats in virtualization Lesson 11 came from debugfs, a third member of the
same family. This lesson makes you deliberately fluent: after it, "check /proc"
becomes your first move whenever you wonder what the kernel thinks.

The division of labor:

- **/proc** — *processes* (one numbered directory per PID) plus a grab-bag of
  system-wide info that predates /sys (`meminfo`, `cpuinfo`, `interrupts`).
- **/sys** — the *device and kernel-object model*: one directory per device,
  driver, module, class; heavily structured, one value per file.

---

## How It Works

Both are implementations of the VFS (Phase 7's topic): the kernel lets any
subsystem register "here's a filename, here's the function to run when someone
reads it." That's why:

- Sizes are lies — most /proc files report size 0 yet produce content; the
  content doesn't exist until your `read()` triggers the generating function.
- Content is **instant-generated**: two reads can differ (it's live state).
- Some files are huge dumps (`/proc/meminfo`), others one value
  (`/sys/class/net/eth0/mtu`) — /sys enforces one-value-per-file by convention.

### The /proc/PID toolbox

For any process (use `$$` for your shell, or `self` for "whoever is reading"):

| File | What it shows |
|---|---|
| `status` | human-readable summary: state, memory, UIDs, signal masks |
| `cmdline` | exact argv (NUL-separated — pipe through `tr '\0' ' '`) |
| `environ` | the environment it was *started* with |
| `maps` | the whole virtual address space (Phase 4 lives here) |
| `fd/` | symlinks: every open file descriptor → what it's connected to |
| `exe`, `cwd`, `root` | symlinks to its binary, working dir, and root |
| `stack` | current kernel stack trace — *why is it stuck?* (root only) |
| `wchan` | one-word version: which kernel function it sleeps in |

`ps`, `top`, `lsof`, `ss` are, to a first approximation, pretty-printers for
these files — the lab proves it.

{: .note }
> **Writable /proc and /sys files are live kernel switches**
> `echo 1 > /sys/class/net/eth0/...` or `sysctl -w` changes take effect
> immediately and vanish on reboot (persistence was networking Lesson 35's
> topic). The kernel validates writes — you can't write garbage — but you *can*
> write unwise values: treat write access to these trees as root-level control
> of the machine, because it is.

---

## Lab

```bash
# ---- /proc/PID: dissect your own shell ----
$ ls /proc/$$/
# status  cmdline  environ  maps  fd/  exe  cwd  root  stack  wchan ...

$ grep -E '^(State|VmRSS|Threads)' /proc/$$/status
# State:   S (sleeping)      ← sleeping = waiting for input (you!)
# VmRSS:      5000 kB        ← actual RAM in use
# Threads:    1

$ tr '\0' ' ' < /proc/$$/cmdline; echo
# -bash

# where is my shell "stuck" right now?
$ cat /proc/$$/wchan; echo
# do_wait   (bash is waiting for its child — the cat you just ran!)

# ---- fd/: what is this process connected to? ----
$ ls -l /proc/$$/fd
# 0 -> /dev/pts/1    1 -> /dev/pts/1    2 -> /dev/pts/1   (terminal)
$ sleep 100 > /tmp/out.log &
$ ls -l /proc/$!/fd
# 1 -> /tmp/out.log         ← redirection is just fd 1 pointing elsewhere!
$ kill %1

# ---- prove ps is a /proc pretty-printer ----
$ strace -e trace=openat ps aux 2>&1 | grep -c '/proc/'
# dozens of opens: /proc/1/stat, /proc/2/stat, ... ps just reads them all

# ---- system-wide /proc ----
$ head -3 /proc/meminfo         # free(1) reads this
$ grep 'model name' /proc/cpuinfo | head -1
$ cat /proc/loadavg             # uptime/top read this
$ cat /proc/sys/kernel/hostname # sysctl kernel.hostname reads this

# ---- /sys: the device model, one value per file ----
$ cat /sys/class/net/lo/mtu
# 65536
$ ls /sys/class/net/            # every netdev you built in networking appears here
$ cat /sys/kernel/uname/release 2>/dev/null || uname -r

# a writable switch you already know, both spellings:
$ sysctl net.ipv4.ip_forward
$ cat /proc/sys/net/ipv4/ip_forward     # same kernel variable, two doors

# ---- the sizes are lies ----
$ ls -l /proc/meminfo /proc/cpuinfo
# -r--r--r-- ... 0 ... /proc/meminfo    ← size 0, yet cat shows plenty
```

---

## Further Reading

| Topic | Link |
|---|---|
| `proc(5)` man page | <https://man7.org/linux/man-pages/man5/proc.5.html> |
| procfs (Wikipedia) | <https://en.wikipedia.org/wiki/Procfs> |
| sysfs (Wikipedia) | <https://en.wikipedia.org/wiki/Sysfs> |
| `sysctl(8)` man page | <https://man7.org/linux/man-pages/man8/sysctl.8.html> |
| `lsof(8)` man page | <https://man7.org/linux/man-pages/man8/lsof.8.html> |

---

## Checkpoint

**Q1.** `/proc/PID/fd` shows symlinks. To what, and why is that useful for
debugging?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Each symlink is named after a file-descriptor number and points at whatever that
fd is connected to: a real file path, <code>socket:[inode]</code>,
<code>pipe:[inode]</code>, <code>/dev/pts/N</code>, <code>anon_inode:[eventfd]</code>…
It answers, for a live process, "what does this program actually have open right
now?" — which log file it's really writing (redirections included), which
sockets it holds, whether it's leaking descriptors (thousands of entries), or
which deleted file still pins disk space (shown as <code>(deleted)</code>).
<code>lsof</code> is essentially this directory, formatted.
</details>

**Q2.** `ls -l /proc/meminfo` reports size 0, but `cat` prints screens of data.
Reconcile this — what does it reveal about what /proc files actually are?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
There is no stored content to have a size. A /proc "file" is a registered
callback: when someone calls <code>read()</code>, the kernel runs a function
that formats live internal state into text at that moment. <code>ls</code> just
shows inode metadata (where size is meaningless and reported as 0);
<code>cat</code> actually triggers the generator. Consequences: content can
change between reads, reading has (small) CPU cost, and the "file" can't be
meaningfully copied ahead of time — a backup of /proc is a photo of a speedometer.
</details>

**Q3.** A process seems hung. Using only /proc (no strace, no gdb), name three
files you'd read and what each could tell you.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<code>/proc/PID/status</code> — the State line: R (spinning on CPU?), S
(waiting), D (stuck in uninterruptible I/O — very informative!), T (someone
stopped it), Z (it's actually dead, parent hasn't reaped).
<code>/proc/PID/wchan</code> or <code>/proc/PID/stack</code> — <em>which kernel
function</em> it's sleeping in: <code>do_wait</code> (waiting for a child),
NFS/disk functions (hung mount!), futex (waiting on a lock — Phase 5).
<code>/proc/PID/fd</code> — what it has open; a socket or a file on a dead NFS
server is a classic culprit. Bonus: <code>syscall</code> shows the exact syscall
and arguments it's blocked in.
</details>

---

## Homework

Write a mini-`ps` in ~15 lines of shell: loop over every numeric directory in
/proc, and for each print the PID, state, and command name, using only `read`
of /proc files (no `ps`!). Then compare your output count with real `ps -e |
wc -l`. Do they match exactly? If not, why might they differ?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<pre>
#!/bin/bash
for d in /proc/[0-9]*; do
  pid=${d#/proc/}
  # Name and State from status (fields are tab-separated)
  name=$(awk -F'\t' '/^Name:/{print $2}' "$d/status" 2>/dev/null) || continue
  state=$(awk -F'\t' '/^State:/{print $2}' "$d/status" 2>/dev/null)
  printf "%8s  %-4s %s\n" "$pid" "${state%% *}" "$name"
done
</pre>
The <code>2>/dev/null || continue</code> matters: processes can exit between
your <code>ls</code> of /proc and your read — a photo of a moving target. Counts
differ from <code>ps -e</code> for exactly that race, and possibly because ps
shows threads/kernel-thread handling differently. Deeper lesson: /proc has no
"snapshot" semantics; every tool that walks it (including real ps) races with
the system it observes.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 5 — Interrupts and the Timer Tick →](lesson-05-interrupts-timers){: .btn .btn-primary }
