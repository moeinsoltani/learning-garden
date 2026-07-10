---
title: "Lesson 56 — seccomp"
nav_order: 3
parent: "Phase 11: Kernel Interfaces & Security"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 56: seccomp

## Concept

Lesson 02 established the truth this lesson exploits: **the syscall list is
everything a program can do to the world.** So restricting *which* syscalls a
process may make is a powerful confinement — if a compromised process can't
call `execve`, `socket`, or `ptrace`, an attacker who owns it can't spawn a
shell, phone home, or trace other processes.

**seccomp** (secure computing) is that filter, enforced by the kernel at the
syscall boundary:

```
   process makes a syscall  ───▶  seccomp filter (a BPF program!)
                                       │  examines syscall number + args
                                       ▼
                    ┌──────────────────────────────────────┐
                    │ ALLOW    │ let it through             │
                    │ ERRNO    │ fail it (return -EPERM)    │
                    │ KILL     │ terminate the process      │
                    │ TRAP     │ deliver SIGSYS             │
                    │ NOTIFY   │ hand to a userspace monitor│
                    └──────────────────────────────────────┘
```

You've been protected by it all course without seeing it: Docker's default
profile blocks ~44 dangerous syscalls (why io_uring is blocked, Lesson 44),
systemd's `SystemCallFilter=` (Lesson 52's hardening), Chrome's renderer
sandbox, Android's app isolation, and QEMU's `--sandbox` (virt Lesson 57).
The filter is written as a **BPF program** — the same technology as the
networking track's eBPF/XDP (Phase 11 there), here classic-BPF filtering
syscalls instead of packets. It's the userspace-controllable complement to
Lesson 11's capabilities: capabilities gate *privileged* operations, seccomp
gates *any* operation by syscall — a process can drop the ability to
`execve` even as root.

---

## How It Works

### Two modes

- **Strict mode** (the original): only `read`, `write`, `_exit`, and
  `sigreturn` allowed — anything else kills the process. Draconian, used by
  pure-compute sandboxes (a process that reads input on an existing fd,
  computes, writes output — nothing else). Rarely practical.
- **Filter mode** (seccomp-BPF, the real one): you install a BPF program that
  inspects each syscall's number and *arguments* (with limits — see below)
  and returns a verdict. This is what everything modern uses.

### The BPF filter, and the once-installed-can't-relax rule

