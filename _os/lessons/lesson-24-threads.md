---
title: "Lesson 24 — Threads vs Processes"
nav_order: 1
parent: "Phase 5: Concurrency"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 24: Threads vs Processes

## Concept

Lesson 06 planted the seed: the kernel schedules *tasks*, and a "process" is a
group of tasks sharing resources. Lesson 07 revealed the creation syscall is
really `clone()` with flags choosing *what to share*. Put together, the big
reveal is:

**Threads and processes are the same thing with different sharing settings.**

```
                clone() flags:        share?
                ─────────────────────────────────────────────
                              fork()          pthread_create()
                              ──────          ────────────────
  address space (CLONE_VM)     copy(CoW)       SHARED  ← the big one
  fd table      (CLONE_FILES)  copy            SHARED
  fs info/cwd   (CLONE_FS)     copy            SHARED
  signal handlers(CLONE_SIGHAND) copy          SHARED
  thread group  (CLONE_THREAD) new TGID        same TGID (one "process")
                ─────────────────────────────────────────────
  NOT shared even between threads:
    registers & stack (each thread runs somewhere different!)
    errno, signal MASK, scheduling state, TID
```

Everything about threads follows from `CLONE_VM`: any thread can read and
write any other's data (that's the *point* — zero-copy communication), which
is also why one thread's wild pointer corrupts them all, and why the rest of
this phase exists. Processes are the safety choice (isolation, independent
failure — Chrome's per-tab processes); threads are the sharing choice
(parallel work on one dataset with no serialization cost).

---

## How It Works

### What the kernel sees

`pthread_create` → `clone(CLONE_VM|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD…)`
→ a new task with its own TID, sharing the group's TGID (Lesson 06's
distinction, now fully loaded). Each thread appears in `/proc/PID/task/TID/`
with its own status, stack, scheduling stats — and the scheduler treats each
as an independent runnable (a 4-thread process can use 4 CPUs: "400% CPU" in
top decoded). Thread stacks are ordinary mmaps (~8 MiB each, with a guard
page — Lesson 18), which is why 10,000 threads cost real address space and why
event loops exist (Lesson 30).

### The shared/private split in practice

