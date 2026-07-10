---
title: "Lesson 54 — Write a Kernel Module"
nav_order: 1
parent: "Phase 11: Kernel Interfaces & Security"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 54: Write a Kernel Module

## Concept

You've watched the kernel from user space for fifty lessons. Now you cross the
line and run your own code in **ring 0** (Lesson 01) — the same privilege as
the kernel itself. It's a small program with an unforgiving environment:

```
   user program                    kernel module
   ────────────                    ─────────────
   int main()                      module_init()  ← called at insmod
     ... code ...                    module_exit() ← called at rmmod
     return                        NO main, NO libc, NO printf —
   printf → libc → syscall           use printk() (writes to dmesg)
   crash → SIGSEGV, one process      crash → kernel OOPS, maybe the whole
     dies (L16)                        system (you're IN the kernel — L01)
   any library, any syscall         only the kernel's OWN functions, and
                                      an UNSTABLE API that changes each release
```

The rules that make this different from every program you've written:
**no standard library** (the kernel has its own `printk`, `kmalloc`,
string functions), **no floating point** (by default), **a crash is a kernel
crash** (a null dereference is an "oops" that can wedge the system — do this
in a throwaway VM, always), and **the API is deliberately unstable** (Lesson
46's ABI stability was a *userspace* promise; in-kernel interfaces change
every release, which is why out-of-tree drivers break and why the kernel
prefers drivers merged upstream).

This isn't just an exercise: it's how every driver, filesystem, and the
netfilter/eBPF hooks you used in the networking track actually plug in. And
it makes Lesson 01's "the kernel is a privileged program" concrete — you're
now *adding* to that program.

---

## How It Works

### The minimal module

Three ingredients: an init function (runs at load — register things, return 0
for success), an exit function (runs at unload — clean up *everything* you
registered, or you leak kernel resources permanently until reboot), and
metadata macros (`MODULE_LICENSE` — "GPL" or the kernel taints and refuses
some symbols; `MODULE_AUTHOR`, `MODULE_DESCRIPTION`). `printk` with a log
level (`KERN_INFO`, `KERN_ERR`) writes to the kernel ring buffer — your
`dmesg` output, and your only debugging window for simple modules.

### Building against the running kernel

Modules compile against kernel headers matching your *exact* running kernel
(`/lib/modules/$(uname -r)/build`) — the Kbuild system, invoked via a tiny
Makefile that calls the kernel's build machinery. This is why you install
`linux-headers-$(uname -r)` (the lab-setup note for Phase 11): the module
must match the kernel's internal structures, which is the flip side of the
unstable-API coin — a module built for 6.8 won't load into 6.9.

### Loading, and what "tainting" means

`insmod ./mod.ko` loads it (or `modprobe` for an installed one — Lesson 53);
`dmesg` shows your printk; `lsmod` lists it; `rmmod` unloads (running your
exit function). Loading a non-GPL or unsigned module **taints** the kernel
(a flag in `/proc/sys/kernel/tainted`) — a marker that "unofficial code ran,"
which kernel developers use to dismiss bug reports (reproduce without the
taint first). Secure Boot (Lesson 50/53) requires modules be *signed*; on a
lab VM you disable Secure Boot or sign your module to load it.

{: .note }
> **Why the kernel API is unstable — and why that's a choice**
> Userspace gets "we never break user space" (Lesson 46); in-kernel modules
> get the opposite: internal functions and structs change freely between
> releases. It's deliberate — a stable internal ABI would freeze the kernel's
> ability to evolve, and Linus's position is that drivers belong
> <em>upstream</em> (merged into the tree, updated with the kernel) rather
> than shipped as binaries against a frozen interface. The pain of
> out-of-tree modules (NVIDIA, ZFS, VirtualBox rebuilding per kernel via DKMS)
> is the intended incentive to upstream. Trade-off, chosen — the recurring
> theme of this course.

---

## Lab

> Run this in a throwaway VM. A module bug can panic the kernel — that's the
> point of the lesson, and the reason for the disposable machine.

