---
title: "Lesson 13 — Machine Types and Accelerators"
nav_order: 13
parent: "Phase 4: QEMU Fundamentals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 13: Machine Types and Accelerators

## Concept

Two flags decide the *fundamental character* of a VM:

- **`-machine`** — which **virtual motherboard** you build. This fixes the chipset,
  the buses, the default devices — the "shape" of the fake computer.
- **`-accel`** — **how guest code is executed**: natively on the CPU (KVM) or
  translated in software (TCG).

```
        -machine  (the virtual mainboard)        -accel (the engine)
   ┌──────────────────────────────────┐     ┌────────────────────────┐
   │  pc  (i440FX)   — legacy chipset  │     │  kvm  — native, fast    │
   │  q35 (ICH9)     — modern, PCIe    │     │  tcg  — emulate, slow   │
   │  microvm        — minimal (Ph.15) │     │  hvf  — macOS host      │
   │  virt           — ARM machines    │     │  whpx — Windows host    │
   └──────────────────────────────────┘     └────────────────────────┘
```

Get these two right and everything else (devices, CPU) slots in. Get them wrong and
your "VM" is either a slow emulator or a chipset too old for modern features like
PCIe passthrough.

---

## How It Works

### Machine types: `pc` vs `q35`

- **`pc`** emulates the ancient Intel **i440FX** chipset (1996-era PC). It uses a
  legacy PCI bus and has lots of built-in ISA-era devices. Maximally compatible
  with old guests, but no native PCIe.
- **`q35`** emulates the **ICH9** chipset, which provides a real **PCI Express**
  topology. This is the modern default — you want it for PCIe devices, better
  device assignment/passthrough (Phase 9), and contemporary guests.

Use `q35` for anything modern; reach for `pc` only for very old guests that choke on
PCIe.

### Versioned machine types and migration

Run `-machine help` and you'll see entries like `pc-q35-8.2`, `pc-q35-7.2`, plus an
unversioned alias `q35` that points at "whatever this QEMU's latest is." The
**versioned** types exist for one critical reason: **migration and hardware
stability**.

A versioned machine type *freezes* the exact virtual hardware layout — device
defaults, ACPI tables, quirks — as of that QEMU version. A guest is sensitive to
its hardware: if the virtual motherboard changes underneath it (different default
device revisions, different ACPI), the guest can misbehave or fail to migrate. So:

- You **pin** a VM to a versioned type (e.g. `pc-q35-8.2`) at creation.
- It then presents *identical* hardware no matter which QEMU version runs it,
  enabling **live migration** (Phase 12) between hosts running different QEMU
  builds, and stable behavior across upgrades.
- Changing a running/migrating VM's machine type changes its virtual hardware — the
  guest may see "new" devices, lose state, or refuse to resume. That's why you never
  bump it under a live guest.

### Accelerators: `-accel kvm` vs `-accel tcg`

- **`kvm`** — guest instructions run **natively** on the physical CPU via VT-x/AMD-V
  (Phase 2). Fast. Requires same-architecture guest and a ready host (Lesson 7).
- **`tcg`** — the **Tiny Code Generator**, QEMU's pure-software dynamic translator.
  It translates guest instructions to host instructions at runtime. Works for *any*
  guest architecture on any host, but is far slower. This is what you fall back to
  without hardware virtualization.
- **`hvf`** (macOS Hypervisor.framework) and **`whpx`** (Windows Hypervisor
  Platform) are the equivalents of KVM on those hosts.

