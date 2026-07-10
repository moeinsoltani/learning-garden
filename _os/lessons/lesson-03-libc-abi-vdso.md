---
title: "Lesson 03 — libc, the ABI, and the vDSO"
nav_order: 3
parent: "Phase 1: The Kernel Boundary"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 03: libc, the ABI, and the vDSO

## Concept

Lesson 02 showed the raw door: registers, numbers, the `syscall` instruction.
But you have never written that in a program. Between your code and the kernel
sits a translator that almost every program on the system shares: the **C
library**, on most distros [glibc](https://en.wikipedia.org/wiki/Glibc).

```
  your program        python, go, rust…   (language runtimes)
       │                     │
       ▼                     ▼
 ┌───────────────────────────────────────┐
 │            libc  (glibc)              │  wrappers: open(), read()…
 │  + buffering, malloc, strings, DNS…   │  plus real functionality
 └──────────────────┬────────────────────┘
                    │ syscall instruction  (the ABI contract)
 ═══════════════════╪══════════════════════
                    ▼
                 kernel          ┌─────────────┐
                    ▲            │    vDSO      │ ← kernel code mapped INTO
                    └────────────│ clock_gettime│   your process, called
                     no trap!    └─────────────┘   without any trap
```

libc plays three roles, and confusing them causes real debugging pain:

1. **Thin wrappers** — `read()`, `write()`, `openat()`: almost 1:1 with
   syscalls; they set up registers, run `syscall`, convert `-errno` to the
   `errno` variable.
2. **Fat functionality** — `printf` (buffering, Lesson 02), `malloc` (a whole
   memory allocator, Phase 4), `getaddrinfo` (a whole DNS client). One libc call
   here may make many syscalls — or none.
3. **The process's plumbing** — program startup before `main()`, thread
   creation, the dynamic loader's partner (Phase 9).

And then there's the cheat code: the **vDSO** (virtual dynamic shared object) —
a tiny library the *kernel itself* maps into every process, so that a few
hot, read-only "syscalls" like `clock_gettime` can run **without crossing the
boundary at all**.

---

## How It Works

### The ABI: the contract that never breaks

The [ABI](https://en.wikipedia.org/wiki/Application_binary_interface) (Application
Binary Interface) is the *binary-level* contract: syscall numbers, which
registers carry arguments, structure layouts, signal behavior. Linux's promise —
"we do not break user space" — is a promise about the ABI: a binary compiled in
2005 still runs, because syscall 1 still means `write` with the same register
convention. Compare: the kernel's *internal* API (what modules use, Lesson 54)
changes constantly. The stable surface is exactly the syscall ABI.

### ltrace vs strace

Two tracers, two layers:

- `strace` shows **syscalls** — the kernel boundary.
- [`ltrace`](https://man7.org/linux/man-pages/man1/ltrace.1.html) shows **library
  calls** — your program's calls into libc (and other .so files).

One `printf` = one ltrace line, zero-or-one strace lines (buffering!). One
`getaddrinfo` = one ltrace line, dozens of strace lines. Seeing both layers at
once is often the fastest way to understand an unfamiliar binary.

### The vDSO: syscalls without syscalls

Some "syscalls" are pure reads of kernel data: what time is it? which CPU am I
on? Trapping into the kernel just to read a counter is wasteful — programs call
`clock_gettime` *millions* of times per second (every log line, every timeout
check). So the kernel maps a small code+data page into every process — the
[vDSO](https://man7.org/linux/man-pages/man7/vdso.7.html) — where these
functions run entirely in user mode, reading a data page the kernel keeps
updated. Result: ~5 ns instead of ~200 ns, and **they never appear in strace**,
which confuses people who expect to see them.

{: .note }
> **musl, and "static" binaries**
> glibc isn't the only libc: Alpine Linux (and many containers) use
> [musl](https://en.wikipedia.org/wiki/Musl), and Go programs mostly bypass libc
> and make raw syscalls themselves. This is why a glibc-linked binary won't run
> on Alpine, and why a Go binary happily runs in a `FROM scratch` container with
> no libc at all. The kernel doesn't care — the ABI is the only real contract.

---

## Lab

```bash
# 1. Which libc, and it's a program too:
$ ldd /bin/cat | grep libc
#  libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
$ /lib/x86_64-linux-gnu/libc.so.6 | head -1
#  GNU C Library (Ubuntu GLIBC 2.39-...) stable release version 2.39

# 2. Same program, two layers. Library calls:
$ ltrace -e 'printf+puts' sh -c '/usr/bin/printf hello' 2>&1 | head
# vs syscalls:
$ strace -e trace=write sh -c '/usr/bin/printf hello' 2>&1 | head
# one library call ends as one write — now try a FAT libc function:

# 3. getaddrinfo: one call, a whole investigation underneath
$ ltrace -e getaddrinfo getent hosts example.com 2>&1 | tail -3
#  getaddrinfo("example.com", ...)                    ← ONE library call
$ strace -f getent hosts example.com 2>&1 | grep -cE 'openat|connect|sendto'
#  ~15+ syscalls: opens nsswitch.conf, resolv.conf, connects to DNS...

# 4. The vDSO is invisible to strace. Prove it:
$ cat > /tmp/clock.c << 'EOF'
#include <time.h>
int main() {
    struct timespec ts;
    for (int i = 0; i < 1000000; i++)
        clock_gettime(CLOCK_MONOTONIC, &ts);
    return 0;
}
EOF
$ gcc -O2 -o /tmp/clock /tmp/clock.c
$ strace -c /tmp/clock 2>&1 | grep -c clock_gettime
# 0        ← one MILLION calls, zero syscalls. The vDSO handled them all.
$ time /tmp/clock
# real ~0m0.02s — about 15-20 ns per call including loop overhead

# 5. See the vDSO mapped into a process (no file on disk backs it):
$ grep -E 'vdso|vvar' /proc/self/maps
# 7fff...-7fff... r-xp ... [vdso]
# 7fff...-7fff... r--p ... [vvar]     ← the kernel-updated data page

# 6. Force the slow path for comparison — CLOCK_BOOTTIME often bypasses vDSO:
$ sed -i 's/CLOCK_MONOTONIC/CLOCK_BOOTTIME/' /tmp/clock.c && gcc -O2 -o /tmp/clock /tmp/clock.c
$ strace -c /tmp/clock 2>&1 | grep clock_gettime
# 1000000 calls — now they're real syscalls; time it and compare
```

Clean up: `rm /tmp/clock /tmp/clock.c`.

---

## Further Reading

| Topic | Link |
|---|---|
| glibc | <https://en.wikipedia.org/wiki/Glibc> |
| Application binary interface | <https://en.wikipedia.org/wiki/Application_binary_interface> |
| `vdso(7)` man page | <https://man7.org/linux/man-pages/man7/vdso.7.html> |
| `ltrace(1)` man page | <https://man7.org/linux/man-pages/man1/ltrace.1.html> |
| `time(7)` — clocks overview | <https://man7.org/linux/man-pages/man7/time.7.html> |
| musl libc | <https://en.wikipedia.org/wiki/Musl> |

---

## Checkpoint

**Q1.** Why does `clock_gettime` often not appear in strace output, even though
the program calls it constantly?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because it usually never becomes a syscall. The kernel maps the vDSO into every
process — a page of kernel-provided code plus a data page the kernel keeps
updated with the current time. libc's <code>clock_gettime</code> calls the vDSO
version, which just reads that shared data in user mode: no trap, no mode
switch, nothing for strace (which watches the syscall boundary via ptrace) to
see. Only clocks the vDSO can't serve fall back to a real syscall.
</details>

**Q2.** Your service logs show DNS problems. A teammate straces it and says
"there's no `getaddrinfo` syscall anywhere — DNS must not be the issue." What's
wrong with that reasoning, and what would you look for instead?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<code>getaddrinfo</code> is a <em>libc function</em>, not a syscall — it will
never appear in strace by name. Underneath, it produces ordinary syscalls:
<code>openat</code> of <code>/etc/nsswitch.conf</code>, <code>/etc/hosts</code>,
<code>/etc/resolv.conf</code>, then socket calls (<code>socket</code>,
<code>connect</code>, <code>sendto</code>/<code>recvfrom</code> to port 53, or
a connect to systemd-resolved's stub at 127.0.0.53). So look for those — or run
<code>ltrace</code> to see the getaddrinfo call itself. Knowing which layer each
tool watches is the whole trick.
</details>

**Q3.** A binary compiled on Ubuntu crashes instantly on Alpine Linux with
"not found" errors, yet a Go binary from the same machine runs fine. Explain
both behaviors with this lesson's concepts.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The Ubuntu binary is dynamically linked against <strong>glibc</strong>; Alpine
ships <strong>musl</strong> instead, so the dynamic loader can't find
<code>libc.so.6</code> (the "not found" is about the missing library/loader,
not the binary). The Go binary doesn't need any libc: Go's runtime makes raw
syscalls directly against the kernel ABI, which is identical on both distros —
"we do not break user space" is a kernel promise, and it's the only contract the
Go binary relies on. libc choice is a <em>distro</em> decision; the syscall ABI
is a <em>kernel</em> guarantee.
</details>

---

## Homework

`ldd /bin/true` shows it needs libc, and `strace -c true` (Lesson 02) showed ~25
syscalls at startup. Build a true that needs neither: write a C program that
makes the `exit` syscall directly — without libc. Compile with
`gcc -static -nostdlib -o /tmp/tinytrue tiny.c` where the program defines
`_start` (not `main`) and uses inline assembly:
`asm("mov $60, %rax; xor %rdi, %rdi; syscall");`
Then compare `strace ./tinytrue` and its file size with `/bin/true`. What do you
observe?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The whole program:
<pre>
void _start(void) {
    asm("mov $60, %rax; xor %rdi, %rdi; syscall");
}
</pre>
strace shows just <strong>two</strong> lines: <code>execve(...)</code> and
<code>exit(0)</code> — versus ~25 syscalls for /bin/true, all of which were
glibc/loader startup (mapping libc, setting up TLS, probing locales…). The
binary is a few KB and fully static; no <code>ldd</code> dependencies at all.
Lesson: the kernel only requires <code>execve</code> to work and the ABI to be
spoken — everything else in "program startup" is userspace convention supplied
by libc. (Syscall 60 is <code>exit</code> on x86-64; register convention from
Lesson 02.)
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 4 — /proc and /sys: the Kernel's Dashboard →](lesson-04-proc-sys){: .btn .btn-primary }
