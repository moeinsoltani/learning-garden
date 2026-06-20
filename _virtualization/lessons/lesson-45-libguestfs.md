---
title: "Lesson 45 — libguestfs: Editing Images Without Booting Them"
nav_order: 45
parent: "Phase 11: Guest Images & Provisioning"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 45: libguestfs — Editing Images Without Booting Them

## Concept

To change something inside a VM disk — install a package, set a password, drop in a file
— the obvious way is to boot the VM and log in. **libguestfs** lets you do it **offline,
without booting the guest at all.** It mounts and edits the disk image directly from the
host.

```
   WITHOUT libguestfs                   WITH libguestfs
   ─────────────────                    ───────────────
   boot VM → log in → make change        host runs virt-customize on disk.qcow2
   → shut down                           → image edited, never booted
   (slow, needs network/creds,           (fast, scriptable, no boot, no creds)
    risky on a broken guest)
```

It's how you prepare images in build pipelines, fix unbootable guests, and turn a VM into
a clean template — all by manipulating the disk file as data.

---

## How It Works

### The appliance model (why it's safe)

libguestfs doesn't loop-mount the image into your host kernel (which is dangerous —
untrusted/corrupt filesystems can crash or exploit the host kernel). Instead it boots a
tiny, isolated **appliance** (a minimal Linux + qemu) that mounts the image *inside* that
sandbox and exposes an API to the host. So even a hostile or broken guest filesystem is
handled in isolation, not by your host kernel.

### The tools

| Tool | Purpose |
|---|---|
| **`guestfish`** | Interactive shell into an image: `ls`, `cat`, `write`, `mount`, etc. The low-level Swiss army knife. |
| **`virt-customize`** | Apply changes to an existing image: install packages, set root password, run commands, copy files, set timezone — non-interactively. |
| **`virt-sysprep`** | "System prep": **de-personalize** an image for templating (remove identity/state). |
| **`virt-cat` / `virt-edit`** | Read / edit a single file inside an image. |
| **`virt-df`** | Disk-usage of filesystems inside an image (without mounting on host). |
| **`virt-resize`** | Resize/expand partitions when growing an image. |
| **`virt-inspector`** | Detect what OS/filesystems an image contains. |

### virt-customize — change an image in place

```
   virt-customize -a disk.qcow2 \
     --install nginx,htop \
     --root-password password:secret \
     --run-command 'systemctl enable nginx' \
     --copy-in ./app.conf:/etc/app/
```

No boot, no SSH — libguestfs mounts the image in its appliance and applies each change.
Perfect for CI/build pipelines and for fixing a guest you can't boot.

### virt-sysprep — prepare a template

When you turn a configured VM disk into a **template** to clone from, you must remove
**machine-specific state** so every clone isn't an identical twin causing collisions.
`virt-sysprep` strips things like:

- **SSH host keys** (else all clones share keys — a security problem).
- **`/etc/machine-id`** (else clones have the same machine-id; breaks DHCP/journald/
  systemd assumptions).
- **Persistent network interface names / udev rules** tied to old MACs.
- **Logs, shell history, cron state, cloud-init state**, temporary files, leftover SSH
  authorized_keys, etc.

The result is a clean "golden" image ready to clone (Lesson 46), where each clone
regenerates its own identity on first boot.

{: .note }
> **Why run virt-sysprep before templating**
> A template is copied to make many VMs. If you don't sysprep, every clone inherits the
> source's machine-specific state: identical SSH <em>host</em> keys (clients can't tell
> hosts apart; an attacker who gets one key compromises all), the same /etc/machine-id
> (DHCP lease collisions, duplicate journald/systemd identities), stale network rules
> bound to the old MAC, and leftover logs/history/credentials. virt-sysprep removes this
> state so each clone is a clean instance that generates fresh identity on first boot.

---

## Lab