You can list multiple with fallback: `-accel kvm:tcg` ("use KVM if available, else
TCG").

### `-cpu host` (preview)

With `-accel kvm`, `-cpu host` passes the host's full CPU feature set to the guest
for maximum performance. Great for single hosts, problematic for migration clusters
— that nuance is Lesson 17.

{: .note }
> **Why "q35" alone is risky for long-lived VMs**
> The bare alias `q35` resolves to the newest machine type of *whatever QEMU is
> installed*. Upgrade QEMU and the same VM definition silently gets a newer virtual
> motherboard — possibly breaking migration to a host with older QEMU, or changing
> device behavior. Production and migration setups always pin a *versioned* type
> like `pc-q35-8.2`. libvirt does this for you automatically when it creates a domain.

---

## Lab

```bash
# 1. List machine types — note the unversioned aliases AND versioned ones:
$ qemu-system-x86_64 -machine help
Supported machines are:
pc                   Standard PC (i440FX + PIIX, 1996) (alias of pc-i440fx-8.2)
pc-i440fx-8.2        Standard PC (i440FX + PIIX, 1996) (default)
q35                  Standard PC (Q35 + ICH9, 2009) (alias of pc-q35-8.2)
pc-q35-8.2           Standard PC (Q35 + ICH9, 2009)
pc-q35-7.2           ...
microvm              microvm (i386)
...

# 2. List accelerators available in this build:
$ qemu-system-x86_64 -accel help
Accelerators supported in QEMU binary:
tcg
kvm

# 3. Boot with a pinned, modern machine type + KVM:
$ qemu-system-x86_64 -machine q35,accel=kvm -cpu host -m 2G -smp 2 \
    -drive file=disk.qcow2,if=virtio -nographic &

# 4. Confirm from inside the running VM (or via the monitor) which machine it is.
#    Via the QEMU monitor (Ctrl-A C), or QMP (Lesson 15):
#    (qemu) info kvm
#    kvm support: enabled
#    (qemu) info qom-tree    # shows the q35 machine object

# 5. Contrast speed: force pure emulation (will be noticeably slower to boot):
$ qemu-system-x86_64 -machine q35,accel=tcg -m 2G -smp 2 \
    -drive file=disk.qcow2,if=virtio -nographic &
$ kill %1 %2 2>/dev/null
```

**Expected result:** `-machine help` shows both unversioned aliases and pinned
versioned types; `-accel help` lists `kvm` and `tcg`. The KVM boot is fast; the TCG
boot of the same image is markedly slower — the native-vs-emulated difference made
tangible.

---

## Further Reading

| Topic | Link |
|---|---|
| QEMU machine types / versioning | [qemu.org — Versioned machine types](https://www.qemu.org/docs/master/system/i386/pc.html) |
| q35 vs i440fx | [qemu.org — Q35 chipset](https://www.qemu.org/docs/master/system/i386/q35.html) |
| Tiny Code Generator (TCG) | [Wikipedia — QEMU (TCG)](https://en.wikipedia.org/wiki/QEMU#Tiny_Code_Generator) |
| Live migration & machine types | [qemu.org — Migration](https://www.qemu.org/docs/master/devel/migration/main.html) |
| KVM accelerator | [qemu.org — KVM](https://www.qemu.org/docs/master/system/i386/kvm-pv.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why do versioned machine types (`pc-q35-8.2`) exist? What breaks if you change a running VM's machine type on migration?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Versioned machine types freeze the exact virtual hardware layout (device defaults/revisions, ACPI tables, quirks) as of a given QEMU version, so a VM presents identical hardware regardless of which QEMU build runs it. That stability is what makes live migration between hosts with different QEMU versions possible and keeps guest behavior consistent across upgrades. If you change a VM's machine type, its virtual hardware changes underneath the running guest — it may see different/new devices, lose device state, mis-handle ACPI, or refuse to resume — which breaks migration and can crash the guest.
</details>

---

**Q2. When should you use `q35` versus `pc` (i440FX)?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Use q35 (ICH9 chipset) for essentially all modern guests: it provides a real PCI Express topology, which is needed for PCIe devices and good device assignment/passthrough, and it matches contemporary hardware expectations. Use pc (i440FX) only for very old or peculiar guests that don't handle PCIe well and need the legacy 1996-era chipset. q35 is the modern default.
</details>

---

**Q3. What is the difference between `-accel kvm` and `-accel tcg`, and when must you use TCG?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`-accel kvm` runs guest instructions natively on the physical CPU using hardware virtualization (VT-x/AMD-V) — fast, but requires the guest architecture to match the host and a KVM-ready host. `-accel tcg` uses QEMU's Tiny Code Generator to translate guest instructions to host instructions in software — much slower, but works for any guest architecture on any host. You must use TCG when there's no hardware virtualization available (disabled/absent, or non-nested VM) or when emulating a *different* architecture than the host (e.g. ARM guest on x86).
</details>

---

## Homework

Run `qemu-system-x86_64 -machine help` and find the unversioned `q35` alias and the specific versioned type it currently points to on your system. Then explain, for a two-host migration cluster where the hosts run *different* QEMU versions, why you'd configure both VMs with the *same explicit* versioned type rather than the bare `q35` alias.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The bare <code>q35</code> alias resolves to each host's newest machine type, so on hosts with different QEMU versions the same VM definition would get *different* virtual motherboards — different device defaults/ACPI — which breaks migration because source and destination must present identical hardware to the guest. Pinning both to the *same explicit* versioned type (e.g. pc-q35-7.2, supported by both QEMU versions) guarantees the virtual hardware is byte-for-byte the same on either host, so the guest can migrate and resume seamlessly. You generally pick the oldest version both hosts support as the common baseline.
</details>