Shared address space means: globals, heap, mmaps — all visible to all threads
instantly (with caveats — Lesson 29's memory ordering!). Private per thread:
stack (locals are safe by default!), registers, `errno` (thread-local by libc
magic), signal *mask* (per-thread; handlers shared — a process-directed signal
is delivered to *one* arbitrary unblocked thread: the classic "why did my
signal land there" mystery), and TLS (`__thread` variables — how errno works).

### Choosing

Threads: shared working set, fine-grained parallelism, cheap create/switch
(Lesson 12: no address-space switch). Processes: fault isolation (one crash ≠
all crash), privilege separation (different uids! — Lesson 11), independent
deployment/restart, no data races *by construction* (share only what you
explicitly IPC — Phase 6). The modern middle grounds: thread pools sized to
cores doing CPU work + processes for isolation boundaries (browsers,
postgres); or async on few threads for I/O concurrency (Lesson 30).

{: .note }
> **Python's asterisk**
> CPython's GIL serializes bytecode execution: Python threads give
> <em>concurrency</em> (waiting on many I/Os) but not CPU <em>parallelism</em> —
> hence multiprocessing for compute. The C labs here show true parallelism;
> Python labs in this phase demonstrate structure, not speedup. (Python 3.13's
> free-threaded build is changing this story.)

---

## Lab

```bash
# ---- 1. Threads ARE tasks: see them individually ----
$ cat > /tmp/th.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/syscall.h>
void *work(void *arg) {
    printf("thread %ld: pid=%d tid=%ld\n",
           (long)arg, getpid(), syscall(SYS_gettid));
    sleep(30);
    return NULL;
}
int main(void) {
    pthread_t t[3];
    for (long i = 0; i < 3; i++) pthread_create(&t[i], NULL, work, (void*)i);
    printf("main:     pid=%d tid=%ld\n", getpid(), syscall(SYS_gettid));
    for (int i = 0; i < 3; i++) pthread_join(t[i], NULL);
    return 0;
}
EOF
$ gcc -pthread -o /tmp/th /tmp/th.c && /tmp/th &
# same pid for all; DIFFERENT tids; main's tid == pid (group leader, L06!)
$ ls /proc/$(pgrep -f /tmp/th)/task/
# four directories — four schedulable tasks
$ ps -eLf | grep /tmp/th | head -5        # ps -L: one line per thread (LWP=tid)
$ grep Threads /proc/$(pgrep -f /tmp/th)/status
# Threads: 4
$ kill %1

# ---- 2. Shared memory, zero effort — and private stacks ----
$ cat > /tmp/share.c << 'EOF'
#include <pthread.h>
#include <stdio.h>
int shared_global = 0;                      /* one copy for all threads   */
void *writer(void *arg) {
    int my_local = 42;                      /* on THIS thread's stack     */
    shared_global = 1337;                   /* visible to everyone        */
    printf("writer: local at %p\n", (void*)&my_local);
    return NULL;
}
int main(void) {
    pthread_t t;
    int my_local = 7;
    pthread_create(&t, NULL, writer, NULL);
    pthread_join(t, NULL);
    printf("main:   local at %p, shared_global = %d\n",
           (void*)&my_local, shared_global);   /* sees 1337 instantly */
    return 0;
}
EOF
$ gcc -pthread -o /tmp/share /tmp/share.c && /tmp/share
# shared_global = 1337 ← no pipes, no sockets: same memory
# the two locals' addresses are FAR apart: different stack mmaps (check maps!)

# ---- 3. True parallelism: N threads, N CPUs ----
$ cat > /tmp/burn.c << 'EOF'
#include <pthread.h>
#include <stdlib.h>
void *spin(void *a){ volatile long i=0; while(++i < 3000000000L); return 0; }
int main(int c, char **v) {
    int n = atoi(v[1]);
    pthread_t t[16];
    for (int i = 0; i < n; i++) pthread_create(&t[i], NULL, spin, NULL);
    for (int i = 0; i < n; i++) pthread_join(t[i], NULL);
    return 0;
}
EOF
$ gcc -pthread -O0 -o /tmp/burn /tmp/burn.c
$ time /tmp/burn 1     # note real vs user
$ time /tmp/burn $(nproc)
# user time ~n×real: n CPUs genuinely burning at once. top shows n×100%.

# ---- 4. clone flags: watch pthread_create vs fork in strace ----
$ strace -f -e trace=clone,clone3 /tmp/th 2>&1 | grep -m1 clone
# clone(child_stack=0x7f..., flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND
#       |CLONE_THREAD|...)              ← the sharing contract, spelled out
$ strace -e trace=clone,clone3 bash -c 'ls > /dev/null' 2>&1 | grep -m1 clone
# clone(... flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD ...)
#   ← fork: no CLONE_VM. Copy, not share.

# ---- 5. A crash takes the whole thread group ----
$ python3 - << 'EOF'
import ctypes, threading, time
def crasher():
    time.sleep(1); ctypes.string_at(0)      # SIGSEGV in a "worker thread"
threading.Thread(target=crasher, daemon=True).start()
print("main thread doing important work...")
time.sleep(3)
print("never printed — the worker's segfault killed EVERYONE")
EOF
# Segmentation fault — one thread's fault, all threads dead. Isolation: zero.

$ rm /tmp/th /tmp/th.c /tmp/share /tmp/share.c /tmp/burn /tmp/burn.c
```

---

## Further Reading

| Topic | Link |
|---|---|
| `pthreads(7)` overview | <https://man7.org/linux/man-pages/man7/pthreads.7.html> |
| `clone(2)` man page | <https://man7.org/linux/man-pages/man2/clone.2.html> |
| Thread (computing) | <https://en.wikipedia.org/wiki/Thread_(computing)> |
| `gettid(2)` man page | <https://man7.org/linux/man-pages/man2/gettid.2.html> |
| Thread-local storage | <https://en.wikipedia.org/wiki/Thread-local_storage> |
| Global interpreter lock | <https://en.wikipedia.org/wiki/Global_interpreter_lock> |

---

## Checkpoint

**Q1.** One thread calls `chdir("/tmp")`. Does it affect its sibling threads?
What about `errno` — and why are the two answers different?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<code>chdir</code> affects all siblings: the working directory lives in the
shared fs-info structure (CLONE_FS) — one cwd per thread group, so a
subsequent relative <code>open()</code> in any thread resolves from /tmp. A
classic real bug: worker A chdirs "temporarily" and worker B's file lands in
the wrong place. <code>errno</code> is the opposite: though it looks like a
global, libc defines it as a thread-local variable (TLS) precisely so one
thread's failing syscall doesn't corrupt another's error handling. The general
rule: kernel-side task-group state (cwd, fds, handlers, address space) is
shared by the clone flags; language-runtime per-thread state (stack, errno,
TLS) is deliberately private. Knowing which list a resource is on predicts a
whole class of bugs.
</details>

**Q2.** Why is creating 10,000 threads for 10,000 idle network connections a
poor design, quantitatively — and what property of threads does the standard
alternative give up?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Costs per thread: a stack mmap (default 8 MiB virtual each — 80 GB of address
space; with a few touched pages each, still real RSS in the GBs), a kernel
task_struct with scheduler bookkeeping, and — when traffic does arrive —
wakeup/context-switch costs multiplied across thousands of mostly-idle tasks
(Lesson 12), plus scheduler load-balancing overhead. Mostly-idle connections
don't need <em>execution contexts</em>; they need <em>state + readiness
notification</em> — a few KB of buffers per connection and one epoll fd
(Lesson 43), served by ~nproc threads. What's given up: the straight-line
"blocking" programming model — each connection's logic must become
callback/state-machine/coroutine-shaped (async/await re-sugars it — Lesson
30), and one slow computation now blocks a whole event loop rather than one
connection's private thread.
</details>

**Q3.** A signal (say SIGTERM) is sent to a multi-threaded process. Which
thread receives it, how do signal masks interact, and what's the standard
pattern for sane signal handling in threaded programs?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Process-directed signals are delivered to <em>exactly one</em> thread, chosen
arbitrarily among those not blocking it — handlers are shared (CLONE_SIGHAND)
but masks are per-thread, so delivery targeting follows the masks. This
arbitrariness breaks naive designs (the handler runs on whichever thread,
interrupting arbitrary state — Lesson 09's async-safety rules apply per
thread). Standard pattern: <strong>block the signals in every thread</strong>
(set the mask in main before spawning — children inherit it), then dedicate
one thread to <code>sigwait()</code> (or signalfd in the event loop): signals
become synchronous messages consumed at a chosen point, no handler async-mess
at all. This is exactly what most servers and runtimes (JVM, Go) do
internally — and another instance of Lesson 35's theme: turn asynchronous
chaos into an fd/queue you read when ready.
</details>

---

## Homework

Chrome uses a process per tab; Apache historically offered both process-based
(prefork) and thread-based (worker) models; PostgreSQL uses a process per
connection; nginx uses a few processes each running an event loop. For each of
the four, name the property from this lesson that drove the choice, and one
cost accepted.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Chrome</strong>: fault + security isolation — a tab's renderer crash
or exploit is contained by process boundaries (separate address spaces,
seccomp'd, different privileges — Lessons 11/56); cost: memory duplication
and IPC complexity for everything cross-tab. <strong>Apache prefork vs
worker</strong>: prefork buys crash isolation and safety with non-thread-safe
modules (early PHP!) at high per-connection memory; worker shares memory for
far more connections per GB, accepting that one heap corruption kills many
connections. <strong>PostgreSQL</strong>: per-connection processes isolate
sessions (one backend's crash doesn't corrupt others' memory — the shared
buffer pool lives in explicit shared memory, Lesson 33, which is the
deliberate, auditable sharing) at the cost of expensive connections — hence
the entire pgbouncer/connection-pooler ecosystem. <strong>nginx</strong>:
mostly-idle connections are state+readiness, not contexts (Q2!) — event loops
serve tens of thousands per process; cost: the async programming model and
careful handling of anything blocking (disk I/O gets thread pools anyway).
One decision, four different right answers — always via the same trade-off
table.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 25 — Race Conditions →](lesson-25-race-conditions){: .btn .btn-primary }
