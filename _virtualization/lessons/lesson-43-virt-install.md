---
title: "Lesson 43 — virt-install and the Desktop Tools"
nav_order: 43
parent: "Phase 10: libvirt"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 43: virt-install and the Desktop Tools

## Concept

You *could* hand-write domain XML (Lesson 40), but nobody does for the common case.
**`virt-install`** creates a complete VM — disk, network, install media, sensible
devices — from a single command, and hands it to libvirt. Around it sits a family of
tools: **`virt-manager`** (GUI), **`virt-viewer`/`remote-viewer`** (display clients), and
**osinfo** (the database that tells the tools what devices each guest OS likes).

```
   virt-install  ──► generates domain XML ──► defines + starts a libvirt domain
   (one command)         (Lesson 40)              (Lesson 39)
        │
        └─ --osinfo ubuntu24.04  ──► osinfo DB ──► picks virtio, q35, proper RAM/CPU
                                                    defaults for THAT guest OS
```

The killer feature is **`--osinfo`** (formerly `--os-variant`): telling the tools which
guest OS you're installing makes them choose correct, performant device defaults
automatically.

---

## How It Works

### virt-install — scripted VM creation

A single command specifies the VM and the install source:

```
   virt-install \
     --name web01 --memory 4096 --vcpus 2 \
     --osinfo ubuntu24.04 \
     --disk size=20,pool=default,format=qcow2 \
     --network network=default,model=virtio \
     --cdrom /isos/ubuntu-24.04.iso          # or --location, --import, --cloud-init
```

virt-install translates this into domain XML (with virtio devices, a q35 machine, a
console, etc.), defines the domain, and launches the installer. Variants:

- **`--cdrom` / `--location`** — boot an installer ISO / network install tree.
- **`--import`** — skip installation; wrap an *existing* disk image (e.g. a cloud image,
  Phase 11) into a VM.
- **`--cloud-init`** — inject cloud-init config for unattended setup (Lesson 44).
- **`--pxe`** — network boot.

### The desktop tools

- **`virt-manager`** — a GTK GUI over libvirt: create, monitor, console into, and edit
  VMs visually. Great for learning and ad-hoc management; it talks the same libvirt API,
  including to remote hosts.
- **`virt-viewer` / `remote-viewer`** — connect to a VM's graphical console (SPICE/VNC,
  Lesson 16). `virt-viewer <domain>` finds the display via libvirt automatically.

### osinfo — why telling it the OS matters

The **osinfo** database (`osinfo-query os`) knows, for each guest OS: which device models
it supports (does it have virtio drivers built in?), recommended RAM/CPU/disk, and
firmware preferences. When you pass `--osinfo`:

- A modern Linux → virtio everything, q35, sensible memory.
- An old OS without virtio drivers → falls back to compatible devices (SATA/e1000) so it
  can actually boot and install.
