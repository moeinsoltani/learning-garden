---
title: "Lesson 08 — The /dev/kvm ioctl API"
nav_order: 8
parent: "Phase 3: KVM Internals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 08: The /dev/kvm ioctl API

## Concept

It's tempting to imagine KVM as a vast, mysterious subsystem. The reality is
humbling: **KVM is a character device** at `/dev/kvm`, and you drive it with
`ioctl()` calls — the same mechanism used to configure a serial port or a webcam.
Everything QEMU does to KVM, you could do by hand in ~150 lines of C.

There is a three-level hierarchy of file descriptors:

```
   open("/dev/kvm")                     ── the KVM SUBSYSTEM fd
        │  ioctl(KVM_CREATE_VM)
        ▼
   vm_fd                                ── one VIRTUAL MACHINE
        │  ioctl(KVM_SET_USER_MEMORY_REGION)   ← give it RAM
        │  ioctl(KVM_CREATE_VCPU)
        ▼
   vcpu_fd                              ── one VIRTUAL CPU
        │  mmap(vcpu_fd) → struct kvm_run      ← shared comms page
        │  ioctl(KVM_RUN)                       ← run guest, loop
        ▼
   (guest executes; returns on VM exit)
```

Each level is a file descriptor you get an `ioctl` handle on. This is the actual
foundation everything in this track sits on.

---

## How It Works

### The setup sequence

1. **`open("/dev/kvm")`** — get a handle to the KVM subsystem. You can query the
   API version (`KVM_GET_API_VERSION`) and supported features here.
2. **`ioctl(kvm_fd, KVM_CREATE_VM)`** → returns `vm_fd`, an empty VM with no CPUs
   and no memory.
3. **`ioctl(vm_fd, KVM_SET_USER_MEMORY_REGION, &region)`** — this is the crucial
   one. You hand KVM a region of *your own process's* memory (obtained via
   `mmap`/`malloc`) and tell it "this host memory *is* guest physical memory from
   GPA X for N bytes." **Guest RAM is just mmap'd host memory.** The EPT (Lesson 6)
   maps guest-physical onto these host pages.
4. **`ioctl(vm_fd, KVM_CREATE_VCPU)`** → returns `vcpu_fd`.
5. **`mmap(vcpu_fd)`** → maps a shared `struct kvm_run`, the communication channel
   between your process and the kernel for this vCPU. After each exit, you read
   `kvm_run->exit_reason` and the associated data here.
6. Set initial register state (`KVM_SET_SREGS`, `KVM_SET_REGS`) — e.g. point the
   instruction pointer at your guest code.
7. **`ioctl(vcpu_fd, KVM_RUN)`** in a loop — runs guest code until a VM exit, then
   returns so you can service it.

### The "run 16 bytes of guest code" example (conceptual)

The classic minimal KVM program puts a tiny real-mode program — a few bytes that
write to an I/O port and halt — into the guest RAM region, then runs it:

```c
   /* guest code: out to port 0x3f8 (serial), then hlt  */
   uint8_t code[] = {
       0xb0, 'H',        /* mov al, 'H'        */
       0xe6, 0x3f8 & 0xff,  /* out 0x3f8, al   (port write → VM exit) */
       0xf4              /* hlt                */
   };
   memcpy(guest_ram, code, sizeof(code));   /* guest_ram is mmap'd host mem */

   for (;;) {
       ioctl(vcpu_fd, KVM_RUN, 0);
       switch (run->exit_reason) {
           case KVM_EXIT_IO:    putchar(*((char*)run + run->io.data_offset)); break;
           case KVM_EXIT_HLT:   return 0;   /* guest halted, done */
       }
   }
```

When the guest runs `out 0x3f8, al`, the CPU takes a VM exit with
`exit_reason == KVM_EXIT_IO`; your loop reads the byte the guest "sent" and prints
it. That `out`/exit/handle cycle is, in miniature, *exactly* how QEMU emulates
every device — just with thousands of devices and far more code.

