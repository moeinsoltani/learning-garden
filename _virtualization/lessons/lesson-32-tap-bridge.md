---
title: "Lesson 32 — TAP + Bridge Networking"
nav_order: 32
parent: "Phase 8: Networking for VMs"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 32: TAP + Bridge Networking

## Concept

To put a guest on the **real L2 network** — a genuine LAN IP, reachable by any machine
— you use the production pattern: a **TAP** device as the guest's virtual "network
cable," plugged into a **Linux bridge** that also connects the host's physical NIC.
This is exactly the TAP (Networking Lesson 14) and bridge (Networking Lesson 11) you
already know, applied to a VM.

```
   ┌──────────── guest ───────────┐
   │  eth0 (virtio-net)            │
   └──────────────┬────────────────┘
                  │  (one end inside the guest, the other on the host)
            ┌─────▼──── tap0 ──────┐         tap0 = the "cable end" on the host
            │      Linux bridge br0 │◄─────── a virtual switch
            │   tap0   eth0(host)   │
            └──────────┬─────────────┘
                       │
                  physical LAN  ──► other machines, router, DHCP
```

The bridge acts as a virtual switch: frames from the guest's TAP and frames from the
physical NIC are switched together, so the guest is just another host on the LAN.

---

## How It Works

### The two pieces

- **TAP** (Networking L14): a virtual L2 interface. One side is `tap0` on the host; the
  other side is the guest's `eth0`. Bytes the guest sends on `eth0` come out of `tap0`
  on the host, and vice-versa — it's the virtual Ethernet cable.
- **Bridge** (Networking L11): a software L2 switch (`br0`). You enslave both `tap0`
  (the guest) and the host's physical `eth0` to it. The bridge learns MACs and forwards
  frames between its ports — so the guest's frames reach the physical LAN and replies
  come back.

### The packet path

A frame from the guest:

```
   guest eth0  ──►  tap0  ──►  br0 (switch)  ──►  host eth0  ──►  physical LAN
   (virtio-net      (host       (forwards by      (uplink         (router, DHCP,
    frontend)        TAP end)    dest MAC)          port)           other hosts)
```

The guest can now DHCP from the LAN's real DHCP server, get a LAN IP, and be pinged by
any machine on the network — it's L2-adjacent to everything else.

### Wiring it up

QEMU side: a TAP netdev + a virtio-net device.

```
   -netdev tap,id=n0,ifname=tap0,script=no,downscript=no \
   -device virtio-net-pci,netdev=n0
```

Host side: create the bridge, add the uplink and the TAP.

```
   ip link add br0 type bridge
   ip link set eth0 master br0          # enslave the physical NIC
   ip link set tap0 master br0          # enslave the guest's TAP
   ip link set br0 up; ip link set tap0 up
```

