---
title: "Lesson 58 — Build and Boot Your Own Kernel"
nav_order: 5
parent: "Phase 11: Kernel Interfaces & Security"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 58: Build and Boot Your Own Kernel

## Concept

**Phase 11 capstone.** You've written kernel modules (54–55), and studied every
subsystem the kernel runs. This capstone demystifies the kernel *itself*: you'll configure, build,
and boot your own kernel in QEMU — turning "the kernel" from a magic binary
into a program you compiled.

```
   git clone the source  ──▶  make menuconfig  ──▶  make -j  ──▶  vmlinuz
        (or the tarball)      (choose features:      (compile,      +
                              drivers =y or =m,      ~minutes-       modules
                              scheduler options,     hours)
                              debug flags — every
                              CONFIG_ you've met!)
                                    │
                                    ▼
              boot it in QEMU (virt L12) with your Phase 10 initramfs (L51)
                                    │
                              your kernel prints "Linux version ...(you@host)"
```

This ties the whole course together: the config options are the subsystems
you studied (`CONFIG_PSI` from Lesson 15, `CONFIG_TRANSPARENT_HUGEPAGE` from
Lesson 23, `CONFIG_IO_URING` from Lesson 44, the schedulers, the LSMs from
Lesson 57, the filesystems from Phase 7). Building it makes concrete that the
kernel is ~30M lines of C compiled into the `vmlinuz` the bootloader loads
(Lesson 50) — and that "compiling a kernel" is just a very large `make`. It's
also the foundation for kernel development, debugging (bisecting — below), and
understanding exactly what's in the kernel you run.

---

## How It Works

### Config: the kernel's ./configure, but enormous

The kernel has ~15,000 config options. You never set them by hand — you start
from a working config and adjust:

