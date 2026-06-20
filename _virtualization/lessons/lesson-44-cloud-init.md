---
title: "Lesson 44 — Cloud Images and cloud-init"
nav_order: 44
parent: "Phase 11: Guest Images & Provisioning"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 44: Cloud Images and cloud-init

## Concept

A **cloud image** is a pre-installed, ready-to-boot OS disk — no installer, no manual
setup. But it ships *deliberately incomplete*: no user password, no SSH keys, generic
hostname. On first boot, **cloud-init** reads configuration from a **data source** and
finishes the setup: creates users, installs SSH keys, sets the hostname, runs commands.

```
   cloud image (generic)        +    seed (your config)     =  configured VM
   ┌────────────────────┐            ┌──────────────────┐
   │ OS installed        │            │ user-data:       │      first boot:
   │ cloud-init enabled  │  ─────►    │  - ssh keys      │  ──► cloud-init applies it
   │ NO user/pass/keys   │            │  - hostname      │      → login with your key
   │ grows on first boot │            │  - packages      │
   └────────────────────┘            └──────────────────┘
```

This is how every cloud VM you've ever launched gets *your* SSH key without anyone typing
it into an installer.

---

## How It Works

### What a cloud image is

A cloud image (e.g. `ubuntu-24.04-server-cloudimg-amd64.img`) is a qcow2 with:

- The OS **pre-installed** — boot it and it runs immediately.
- **cloud-init** baked in, running early on first boot.
- A small root filesystem that **auto-grows** to fill the disk on first boot (so you can
  resize the qcow2 and the guest expands into it).
- No credentials — useless until cloud-init injects configuration.

### cloud-init data sources

cloud-init looks for configuration from a **data source**. In real clouds it's the
provider's metadata service (an HTTP endpoint at a link-local address). For *local labs*
the easiest is the **NoCloud** datasource: cloud-init reads from a small attached disk
(a "seed" ISO or vfat image) labeled `cidata`.

### user-data and meta-data

The seed provides (at least) two files:

- **`meta-data`** — instance identity: `instance-id`, `local-hostname`.
- **`user-data`** — the interesting part, a **cloud-config** YAML (starts with
  `#cloud-config`): users, SSH keys, packages to install, `runcmd` commands, written
  files, etc.

```yaml
   #cloud-config
   hostname: web01
   users:
     - name: moein
       sudo: ALL=(ALL) NOPASSWD:ALL
       ssh_authorized_keys:
         - ssh-ed25519 AAAA... you@host
   packages: [nginx, htop]
   runcmd:
     - systemctl enable --now nginx
```

(Optionally `network-config` for static networking.)

### Building a seed ISO

You package `user-data` + `meta-data` into an ISO with volume label `cidata` (or use
`cloud-localds`):

```
   cloud-localds seed.iso user-data meta-data
   # or: genisoimage -output seed.iso -volid cidata -joliet -rock user-data meta-data
```

Attach both the cloud image and `seed.iso` to the VM; on first boot cloud-init finds the
`cidata` disk and applies it.

### First-boot vs per-boot modules

cloud-init runs in stages. Some modules run **only once** (first boot) — creating users,
generating SSH host keys, growing the filesystem, running `runcmd` — keyed off the
`instance-id`. Others run **every boot** (e.g. setting hostname, some network config). If
you reuse a disk but want first-boot logic to run again, you change the `instance-id` (or
clean cloud-init state with `cloud-init clean`), which is why templating tools reset it
(Lesson 46).

{: .note }
> **Why a fresh cloud image needs a seed**
> The cloud image intentionally has no password, no SSH keys, and a generic hostname —
> so it's not directly loginable. cloud-init exists to fill that gap on first boot, but it
> needs a *source* of configuration. In a real cloud that's the provider's metadata
> service; in a local lab there's no such service, so you supply a NoCloud seed (an ISO
> labeled <code>cidata</code> with user-data/meta-data). Without any data source,
> cloud-init has nothing to apply and you can't log in — hence the seed is required.

---

## Lab

