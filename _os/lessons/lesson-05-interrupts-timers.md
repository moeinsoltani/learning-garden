---
title: "Lesson 05 — Interrupts and the Timer Tick"
nav_order: 5
parent: "Phase 1: The Kernel Boundary"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 05: Interrupts and the Timer Tick

## Concept

So far the kernel only ran when a program *asked* (a syscall). That leaves a
hole in the story: if my program runs an infinite loop and never makes a
syscall, how does the kernel ever take the CPU back? And how does it notice a
packet arriving while *nothing* is asking?

The answer is the third and final way into the kernel: the **interrupt**.
Devices (and timers) can electrically signal the CPU; the CPU finishes its
current instruction, then forcibly jumps into a kernel handler — no matter what
was running.

```
 There are exactly THREE ways from user space into the kernel:

  1. syscall     "I want something"        program's choice   (Lesson 02)
  2. exception   "you did something bad"   program's fault    (page fault →
                                                               Phase 4; ÷0)
  3. interrupt   "hardware needs you"      nobody's choice —
                                           async, any time

 ┌───────────┐  IRQ: "packet!"  ┌────────────────────┐
 │    NIC    │ ───────────────▶ │ CPU: suspend user  │
 └───────────┘                  │ code, run kernel   │
 ┌───────────┐  IRQ: "tick!"    │ handler, maybe     │
 │   timer   │ ───────────────▶ │ schedule someone   │
 └───────────┘                  │ else, resume       │
                                └────────────────────┘
```

The **timer interrupt** is the kernel's heartbeat: a programmable clock chip
interrupts the CPU periodically (the *tick*), guaranteeing the scheduler runs
and no process can hold a CPU hostage. Preemptive multitasking is exactly this:
Lesson 01 promised the kernel "can interrupt any program at any time" — the
timer interrupt is how.

You've been near this before: virtualization Lesson 10's VM-exits are "interrupts
one level up" (hardware yanking control from a *guest kernel* to the hypervisor),
and networking's NAPI/softirq packet processing starts here.

---

## How It Works

### Top halves and bottom halves

Interrupt handlers run in a special context: they borrow whatever CPU/task was
running, other interrupts may be masked, and they can't sleep. So handlers are
split:

- **Top half (the hard IRQ)** — microseconds: acknowledge the device, grab the
  urgent data, schedule the rest. Counted in `/proc/interrupts`.
- **Bottom half (softirq / tasklet / workqueue)** — the deferred bulk of the
  work, run soon after with interrupts enabled. Counted in `/proc/softirqs`.
  Network RX processing (`NET_RX`) is the classic example — one you saw as
  `ksoftirqd` CPU usage during the networking track's flood tests.

### From tick to tickless

Historically the timer fired at a fixed HZ (100–1000/second) on every CPU
forever — cheap scheduling, but it wakes idle CPUs pointlessly (bad for power
and for VMs). Modern kernels are **tickless (NOHZ)**: an idle CPU programs the
timer for "wake me when the next timer is actually due" and sleeps; a busy CPU
keeps a regular tick (and `nohz_full` can remove even that for dedicated cores —
the trick low-latency and DPDK setups from networking Lesson 64 use).

Precise sleeps and timeouts (`sleep 0.001`, socket timeouts, `timerfd` from the
upcoming IPC phase) use **high-resolution timers** programmed on demand — the
tick is for scheduling housekeeping, not for every timer in the system.

{: .note }
> **Context switches vs interrupts**
> An interrupt is not a context switch: the interrupted task stays "current",
> the handler borrows it briefly, and usually the same task resumes. Only if the
> handler wakes something more important (or the tick decides the timeslice is
> over) does the scheduler then perform an actual switch — Lesson 12 measures
> exactly this pair.

---

## Lab

```bash
# 1. The interrupt scoreboard: one row per IRQ source, one column per CPU
$ head -6 /proc/interrupts
#            CPU0       CPU1
#   0:         12          0   IO-APIC   2-edge      timer
#  24:      51234          0   PCI-MSI   ...         virtio0-input.0   ← your NIC
# LOC:     823451     801234   Local timer interrupts                  ← THE tick

# 2. Watch the timer tick beat (LOC = local APIC timer):
$ grep -E 'LOC|NET_RX' /proc/interrupts; sleep 1; grep LOC /proc/interrupts
# LOC grows by ~100-1000 per CPU-second when busy; much less when idle (NOHZ!)

# 3. Generate device interrupts on demand — ping yourself from another window:
$ grep virtio /proc/interrupts        # note the count (any NIC line works)
$ ping -c 50 -i 0.01 -q localhost >/dev/null  # (lo may not hit the NIC — try
$ grep virtio /proc/interrupts        #  your eth0 IP or a download instead)
# the NIC row's counter jumped — every packet batch = an interrupt

# 4. Bottom halves have their own scoreboard:
$ cat /proc/softirqs
#          CPU0     CPU1
# TIMER:  812345   798123
# NET_RX:  51234    12345     ← packet processing deferred from hard IRQs
# SCHED:  423123   410098

# 5. An infinite loop CANNOT hold the CPU. Prove preemption works:
$ (while :; do :; done) &     # pure userspace spin, zero syscalls
$ BUSY=$!
$ sleep 2                     # if the spinner owned the CPU, this would hang...
$ echo "still alive"          # ...but the tick preempted it. still alive
$ grep -c nonvoluntary_ctxt /proc/$BUSY/status   # exists?
$ grep nonvoluntary /proc/$BUSY/status
# nonvoluntary_ctxt_switches:  200+   ← forcibly preempted, again and again
$ kill $BUSY

# 6. Who cleans up the deferred work? Meet ksoftirqd:
$ ps -e | grep ksoftirqd
#  one per CPU — when softirq load is heavy, you'll SEE these burn CPU
```

