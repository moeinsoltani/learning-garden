---
title: "Lesson 66 — Debugging Methodology (Capstone)"
nav_order: 4
parent: "Phase 13: Tracing & Debugging"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 66: Debugging Methodology — Capstone

## Concept

**Course capstone (Phase 13).** You've learned the subsystems (Phases 1–12) and the tools
to observe them (63–65). This final lesson assembles them into what actually
matters on the job: a **methodology** for turning "the system is
misbehaving" — vague, stressful, open-ended — into a bounded, evidence-driven
diagnosis. The knowledge is only useful if you can deploy it under pressure, in
the right order.

The core discipline, learned implicitly throughout the course and now made
explicit:

```
   1. CHARACTERIZE   what exactly is wrong? (slow? crashing? hung? wrong output?)
                     get a specific, measurable symptom — not "it's broken"
   2. LOCALIZE       which resource / layer? CPU, memory, I/O, network, or logic?
                     broad tools first: load/PSI (L15) → vmstat → the split
   3. DRILL          the right tool for that layer:
                     CPU → perf (L63)   |  events → bpftrace (L64)
                     stuck → /proc, gdb (L04, L65)  |  I/O → iostat, biolatency
   4. HYPOTHESIZE    a specific, falsifiable claim: "the DB is OOM-killed
                     because the batch job's leak crosses memory.max"
   5. TEST           confirm or refute with evidence — don't fix on a guess
   6. FIX & VERIFY   change one thing, measure that it worked, watch for
                     second-order effects
```

Two systematic frameworks make step 2 rigorous:

- **The USE method** (Brendan Gregg): for every resource (CPU, memory, disk,
  network), check **U**tilization, **S**aturation, **E**rrors. A quick sweep
  that finds the bottleneck resource before you drill.
- **A layered toolbox map**: match the symptom to the layer, and the layer to
  its tool — so you reach for the right instrument instead of flailing.

The anti-pattern this replaces: guessing, restarting things, and cargo-culting
tweaks (the "have you tried turning it off and on again" of engineering).
Evidence over guessing is the thread running through the entire course — read
/proc before restarting (Lesson 04), check PSI before tuning the scheduler
(Lesson 15), strace before assuming DNS (Lesson 03), profile before optimizing
(Lesson 63). This lesson names the discipline and gives you a runbook.

---

## How It Works

### The layered toolbox map

Match the question to the tool (every one a lesson):

| Symptom / question | First tools | Lessons |
|---|---|---|
| "Is the box overloaded?" | `uptime`, `/proc/pressure/*` (PSI), `vmstat` | 15 |
| "CPU-bound? which resource?" | USE sweep: `mpstat`, `free`, `iostat`, `sar` | 15, 42 |
| "Where's the CPU time?" | `perf stat` (IPC), `perf record -g` | 63 |
| "What is it doing? (syscalls)" | `strace -c -f`, `ltrace` | 02, 03 |
| "Which process/event?" | `bpftrace`, BCC (execsnoop/opensnoop/biolatency) | 64 |
| "Why is it stuck?" | `/proc/PID/{status,wchan,stack}`, `gdb -p` | 04, 26, 28 |
| "Why did it crash?" | `coredumpctl`, `gdb core`, `dmesg` | 65, 22 |
| "Memory: leak? pressure?" | `/proc/meminfo`, `smaps`, PSI, `dmesg | grep oom` | 18-22 |
| "I/O slow?" | `iostat -x` (await!), `biolatency`, `biosnoop` | 42 |
| "Network?" | `ss`, `tcpdump`, `tcpconnect`, the whole track | networking |

### The USE method in practice

For each resource, one pass: **Utilization** (how busy — %CPU, memory used,
disk %util-with-caveats), **Saturation** (queued work waiting — run queue >
cores, PSI, disk aqu-sz, swap activity), **Errors** (dmesg, dropped packets,
I/O errors). The resource with high saturation is usually your bottleneck —
and saturation (things *waiting*) matters more than utilization (things
*busy*), which is why PSI (Lesson 15) was such an advance.

### One thing at a time

The cardinal rule that saves you from making things worse: change **one**
variable, measure its effect, keep or revert, then the next. Multiple
simultaneous changes make it impossible to know what helped (or what your
"fix" broke) — and under incident pressure, the temptation to change five
things at once is exactly when discipline matters most.

