---
title: "Lesson 31 — User-Mode Networking (SLIRP)"
nav_order: 31
parent: "Phase 8: Networking for VMs"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 31: User-Mode Networking (SLIRP)

## Concept

The simplest way to give a VM internet access requires **zero configuration and zero
privileges**: QEMU's built-in **user-mode networking**, historically called **SLIRP**.
QEMU pretends to be a tiny router with NAT, DHCP, and DNS, all implemented in userspace
inside the QEMU process.

```
   ┌───────── QEMU process ──────────┐
   guest eth0 ──► [ SLIRP: NAT + DHCP + DNS, all in userspace ] ──► host sockets
   10.0.2.15        gateway 10.0.2.2                                    │
                    DNS     10.0.2.3                                    ▼
                                                              the host's normal network
```

The guest gets an address like `10.0.2.15`, talks to a virtual gateway `10.0.2.2`, and
QEMU translates the guest's outbound connections into ordinary host socket calls. No
root, no bridge, no TAP — but with real limitations.

---

## How It Works

### What SLIRP provides

`-netdev user` activates it. Inside the QEMU process, SLIRP emulates:

- A **virtual NAT router** at `10.0.2.2` — the guest's default gateway.
- A built-in **DHCP server** that hands the guest `10.0.2.15` (first lease) and
  configures gateway/DNS automatically.
- A built-in **DNS forwarder** at `10.0.2.3`.
- The default network is **`10.0.2.0/24`**.

When the guest opens a TCP/UDP connection, SLIRP intercepts it and makes the equivalent
connection from the host using normal sockets — so it works with just the host's
unprivileged network access. That's why it's the default and needs no setup.

### Port forwarding with hostfwd

By default nothing on the outside can reach the guest — it's behind userspace NAT with
no inbound mapping. To expose a guest service you add **`hostfwd`**:

```
   -netdev user,id=n0,hostfwd=tcp::2222-:22
   #                          │     │    └ guest port 22
   #                          │     └ (guest addr, default the guest)
   #                          └ host listen port 2222
```

Now `ssh -p 2222 user@localhost` on the host reaches the guest's SSH. You can add
multiple `hostfwd` rules.

### Why SLIRP is limited

- **Slow:** the entire datapath runs in QEMU userspace, reimplementing a TCP/IP stack —
  no vhost acceleration, lots of copying. Fine for management/light use, poor for
  throughput.
- **No inbound by default:** like any NAT, external hosts can't initiate connections to
  the guest. The guest is invisible on the LAN; only `hostfwd`-mapped ports are
  reachable, and only via the host.
- **Some protocols struggle:** ICMP/ping support is limited/emulated; protocols that
  embed addresses can misbehave.

### When to use it

Perfect for: quick tests, a guest that only needs *outbound* internet (downloading
packages), labs where you don't want to touch host networking, or environments without
root. For anything needing real LAN presence or performance, use TAP + bridge
(Lesson 32).

{: .note }
> **Why the LAN can't reach a SLIRP guest without hostfwd**
> SLIRP is NAT implemented inside the QEMU process: the guest has a private 10.0.2.0/24
> address that exists nowhere on the real network, and QEMU only translates
> guest-*initiated* outbound connections into host sockets. There's no listening socket
> or route on the host that maps inbound LAN traffic to the guest — so another machine
> has nothing to connect to. <code>hostfwd</code> fixes specific ports by making QEMU
> open a host listening socket that forwards into the guest, but the guest still has no
> general LAN presence.

---

## Lab

```bash
# 1. Zero-config outbound internet (no root needed):
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -netdev user,id=n0 -device virtio-net-pci,netdev=n0 \
    -drive file=disk.qcow2,if=virtio -nographic &

# 2. Inside the guest, see the SLIRP-assigned config and reach the internet:
#   (guest)$ ip addr show eth0
#   inet 10.0.2.15/24 ...
#   (guest)$ ip route
#   default via 10.0.2.2 dev eth0
#   (guest)$ cat /etc/resolv.conf
#   nameserver 10.0.2.3
#   (guest)$ curl -s https://example.com | head -1   # outbound works

# 3. Forward host :2222 -> guest :22 for SSH:
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -netdev user,id=n0,hostfwd=tcp::2222-:22 \
    -device virtio-net-pci,netdev=n0 \
    -drive file=disk.qcow2,if=virtio -nographic &

# 4. From the HOST, SSH into the guest via the forwarded port:
$ ssh -p 2222 user@localhost
# (works); but from ANOTHER machine on the LAN, ssh to the guest fails —
# the guest has no LAN presence, only this host-local forward.

# 5. Add multiple forwards (e.g. also expose a web server):
$ qemu-system-x86_64 -accel kvm -m 2G \
    -netdev user,id=n0,hostfwd=tcp::2222-:22,hostfwd=tcp::8080-:80 \
    -device virtio-net-pci,netdev=n0 \
    -drive file=disk.qcow2,if=virtio -nographic &
$ kill %1 %2 %3 2>/dev/null
```