---

## Further Reading

| Topic | Link |
|---|---|
| Interrupt (Wikipedia) | <https://en.wikipedia.org/wiki/Interrupt> |
| Preemption (computing) | <https://en.wikipedia.org/wiki/Preemption_(computing)> |
| `softirqs` / bottom halves (kernel docs, Wikipedia) | <https://en.wikipedia.org/wiki/Interrupt_handler> |
| Tickless kernel | <https://en.wikipedia.org/wiki/Tickless_kernel> |
| `proc(5)` — /proc/interrupts | <https://man7.org/linux/man-pages/man5/proc.5.html> |

---

## Checkpoint

**Q1.** A program runs an infinite loop and never makes a syscall. Walk through
exactly how the kernel schedules someone else anyway.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The timer hardware raises an interrupt on that CPU (the tick). The CPU finishes
the current instruction, switches to kernel mode, and runs the timer handler
regardless of what the loop wants. The handler does scheduler bookkeeping:
charges the running task for the time used, and checks whether its timeslice is
exhausted or a higher-priority task is waiting. If so, it sets the "reschedule"
flag; on the way back toward user space the kernel notices the flag and performs
a context switch instead of resuming the loop. The loop experiences this as a
<em>nonvoluntary context switch</em> — visible in
<code>/proc/PID/status</code>, exactly as the lab showed.
</details>

**Q2.** Why are interrupt handlers split into a tiny top half and a deferred
bottom half? Use the NIC as the example.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
While a hard IRQ handler runs, that CPU can't do anything else and further
interrupts (at least that one, often more) are masked — so time in the handler
is time the whole system is deaf. Processing a packet fully (protocols,
firewall, socket delivery) takes far too long for that. So the top half only
does the minimum: acknowledge the NIC and note "work pending", microseconds.
The bulk runs later as the NET_RX softirq with interrupts re-enabled — the
system stays responsive, and packets that arrived meanwhile are batched. Heavy
load pushes this work into the per-CPU <code>ksoftirqd</code> threads — which
is why packet floods show as ksoftirqd CPU usage.
</details>

**Q3.** Your VM is idle, yet LOC (timer interrupts) still slowly increases —
but much slower than when busy. Explain both observations.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Modern kernels are tickless-idle (NOHZ): an idle CPU doesn't take the periodic
tick — it programs the timer for the next <em>actually scheduled</em> event and
sleeps, so LOC grows only when real timers (background daemons' timeouts,
kernel housekeeping) expire. That's the slow growth. When busy, the CPU keeps a
regular scheduling tick (100–1000 Hz) to enforce timeslices, so LOC climbs
fast. The counter never fully stops because some deferred work and watchdogs
still need occasional timer service.
</details>

---

## Homework

The three doors into the kernel are syscalls, exceptions, and interrupts. For
one second of an ordinary desktop system, estimate the *relative frequency* of
each (order of magnitude is fine), then verify what you can: syscalls with
`strace -c` on a busy process, interrupts via two snapshots of
`/proc/interrupts`, and for exceptions read `/proc/vmstat`'s `pgfault` counter
twice (page faults are the dominant exception — Phase 4 explains them). Which
door is busiest on your VM, and does that surprise you?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Typical idle-ish VM, per second: <strong>syscalls</strong> easily thousands to
hundreds of thousands (every process's reads/writes/polls — trace any active
daemon and multiply); <strong>exceptions</strong> mostly page faults, from
hundreds (idle) to hundreds of thousands (starting programs — every fresh page
of every new process faults in, watch <code>pgfault</code> in /proc/vmstat jump
when you run a big command); <strong>interrupts</strong> usually the smallest:
hundreds to a few thousand (ticks on busy CPUs + device batches — NICs coalesce
many packets per IRQ). Most people expect interrupts to dominate; in fact the
syscall door is by far the busiest, which is why its per-call cost (Lesson 02)
and vDSO/io_uring-style avoidance (Lessons 03/44) matter so much.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 2 — Processes (Lesson 6: Anatomy of a Process) →](lesson-06-process-anatomy){: .btn .btn-primary }
