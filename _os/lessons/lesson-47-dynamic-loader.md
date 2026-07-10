---
title: "Lesson 47 — The Dynamic Loader"
nav_order: 2
parent: "Phase 9: Linking & Loading"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 47: The Dynamic Loader

## Concept

Lesson 46 left `printf` unresolved in the dynamic binary. Someone must
finish the job at runtime — and it isn't the kernel. Meet the busiest
program you've never run deliberately: **the dynamic loader**,
`/lib64/ld-linux-x86-64.so.2`.

The mechanism is elegant: every dynamic ELF names an **interpreter** in a
special segment (`PT_INTERP`). At `execve`, the kernel maps your program,
*maps the interpreter too, and starts execution there* — the loader runs
first, inside your process, before one instruction of yours:

```
  execve("/tmp/hello")
     kernel: map hello's LOAD segments (L46)
             read PT_INTERP → map /lib64/ld-linux-x86-64.so.2
             jump to the LOADER's entry point
        │
        ▼  the loader, in YOUR process:
        1. read hello's DT_NEEDED list:  libc.so.6
        2. FIND each library (the search order — below)
        3. mmap each library's LOAD segments (shared, r-x — L16!)
        4. resolve symbols across everything (relocations)
        5. run initializers, THEN jump to _start → main
```

Between execve and main, an entire linker ran. That's the ~25 startup
syscalls of Lesson 02's `strace -c true`, the mmaps of libc you keep seeing,
and the reason a "not found" error can appear for a file that *exists*
(Lesson 03 Q3 — the missing thing was the interpreter or a DT_NEEDED
library, and now you know who prints the message: the loader itself).

---

## How It Works

### The search order (memorize — it's a debugging staple *and* an attack surface)

For each DT_NEEDED name, in order:

1. `DT_RPATH` (legacy, baked into the binary) — unless RUNPATH exists
2. **`LD_LIBRARY_PATH`** (environment — the debugging/deploy knob)
3. `DT_RUNPATH` (baked in, the modern rpath: `-Wl,-rpath,...`)
4. **`/etc/ld.so.cache`** — a mmap-able index built by `ldconfig` from
   `/etc/ld.so.conf*` (so the loader doesn't walk directories)
5. Defaults: `/lib`, `/usr/lib` (and 64-bit variants)

`ldd binary` shows the resolution result (careful: classic ldd *executes*
the loader in trace mode — don't ldd untrusted binaries; use
`objdump -p | grep NEEDED`). `LD_DEBUG=libs ./prog` narrates every step of
the hunt — the single best tool for "why is it loading THAT one?"

### Lazy binding: the PLT and GOT

Resolving *every* function upfront would slow startup (libc exports
thousands). So function calls go through a pair of tables: the **PLT**
(procedure linkage table — tiny code stubs) and the **GOT** (global offset
table — addresses, writable). First call to `printf`: the PLT stub jumps
through a GOT slot that initially points *back into the loader*, which
resolves `printf` now, patches the GOT slot, and jumps there; every later
call flows straight through the patched slot. Pay-per-use resolution — and
visible in a debugger as `printf@plt`.

Two switches every engineer should know: `LD_BIND_NOW=1` (resolve
everything at startup — trades startup time for no mid-run resolution
stalls, and surfaces missing symbols immediately) and **full RELRO**
(`-Wl,-z,relro,-z,now`): bind now, then mprotect the GOT read-only — closing
the GOT-overwrite exploit class (a writable table of code pointers is a
attacker's favorite — Lesson 16's page permissions as hardening).

### PIE and ASLR, joined up

Modern executables are **PIE** (position-independent executables): the
program itself is relocatable like a library, so ASLR (Lesson 16) can place
*everything* — binary, libraries, stack, heap — at random addresses. That's
why Lesson 16 needed `-no-pie` to pin `0x404040`, and why library code is
compiled `-fPIC`: code that works at any address is what makes one physical
copy shareable at different virtual addresses in every process
(Lesson 16 Q1's caveat, resolved).

{: .note }
> **The loader is configurable per binary — and that's a feature and a risk**
> <code>patchelf --set-interpreter --set-rpath</code> rewrites these fields
> (how Nix/prebuilt-binary distributors relocate software). The same
> knobs are the attack surface: writable RPATH directories,
> LD_LIBRARY_PATH in privileged contexts. The loader ignores LD_* for
> setuid binaries (Lesson 11's cliff-guard — "secure-execution mode"),
> which Lesson 49 revisits.

---

## Lab

```bash
# ---- 1. Meet the interpreter ----
$ readelf -l /bin/cat | grep -A1 INTERP
# [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
$ /lib64/ld-linux-x86-64.so.2 --list /bin/cat    # the loader is a program:
# linux-vdso.so.1 / libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 ...
$ /lib64/ld-linux-x86-64.so.2 /bin/echo "loader ran me directly"
# you can EXECUTE a program via its interpreter — proof of who's in charge

# ---- 2. Narrate the hunt: LD_DEBUG ----
$ LD_DEBUG=libs /bin/true 2>&1 | head -12
# find library=libc.so.6; searching
#  search cache=/etc/ld.so.cache
#   trying file=/lib/x86_64-linux-gnu/libc.so.6
# the search order, live. Try LD_DEBUG=help for the full menu
# (bindings shows every symbol resolution — thousands of lines!)

# ---- 3. Lazy binding: catch the GOT being patched ----
$ cat > /tmp/lazy.c << 'EOF'
#include <stdio.h>
int main(void) {
    printf("first call\n");         /* resolves printf NOW (via loader) */
    printf("second call\n");        /* straight through the patched GOT */
    return 0;
}
EOF
$ gcc -o /tmp/lazy /tmp/lazy.c
$ LD_DEBUG=bindings /tmp/lazy 2>&1 | grep -m2 "printf"
# binding file /tmp/lazy ... to libc: normal symbol `printf'  ← ONCE
$ objdump -d /tmp/lazy | grep -A3 'printf@plt>:'
#  jmp *0x...(%rip)   ← the PLT stub jumping through its GOT slot

# ---- 4. LD_BIND_NOW and RELRO ----
$ LD_BIND_NOW=1 LD_DEBUG=bindings /tmp/lazy 2>&1 | grep -c binding
# many more bindings, all BEFORE main runs
$ checksec --file=/tmp/lazy 2>/dev/null || readelf -d /tmp/lazy | grep -E 'BIND_NOW|FLAGS'
$ gcc -Wl,-z,relro,-z,now -o /tmp/hardened /tmp/lazy.c
$ readelf -d /tmp/hardened | grep -E 'BIND_NOW|FLAGS'
# FLAGS: NOW           ← full RELRO: GOT resolved then sealed read-only

# ---- 5. The search order as a deployment tool ----
$ mkdir -p /tmp/mylibs
$ cat > /tmp/greet.c << 'EOF'
#include <stdio.h>
void greet(void) { printf("greetings from v1\n"); }
EOF
$ gcc -shared -fPIC -o /tmp/mylibs/libgreet.so /tmp/greet.c
$ cat > /tmp/app.c << 'EOF'
void greet(void);
int main(void) { greet(); return 0; }
EOF
$ gcc /tmp/app.c -L/tmp/mylibs -lgreet -o /tmp/app
$ /tmp/app 2>&1 | tail -1
# error while loading shared libraries: libgreet.so: cannot open...
#   ← -L is LINK-time only! runtime search knows nothing of /tmp/mylibs
$ LD_LIBRARY_PATH=/tmp/mylibs /tmp/app          # fix 1: environment
$ gcc /tmp/app.c -L/tmp/mylibs -lgreet -Wl,-rpath,/tmp/mylibs -o /tmp/app2
$ /tmp/app2 && readelf -d /tmp/app2 | grep RUNPATH   # fix 2: baked-in RUNPATH

# ---- 6. PIE + ASLR: everything moves (L16, completed) ----
$ for i in 1 2; do bash -c 'grep -m1 -E "bin/bash" /proc/$$/maps'; done
# the BINARY's own base address differs run to run — PIE made the
# executable itself relocatable, so ASLR randomizes it like a library
$ rm -f /tmp/lazy* /tmp/hardened /tmp/app* /tmp/greet.c; rm -rf /tmp/mylibs
```

---

## Further Reading

| Topic | Link |
|---|---|
| `ld.so(8)` — the dynamic loader | <https://man7.org/linux/man-pages/man8/ld.so.8.html> |
| Dynamic linker (Wikipedia) | <https://en.wikipedia.org/wiki/Dynamic_linker> |
| `ldconfig(8)` man page | <https://man7.org/linux/man-pages/man8/ldconfig.8.html> |
| PLT and GOT — position-independent code | <https://en.wikipedia.org/wiki/Position-independent_code> |
| `ldd(1)` man page (and its security note) | <https://man7.org/linux/man-pages/man1/ldd.1.html> |
| Address space layout randomization | <https://en.wikipedia.org/wiki/Address_space_layout_randomization> |

---

## Checkpoint

**Q1.** `execve` succeeded but the program printed "error while loading
shared libraries". Who printed that — the kernel, libc, or something else —
and what already happened / never happened at that instant?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The dynamic loader printed it. Sequence: the kernel's execve succeeded
fully — it mapped the program's LOAD segments, saw PT_INTERP, mapped
<code>ld-linux</code>, and transferred control to the loader; the old
program image is gone (Lesson 07: exec's point of no return). The loader
then walked DT_NEEDED, ran the search order, and failed to find a library —
so it wrote the error to stderr and exited. libc never entered the picture:
it's typically the thing that <em>couldn't be loaded</em>, and the loader
is deliberately self-contained (statically linked within itself) so it can
report exactly this failure. Never happened: no relocations for your code,
no initializers, no _start, no main — your program's first instruction was
never reached. Debugging follows the mechanism: <code>objdump -p | grep
NEEDED</code> for what's wanted, <code>LD_DEBUG=libs</code> for where it
looked, then fix via ldconfig/RUNPATH/LD_LIBRARY_PATH (Lab 5's ladder).
</details>

**Q2.** Explain lazy binding's PLT/GOT dance and the security trade it
creates — why do hardened builds choose `-z now` + RELRO, and what exactly
does each half close?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Dance: a call to printf compiles as a call to the local
<code>printf@plt</code> stub; the stub jumps through its GOT slot. Initially
that slot points at the loader's resolver: first call detours into the
loader, which finds printf in libc, writes libc's real address into the GOT
slot, and jumps on; thereafter calls flow stub→GOT→libc with no loader —
pay-per-symbol resolution amortizing startup across use. The trade: the GOT
must stay <em>writable for the process's whole life</em> — a table of
function pointers, at a findable location, that an attacker with any write
primitive can redirect ("GOT overwrite": point free() at system()). The
hardened pair: <code>-z now</code> resolves every symbol at startup, so no
future writes are needed; RELRO then <code>mprotect</code>s the
GOT/relocation area read-only (Lesson 16's page permissions as a lock).
Each half alone is incomplete: now-binding without RELRO leaves the table
writable (pointlessly); RELRO without now-binding could seal only part
(partial RELRO — the common default — protects some sections but the lazy
GOT stays writable). Cost: slower startup (all symbols now), which servers
happily pay; the attack class removed is one of exploitation's classic
stepping stones.
</details>

**Q3.** A vendor ships you a prebuilt binary that must run on hosts where
you can't install libraries system-wide or set environment variables (it's
launched by systemd units you don't control). Rank the mechanisms available
to make its libraries resolvable, with the trade-off of each.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <strong>patchelf --set-rpath '$ORIGIN/../lib'</strong> — bake a RUNPATH
relative to the binary's own location ($ORIGIN expands at load time):
self-contained, relocatable directory tree, no host mutation, survives any
launcher. Trade: you're modifying a vendor artifact (checksums/signatures
break; re-do on updates) — this is precisely how Nix and many vendor
tarballs ship. (2) <strong>ldconfig drop-in</strong>
(/etc/ld.so.conf.d/vendor.conf + ldconfig) — clean, systemic, survives
everything. Trade: requires root and is host-global: you've made the
vendor's libraries visible to every process (version-conflict and
attack-surface implications — exactly what you were told not to do,
usually). (3) <strong>wrapper script</strong> exporting LD_LIBRARY_PATH
then exec'ing the binary — no artifact or host changes. Trade: you said
you can't control the unit... but you can often point it at the wrapper;
fragile (anything exec'ing the binary directly bypasses it), and the
variable leaks to children. (4) <strong>containerize it</strong> — its
whole userland travels along (Lesson 46 Q3's logic). Trade: heaviest
machinery for one binary. Ranked for the stated constraints: patchelf ≥
container > ldconfig > wrapper. The meta-skill: each mechanism is one rung
of the loader's search order (Lab 5) — deployment engineering is choosing
which rung you're allowed to own.
</details>

---

## Homework

`LD_DEBUG=statistics /bin/true` prints the loader's own accounting of
startup work. Run it, then compare against a heavyweight binary on your VM
(python3, or anything Qt/GTK). Report: total relocations processed, time
share of relocation vs load, and the effect of `LD_BIND_NOW=1` on each.
Then explain why prelink died as a technique (what ASLR did to it) and what
glibc's modern answer is (hint: `ld.so --list-diagnostics` mentions it —
the "dl cache"/DT_RELR story).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Typical findings: /bin/true — a few hundred relocations, loader time
dominated by mmap/open of libc, relocation share small;
python3 — thousands of relocations across a dozen libraries (readline,
ssl, m…), relocation processing becoming a visible fraction (loader
statistics show "time needed for relocation: …%"); with LD_BIND_NOW=1 both
process <em>all</em> function relocations up front — true's numbers barely
move (tiny import set), python3's relocation count and time jump
noticeably: lazy binding was deferring real work proportional to API
surface. Prelink's idea was to precompute relocations at package-install
time by assigning every library a fixed address — which is exactly what
ASLR forbids: randomized load addresses invalidate precomputed absolute
relocations by design, so prelink either disabled ASLR (unacceptable) or
its work was discarded (useless); it was abandoned. Modern answers attack
the cost, not the randomness: <strong>DT_RELR</strong> — a compact
encoding for the dominant class of relocations (relative ones, which
depend only on the load base) shrinking both file size and processing
time; the ld.so cache mapping (avoid path searching); and PIE-friendly
GOT/PLT layouts + <code>-z now</code>+RELRO as the standard hardened
combo whose startup cost DT_RELR helps repay. Theme: the loader keeps
per-symbol work, drops per-address assumptions — everything position-
relative survives ASLR; anything absolute died with prelink.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 48 — Shared Libraries and Versioning →](lesson-48-shared-libraries){: .btn .btn-primary }
