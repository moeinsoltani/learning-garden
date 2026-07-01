---
title: "Lesson 57 — QEMU Hardening and Sandboxing"
nav_order: 57
parent: "Phase 14: Nested Virt & Security"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 57: QEMU Hardening and Sandboxing

## Concept

sVirt (Lesson 56) contains a compromised QEMU's *file* access. This lesson reduces the
chance of compromise in the first place and shrinks what a compromised QEMU can *do* —
defense in depth around the process itself: run it unprivileged, filter its syscalls, drop
capabilities, and minimize its device model.

```
   LAYERS OF QEMU HARDENING (defense in depth)
   ┌──────────────────────────────────────────────────────────┐
   │ unprivileged user   run as 'qemu', not root (-runas)       │
   │ seccomp sandbox     filter the syscalls QEMU may make      │
   │ drop capabilities   remove privileges it doesn't need      │
   │ namespaces          isolate mount/pid/net around QEMU      │
   │ minimal devices     fewer emulated devices = smaller surface│
   │ + sVirt (Lesson 56) MAC label confinement                  │
   └──────────────────────────────────────────────────────────┘
```

Each layer assumes the others might fail. The fewer privileges and the smaller the attack
surface, the less a guest escape can achieve.

---

## How It Works

### Run QEMU unprivileged