**Expected result:** With `-netdev user` the guest auto-gets `10.0.2.15`, gateway
`10.0.2.2`, DNS `10.0.2.3`, and outbound internet — no host setup. `hostfwd` makes
`ssh -p 2222 localhost` reach the guest from the host, but other LAN machines cannot
reach the guest at all.

---

## Further Reading

| Topic | Link |
|---|---|
| QEMU networking (user/SLIRP) | [qemu.org — Network emulation](https://www.qemu.org/docs/master/system/devices/net.html) |
| libslirp | [gitlab.freedesktop.org — libslirp](https://gitlab.freedesktop.org/slirp/libslirp) |
| Network address translation | [Wikipedia — Network address translation](https://en.wikipedia.org/wiki/Network_address_translation) |
| Port forwarding | [Wikipedia — Port forwarding](https://en.wikipedia.org/wiki/Port_forwarding) |
| Networking Lesson 22 (NAT) | [NAT with nftables]({{ '/networking/lessons/lesson-22-nat.html' | relative_url }}) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why can't another machine on your LAN initiate a connection to a SLIRP guest without `hostfwd`?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
SLIRP is NAT implemented inside the QEMU process. The guest has a private 10.0.2.0/24 address that doesn't exist anywhere on the real network, and QEMU only translates connections the guest *initiates* outbound into host sockets. There's no route or listening socket on the host that maps inbound LAN traffic to the guest, so another machine has no address/port to connect to. hostfwd works around this for specific ports by having QEMU open a host listening socket that forwards into the guest — but the guest still has no general LAN presence.
</details>

---

**Q2. What network services does SLIRP provide to the guest automatically, and at which addresses?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
SLIRP provides, all inside the QEMU process: a virtual NAT gateway/router at 10.0.2.2, a DHCP server that assigns the guest 10.0.2.15 (and configures gateway/DNS), and a DNS forwarder at 10.0.2.3, on the default 10.0.2.0/24 network. So a guest with -netdev user gets a working address, route, and DNS with no configuration on the host or guest.
</details>

---

**Q3. Give two reasons SLIRP is unsuitable for a high-throughput production guest.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) Performance: the entire datapath, including a reimplemented TCP/IP stack, runs in QEMU userspace with no vhost acceleration and lots of copying, so throughput is low. (2) No real LAN presence / inbound: the guest sits behind userspace NAT, so external hosts can't reach it except through explicit per-port hostfwd via the host — it can't act as a normal server on the network. (Also, some protocols like ICMP are only partially emulated.) For production you use TAP + bridge to put the guest directly on the L2 network.
</details>

---

## Homework

Boot a guest with `-netdev user,id=n0,hostfwd=tcp::2222-:22`. From the host, confirm `ssh -p 2222 localhost` reaches the guest. Then, from a *different* machine on your LAN, try to SSH to the guest (you'll need to figure out what address you'd even target). Explain why it fails and what you'd switch to (and which lesson covers it) to give the guest a real LAN address.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
From the host, ssh -p 2222 localhost works because QEMU opened a host-local listening socket forwarding into the guest. From another LAN machine it fails: the guest's 10.0.2.15 is private to SLIRP and unroutable on the LAN, and the only mapping (port 2222) is bound on the QEMU *host* — at best you could target the host's LAN IP on :2222 (which would work only if the host firewall allows it and you bound hostfwd to the host's external address), but the guest itself has no LAN identity. To give the guest a real LAN address you switch to TAP + bridge networking (Lesson 32), which attaches the guest's NIC to a Linux bridge on the host's physical network so it gets a genuine LAN IP and is directly reachable.
</details>
