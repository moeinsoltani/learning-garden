---
title: "Lesson 09 — KVM Kernel Modules and Parameters"
nav_order: 9
parent: "Phase 3: KVM Internals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 09: KVM Kernel Modules and Parameters

## Concept

KVM ships as **two layers of kernel module**: a generic core and a vendor-specific
glue layer. Keeping them separate means the huge, architecture-independent logic
lives once, while the small VMX/SVM differences are isolated.

```
   ┌───────────────────────────────────────────────┐
   │  kvm.ko        — arch-independent CORE          │
   │   • the ioctl API, the VM/vCPU objects          │
   │   • the generic VM-exit dispatch, MMU logic     │
   └───────────────┬───────────────────────────────┘
        depends on │ (loaded automatically with one of:)
        ┌──────────┴───────────┐
   ┌────▼────────┐     ┌────────▼────────┐
   │ kvm_intel.ko │     │  kvm_amd.ko     │
   │  VMX glue    │     │   SVM glue      │
   │  (Intel)     │     │   (AMD)         │
   └─────────────┘     └─────────────────┘
```

Each module exposes **parameters** — tunable knobs — under
`/sys/module/<name>/parameters/`. A few of them materially affect what your guests
can do (nested virtualization) and how they perform (halt polling).

---

## How It Works

### The modules

| Module | Role |
|---|---|
| `kvm` | Generic KVM: the ioctl interface, VM/vCPU lifecycle, the architecture-independent MMU and exit dispatch. Always present. |
| `kvm_intel` | Intel VMX specifics: VMCS management, VMLAUNCH/VMRESUME, EPT. Loaded on Intel hosts. |
| `kvm_amd` | AMD SVM specifics: VMCB management, VMRUN, NPT. Loaded on AMD hosts. |

