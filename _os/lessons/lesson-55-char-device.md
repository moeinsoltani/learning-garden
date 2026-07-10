---
title: "Lesson 55 — A Character Device"
nav_order: 2
parent: "Phase 11: Kernel Interfaces & Security"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 55: A Character Device

## Concept

Lesson 37 showed device nodes are special inodes with major/minor numbers;
Lesson 53 showed udev creating them. This lesson reveals the *other end*:
what happens when a program `read()`s `/dev/yourdevice` — whose code runs?
You'll write it, and finally see "everything is a file" from inside.

```
   program:  fd = open("/dev/mylab");  read(fd, buf, 100);
                            │
                    VFS routes by the node's MAJOR number (L37)
                            │
                            ▼  to YOUR module's handlers:
   struct file_operations {
       .open    = mylab_open,      ← runs in the READER's syscall context
       .read    = mylab_read,      ← YOU fill their buffer
       .write   = mylab_write,     ← YOU consume their bytes
       .release = mylab_close,
   };
```

This is the driver model in miniature. A **character device** transfers data
as a stream of bytes (vs block devices' fixed blocks — Lesson 42):
`/dev/null` (discards writes, returns EOF on read), `/dev/zero` (infinite
zeros — Lesson 16's zero page's userspace face), `/dev/urandom` (the CSPRNG),
`/dev/ttyS0` (serial), every GPU and sound card's control interface. Each is
a `file_operations` struct in some driver, exactly like the one you're about
to write. After this lesson, `cat /dev/zero` is no longer magic — it's a
kernel function returning zeros into your buffer.

---

## How It Works

### Registering a char device

Three steps in your module's init: (1) get a **device number**
(`alloc_chrdev_region` — a major, assigned dynamically, plus minor); (2)
initialize a `cdev` with your `file_operations` and `cdev_add` it; (3)
optionally create the `/dev` node automatically (`class_create` +
`device_create` — which emits the uevent that makes udev create the node,
Lesson 53's chain, triggered by your code!). Exit undoes all three in reverse
(Lesson 54's exact-undo rule — dangling device nodes are use-after-free).

### The handlers and the userspace boundary

Your `read(file, user_buf, count, offset)` must move data to *user space* —
and here's a rule that catches everyone: you **cannot** just `memcpy` into
`user_buf`. That pointer is a *user* virtual address (Lesson 16), possibly
unmapped, swapped, or malicious; dereferencing it directly from the kernel
is a security hole and a crash risk. You must use `copy_to_user()` /
`copy_from_user()` — kernel helpers that safely validate and transfer across
the boundary (handling faults, checking the address belongs to the caller).
This is the kernel/user boundary (Lesson 01) enforced *in code you write* —
the same boundary strace watches and syscalls cross, now your responsibility.

`read` returns the number of bytes provided (0 = EOF — Lesson 02's contract,
now from the producing side!); `write` returns bytes consumed. The context is
the *calling process's* (Lesson 54): `current` points to the reader's task,
you may sleep here (it's process context, not interrupt), and you're subject
to being interrupted by signals (Lesson 09 — return -EINTR).

{: .note }
> **How /dev/null and /dev/zero actually work**
> They're the simplest possible char drivers, in the kernel tree
> (<code>drivers/char/mem.c</code>): /dev/null's read returns 0 (instant EOF)
> and its write returns count (consumed, discarded); /dev/zero's read
> <code>clear_user()</code>s the buffer (fills zeros) and returns count —
> forever. Your lab device is these, with a message. Reading their source
> after this lesson is genuinely illuminating: they're a dozen lines each.

---

## Lab

> Throwaway VM (Lesson 54's rule still applies — this is a real kernel module).

```bash
$ sudo apt install -y build-essential linux-headers-$(uname -r) 2>/dev/null | tail -1
$ mkdir -p /tmp/chardev && cd /tmp/chardev

# ---- 1. A char device that returns a message and counts reads ----
$ cat > mylab.c << 'EOF'
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/uaccess.h>      /* copy_to_user / copy_from_user */
#include <linux/device.h>

#define NAME "mylab"
static dev_t devno;
static struct cdev my_cdev;
static struct class *my_class;
static int read_count = 0;

static ssize_t mylab_read(struct file *f, char __user *ubuf,
                          size_t len, loff_t *off) {
    char msg[128];
    int n = snprintf(msg, sizeof msg,
                     "Hello from a char device! (read #%d)\n", ++read_count);
    if (*off >= n) return 0;                       /* EOF (L02 contract) */
    if (len > n - *off) len = n - *off;
    if (copy_to_user(ubuf, msg + *off, len))       /* SAFE boundary crossing */
        return -EFAULT;
    *off += len;
    return len;                                     /* bytes provided */
}
static ssize_t mylab_write(struct file *f, const char __user *ubuf,
                           size_t len, loff_t *off) {
    char buf[128];
    size_t n = min(len, sizeof buf - 1);
    if (copy_from_user(buf, ubuf, n)) return -EFAULT;
    buf[n] = 0;
    printk(KERN_INFO "mylab: userspace wrote: %s", buf);
    return len;                                     /* consumed all (like null) */
}
static struct file_operations fops = {
    .owner = THIS_MODULE, .read = mylab_read, .write = mylab_write,
};
static int __init mylab_init(void) {
    alloc_chrdev_region(&devno, 0, 1, NAME);        /* 1: get a major/minor */
    cdev_init(&my_cdev, &fops); cdev_add(&my_cdev, devno, 1);   /* 2: register */
    my_class = class_create("mylab_class");         /* 3: trigger udev → /dev */
    device_create(my_class, NULL, devno, NULL, NAME);
    printk(KERN_INFO "mylab: /dev/%s ready (major %d)\n", NAME, MAJOR(devno));
    return 0;
}
static void __exit mylab_exit(void) {               /* undo EXACTLY (L54) */
    device_destroy(my_class, devno);
    class_destroy(my_class);
    cdev_del(&my_cdev);
    unregister_chrdev_region(devno, 1);
    printk(KERN_INFO "mylab: unloaded\n");
}
module_init(mylab_init); module_exit(mylab_exit);
MODULE_LICENSE("GPL"); MODULE_DESCRIPTION("Phase 11 char device");
EOF
$ cat > Makefile << 'EOF'
obj-m += mylab.o
all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
EOF
$ make 2>&1 | tail -2

# ---- 2. Load it — and watch YOUR code create a /dev node (L53 chain!) ----
$ sudo insmod ./mylab.ko
$ sudo dmesg | tail -1                       # mylab: /dev/mylab ready (major NNN)
$ ls -l /dev/mylab                           # crw------- ... YOUR device node
# device_create emitted a uevent → udev created this. You triggered the chain.

# ---- 3. read() it — your mylab_read runs in cat's context ----
$ sudo cat /dev/mylab
# Hello from a char device! (read #1)         ← your kernel function filled the buffer
$ sudo cat /dev/mylab
# Hello from a char device! (read #2)         ← the counter proves it's live code
$ sudo head -c 20 /dev/mylab; echo            # partial read — offset logic works

# ---- 4. write() to it — your mylab_write consumes the bytes ----
$ echo "greetings, kernel" | sudo tee /dev/mylab >/dev/null
$ sudo dmesg | tail -1                        # mylab: userspace wrote: greetings, kernel

# ---- 5. Compare with the real /dev/null and /dev/zero it imitates ----
$ echo "into the void" > /dev/null            # write returns count, discards
$ head -c 8 /dev/zero | xxd                   # read fills zeros forever
# your device is these two combined + a message: SAME mechanism, same fops model

# ---- 6. Unload cleanly — node disappears (exit undid init) ----
$ sudo rmmod mylab
$ ls /dev/mylab 2>&1                          # No such file — device_destroy worked
$ cd / && make -C /tmp/chardev clean >/dev/null 2>&1; rm -rf /tmp/chardev
```

---

## Further Reading

| Topic | Link |
|---|---|
| Device file (Wikipedia) | <https://en.wikipedia.org/wiki/Device_file> |
| Character device drivers — LKMPG | <https://sysprog21.github.io/lkmpg/#character-device-drivers> |
| `file_operations` — kernel source | <https://www.kernel.org/doc/html/latest/filesystems/vfs.html> |
| `copy_to_user` / user access — kernel docs | <https://www.kernel.org/doc/html/latest/core-api/unaligned-memory-access.html> |
| Linux Device Drivers (LDD3) book | <https://lwn.net/Kernel/LDD3/> |
| `mknod(2)` man page | <https://man7.org/linux/man-pages/man2/mknod.2.html> |

---

## Checkpoint

**Q1.** When your program reads `/dev/mylab`, whose code executes — the
reader's, the kernel's, or your module's — and in what context?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Your module's code executes — <code>mylab_read</code> — but running in the
<strong>kernel</strong> (ring 0) <em>in the reader's process context</em>.
The chain: the reader calls <code>read(fd)</code> (a syscall — Lesson 02,
the reader's code stops at the boundary); the kernel's VFS dispatches by the
node's major number (Lesson 37) to your registered <code>file_operations</code>;
your <code>mylab_read</code> runs with kernel privilege but on behalf of, and
in the context of, the calling process — <code>current</code> is the reader's
task_struct, the code runs on the reader's kernel stack, it's counted as that
process's kernel CPU time, and it may sleep (process context — you could block
waiting for data, unlike interrupt context, Lesson 05). So the answer is
"all three, layered": the reader initiated it and its context frames the call,
the kernel's VFS routed it, and your module supplies the actual behavior. This
is the driver model's whole shape — the kernel provides the mechanism
(dispatch, boundary-crossing helpers) and policy (what the device does) is
your <code>file_operations</code>, the same mechanism/policy split as Lesson
01, now with you writing the policy.
</details>

**Q2.** Why must `mylab_read` use `copy_to_user()` instead of a plain
`memcpy` into the user's buffer? What are you being protected from?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The user buffer pointer is a <em>user-space virtual address</em> (Lesson 16) —
meaningful in the caller's address space, and it may be: unmapped (the program
passed a bad pointer), not yet faulted in or swapped out (Lessons 19/21 — a
plain access from the kernel wouldn't run the normal fault path safely),
pointing at memory the process shouldn't access, or even <em>maliciously
crafted to point at kernel memory</em> (a classic exploit: pass a kernel
address as your "buffer" and trick a driver into writing kernel data there, or
reading kernel secrets out). <code>copy_to_user</code>/<code>copy_from_user</code>
are the kernel's guarded boundary-crossing helpers: they verify the address
range belongs to user space (not kernel), handle page faults during the copy
gracefully (returning an error rather than oopsing — the fault path made
safe), and are the single sanctioned way to move data across the privilege
boundary you've studied all course. A plain memcpy would (a) crash the kernel
on any bad/unmapped user pointer (Q1's oops, now attacker-triggerable via a
bad read() argument) and (b) blow a hole in the entire user/kernel isolation
model — letting userspace read or write arbitrary kernel memory through your
driver. Every real driver's read/write/ioctl uses these helpers; forgetting
them is one of the most common (and most serious) driver vulnerabilities,
which is why the <code>__user</code> annotation on the pointer type exists —
static analyzers (sparse) flag any direct dereference of a __user pointer.
</details>

**Q3.** `/dev/zero` returns infinite zeros; `/dev/null` discards everything.
Using this lesson's `file_operations` model, describe each one's read and
write handlers — and connect /dev/zero to a mechanism from the memory phase.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>/dev/null</strong>: <code>read</code> immediately returns 0 (EOF —
Lesson 02's contract from the producer side: "no data, ever"); <code>write</code>
returns <code>count</code> (claims to have consumed everything) and does
nothing with the bytes — hence redirecting output to /dev/null "discards" it:
the write succeeds instantly, the data evaporates. <strong>/dev/zero</strong>:
<code>write</code> is identical to null's (consume-and-discard); <code>read</code>
fills the user's buffer with zeros (<code>clear_user()</code> — the safe
boundary-crossing zero-fill) and returns <code>count</code>, never EOF — an
infinite source. Both are ~a dozen lines in <code>drivers/char/mem.c</code>,
pure <code>file_operations</code> structs like your mylab device. The memory-
phase connection: /dev/zero's infinite zeros are the userspace face of the
<strong>zero page</strong> (Lesson 19) — and historically, mmap'ing
/dev/zero was the standard way to get zero-initialized anonymous memory
before <code>MAP_ANONYMOUS</code> existed (you'd <code>mmap("/dev/zero")</code>
to allocate demand-zero pages; the driver's mmap handler wired the mapping to
the shared zero page, copy-on-write on first write — exactly Lesson 19's
mechanism). So /dev/zero isn't just a curiosity: it was the original
allocator primitive, and reading its source ties the char-device model
(this lesson) to demand paging (Lesson 19) to anonymous memory (Lesson 18) —
the whole memory phase surfacing as a twelve-line character driver.
</details>

---

## Homework

Add an `ioctl` handler to your char device — the escape hatch for
device-specific commands that don't fit read/write (setting a mode, querying
status, resetting the counter). Sketch the handler (the `switch (cmd)`
structure, and how arguments cross the boundary), then explain: why does
ioctl have a reputation as "the kernel's junk drawer" and a security-sensitive
surface, and what modern alternatives (from earlier in this course) are
preferred for new interfaces?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Sketch: add <code>.unlocked_ioctl = mylab_ioctl</code> to fops;
<pre>
static long mylab_ioctl(struct file *f, unsigned int cmd, unsigned long arg) {
    switch (cmd) {
      case MYLAB_RESET:   read_count = 0; return 0;
      case MYLAB_GET_CNT:
          if (copy_to_user((int __user *)arg, &read_count, sizeof(int)))
              return -EFAULT;                   /* boundary-cross, as always */
          return 0;
      default: return -ENOTTY;                  /* "not a typewriter" — the
                                                   traditional bad-ioctl errno */
    }
}
</pre>
cmd values are encoded with <code>_IOR/_IOW/_IOWR</code> macros (packing
direction, size, a magic number) so the kernel and userspace agree on the
argument's shape; arg is an opaque <code>unsigned long</code> — often a user
pointer requiring copy_to/from_user. Junk-drawer reputation: ioctl is an
<em>untyped, per-driver command multiplexer</em> — thousands of independent
command codes across drivers, each with ad-hoc argument structures, no
uniform interface, poor discoverability, and no type safety at the boundary
(arg is just a number/pointer you must manually and correctly interpret).
That makes it security-sensitive: it's a huge, sprawling attack surface where
each handler independently re-implements boundary crossing — a single
missing copy_from_user validation, integer overflow in a size field, or
missing permission check (many ioctls do privileged things but forget to
check capabilities — Lesson 11) becomes a kernel vulnerability; historically
ioctl handlers are a top source of CVEs and privilege escalations, and
they're prime seccomp-filter targets (Lesson 56) precisely because they can
do so much. Preferred modern alternatives from this course: <strong>sysfs
attributes</strong> (Lesson 04/53 — one value per file, typed-ish, uniform,
udev-visible, permission-controlled by file mode) for simple config/status;
<strong>netlink sockets</strong> (the networking track's <code>ip</code>
interface — structured, extensible, socket-based) for richer control planes;
<strong>eBPF</strong> for safe programmable behavior; and for genuinely
file-like data, plain read/write with a clear protocol. The guidance: expose
new interfaces as sysfs/netlink/configfs with real structure and
permissions, reserve ioctl for genuinely device-specific operations that
can't be modeled otherwise — and when you do use it, treat every arg as
hostile input crossing the boundary you learned to respect in this lesson.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 56 — seccomp →](lesson-56-seccomp){: .btn .btn-primary }