QEMU does not need to be root. Run it as a dedicated low-privilege user (`-runas qemu`, or
libvirt's `qemu`/`libvirt-qemu` user, configured in `/etc/libvirt/qemu.conf`). Then even a
full QEMU takeover yields only that user's limited rights — not root on the host. libvirt
does this by default for `qemu:///system`.

### seccomp sandbox

**seccomp** (secure computing) filters which **syscalls** a process may make. QEMU's
`-sandbox on` installs a seccomp filter that blocks syscalls QEMU has no legitimate reason
to use (e.g. `mount`, `ptrace`, module loading, many others). If exploited code tries a
forbidden syscall, the kernel kills the process. This dramatically narrows the kernel
attack surface reachable from a compromised QEMU. libvirt enables a seccomp sandbox by
default.

```
   -sandbox on,obsolete=deny,elevateprivileges=deny,spawn=deny,resourcecontrol=deny
```

### Drop capabilities and use namespaces

- **Capabilities:** Linux splits root's power into capabilities; QEMU should hold as few as
  possible. After setup it drops the ones it doesn't need, so a takeover can't, e.g., load
  kernel modules or change system time.
- **Namespaces:** libvirt can run each QEMU in its own **mount/PID** namespace, so a
  compromised QEMU sees a minimal filesystem/process view — it can't even *see* most of the
  host or other VMs' files.

### Minimize the device model (smaller attack surface)

This is the high-leverage, often-overlooked one. Each emulated device is **code that parses
guest input** — and historically, device emulation bugs (in audio, USB, network, display
adapters) have been the source of VM-escape vulnerabilities. Every device you *don't* add is
attack-surface you remove:

- Drop legacy/unused devices (floppy, parallel/serial ports, sound, USB controllers, extra
  NICs) you don't need.
- Prefer minimal machine types (`q35`, or `microvm` — Phase 15) over the kitchen-sink `pc`
  with its pile of ISA-era devices.
- Use virtio (small, well-audited) over complex emulated hardware where possible.

A VM with five devices has far less exploitable surface than one with thirty. So minimizing
devices improves **security** (fewer parsers exposed to the guest) *as well as* performance
(fewer exits, Lesson 10).

{: .note }
> **Why removing unused emulated devices improves security, not just performance**
> Every emulated device is a chunk of host-side code that takes input directly from the
> (untrusted) guest — register writes, DMA descriptors, packets. Bugs in that parsing code
> are a classic VM-escape vector (many CVEs have been in device emulation). A device that
> isn't present can't be attacked: removing unused devices deletes whole categories of
> reachable, guest-facing code from the QEMU process. So a minimal device model shrinks the
> attack surface (security) at the same time as it reduces VM exits and emulation overhead
> (performance) — the two goals align.

---

## Lab

```bash
# 1. Confirm QEMU is running UNPRIVILEGED (as the qemu/libvirt user, not root):
$ ps -o user,comm -p $(pgrep -f 'guest=vmA')
USER     COMMAND
qemu     qemu-system-x86      ← not root
# libvirt's setting:
$ grep -E '^user|^group' /etc/libvirt/qemu.conf
#user = "qemu"
#group = "qemu"

# 2. Run QEMU directly with seccomp sandbox + drop privileges:
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -sandbox on,obsolete=deny,elevateprivileges=deny,spawn=deny,resourcecontrol=deny \
    -runas qemu \
    -drive file=disk.qcow2,if=virtio -nographic &

# 3. Verify a seccomp filter is installed on the QEMU process:
$ grep -i seccomp /proc/$(pgrep -f 'qemu-system' | head -1)/status
Seccomp:        2            ← 2 = filter mode active (0 = none)

# 4. Inspect capabilities held by the QEMU process (should be minimal):
$ grep -i cap /proc/$(pgrep -f 'qemu-system' | head -1)/status

# 5. MINIMIZE the device model — list what's attached, then trim:
$ qemu-system-x86_64 -accel kvm -m 2G \
    -nodefaults \                 # don't auto-add the usual devices...
    -machine q35 \
    -drive file=disk.qcow2,if=virtio \
    -device virtio-net-pci,netdev=n0 -netdev user,id=n0 \
    -serial mon:stdio &           # ...add back ONLY what you need
# Compare attack surface vs a default invocation that adds VGA, USB, sound, floppy, etc.

# 6. See namespace isolation libvirt applies (per-VM mount/pid namespace):
$ sudo ls -l /proc/$(pgrep -f 'guest=vmA')/ns/      # mnt, pid namespaces differ from host
$ kill %1 %2 2>/dev/null
```

**Expected result:** QEMU runs as an unprivileged user with `Seccomp: 2` (active filter)
and minimal capabilities. Using `-nodefaults` plus only the devices you need yields a far
smaller device model than the default — visibly fewer emulated devices, hence less
attack surface.

---

## Further Reading

| Topic | Link |
|---|---|
| QEMU security / sandbox | [qemu.org — Security](https://www.qemu.org/docs/master/system/security.html) |
| seccomp | [man7.org — seccomp(2)](https://man7.org/linux/man-pages/man2/seccomp.2.html) |
| Linux capabilities | [man7.org — capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html) |
| Linux namespaces | [man7.org — namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html) |
| Attack surface | [Wikipedia — Attack surface](https://en.wikipedia.org/wiki/Attack_surface) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why does removing unused emulated devices from a VM improve security, not just performance?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Every emulated device is host-side code that parses input coming directly from the untrusted guest (register accesses, DMA descriptors, packets). Bugs in that parsing code are a classic VM-escape vector — many CVEs have been in device emulation (audio, USB, NICs, display). A device that isn't present can't be attacked, so removing unused devices deletes whole categories of guest-reachable code from QEMU, shrinking the attack surface. It also reduces VM exits/emulation overhead, so security and performance improve together — but the security benefit is the elimination of exploitable, guest-facing parsers.
</details>

---

**Q2. What does QEMU's `-sandbox on` (seccomp) do, and how does it limit a compromised QEMU?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
-sandbox on installs a seccomp filter that restricts which syscalls the QEMU process is allowed to make, blocking ones it has no legitimate need for (e.g. mount, ptrace, module loading). If exploited code inside a compromised QEMU attempts a forbidden syscall, the kernel terminates the process. This narrows the kernel attack surface reachable from QEMU: even if a guest escapes into the QEMU process, it can't invoke the broad set of syscalls an attacker would use to escalate or pivot, because the seccomp policy denies them at the kernel boundary.
</details>

---

**Q3. Name two other hardening measures (besides seccomp and device minimization) and what each protects against.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) Running QEMU as an unprivileged user (-runas qemu / libvirt's qemu user) — so a full QEMU takeover yields only that limited user's rights, not host root. (2) Dropping Linux capabilities — removing privileges QEMU doesn't need so a compromise can't, e.g., load kernel modules or change the clock. (3) Namespaces (mount/PID) around QEMU — so a compromised process sees only a minimal filesystem/process view and can't see most of the host or other VMs' files. (Plus sVirt/MAC from Lesson 56 for label-based file confinement.) Any two are valid; together they form defense in depth.
</details>

---

## Homework

Launch a guest two ways: once with QEMU's default device set, and once with `-nodefaults` plus only the devices it needs (virtio disk, one virtio NIC, a serial console). For each, list the emulated devices (e.g. via the QEMU monitor `info qtree` or `info pci`). Count the difference. Also verify `Seccomp: 2` in `/proc/<pid>/status` for both. Explain how the trimmed VM is both faster and more secure.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The default invocation attaches many devices (VGA, USB controller, possibly sound, floppy, extra serial/parallel ports, a default NIC) — visible as a long list in <code>info qtree</code>/<code>info pci</code>. The <code>-nodefaults</code> + explicit-devices VM shows only the handful you added (virtio-blk, one virtio-net, serial), a much shorter list. Both show Seccomp: 2 (the syscall filter is active regardless of device count). The trimmed VM is more secure because each removed emulated device was host-side code parsing untrusted guest input — a potential VM-escape vector — so fewer devices means fewer exploitable parsers reachable from the guest. It's also faster because fewer/simpler (virtio) devices cause fewer MMIO/PIO VM exits and less emulation work (Lesson 10). Minimizing the device model advances both goals at once.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 58 — Confidential Computing →](lesson-58-confidential-computing){: .btn .btn-primary }
