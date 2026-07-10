---
title: "Lesson 01 — What an Operating System Actually Does"
nav_order: 1
parent: "Phase 1: The Kernel Boundary"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 01: What an Operating System Actually Does

## Concept

Strip away everything and a computer is a CPU, some RAM, and devices. Any program
could, in principle, use all of it directly — and on tiny embedded systems, one
program does exactly that.

The problem starts when you want to run **more than one program**, written by
people who don't trust each other, on the same hardware. Who gets the CPU? Who
gets which part of RAM? What stops my buggy program from scribbling over yours?

The operating system's answer: one special program — the **kernel** — owns *all*
the hardware, and every other program gets only what the kernel rents out.

```
        user space (unprivileged)
 ┌──────────┐ ┌──────────┐ ┌──────────┐
 │  firefox │ │   sshd   │ │ your app │   each sees only what the
 └────┬─────┘ └────┬─────┘ └────┬─────┘   kernel chose to show it
      │            │            │
 ═════╪════════════╪════════════╪═════════  the privilege boundary
      ▼            ▼            ▼
 ┌─────────────────────────────────────┐
 │               kernel                │   owns everything:
 │  scheduler · memory · VFS · net · … │   CPU time, RAM, devices
 └───────────────┬─────────────────────┘
                 ▼
        CPU · RAM · disk · NIC
```

You have already met this boundary twice in other tracks without naming it:

- In **virtualization** Lesson 04 you saw CPU privilege rings — the kernel runs
  in ring 0, user programs in ring 3, and a hypervisor plays the same trick one
  level deeper.
- In **networking**, every `ip` command you typed was a user-space program asking
  the kernel to change *kernel-owned* state (interfaces, routes, firewall rules).

This track is about that boundary itself: what lives on each side, and how the
two sides talk.

---

## How It Works

The kernel does four jobs, and almost everything you'll learn is one of them:

**1. It multiplexes the CPU.** There are more runnable processes than CPU cores.
The kernel's *scheduler* gives each a slice and swaps them fast enough to fake
simultaneity (Phase 3). The swap is possible because the kernel can interrupt any
program at any time — we'll see the mechanism in Lesson 05.

**2. It virtualizes memory.** Every process is handed a private "pretend" address
space; the kernel plus the CPU's [MMU](https://en.wikipedia.org/wiki/Memory_management_unit)
translate pretend addresses to real RAM (Phase 4). Your bug can't scribble on my
memory because your pretend addresses simply don't map to my RAM.

**3. It mediates devices and files.** Programs never poke the disk or NIC
directly; they ask the kernel, which imposes order (filesystems, Phase 7; you saw
the network side in the whole networking track).

**4. It enforces policy.** Permissions, users, resource limits, namespaces,
capabilities — the rules about *who may do what* (Lesson 11 and Phase 12).

### Mechanism vs policy

A useful pair of words you'll meet constantly: the **mechanism** is the ability
to do something (the kernel *can* stop a process at any moment); the **policy**
is the decision about when to use it (the scheduler decides *which* process runs
next, for *how long*). Good kernel design keeps them separate — the same
mechanism (cgroups, Phase 12) serves many policies (Docker's, systemd's, yours).

### Kernel vs user space is enforced by hardware, not convention

The CPU itself has a privilege mode bit. In user mode, instructions that touch
hardware state (talk to devices, change the memory mappings, disable interrupts)
simply **trap** — the CPU refuses and jumps into the kernel instead. That's why
this isn't a gentleman's agreement: user code *physically cannot* take over,
because the dangerous instructions don't work in user mode. (This is exactly the
trap-and-emulate story from virtualization Lesson 04, one ring up.)

