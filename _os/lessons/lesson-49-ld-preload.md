---
title: "Lesson 49 — LD_PRELOAD and Symbol Interposition"
nav_order: 4
parent: "Phase 9: Linking & Loading"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 49: LD_PRELOAD and Symbol Interposition

## Concept

Lesson 48 hinted at it; now wield it. **LD_PRELOAD** is a userspace
superpower: force the dynamic loader to load *your* library first, so *your*
functions get resolved before the real ones — and every call the target
program makes to that function runs your code instead. No recompiling, no
source, no cooperation from the program.

```
  normal:   app calls malloc  →  loader resolves to  →  libc's malloc

  LD_PRELOAD=./mymalloc.so app:
            app calls malloc  →  loader searches PRELOAD FIRST  →  YOUR malloc
                                                                       │
                                            (which may call the real one via
                                             dlsym(RTLD_NEXT, "malloc"))
```

This works because of **symbol interposition**: the loader resolves each
symbol to the *first* definition in its search scope, and preloaded
libraries go to the front (Lesson 47's search order, weaponized). It's the
same mechanism as RTLD_GLOBAL's accidental clobbering (Lesson 48 Q3), made
deliberate.

Legitimate uses are everywhere: `fakeroot` (intercept chown/stat to fake
root during package builds), jemalloc/tcmalloc (`LD_PRELOAD=libjemalloc.so`
— swap the allocator with zero code changes), profilers and leak detectors,
`proxychains`/`torsocks` (intercept connect() to route through a proxy),
LD_PRELOAD-based sandboxes, and debugging shims that log every `open()` a
misbehaving binary makes. It's how you inspect and modify closed programs at
the library boundary — the userspace complement to strace's syscall
boundary (Lesson 02) and ftrace's kernel boundary (Lesson 64).

---

## How It Works

### Writing an interposer

Define a function with the exact signature of the one you're replacing; the
loader picks yours. To *also* call the original (the common case — log, then
delegate), use `dlsym(RTLD_NEXT, "name")` — "find the *next* definition after
me in the search order," i.e. the real libc one. The pattern:

```c
ssize_t write(int fd, const void *buf, size_t n) {
    static ssize_t (*real)(int, const void*, size_t);
    if (!real) real = dlsym(RTLD_NEXT, "write");   // find the real write
    fprintf(stderr, "[intercepted write of %zu bytes]\n", n);
    return real(fd, buf, n);                        // delegate
}
```

Set `LD_PRELOAD=./shim.so command`. `__attribute__((constructor))` functions
run at load (before main — the initializer slot from Lesson 47) for setup.

### The gaps — and why they're security features

LD_PRELOAD has exactly the reach of the dynamic library boundary, so:

- **Static binaries**: nothing to interpose (libc is baked in, calls are
  direct — Lesson 46). Go binaries mostly ignore it too. A statically-linked
  target is immune.
- **Raw syscalls**: a program that invokes `syscall(SYS_write, ...)` directly
  bypasses libc's `write` — your shim never fires. Interposition is at the
  *symbol* layer, below it is the syscall layer (seccomp's domain, Lesson
  56); above it, application-level calls it can't see.
- **setuid/setgid binaries**: the loader runs in **secure-execution mode**
  and *ignores* LD_PRELOAD/LD_LIBRARY_PATH entirely (Lesson 11's cliff-guard,
  Lesson 47's note). Otherwise LD_PRELOAD would be an instant root exploit —
  preload a malicious libc into `sudo`. This single rule is why the
  environment-based mechanism is safe to have at all.

These aren't bugs; they're the boundary's edges, and knowing them tells you
exactly when LD_PRELOAD is the right tool and when you need strace/seccomp/
ftrace instead.

{: .note }
> **Interposition without the environment**
> The linker flag <code>-Wl,--wrap=malloc</code> does compile-time
> interposition (calls to <code>malloc</code> become <code>__wrap_malloc</code>,
> the original is <code>__real_malloc</code>) — same idea, baked in, works
> on the symbols you control at build time. And RTLD_GLOBAL dlopen
> (Lesson 48) interposes at runtime without LD_PRELOAD. Three routes to one
> mechanism, for three situations.

---

## Lab

```bash
# ---- 1. Your first interposer: log every open() ----
$ cat > /tmp/spy.c << 'EOF'
#define _GNU_SOURCE
#include <dlfcn.h>
#include <stdio.h>
#include <fcntl.h>
#include <stdarg.h>
int open(const char *path, int flags, ...) {
    static int (*real_open)(const char*, int, ...);
    if (!real_open) real_open = dlsym(RTLD_NEXT, "open");
    fprintf(stderr, "[SPY] open(\"%s\")\n", path);
    mode_t mode = 0;
    if (flags & O_CREAT) { va_list a; va_start(a, flags); mode = va_arg(a, int); va_end(a); }
    return real_open(path, flags, mode);
}
EOF
$ gcc -shared -fPIC -o /tmp/spy.so /tmp/spy.c -ldl
$ LD_PRELOAD=/tmp/spy.so cat /etc/hostname 2>&1 | head -5
# [SPY] open("/etc/hostname")     ← you hooked cat WITHOUT touching cat
# (glibc internals may use openat directly — hook that too for full coverage;
#  the gap itself is instructive)

# ---- 2. Swap the allocator, zero code changes ----
$ command -v gcc >/dev/null && cat > /tmp/countmalloc.c << 'EOF'
#define _GNU_SOURCE
#include <dlfcn.h>
#include <stdio.h>
#include <stdlib.h>
static long count = 0;
void *malloc(size_t n) {
    static void *(*real)(size_t);
    if (!real) real = dlsym(RTLD_NEXT, "malloc");
    count++;
    return real(n);
}
__attribute__((destructor)) void report(void) {
    fprintf(stderr, "[STATS] %ld mallocs total\n", count);
}
EOF
$ gcc -shared -fPIC -o /tmp/cm.so /tmp/countmalloc.c -ldl
$ LD_PRELOAD=/tmp/cm.so /bin/echo hi 2>&1 | tail -2
# hi
# [STATS] N mallocs total     ← counted every allocation in echo. This is
# how jemalloc/tcmalloc are dropped into unmodified programs.

# ---- 3. Gap 1: static binaries are immune ----
$ echo 'int main(){open("/etc/hostname",0);return 0;}' > /tmp/st.c
$ gcc -static -include fcntl.h -o /tmp/st_static /tmp/st.c 2>/dev/null
$ gcc -include fcntl.h -o /tmp/st_dynamic /tmp/st.c
$ echo "dynamic:"; LD_PRELOAD=/tmp/spy.so /tmp/st_dynamic 2>&1 | grep SPY
$ echo "static:";  LD_PRELOAD=/tmp/spy.so /tmp/st_static  2>&1 | grep SPY || echo "  (silent — nothing to interpose)"
# dynamic hooked; static SILENT — libc is baked in, calls are direct

# ---- 4. Gap 2: raw syscalls slip under it ----
$ cat > /tmp/raw.c << 'EOF'
#include <sys/syscall.h>
#include <unistd.h>
int main(void) {
    syscall(SYS_open, "/etc/hostname", 0);   /* bypass libc's open wrapper */
    return 0;
}
EOF
$ gcc -o /tmp/raw /tmp/raw.c
$ LD_PRELOAD=/tmp/spy.so /tmp/raw 2>&1 | grep SPY || echo "  (silent — syscall() went under the symbol layer)"
$ strace -e trace=open,openat /tmp/raw 2>&1 | grep hostname
# strace SEES it — because strace watches the SYSCALL boundary, LD_PRELOAD
# the SYMBOL boundary. Different layers, different tools (L02 vs this).

# ---- 5. Gap 3: setuid ignores LD_PRELOAD (the security cliff-guard) ----
$ cp /bin/id /tmp/id_copy && sudo chown root /tmp/id_copy && sudo chmod u+s /tmp/id_copy
$ LD_PRELOAD=/tmp/cm.so /tmp/id_copy 2>&1 | grep -c STATS || echo "  preload IGNORED for setuid — as it MUST be"
# 0 — secure-execution mode dropped LD_PRELOAD. Otherwise: preload a fake
# getuid into any setuid-root binary = instant root. Lesson 11's guard, live.
$ sudo rm /tmp/id_copy

# ---- 6. A real-world shape: fault injection for testing ----
$ cat > /tmp/flaky.c << 'EOF'
#define _GNU_SOURCE
#include <dlfcn.h>
#include <stdlib.h>
#include <errno.h>
void *malloc(size_t n) {
    static void *(*real)(size_t);
    if (!real) real = dlsym(RTLD_NEXT, "malloc");
    if (rand() % 5 == 0) { errno = ENOMEM; return NULL; }   /* 20% failures */
    return real(n);
}
EOF
$ gcc -shared -fPIC -o /tmp/flaky.so /tmp/flaky.c -ldl
$ LD_PRELOAD=/tmp/flaky.so python3 -c "print('did python survive random OOM?')" 2>&1 | tail -1
# inject malloc failures into ANY program to test error paths — a real
# chaos-engineering technique, zero target changes.
$ rm -f /tmp/spy* /tmp/cm* /tmp/countmalloc.c /tmp/st* /tmp/raw* /tmp/flaky*
```

---

## Further Reading

| Topic | Link |
|---|---|
| `ld.so(8)` — LD_PRELOAD | <https://man7.org/linux/man-pages/man8/ld.so.8.html> |
| `dlsym(3)` — RTLD_NEXT | <https://man7.org/linux/man-pages/man3/dlsym.3.html> |
| Interposition / function hooking | <https://en.wikipedia.org/wiki/Hooking> |
| fakeroot — a real interposer | <https://wiki.debian.org/FakeRoot> |
| jemalloc (drop-in via LD_PRELOAD) | <https://github.com/jemalloc/jemalloc> |
| `--wrap` linker option | <https://man7.org/linux/man-pages/man1/ld.1.html> |

---

## Checkpoint

**Q1.** Your LD_PRELOAD `open()` shim doesn't fire for a static binary or
for a program using raw `syscall()`. Explain both gaps — and what each tells
you about which layer LD_PRELOAD operates on.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Static binary: LD_PRELOAD works by making the dynamic loader resolve symbols
to your library first — but a static binary has <em>no dynamic
resolution</em>: libc was linked in at build time (Lesson 46), so its calls
to <code>open</code> are direct jumps to the copied-in code, decided at link
time, unreachable by any runtime interposition. There is no symbol to
intercept. Raw syscall: <code>syscall(SYS_open, ...)</code> invokes the
kernel directly (the instruction from Lesson 02), never calling libc's
<code>open</code> wrapper — your shim replaces the <em>wrapper</em>, but the
program didn't use it. Both gaps locate LD_PRELOAD precisely: it operates at
the <strong>dynamic symbol boundary</strong> — above raw syscalls (which
seccomp/ftrace catch, Lessons 56/64), below application logic (which it
can't see), and entirely absent when there's no dynamic linking. Picking the
right observation tool means knowing which boundary your question lives at:
symbols → LD_PRELOAD/ltrace; syscalls → strace/seccomp; kernel internals →
ftrace.
</details>

**Q2.** Why does the loader ignore LD_PRELOAD for setuid-root binaries, and
what exact attack does that prevent? Connect it to Lesson 11.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A setuid-root binary runs with root's effective UID while launched by an
unprivileged user (Lesson 11): the user controls its <em>environment</em>
but the process gets root's power. If LD_PRELOAD were honored, the attacker
would write a malicious library — say one whose constructor spawns a root
shell, or that interposes a trivial function to run before the program does
its privilege-sensitive work — set <code>LD_PRELOAD=./evil.so</code>, and
run <code>sudo</code> (or passwd, or any setuid-root binary): the loader
would dutifully load evil.so <em>into the root process</em> and run its code
with root privileges. Instant, universal privilege escalation from any
setuid binary on the system. So the loader detects "secure-execution mode"
(auxv AT_SECURE — set by the kernel when real≠effective uid/gid or
capabilities were gained on exec) and strips LD_PRELOAD, LD_LIBRARY_PATH,
and the other dangerous LD_* variables, plus ignores RPATH in insecure
locations. It's the same cliff-guard family as Lesson 11's "setuid is
attack surface — every input becomes hostile": the environment is input,
and the loader must not let attacker-controlled input redirect a privileged
process's code. This one rule is what makes the entire convenient,
unprivileged LD_PRELOAD mechanism safe to ship.
</details>

**Q3.** You need to trace which files a *closed-source, dynamically-linked*
application opens, and separately to force all its network connections
through a proxy. For each task, argue whether LD_PRELOAD or strace is the
better tool and why — including one case where LD_PRELOAD can do something
strace fundamentally cannot.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Tracing file opens</strong>: strace is the better <em>observer</em>
— it catches every open/openat at the syscall boundary regardless of how the
program made the call (libc wrapper, raw syscall, static or dynamic), with
no code to write and no gaps; LD_PRELOAD would miss raw syscalls and
internal openat variants (lab 1's caveat) and requires a shim per function.
For pure observation of a black box, strace wins. <strong>Proxying
connections</strong>: LD_PRELOAD is the standard tool (this is literally how
proxychains/torsocks work) — and here's the case strace <em>cannot</em>
match: LD_PRELOAD can <strong>modify behavior</strong>, not just observe it.
Its interposed <code>connect()</code> can rewrite the destination address to
the proxy, perform the SOCKS handshake, and return a redirected socket — the
application believes it connected to the origin. strace can only watch
syscalls go by (and, with fault injection, fail them) — it cannot
transparently rewrite arguments and results to implement new functionality.
That's the fundamental divide: strace is a read-only window on the syscall
boundary; LD_PRELOAD is read-<em>write</em> on the symbol boundary — so
"tell me what it does" favors strace's completeness, while "change what it
does without its cooperation" is LD_PRELOAD's unique territory (with the
caveat that a program using raw syscalls for connect() would escape the
LD_PRELOAD proxy — at which point you'd need a network namespace + transparent
redirect, the networking-track answer).
</details>

---

## Homework

Combine the phase: write an LD_PRELOAD library that makes any program's
`getenv("HOME")` return `/tmp/fakehome` while leaving all other environment
lookups untouched — then explain why this technique is used in test
harnesses, why it's *not* a security boundary (how a program could defeat
it), and how the same effect could be achieved three other ways from this
phase and earlier (hint: `--wrap`, a wrapper script, a mount namespace).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<pre>
#define _GNU_SOURCE
#include &lt;dlfcn.h&gt;
#include &lt;string.h&gt;
char *getenv(const char *name) {
    static char *(*real)(const char*);
    if (!real) real = dlsym(RTLD_NEXT, "getenv");
    if (strcmp(name, "HOME") == 0) return "/tmp/fakehome";
    return real(name);                       /* everything else: real */
}
</pre>
<code>LD_PRELOAD=./fakehome.so program</code> — the program reads
<code>/tmp/fakehome</code> as its home, useful for <strong>test
harnesses</strong>: run a tool against a clean, disposable HOME without
touching the real one, isolating config/cache side effects (many test
suites do exactly this). <strong>Not a security boundary</strong>: it only
intercepts the libc <code>getenv</code> symbol — a program can defeat it
trivially by reading <code>environ</code> directly (the raw
<code>char **environ</code> array libc parses), calling
<code>secure_getenv</code>, parsing <code>/proc/self/environ</code>, or
using a raw syscall path — none of which route through your symbol; and it's
gone entirely for static binaries and setuid programs (the three gaps). It
<em>redirects a cooperating lookup</em>, it does not <em>confine</em>.
Three other routes to the same effect: (1) <strong>--wrap at build
time</strong> (<code>-Wl,--wrap=getenv</code>) — same interposition,
compiled in, only for code you build (no good for closed binaries but robust
where it applies); (2) <strong>a wrapper script</strong> — the crudest and
often best: <code>HOME=/tmp/fakehome exec program "$@"</code> sets the real
environment variable, which every access method (getenv, environ, /proc)
sees consistently — no symbol games, no gaps, but it changes HOME for the
whole process rather than faking one API; (3) <strong>a mount namespace</strong>
(Lesson 40) — unshare -m and bind-mount /tmp/fakehome over the real home
directory: now HOME's <em>value</em> is unchanged but its <em>contents</em>
are redirected at the filesystem layer, invisible to any environment trick
and confined to the namespace — the strongest isolation, and closest to how
containers give each workload its own view. The progression — symbol → env
var → namespace — is the whole book: pick the layer that matches how much
you need to fake versus how much you need to <em>enforce</em>.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 10 — Boot & Init (Lesson 50: From Firmware to Kernel) →](lesson-50-boot-process){: .btn .btn-primary }
