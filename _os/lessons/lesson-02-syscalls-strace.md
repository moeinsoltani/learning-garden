---
title: "Lesson 02 — System Calls: the Only Door In"
nav_order: 2
parent: "Phase 1: The Kernel Boundary"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 02: System Calls — the Only Door In

## Concept

Lesson 01 established a wall between user space and the kernel. But programs
obviously *do* get things from the kernel — they open files, send packets, start
processes. How do requests cross the wall?

Through exactly one door: the **system call** (syscall). A syscall is a special
CPU instruction that deliberately triggers the trap from Lesson 01 — it switches
the CPU to kernel mode and jumps to a fixed, kernel-chosen address. The program
can't choose *where* it lands; it can only choose *which service* it's asking
for (a number) and the arguments.

```
 user space                          kernel space
 ──────────                          ────────────
 open("data.txt", O_RDONLY)
        │
        │  put syscall number (257=openat) + args in registers
        ▼
   syscall instruction  ═══ CPU switches to kernel mode ═══▶  fixed entry point
                                                                   │
                                                              dispatch by number
                                                                   │
                                                              do_sys_openat2()
                                                              checks permissions,
                                                              builds fd, or fails
                                                                   │
   fd = 3   ◀══ CPU switches back to user mode ══════════════ return value
```

Think of it as a bank counter: you can't walk into the vault; you fill in a
numbered form at the counter, slide it across, and wait. The teller (kernel)
decides. **Everything** your programs ever do to the outside world — every file,
packet, pixel, and child process — went through this counter.

