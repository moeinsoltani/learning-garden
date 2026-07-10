---
title: Lab Setup
nav_order: 2
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lab Setup — Prepare Your Environment

Every lesson on this site ends in a hands-on lab. Set your environment up **once**,
before starting any track, and the labs will just work. This page covers the machine
you need, the packages each track uses, and how to study a lesson.

---

## The machine

Labs assume a **Linux VM** (Ubuntu 22.04/24.04 or Debian 12) where you have `sudo`.
Commands use `apt`; adapt package names if you run another distro.

{: .warning }
> **Use a dedicated VM, not your daily machine.**
> The networking labs create and delete interfaces, rewrite routing tables, and
> load firewall rules. The virtualization labs load kernel modules and run nested
> guests. A throwaway VM means a mistake costs you a reboot, not your connectivity.
> If your hypervisor supports it, **take a snapshot** of the fresh VM so you can
> roll back at any time.

- **WSL2 works for early lessons** (networking Phases 1–5, all of the security
  track), but a real VM is required once labs touch kernel modules, DHCP, or
  nested virtualization.
- The **virtualization track** additionally needs hardware virtualization visible
  inside your VM (`/dev/kvm`). If your Linux VM runs under Hyper-V, VMware, or
  VirtualBox, enable *nested virtualization* in the hypervisor's VM settings.
  Lesson 07 of that track shows how to verify it.

---

## Base kit (all tracks)

```bash
$ sudo apt update
$ sudo apt install -y git curl man-db iproute2 tcpdump jq
```

---

## Per-track packages

Install the kit for the track you are starting. A few later lessons need extra
tools — those lessons say so inline (e.g. FRR in networking Lesson 19, eBPF
toolchain in Lesson 36, `step-ca` in security Lesson 12).

### 🌐 Networking

```bash
$ sudo apt install -y nftables ethtool iputils-ping dnsutils conntrack \
    iperf3 dnsmasq tshark wireguard-tools socat
# dnsmasq: answer "No" if asked to start it as a system service — labs run it manually
```

Verify: `ip -V` and `sudo tcpdump --version` both print a version.

### 🖥️ Virtualization

```bash
$ sudo apt install -y cpu-checker qemu-system-x86 qemu-utils \
    libvirt-daemon-system libvirt-clients virtinst ovmf \
    libguestfs-tools cloud-image-utils
$ sudo usermod -aG libvirt $USER   # then log out and back in
```

Verify: `kvm-ok` reports "KVM acceleration can be used". If it doesn't, enable
nested virtualization (see above) — the first lessons still work without it, and
Lesson 07 covers the fix in depth.

### 🔐 Security & Identity

```bash
$ sudo apt install -y openssl curl python3 oathtool
```

Verify: `openssl version` prints 3.x. Later lessons introduce their own tools as
needed: `step-ca` (Lesson 12), `certbot` (Lesson 13), `krb5-user` (Lesson 27),
Docker for Keycloak (Lesson 34) — install Docker with
`curl -fsSL https://get.docker.com | sh` when you reach it.

### 🧠 Operating Systems

```bash
$ sudo apt install -y build-essential gdb ltrace strace \
    linux-tools-common linux-tools-generic trace-cmd sysstat
```

Verify: `gcc --version` and `strace -V` both print a version. Phase 11 (kernel
modules) additionally needs `linux-headers-$(uname -r)` — that lesson covers it.

### 🧭 Engineering Leadership & 💬 English for Work

Nothing to install — labs in these two tracks are written exercises (scenarios
and rewrite drills), not terminal commands. All you need is somewhere to write
your answers before revealing the model ones: a notebook, a text file, or a
document per phase.

---

## How to study a lesson

Each lesson has the same shape. Work it in this order:

1. **Concept** — read the mental model first; don't skip to the commands.
2. **Lab** — type every command yourself (lines starting with `$`; lines starting
   with `#` are comments). Compare what you see against the expected output shown.
   If they differ, that's a learning opportunity, not a failure — investigate.
3. **Checkpoint** — write your answer in your head (or on paper) **before**
   clicking "Show Answer". The struggle is where the learning happens.
4. **Homework** — a harder task, also with a hidden model answer. Attempt it
   honestly before peeking.
5. Follow the **Next lesson** link at the bottom. Lessons build strictly on the
   ones before them, so go in order within a track.

The tracks are independent — you can do one at a time or interleave them.
Where they overlap (TLS, VPN crypto, VM networking, leadership ↔ English) the
lessons cross-link.

{: .note }
> **Cleaning up after labs**
> Most networking labs build state (namespaces, interfaces, nft tables) that
> survives until reboot. Each lab tells you how to tear down what it created;
> when in doubt, a reboot of the lab VM returns you to a clean slate — that's
> what it's for.