- Windows → appropriate models plus hints (you'd still add virtio-win, Lesson 28).

**Omitting or mis-setting `--osinfo`** can leave you with slow emulated devices, or a
guest that can't find its disk because it lacks drivers for the chosen controller. So
`--osinfo` is what gets you correct *and* fast defaults without manual tuning.

### Lifecycle helpers

- **Autostart:** `virsh autostart <dom>` starts the VM at host boot.
- **Managed save:** `virsh managedsave <dom>` saves RAM state to disk and stops; next
  start restores it (like hibernate).
- **Lifecycle hooks:** scripts in `/etc/libvirt/hooks/` run on VM/network/storage events
  (used, e.g., for single-GPU passthrough, Lesson 37).

{: .note }
> **Why --osinfo matters for device selection**
> virt-install uses the osinfo database to pick devices the *specific guest OS* can drive.
> Tell it the OS and it enables virtio (fast paravirtual) for an OS that has the drivers,
> or falls back to emulated SATA/e1000 for one that doesn't — so the VM both boots and
> performs well. Get it wrong (or omit it) and you may get a guest that's slow (emulated
> devices when virtio was possible) or unbootable (virtio disk an old OS can't see). It's
> the difference between sensible automatic defaults and a misconfigured VM.

---

## Lab

```bash
# 1. Find the osinfo identifier for your guest OS:
$ osinfo-query os | grep -i ubuntu
 ubuntu24.04 | Ubuntu 24.04 | 24.04 | http://ubuntu.com/ubuntu/24.04
$ osinfo-query os | grep -i 'win'      # e.g. win11, win2k22

# 2. Create a VM from an installer ISO in one command:
$ virt-install \
    --name web01 --memory 4096 --vcpus 2 \
    --osinfo ubuntu24.04 \
    --disk size=20,pool=default,format=qcow2 \
    --network network=default,model=virtio \
    --graphics spice \
    --cdrom /isos/ubuntu-24.04-live-server.iso
# virt-install defines the domain, starts it, and opens a console (virt-viewer).

# 3. IMPORT an existing image (e.g. a cloud image) instead of installing:
$ virt-install \
    --name app01 --memory 2048 --vcpus 2 \
    --osinfo ubuntu24.04 \
    --disk /var/lib/libvirt/images/ubuntu-cloud.qcow2,device=disk \
    --network network=default,model=virtio \
    --import --noautoconsole

# 4. Confirm libvirt now manages it, and see the generated XML:
$ virsh list --all
$ virsh dumpxml web01 | grep -E 'machine|model type|<disk'

# 5. Lifecycle helpers:
$ virsh autostart web01           # start at host boot
$ virsh managedsave web01         # hibernate-like save+stop
$ virsh start web01               # restores from managed save

# 6. Open the graphical console any time:
$ virt-viewer web01
# Or use the GUI:
$ virt-manager
```

**Expected result:** A single `virt-install` command builds and starts a fully-formed VM
with virtio devices and a q35 machine (chosen via `--osinfo`), defined in libvirt. The
`--import` form wraps an existing image without reinstalling, and `virt-viewer`/
`virt-manager` give you console/GUI access.

---

## Further Reading

| Topic | Link |
|---|---|
| `virt-install` | [man7.org / virt-manager.org — virt-install](https://virt-manager.org/) |
| `virt-manager` | [virt-manager.org](https://virt-manager.org/) |
| libosinfo / osinfo-query | [libosinfo.org](https://libosinfo.org/) |
| `virt-viewer` | [virt-manager.org — virt-viewer](https://virt-manager.org/) |
| libvirt hooks | [libvirt.org — Hooks](https://libvirt.org/hooks.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why does telling `virt-install` the correct guest OS (`--osinfo`) matter for the devices it picks?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
virt-install consults the osinfo database to choose device models the specific guest OS can actually drive. With the right --osinfo, it enables virtio (fast paravirtual disk/NIC) for an OS that has those drivers built in, or falls back to compatible emulated devices (SATA/e1000) for an older OS that doesn't — and sets sane RAM/CPU/firmware defaults. Get it wrong or omit it and you risk a slow VM (emulated devices where virtio was possible) or an unbootable one (a virtio disk an old OS can't see). So --osinfo is what produces correct *and* performant defaults automatically.
</details>

---

**Q2. What does `virt-install --import` do differently from a normal install, and when is it useful?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
--import skips the OS installation step entirely: instead of booting an installer ISO, it wraps an *existing* disk image into a new libvirt domain and boots straight from it. It's useful for cloud images (Phase 11) and pre-built/templated disks — you already have a ready-to-run image, so you just need libvirt to define a VM around it with the right devices. Combined with --osinfo and --cloud-init, it's the fast path to launching cloud-image-based VMs without ever touching an installer.
</details>

---

**Q3. What does `virsh managedsave` do, and how does it differ from a normal shutdown?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
managedsave saves the running VM's RAM/CPU state to disk and stops the VM — like hibernate. On the next <code>virsh start</code>, libvirt restores that saved state so the guest resumes exactly where it was, rather than booting fresh. A normal shutdown powers the guest off cleanly with no saved memory state, so the next start is a full boot. managedsave preserves the in-memory session; shutdown discards it.
</details>

---

## Homework

Use `osinfo-query os` to find the identifier for an OS you have an ISO or cloud image for. Create a VM two ways: once with the correct `--osinfo`, and once with a deliberately wrong/old value (e.g. an ancient OS id). Compare the generated `virsh dumpxml` (machine type, disk bus, NIC model) between the two. Explain how the device choices differ and why the correct `--osinfo` produces a better VM.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With the correct, modern --osinfo, dumpxml shows a q35 machine and virtio devices (disk bus='virtio', NIC model='virtio') plus appropriate defaults. With an old/wrong osinfo value, virt-install assumes the guest lacks virtio drivers and falls back to compatible emulated hardware — e.g. an i440fx/pc machine, a SATA or IDE disk bus, and an e1000/rtl8139 NIC. The device choices differ because osinfo encodes each OS's driver support and preferences. The correct --osinfo produces a better VM because virtio devices cause far fewer VM exits and run much faster (Phase 7), and the modern machine type/defaults match what the OS expects — whereas the wrong value either needlessly uses slow emulated devices or, in the opposite case (claiming virtio for an OS that lacks it), could leave the guest unable to see its disk.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 44 — Cloud Images and cloud-init →](lesson-44-cloud-init){: .btn .btn-primary }
