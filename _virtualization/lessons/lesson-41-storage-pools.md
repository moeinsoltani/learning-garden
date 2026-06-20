---
title: "Lesson 41 — Storage Pools and Volumes"
nav_order: 41
parent: "Phase 10: libvirt"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 41: Storage Pools and Volumes

## Concept

In Phase 6 you created disk images by hand with `qemu-img` and tracked their paths
yourself. libvirt abstracts that with **pools** and **volumes**: a **pool** is a *source
of storage* (a directory, an LVM volume group, an NFS export, a Ceph cluster), and a
**volume** is an individual disk *within* a pool. libvirt manages allocation, naming,
permissions, and lifecycle for you.

```
   POOL (a storage backend)                VOLUMES (disks in it)
   ┌──────────────────────────┐
   │ pool: default (dir)       │  ──►  db01.qcow2   (10G)
   │   /var/lib/libvirt/images │  ──►  web01.qcow2  (20G)
   ├──────────────────────────┤
   │ pool: fast (LVM VG)       │  ──►  lv-cache     (raw LV)
   ├──────────────────────────┤
   │ pool: shared (NFS)        │  ──►  migratable.qcow2
   └──────────────────────────┘
```

The point: you stop juggling `qemu-img create` and absolute paths, and instead say
"make a 20 GiB volume in pool *fast*."

---

## How It Works

### Pool types

A pool's **type** determines where/how storage comes from:

| Type | Backing |
|---|---|
| **dir** | A host directory of image files (qcow2/raw). The simplest, default `/var/lib/libvirt/images`. |
| **logical (LVM)** | An LVM volume group; volumes are logical volumes (raw block devices). |
| **netfs (NFS)** | A mounted NFS export — useful for **shared storage** (live migration, Lesson 49). |
| **iscsi** | An iSCSI target; volumes are LUNs. |
| **rbd (Ceph)** | A Ceph RBD pool — scalable, shared, replicated. |
| **zfs** | A ZFS dataset/zvol. |

### Managing pools and volumes

```
   virsh pool-define-as default dir --target /var/lib/libvirt/images
   virsh pool-build default        # create the dir/structure if needed
   virsh pool-start default        # activate it
   virsh pool-autostart default    # start at boot
   virsh pool-list --all

   virsh vol-create-as default db01.qcow2 10G --format qcow2   # make a volume
   virsh vol-list default
   virsh vol-clone db01.qcow2 db02.qcow2 --pool default        # clone a volume
   virsh vol-info db01.qcow2 --pool default
```

A domain's `<disk>` then references the pool/volume instead of a raw path, and libvirt
resolves it. Cloning, resizing, and deletion go through `vol-*` commands rather than
manual `qemu-img`.

### Relationship to Phase 6 formats

Pools don't replace qcow2/raw — they **manage** them. A `dir` pool's volumes are still
qcow2 or raw files (the formats from Lesson 22), and backing-file overlays (Lesson 23)
still work. An `logical` (LVM) pool's volumes are raw block devices (no qcow2 layer —
you'd snapshot via LVM instead). So the pool is an organizational/lifecycle layer on top
of the storage concepts you already know.

### Permissions and sVirt labelling (preview Phase 14)

libvirt doesn't just track files — it manages their **security context**. When a domain
starts, libvirt sets ownership/permissions and applies an **sVirt** label (SELinux/
AppArmor, Lesson 56) to the volume so *only that VM's QEMU process* can access it, then
restores it on stop. Doing storage through pools/volumes lets libvirt handle this
labelling automatically; hand-placed files outside a pool may need manual permission/
label fixes (the classic "permission denied" on a manually-created image).

{: .note }
> **What a pool abstracts that bare qemu-img didn't**
> With raw qemu-img you manually chose paths, created files, set ownership/permissions,
> remembered where everything lived, and cloned by hand. A libvirt <em>pool</em> abstracts
> the storage <em>source</em> (dir/LVM/NFS/Ceph) and provides uniform create/clone/resize/
> delete/list operations, automatic naming and capacity tracking, correct permissions and
> sVirt labelling per VM, and autostart — all behind the same API whether the backend is a
> local directory or a Ceph cluster. The same XML/commands work across wildly different
> storage.

---

## Lab

