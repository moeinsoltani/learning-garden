---
title: "Lesson 46 — Templates and Cloning"
nav_order: 46
parent: "Phase 11: Guest Images & Provisioning"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 46: Templates and Cloning

## Concept

With a clean golden image (Lesson 45) you can mass-produce VMs by **cloning**. There are
two kinds, built on the storage knowledge from Phase 6:

```
   FULL CLONE                           LINKED CLONE
   ──────────                           ────────────
   golden.qcow2 ──copy──► vm1.qcow2      golden.qcow2 (read-only base)
                ──copy──► vm2.qcow2          ▲ backing       ▲ backing
   each is a standalone, independent     vm1-overlay.qcow2  vm2-overlay.qcow2
   image (big, slow to create)          tiny overlays (instant, share the base)
```

A **full clone** is an independent copy (Lesson 22 `convert`). A **linked clone** is just
a copy-on-write overlay over the shared base (Lesson 23) — instant and tiny, but tied to
the base. The golden workflow: **customize → sysprep → template → clone**.

---

## How It Works

### Full clone vs linked clone

- **Full clone:** a complete, standalone copy of the base disk. Takes time and full disk
  space, but is independent — deleting the base doesn't affect it, and there's no
  chain-traversal read cost. Best for long-lived, performance-sensitive VMs.
