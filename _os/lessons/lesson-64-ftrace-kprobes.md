---
title: "Lesson 64 — ftrace and kprobes"
nav_order: 2
parent: "Phase 13: Tracing & Debugging"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 64: ftrace and kprobes

## Concept

perf (Lesson 63) samples — it catches the CPU *sometimes* and builds a
statistical picture. Sometimes you need the opposite: trace *every* occurrence
of a specific kernel event, with full detail. That's **ftrace** and
**kprobes** — the kernel's built-in tracing framework, and the foundation the
whole eBPF observability ecosystem is built on.

```
   perf (L63):  statistical — "where does the CPU spend time, roughly?"
   ftrace:      exhaustive — "trace EVERY call to this function, exactly"

   ftrace can:
     • trace which kernel functions run (function tracer)
     • graph the call tree with timing (function_graph)
     • fire on TRACEPOINTS (stable, curated hooks: sched_switch, block_rq,
       syscall enter/exit) — the events, named
     • fire on KPROBES (dynamic: trap ANY kernel function by name, read its
       args) — go anywhere, even where no tracepoint exists
```

You've been near this the whole course: the "which process is calling X"
questions (Lesson 04's /proc detective work, Lesson 42's "who's writing to
disk") have exact answers here. And the connection you already have: the eBPF
tools from the networking track (Phase 11 there — `bpftrace`, BCC's
`biolatency`/`opensnoop`/`execsnoop`) are *built on* kprobes and tracepoints —
they attach BPF programs to these same hooks. This lesson shows the raw
mechanism those friendly tools wrap, so you understand what they're doing and
can drop to the primitive when needed.

---

## How It Works

### tracefs: tracing as files (of course)

ftrace is controlled through `/sys/kernel/tracing/` (or debugfs) — write to
files to configure, read `trace` to get output (the "everything is a file"
theme, Lesson 04/35, reaching the tracer itself). The key files: `current_tracer`
(function / function_graph / nop), `set_ftrace_filter` (limit to specific
functions — essential, tracing *everything* floods instantly),
`available_tracers`, `tracing_on`. Nobody writes these by hand for real work —
`trace-cmd` (a CLI front-end) and, above it, `perf trace` and `bpftrace` do —
but seeing the raw files demystifies the stack.

### Tracepoints vs kprobes

- **Tracepoints** — *static, stable* hooks the kernel developers placed at
  important events (`sched:sched_switch` — every context switch, Lesson 12;
  `block:block_rq_issue` — every disk request, Lesson 42; `syscalls:sys_enter_*`
  — every syscall, Lesson 02). They have defined arguments and a stability
  guarantee (like the syscall ABI's spirit — Lesson 46), so tools built on
  them keep working. `ls /sys/kernel/tracing/events/` is the catalog.
- **Kprobes** — *dynamic* traps: insert a breakpoint at (almost) any kernel
  function's address, read its registers/arguments when hit. Total flexibility
  (trace functions no tracepoint covers) at the cost of no stability guarantee
  (the function may be renamed/inlined next release — the unstable-internal-ABI
  theme, Lesson 54). kretprobes fire on function *return* (capture return
  values, measure latency).

### The eBPF connection

Modern practice attaches **eBPF programs** to these same tracepoints/kprobes:
instead of dumping raw events (ftrace's firehose) or fixed output, a small
verified BPF program (networking Phase 11) runs *in the kernel* at each event
and does in-kernel aggregation — histograms, per-process counts, filtering —
emitting only the summary. `bpftrace` makes this a one-liner language;
BCC/libbpf-tools (`execsnoop`, `opensnoop`, `biolatency`, `tcpconnect`) are
production-grade tools built this way. This is why the observability revolution
happened: kprobes/tracepoints give the *hooks*, eBPF gives *safe, efficient
programmable reactions* to them. This lesson's raw ftrace is the ground truth;
bpftrace is how you'll actually use it.

{: .note }
> **The BCC/bpftrace toolkit is the payoff</br>**
> After this lesson, the one-liners write themselves:
> <code>execsnoop</code> (every process exec — Lesson 07),
> <code>opensnoop</code> (every file open — Lesson 36),
> <code>biolatency</code> (disk I/O latency histogram — Lesson 42),
> <code>tcpconnect</code> (every connection — networking track),
> <code>runqlat</code> (scheduler run-queue latency — Lesson 15). Each is a
> BPF program on a kprobe/tracepoint you now understand. Install
> <code>bpfcc-tools</code>/<code>bpftrace</code> and the whole course becomes
> observable live.

---

## Lab

```bash
$ sudo apt install -y trace-cmd bpftrace linux-tools-$(uname -r) 2>/dev/null | tail -1
$ TR=/sys/kernel/tracing; [ -d $TR ] || TR=/sys/kernel/debug/tracing

# ---- 1. The tracepoint catalog (curated, stable hooks) ----
$ sudo ls $TR/events/ | head                     # sched block syscalls net ...
$ sudo ls $TR/events/sched/ | head -5            # sched_switch, sched_wakeup...
$ sudo ls $TR/events/block/ | grep rq            # block_rq_issue/complete (L42)

# ---- 2. function_graph: trace the kernel executing a syscall (L02, live) ----
$ sudo trace-cmd record -p function_graph -g __x64_sys_openat \
    -F cat /etc/hostname >/dev/null 2>&1
$ sudo trace-cmd report 2>/dev/null | head -15
#   __x64_sys_openat() {                          ← the open syscall's KERNEL path
#     do_sys_openat2() {
#       getname() { ... }
#       do_filp_open() { ... path_openat ... }    ← the VFS walk (L37!)
#     }
#   }
# YOU are watching Lesson 02's syscall descend into Lesson 37's VFS. Live.
$ sudo rm -f trace.dat

# ---- 3. Who is calling unlink? (the classic "who deleted my file?") ----
$ sudo bpftrace -e 'tracepoint:syscalls:sys_enter_unlinkat { printf("%s (pid %d) unlinking %s\n", comm, pid, str(args->pathname)); }' &
$ sleep 1; touch /tmp/victim && rm /tmp/victim    # trigger it
$ sleep 1; sudo pkill bpftrace 2>/dev/null
# rm (pid NNNN) unlinking /tmp/victim  ← caught the culprit, comm+pid+path.
# This is the exact answer to Lesson 04-style "who touched my files" mysteries.

# ---- 4. The BCC toolkit: the course, made observable (one-liners) ----
$ command -v execsnoop-bpfcc >/dev/null && sudo timeout 3 execsnoop-bpfcc 2>/dev/null | head -5 || \
    sudo timeout 3 bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("exec: %s\n", str(args->filename)); }' 2>/dev/null | head -5 &
$ sleep 0.5; ls >/dev/null; date >/dev/null       # cause some execs (L07)
$ wait 2>/dev/null
# every exec on the system, live — Lesson 07's fork/exec, observed globally

# ---- 5. A latency histogram in-kernel (why eBPF beats raw ftrace) ----
$ sudo timeout 4 bpftrace -e '
    tracepoint:syscalls:sys_enter_read { @start[tid] = nsecs; }
    tracepoint:syscalls:sys_exit_read /@start[tid]/ {
        @us = hist((nsecs - @start[tid]) / 1000); delete(@start[tid]); }' 2>/dev/null | tail -12 &
$ sleep 1; cat /etc/hostname >/dev/null; dd if=/dev/zero of=/dev/null bs=1k count=1000 2>/dev/null
$ wait 2>/dev/null
# @us: [0,1) ###   [1,2) #######   ...  ← a HISTOGRAM of read() latency,
# aggregated IN THE KERNEL (not one line per event — that's the eBPF win:
# ftrace would flood; BPF summarizes at the source). L42's biolatency idea.

# ---- 6. kprobe: trap a function with NO tracepoint ----
$ sudo timeout 3 bpftrace -e 'kprobe:vfs_write { @writes[comm] = count(); }' 2>/dev/null | tail -8 &
$ sleep 1; echo test > /tmp/kp; sync
$ wait 2>/dev/null; rm -f /tmp/kp /tmp/victim
# @writes[bash]: N  @writes[systemd-journal]: M ...  ← per-process write counts
# vfs_write is an internal function; kprobe reached it dynamically (L54's
# unstable-API flexibility, used for observation instead of extension).
```

---

## Further Reading

| Topic | Link |
|---|---|
| ftrace — kernel docs | <https://www.kernel.org/doc/html/latest/trace/ftrace.html> |
| kprobes — kernel docs | <https://www.kernel.org/doc/html/latest/trace/kprobes.html> |
| `trace-cmd(1)` | <https://man7.org/linux/man-pages/man1/trace-cmd.1.html> |
| bpftrace — reference guide | <https://github.com/bpftrace/bpftrace/blob/master/man/adoc/bpftrace.adoc> |
| BCC tools | <https://github.com/iovisor/bcc> |
| Brendan Gregg — Linux tracing | <https://www.brendangregg.com/linuxperf.html> |

---

## Checkpoint

**Q1.** You want to know which process is calling `unlink` on your files.
Build the exact one-liner (tracepoint or kprobe — justify), and explain why
this succeeds where polling /proc would fail.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<code>sudo bpftrace -e 'tracepoint:syscalls:sys_enter_unlinkat { printf("%s
(pid %d, ppid %d) unlinking %s\n", comm, pid, curtask->real_parent->tgid,
str(args->pathname)); }'</code> — use the <strong>tracepoint</strong>
(<code>syscalls:sys_enter_unlinkat</code>), not a kprobe, because deletion is a
syscall with a stable, curated tracepoint that exposes the pathname argument
directly and won't break across kernel versions (Lesson 46's stability spirit)
— a kprobe on an internal function like <code>vfs_unlink</code> would also work
but is more fragile (unstable internal names — Lesson 54) and the pathname is
harder to reach there. Note it's <code>unlinkat</code> (and possibly
<code>unlink</code>/<code>rmdir</code> — cover the variants, Lesson 56's
open/openat trap applies to tracing too!). Why this beats polling /proc: the
event you're hunting is <em>instantaneous</em> — a file is unlinked in
microseconds, and the offending process may exit right after; polling
<code>/proc</code> (Lesson 04) samples state at intervals and would almost
never catch the exact moment, would miss the short-lived culprit entirely
(a script that deletes and exits between polls — the race that also plagued
Lesson 04's mini-ps homework), and couldn't attribute <em>which</em> unlink
deleted <em>which</em> file even if it caught the process running. Tracing is
<em>event-driven and exhaustive</em>: it fires synchronously <em>at</em> every
unlink, capturing the process (comm/pid — even for a process that immediately
dies), the exact path, and the moment — turning "something keeps deleting my
file and I can't catch it" (the classic unsolvable-by-polling mystery) into a
line of output naming the culprit. This is the fundamental advantage of
tracing over sampling/polling for rare or transient events: you don't hope to
observe it, you're notified every time it happens.
</details>

**Q2.** ftrace can dump every event, but the lab used bpftrace to build a
histogram *in the kernel* instead. Explain why the eBPF approach is often
essential for production tracing — what problem does dumping raw events
create?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Dumping raw events creates a <strong>firehose and overhead</strong> problem
that makes naive tracing unusable in production. A busy system makes millions
of reads, context switches, or block requests per second; ftrace writing one
record per event to the trace buffer (a) generates enormous data volume
(gigabytes/second — you can't read, store, or reason about it), (b) the buffer
overflows and drops events (losing exactly the data you need), and (c) the
tracing itself perturbs the system heavily — the cost of recording each event
(formatting, buffer writes) slows the very workload you're measuring, a
Heisenberg problem (Lesson 25's heisenbug, at the tracer). For rare events raw
dumping is fine (the unlink hunt, Q1 — few per second); for high-frequency
events it's hopeless. eBPF solves it by moving the <em>reaction</em> into the
kernel at the hook: instead of exporting every event, a small verified BPF
program runs at each event and <strong>aggregates in place</strong> — bumping a
histogram bucket, incrementing a per-process counter, filtering to only
interesting cases — so only the compact <em>summary</em> crosses to userspace
(a histogram of read latencies, not a billion individual read records; the lab
step 5). The data volume drops by orders of magnitude, no events are dropped
(aggregation keeps up), and the overhead is a few instructions per event
(BPF's verified efficiency — networking Phase 11) instead of a buffer write.
This is why eBPF/bpftrace/BCC displaced raw ftrace for most production
observability: the same kprobes and tracepoints (this lesson's hooks), but with
programmable in-kernel processing so you can trace high-frequency events on
live production systems without drowning in data or perturbing the workload —
the exact capability that made deep, always-on production observability
(biolatency, runqlat, tcplife running continuously) practical, where before you
could only afford to trace briefly and sparsely.
</details>

**Q3.** The BCC tools (execsnoop, opensnoop, biolatency, tcpconnect, runqlat)
each map to a lesson in this course. Match at least four to their subsystem
and explain what "observability" adds beyond the per-subsystem tools you
already learned (ps, iostat, ss, etc.).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Mapping: <code>execsnoop</code> → process creation (Lesson 07 fork/exec — every
program launched, with args and parent); <code>opensnoop</code> → the fd/VFS
layer (Lessons 36/37 — every file open, by which process, success/failure);
<code>biolatency</code> → the block layer (Lesson 42 — a histogram of disk I/O
latency, the per-request truth iostat's averages hide); <code>tcpconnect</code>/
<code>tcplife</code> → networking (the whole networking track — every TCP
connection's endpoints and lifetime); <code>runqlat</code> → the scheduler
(Lesson 15 — run-queue latency, how long tasks wait for a CPU, quantifying the
contention load average only hints at); <code>oomkill</code> → memory (Lesson 22),
<code>ext4slower</code> → filesystems (Lesson 38, slow ops only). What
observability adds beyond ps/iostat/ss: those tools show <strong>current
aggregate state</strong> — snapshots and averages sampled from /proc/sys (ps:
processes now; iostat: I/O rates averaged over an interval; ss: connections
now). Tracing tools show <strong>individual events as they happen, with full
attribution and distribution</strong>: (1) <em>causation and attribution</em> —
iostat says "the disk is busy," biolatency says "and here's the latency
distribution," but <code>biosnoop</code>/ext4slower says "<em>this</em> process
issued <em>this</em> slow request to <em>this</em> file" — from symptom to
culprit; (2) <em>transient and rare events</em> — ps can't catch a program that
runs for 5ms, execsnoop catches every one (Q1's polling-vs-tracing point); (3)
<em>distributions, not just averages</em> — iostat's average await hides the
p99.9 tail that's actually hurting you (Lesson 42), a histogram reveals it; (4)
<em>cross-subsystem correlation</em> — you can trace a request from tcpaccept
through the scheduler (runqlat) to disk (biolatency) to see where its latency
accrued, spanning the whole boundary this course studied. The per-subsystem
tools answer "is X healthy?"; observability answers "why is this specific thing
slow, exactly where, caused by whom?" — the difference between monitoring
(known metrics, dashboards) and debugging (arbitrary questions about live
behavior). This is the capstone's toolkit: ps/iostat/ss/PSI to notice and
localize (Lesson 15), perf to profile CPU (Lesson 63), ftrace/bpftrace to trace
any event with full detail (this lesson) — a ladder from "something's wrong" to
"here's the exact line and the exact cause," which is where the next and final
lesson takes you.
</details>

---

## Homework

Install `bpftrace` (and `bpfcc-tools` if available) and spend 20 minutes making
this course's subsystems observable live. Run at least four tools/one-liners
covering four different phases (e.g. execsnoop for Phase 2, opensnoop for
Phase 7, runqlat for Phase 3, biolatency for Phase 7's block layer,
tcpconnect for networking), and for each write one sentence: what event it
traces, which lesson's mechanism it observes, and one real problem it would
help you debug. Then reflect: how does having this toolkit change how you'd
approach "the server is slow" compared to before this course?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Example run: <code>execsnoop</code> — traces every execve (Lesson 07); observes
process creation; debugs "what's spawning all these short-lived processes?"
(a runaway cron job, a fork-storm, a mystery CPU spike from a script that
exits before ps sees it). <code>opensnoop</code> — every file open (Lessons
36/37); debugs "which config file is this program actually reading?" (Lesson
03's LS_COLORS mystery, generalized) or "what's opening files on this
disk-thrashing box?". <code>runqlat</code> — scheduler run-queue latency
(Lesson 15); observes how long tasks wait for a CPU; debugs "the box isn't at
100% CPU but everything feels sluggish" (scheduling latency/contention that
load average obscured — Lesson 15's PSI, as a distribution).
<code>biolatency</code> — disk I/O latency histogram (Lesson 42); debugs "the
disk looks fine in iostat averages but users report random slow requests" (the
p99 tail — a dying disk or occasional long queue). <code>tcpconnect</code> —
every outbound connection (networking track); debugs "what is this server
connecting to?" (unexpected external calls, a misconfigured client, a security
question). How the toolkit changes "the server is slow": before this course,
"slow" was a vague dread met with guessing, restarting things, and cargo-culted
tweaks. Now it's a <em>structured investigation with a decision tree</em>:
start broad (load average + PSI — Lesson 15 — to classify CPU vs memory vs I/O
contention), confirm the resource (vmstat r/b split, free, iostat), then drill
with the right tool — perf if CPU-bound (Lesson 63: memory-bound vs
compute-bound via IPC), bpftrace/BCC if you need to attribute events to
processes and see distributions (this lesson), gdb if a specific process is
stuck or crashed (Lesson 65 next). Each layer of tooling answers a sharper
question, and crucially you now have a <em>mental model of every subsystem</em>
(the whole course) so the numbers <em>mean</em> something — high futex time is
lock contention (Lesson 26), high major faults is memory pressure (Lesson 19),
D-state tasks is I/O waits (Lesson 08), await tail is storage (Lesson 42). "The
server is slow" stops being a panic and becomes a diagnosis you can complete in
minutes with evidence — which is exactly the methodology the final capstone
(Lesson 66) formalizes. The transformation this course delivers isn't knowing
facts about Linux; it's turning "something's wrong with the computer" into a
tractable, evidence-driven engineering problem.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 65 — Core Dumps and Crash Analysis →](lesson-65-core-dumps){: .btn .btn-primary }
