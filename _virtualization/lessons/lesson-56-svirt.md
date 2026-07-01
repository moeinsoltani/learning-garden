---
title: "Lesson 56 — sVirt: Confining QEMU with MAC"
nav_order: 56
parent: "Phase 14: Nested Virt & Security"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 56: sVirt — Confining QEMU with MAC

## Concept

A QEMU process is a large, complex userspace program that touches guest data, host files,
and devices. If an attacker escapes a guest via a QEMU bug, what stops them from reading
*other* VMs' disks? **sVirt** does — by applying **Mandatory Access Control** (SELinux or
AppArmor) so each VM's QEMU can only touch *its own* resources, even though all VMs run as
the same user.

```
   WITHOUT sVirt                        WITH sVirt
   ─────────────                        ──────────
   all QEMU procs run as 'qemu' user     each VM gets a UNIQUE MAC label
   → a compromised VM-A QEMU can read     VM-A QEMU: label c123,c456
     VM-B's disk (same uid perms)        VM-B QEMU: label c789,c012
                                         VM-A's disk labeled c123,c456 too
                                         → VM-A QEMU CANNOT open VM-B's disk:
                                           the kernel MAC check denies it
```

Discretionary permissions (uid/gid) aren't enough — same-user processes can access each
other's files. MAC adds a second, kernel-enforced layer keyed on *labels*, not just user.

---

## How It Works

### The threat sVirt addresses

VMs commonly run under one shared service account (e.g. `qemu`/`libvirt-qemu`). Under
ordinary discretionary access control (DAC), two processes with the same uid can access the
same files — so a compromised QEMU for VM-A could open VM-B's disk image. sVirt closes this
by adding **Mandatory Access Control**: the kernel enforces, independent of uid, that a
process labeled X may only access resources labeled X.

### Per-domain dynamic labels

libvirt, with sVirt, assigns **each running domain a unique MAC label**:

- **SELinux:** the QEMU process runs in the `svirt_t` type, with a unique **MCS category
  pair** (e.g. `s0:c123,c456`) per VM. libvirt **labels that VM's disks, sockets, and other
  resources with the same category pair**.
- **AppArmor:** libvirt generates a per-VM profile (e.g. under
  `/etc/apparmor.d/libvirt/`) restricting that QEMU to its own files.

Because VM-A's process and VM-A's disk share a unique label that VM-B's process lacks, the
kernel **denies** VM-A's QEMU any attempt to open VM-B's disk — even as the same user. The
labels are **dynamic**: assigned at start, restored at stop, so they don't collide and
don't linger.

### How this stops cross-VM access

When QEMU-A (label `c123,c456`) tries to `open()` VM-B's disk (label `c789,c012`), the
SELinux/AppArmor check in the kernel sees the labels don't match and returns permission
denied — regardless of uid/gid. So a guest escape into QEMU-A is contained to VM-A's own
resources; it cannot pivot to other VMs' disks or sockets. This is *defense in depth* on top
of DAC and the sandboxing in Lesson 57.

### Debugging denials

