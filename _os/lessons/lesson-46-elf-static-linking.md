---
title: "Lesson 46 — ELF and Static Linking"
nav_order: 1
parent: "Phase 9: Linking & Loading"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 46: ELF and Static Linking

## Concept

`gcc hello.c -o hello` looks like one step. It's four, and the last one — the
**linker** — is where "a program" is actually assembled:

```
  hello.c ──preprocess──▶ hello.i ──compile──▶ hello.s ──assemble──▶ hello.o
                                                                        │
   an OBJECT file: machine code with HOLES —                            │
   "call printf" = "call <address unknown, ask later>"                  │
                                                                        ▼
              crt1.o + hello.o + libc ──────LINK (ld)──────▶  hello (ELF)
              resolve every hole: each symbol NAME gets an ADDRESS
```

Everything on disk — executables, `.o` files, `.so` libraries, even core
dumps (Lesson 65) — is **ELF** (Executable and Linkable Format), one format
wearing two hats that this lesson teaches you to tell apart:

- **Sections** — the *linker's* view: fine-grained named pieces (`.text`
  code, `.data` initialized globals, `.bss` zero-initialized globals,
  `.rodata` constants, symbol tables, debug info). Dozens of them; they
  exist to be merged and resolved at link time.
- **Segments** (program headers) — the *kernel's* view: coarse load
  instructions — "map this file range at this address with these
  permissions." A handful (`LOAD` r-x, `LOAD` rw-, ...). At `execve`, the
  kernel reads *only* segments — and you've already seen them: the
  `r-xp` / `rw-p` regions of `/proc/PID/maps` (Lesson 18) are LOAD segments,
  mapped.

The linker's core job is **symbol resolution + relocation**: collect every
symbol *definition* (function/variable → location) and *reference* (the
holes), match them by name, patch the holes. `nm` shows a file's symbol
table; an unresolved reference at the end = the famous
`undefined reference to 'foo'` — now a mechanical, diagnosable event rather
than an incantation failure.

---

## How It Works

### Anatomy worth memorizing