The filter runs in the kernel on every syscall, examining the `seccomp_data`
struct (syscall nr, arch, args, instruction pointer). Filters are
**one-way**: once installed you can only *add* more restrictive filters (they
stack, most-restrictive-wins), never remove — a compromised process can't
lift its own sandbox. Installing requires either privilege or
`NO_NEW_PRIVS` (Lesson 11's `NoNewPrivileges=`) — which also guarantees the
process can't regain privileges via setuid, closing an escape. Writing raw
BPF is painful, so **libseccomp** (`seccomp_rule_add`) provides a friendly
allowlist/denylist API that everything uses.

### The critical limitation: no dereferencing pointer arguments

seccomp can inspect syscall arguments *by value* (the file descriptor number,
the flags, the syscall number) but **cannot dereference pointers** — it
can't read the *path string* passed to `open`, because at filter time that's
a user pointer and following it would race (the classic TOCTOU, Lesson 25:
the process could change the string after the check, before the syscall reads
it). So seccomp filters "may this process call `open` at all / with these
flags," not "may it open *this specific file*" — path-based policy is LSM's
job (Lesson 57). This boundary between the two mechanisms is exactly why both
exist. (The NOTIFY action lets a userspace monitor do deeper, safe checks —
the modern escape hatch, used by container runtimes for fine-grained policy.)

{: .note }
> **Blocking by number is subtler than it looks**
> Syscalls have overlapping variants: block <code>open</code> but not
> <code>openat</code> and the process just uses openat (glibc already prefers
> it!). Block <code>fork</code> but not <code>clone</code>/<code>clone3</code>
> and it still spawns. Effective filters must cover <em>all</em> the ways to
> reach a capability — which is why maintained profiles (Docker's, libseccomp's
> groups like <code>@system-service</code>) are the right starting point, not
> hand-rolled lists (the homework explores this trap).

---

## Lab

```bash
$ sudo apt install -y libseccomp-dev 2>/dev/null | tail -1

# ---- 1. A process that sandboxes ITSELF, then can't escape ----
$ cat > /tmp/sandbox.c << 'EOF'
#include <seccomp.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
int main(void) {
    printf("before sandbox: I can do anything\n");
    fflush(stdout);

    scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);   /* default: KILL */
    /* allowlist only what we need to run + print + exit: */
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(brk), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(fstat), 0);
    seccomp_load(ctx);                                    /* ARMED. one-way. */

    printf("after sandbox: write() still works (allowlisted)\n");
    fflush(stdout);

    /* now try something forbidden — spawn a shell: */
    printf("attempting execve...\n"); fflush(stdout);
    execl("/bin/sh", "sh", NULL);                         /* → KILLED here */
    printf("this never prints\n");
    return 0;
}
EOF
$ gcc /tmp/sandbox.c -lseccomp -o /tmp/sandbox
$ /tmp/sandbox; echo "exit status: $?"
# before sandbox / after sandbox / attempting execve...
# Bad system call (core dumped)
# exit status: 159      ← 128 + 31 (SIGSYS): the kernel killed it AT execve.
# The process cannot un-sandbox itself: a compromise here can't spawn a shell.

# ---- 2. See it from outside with strace ----
$ strace -f /tmp/sandbox 2>&1 | tail -4
# ...write(...) = ... / execve("/bin/sh"...) = ?    ← trace shows the fatal call
# +++ killed by SIGSYS +++

# ---- 3. The number-not-name subtlety: block open, openat escapes ----
$ cat > /tmp/leaky.c << 'EOF'
#include <seccomp.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>
int main(void) {
    scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_ALLOW);   /* allow all... */
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(1), SCMP_SYS(open), 0);  /* ...but block open */
    seccomp_load(ctx);
    int a = open("/etc/hostname", O_RDONLY);
    int b = openat(AT_FDCWD, "/etc/hostname", O_RDONLY);  /* NOT blocked! */
    printf("open() = %d (blocked), openat() = %d (slipped through)\n", a, b);
    return 0;
}
EOF
$ gcc /tmp/leaky.c -lseccomp -o /tmp/leaky && /tmp/leaky
# open() = -1 (blocked), openat() = 3 (slipped through)
# blocking one variant is USELESS — the lesson every real profile learns

# ---- 4. See the profiles protecting you already ----
$ grep Seccomp /proc/self/status                  # mode of your shell (0 = off)
$ systemctl show systemd-resolved -p SystemCallFilter | fold -w76 | head -4
# a real service's syscall allowlist — the @groups libseccomp provides
$ docker run --rm alpine grep Seccomp /proc/1/status 2>/dev/null || \
    echo "(Docker applies its ~44-syscall-deny profile: Seccomp: 2 = filtered)"

# ---- 5. Argument filtering (by VALUE, not pointer) ----
$ cat > /tmp/argfilter.c << 'EOF'
#include <seccomp.h>
#include <unistd.h>
#include <stdio.h>
int main(void) {
    scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_ALLOW);
    /* allow write ONLY to fd 1 (stdout); block write to any other fd */
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(1), SCMP_SYS(write), 1,
                     SCMP_A0(SCMP_CMP_NE, 1));    /* arg0 (fd) != 1 → block */
    seccomp_load(ctx);
    write(1, "to stdout: allowed\n", 19);
    write(2, "to stderr: blocked\n", 19);         /* fails: -1 */
    return 0;
}
EOF
$ gcc /tmp/argfilter.c -lseccomp -o /tmp/argfilter && /tmp/argfilter
# to stdout: allowed          ← stderr write silently blocked (arg0 filtered)
# it can filter fd NUMBERS (by value) but NOT the path STRING open() gets
$ rm -f /tmp/sandbox* /tmp/leaky* /tmp/argfilter*
```

---

## Further Reading

| Topic | Link |
|---|---|
| `seccomp(2)` man page | <https://man7.org/linux/man-pages/man2/seccomp.2.html> |
| seccomp — kernel docs | <https://www.kernel.org/doc/html/latest/userspace-api/seccomp_filter.html> |
| libseccomp | <https://github.com/seccomp/libseccomp> |
| Seccomp (Wikipedia) | <https://en.wikipedia.org/wiki/Seccomp> |
| Docker seccomp profile | <https://docs.docker.com/engine/security/seccomp/> |
| `prctl(2)` — PR_SET_NO_NEW_PRIVS | <https://man7.org/linux/man-pages/man2/prctl.2.html> |

---

## Checkpoint

**Q1.** seccomp blocks syscalls by number. Why is blocking `open` but
allowing `openat` a useless filter?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because they reach the same capability by different numbers, and userspace
readily uses either. <code>open(path, flags)</code> and <code>openat(dirfd,
path, flags)</code> both open files; modern glibc actually implements
<code>open()</code> via the <code>openat</code> syscall already, so blocking
<code>open</code> often does nothing even to code that <em>calls</em> open,
and any program (or attacker's payload) can trivially call openat directly
(the lab proved it: open blocked, openat sailed through and opened the file).
A syscall filter must cover <em>every</em> syscall that reaches the capability
you're trying to deny — open, openat, openat2, and any future variant — or the
restriction is theater. This generalizes across the syscall table's redundancy:
fork/vfork/clone/clone3 all create processes; dup/dup2/dup3/fcntl(F_DUPFD) all
duplicate fds; the *at() family, the compat/32-bit entry points, etc. It's why
you never hand-roll a denylist (you'll miss a variant — a real
sandbox-escape pattern) and instead use maintained allowlists (Docker's
profile, libseccomp's <code>@</code>-groups) or, better, an <em>allowlist</em>
approach (deny by default, permit the specific syscalls the program needs) so
a forgotten variant fails closed rather than open — the same fail-safe
principle as firewall default-deny (networking track).
</details>

**Q2.** Why can seccomp filter a syscall's integer arguments (like the fd
number in `write`) but not the path string in `open`? What does this
limitation reveal about the boundary between seccomp and LSM (Lesson 57)?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Integer arguments are passed <em>by value</em> in registers (Lesson 02's
calling convention) — the fd number, the flags, the syscall number are right
there in the <code>seccomp_data</code> struct, fixed and safe to compare.
The path is passed as a <em>pointer</em> to a string in user memory — and
seccomp deliberately refuses to dereference it, for a decisive reason:
following a user pointer at filter time is a TOCTOU race (Lesson 25). The
process could pass a pointer to "/tmp/ok", let the filter read and approve it,
then (on another thread) change the memory to "/etc/shadow" before the syscall
itself dereferences it — a time-of-check/time-of-use gap that would make any
path-based seccomp decision unenforceable and exploitable. So seccomp is
restricted to what it can check atomically and safely: the register values.
This reveals the exact division of labor with LSM: seccomp answers "may this
process invoke this <em>operation</em> (syscall + value args) at all" —
coarse, syscall-shaped, path-blind, cheap, userspace-installable; LSM
(Lesson 57) answers "may this <em>subject</em> access this specific
<em>object</em>" — the file at this path/inode, checked <em>inside</em> the
kernel after the pointer is safely resolved and the object identified,
immune to the race. Two layers because one mechanism structurally can't do
the other's job: seccomp shrinks the syscall attack surface, LSM enforces
object-level access policy — defense in depth with each covering the other's
blind spot (which is why hardened systems, and every container, run both).
</details>

**Q3.** Docker blocks io_uring's syscalls in its default seccomp profile
(Lesson 44). Using this lesson's model, explain why io_uring is *especially*
awkward for seccomp — what does the filter lose visibility into once
io_uring is allowed?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
seccomp filters <em>syscalls</em> — it intercepts each syscall as the process
crosses the boundary. io_uring's entire design (Lesson 44) is to <em>stop
making per-operation syscalls</em>: you submit operations (read, write,
openat, connect, even execve-like ops) as entries in a shared-memory ring,
and the kernel's io_uring machinery <em>performs them on your behalf</em> —
often via kernel worker threads, sometimes with zero syscalls at all (SQPOLL).
So once <code>io_uring_setup</code>/<code>io_uring_enter</code> are allowed,
the actual I/O operations happen <em>without corresponding syscalls</em> for
seccomp to see: a process could be blocked from calling <code>openat</code>
directly, yet freely submit an openat <em>operation</em> through io_uring that
the filter never inspects — the sandbox is bypassed. io_uring effectively
opens a second, unfiltered path to much of the syscall table's functionality,
routed around the checkpoint seccomp guards. That's why the safe default is
to block io_uring wholesale (its three syscalls) rather than try to filter
what flows through it — seccomp can't, by construction. (The kernel has since
added io_uring-side restrictions — IORING_RESTRICTION, and integration work
to make LSM hooks fire on uring operations — but the mismatch is fundamental:
a batching, kernel-executes-for-you interface is opaque to a per-syscall
filter.) It's the same tension as Lesson 44's security discussion:
performance features that relocate work into the kernel's async engine
concentrate privilege and evade per-syscall observation — so they're gated
where trust is lowest (containers) and enabled where it's highest (dedicated
storage hosts).
</details>