sVirt sometimes denies *legitimate* access (e.g. a manually-placed disk image outside a
libvirt pool that didn't get labeled — recall Lesson 41). Tools:

- **SELinux:** denials appear in the audit log; `ausearch -m avc -ts recent`, and
  `audit2why` / `audit2allow` explain/translate them. `restorecon` / correct `chcon` (or
  letting libvirt manage the file via a pool) fixes labels. The boolean
  `virt_use_nfs`/`virt_sandbox_use_*` may be needed for special backends.
- **AppArmor:** denials appear in `dmesg`/`/var/log/syslog` as `apparmor="DENIED"`;
  inspect the per-VM profile. `aa-status` shows loaded profiles.

The usual root cause is a resource libvirt didn't create/label; routing storage through
pools (Lesson 41) makes labeling automatic.

{: .note }
> **How sVirt stops a compromised QEMU for VM-A from reading VM-B's disk**
> Even though both QEMU processes run as the same user (so DAC permits same-uid access),
> sVirt gives each VM a <em>unique</em> MAC label and labels that VM's disk/sockets to match.
> The kernel's mandatory access check then only allows a process to touch resources carrying
> its own label. VM-A's QEMU carries a different label than VM-B's disk, so any attempt by a
> compromised VM-A QEMU to open VM-B's disk is denied at the kernel level — uid is irrelevant.
> That label-based isolation is exactly what plain file permissions cannot provide between
> same-user processes.

---

## Lab

```bash
# 1. Confirm a MAC system is enforcing (SELinux OR AppArmor):
$ getenforce 2>/dev/null            # SELinux: Enforcing
Enforcing
$ aa-status 2>/dev/null | head -3   # AppArmor: profiles loaded (Debian/Ubuntu)

# ===== SELinux path =====
# 2. See the UNIQUE label libvirt gave a running VM's QEMU process:
$ ps -eZ | grep qemu
system_u:system_r:svirt_t:s0:c123,c456  ... qemu-system-x86_64 ... guest=vmA
system_u:system_r:svirt_t:s0:c789,c012  ... qemu-system-x86_64 ... guest=vmB
#   ^ note the DIFFERENT category pairs (c123,c456 vs c789,c012)

# 3. See that the VM's DISK carries the matching label:
$ ls -Z /var/lib/libvirt/images/vmA.qcow2
system_u:object_r:svirt_image_t:s0:c123,c456 vmA.qcow2   ← same cats as vmA's QEMU
$ ls -Z /var/lib/libvirt/images/vmB.qcow2
system_u:object_r:svirt_image_t:s0:c789,c012 vmB.qcow2   ← matches vmB, NOT vmA

# 4. Prove cross-VM denial: try to make vmA's QEMU read vmB's disk → kernel denies it.
#    (Manually, you can simulate by checking a denial appears if mislabeled:)
$ sudo ausearch -m avc -ts recent | grep svirt_t | tail
type=AVC ... denied { read } ... scontext=...:c123,c456 tcontext=...:c789,c012

# 5. Debug a denial (e.g. a hand-placed image libvirt didn't label):
$ sudo ausearch -m avc -ts recent | audit2why | head
$ sudo restorecon -v /var/lib/libvirt/images/manual.qcow2   # fix labels
#   (or place the image in a libvirt POOL so it's labeled automatically — Lesson 41)

# ===== AppArmor path (Debian/Ubuntu) =====
$ ls /etc/apparmor.d/libvirt/         # per-VM generated profiles
$ sudo dmesg | grep -i 'apparmor=\"DENIED\"' | tail   # see denials
```

**Expected result:** Each VM's QEMU runs under `svirt_t` with a *unique* MCS category pair,
and its disk image carries the *same* pair — so VM-A's label differs from VM-B's disk's
label. A cross-VM access attempt is denied by the kernel (visible as an AVC). Denials of
legitimate access trace back to unlabeled, hand-placed resources.

---

## Further Reading

| Topic | Link |
|---|---|
| sVirt | [libvirt.org — sVirt / security](https://libvirt.org/drvqemu.html#security-considerations) |
| SELinux | [Wikipedia — Security-Enhanced Linux](https://en.wikipedia.org/wiki/Security-Enhanced_Linux) |
| AppArmor | [Wikipedia — AppArmor](https://en.wikipedia.org/wiki/AppArmor) |
| Mandatory access control | [Wikipedia — Mandatory access control](https://en.wikipedia.org/wiki/Mandatory_access_control) |
| MCS (Multi-Category Security) | [SELinux — MCS](https://selinuxproject.org/page/NB_MLS) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. How does sVirt stop a compromised QEMU process for VM A from reading VM B's disk image, even though both run as the same user?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
sVirt adds Mandatory Access Control on top of normal uid/gid permissions. libvirt assigns each running VM a unique MAC label (SELinux svirt_t with a per-VM MCS category pair, or a per-VM AppArmor profile) and labels that VM's disk/sockets to match. The kernel then enforces that a process may only access resources carrying its own label, regardless of user. VM-A's QEMU carries a different label than VM-B's disk, so when a compromised VM-A QEMU tries to open VM-B's disk, the kernel's MAC check denies it. Plain DAC would have allowed it (same uid), but the label mismatch blocks it.
</details>

---

**Q2. Why isn't running all VMs under the same user account, with normal file permissions, sufficient isolation?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because discretionary access control (DAC) keys on uid/gid: two processes with the same uid can access the same files. If all VMs' QEMU processes run as the one shared 'qemu' user, then under DAC a compromised QEMU for one VM has the same access rights as every other VM's QEMU — it can open other VMs' disk images. There's no per-VM boundary at the file-permission level. sVirt's MAC adds that boundary by isolating per-VM resources with unique labels the kernel enforces independent of uid.
</details>

---

**Q3. A VM fails to start with a permission/AVC denial on a disk image you placed by hand. What's the likely cause and fix?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The likely cause is that the hand-placed image wasn't labeled by libvirt/sVirt, so its SELinux/AppArmor label doesn't match the VM's QEMU label and the kernel denies access (an AVC denial for svirt_t, or AppArmor DENIED in dmesg). The fix is to give the file the correct label — e.g. <code>restorecon</code>/<code>chcon</code> to the svirt_image_t type (SELinux) or adjust the AppArmor profile — or, better, place the image inside a libvirt storage pool (Lesson 41) so libvirt labels it automatically on VM start and restores it on stop. Use ausearch/audit2why (SELinux) or dmesg (AppArmor) to confirm the denial.
</details>

---

## Homework

On a host with two running VMs, run `ps -eZ | grep qemu` (SELinux) to see each QEMU's label, and `ls -Z` on each VM's disk. Confirm that each VM's process label matches *its own* disk's label and differs from the other VM's. Then explain, step by step, what happens at the kernel level if VM-A's QEMU were tricked into opening VM-B's disk path. (On AppArmor systems, inspect `/etc/apparmor.d/libvirt/` and explain the equivalent.)

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
ps -eZ shows each QEMU under svirt_t with a unique MCS category pair (e.g. vmA: c123,c456; vmB: c789,c012), and ls -Z shows each disk labeled svirt_image_t with the *same* pair as its own VM — so vmA's process and vmA's disk share c123,c456, distinct from vmB's c789,c012. If vmA's QEMU tried to open vmB's disk: the open() syscall triggers the kernel's SELinux check, which compares the process's context (c123,c456) against the file's context (c789,c012); because the MCS categories don't match, the access vector cache denies the operation and open() returns EACCES — even though both run as the same uid (DAC would have allowed it). An AVC denial is logged. On AppArmor, /etc/apparmor.d/libvirt/ holds a per-VM generated profile that only permits that QEMU to access its own files; an attempt to open another VM's disk isn't in the profile, so AppArmor denies it (logged as apparmor="DENIED" in dmesg). Either way, label/profile-based MAC contains the breach to VM-A's own resources.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 57 — QEMU Hardening and Sandboxing →](lesson-57-qemu-hardening){: .btn .btn-primary }
