---
title: "Lesson 48 — Shared Libraries and Versioning"
nav_order: 3
parent: "Phase 9: Linking & Loading"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 48: Shared Libraries and Versioning

## Concept

Lesson 47 loaded `libc.so.6`. That `.6` is not decoration — it's a promise
about compatibility, and this lesson is about how thousands of programs
share libraries that keep evolving without the world shattering on every
update.

The famous **three names** confuse everyone until you see the job each does:

```
   /usr/lib/libssl.so.3.0.2        REAL NAME — the actual file, full version
        ▲   symlink
   /usr/lib/libssl.so.3            SONAME — the ABI-compatibility promise
        ▲   symlink                        ("anything that worked with
   /usr/lib/libssl.so              LINKER NAME  any .so.3 works with me")
                                            — dev-only, what -lssl finds

   at BUILD time:  ld follows libssl.so → records the SONAME (libssl.so.3)
                   into your binary's DT_NEEDED
   at RUN time:    loader wants "libssl.so.3" → finds it via the symlink
                   → whichever 3.x.y is installed. Patch 3.0.2→3.0.9:
                   just repoint the symlink. Your binary never noticed.
```

The pivot concept is **ABI (Application Binary Interface) compatibility** —
distinct from API (source) compatibility:

- **API** = source compiles (function names, signatures at the C level).
- **ABI** = compiled binary keeps working: struct layouts, sizes,
  calling conventions, symbol names — the *binary* contract Lesson 03 named.

A library can break API but keep ABI, or vice versa. The **SONAME bump**
(`.so.3` → `.so.4`) is the maintainer declaring "ABI broke — old binaries
must NOT load me": the two versions coexist on disk, each binary finds its
own, no crash. glibc's near-legendary stability comes from *never* bumping
its SONAME (still `libc.so.6` since 1997) — achieved through **symbol
versioning** (below), which lets one file serve twenty years of binaries.

---

## How It Works

### Building one, correctly

`gcc -shared -fPIC -Wl,-soname,libfoo.so.1 -o libfoo.so.1.2.3 …`, then
symlinks `libfoo.so.1 → libfoo.so.1.2.3` (runtime, via ldconfig) and
`libfoo.so → libfoo.so.1` (dev). `-fPIC` is mandatory (position-independent
— Lesson 47: one physical copy at any virtual address); the `-soname`
records the compatibility promise into the file. `readelf -d` shows a
library's own SONAME and a binary's DT_NEEDED — the two ends of the contract.

### Symbol versioning: glibc's superpower