{: .note }
> **Monolithic vs microkernel**
> Linux is a *monolithic* kernel: the scheduler, memory manager, filesystems,
> and network stack all run inside one big privileged program. The alternative —
> a *microkernel* — keeps only the minimum in ring 0 and runs filesystems,
> drivers, etc. as user-space servers (e.g. [MINIX](https://en.wikipedia.org/wiki/MINIX),
> seL4, QNX). Monolithic won the mainstream on performance; you'll still feel
> the microkernel idea in modern designs (FUSE filesystems, userspace drivers,
> microVMs from virtualization Lesson 59).

---

## Lab

No C yet — today you just locate the boundary on your own VM.

```bash
# 1. One kernel, many processes. The kernel is NOT a process — but its
#    housekeeping threads appear in brackets in ps output:
$ ps -ef | head -8
# UID  PID  PPID ... CMD
# root   1     0 ... /sbin/init          ← first user-space process
# root   2     0 ... [kthreadd]          ← parent of all kernel threads
# root   3     2 ... [rcu_gp]            ← kernel thread (brackets!)

# 2. Which kernel is running?
$ uname -r
# 6.8.0-xx-generic

# 3. The kernel's own memory is invisible to you. Try to read it:
$ sudo head -c 16 /dev/mem
# head: error reading '/dev/mem': Operation not permitted
# (even root is blocked by default — CONFIG_STRICT_DEVMEM)

# 4. User programs "own" nothing. Prove your shell sees virtual addresses:
$ grep stack /proc/$$/maps
# 7ffd1a2b3000-7ffd1a2d4000 rw-p ... [stack]
#   ^ this address is fake/virtual — Phase 4 shows the real mapping

# 5. Feel the trap: a normal user asking for kernel-owned state changes
$ ip link set lo down
# RTNETLINK answers: Operation not permitted
#   the request REACHED the kernel; the kernel's policy said no
$ sudo ip link set lo down && sudo ip link set lo up
#   same mechanism, different credentials → allowed

# 6. Count how much of "your" system is kernel threads:
$ ps -ef | awk '$3==2 || $2==2' | wc -l
# 60-ish on a typical VM — a whole hidden workforce
```

Expected takeaways: the kernel is not in the process list as a normal process;
its memory is unreachable; every privileged action is a *request* the kernel may
refuse.

---

## Further Reading

| Topic | Link |
|---|---|
| Kernel (operating systems) | <https://en.wikipedia.org/wiki/Kernel_(operating_system)> |
| User space and kernel space | <https://en.wikipedia.org/wiki/User_space_and_kernel_space> |
| Protection ring | <https://en.wikipedia.org/wiki/Protection_ring> |
| Monolithic kernel | <https://en.wikipedia.org/wiki/Monolithic_kernel> |
| Microkernel | <https://en.wikipedia.org/wiki/Microkernel> |
| `proc(5)` man page | <https://man7.org/linux/man-pages/man5/proc.5.html> |

---

## Checkpoint

**Q1.** Why can't a user program simply read another program's memory?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because a user program never touches real RAM addresses at all. Every address it
uses is a <strong>virtual</strong> address, translated by the MMU using page
tables that the <strong>kernel</strong> controls. Another process's memory is
simply not mapped into your address space — there is no address you could use
that translates to their pages. And you can't fix that yourself, because the
instructions that modify page tables are privileged: executing them in user mode
traps into the kernel. Isolation is enforced by hardware translation + privileged
control of that translation, not by programs politely agreeing not to peek.
</details>

**Q2.** `[kthreadd]` appears in `ps` with brackets. What are kernel threads, and
why is the kernel itself still not "just another process"?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Kernel threads are workers the kernel creates for background jobs (flushing dirty
pages, RCU callbacks, etc.). They run entirely in kernel space and have no
user-space program or memory image — that's what the brackets mean. But the
kernel as a whole is not a process: it has no PID, isn't scheduled as a unit,
and doesn't "run" the way processes do. It is better thought of as a library of
privileged code that runs <em>within</em> the context of whoever entered it —
a process making a syscall, an interrupt, or one of these kernel threads.
</details>

**Q3.** In the lab, `ip link set lo down` failed as a normal user but the
command still *ran*. Where exactly was the request refused — in the `ip`
program, in the CPU, or in the kernel — and how is that different from what
would happen if `ip` tried to write to the NIC hardware directly?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The request was refused <strong>inside the kernel</strong>: <code>ip</code>
successfully made its request (a netlink message), the kernel received it,
checked the caller's credentials against its policy, and answered "Operation not
permitted." That's a <em>policy</em> refusal at the official door. If
<code>ip</code> instead tried to program the NIC hardware directly from user
space, it would never get that far: the I/O instructions / MMIO access would
trap or fault in the CPU immediately — a <em>mechanism</em> refusal. Policy says
"you may not"; the hardware privilege mechanism says "you cannot."
</details>

---

## Homework

The kernel's four jobs are multiplexing the CPU, virtualizing memory, mediating
devices/files, and enforcing policy. Take one ordinary command — `curl
https://example.com` — and list at least one way it depends on **each** of the
four jobs. Be specific (name the kernel-owned thing involved).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>CPU multiplexing:</strong> while curl waits for the network, the
scheduler runs other processes; when the reply arrives, curl is made runnable
and given CPU time again — curl never "owns" the CPU. <br>
<strong>Memory virtualization:</strong> curl's buffers, TLS state, and the
downloaded bytes live in virtual pages the kernel mapped for it; libraries like
libssl are shared read-only pages mapped into many processes at once. <br>
<strong>Device/file mediation:</strong> curl never touches the NIC. It opens a
socket (a kernel object), and the kernel's network stack — routes, ARP,
conntrack, everything from the networking track — moves the packets. Name
resolution also reads files like <code>/etc/resolv.conf</code> through the
kernel's VFS. <br>
<strong>Policy:</strong> the kernel checks that curl's user may create sockets,
applies any nftables rules to its traffic, counts the memory it allocates
against limits, and would refuse a bind to port 80. Every step of the command is
rented, checked, and revocable.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 2 — System Calls: the Only Door In →](lesson-02-syscalls-strace){: .btn .btn-primary }
