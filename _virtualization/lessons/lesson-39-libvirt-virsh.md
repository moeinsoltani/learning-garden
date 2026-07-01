---
title: "Lesson 39 — libvirt Architecture and virsh"
nav_order: 39
parent: "Phase 10: libvirt"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 39: libvirt Architecture and virsh

## Concept

You can run a VM with a 12-line QEMU command — but you have to retype it every time, and
nothing remembers it exists. **libvirt** is a management layer that gives VMs a
persistent identity, a stable API, and a uniform toolset, while *generating the QEMU
command line for you* under the hood.

```
   you / tools                          ┌──────── the host ────────┐
   ┌──────────┐   libvirt API           │  libvirtd / modular      │
   │  virsh   │ ─────────────────────►   │  daemons (virtqemud …)   │
   │ virt-mgr │   (local or remote       │      │ generates           │
   │ openstack│    over qemu:///…)       │      ▼                     │
   └──────────┘                          │   qemu-system-x86_64 ...   │ ◄ actual VM
                                         │   (driven via QMP)         │
                                         └────────────────────────────┘
```

libvirt is a stable abstraction *over* QEMU/KVM (and Xen, LXC, …). You describe a VM
once as a **domain**, and libvirt persists it, starts/stops it, and talks to the running
QEMU via QMP (Lesson 15).

---

## How It Works

### The daemons

Historically a single **`libvirtd`**. Modern libvirt uses **modular daemons**, each
handling one area: **`virtqemud`** (QEMU/KVM domains), **`virtnetworkd`** (networks),
**`virtstoraged`** (storage), **`virtnodedevd`** (host devices), etc. Benefits: smaller
attack surface, independent restart, and the API is **stateless** in the sense that the
daemon reflects/persists configuration rather than holding all VM logic in one process.

### Drivers and connection URIs

libvirt speaks to multiple hypervisors via **drivers** (qemu/kvm, lxc, xen, …). You pick
which, and *where*, with a **connection URI**:

- **`qemu:///system`** — the **system-wide** instance: VMs run as the privileged libvirt
  QEMU user, can use host bridges, system storage, and start at boot. This is what you
  use for "real" server VMs. Requires privileges (libvirt/`libvirt` group).
- **`qemu:///session`** — a **per-user** instance: VMs run as *you*, with user-owned
  storage and (by default) user-mode networking. No root needed, but limited (e.g. no
  system bridge by default). Good for unprivileged desktop use.
- **Remote URIs** — `qemu+ssh://host/system` manages a remote host over SSH, using the
  exact same commands. This uniformity is a major libvirt win.

### virsh — the command-line client

`virsh` is the primary CLI. The essentials:

| Command | Does |
|---|---|
| `virsh list --all` | List domains (running and defined). |
| `virsh define vm.xml` | Persistently register a domain from XML. |
| `virsh start <dom>` | Boot a defined domain. |
| `virsh shutdown <dom>` | Graceful ACPI shutdown. |
| `virsh destroy <dom>` | Force-stop (pull the plug) — does **not** delete it. |
| `virsh undefine <dom>` | Remove the persistent definition. |
| `virsh edit <dom>` | Edit the domain XML (Lesson 40). |
| `virsh dominfo <dom>` | State, vCPUs, memory, etc. |

Note the **define/undefine** vs **start/destroy** distinction: *define* manages the
*persistent definition*; *start/destroy* manage the *running instance*. `destroy` sounds
scary but only force-stops; `undefine` is what deletes the config.

### libvirt generates the QEMU command line

The payoff: you write portable XML, and libvirt computes the exact QEMU invocation —
including machine type pinning, device addressing, and security labels. You can *see*
this translation:

```
   virsh domxml-to-native qemu-argv vm.xml
   # prints the literal qemu-system-x86_64 ... command libvirt would run
```

This is the bridge between everything you learned in Phases 4–9 (raw flags) and the
managed world (XML).

{: .note }
> **qemu:///system vs qemu:///session in one line**
> <code>system</code> = one privileged host-wide instance (root/libvirt-managed QEMU,
> host bridges, system storage, autostart at boot) — for servers and anything needing
> real networking/storage. <code>session</code> = your own unprivileged instance (runs as
> you, user storage, user-mode networking by default) — for desktop/dev use without root.
> Picking the wrong one is a classic "my VM can't reach the bridge" or "permission
> denied" confusion.

---

## Lab