```bash
# 1. List existing pools (the 'default' dir pool usually exists):
$ virsh pool-list --all
 Name      State    Autostart
-------------------------------
 default   active   yes

$ virsh pool-info default
Name:           default
State:          running
Capacity:       500.00 GiB
Available:      420.00 GiB

# 2. Create a volume in the pool (replaces manual qemu-img create):
$ virsh vol-create-as default app01.qcow2 20G --format qcow2
Vol app01.qcow2 created
$ virsh vol-list default
 Name           Path
------------------------------------------------------------
 app01.qcow2    /var/lib/libvirt/images/app01.qcow2

# 3. Inspect it (libvirt-tracked, but still a qcow2 from Phase 6):
$ virsh vol-info app01.qcow2 --pool default
Type:           file
Capacity:       20.00 GiB
Allocation:     200.00 KiB        ← thin, just like Lesson 22

# 4. Clone a volume (libvirt handles the copy + naming):
$ virsh vol-clone app01.qcow2 app02.qcow2 --pool default

# 5. Define a NEW pool — e.g. an NFS pool for shared storage (migration):
$ virsh pool-define-as shared netfs \
    --source-host nfs.example.com --source-path /export/vmstore \
    --target /mnt/libvirt-shared
$ virsh pool-build shared
$ virsh pool-start shared
$ virsh pool-autostart shared

# 6. Reference a volume from a domain disk (libvirt resolves pool+vol -> path):
#   <disk type='volume' device='disk'>
#     <source pool='default' volume='app01.qcow2'/>
#     <target dev='vda' bus='virtio'/>
#   </disk>
```

**Expected result:** `virsh pool-list` shows your pools and capacity; `vol-create-as`
makes a thin qcow2 volume (same format as Phase 6) without manual `qemu-img`; cloning and
new pool types (NFS for shared storage) are all driven through uniform `virsh` commands.

---

## Further Reading

| Topic | Link |
|---|---|
| libvirt storage management | [libvirt.org — Storage management](https://libvirt.org/storage.html) |
| Storage pool XML | [libvirt.org — Storage pool format](https://libvirt.org/formatstorage.html) |
| `virsh` pool/vol commands | [libvirt.org — virsh](https://libvirt.org/manpages/virsh.html) |
| LVM | [Wikipedia — Logical Volume Manager (Linux)](https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)) |
| Ceph RBD | [docs.ceph.com — RBD](https://docs.ceph.com/en/latest/rbd/) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What does a libvirt storage *pool* abstract that you were doing by hand with `qemu-img` earlier?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A pool abstracts the storage *source* and its lifecycle. With bare qemu-img you manually picked paths, created files, set ownership/permissions, tracked where images lived, and cloned by hand. A pool gives uniform create/clone/resize/delete/list operations, automatic naming and capacity tracking, correct permissions plus per-VM sVirt labelling, and autostart — all behind one API regardless of whether the backend is a local directory, LVM VG, NFS export, iSCSI target, or Ceph cluster. It turns "remember this path and run qemu-img" into "make a volume in this pool."
</details>

---

**Q2. Name three pool types and a situation where each is appropriate.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Examples: (1) <strong>dir</strong> — a host directory of qcow2/raw files; appropriate for simple single-host setups (the default). (2) <strong>netfs (NFS)</strong> — a shared NFS export; appropriate for shared storage so VMs can be live-migrated between hosts. (3) <strong>logical (LVM)</strong> — an LVM volume group with raw LV volumes; appropriate when you want block-level performance and LVM snapshots. (Others: iscsi for SAN LUNs, rbd/Ceph for scalable replicated shared storage, zfs for ZFS-backed volumes.)
</details>

---

**Q3. How do pools relate to the qcow2/raw formats from Phase 6 — do they replace them?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
They don't replace formats; they manage them. A dir pool's volumes are still qcow2 or raw files (Lesson 22), and qcow2 features like backing-file overlays (Lesson 23) still apply. An LVM pool's volumes are raw block devices (so you'd snapshot via LVM rather than qcow2). The pool is an organizational and lifecycle layer — naming, allocation, permissions, security labels — sitting on top of the same underlying storage formats and concepts you already learned.
</details>

---

## Homework

Create a volume with `virsh vol-create-as default test.qcow2 5G --format qcow2`, then run both `virsh vol-info test.qcow2 --pool default` and `qemu-img info /var/lib/libvirt/images/test.qcow2`. Compare what each tells you. Then explain why creating the image *inside* a pool is preferable to dropping a hand-made qcow2 into the same directory and pointing a domain at it.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Both report the same underlying qcow2 (virtual capacity 5G, thin allocation), because the pool volume *is* a qcow2 file — vol-info shows libvirt's tracked view (pool, capacity, allocation) while qemu-img info shows the raw format metadata directly. Creating inside the pool is preferable because libvirt then knows the volume exists (it appears in vol-list, capacity tracking is accurate), and it applies correct ownership/permissions and per-VM sVirt labels when the domain starts and restores them on stop. A hand-placed qcow2 outside libvirt's management often hits "permission denied" or sVirt denials because libvirt didn't create/label it, and it isn't tracked for capacity, cloning, or cleanup. Going through the pool keeps storage consistent, secure, and manageable.
</details>