`.text` (code, read-only+execute), `.rodata` (string literals — write to one
and segfault, Lesson 16's permissions!), `.data` (globals with initializers
— stored in the file), `.bss` (zero-initialized globals — **not stored**,
just a size: the loader/kernel provides zero pages, Lesson 19's zero page!).
Your Lesson 16 lab variable `value` lived in `.bss`/`.data` at `0x404040` —
that number came from the linker's layout, and `-no-pie` was what made it
fixed.

### Static linking, precisely

`-static` copies the needed objects *out of* `libar.a` archives into the
binary: self-contained (no loader, no .so dependencies — Lesson 03's
homework tinytrue!), fast startup, immune to library-version drift — and
big (a static hello: ~800 KB), memory-unshared (every static binary carries
private libc pages — no Lesson 16 sharing), and security-patch-proof in the
bad sense: a libc CVE means *relinking every binary*, not updating one .so.
Go made static its default (deployment simplicity — Lesson 03 Q3); C
distros went dynamic (share + patch centrally). Both answers are correct
for their constraints.

### What else the linker settles

Duplicate symbols (two definitions of `foo` → error, or weak-symbol
resolution); `--gc-sections` dead code removal; layout (which section lands
at which address — linker scripts, the embedded world's dark art); and the
entry point: not `main` but `_start` (from `crt1.o`), which sets up
argc/argv/environ and *calls* main — the reason Lesson 03's `-nostdlib`
homework had to define `_start` itself.

{: .note }
> **The toolbox**
> <code>file</code> — what is this?; <code>readelf -h/-S/-l</code> —
> header/sections/segments; <code>nm</code> — symbols (<code>U</code> =
> undefined/needed, <code>T</code> = defined in .text);
> <code>objdump -d</code> — disassembly; <code>strings</code> — the
> embarrassing constants; <code>size</code> — section budget. These work on
> any ELF you'll ever meet.

---

## Lab

```bash
# ---- 1. Watch the four stages ----
$ cat > /tmp/hello.c << 'EOF'
#include <stdio.h>
int answer = 42;              /* .data */
int uninit[1000];             /* .bss  */
int main(void) { printf("answer: %d\n", answer); return 0; }
EOF
$ gcc -c /tmp/hello.c -o /tmp/hello.o          # stop before linking
$ file /tmp/hello.o
# ELF 64-bit LSB relocatable    ← "relocatable" = full of holes

# ---- 2. The holes, visible ----
$ nm /tmp/hello.o
# 0000000000000000 D answer     ← Defined, .data
# 0000000000000000 T main       ← defined, .text
#                  U printf     ← UNDEFINED: the hole. someone must supply it
$ objdump -d -r /tmp/hello.o | grep -A1 'call'
#   call ... <main+0x??>  R_X86_64_PLT32  printf-0x4   ← the RELOCATION:
#                          "patch this call site with printf's address"

# ---- 3. Link, and the hole is gone ----
$ gcc /tmp/hello.o -o /tmp/hello && nm /tmp/hello | grep -E ' (main|printf|answer)'
# main and answer have real addresses; printf shows as U still — because
# the DYNAMIC linker will finish that one at runtime (next lesson!)
$ gcc -static /tmp/hello.o -o /tmp/hello_static && nm /tmp/hello_static | grep -w printf
# now printf has an address: libc's code was COPIED IN
$ ls -l /tmp/hello /tmp/hello_static
# ~16KB vs ~800KB — the price of self-containment

# ---- 4. Sections vs segments: two views of one file ----
$ readelf -S /tmp/hello | grep -E 'Name|text|rodata|\.data|\.bss' | head -6
# .text PROGBITS / .rodata PROGBITS / .data PROGBITS / .bss NOBITS
#                                              ^^^^^^ .bss occupies NO file
#                                              space — just a size to zero!
$ readelf -l /tmp/hello | grep -E 'LOAD|Segment' | head -5
# LOAD ... R E     ← segment 1: code+rodata, read+execute
# LOAD ... RW      ← segment 2: data+bss, read+write
$ readelf -l /tmp/hello | grep -A3 'Section to Segment'
# the mapping: which sections ride in which segment — the two views joined

# ---- 5. .bss's free lunch, proven ----
$ python3 -c "print(open('/tmp/hello','rb').read().count(b'\x00'))" >/dev/null
$ size /tmp/hello
#  text   data    bss    dec
#  1500    600   4032         ← 4000 bytes of uninit[] in bss...
$ ls -l /tmp/hello            # ...but the FILE didn't grow by 4KB. NOBITS.

# ---- 6. Segments become maps (L18, closing the loop) ----
$ /tmp/hello & p=$!
$ grep hello /proc/$p/maps
# r-xp = the LOAD RE segment;  rw-p = the LOAD RW segment. execve just
# mmap'd the program headers. The kernel never read a single section.
$ wait

# ---- 7. undefined reference, demystified forever ----
$ cat > /tmp/broken.c << 'EOF'
void mystery_function(void);
int main(void) { mystery_function(); return 0; }
EOF
$ gcc /tmp/broken.c -o /tmp/broken 2>&1 | tail -2
# undefined reference to `mystery_function'   ← a hole nobody filled:
# nm any .o/.a you MEANT to link and look for 'T mystery_function'
$ rm /tmp/hello* /tmp/broken*
```

---

## Further Reading

| Topic | Link |
|---|---|
| ELF (Wikipedia) | <https://en.wikipedia.org/wiki/Executable_and_Linkable_Format> |
| `elf(5)` man page | <https://man7.org/linux/man-pages/man5/elf.5.html> |
| `readelf(1)` man page | <https://man7.org/linux/man-pages/man1/readelf.1.html> |
| `nm(1)` man page | <https://man7.org/linux/man-pages/man1/nm.1.html> |
| Linkers and Loaders (Levine) — classic text | <https://en.wikipedia.org/wiki/Linker_(computing)> |
| `ld(1)` man page | <https://man7.org/linux/man-pages/man1/ld.1.html> |

---

## Checkpoint

**Q1.** What's the difference between a section and a segment, and which
does the kernel care about at exec time? Prove your answer from something
you observed in the lab.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Sections are the linker's inventory: fine-grained, named, typed pieces
(.text/.data/.symtab/debug…) used to merge object files and resolve
symbols — link-time metadata. Segments are load directives: "map file
offset X..Y at virtual address Z with permissions P" — a handful of
coarse LOAD entries grouping compatible sections (code+rodata → R+E;
data+bss → RW). The kernel reads only the program headers (segments) at
execve: it mmaps each LOAD and jumps to the entry point — sections could be
stripped entirely (<code>strip</code> removes .symtab and the binary still
runs!) because the kernel never looks. Lab proof: /proc/PID/maps showed
exactly two file-backed regions matching the two LOAD segments (r-xp,
rw-p) — not dozens matching the sections; and readelf's section-to-segment
map showed the many-to-few packing. One file, two consumers, two tables of
contents.
</details>

**Q2.** `.data` and `.bss` both hold writable globals. Why do they exist as
separate sections, and which of Lesson 19's mechanisms makes `.bss`
effectively free at both file- and load-time?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<code>.data</code> holds globals with initializers — those byte values must
be <em>stored in the file</em> and mapped in (they're arbitrary data only
the file knows). <code>.bss</code> holds zero-initialized globals — storing
four thousand zeros would be waste: the section is marked NOBITS with just
a size, occupying zero file bytes (lab 5: 4 KB of array, no file growth).
At load, the kernel extends the RW mapping by the .bss size backed by
<em>anonymous zero pages</em> — Lesson 19's zero-page + demand-paging
machinery: no physical frames until first write, and reads of untouched
.bss all share the one global zero page. So .bss costs nothing on disk,
nothing at exec, and physical memory only as actually dirtied — which is
why the C standard's "static objects are zero-initialized" guarantee is
free to honor, and why huge static arrays are cheap right up until you
touch them (Lesson 16's VmSize-vs-RSS gap, born here).
</details>

**Q3.** Your company debates shipping its service statically linked vs
dynamically. Argue both sides in operational terms — deployment, memory,
security patching, and the container angle.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Static: one artifact, zero runtime deps — deploys onto any kernel-compatible
host (musl-static runs on Alpine, distroless, scratch containers — no
loader, no libc match needed: Lesson 03 Q3's Go story); no
works-on-my-machine library drift; startup skips the dynamic loader
(Lesson 47's work). Costs: every binary carries its libraries — N services
× libc with no page sharing (Lesson 16's economics reversed — real RAM at
container density); and patching means <em>rebuilding and redeploying every
image</em> on each OpenSSL/libc CVE — your patch latency is your slowest
CI pipeline, and scanners must inspect binaries, not package lists.
Dynamic: central patching (update one .so, restart services — the distro
security model), shared code pages across processes, smaller artifacts.
Costs: dependency management forever (version skew, the loader's search
path as an attack/config surface — Lessons 47–49), and container images
must ship a coherent userland anyway (which erodes the sharing argument:
two containers' identical .so files in different image layers don't share
pages unless the storage driver + page cache dedupe them — overlayfs same-
layer sharing does work, Lesson 39). Modern synthesis: static (or mostly-
static) for hermetic, horizontally-replicated microservices where image
rebuild IS the patch path; dynamic for long-lived hosts, plugin
architectures (dlopen needs it), and anything whose patch cadence must
outrun its rebuild cadence.
</details>

---

## Homework

The linker resolves symbols left-to-right with archives: `gcc main.o -lfoo
-lbar` works, but `gcc -lfoo -lbar main.o` may fail with undefined
references *even though the same objects are present*. Explain the archive
extraction rule that causes this, construct a two-archive circular
dependency that no single ordering fixes, and name the linker flag that
resolves it (and why it's not the default).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Rule: the linker processes inputs strictly left to right, maintaining a set
of currently-undefined symbols; from an archive (.a) it extracts
<em>only</em> the members that satisfy a symbol already in that set, then
moves on and never looks back. With <code>-lfoo</code> first, no symbols
are wanted yet → nothing extracted → main.o then adds unmet references →
too late. (Position matters for .o files differently: they're always
included.) Circular construction: liba.a's <code>a1.o</code> defines
<code>alpha()</code> but calls <code>beta()</code>; libb.a's
<code>b1.o</code> defines <code>beta()</code> but calls <code>gamma()</code>;
liba.a's <code>a2.o</code> defines <code>gamma()</code>. Ordering
<code>main.o -la -lb</code>: alpha pulled, beta wanted → -lb pulls b1,
gamma wanted → but -la is already passed: undefined gamma. Ordering
<code>-lb -la</code> fails symmetrically for beta. Fix:
<code>-Wl,--start-group -la -lb -Wl,--end-group</code> — the linker loops
over the group until no new members are extracted (a fixpoint iteration).
Not default because it's O(N×passes) on large archives and — the social
reason — it hides genuine layering violations: circular library
dependencies are usually an architecture smell the one-pass rule forces
you to confront (or at least document with an explicit group). Modern
mitigations: <code>ld.gold/lld</code> are faster so groups hurt less, and
shared libraries sidestep it entirely (their symbols resolve lazily at
runtime — next lesson).
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 47 — The Dynamic Loader →](lesson-47-dynamic-loader){: .btn .btn-primary }