The tool that lets you watch the counter is [`strace`](https://man7.org/linux/man-pages/man1/strace.1.html),
and it will be your companion for the whole track, exactly like tcpdump in the
networking track.

---

## How It Works

Each syscall has a **number** (on x86-64: `read`=0, `write`=1, `openat`=257 …).
The calling convention is fixed by the ABI: number in register `rax`, arguments
in `rdi, rsi, rdx, r10, r8, r9`, then execute the `syscall` instruction. The
kernel looks the number up in its **syscall table** and calls the handler.

Return values are also just a register: `>= 0` means success, and a small
negative value `-errno` means failure — libc turns that into the `errno` your C
programs check and the messages like `EACCES (Permission denied)` you see in
strace.

Two things to internalize:

1. **A syscall is expensive compared to a function call** — a mode switch,
   register saves, mitigation flushes; roughly hundreds of nanoseconds vs ~1 ns
   for a call. That's why buffered I/O exists (see the lab) and why Lesson 44's
   io_uring tries to batch syscalls away.
2. **The syscall list is the kernel's true API.** There are ~350 of them, stable
   for decades ("we do not break user space" — Torvalds). Everything else (glibc,
   Python, Go) is wrappers around this list.

{: .note }
> **strace works via ptrace**
> strace uses the [`ptrace(2)`](https://man7.org/linux/man-pages/man2/ptrace.2.html)
> facility to have the kernel stop the target at every syscall entry/exit and let
> strace read its registers. That's also (part of) how gdb and debuggers work.
> It slows the target dramatically — fine for learning and debugging, wrong for
> measuring performance (use `perf`, Lesson 63).

---

## Lab

```bash
# strace ships in the lab-setup kit. First: the hello world of tracing.
$ strace -c true
# % time     seconds  usecs/call     calls    errors syscall
# ------ ----------- ----------- --------- --------- ----------------
#  ... execve, brk, mmap, openat, read, close, ...
# even the do-nothing program makes ~25 syscalls just to start and stop!

# 1. Watch cat read a file. Filter to the interesting ones:
$ echo hi > /tmp/f
$ strace -e trace=openat,read,write,close cat /tmp/f
# openat(AT_FDCWD, "/tmp/f", O_RDONLY)    = 3
# read(3, "hi\n", 131072)                 = 3
# write(1, "hi\n", 3)                     = 3      ← hi appears on screen HERE
# read(3, "", 131072)                     = 0      ← 0 bytes = end of file
# close(3)                                = 3? no — = 0
# Note the pattern: open→fd 3, read until 0, write to fd 1 (stdout), close.

# 2. Errors are return values, not exceptions:
$ strace -e trace=openat cat /nonexistent 2>&1 | grep nonexist
# openat(AT_FDCWD, "/nonexistent", O_RDONLY) = -1 ENOENT (No such file or directory)

# 3. Which syscalls dominate a real command? Summarize:
$ strace -c ls /usr/bin > /dev/null
# look at the calls column: getdents64 (directory entries!), write, openat...

# 4. See the cost of unbuffered I/O. dd can write 1 MiB in single bytes:
$ strace -c dd if=/dev/zero of=/tmp/out bs=1 count=100000 2>&1 | tail -6
#   ~100000 read + ~100000 write syscalls — and dd takes visible time
$ strace -c dd if=/dev/zero of=/tmp/out bs=100000 count=1 2>&1 | tail -6
#   1 read + 1 write for the same data. Syscalls have a price; batching pays.

# 5. Follow a program that starts children (-f) and watch execve:
$ strace -f -e trace=execve sh -c 'ls | wc -l' 2>&1 | grep execve
# execve("/usr/bin/sh", ...) then execve("/usr/bin/ls", ...) and execve("/usr/bin/wc", ...)
# (fork/clone comes in Lesson 07 — spot it in the full output if curious)

# 6. Attach to a RUNNING process (this is the real superpower):
$ sleep 300 &
$ strace -p $! -e trace=all
# strace: Process ... attached
# restart_syscall(...          ← it's just sitting in a syscall, waiting
# Ctrl-C strace (the sleep survives), then: kill %1
```

Clean up: `rm /tmp/f /tmp/out`.

---

## Further Reading

| Topic | Link |
|---|---|
| `strace(1)` man page | <https://man7.org/linux/man-pages/man1/strace.1.html> |
| `syscalls(2)` — the full list | <https://man7.org/linux/man-pages/man2/syscalls.2.html> |
| System call (Wikipedia) | <https://en.wikipedia.org/wiki/System_call> |
| `errno(3)` man page | <https://man7.org/linux/man-pages/man3/errno.3.html> |
| `ptrace(2)` man page | <https://man7.org/linux/man-pages/man2/ptrace.2.html> |

---

## Checkpoint

**Q1.** `printf("hello\n")` is not a syscall. What syscall does it eventually
make, and why might that syscall happen much later than the `printf` — or never?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Eventually it makes <code>write(1, "hello\n", 6)</code> — writing bytes to file
descriptor 1 (stdout). But printf goes through libc's <em>userspace buffer</em>
first: to amortize the syscall cost, libc collects output and flushes it later —
when the buffer fills, on <code>fflush</code>, at exit, or (for terminals) at
each newline. If stdout is redirected to a file, it's fully buffered, so the
write can happen thousands of printfs later — and if the process crashes before
flushing, <strong>never</strong>: the output is lost even though printf
"succeeded". This is why crashed programs sometimes lose their last log lines.
</details>

**Q2.** In the lab, `cat` read the file with `read(3, ..., 131072) = 3` and then
called `read` again, getting `= 0`. Why does cat need the second read — why not
stop after the first?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because a short read doesn't mean end-of-file. <code>read</code> returning 3
means "I gave you 3 bytes" — there could be more coming (pipes and sockets
return whatever is available right now; even files can return less than asked).
The only reliable end-of-file signal is a return of <strong>0</strong>. So the
standard reading loop is: call read until it returns 0 (EOF) or -1 (error).
cat's second read is that loop doing its job.
</details>

**Q3.** A colleague says: "My program is slow, and strace shows millions of tiny
`write` calls. But syscalls are just function calls into the kernel, right?"
Correct the misconception, and name the fix (which the lab demonstrated).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A syscall is much more expensive than a function call: it's a CPU mode switch —
save user registers, switch to the kernel stack and address-space view, run
kernel entry code (including speculative-execution mitigations), dispatch, and
switch back. Hundreds of nanoseconds each, vs ~a nanosecond for a call. A
million one-byte writes pays that toll a million times for the same bytes that
one buffered write pays once. The fix is batching: buffer in user space and
write in large chunks — exactly the dd bs=1 vs bs=100000 experiment, and exactly
what libc's printf buffering (Q1) exists to do.
</details>

---

## Homework

Use strace to answer a question you couldn't answer by reading docs: **when you
run `ls --color=auto`, how does ls decide what colors to use — where does the
configuration come from?** Trace it (`strace -e trace=openat,read ls
--color=auto`), identify the environment/config sources it touches, and write
two sentences on what you found. Hint: also compare with `LS_COLORS= strace -e
trace=openat ls --color=auto`.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
strace shows that ls does <strong>not</strong> open any color config file by
default: the color database arrives via the <code>LS_COLORS</code> environment
variable (visible in the <code>execve</code> call's environment, set up by your
shell from <code>dircolors</code>). With LS_COLORS present, ls just reads the
directory (<code>openat(".", O_RDONLY|O_DIRECTORY)</code>, <code>getdents64</code>)
and stats entries to classify them. With <code>LS_COLORS=</code> emptied, ls
falls back to built-in defaults — still no config file opened. Moral: strace
reveals the <em>actual</em> inputs of a program — files it opens, variables it
receives — which is exactly how you debug "where does this setting come from?"
mysteries in real systems.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 3 — libc, the ABI, and the vDSO →](lesson-03-libc-abi-vdso){: .btn .btn-primary }