```bash
# (Install: sudo apt install libguestfs-tools)

# 1. Inspect what's inside an image WITHOUT booting it:
$ virt-inspector -a web01.qcow2 | head -30      # OS, version, mountpoints
$ virt-df -a web01.qcow2 -h                      # filesystem usage inside
$ virt-cat -a web01.qcow2 /etc/os-release        # read a file from the image

# 2. Interactive exploration with guestfish:
$ guestfish -a web01.qcow2 -i
><fs> ll /etc | head
><fs> cat /etc/hostname
><fs> exit

# 3. Customize an image offline: install a package + set root password + run a cmd:
$ virt-customize -a web01.qcow2 \
    --install htop,curl \
    --root-password password:lab123 \
    --run-command 'systemctl enable systemd-timesyncd' \
    --copy-in ./motd:/etc/
[   0.0] Examining the guest ...
[  12.3] Installing packages: htop curl
[  18.7] Setting a random seed
[  19.0] Finishing off

# 4. Edit a single config file in place:
$ virt-edit -a web01.qcow2 /etc/ssh/sshd_config

# 5. Turn the configured image into a clean TEMPLATE (de-personalize):
$ cp web01.qcow2 template.qcow2
$ virt-sysprep -a template.qcow2
[   0.0] Examining the guest ...
[   8.1] Performing "abrt-data" ...
[   8.2] Performing "machine-id" ...        ← removes /etc/machine-id
[   8.3] Performing "ssh-hostkeys" ...      ← removes SSH host keys
[   8.4] Performing "logfiles" ...          ← clears logs
... (many operations) ...
# template.qcow2 is now a clean golden image (Lesson 46 clones from it).

# 6. See exactly which operations sysprep would perform:
$ virt-sysprep --list-operations | head
```

**Expected result:** You read and modify files inside `web01.qcow2` and install packages
into it **without ever booting it**, then `virt-sysprep` strips machine-id, SSH host
keys, and logs to produce a clean template.

---

## Further Reading

| Topic | Link |
|---|---|
| libguestfs | [libguestfs.org](https://libguestfs.org/) |
| `virt-customize` | [libguestfs.org — virt-customize](https://libguestfs.org/virt-customize.1.html) |
| `virt-sysprep` | [libguestfs.org — virt-sysprep](https://libguestfs.org/virt-sysprep.1.html) |
| `guestfish` | [libguestfs.org — guestfish](https://libguestfs.org/guestfish.1.html) |
| machine-id | [man7.org — machine-id(5)](https://man7.org/linux/man-pages/man5/machine-id.5.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why run `virt-sysprep` before turning a VM disk into a template? What kinds of state does it remove?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because a template is copied to make many VMs, and any machine-specific state in it would be duplicated across all clones, causing collisions and security problems. virt-sysprep removes that state: SSH host keys (so clones don't share identity-defining keys), /etc/machine-id (so clones don't collide on DHCP/journald/systemd identity), persistent network/udev rules tied to the old MAC, and logs, shell history, cron/cloud-init state, temp files, and leftover credentials. The result is a clean golden image where each clone regenerates its own identity on first boot.
</details>

---

**Q2. How does libguestfs edit a disk image safely without loop-mounting it into the host kernel?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It uses an "appliance" model: instead of mounting the (possibly untrusted or corrupt) guest filesystem with the host kernel — which could crash or be exploited — libguestfs launches a small isolated Linux+qemu appliance, mounts the image *inside* that sandbox, and exposes an API to the host. So the guest filesystem is parsed/handled in isolation, and a malicious or broken filesystem can't compromise the host kernel.
</details>

---

**Q3. Give two things you can do with `virt-customize` and why doing them offline is advantageous.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Examples: install packages (--install nginx), set the root password (--root-password), run commands (--run-command), copy files in (--copy-in), set timezone/hostname, etc. Doing it offline is advantageous because it's fast and fully scriptable (great for CI/build pipelines), needs no booting, no network, and no login credentials, doesn't risk side effects from a running system, and works even on a guest that won't boot — letting you fix or prepare an image purely as data.
</details>

---

## Homework

Take a cloud image (or any qcow2), use `virt-customize` to install a package and set a custom MOTD file, then use `virt-cat`/`virt-inspector` to confirm the change — all without booting. Next, copy the image and run `virt-sysprep` on the copy, then diff what changed (e.g. check `/etc/machine-id` and SSH host keys exist in the original but not the sysprepped copy). Explain why those specific files matter for cloning.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
virt-customize installs the package and writes the MOTD into the image offline; virt-cat/virt-inspector confirm the file/package are present without booting. After virt-sysprep on the copy, /etc/machine-id is emptied/removed and the SSH host keys under /etc/ssh/ssh_host_* are gone, whereas the original still has them. These files matter for cloning because: a shared machine-id causes clones to collide on DHCP leases and systemd/journald identity (they think they're the same machine), and shared SSH host keys mean clients can't distinguish hosts and compromising one clone's key compromises all of them. Removing them lets each clone regenerate unique identity on first boot, so the clones behave as independent machines.
</details>