- **Linked clone:** a new qcow2 **overlay** whose backing file is the read-only golden
  image (Lesson 23). Creating it is instant and it consumes almost no space initially
  (only each VM's own writes land in its overlay). The catch (from Lesson 23): it
  **depends on the base** — the base must remain present and unchanged, and reads of
  unmodified data traverse the chain (a slight cost). Ideal for many short-lived/identical
  VMs (CI runners, VDI, lab fleets).

### The golden image workflow

1. **Customize** — install base packages/config (`virt-customize`, Lesson 45) or boot and
   configure a VM normally.
2. **Sysprep** — `virt-sysprep` to strip machine-specific identity (Lesson 45).
3. **Template** — keep the sysprepped image read-only as the golden master; in libvirt you
   might mark a volume as a template or just keep it pristine.
4. **Clone** — produce per-VM disks (full or linked) from it.

### Cloning tools

- **`virt-clone`** — clones a *libvirt domain*: copies/links its disk(s), and crucially
  **generates new MAC addresses** and a new UUID/name so the clone isn't a network twin.

  ```
  virt-clone --original golden-vm --name web02 --auto-clone
  ```

- **libvirt volume cloning** — `virsh vol-clone` copies a volume within a pool (Lesson 41).
- **qemu-img for linked clones** — `qemu-img create -b golden.qcow2 -F qcow2
  web02.qcow2` makes the overlay (Lesson 23), which you then wrap in a domain.

### Identity collisions — the thing that bites you

If you clone naively (just copy the disk and define a new VM), clones collide on identity:

- **MAC addresses** — duplicate MACs on the same L2 → ARP chaos, dropped traffic.
  `virt-clone` regenerates MACs; otherwise you must edit the XML.
- **machine-id** — duplicates cause DHCP lease collisions and systemd/journald confusion.
  Sysprep clears it so each clone regenerates one on first boot.
- **SSH host keys** — duplicates mean clients can't distinguish hosts and one stolen key
  unlocks all. Sysprep removes them; first boot regenerates unique keys.
- **Hostname** — set per-clone (via cloud-init, Lesson 44, or virt-customize).

The combination **sysprep (clear identity) + virt-clone (new MAC/UUID) + cloud-init (set
per-VM hostname/keys)** is the clean recipe.

{: .note }
> **Linked clone: what it's built on, and the downside**
> A linked clone is exactly a qcow2 backing-file overlay (Lesson 23) over the shared
> golden image: instant to create and near-zero initial disk use because only the clone's
> own writes occupy space. The downside is dependency and performance: every clone needs
> the base to stay present and unmodified (delete or alter it and all clones break), reads
> of unchanged blocks fall through the chain (slightly slower), and many overlays on one
> base concentrate I/O on it. Full clones avoid these by being independent, at the cost of
> space and creation time.

---

## Lab

```bash
# 0. Start from a sysprepped golden image (Lesson 45): golden.qcow2 (read-only master)
$ chmod -w golden.qcow2     # protect the base

# ---- LINKED clones (instant, tiny) via qcow2 overlays (Lesson 23) ----
$ qemu-img create -f qcow2 -b golden.qcow2 -F qcow2 web01.qcow2
$ qemu-img create -f qcow2 -b golden.qcow2 -F qcow2 web02.qcow2
$ qemu-img info web01.qcow2 | grep -E 'backing|disk size'
disk size: 196 KiB                    ← near-zero; shares the base
backing file: golden.qcow2

# ---- FULL clone (independent copy) ----
$ qemu-img convert -O qcow2 golden.qcow2 standalone01.qcow2
$ qemu-img info standalone01.qcow2 | grep backing || echo "no backing — independent"

# ---- Clone a libvirt DOMAIN with fresh identity (new MAC/UUID/name) ----
# Define a golden domain first (shut off), then:
$ virt-clone --original golden-vm --name web03 --auto-clone
Allocating 'web03.qcow2'        ...
Clone 'web03' created successfully.
$ virsh dumpxml web03 | grep -E 'mac address|<uuid>'
  <uuid>...new-uuid...</uuid>
  <mac address='52:54:00:NEW:MAC:01'/>     ← regenerated, no collision

# ---- Per-VM identity on first boot via cloud-init (Lesson 44) ----
# Attach a per-clone seed.iso setting a unique hostname so they don't all say 'golden'.

# ---- Demonstrate the linked-clone dependency danger ----
$ ls -l golden.qcow2          # the base both linked clones depend on
# If you (wrongly) deleted/modified golden.qcow2, web01/web02 would break — they only
# hold diffs. Full clones (standalone01.qcow2) would be unaffected.
```

**Expected result:** Linked clones are created instantly with near-zero disk use and a
`backing file: golden.qcow2`; a full clone is a standalone image with no backing.
`virt-clone` produces a domain with a fresh MAC and UUID, avoiding identity collisions.

---

## Further Reading

| Topic | Link |
|---|---|
| `virt-clone` | [virt-manager.org — virt-clone](https://virt-manager.org/) |
| qcow2 backing files (Lesson 23) | [qcow2 internals]({{ '/virtualization/lessons/lesson-23-qcow2-internals.html' | relative_url }}) |
| `virt-sysprep` | [libguestfs.org — virt-sysprep](https://libguestfs.org/virt-sysprep.1.html) |
| machine-id | [man7.org — machine-id(5)](https://man7.org/linux/man-pages/man5/machine-id.5.html) |
| Golden image / system image | [Wikipedia — System image](https://en.wikipedia.org/wiki/System_image) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. A linked clone starts instantly and uses almost no disk. What is it built on (from Phase 6), and what's the downside?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It's built on a qcow2 backing-file overlay (Lesson 23): the linked clone is a new overlay whose backing file is the read-only golden image, so it starts instantly and uses almost no space because only the clone's own writes land in its overlay. The downside is dependency and some performance cost: it requires the base to stay present and unchanged (deleting or modifying the base breaks every clone, since the overlay only holds the diff), reads of unmodified blocks must traverse the backing chain (slightly slower), and many overlays concentrate read I/O on the single shared base. Full clones avoid these by being independent copies, at the cost of space and creation time.
</details>

---

**Q2. What identity collisions occur if you clone a VM naively, and how does the customize→sysprep→clone workflow prevent each?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Naive cloning duplicates: MAC addresses (duplicate MACs on one L2 cause ARP chaos/dropped traffic), /etc/machine-id (DHCP lease collisions and duplicate systemd/journald identity), SSH host keys (clients can't distinguish hosts; one stolen key compromises all), and hostnames (all clones identical). The workflow prevents them: virt-sysprep strips machine-id and SSH host keys (regenerated uniquely on first boot) plus other stale state; virt-clone generates a fresh MAC and UUID for the cloned domain; and cloud-init (or virt-customize) sets a unique hostname (and can inject per-VM keys) on first boot. Together they ensure each clone is a distinct machine.
</details>

---

**Q3. When would you choose a full clone over a linked clone?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Choose a full clone for long-lived and/or performance-sensitive VMs that should be independent: it's a standalone copy, so it doesn't depend on the base (the base can be deleted/changed safely), there's no chain-traversal read penalty, and I/O isn't concentrated on a shared base. The cost is full disk space and slower creation. Linked clones are better for many short-lived or identical VMs (CI runners, VDI, lab fleets) where instant creation and space savings matter more than independence and peak I/O.
</details>

---

## Homework

Create two linked clones from a sysprepped golden image and one full clone. Compare their `qemu-img info` (disk size and backing file). Boot the clones (ideally with per-VM cloud-init seeds) and confirm they each get a unique hostname and machine-id. Then explain what would happen to the linked clones if you deleted the golden base, versus the full clone.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The two linked clones show near-zero disk size and <code>backing file: golden.qcow2</code>; the full clone shows its full data size and no backing file. Booted with per-VM cloud-init seeds (and from a sysprepped base), each clone generates a unique hostname (from user-data), a unique machine-id, and unique SSH host keys on first boot — no collisions. If you deleted the golden base, both linked clones would break: they only contain their own diffs and rely on the base for all unchanged data, so reads that fall through to the (now-missing) base fail and the VMs become unusable. The full clone is unaffected because it's a complete, independent image with no dependency on the base.
</details>