{: .note }
> **Why this matters even though you'll never write it**
> You will always use QEMU (or libvirt) in practice. But understanding that a VM is
> "create fd → give it mmap'd RAM → KVM_RUN loop" demystifies everything: guest RAM
> overcommit (it's just host memory you may not have touched yet), why a device
> access is expensive (it's an ioctl round-trip), and why QEMU has one thread per
> vCPU (one KVM_RUN loop each).

---

## Lab

```bash
# You don't have to write the C program — a famous ~150-line example exists.
# Fetch and read the canonical minimal KVM example, then map each ioctl to the
# steps above. (kvmtest.c by David Matlack / the LWN article version.)

# 1. Inspect the ioctl numbers the kernel exposes — they're in the UAPI header:
$ grep -E 'KVM_CREATE_VM|KVM_CREATE_VCPU|KVM_RUN|KVM_SET_USER_MEMORY_REGION' \
     /usr/include/linux/kvm.h
#define KVM_CREATE_VM             _IO(KVMIO,   0x01)
#define KVM_CREATE_VCPU           _IO(KVMIO,   0x41)
#define KVM_RUN                   _IO(KVMIO,   0x80)
#define KVM_SET_USER_MEMORY_REGION _IOW(KVMIO, 0x46, struct kvm_user_memory_region)

# 2. See KVM's API version, which the open()ed fd would report:
$ cat /sys/module/kvm/version 2>/dev/null || echo "(query via KVM_GET_API_VERSION)"

# 3. Watch a REAL VM's vcpu fds — QEMU holds one vcpu_fd per vCPU plus the vm_fd.
#    Start a guest, then inspect its open anon_inode:kvm-vcpu descriptors:
$ qemu-system-x86_64 -accel kvm -m 256 -smp 2 -nographic -name lab08 &
$ sudo ls -l /proc/$(pgrep -f lab08)/fd | grep -i kvm
lrwx------ ... -> /dev/kvm
lrwx------ ... -> anon_inode:kvm-vm
lrwx------ ... -> anon_inode:kvm-vcpu:0
lrwx------ ... -> anon_inode:kvm-vcpu:1
$ kill %1
```

**Expected result:** The ioctl constants exist in `kvm.h`, and a running QEMU
process holds exactly the fd hierarchy from the diagram: one `/dev/kvm`, one
`kvm-vm`, and one `kvm-vcpu:N` per vCPU.

---

## Further Reading

| Topic | Link |
|---|---|
| KVM API reference | [kernel.org — Virtual Machine API](https://docs.kernel.org/virt/kvm/api.html) |
| Using the KVM API (LWN) | [lwn.net — Using the KVM API](https://lwn.net/Articles/658511/) |
| `ioctl` | [man7.org — ioctl(2)](https://man7.org/linux/man-pages/man2/ioctl.2.html) |
| `mmap` | [man7.org — mmap(2)](https://man7.org/linux/man-pages/man2/mmap.2.html) |
| Character devices | [Wikipedia — Device file](https://en.wikipedia.org/wiki/Device_file#Character_devices) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. How does guest RAM physically exist on the host? What KVM call establishes it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Guest RAM is just ordinary host memory that the VMM process obtained (e.g. via mmap/malloc) in its own address space. The call <code>ioctl(vm_fd, KVM_SET_USER_MEMORY_REGION, &region)</code> registers that host memory region with KVM and tells it "this maps to guest-physical addresses starting at X." KVM then sets up the EPT/NPT so guest-physical accesses land on those host pages. There is no special "VM memory" — it's the VMM's mmap'd memory.
</details>

---

**Q2. List the three levels of file descriptor in the KVM API and what each represents.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) The fd from open("/dev/kvm") represents the KVM subsystem (query version/capabilities, create VMs). (2) The vm_fd from ioctl(KVM_CREATE_VM) represents one virtual machine (you give it memory and create vCPUs on it). (3) The vcpu_fd from ioctl(KVM_CREATE_VCPU) represents one virtual CPU; you mmap it to get the shared kvm_run struct and call ioctl(KVM_RUN) on it.
</details>

---

**Q3. In the minimal example, the guest runs `out 0x3f8, al`. Trace what happens from that instruction to a character appearing in the host program's output.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The guest executes the OUT instruction to I/O port 0x3f8. Because port I/O isn't allowed to touch real hardware directly, the CPU takes a VM exit with exit_reason KVM_EXIT_IO and KVM_RUN returns to the host loop. The host reads the byte the guest wrote from the shared kvm_run struct (at run->io.data_offset) and prints it with putchar. Then it calls KVM_RUN again to resume the guest. This out→exit→handle→resume cycle is the same mechanism QEMU uses to emulate every device.
</details>

---

## Homework

Find and skim the LWN "Using the KVM API" example program (linked in Further Reading). Identify the exact line/call that (a) creates the VM, (b) gives it memory, and (c) enters the run loop. Then explain in one sentence why the program `mmap`s the vcpu fd before calling `KVM_RUN`.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) The VM is created by <code>ioctl(kvm, KVM_CREATE_VM, 0)</code>; (b) memory is given by <code>ioctl(vmfd, KVM_SET_USER_MEMORY_REGION, &region)</code> after mmap-ing the guest RAM; (c) the run loop is the <code>for(;;) ioctl(vcpufd, KVM_RUN, 0)</code>. The program mmaps the vcpu fd first because KVM_RUN communicates results through the shared <code>struct kvm_run</code> located in that mapping — after each exit the host reads exit_reason and any I/O data from that mapped page, so it must exist before the first KVM_RUN.
</details>