{: .note }
> **Build your own runbook</br>**
> The homework asks you to write a personal one-page runbook: symptom → first
> three commands → decision tree. This is what senior engineers actually carry
> (in their head or a wiki) — not memorized facts, but a <em>procedure</em>
> that turns panic into process. The best debugging skill isn't knowing more;
> it's having a method you trust when the pressure is on.

---

## Lab — Three Staged Mysteries

Each mystery injects a real problem. Diagnose it with the methodology
(characterize → localize → drill → hypothesize → test) *before* reading the
solution. Don't jump to the fix — practice the *process*.

```bash
# ═══ MYSTERY 1: "The box is slow, but top shows low CPU%" ═══
$ (for i in $(seq $(nproc)); do dd if=/dev/zero of=/tmp/m1_$i bs=1M count=3000 oflag=dsync 2>/dev/null & done) 
$ sleep 3
# --- YOUR TURN: run the USE sweep before scrolling ---
$ uptime                                    # load high?
$ cat /proc/pressure/io /proc/pressure/cpu  # WHICH pressure? (L15)
$ vmstat 1 3                                 # r vs b column (L15)
$ iostat -x 1 2 2>/dev/null | grep -A3 Device | tail    # await? (L42)
# DIAGNOSIS: io pressure HIGH, cpu pressure ~0, b (blocked) > 0, high await.
# → I/O-bound, not CPU-bound. Load average was D-state I/O waits (L08/L15),
#   NOT CPU demand. Fixing the "CPU" would be optimizing the wrong resource.
$ wait; rm -f /tmp/m1_*

# ═══ MYSTERY 2: "A process is stuck — 0% CPU, not responding" ═══
$ python3 -c "import threading as t
a=t.Lock(); b=t.Lock()
def f1():
 a.acquire(); __import__('time').sleep(0.2); b.acquire()
def f2():
 b.acquire(); __import__('time').sleep(0.2); a.acquire()
x=t.Thread(target=f1); y=t.Thread(target=f2); x.start(); y.start(); x.join()" &
$ STUCK=$!; sleep 2
# --- YOUR TURN: it's using 0% CPU. Is it done, or stuck? How do you tell? ---
$ ps -o pid,state,pcpu -p $STUCK               # S state, 0% — sleeping (L08)
$ cat /proc/$STUCK/task/*/wchan 2>/dev/null; echo   # futex_wait? (L26)
$ sudo cat /proc/$STUCK/task/*/stack 2>/dev/null | grep -i futex | head -2
# DIAGNOSIS: threads parked in futex_wait, 0% CPU, no progress = DEADLOCK (L28).
# Two locks, acquired in opposite orders. gdb 'thread apply all bt' would show
# the cycle (L28's exact method). Fix: lock ordering (L28). NOT a restart-loop.
$ kill -9 $STUCK 2>/dev/null

# ═══ MYSTERY 3: "The service keeps dying" ═══
$ sudo mkdir -p /sys/fs/cgroup/m3 2>/dev/null
$ echo "80M" | sudo tee /sys/fs/cgroup/m3/memory.max >/dev/null 2>&1
$ echo 0 | sudo tee /sys/fs/cgroup/m3/memory.swap.max >/dev/null 2>&1
$ sudo bash -c 'echo $$ > /sys/fs/cgroup/m3/cgroup.procs 2>/dev/null
    python3 -c "b=bytearray()
while True: b+=bytes(2_000_000)" ' 2>/dev/null; echo "exit: $?"
# --- YOUR TURN: it exited. Crash? Killed? By whom? ---
# exit: 137                                   ← 128+9 = SIGKILL (L08 math!)
$ sudo dmesg | grep -i 'oom\|killed process' | tail -2      # the OOM report (L22)
$ cat /sys/fs/cgroup/m3/memory.events 2>/dev/null | grep oom_kill   # oom_kill 1
# DIAGNOSIS: exit 137 (SIGKILL) + dmesg OOM + memory.events = OOM-killed at the
# cgroup's memory.max (L22/L60). Not a crash (no core, no SIGSEGV) — a KILL.
# Fix: raise the limit, fix the leak, or add memory.high to degrade gracefully.
$ sudo rmdir /sys/fs/cgroup/m3 2>/dev/null

# ═══ The pattern across all three ═══
# CHARACTERIZE the symptom precisely (slow ≠ stuck ≠ dying), LOCALIZE with
# broad tools (PSI/vmstat/ps/exit-code), DRILL with the right instrument
# (iostat/wchan/dmesg), then the DIAGNOSIS names a specific lesson's fix.
# Same method, three different subsystems — that transfer IS the course.
```

