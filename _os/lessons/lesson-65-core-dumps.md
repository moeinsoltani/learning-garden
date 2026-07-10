---
title: "Lesson 65 — Core Dumps and Crash Analysis"
nav_order: 3
parent: "Phase 13: Tracing & Debugging"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 65: Core Dumps and Crash Analysis

## Concept

perf and ftrace observe *running* systems. This lesson is about the opposite:
a process already died — how do you find out *why*? The answer is the **core
dump** — a snapshot of a process's entire memory and register state at the
instant it crashed, frozen for post-mortem analysis.

```
   process hits SIGSEGV/SIGABRT (L09)
          │  the kernel, per the signal's "terminate + core" default action
          ▼
   CORE DUMP written: an ELF file (L46!) containing:
     • every memory region (the maps of L18 — heap, stack, data)
     • all CPU registers at the crash instant
     • the call stack, thread states, open fds
          │
          ▼  open it in gdb with the binary + symbols:
   gdb ./program core
   (gdb) bt          ← the backtrace: exactly where it died, and how it got there
```

A core dump is a crime-scene photograph: the process is dead, but everything
about its final moment is preserved. It's an ELF file (Lesson 46 — the same
format as executables, a third hat), containing the process's address space
(Lesson 18's regions) and register state, which gdb reconstructs into a
readable stack trace, variable values, and thread states.

This ties together the whole course's memory and process model: the segfault
(Lesson 16's verdict), the signal that killed it (Lesson 09), the memory layout
you're examining (Lesson 18), the symbols that make it readable (Lesson 46) —
all converge in the post-mortem. And it's a daily production skill: services
crash, and the core dump plus `coredumpctl` is how you find out why without
reproducing it.

---

## How It Works

### Getting the dump

Core dumps are gated by a resource limit (`ulimit -c` — often 0, i.e.
disabled) and routed by `/proc/sys/kernel/core_pattern` — which on modern
systems pipes to **systemd-coredump** (dumps go to a journal-integrated store,
queried with `coredumpctl`). So: `ulimit -c unlimited` (or configure the
service's `LimitCore=`), crash, then `coredumpctl list` / `coredumpctl gdb
PID` to open the latest. Containers and cloud environments often need explicit
core-dump configuration (the pattern is host-wide, a gotcha).

### Reading it in gdb

`gdb ./binary core` (or `coredumpctl gdb`). The essential commands:

- **`bt`** (backtrace) — the call stack at the crash: the sequence of function
  calls that led to death. The single most valuable output.
- **`frame N`** / **`up`/`down`** — move to a stack frame to inspect its
  locals.
- **`print expr`** / **`info locals`** — the values of variables at the crash
  (why was that pointer NULL? — read it).
- **`info registers`**, **`x/i $pc`** — the instruction that faulted.
- **`thread apply all bt`** — all threads (the deadlock diagnosis from Lesson
  28!).

### Symbols: the difference between insight and `??`

A backtrace showing `??` at every frame is useless — that means gdb has no
**debug symbols** (Lesson 46's `.symtab`/DWARF) mapping addresses to function
names and source lines. Fixes: build with `-g` (embed DWARF), install the
distro's `-dbgsym`/`-debuginfo` packages (matching the exact binary version),
or ensure the binary wasn't stripped. This is why production builds ship
separate debuginfo (the binary stays small, symbols available when needed) and
why version-matched symbols matter (wrong-version symbols give plausible-but-
wrong answers). The same idea scales to the kernel: **kdump** captures a
kernel core on panic, analyzed with `crash` and kernel debug symbols — the
post-mortem for the crashes Lesson 54 warned about.

{: .note }
> **The `??` frame's two causes**
> A backtrace of <code>??</code> means either (1) missing symbols — install
> debuginfo / rebuild with -g (the common case); or (2) a <em>corrupted
> stack</em> — the crash overwrote the return addresses (buffer overflow,
> stack smashing), so there's no valid chain to walk. Distinguishing them:
> missing symbols still show valid-looking addresses that resolve with the
> right debuginfo; a smashed stack shows garbage/unmapped addresses even
> with symbols — itself a diagnosis (memory corruption, Lesson 18's guard
> pages breached).

---

## Lab

```bash
# ---- 1. Enable core dumps ----
$ ulimit -c                             # probably 0 (disabled)
$ ulimit -c unlimited                   # enable for this shell
$ cat /proc/sys/kernel/core_pattern     # where dumps go (often |systemd-coredump)

# ---- 2. A program that crashes, with symbols (-g) ----
$ cat > /tmp/crash.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
struct node { int value; struct node *next; };
int sum_list(struct node *n) {
    int total = 0;
    while (n) { total += n->value; n = n->next; }   /* crash if n is corrupt */
    return total;
}
int main(void) {
    struct node *head = malloc(sizeof *head);
    head->value = 10;
    head->next = (struct node *)0xdeadbeef;          /* BAD pointer — bug! */
    printf("sum = %d\n", sum_list(head));            /* SIGSEGV in sum_list */
    return 0;
}
EOF
$ gcc -g -O0 -o /tmp/crash /tmp/crash.c    # -g = debug symbols (L46)
$ cd /tmp && /tmp/crash
# Segmentation fault (core dumped)          ← the "terminate + core" action (L09)
$ echo "exit: $?"                            # 139 = 128 + 11 (SIGSEGV) (L08 math)

# ---- 3. Open the dump and read the backtrace ----
$ ls core* 2>/dev/null && gdb -q -batch -ex bt /tmp/crash core* 2>/dev/null | head -8 \
  || coredumpctl gdb /tmp/crash 2>/dev/null <<< $'bt\nquit' 2>/dev/null | grep -A6 '#0'
# #0  sum_list (n=0xdeadbeef) at /tmp/crash.c:5        ← EXACTLY where it died
# #1  main () at /tmp/crash.c:13                        ← and who called it
#     the crash is n->value with n=0xdeadbeef — the bad pointer, named.

# ---- 4. Inspect the state at the crash ----
$ gdb -q -batch -ex 'bt' -ex 'frame 0' -ex 'print n' -ex 'info locals' \
    /tmp/crash core* 2>/dev/null | grep -E 'n = |total|#' | head -6
# n = (struct node *) 0xdeadbeef    ← the smoking gun: the corrupt pointer value
# you can READ the dead program's variables — post-mortem debugging

# ---- 5. The ?? problem: strip the symbols, lose the insight ----
$ gcc -O2 -o /tmp/crash_stripped /tmp/crash.c && strip /tmp/crash_stripped
$ cd /tmp && /tmp/crash_stripped 2>/dev/null; ls core* >/dev/null 2>&1
$ gdb -q -batch -ex bt /tmp/crash_stripped core* 2>/dev/null | head -4
# #0  0x... in ?? ()          ← useless! no symbols → no function names/lines
# THIS is why production ships debuginfo separately (L46) — same binary, but
# with matching -dbgsym you'd get frame 3's story back.

# ---- 6. Multi-thread: thread apply all bt (the deadlock post-mortem, L28) ----
$ cat > /tmp/mt.c << 'EOF'
#include <pthread.h>
#include <stdlib.h>
void *crasher(void *a){ int *p = NULL; return (void*)(long)*p; }  /* SEGV */
int main(){ pthread_t t; pthread_create(&t,0,crasher,0); pthread_join(t,0); return 0; }
EOF
$ gcc -g -pthread -o /tmp/mt /tmp/mt.c && cd /tmp && /tmp/mt 2>/dev/null
$ gdb -q -batch -ex 'thread apply all bt' /tmp/mt core* 2>/dev/null | grep -E 'Thread|crasher|#0' | head -6
# shows WHICH thread crashed and its stack — the same command that diagnoses
# deadlocks (L28) works on the corpse: all threads' states, frozen.
$ rm -f /tmp/crash* /tmp/mt* /tmp/core*
```

---

## Further Reading

| Topic | Link |
|---|---|
| `core(5)` — core dump files | <https://man7.org/linux/man-pages/man5/core.5.html> |
| `coredumpctl(1)` | <https://www.freedesktop.org/software/systemd/man/latest/coredumpctl.html> |
| GDB documentation | <https://sourceware.org/gdb/documentation/> |
| Core dump (Wikipedia) | <https://en.wikipedia.org/wiki/Core_dump> |
| kdump — kernel crash dumps | <https://www.kernel.org/doc/html/latest/admin-guide/kdump/kdump.html> |
| DWARF debugging format | <https://en.wikipedia.org/wiki/DWARF> |

---

## Checkpoint

**Q1.** The backtrace shows `??` for every frame. Two likely causes and the
fix for each.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Cause 1 — <strong>missing debug symbols</strong> (the common case): the binary
was built without <code>-g</code> or was stripped (Lesson 46 — the .symtab
removed), so gdb has addresses but no mapping to function names or source
lines, showing <code>??</code>. Fix: get matching symbols — rebuild with
<code>-g</code>, install the distro's <code>-dbgsym</code>/<code>-debuginfo</code>
package for the <em>exact</em> binary version (wrong version = wrong or
misleading answers), or point gdb at a separate debuginfo file
(<code>.debug</code>). Telltale: the frame addresses look valid and resolve
sensibly once symbols are supplied. Cause 2 — <strong>a corrupted stack</strong>:
the crash smashed the stack (buffer overflow, stack-buffer overrun — the return
addresses gdb walks to build the backtrace were overwritten with garbage/
attacker data, Lesson 18's stack-overflow territory), so there's no valid call
chain to reconstruct even <em>with</em> symbols — gdb follows garbage return
addresses into nowhere. Fix: this is itself the diagnosis (memory corruption) —
the fix is finding the overflow (rebuild with AddressSanitizer
<code>-fsanitize=address</code> or valgrind to catch the out-of-bounds write
that smashed the stack, then fix the bug). Distinguishing them: with correct
symbols supplied, missing-symbols resolves to a clean backtrace; a smashed
stack still shows <code>??</code> at unmapped/garbage addresses — and often the
crash instruction itself is a nonsense address (jumped through a corrupted
return pointer). So <code>??</code> everywhere is triaged by "do I have
matching symbols?": if adding them fixes it → cause 1; if it doesn't → cause 2,
and you've learned the crash is corruption, not a simple null deref. (A
frame_0-only <code>??</code> with good frames below often means a crash in a
stripped library, e.g. a distro .so without dbgsym — install that library's
debuginfo.)
</details>

**Q2.** A core dump lets you debug a crash that happened in production without
reproducing it. Why is that so valuable, and what specifically does the dump
preserve that a log message can't?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Reproducing a production crash is often impossible or expensive: it may depend
on a specific input, a race (Lesson 25's non-deterministic timing — the crash
happened once at 3am under load you can't recreate), a particular data state,
or an environment you can't replicate in a test rig. Waiting for it to recur
means the bug lives in production longer, and some crashes recur rarely enough
that "reproduce it" is not a plan. The core dump captures the crash <em>the
first time</em>, so you debug the actual failure from the actual conditions,
once, offline. What it preserves that a log can't: a log line is whatever the
programmer <em>thought to print in advance</em> — it can't answer questions you
didn't anticipate. The core dump preserves the <em>entire machine state at the
crash instant</em>: the full call stack (how the code got there — Q1's
backtrace), every local and global variable's value (why was that pointer bad?
what was the loop counter? — read them directly, lab 4), the CPU registers and
the exact faulting instruction, all threads' states (which thread crashed and
what the others were doing — Lesson 28's deadlock view), and the memory regions
(Lesson 18 — you can inspect the heap, find the corrupted structure). It lets
you ask <em>arbitrary</em> questions after the fact — "what was in this buffer?
what did this pointer point to? what was thread 3 waiting on?" — that no amount
of pre-planned logging could cover, because you can't log everything (volume,
Lesson 64's firehose) and you don't know in advance what you'll need. Logs tell
you the story the author expected; the core dump lets you interrogate the crime
scene. Best practice combines them: logs to notice and narrow ("service crashed
processing request X"), core dumps to diagnose precisely — which is why
production services enable core dumps (LimitCore, coredumpctl) and ship
separate debuginfo (Q1) so the dumps are readable when they matter.
</details>

**Q3.** Connect the core dump to four earlier lessons: it's an ELF file
containing a process's state at the moment a signal killed it. Name the four
lessons and what each contributes to understanding what you're looking at.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <strong>Lesson 46 (ELF)</strong>: the core dump <em>is</em> an ELF file —
the same format as executables and shared libraries, wearing a third hat (a
core-type ELF). Its program headers describe memory segments to reconstruct
rather than to load; understanding ELF is understanding what a core dump
physically is and why gdb can parse it. (2) <strong>Lesson 18 (process memory
layout)</strong>: the dump contains the process's memory regions — heap, stack,
data, mmap'd areas (the <code>/proc/PID/maps</code> regions) — so examining a
core is examining the address space you learned to read; when gdb prints a
variable or walks the heap, it's reading these preserved regions, and knowing
the layout tells you what part of the crash you're inspecting (a stack-local vs
a heap object vs a global). (3) <strong>Lesson 09 (signals)</strong>: the dump
exists <em>because</em> a signal with the "terminate + core" default action
(SIGSEGV, SIGABRT, SIGFPE, SIGQUIT) killed the process — the exit status 139 =
128+11 (Lesson 08's arithmetic) names SIGSEGV as the killer, and knowing which
signal produced the dump tells you the failure <em>class</em> (SIGSEGV = bad
memory access — Lesson 16's verdict; SIGABRT = an assertion or abort()/
double-free; SIGFPE = arithmetic). (4) <strong>Lesson 16 (virtual memory)</strong>:
the crash itself is Lesson 16's story — the faulting instruction accessed a
virtual address with no valid mapping (or wrong permissions), the MMU faulted,
the kernel found no legitimate region and delivered SIGSEGV (the "verdict, not
an accident"); the core dump captures the exact virtual address that had no
home (the <code>n=0xdeadbeef</code> of the lab), and understanding virtual
memory is understanding why that address was invalid. Together: an ELF-format
(46) snapshot of a virtual address space (16/18), created because a fatal
signal (09) fired — every layer of the course's process and memory model
converging in one file that lets you reconstruct a death. The post-mortem is
where the whole course pays off as a single diagnostic competence, which the
final capstone (Lesson 66) formalizes into a methodology.
</details>

---

## Homework

Deliberately create three different crashes (a NULL dereference, a
stack-buffer overflow with `-fno-stack-protector`, and an `abort()`/failed
assertion), enable core dumps, and analyze each with gdb. For each, report:
the signal that killed it (and the exit status arithmetic), what `bt` showed
(clean backtrace vs `??`), and how the dump distinguished this crash class
from the others. Then explain why the stack-overflow case is the hardest to
debug from the core alone — connecting to Q1 and Lesson 18.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>NULL dereference</strong> (<code>int *p=NULL; *p=1;</code>): killed by
<strong>SIGSEGV</strong> (signal 11, exit 139 = 128+11); <code>bt</code> shows
a clean backtrace pointing exactly at the dereferencing line with the faulting
pointer visible as 0x0 (or a small offset) — the easiest to debug: the crash
site, the bad address (0x0 = obviously null), and the call chain are all clear
(the lab's shape). Diagnosis: something was null that shouldn't be; read
<code>bt</code> and the null variable, trace back to why it wasn't initialized.
<strong>abort()/failed assertion</strong> (e.g. <code>assert(x)</code> failing,
or glibc detecting a double-free/heap corruption): killed by <strong>SIGABRT</strong>
(signal 6, exit 134 = 128+6); <code>bt</code> shows a clean backtrace whose top
frames are <code>abort</code> → <code>raise</code> → and below them <em>the code
that called abort</em> — often glibc's malloc detecting corruption
("free(): invalid pointer") or your assert; the backtrace's lower frames name
where the program decided to die, which is usually near (but not always at) the
real bug. Distinguished from SIGSEGV by the signal (6 vs 11) and the
abort/raise frames at the top. <strong>Stack-buffer overflow</strong>
(<code>char buf[8]; strcpy(buf, longstring);</code> with
<code>-fno-stack-protector</code>): typically killed by <strong>SIGSEGV</strong>
(the corrupted return address makes the function "return" to a garbage/unmapped
address — Lesson 18's stack region overrun into the return-address chain); exit
139. But <code>bt</code> shows <strong><code>??</code></strong> or a nonsensical
backtrace even <em>with</em> symbols — because the overflow overwrote the saved
return addresses that gdb walks to build the stack trace: there's no valid call
chain left to reconstruct (Q1's cause 2). This is why it's hardest from the core
alone: the very evidence you'd use (the backtrace) is what the bug destroyed —
the crash location (some garbage address the function "returned" to) tells you
nothing about where the overflow <em>happened</em>, and the smashed stack means
you can't trace back to the overflowing write. You need other tools:
AddressSanitizer (<code>-fsanitize=address</code>) or valgrind to catch the
out-of-bounds write <em>at the moment it happens</em> (before the corrupted
return is used), or examine the raw stack memory in the core for the overflow
pattern (the long string sitting where return addresses should be — recognizing
your input data in the stack frame is the tell). It connects to Lesson 18: the
stack's guard page catches overflows that run <em>off</em> the stack, but an
overflow <em>within</em> the stack (clobbering this frame's saved registers/
return address) corrupts the call chain silently until the bad return — and to
Q1: <code>??</code> from a smashed stack is itself the diagnosis (memory
corruption, not missing symbols), redirecting you from "read the core" to "catch
the write with ASan." The three crashes teach the triage: the signal names the
class (11 vs 6), the backtrace's <em>quality</em> (clean vs <code>??</code>)
distinguishes a simple fault from corruption, and corruption sends you from
post-mortem tools to live-detection tools — the debugging decision tree the
capstone assembles.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 66 — Debugging Methodology (Capstone) →](lesson-66-debugging-capstone){: .btn .btn-primary }
