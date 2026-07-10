---
title: "Lesson 07 — fork and exec"
nav_order: 2
parent: "Phase 2: Processes"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 07: fork and exec

## Concept

How does a new process come to exist? Unix's answer is famously strange: you
don't "create a process running program X". Instead:

1. [`fork()`](https://man7.org/linux/man-pages/man2/fork.2.html) — the calling
   process **clones itself**. Now two nearly identical processes run the same
   code at the same point; one is the parent, one the child.
2. [`execve()`](https://man7.org/linux/man-pages/man2/execve.2.html) — the child
   **replaces itself** with program X: new code, new memory — same process shell
   (same PID, same open fds, same credentials).

```
   shell (PID 100)
        │ fork()
        ├──────────────────────┐
        │                      │
   shell (PID 100)        shell (PID 101)      ← identical clone
   fork returned 101      fork returned 0
   (parent: waits)             │ execve("/bin/ls")
                               │
                          ls (PID 101)          ← same process, new program
                               │ ...runs, exits
```

Why two steps? Because **the gap between fork and exec is where all setup
happens**. The child — still running the parent's code — can redirect its own
stdout to a file, cd somewhere, drop privileges, join a namespace… and *then*
exec. That's how the shell implements `ls > out.txt 2>&1` without any special
kernel support for redirection: the child rewires its own fds (Lesson 36), then
execs `ls`, which inherits them, blissfully unaware. Docker does the same dance
with namespaces before exec'ing your entrypoint.

---

## How It Works

### fork is (almost) free: copy-on-write

Cloning a 2 GiB process sounds insanely expensive. It isn't, because nothing is
copied: parent and child share all memory pages, marked read-only. Only when
either side *writes* a page does the CPU fault (Lesson 05's exceptions!) and the
kernel copies that one page. This is **copy-on-write (CoW)** — the same trick as
qcow2 snapshots (virtualization Lesson 23) applied to RAM. A fork that
immediately execs touches almost nothing, so almost nothing is ever copied.

What the child gets: a copy (CoW) of memory, a **copy of the fd table** (same
open files! — this is deep, Lesson 36), same credentials, cwd, signal handlers,
environment. What it doesn't: the PID (new), pending signals, file locks,
timers.

### exec replaces everything — except what it deliberately keeps

`execve(path, argv, envp)` throws away the address space and builds a new one
from the ELF binary (Phase 9 covers how). Kept across exec: the PID/PPID, open
fds (unless marked `O_CLOEXEC`), cwd, credentials (unless the file is
setuid — Lesson 11). This keep-list is exactly what makes the fork→setup→exec
pattern work.

### The modern spellings

Under the hood Linux implements fork via [`clone(2)`](https://man7.org/linux/man-pages/man2/clone.2.html) —
the general "create a task, choose what to share" syscall whose flags also
create threads (Lesson 24) and namespaced processes (Phase 12): fork = "share
nothing", thread = "share everything". You'll see `clone` in strace, not
`fork`. Convenience APIs like `posix_spawn` bundle fork+exec for the common
case; `vfork` is a historical footgun to recognize and avoid.

{: .note }
> **fork's dark side at scale**
> CoW makes fork cheap — but not free: page tables for 2 GiB still get copied,
> and a huge process forking under memory pressure can trip the overcommit/OOM
> machinery (Lesson 22). That's why big daemons (databases, JVMs) prefer
> posix_spawn, a small helper process, or threads for parallel work.

---

## Lab

Today you write real C — a miniature shell.

```bash
$ cat > /tmp/minishell.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/wait.h>
#include <fcntl.h>

int main(void) {
    char line[256];
    while (printf("mini> "), fgets(line, sizeof line, stdin)) {
        line[strcspn(line, "\n")] = 0;
        if (!*line) continue;
        if (!strcmp(line, "exit")) break;

        pid_t pid = fork();
        if (pid == 0) {                       /* child: same code, pid==0 */
            /* the fork/exec gap: redirect if command ends in ">file"   */
            char *gt = strchr(line, '>');
            if (gt) {
                *gt = 0;
                int fd = open(gt + 1, O_WRONLY|O_CREAT|O_TRUNC, 0644);
                dup2(fd, 1);                  /* stdout now goes to file */
                close(fd);
            }
            char *argv[16]; int n = 0;        /* naive word split */
            for (char *t = strtok(line, " "); t && n < 15; t = strtok(NULL, " "))
                argv[n++] = t;
            argv[n] = NULL;
            execvp(argv[0], argv);            /* replace myself */
            perror("exec failed");            /* only reached on failure! */
            exit(127);
        }
        int status;                           /* parent: wait for child */
        waitpid(pid, &status, 0);
        printf("[exit status %d]\n", WEXITSTATUS(status));
    }
    return 0;
}
EOF
$ gcc -Wall -o /tmp/minishell /tmp/minishell.c && /tmp/minishell
# mini> echo hello
# hello
# [exit status 0]
# mini> ls /nonexistent
# ls: cannot access '/nonexistent': ...
# [exit status 2]
# mini> echo redirected >/tmp/redir.txt        ← no space after >
# [exit status 0]
# mini> exit
$ cat /tmp/redir.txt
# redirected                                   ← your fd surgery worked!

# Now watch your shell do the same dance:
$ strace -f -e trace=clone,execve,wait4,dup2,openat bash -c 'echo hi > /tmp/x' 2>&1 | grep -E 'clone|execve|dup2|wait4' | head
# clone(...)                        = <child pid>       ← the fork
# [pid N] dup2(3, 1)                                    ← redirection in the gap
# (echo is a builtin — try /bin/echo or ls to see the child execve!)
$ strace -f -e trace=clone,execve bash -c 'ls > /tmp/x' 2>&1 | grep -E 'clone|execve'
# execve("/usr/bin/bash"...)  then  clone(...)  then  [pid N] execve("/usr/bin/ls"...)

# CoW in action: a big process forks instantly
$ python3 -c "
import os, time
big = bytearray(500_000_000)          # 500 MB
t = time.time()
pid = os.fork()
if pid == 0: os._exit(0)
os.waitpid(pid, 0)
print(f'fork+exit took {time.time()-t:.3f}s despite 500MB')  # a few ms
"
```

Clean up: `rm /tmp/minishell /tmp/minishell.c /tmp/redir.txt /tmp/x`.

---

## Further Reading

| Topic | Link |
|---|---|
| `fork(2)` man page | <https://man7.org/linux/man-pages/man2/fork.2.html> |
| `execve(2)` man page | <https://man7.org/linux/man-pages/man2/execve.2.html> |
| `clone(2)` man page | <https://man7.org/linux/man-pages/man2/clone.2.html> |
| Copy-on-write | <https://en.wikipedia.org/wiki/Copy-on-write> |
| Fork–exec (Wikipedia) | <https://en.wikipedia.org/wiki/Fork%E2%80%93exec> |
| `posix_spawn(3)` man page | <https://man7.org/linux/man-pages/man3/posix_spawn.3.html> |

---

## Checkpoint

**Q1.** After `fork()`, both processes continue from the same line of code. How
does each know which one it is?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
By fork's return value — the one syscall that returns twice, differently. In
the parent it returns the child's PID (a positive number, useful for waitpid);
in the child it returns 0. Every fork call is therefore followed by an
if/else on the return value: <code>pid == 0</code> → "I'm the child, do child
things (setup + exec)"; <code>pid > 0</code> → "I'm the parent, remember and
wait"; <code>-1</code> → the fork failed and there is no child.
</details>

**Q2.** In minishell, the redirection happens in the child *between* fork and
exec. Why is this the only place it can happen? What would go wrong doing it
(a) in the parent before fork, or (b) after exec?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) In the parent before fork, you'd redirect the <em>shell's own</em> stdout —
every subsequent prompt and command would go to the file too; you'd have to
carefully undo it, racing against other activity. (b) "After exec" doesn't
exist: exec replaces the program — ls's code is now running and it has no idea
any redirection was wanted; there's no one left to do it. The fork/exec gap is
the unique moment when code <em>you control</em> runs <em>inside the process
that will become the command</em> — the child rewires only itself, then execs,
and the new program simply inherits fd 1 pointing at the file. This gap is the
entire reason fork and exec are separate.
</details>

**Q3.** `perror("exec failed")` in minishell is only reached "on failure". Why
does successful exec never return, and what does its non-return imply got
destroyed?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
execve replaces the process's entire program image: address space, code, stack —
all of it is torn down and rebuilt from the new binary. "Returning" means
resuming the caller's next instruction, but that instruction (and the whole
calling program, including your minishell's code, variables, and the line
buffer) no longer exists in this process. So success has nowhere to return to;
only failure (file not found, permission denied) leaves the old image intact
and returns -1. The process shell — PID, fds, cwd, credentials — survives;
the program contents don't.
</details>

---

## Homework

Between fork and exec, minishell's child could do much more than redirect.
Extend it (or reason it out on paper) to support: **(a)** `2>file` (stderr
redirection), and **(b)** running the command with a modified environment
variable, like `FOO=bar cmd`. State exactly which lines go where, and *why
neither feature requires any change to the executed programs*.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) Parse <code>2></code>; in the child, open the target and
<code>dup2(fd, 2)</code> instead of <code>dup2(fd, 1)</code> — same fd surgery,
different slot in the table. (b) Parse the <code>NAME=value</code> prefix; in
the child call <code>setenv("FOO", "bar", 1)</code> before exec — or pass a
custom <code>envp</code> to <code>execve</code>. Both go in the child, in the
fork/exec gap. No executed program changes because both features are
implemented purely as <em>inherited state</em>: exec preserves the fd table and
passes the environment to the new program. ls/grep/anything just reads fd 2 and
getenv("FOO") as always, and finds whatever the child arranged. The Unix
composition model in one sentence: parents configure inheritance; programs stay
generic.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 8 — Process Lifecycle: Zombies, Orphans, wait →](lesson-08-process-lifecycle){: .btn .btn-primary }