```bash
# ---- 0. get the build tools (Phase 11 kit) ----
$ sudo apt install -y build-essential linux-headers-$(uname -r) 2>/dev/null | tail -1

# ---- 1. The whole module: two functions + metadata ----
$ mkdir -p /tmp/mymod && cd /tmp/mymod
$ cat > hello.c << 'EOF'
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

static int lab_count = 1;
module_param(lab_count, int, 0644);              /* a module parameter! (L53) */
MODULE_PARM_DESC(lab_count, "how many greetings to print");

static int __init hello_init(void) {             /* runs at insmod */
    for (int i = 0; i < lab_count; i++)
        printk(KERN_INFO "hello_lab: greeting %d from ring 0!\n", i);
    printk(KERN_INFO "hello_lab: loaded. I am kernel code now.\n");
    return 0;                                     /* 0 = success */
}
static void __exit hello_exit(void) {            /* runs at rmmod */
    printk(KERN_INFO "hello_lab: unloaded, cleaned up.\n");
}
module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");                            /* untaint */
MODULE_AUTHOR("OS course");
MODULE_DESCRIPTION("Phase 11 hello-world module");
EOF

# ---- 2. The Kbuild Makefile ----
$ cat > Makefile << 'EOF'
obj-m += hello.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
EOF
$ make 2>&1 | tail -3
# ...  CC [M]  hello.o  /  MODPOST  /  LD [M]  hello.ko
$ ls hello.ko && modinfo hello.ko | grep -E 'description|parm|license'

# ---- 3. Load it — your code runs in the kernel ----
$ sudo insmod ./hello.ko
$ sudo dmesg | tail -3
# hello_lab: greeting 0 from ring 0!
# hello_lab: loaded. I am kernel code now.      ← YOUR printk, from ring 0
$ lsmod | grep hello                            # it's a loaded module now
$ ls /sys/module/hello/parameters/              # lab_count — tunable (L53)!

# ---- 4. Reload with a parameter ----
$ sudo rmmod hello
$ sudo dmesg | tail -1                           # hello_lab: unloaded...
$ sudo insmod ./hello.ko lab_count=3            # pass the parameter
$ sudo dmesg | tail -4                           # three greetings now
$ cat /sys/module/hello/parameters/lab_count    # 3 — readable live

# ---- 5. Tainting: check before and after ----
$ cat /proc/sys/kernel/tainted                   # 0 (or nonzero from other modules)
# our GPL module doesn't add the proprietary taint; an unsigned one on a
# Secure Boot system would REFUSE to load entirely (L53)
$ sudo rmmod hello

# ---- 6. (Optional, DANGEROUS — snapshot first) feel the difference ----
# Uncomment the deref in a COPY to see an oops instead of a segfault:
#   static int __init bad_init(void){ int *p=NULL; printk("%d\n",*p); return 0; }
# insmod → kernel OOPS in dmesg (stack trace, "BUG: kernel NULL pointer
# dereference"), the insmod process killed, the system possibly unstable.
# A userspace null deref (L16) is one dead process; this is the kernel.
# THIS is why lesson intro says: throwaway VM.
$ cd / && make -C /tmp/mymod clean >/dev/null 2>&1; rm -rf /tmp/mymod
```

---

## Further Reading

| Topic | Link |
|---|---|
| The Linux Kernel Module Programming Guide | <https://sysprog21.github.io/lkmpg/> |
| Loadable kernel module | <https://en.wikipedia.org/wiki/Loadable_kernel_module> |
| `printk` — kernel docs | <https://www.kernel.org/doc/html/latest/core-api/printk-basics.html> |
| Building external modules — kernel docs | <https://www.kernel.org/doc/html/latest/kbuild/modules.html> |
| Tainted kernels — kernel docs | <https://www.kernel.org/doc/html/latest/admin-guide/tainted-kernels.html> |
| `insmod(8)` / `rmmod(8)` | <https://man7.org/linux/man-pages/man8/insmod.8.html> |

---

## Checkpoint

**Q1.** Your module has a bug that dereferences NULL. What happens to the
system, compared to the same bug in a user program?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
In a user program, a NULL dereference triggers a page fault the kernel
resolves as "no valid mapping" and delivers SIGSEGV (Lessons 16/19): one
process dies, cleanly, everything else unaffected — the MMU and the kernel
contained it. In a module you <em>are</em> the kernel: the same dereference
faults, but now the fault handler finds the fault happened <em>in kernel
mode</em>, which is a bug in the kernel itself — it produces an
<strong>oops</strong>: a stack trace dumped to dmesg, the offending task
killed, and the kernel marked tainted/unstable. Consequences range from
"the insmod process dies and that subsystem is broken" to a full
<strong>panic</strong> (if it happened while holding a critical lock or in
interrupt context — Lesson 05) that halts the machine, or subtle corruption
that crashes something unrelated minutes later. There's no outer container to
catch a ring-0 fault — the kernel is the last line. This asymmetry (contained
vs catastrophic) is the entire reason for the disposable-VM rule, for the
kernel's defensive coding culture, and for pushing risky logic into userspace
(FUSE, userspace drivers) or sandboxed eBPF (verified before it runs —
networking Phase 11) rather than raw modules.
</details>