---

## Further Reading

| Topic | Link |
|---|---|
| Brendan Gregg — The USE Method | <https://www.brendangregg.com/usemethod.html> |
| Systems Performance (Gregg) — the book | <https://www.brendangregg.com/systems-performance-2nd-edition-book.html> |
| Linux performance tools (the famous diagram) | <https://www.brendangregg.com/linuxperf.html> |
| Site Reliability Engineering (Google) — debugging | <https://sre.google/sre-book/effective-troubleshooting/> |
| `procfs` / diagnostic files | <https://man7.org/linux/man-pages/man5/proc.5.html> |
| Julia Evans — debugging zines | <https://wizardzines.com/> |

---

## Checkpoint

**Q1.** Write your personal one-page runbook: symptom → first three commands →
decision tree. (Compare with the model.)

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A model runbook (yours should be shaped by your systems, but this is the
skeleton):
<br><br>
<strong>ANY symptom — first 3 commands (always):</strong> <code>uptime</code>
(load vs nproc — is there demand?), <code>cat /proc/pressure/*</code> (PSI —
cpu/mem/io: <em>which</em> resource is saturated? — Lesson 15), <code>vmstat
1 3</code> (r vs b split, si/so swap, us/sy/wa CPU breakdown). These three
localize to a resource in ten seconds.
<br><br>
<strong>Decision tree from there:</strong><br>
• <em>High CPU pressure / high r / high us</em> → CPU-bound → <code>perf stat</code>
(IPC: memory-bound vs compute-bound — Lesson 63) → <code>perf record -g</code>
(which code) or <code>bpftrace</code> (which process/event — Lesson 64).<br>
• <em>High IO pressure / high b / high wa</em> → I/O-bound → <code>iostat -x</code>
(await, aqu-sz — Lesson 42) → <code>biolatency</code>/<code>biosnoop</code>
(distribution + culprit) → check the storage/dependency.<br>
• <em>High memory pressure / si+so churning / low MemAvailable</em> →
memory → <code>/proc/meminfo</code> (Dirty, Cached, Swap), <code>smaps</code>
(which process — Lesson 18), <code>dmesg | grep -i oom</code> (was something
killed? — Lesson 22).<br>
• <em>A specific process stuck (0% CPU, not responding)</em> →
<code>ps -o state,wchan</code> (S/D/T/Z? what's it sleeping in? — Lesson 08) →
<code>/proc/PID/stack</code>, <code>gdb -p</code> <code>thread apply all bt</code>
(deadlock? blocked on I/O? — Lessons 26/28).<br>
• <em>A process died</em> → exit code (137=SIGKILL/OOM, 139=SIGSEGV, 134=SIGABRT
— Lesson 08 arithmetic) → <code>dmesg</code>, <code>coredumpctl gdb</code>
(Lesson 65).<br>
• <em>Network</em> → <code>ss</code>, <code>tcpdump</code>, the networking
track's tools.
<br><br>
<strong>Then always:</strong> form a specific falsifiable hypothesis, test it
with evidence before fixing, change <em>one</em> thing, verify it worked, watch
for second-order effects. The runbook's value isn't the specific commands (you
know them) — it's having a <em>fixed starting procedure</em> so that under
incident pressure you follow a method instead of guessing. The first three
commands are the most important line: they turn "everything is on fire" into
"it's an I/O problem" in seconds, and everything after is drilling into a known
resource. Senior engineers aren't faster because they know more commands;
they're faster because they don't waste the first ten minutes flailing — they
have this tree, and they trust it.
</details>

**Q2.** Across the three lab mysteries, the same methodology localized three
completely different problems (I/O saturation, deadlock, OOM kill). Explain why
"characterize precisely" (step 1) is the step most people skip — and the
specific way skipping it wastes time in each mystery.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
People skip precise characterization because the symptom feels obvious and the
pressure to <em>act</em> is high — "the box is slow, let me tune/restart
something" feels productive, while stopping to define "slow how, exactly?"
feels like delay. But the three mysteries all present as vague "it's broken,"
and the vague version sends you the wrong way: <strong>Mystery 1</strong>
("slow") — the instinct is "high load = need more CPU," so you'd profile CPU,
add cores, or optimize code — all wasted, because characterizing precisely
(load is high but CPU% is low, tasks are in D-state waiting) reveals it's
I/O-bound: every minute on CPU tuning is a minute not spent on the actual disk
bottleneck, and worse, you might "fix" it by adding CPU capacity that changes
nothing. <strong>Mystery 2</strong> ("stuck/not responding") — the instinct is
to restart it (the universal non-diagnosis), which "works" (the deadlock
clears) but guarantees recurrence because you never found the lock-ordering bug
(Lesson 28); characterizing precisely (0% CPU + futex_wait + no progress =
deadlock, not a slow computation or a hang-on-input) points at the code fix
instead of a restart-loop that masks the bug until it bites in production at
2am. <strong>Mystery 3</strong> ("keeps dying") — the instinct is to assume a
crash and hunt for a bug in the code, maybe attach a debugger looking for a
segfault; but characterizing precisely (exit 137 = SIGKILL, no core dumped,
dmesg OOM report) reveals it wasn't a crash at all — the kernel <em>killed</em>
it for exceeding memory.max (Lessons 22/60), so the fix is memory limits/leak,
not a code-logic bug you'd waste hours failing to find (there's no segfault to
find). In each case, the imprecise symptom ("slow"/"stuck"/"dying") maps to
multiple root causes, and only precise characterization — one measurable,
specific observation (which pressure? which state? which exit code?) — selects
the right branch of the decision tree. Skipping step 1 doesn't save the time it
seems to; it spends far more time drilling into the wrong resource, applying
fixes that don't address the cause, and "solving" the problem in ways that make
it recur. The discipline "define the symptom measurably before acting" is
exactly what separates a ten-minute diagnosis from an afternoon of flailing —
and it's the hardest to follow precisely when the pressure to act is highest,
which is why it must be a <em>habit</em> (the runbook's first step), not a
decision you make in the moment.
</details>

**Q3.** This capstone claims the whole course becomes "a diagnostic
vocabulary." Pick any three subsystems you studied and show how a debugging
observation in each is only meaningful *because* you understand the mechanism.
Then state what you'd tell someone about why they should learn OS internals
even if they "just write application code."

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Three observations that are meaningless without the mechanism: (1) <strong>High
<em>voluntary</em> context switches + low CPU + slow throughput</strong> (Lesson
12) — to someone who doesn't know context switches, this is just numbers; to
you it means threads are repeatedly blocking (on locks, I/O), and combined with
<code>wchan = futex_wait</code> (Lesson 26) it's <em>lock contention</em>, with
a specific fix (shrink critical sections, shard the lock — Lesson 26). The
numbers only <em>mean</em> something because you know what a context switch is
and why voluntary vs nonvoluntary matters. (2) <strong>Major faults climbing
while minor faults are flat</strong> (Lesson 19) — meaningless as raw counters,
but knowing demand paging tells you major = disk in the memory path
(page cache misses, or swapping — Lesson 21), so a rising majflt is memory
pressure reaching storage, a completely different problem from high minflt
(normal workload materializing memory) — same "faults are high" surface,
opposite diagnoses, distinguishable only by understanding the fault mechanism.
(3) <strong>%util 100% on an NVMe with await 0.3ms</strong> (Lesson 42) — the
naive read is "disk is saturated!"; knowing %util is a legacy serial-device
metric that's meaningless on parallel NVMe (Lesson 42) tells you to ignore it
and read await/aqu-sz, which say the disk is <em>fine</em> — the mechanism
knowledge stops you from chasing a non-problem. In every case, the debugging
tool <em>shows</em> the same to everyone; only the mechanism knowledge turns
the observation into a diagnosis. Why learn OS internals even for "just
application code": your application <em>runs on</em> these subsystems, and its
real-world behavior — the parts that page you at 3am — is dominated by them,
not by your business logic. Your service is slow because of page-cache misses
(Lesson 20), GC pauses interacting with the scheduler (Lesson 13), lock
contention (Lesson 26), fsync latency (Lesson 41), an OOM kill (Lesson 22),
TLB misses on a big hash table (Lesson 17), or a noisy neighbor's I/O
(Lesson 42) — none of which are visible or fixable from inside the application
abstraction. Without OS internals, these present as inexplicable, non-
reproducible "the computer is being weird," met with guessing and restarting;
with them, they're diagnosable, specific engineering problems with known fixes.
The abstraction (a language runtime, a framework) is a leaky one — it works
until it doesn't, and when it doesn't, the leak is always into the layers this
course covered. Application developers who understand the OS write faster,
more reliable software (they know why the second run is faster — Lesson 20 —
and design for it; they know threads-per-connection doesn't scale — Lesson 24 —
and choose an event loop), debug production incidents in minutes instead of
days, and stop being helpless when the abstraction leaks. "Just application
code" runs on an operating system; understanding it is the difference between
being at the mercy of the machine and being in command of it — which, from
Lesson 01's "the kernel rents you the hardware" to this final diagnosis, is
what the whole course was for.
</details>

---

## Homework

The real capstone homework is open-ended and ongoing: **apply the methodology
to a genuine problem on your own systems.** Next time something is slow, stuck,
or dying, resist the urge to guess or restart — instead run your runbook (Q1),
write down the characterize→localize→drill→hypothesize→test→fix steps as you
go, and note which lesson's mechanism each observation drew on. Do this for
three real incidents over the coming weeks. For a self-contained version now:
pick one of the three lab mysteries, extend it into a harder variant (e.g.
combine two — an OOM kill *caused by* an I/O stall backing up memory), and
write the full diagnostic narrative.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
There's no single model answer — the deliverable is the <em>habit</em>. A
strong response demonstrates the method's discipline on a real case: a precise
symptom ("p99 latency jumped from 20ms to 400ms at 14:30, p50 unchanged" — not
"the API got slow"), a localization pass that names the resource with evidence
(PSI/vmstat/USE sweep showed io pressure rising, not cpu — Lesson 15), a drill
that attributes it (biosnoop showed the batch-export job issuing large
sequential reads that evicted the hot page cache — Lessons 20/42, so the API's
formerly-cached reads became disk reads — the p99-not-p50 signature of a cache
working-set problem), a falsifiable hypothesis ("the export job's cache
eviction is causing the API's read latency"), a test (pause the export → p99
recovers; or check page-cache residency of the hot files before/after —
Lesson 20's fincore), and a fix with verification and second-order awareness
(cgroup io.max + memory.low to protect the API's cache from the batch job —
Lessons 60/22 — verified p99 stays flat under export load, with the noted
trade-off that the export now runs slower). The combined-mystery variant
teaches the real-world truth that incidents are rarely one clean resource:
memory pressure and I/O interact (reclaim writes dirty pages → I/O; I/O stalls
back up memory), and the methodology handles it precisely because each step
localizes with evidence rather than assuming — you follow the causal chain
(high latency → cache misses → why? → eviction → by whom? → the batch job →
why does that hurt? → shared page cache, no isolation) instead of guessing at
any link. Keep the narratives; re-reading your own past diagnoses is how the
runbook improves and how the method becomes automatic. The habit — not any
fact — is what you take from this course into every system you'll ever operate.
</details>

---

## Course Complete

You've gone from "what is an operating system" (Lesson 01) to building a
container by hand (Lesson 62) and diagnosing live systems with the kernel's own
tools (this phase). The 66 lessons were one continuous argument: the kernel is
a privileged program that multiplexes the CPU, virtualizes memory, mediates
devices, and enforces policy — and everything else (processes, scheduling,
memory management, concurrency, IPC, filesystems, async I/O, linking, boot,
security, containers, tracing) is that argument elaborated. You now have both
the mental model of every subsystem and the diagnostic method to apply it.

Where to go next: kernel development (Lessons 54–58 were your on-ramp — pick a
subsystem and read its source, now that you know what it does), performance
engineering (Brendan Gregg's *Systems Performance*), or simply applying this
lens to your own systems — because the best way to keep this knowledge is to
use it: next time something is slow, stuck, or dying, run the runbook.

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Operating Systems learning plan →]({{ '/os/learning-plan.html' | relative_url }}){: .btn .btn-primary }