```bash
# 1. Connect and list (note the URI; default is usually qemu:///system if privileged):
$ virsh -c qemu:///system list --all
 Id   Name      State
-------------------------
 -    web01     shut off
 3    db01      running

# 2. Inspect a domain and its lifecycle:
$ virsh dominfo db01
Id:             3
Name:           db01
State:          running
CPU(s):         4
Max memory:     8388608 KiB

# 3. Start / graceful shutdown / force-stop (destroy != delete):
$ virsh start web01
$ virsh shutdown web01        # ACPI graceful
$ virsh destroy web01         # force stop (config still defined)
$ virsh list --all            # web01 shows 'shut off', still listed

# 4. See the modular daemons running:
$ systemctl status virtqemud virtnetworkd virtstoraged 2>/dev/null | grep -E 'Active|●'

# 5. THE key insight: have libvirt show the raw QEMU command it would run:
$ virsh domxml-to-native qemu-argv --domain db01
/usr/bin/qemu-system-x86_64 -name guest=db01 -machine pc-q35-8.2,accel=kvm ... \
  -m 8192 -smp 4,sockets=1,cores=4,threads=1 -device virtio-net-pci,... \
  -blockdev ... -device virtio-blk-pci,... -monitor ... (QMP socket)

# 6. Remote management over SSH with the SAME commands:
$ virsh -c qemu+ssh://user@otherhost/system list --all
```

**Expected result:** `virsh list --all` shows defined and running domains; the modular
daemons (virtqemud etc.) are active; and `virsh domxml-to-native` reveals the literal
QEMU command libvirt generates from the domain — connecting the managed layer back to the
raw flags of Phases 4–9.

---

## Further Reading

| Topic | Link |
|---|---|
| libvirt | [Wikipedia — libvirt](https://en.wikipedia.org/wiki/Libvirt) |
| libvirt connection URIs | [libvirt.org — Connection URIs](https://libvirt.org/uri.html) |
| `virsh` | [libvirt.org — virsh](https://libvirt.org/manpages/virsh.html) |
| libvirt modular daemons | [libvirt.org — Daemons](https://libvirt.org/daemons.html) |
| QEMU driver | [libvirt.org — QEMU driver](https://libvirt.org/drvqemu.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What's the difference between `qemu:///system` and `qemu:///session`, and when would you use each?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
qemu:///system is the host-wide, privileged libvirt instance: VMs run under the libvirt QEMU user, can use host bridges and system storage pools, and can autostart at boot — used for servers and any VM needing real networking/storage (requires privileges / libvirt group membership). qemu:///session is a per-user, unprivileged instance: VMs run as your user with user-owned storage and user-mode (SLIRP) networking by default, needing no root — good for desktop/development use, but limited (e.g. no system bridge by default). You pick system for production/serious VMs and session for quick unprivileged personal VMs.
</details>

---

**Q2. In libvirt, what's the difference between `destroy`/`start` and `define`/`undefine`?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
start and destroy manage the *running instance*: start boots a defined domain, destroy force-stops it (pulls the virtual plug) but leaves the definition intact. define and undefine manage the *persistent definition*: define registers a domain from XML so the host remembers it, undefine removes that stored configuration. So "destroy" does NOT delete the VM — it only stops it; "undefine" is what removes the VM's definition. A VM can be defined but shut off, or transient (running without a persistent definition).
</details>

---

**Q3. How can you see the actual QEMU command line libvirt generates from a domain, and why is that useful?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Run <code>virsh domxml-to-native qemu-argv --domain &lt;name&gt;</code> (or feed it an XML file), which prints the literal qemu-system-x86_64 invocation libvirt would execute. It's useful because it bridges the managed XML world to the raw QEMU flags from Phases 4–9: you can verify exactly what machine type, devices, addressing, caching, and security options libvirt chose, debug why a VM behaves a certain way, and learn how high-level XML maps onto concrete QEMU options.
</details>

---

## Homework

Define or pick a domain and run `virsh domxml-to-native qemu-argv --domain <name>`. In the output, locate (a) the machine type (is it a pinned versioned type like `pc-q35-8.2`?), (b) the accelerator (kvm?), and (c) how a disk is expressed (`-blockdev`/`-device`). Map each back to the lesson in Phases 4–6 that introduced it. Then explain one thing libvirt added that you didn't have to write yourself.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) The machine type appears as a pinned versioned type (e.g. -machine pc-q35-8.2,accel=kvm) — Lesson 13; libvirt pins it for migration stability rather than using the bare alias. (b) accel=kvm is the KVM accelerator — Lesson 13. (c) The disk is expressed as a -blockdev backend plus a -device virtio-blk-pci/virtio-scsi frontend referencing it — Lessons 12/24. Things libvirt added for free include: a QMP monitor socket (Lesson 15), explicit PCI device addressing, an OVMF/pflash setup if UEFI, sVirt security labels (Lesson 56), and sensible cache/aio defaults — all generated from the XML so you didn't hand-write them. (Any one such addition is a valid answer.)
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 40 — The Domain XML →](lesson-40-domain-xml){: .btn .btn-primary }