(Note: enslaving the host's only NIC to a bridge moves the host's IP onto `br0` — plan
for that so you don't lose connectivity.)

### The qemu-bridge-helper

Creating TAPs and enslaving them needs privileges. The **qemu-bridge-helper** (a small
setuid helper, configured via `/etc/qemu/bridge.conf` with `allow br0`) lets an
unprivileged QEMU attach a TAP to an allowed bridge automatically:

```
   -netdev bridge,id=n0,br=br0 -device virtio-net-pci,netdev=n0
```

This is the scripted/automatic alternative to manually pre-creating `tap0`. libvirt
(Phase 10) does all of this for you.

{: .note }
> **TAP vs bridge — which is which**
> The <strong>TAP</strong> is the per-VM virtual cable: it has two ends, one in the
> guest (eth0) and one on the host (tap0). The <strong>bridge</strong> is the shared
> virtual switch that many TAPs and the physical uplink all plug into. You need both:
> the TAP connects one guest to the host; the bridge connects all the TAPs and the real
> NIC together so guests reach the LAN and each other. Confusing them is the most common
> beginner mistake.

---

## Lab

```bash
# ---- Host setup: a bridge with the uplink (do this carefully over a console!) ----
$ sudo ip link add br0 type bridge
$ sudo ip link set br0 up
# Move the uplink onto the bridge (host IP will move to br0):
$ sudo ip link set eth0 master br0
$ sudo dhclient br0          # or assign br0 a static IP

# ---- Create a TAP for the guest and enslave it ----
$ sudo ip tuntap add dev tap0 mode tap user $(whoami)
$ sudo ip link set tap0 master br0
$ sudo ip link set tap0 up

# ---- Boot the guest attached to tap0 ----
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -netdev tap,id=n0,ifname=tap0,script=no,downscript=no \
    -device virtio-net-pci,netdev=n0 \
    -drive file=disk.qcow2,if=virtio -nographic &

# ---- Verify the bridge has both ports ----
$ bridge link show
tap0  ... master br0 state forwarding
eth0  ... master br0 state forwarding

# ---- Inside the guest: it gets a REAL LAN IP from the LAN's DHCP ----
#   (guest)$ ip addr show eth0
#   inet 192.168.1.57/24 ...        ← a real LAN address, not 10.0.2.x
#   (guest)$ ping 192.168.1.1       ← the real LAN gateway responds
# And from ANOTHER LAN machine you can now ping/ssh 192.168.1.57 directly.

# ---- Automatic alternative via the bridge helper ----
$ echo 'allow br0' | sudo tee -a /etc/qemu/bridge.conf
$ qemu-system-x86_64 -accel kvm -m 2G \
    -netdev bridge,id=n0,br=br0 -device virtio-net-pci,netdev=n0 \
    -drive file=disk.qcow2,if=virtio -nographic &
$ sudo kill %1 %2 2>/dev/null
```

**Expected result:** `bridge link show` lists both `tap0` and `eth0` enslaved to
`br0`. The guest receives a genuine LAN IP from the network's DHCP server and is
reachable (ping/SSH) from other machines on the LAN — true L2 presence, unlike SLIRP.

---

## Further Reading

| Topic | Link |
|---|---|
| Networking Lesson 14 (TAP) | [TAP interfaces]({{ '/networking/lessons/lesson-14-tap.html' | relative_url }}) |
| Networking Lesson 11 (bridges) | [Linux bridges]({{ '/networking/lessons/lesson-11-bridges.html' | relative_url }}) |
| TUN/TAP | [Wikipedia — TUN/TAP](https://en.wikipedia.org/wiki/TUN/TAP) |
| QEMU networking (tap) | [qemu.org — Network emulation](https://www.qemu.org/docs/master/system/devices/net.html) |
| `bridge` command | [man7.org — bridge(8)](https://man7.org/linux/man-pages/man8/bridge.8.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Trace a packet from the guest's `eth0` to the physical LAN. Which piece is the TAP and which is the bridge?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The guest sends a frame on its eth0 (virtio-net). The other end of that virtual cable is tap0 on the host — the TAP. tap0 is a port on the Linux bridge br0 — the bridge, a software L2 switch. The bridge forwards the frame by destination MAC out its uplink port, the host's physical eth0, onto the physical LAN, where the router/other hosts receive it. Replies traverse the reverse path. So: TAP = the per-VM virtual cable (guest eth0 ↔ host tap0); bridge = the shared virtual switch joining tap0 and the physical NIC.
</details>

---

**Q2. Why does enslaving the host's physical NIC to the bridge usually move the host's IP onto the bridge interface?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Once the physical NIC (eth0) becomes a bridge port, it operates at L2 as a switch port and no longer terminates L3/IP itself — the bridge (br0) becomes the L3 interface for that segment. So the IP/default route belong on br0, not eth0. If you don't move the IP to br0 (via DHCP or static config on br0), the host loses its network connectivity. That's why you configure addressing on br0 and why this should be done from a console or with care, to avoid locking yourself out.
</details>

---

**Q3. What does the qemu-bridge-helper let you do, and what configuration authorizes it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The qemu-bridge-helper is a small setuid helper that lets an unprivileged QEMU automatically create a TAP and attach it to a bridge (via -netdev bridge,br=br0), instead of you pre-creating and enslaving the TAP manually as root. It's authorized by listing the permitted bridge in /etc/qemu/bridge.conf (e.g. <code>allow br0</code>); only bridges in that allowlist may be used, limiting what an unprivileged user can plug into.
</details>

---

## Homework

Build a bridge `br0` containing your host uplink, create `tap0`, and boot a guest on it. Confirm with `bridge link show` that both `tap0` and the uplink are enslaved, and from another LAN machine ping the guest's LAN IP. Then explain why this gives the guest something SLIRP (Lesson 31) fundamentally cannot, and name one risk of bridging the host's only NIC.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With both ports on br0, the guest gets a real LAN IP from the network's DHCP and another machine can ping/SSH it directly — proving true L2 presence. This is what SLIRP fundamentally cannot give: SLIRP is userspace NAT with a private 10.0.2.0/24 and no inbound except per-port hostfwd via the host, so the guest is never a first-class LAN host. The risk of bridging the host's only NIC is connectivity loss/lockout: enslaving eth0 moves L3 to br0, so if addressing on br0 isn't configured correctly (or you do it remotely over that same NIC) the host can drop off the network — which is why you do it from a local console and assign br0 an IP immediately.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 33 — Accelerated Networking: vhost-net and Multiqueue →](lesson-33-accelerated-net){: .btn .btn-primary }