```bash
# 1. Download a cloud image and make a working copy (don't taint the pristine one):
$ wget https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img
$ cp ubuntu-24.04-server-cloudimg-amd64.img web01.qcow2
$ qemu-img resize web01.qcow2 20G        # the FS will auto-grow into this on boot

# 2. Write the seed files. user-data injects YOUR ssh key and a hostname:
$ cat > user-data <<'EOF'
#cloud-config
hostname: web01
users:
  - name: moein
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAA...REPLACE_WITH_YOUR_PUBKEY... you@host
ssh_pwauth: false
EOF
$ cat > meta-data <<'EOF'
instance-id: web01-0001
local-hostname: web01
EOF

# 3. Build the NoCloud seed ISO (label MUST be cidata):
$ cloud-localds seed.iso user-data meta-data
# (or: genisoimage -output seed.iso -volid cidata -joliet -rock user-data meta-data)

# 4. Boot the cloud image WITH the seed attached:
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -drive file=web01.qcow2,if=virtio \
    -drive file=seed.iso,if=virtio,format=raw \
    -netdev user,id=n0,hostfwd=tcp::2222-:22 \
    -device virtio-net-pci,netdev=n0 -nographic &

# 5. cloud-init applies the config on first boot; log in with your KEY (no installer!):
$ ssh -p 2222 moein@localhost          # works, using the injected ssh key

# 6. Inside the guest, inspect what cloud-init did:
#   (guest)$ cloud-init status --long
#   status: done
#   (guest)$ hostname            # web01
#   (guest)$ sudo cloud-init query userdata | head
$ kill %1 2>/dev/null
```

**Expected result:** A generic cloud image boots and, via the NoCloud seed, configures
its hostname and your SSH key on first boot — you log in with your key and never touch an
installer. `cloud-init status` reports `done`.

---

## Further Reading

| Topic | Link |
|---|---|
| cloud-init | [cloudinit.readthedocs.io](https://cloudinit.readthedocs.io/) |
| NoCloud datasource | [cloud-init — NoCloud](https://cloudinit.readthedocs.io/en/latest/reference/datasources/nocloud.html) |
| cloud-config examples | [cloud-init — Examples](https://cloudinit.readthedocs.io/en/latest/reference/examples.html) |
| Cloud image (concept) | [Wikipedia — Cloud image / golden image](https://en.wikipedia.org/wiki/System_image) |
| `cloud-localds` (cloud-utils) | [Ubuntu — cloud-utils](https://manpages.ubuntu.com/manpages/jammy/man1/cloud-localds.1.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Where does cloud-init get its configuration from on first boot, and why does a fresh cloud image need *something* like a seed ISO?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
cloud-init reads configuration from a *data source*. In a real cloud that's the provider's metadata service (an HTTP endpoint); for local use you supply the NoCloud datasource — a small attached disk (a seed ISO/vfat image) labeled <code>cidata</code> containing user-data and meta-data. A fresh cloud image is deliberately incomplete (no password, no SSH keys, generic hostname), so it's not directly loginable; cloud-init fills that gap on first boot, but it needs a source of config. Locally there's no metadata service, so without a seed cloud-init has nothing to apply and you can't log in — hence the seed is required.
</details>

---

**Q2. What's the difference between `user-data` and `meta-data` in a NoCloud seed?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
meta-data provides instance identity — primarily instance-id and local-hostname. user-data is the cloud-config payload (YAML beginning with #cloud-config) describing what to configure: users and their SSH keys, sudo rules, packages to install, files to write, and runcmd commands to execute. meta-data identifies the instance (and its instance-id gates first-boot-only modules); user-data is the actual provisioning content you care about.
</details>

---

**Q3. Why do some cloud-init modules run only on first boot, and how is "first boot" determined?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Some actions should happen exactly once — creating users, generating SSH host keys, growing the root filesystem, running runcmd — because repeating them every boot would be wrong or wasteful. cloud-init determines "first boot" by the instance-id: if it sees a new/unseen instance-id it treats this as a fresh instance and runs the per-instance (first-boot) modules; if the instance-id matches what it ran before, it skips them. That's why reusing a disk for a new VM requires changing the instance-id (or running <code>cloud-init clean</code>) so first-boot logic runs again — which templating tools do automatically (Lesson 46).
</details>

---

## Homework

Boot a cloud image with a NoCloud seed that injects your SSH key and sets the hostname to something custom. SSH in with your key and confirm the hostname. Then shut down, copy the *same* disk to a new file, and boot it again with the same seed — does cloud-init re-run the first-boot steps? Explain what you'd change in the seed (or run in the guest) to force first-boot logic to run on the copy.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
On first boot the hostname and SSH key are applied and you log in with your key. Booting a copy of the *same disk* with the *same seed* (same instance-id) does NOT re-run first-boot modules, because cloud-init records that instance-id as already processed — it considers it the same instance, so user creation/host-key generation/runcmd are skipped. To force first-boot logic on the copy, change the <code>instance-id</code> in the seed's meta-data (e.g. web01-0002) so cloud-init treats it as a new instance, or run <code>cloud-init clean</code> (optionally with --logs) inside the guest before re-imaging to wipe the recorded state. This is exactly why templating (Lesson 46) resets cloud-init state / instance-id when producing clones.
</details>