`kvm_intel`/`kvm_amd` *depend on* `kvm`, so loading the vendor module pulls in the
core automatically (you saw this in Lesson 2's `lsmod` output: `kvm 1 kvm_intel`).

### Parameters that matter

- **`nested`** (`kvm_intel`/`kvm_amd`) — `Y` exposes `vmx`/`svm` to *guests*, so a
  guest can itself run KVM (nested virtualization, Lesson 55). Off by default on
  some distros.
- **`ept`** (`kvm_intel`) / **`npt`** (`kvm_amd`) — enable hardware SLAT
  (Lesson 6). You want `Y`; turning it off forces slow shadow paging.
- **`halt_poll_ns`** (`kvm`) — when a vCPU halts (HLT), KVM can *briefly busy-poll*
  for the next event before truly sleeping the thread. This trades a little CPU for
  much lower wakeup latency on bursty workloads. Tunable in nanoseconds.

### Reading and changing parameters

Read them from sysfs:

```
$ cat /sys/module/kvm_intel/parameters/nested
N
```

Some are writable at runtime (`echo Y | sudo tee ...`), but parameters like
`nested` are bound at module load. To change those you must reload the module —
which means **no VMs can be running**, because they hold the module busy:

```
$ sudo modprobe -r kvm_intel        # remove (fails if a VM is using it)
$ sudo modprobe kvm_intel nested=1  # reload with the new value
```

To make it persistent across reboots, drop a file in `/etc/modprobe.d/`:

```
$ echo "options kvm_intel nested=1" | sudo tee /etc/modprobe.d/kvm.conf
```

{: .note }
> **Why module reload requires stopping VMs**
> A running guest's vCPU threads are executing inside `kvm_intel`'s code via
> KVM_RUN; the module's reference count is nonzero. `modprobe -r` refuses to unload
> a busy module (you'd see "Module kvm_intel is in use"). So `nested=1` and similar
> load-time params can only change when the host has zero live VMs — plan such
> changes during a maintenance window, or set them via /etc/modprobe.d and reboot.

---

## Lab

```bash
# 1. See the module dependency: the vendor module USES the core.
$ lsmod | grep kvm
kvm_intel             380928  0           # 0 = not currently in use by a VM
kvm                  1142784  1 kvm_intel  # used by kvm_intel

# 2. List ALL parameters the core module exposes:
$ ls /sys/module/kvm/parameters/
halt_poll_ns  halt_poll_ns_grow  halt_poll_ns_shrink  ...

# 3. Check the most consequential vendor params:
$ cat /sys/module/kvm_intel/parameters/nested
N
$ cat /sys/module/kvm_intel/parameters/ept
Y

# 4. Read halt_poll_ns (default often 200000 ns = 200 µs):
$ cat /sys/module/kvm/parameters/halt_poll_ns
200000

# 5. Enable nested virtualization (only works with NO running VMs):
$ sudo modprobe -r kvm_intel
$ sudo modprobe kvm_intel nested=1
$ cat /sys/module/kvm_intel/parameters/nested
Y

# 6. Make it survive reboot:
$ echo "options kvm_intel nested=1" | sudo tee /etc/modprobe.d/kvm.conf
```

**Expected result:** You can see `kvm` is used by `kvm_intel`, read each parameter
from sysfs, and flip `nested` to `Y` by reloading the vendor module — but only with
no VMs running.

---

## Further Reading

| Topic | Link |
|---|---|
| KVM nested virtualization | [kernel.org — Nested VMX](https://docs.kernel.org/virt/kvm/x86/nested-vmx.html) |
| `modprobe` | [man7.org — modprobe(8)](https://man7.org/linux/man-pages/man8/modprobe.8.html) |
| `modprobe.d` config | [man7.org — modprobe.d(5)](https://man7.org/linux/man-pages/man5/modprobe.d.5.html) |
| Kernel module parameters | [kernel.org — Module parameters](https://docs.kernel.org/admin-guide/kernel-parameters.html) |
| `lsmod` | [man7.org — lsmod(8)](https://man7.org/linux/man-pages/man8/lsmod.8.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Which module provides the generic KVM logic, and which provides the Intel-specific VMX glue? How are they related?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The generic, architecture-independent logic (ioctl API, VM/vCPU objects, exit dispatch, MMU) is in <code>kvm.ko</code>. The Intel-specific VMX glue (VMCS handling, VMLAUNCH/VMRESUME, EPT) is in <code>kvm_intel.ko</code> (and <code>kvm_amd.ko</code> for AMD SVM). The vendor module depends on the core, so loading kvm_intel automatically loads kvm, and lsmod shows kvm being "used by" kvm_intel.
</details>

---

**Q2. What does the `nested` parameter do, and why might you need to reload the module to change it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The <code>nested</code> parameter, when set to Y, exposes the vmx/svm virtualization features to guests so a guest can itself run KVM (nested virtualization). It's a load-time parameter bound when the module loads, so changing it requires unloading and reloading kvm_intel/kvm_amd. That only works when no VMs are running, because a live guest keeps the module's reference count nonzero and modprobe -r refuses to unload a busy module. For persistence, set it in /etc/modprobe.d and reboot.
</details>

---

**Q3. What is `halt_poll_ns` and what trade-off does it represent?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
When a guest vCPU halts (idle), KVM can busy-poll for up to halt_poll_ns nanoseconds waiting for the next event (interrupt/wakeup) before actually putting the vCPU thread to sleep. If an event arrives during the poll window, the vCPU resumes immediately with very low latency, avoiding the cost of a full sleep/wake cycle. The trade-off is spending host CPU cycles spinning during that window — good for latency-sensitive, bursty workloads, wasteful for idle guests.
</details>

---

## Homework

On your host, determine whether you're on Intel or AMD, then report the current value of `nested` and `ept` (or `npt`). If `nested` is `N`, write the exact two commands you'd run to enable it right now (assuming no VMs are running) and the one line you'd add to `/etc/modprobe.d/` to make it permanent.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
On Intel: <code>cat /sys/module/kvm_intel/parameters/nested</code> and <code>.../ept</code>. To enable now: <code>sudo modprobe -r kvm_intel</code> then <code>sudo modprobe kvm_intel nested=1</code>. Persistent: add <code>options kvm_intel nested=1</code> to a file like /etc/modprobe.d/kvm.conf. On AMD, substitute kvm_amd and check <code>npt</code> instead of <code>ept</code>: <code>options kvm_amd nested=1</code>. (ept/npt should already read Y; you want hardware SLAT on.)
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 10 — The VM-Exit Loop in Depth →](lesson-10-vm-exit-loop){: .btn .btn-primary }