How does one `libc.so.6` provide both old and new `realpath` when their
behavior changed? **Versioned symbols**: the library exports
`realpath@GLIBC_2.3` *and* `realpath@@GLIBC_2.30` (the `@@` = default for
new links). A binary built in 2010 recorded a dependency on the old
version; it keeps getting the old behavior from the *same* modern file;
new builds get the new one. `objdump -T libc.so.6 | grep GLIBC_` shows the
version zoo. This is why "compiled on newer glibc, won't run on older"
(forward references to symbol versions that don't exist yet) but the reverse
works — the asymmetry every CI matrix eventually hits.

### dlopen: libraries as runtime plugins

`dlopen("plugin.so", RTLD_NOW)` + `dlsym(handle, "entry")` loads a library
*after* startup and looks up symbols by string — the plugin architecture
(nginx modules, Python C extensions, Postgres extensions, codecs). It's the
runtime face of everything above (search order, versioning, relocation all
apply) and the reason static linking can't be universal: `dlopen` inherently
needs the dynamic loader. RTLD_LAZY vs RTLD_NOW mirror Lesson 47's binding
choice; RTLD_GLOBAL vs LOCAL controls whether the plugin's symbols join the
global namespace (interposition implications — Lesson 49).

{: .note }
> **The version-mismatch error, decoded**
> <code>./app: /lib/libc.so.6: version 'GLIBC_2.38' not found (required by
> ./app)</code> means: app was built against a glibc that added a symbol
> version 2.38; this host's glibc is older and doesn't provide it. Not a
> missing file (Lesson 47's error) — a missing <em>version</em> within a
> present file. Fixes: build on the oldest target glibc, static-link, or
> ship in a container with a matching userland (Lesson 46 Q3's recurring
> answer).

---

## Lab

```bash
# ---- 1. The three names, live ----
$ ls -l /usr/lib/x86_64-linux-gnu/libssl.so.3* 2>/dev/null || ls -l /lib/x86_64-linux-gnu/libc.so.6
$ readlink -f /lib/x86_64-linux-gnu/libc.so.6   # chase to the real file
$ readelf -d /lib/x86_64-linux-gnu/libc.so.6 | grep SONAME
#  SONAME  libc.so.6         ← the promise baked into the file itself

# ---- 2. Build a shared library the right way ----
$ cat > /tmp/mathlib.c << 'EOF'
int square(int x) { return x * x; }
EOF
$ gcc -shared -fPIC -Wl,-soname,libmath.so.1 -o /tmp/libmath.so.1.0.0 /tmp/mathlib.c
$ ln -sf /tmp/libmath.so.1.0.0 /tmp/libmath.so.1     # runtime name
$ ln -sf /tmp/libmath.so.1 /tmp/libmath.so           # linker name
$ readelf -d /tmp/libmath.so.1.0.0 | grep SONAME     # SONAME: libmath.so.1
$ cat > /tmp/user.c << 'EOF'
#include <stdio.h>
int square(int);
int main(void){ printf("square(7)=%d\n", square(7)); return 0; }
EOF
$ gcc /tmp/user.c -L/tmp -lmath -Wl,-rpath,/tmp -o /tmp/user && /tmp/user
$ readelf -d /tmp/user | grep NEEDED | grep math
#  NEEDED  libmath.so.1     ← recorded the SONAME, NOT the real filename!

# ---- 3. Patch the library under a running program's feet ----
$ cat > /tmp/mathlib2.c << 'EOF'
int square(int x) { return x * x + 1000; }   /* "bugfix" — same ABI */
EOF
$ gcc -shared -fPIC -Wl,-soname,libmath.so.1 -o /tmp/libmath.so.1.0.1 /tmp/mathlib2.c
$ ln -sf /tmp/libmath.so.1.0.1 /tmp/libmath.so.1     # repoint the SONAME link
$ /tmp/user
# square(7)=1049      ← picked up 1.0.1 with ZERO relinking. The symlink
#                       indirection IS the patch mechanism (openssl updates!)

# ---- 4. glibc symbol versioning, exposed ----
$ objdump -T /lib/x86_64-linux-gnu/libc.so.6 | grep -E 'GLIBC_2\.(3|34)' | head -5
# realpath ... GLIBC_2.3 ... / ... @@GLIBC_2.34 ...  ← same name, two versions
$ objdump -T /tmp/user | grep GLIBC | head -3
# YOUR binary's recorded version deps — the max determines your min glibc
$ echo 'int main(){return 0;}' | gcc -x c - -o /tmp/tiny
$ objdump -T /tmp/tiny | grep -oE 'GLIBC_[0-9.]+' | sort -V | tail -1
# the newest glibc version this binary requires = its portability floor

# ---- 5. dlopen: a plugin at runtime ----
$ cat > /tmp/plugin.c << 'EOF'
#include <stdio.h>
void run(void) { printf("plugin loaded and executed at runtime\n"); }
EOF
$ gcc -shared -fPIC -o /tmp/plugin.so /tmp/plugin.c
$ cat > /tmp/host.c << 'EOF'
#include <dlfcn.h>
#include <stdio.h>
int main(void) {
    void *h = dlopen("/tmp/plugin.so", RTLD_NOW);
    if (!h) { printf("dlopen: %s\n", dlerror()); return 1; }
    void (*run)(void) = dlsym(h, "run");   /* symbol by STRING */
    run();
    dlclose(h);
    return 0;
}
EOF
$ gcc /tmp/host.c -ldl -o /tmp/host && /tmp/host
# plugin loaded and executed at runtime — no NEEDED entry, chosen at runtime
$ readelf -d /tmp/host | grep -c plugin    # 0 — the binary never declared it

# ---- 6. The version-not-found error, on purpose ----
$ objdump -T /tmp/user | grep -oE 'GLIBC_[0-9.]+' | sort -V | tail -1
# imagine this host had older glibc: the loader would refuse with
# "version `GLIBC_x.yz' not found" — a version gap, not a file gap
$ rm -f /tmp/mathlib* /tmp/libmath* /tmp/user* /tmp/tiny /tmp/plugin* /tmp/host*
```

---

## Further Reading

| Topic | Link |
|---|---|
| Shared library / soname | <https://en.wikipedia.org/wiki/Library_(computing)#Shared_libraries> |
| Application binary interface | <https://en.wikipedia.org/wiki/Application_binary_interface> |
| Symbol versioning — glibc / ld docs | <https://sourceware.org/glibc/wiki/GLIBC_2.29_symbol_versioning> |
| `dlopen(3)` man page | <https://man7.org/linux/man-pages/man3/dlopen.3.html> |
| How to write shared libraries (Drepper) | <https://akkadia.org/drepper/dsohowto.pdf> |
| `ldconfig(8)` man page | <https://man7.org/linux/man-pages/man8/ldconfig.8.html> |

---

## Checkpoint

**Q1.** Why do library files have three names (libfoo.so → libfoo.so.1 →
libfoo.so.1.2.3), and who uses each — at build time and run time?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Each name serves a different actor at a different time. <strong>Linker name</strong>
(<code>libfoo.so</code>, no version): only the <em>developer's</em> link step
uses it — <code>-lfoo</code> finds it, follows the symlink chain, and records
what it finds. It's shipped only in -dev packages precisely because runtime
doesn't need it. <strong>SONAME</strong> (<code>libfoo.so.1</code>): the
<em>compatibility promise</em> — the link step reads the real file's embedded
SONAME and stamps <em>that</em> into your binary's DT_NEEDED, so your binary
asks for "libfoo.so.1" (an ABI generation), never a specific patch level. At
<em>run</em> time the loader resolves "libfoo.so.1" via this symlink to
whatever patch version is installed. <strong>Real name</strong>
(<code>libfoo.so.1.2.3</code>): the actual file with full version. The payoff
(lab 3): security/bugfix updates ship 1.2.4, repoint the SONAME symlink, and
every installed binary transparently uses it on next run — no relinking —
because binaries bind to the generation (SONAME), not the file. A
compatibility-breaking release ships as SONAME .so.2, coexisting on disk, and
old binaries keep finding .so.1: no forced flag-day.
</details>

**Q2.** glibc has kept the SONAME `libc.so.6` since 1997 while adding
thousands of features and changing behaviors. Explain the mechanism that
makes this possible, and the practical asymmetry it creates for
build-vs-run compatibility.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Symbol versioning. Rather than one definition per function, libc exports
version-tagged definitions — <code>foo@GLIBC_2.2</code>,
<code>foo@@GLIBC_2.28</code> (the <code>@@</code> marks the default for new
links) — and each is a genuinely separate implementation kept alive
forever. A binary links against whatever version was default at build time,
recording that dependency; the same modern libc file serves it the old
behavior while new builds get the new one. Because no <em>existing</em>
symbol version is ever removed or changed, every binary from 1997 onward
still finds exactly the contract it was built against in today's file —
hence never needing a SONAME bump. The asymmetry: a binary requires
<em>at least</em> the glibc version of the newest symbol version it uses.
Build on new glibc → your binary references, say, <code>GLIBC_2.38</code>
symbols → it <em>cannot</em> run on a host with older glibc (the version
literally isn't in that file — the "version not found" error). Build on old
glibc → runs everywhere newer (old versions persist). So the portable move
is building against the <em>oldest</em> glibc you must support — the reason
release-engineering uses ancient build images or static linking, and why
"works on my new laptop, fails on the old server" is a weekly occurrence.
</details>

**Q3.** A plugin loaded via `dlopen(..., RTLD_LOCAL)` works, but the same
plugin loaded `RTLD_GLOBAL` mysteriously changes the behavior of the host
program's *unrelated* functions. Explain using symbol namespaces — and why
this is Lesson 49's topic knocking.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
RTLD_LOCAL keeps the plugin's exported symbols out of the process's global
symbol scope — they're reachable only through the dlopen handle
(dlsym) — so the plugin can define its own <code>parse()</code> or even
<code>malloc()</code> without colliding with anyone. RTLD_GLOBAL merges the
plugin's symbols <em>into the global scope</em>, where the loader's symbol
resolution can now pick them up for <em>other</em> libraries' unresolved
references — and resolution follows load order / scope rules, so a
plugin's <code>malloc</code> can start satisfying calls from libraries
loaded later (or lazily-bound calls from anywhere), silently replacing the
intended implementation: the host's unrelated functions now call the
plugin's version. That's <strong>symbol interposition</strong> — the same
mechanism that makes LD_PRELOAD work (Lesson 49), here triggered
accidentally by namespace choice. The defensive practice: plugins use
RTLD_LOCAL and hidden visibility (<code>-fvisibility=hidden</code>,
exporting only their documented entry points) so they can't clobber the
host; frameworks that <em>want</em> plugins to share symbols (some
interpreter C-extension systems) use RTLD_GLOBAL deliberately and accept
the namespace discipline it demands. One flag, two philosophies — and a
whole class of "impossible" bugs when it's set without understanding scope.
</details>

---

## Homework

The C++ ABI (name mangling, vtable layout, exception tables) breaks far more
easily than C's — which is why C++ libraries version-bump aggressively and
why the standard advice for stable plugin ABIs is "expose a C interface even
from a C++ implementation." Using this lesson: explain why `std::string`
changing its internal layout (the real "libstdc++ dual ABI" event of GCC 5)
was an ABI break invisible at the source level, how symbol versioning
(`abi:cxx11`) let one libstdc++ serve both, and why a C `extern "C"`
boundary sidesteps the whole problem.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
C++ encodes structure into the binary far beyond C: function names are
<em>mangled</em> to include argument types (overloading), and — the crux —
callers embed assumptions about type <em>layout</em> (a struct's field
offsets, size, which registers hold what). When GCC 5 changed
<code>std::string</code>'s internals (C++11 forbade copy-on-write strings,
forcing a new layout), the source was unchanged — you still write
<code>std::string s</code> — but every compiled function taking or returning
a string now expected a <em>different-sized, different-shaped</em> object:
a caller compiled old passing a string to a callee compiled new reads fields
at wrong offsets → silent corruption or crash, with identical source. That's
a pure ABI break, invisible to the compiler front-end. Symbol versioning
rescued it: libstdc++ mangles the two string flavors into <em>distinct
symbol names</em> via the <code>[abi:cxx11]</code> tag, so
<code>function(old_string)</code> and <code>function(new_string)</code> are
different symbols coexisting in one library — a binary resolves to whichever
matches how it was compiled (the <code>_GLIBCXX_USE_CXX11_ABI</code> macro
picks at build time), and mixing them becomes a link error rather than a
runtime disaster. The <code>extern "C"</code> boundary sidesteps everything
because it forbids exactly the fragile parts: no mangling (plain symbol
names — the SONAME/versioning tools work as for C), and you pass only C
types (pointers, ints, opaque handles) whose layout is nailed down by the
platform C ABI (the stable contract of Lesson 03) — no templates, no
classes, no exceptions crossing the line. So stable-plugin advice is: C++
inside, C at the seam — the interface inherits C's decades-stable ABI while
the implementation enjoys C++, which is why nearly every long-lived plugin
system (and every FFI target) speaks C at its boundary.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 49 — LD_PRELOAD and Symbol Interposition →](lesson-49-ld-preload){: .btn .btn-primary }