**Q2.** Why must a module be built against the exact running kernel version,
and how does this relate to Lesson 46's stable-ABI discussion — but with the
opposite conclusion?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A module calls kernel functions and manipulates kernel data structures
directly (no syscall boundary — it's in ring 0). The kernel's internal
function signatures, struct layouts (field offsets, sizes), and even config
options (SMP on/off, various CONFIG_ flags) change between releases, so a
module compiled against 6.8's headers embeds 6.8's assumptions; loading it
into 6.9 would read struct fields at wrong offsets and call functions with
mismatched signatures — instant corruption. The kernel guards against this
with vermagic (a version string checked at load — mismatches are refused) and
symbol versioning (modversions). Relation to Lesson 46 with the inverted
conclusion: userspace enjoys a <em>frozen</em> ABI ("we never break user
space") because the syscall interface is the stable public contract; the
<em>in-kernel</em> ABI is deliberately <em>unstable</em> — the kernel
reserves the right to change any internal interface every release. Same
concept (binary compatibility across versions), opposite policy, because the
audiences differ: userspace must be protected (billions of binaries the
kernel can't recompile), while in-kernel modules are expected to live
upstream and be recompiled with the kernel (or endure DKMS rebuilds). The
stability boundary is exactly the syscall layer — everything outside it is
promised forever, everything inside it changes freely, and modules live on
the volatile side.
</details>

**Q3.** Loading your module printed to `dmesg` via `printk`, not `printf`.
Beyond "no libc," what deeper constraints does running in kernel context
impose, and why can't a module just call the syscalls you've used all course?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Syscalls are the <em>userspace-to-kernel</em> door (Lesson 02) — a module is
already <em>inside</em> the kernel, so "calling a syscall" is nonsensical
(there's no user context to trap from, and the syscall handlers expect to be
entered from user mode); instead a module calls the kernel functions the
syscalls themselves use internally (e.g. not <code>write()</code> but the VFS
functions, not <code>malloc()</code> but <code>kmalloc</code>). Deeper
constraints of kernel context: (1) <strong>a limited kernel stack</strong>
(historically ~8–16 KB total, not the megabytes of userspace — Lesson 18) —
deep recursion or big local buffers overflow it and corrupt memory; (2)
<strong>you may run in contexts that cannot sleep</strong> — interrupt
handlers and code holding spinlocks (Lesson 05/26) must not block, so many
functions come in "may sleep" vs "atomic" variants (GFP_KERNEL vs GFP_ATOMIC
for allocation) and getting it wrong deadlocks or panics; (3) <strong>no
memory protection from the rest of the kernel</strong> — a stray pointer
corrupts anything (Q1); (4) <strong>no floating point</strong> by default
(the FPU state isn't saved on kernel entry, Lesson 12); (5) concurrency is
everywhere and unforgiving — your code runs on all CPUs at once, preemptible,
so every shared structure needs the locking of Phase 5 with kernel
primitives. Kernel programming is systems programming with the safety rails
removed and the blast radius maximized — which is why the culture is
paranoid, the review brutal, and the modern trend is to push extensibility
into <em>verified</em> mechanisms (eBPF) that give kernel-context power with
a verifier enforcing these constraints before the code ever runs.
</details>

---

## Homework

Extend the hello module into something that *observes* the kernel: add a
`/proc` entry (via `proc_create`) that, when read, returns the current number
of processes or the system uptime — turning your module into a `/proc` file
like the ones you've read all course (Lesson 04). Sketch the code structure
(what you register in init, what you must clean up in exit) and explain: why
must exit undo *exactly* what init did, and what happens if it doesn't
(connect to Lesson 08's resource-leak theme but at kernel scale)?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Structure: in <code>init</code>, call
<code>proc_create("mylab", 0444, NULL, &my_fops)</code> where
<code>my_fops</code> supplies a read/show function that formats data (e.g.
via a seq_file showing <code>jiffies</code>/HZ for uptime, or walking task
structs for a process count) — this registers a <code>/proc/mylab</code> file
exactly like the /proc entries of Lesson 04, backed by your callback (the
VFS-callback mechanism of Lesson 37, now written by you). In <code>exit</code>,
call <code>proc_remove()</code> (or remove_proc_entry) to unregister it.
Why exit must precisely undo init: kernel resources have no owning process to
clean up after — when a <em>user</em> process leaks an fd or memory, exit
reaps it (Lesson 08) and the kernel reclaims everything at process death; but
a <em>module</em> is part of the kernel, which never "exits," so anything you
register persists until <em>you</em> remove it. If exit forgets
<code>proc_remove</code>: the <code>/proc/mylab</code> entry survives after
rmmod, but its read callback points into the module's code — which was just
<em>unloaded and its memory freed</em>. The next process to
<code>cat /proc/mylab</code> jumps into freed memory: a use-after-free in the
kernel → oops or exploitable corruption (Q1's catastrophe). More generally,
leaked kernel allocations (kmalloc without kfree), un-freed timers, or
registered hooks that outlive the module are permanent leaks until reboot —
there is no OOM-killer-for-the-kernel, no parent to reap. This is why module
exit functions are written first and reviewed hardest, why the kernel has
managed-resource helpers (devm_*) that auto-clean, and why "unload cleanly"
is a much stronger requirement than "exit cleanly" in userspace: at kernel
scale, a leak is forever and a dangling reference is a security hole.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 55 — A Character Device →](lesson-55-char-device){: .btn .btn-primary }