- **`make menuconfig`** — the ncurses menu; search with `/` (find `PSI`, see
  it's `CONFIG_PSI`, read its help — every option is documented inline).
- Each option is `y` (built into vmlinuz), `m` (a loadable module — Lesson 54),
  or `n` (excluded). The `=y`/`=m` distinction is Lesson 51's initramfs story:
  built-in drivers need no initramfs; modular ones do.
- **Start from your running config**: `zcat /proc/config.gz` or
  `/boot/config-$(uname -r)` → `.config`, then `make olddefconfig` to fill new
  options with defaults. For a *fast lab build*, `make tinyconfig` or
  `make defconfig` gives a minimal bootable kernel in far less time.

### Building and booting

`make -j$(nproc)` compiles (parallelism from Lesson 12/24 — a kernel build is
the canonical CPU-bound multicore workload). Output: `arch/x86/boot/bzImage`
(the compressed kernel — Lesson 50's self-decompressing image) and modules.
Then boot it in QEMU without installing it system-wide:
`qemu-system-x86_64 -kernel bzImage -initrd yourinitramfs ...` (Lesson 51's
lab, now with *your* kernel). It prints its version string with *your*
username and build host — proof you compiled the thing running.

### Bisecting: debugging as a superpower

`git bisect` is why kernel development is tractable: a bug appeared somewhere
in 10,000 commits between "worked" and "broken"; bisect does binary search
(Lesson 12's O(log n)!) — build+boot+test the midpoint, tell git "good" or
"bad," repeat ~13 times to pinpoint the exact commit that introduced it.
Combined with the build skills here, it's how kernel regressions are hunted —
and understanding it demystifies how the kernel's quality is maintained across
thousands of contributors.

{: .note }
> **Realistic expectations for a self-study lab**
> A full <code>defconfig</code> kernel build takes 20–90 minutes and needs
> ~20 GB disk + build tools; a <code>tinyconfig</code> is much faster. The
> QEMU <code>-kernel</code> boot skips firmware/bootloader (QEMU acts as the
> loader), so you can boot your kernel in seconds without touching your VM's
> real boot setup — the safe way to experiment. You do NOT need to install a
> custom kernel on your actual VM to complete this lesson; boot it in QEMU.

---

## Lab

```bash
# ---- 0. Tools + source (this is a big download/build — plan accordingly) ----
$ sudo apt install -y build-essential libncurses-dev bison flex libssl-dev \
    libelf-dev bc git 2>/dev/null | tail -1
$ mkdir -p /tmp/kbuild && cd /tmp/kbuild
# option A (fast, small): shallow clone one stable version
$ git clone --depth 1 --branch v6.8 \
    https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git 2>/dev/null \
  || echo "(or: wget a linux-6.x.tar.xz from kernel.org and untar)"
$ cd linux 2>/dev/null || cd /tmp/kbuild/linux*

# ---- 1. Config: see that CONFIG_ options ARE the course ----
$ make tinyconfig >/dev/null 2>&1        # minimal starting point (FAST build)
$ ./scripts/config --enable 64BIT --enable PRINTK --enable BLK_DEV_INITRD \
    --enable TTY --enable SERIAL_8250 --enable SERIAL_8250_CONSOLE \
    --enable BINFMT_ELF --enable PROC_FS --enable SYSFS 2>/dev/null
$ make olddefconfig >/dev/null 2>&1
# search for options you studied:
$ grep -E 'CONFIG_(PSI|IO_URING|TRANSPARENT_HUGEPAGE|SECCOMP)=' .config
# every subsystem of this course is a switch here — YOU decide what's in

# ---- 2. Explore the config menu (find a lesson's option) ----
$ cat << 'NOTE'
  make menuconfig     # then press '/' and search: PSI, HUGEPAGE, SECCOMP,
                      # SCHED, EXT4, APPARMOR ... read each option's inline
                      # help. This IS the table of contents of the OS course,
                      # rendered as a configuration tree.
NOTE

# ---- 3. Build it — the canonical multicore CPU-bound job (L12/L24) ----
$ time make -j$(nproc) bzImage 2>&1 | tail -5
#   ...  LD  vmlinux  /  OBJCOPY arch/x86/boot/bzImage
# watch `top` in another terminal: all cores at 100% — kernel build is the
# reference parallel workload (and a great stress/thermal test)
$ ls -lh arch/x86/boot/bzImage           # YOUR kernel, a few MB

# ---- 4. Boot YOUR kernel in QEMU with the Lesson 51 initramfs ----
$ cat << 'NOTE'
  # reuse (or rebuild) the busybox initramfs from Lesson 51, then:
  qemu-system-x86_64 -m 256 -nographic \
    -kernel arch/x86/boot/bzImage \
    -initrd /tmp/myinitramfs.gz \
    -append "console=ttyS0"

  → boots the kernel YOU compiled. First line:
      Linux version 6.8.0 (you@yourhost) ... 
    ^ your username, your build. Run `uname -a` in the busybox shell —
    it reports the kernel you just built. Ctrl-A X to quit.
NOTE

# ---- 5. Bisect: the debugging superpower, demonstrated in miniature ----
$ cat << 'NOTE'
  git bisect start
  git bisect bad v6.8            # this version has the bug
  git bisect good v6.7          # this one didn't
  # git checks out the midpoint; you: make -j, boot in QEMU, test:
  git bisect good   # or 'bad' — git halves the range each time (~log2(N) steps)
  # after ~13 iterations over thousands of commits:
  #   <sha> is the first bad commit    ← the exact change that broke it
  git bisect reset
NOTE
# O(log n) over 10,000 commits = ~13 build/boot/test cycles. This is how
# kernel regressions are actually hunted (L12's binary-search complexity, live).

# ---- 6. What you built, quantified ----
$ find . -name '*.c' | wc -l              # thousands of source files
$ cat .config | grep -c '=y'              # features compiled IN
$ cat .config | grep -c '=m'              # features as modules
$ cd / && echo "cleanup: rm -rf /tmp/kbuild when done (it's ~20GB built)"
```

---

## Further Reading

| Topic | Link |
|---|---|
| Kernel build system (Kbuild) — docs | <https://www.kernel.org/doc/html/latest/kbuild/index.html> |
| Building the kernel — kernelnewbies | <https://kernelnewbies.org/KernelBuild> |
| `menuconfig` — kconfig docs | <https://www.kernel.org/doc/html/latest/kbuild/kconfig.html> |
| `git bisect` — git docs | <https://git-scm.com/docs/git-bisect> |
| kernel.org — the source | <https://www.kernel.org/> |
| Linux kernel (Wikipedia) | <https://en.wikipedia.org/wiki/Linux_kernel> |

---

## Checkpoint

**Q1.** Your custom kernel boots but has no network card in QEMU. Describe
your debugging path — is it a config, a module, or a command-line problem, and
how do you tell?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Almost certainly a <strong>config</strong> problem: your minimal (tinyconfig-
derived) kernel likely lacks the virtio-net driver QEMU's default NIC needs.
Diagnosis path: (1) <em>Is the driver present at all?</em> —
<code>grep VIRTIO_NET .config</code>; if it's <code>=n</code>, the kernel
can't drive the card regardless of anything else — enable it (<code>=y</code>
to build in, avoiding the initramfs-module question — Lesson 51) and rebuild.
(2) <em>Is it a module the initramfs didn't load?</em> — if it's
<code>=m</code>, the kernel has it but your hand-built busybox initramfs
(Lesson 51) never ran <code>modprobe virtio_net</code>, so the device exists
but no driver bound: check <code>dmesg</code>/<code>lspci</code> in the guest
(the PCI device shows up, unclaimed) — fix by building it <code>=y</code> or
adding the modprobe to /init. (3) <em>Is the hardware even presented?</em> —
check the QEMU command line: without <code>-nic</code>/<code>-netdev</code>
QEMU may present no NIC (a command-line problem, not the kernel's fault) —
<code>-nic user,model=virtio-net-pci</code> gives it one. How to tell them
apart: <code>lspci</code> in the guest distinguishes "no device" (command-line
issue — nothing on the PCI bus) from "device present, no driver" (config/module
issue — device listed but no kernel driver claimed it), and
<code>dmesg | grep -i virtio</code> shows whether the driver probed. The
systematic move — is the capability compiled in? is it loaded? is the hardware
presented? — is Lesson 53's device chain (device → driver → bind) run in
reverse as a checklist, and it generalizes to every "my new kernel lost
feature X" (storage, console, filesystem): each is "did I configure the driver,
=y or =m-plus-initramfs, and is the underlying thing present."
</details>

**Q2.** The `=y` vs `=m` choice for a driver connects directly to two earlier
lessons. Which two, and what's the trade-off you're making with each choice?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It connects to <strong>Lesson 51 (initramfs)</strong> and <strong>Lesson 46
(static vs dynamic linking)</strong> — it's the same trade-off at the kernel
level. <code>=y</code> builds the driver <em>into</em> the vmlinuz: always
present, available at the earliest boot moment, no initramfs needed to reach
it (Lesson 51's "no special drivers → no initramfs" case) — but it makes the
kernel image bigger and carries code for hardware this machine may not have,
and you can't unload it (it's permanent). <code>=m</code> builds a loadable
module (Lesson 54): the kernel stays small and generic, the driver loads only
when needed (on-demand via udev/modalias — Lesson 53) and can be unloaded,
but it must live in the filesystem — and if it's needed to <em>reach root</em>
(the disk/LVM/crypto drivers), it must be in the initramfs (Lesson 51's whole
reason to exist), and a module can be built/shipped separately (DKMS,
out-of-tree). This is exactly Lesson 46's static-vs-dynamic debate: <code>=y</code>
is static linking (self-contained, bigger, everything present, patch = rebuild),
<code>=m</code> is dynamic (small core, load on demand, flexible, but needs the
loader/initramfs infrastructure and correct module presence). Distros choose
<code>=m</code> for almost everything (one generic kernel boots any hardware,
loading only what's found — the modular-kernel design), <code>=y</code> for
the essentials that must be present before any module could load (the
console, the initial filesystem, sometimes the boot storage controller). A
custom/embedded kernel for known-fixed hardware often goes <code>=y</code> for
everything and skips the initramfs entirely (Lesson 51's "no initramfs" path)
— trading flexibility for simplicity, the same call as choosing a static
binary for a container.
</details>

**Q3.** How does building a kernel change your mental model of "the kernel" —
and how does `git bisect` over a kernel build embody a concept from the
scheduling phase?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Mental model shift: "the kernel" stops being an opaque, given artifact and
becomes <em>a program you compiled from source you can read and modify</em> —
~30 million lines of C producing the vmlinuz the bootloader loads (Lesson 50),
whose every capability is a <code>CONFIG_</code> switch corresponding to a
subsystem you studied (the config tree literally is this course's table of
contents). It demystifies the boundary you respected all along: ring 0 isn't
magic, it's code with different rules (Lessons 54–55), compiled the same way
your programs are, just larger and unforgiving. You realize you could add a
printk to <code>do_fork</code>, change the scheduler's constants, or write a
new syscall — the kernel is editable, which reframes it from "the system" to
"the biggest program on the system." <code>git bisect</code> embodies
<strong>binary search — O(log n)</strong> — the complexity idea underlying so
much of the scheduling/data-structure discussion (Lesson 12's context-switch
math, Lesson 13's runqueue efficiency): a regression hides in one of ~10,000
commits, and rather than checking each (O(n) — days of builds), bisect halves
the candidate range every iteration, so ~13 build-boot-test cycles
(log₂(10000) ≈ 13.3) pinpoint the exact culprit commit. It's the same reason
per-CPU runqueues avoid global scans and epoll returns O(ready) not O(N)
(Lesson 43) — logarithmic (or better) beats linear at scale, turning an
intractable search into a tractable one. That kernel development at the scale
of thousands of contributors is even <em>possible</em> rests on this: bisect
(plus the build reproducibility this lesson gives you) converts "a bug appeared
somewhere in a year of changes" into a mechanical, bounded hunt — algorithmic
thinking as the load-bearing tool of real systems maintenance.
</details>

---

## Homework

You've now built the kernel whose every subsystem this phase and the previous
ten explored. As a capstone reflection (and a real skill-builder): pick THREE
config options from three different phases of this course
(`make menuconfig`, search for them), read each one's inline help, and for
each write: which lesson it implements, what changes in the running system if
you flip it, and one observable effect you could measure (tying back to that
lesson's lab). Suggested starting points: `CONFIG_PSI` (Lesson 15),
`CONFIG_PREEMPT` variants (Lesson 13/14), `CONFIG_IO_URING` (Lesson 44),
`CONFIG_TRANSPARENT_HUGEPAGE` (Lesson 23), `CONFIG_SECCOMP` (Lesson 56).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Example set: <strong>CONFIG_PSI</strong> (Lesson 15, Scheduling phase) — help
says it enables Pressure Stall Information accounting; flipping it off makes
<code>/proc/pressure/{cpu,memory,io}</code> disappear entirely, and
systemd-oomd/PSI-based tools lose their signal; observable effect: with it on,
the Lesson 15 lab's cpu/io pressure verdicts work — with it off (or
<code>psi=0</code> on the cmdline, Lesson 50), the triage script's PSI reads
fail, forcing you back to the ambiguous load average the lesson critiqued.
<strong>CONFIG_TRANSPARENT_HUGEPAGE</strong> (Lesson 23, Memory phase) — help
covers automatic 2 MiB page promotion; the choice of always/madvise/never as
the default, and whether it's compiled in at all, changes whether
<code>/sys/kernel/mm/transparent_hugepage/</code> exists and whether big
anonymous mappings get huge pages; observable: re-run Lesson 23's fault-count
lab — with THP the 512 MB write costs ~256 faults (one per 2 MiB), without it
~131072 (one per 4 KiB) — the 512:1 dividend appears or vanishes with this one
switch, and <code>grep AnonHugePages /proc/meminfo</code> reads zero without
it. <strong>CONFIG_IO_URING</strong> (Lesson 44, Async I/O phase) — help notes
it enables the io_uring interface; off means <code>io_uring_setup</code>
returns ENOSYS and the entire subsystem is absent (the strongest form of
Docker's seccomp block — Lesson 44/56); observable: the Lesson 44 liburing
labs fail at setup, and <code>fio --ioengine=io_uring</code> errors out —
proving the feature is a compile-time presence, not just a runtime toggle
(<code>io_uring_disabled</code> sysctl only exists if this is =y). The
reflection's real payoff: seeing that the running system's <em>behavior</em>
— the exact numbers your labs measured — traces to individual documented
switches you now control, closing the loop from "the kernel does X" (observed
from userspace across the whole course) to "the kernel does X because
CONFIG_X=y, and here's the source that implements it, which I can read, change,
and rebuild." That's the transition from kernel <em>user</em> to kernel
<em>developer</em> — and the foundation for everything past this course.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 12 — Containers & Resource Control (Lesson 59: Namespaces) →](lesson-59-namespaces){: .btn .btn-primary }