---

## Homework

Build a minimal seccomp allowlist for a hypothetical "PDF thumbnail
generator" that must: read the input PDF (already-open fd), allocate memory,
compute, and write the output (already-open fd) — nothing else (no network,
no exec, no new files). List the syscalls you'd allow and why, then explain
two ways your allowlist could be *too tight* (breaking the program) and one
way a hand-built list could still be *too loose* (letting the attacker do
something), connecting to why real projects derive profiles empirically
(hint: strace) rather than by guessing.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Allowlist (deny-by-default, SCMP_ACT_KILL): <code>read</code>,
<code>write</code> (the I/O on existing fds — the whole job),
<code>mmap</code>/<code>munmap</code>/<code>mprotect</code>/<code>brk</code>
(memory — allocation, and mprotect because allocators and the runtime need
it), <code>exit_group</code> (terminate), <code>rt_sigreturn</code>/<code>
rt_sigprocmask</code> (signal handling the runtime installs),
<code>futex</code> (any threading/locking — Lesson 26), <code>close</code>,
<code>fstat</code>/<code>lseek</code> (fd housekeeping),
<code>clock_gettime</code>/<code>getrandom</code> (time, RNG the library may
want). Notably absent — the point of the sandbox: no <code>execve</code>/
<code>clone</code> (can't spawn — Lesson 07), no <code>socket</code>/<code>
connect</code> (can't exfiltrate), no <code>open</code>/<code>openat</code>
(can't touch new files — it only gets the two fds handed to it, the
privilege-separation pattern of Lesson 32/11). Too tight (breaks the program):
(1) forgetting <code>mprotect</code> or a memory syscall the allocator/JIT
needs — glibc malloc, the language runtime, or a PDF library's decompressor
may call syscalls you didn't anticipate, and the process dies mid-render on a
syscall you'd never guess; (2) forgetting <code>futex</code> or thread-related
calls if any dependency is internally threaded — it works in your simple test
and dies under the real library. Too loose (hand-built list leaks): allowing
<code>write</code> unconditionally lets a compromised renderer write to
<em>any</em> fd it can get — including inherited ones it shouldn't (a log
socket, the parent's pipe) — so you'd want to restrict by fd value
(SCMP_CMP, lab 5) or ensure O_CLOEXEC hygiene (Lesson 36); and allowing a
broad memory or ioctl syscall could enable more than intended. This is
exactly why real projects (systemd's filters, Docker, Chrome, Firefox's
sandbox) <strong>derive profiles empirically</strong>: run the real workload
under <code>strace -c -f</code> across many inputs to enumerate the syscalls
it <em>actually</em> makes (catching the allocator/runtime/library calls you'd
never guess — the too-tight risk), then allowlist those, then audit for
over-broad grants (the too-loose risk) — an observe-then-restrict cycle, never
a guess. The general principle: a sandbox tuned by guessing is either broken
or porous; tune it against the observed behavior of the real program, and
prefer allowlists that fail closed.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 57 — LSM: AppArmor and SELinux →](lesson-57-lsm){: .btn .btn-primary }
