---
title: "Lesson 14 — TAP Interfaces"
nav_order: 14
parent: "Phase 3: Layer-2 Networking"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 14: TAP Interfaces

## Concept

A **TAP** interface is a virtual interface whose "other end" is a **userspace program**, not another part of the kernel. When the kernel sends a frame to a TAP, instead of going out a wire it is handed to whatever process opened the TAP device file. When that process writes bytes back, they appear to the kernel as frames arriving on the interface.

This is how virtual machines get NICs. QEMU/KVM opens a TAP, and the guest's "Ethernet card" is really QEMU reading and writing that TAP.

```
       kernel network stack
              │
          ┌───┴───┐
          │ tap0  │   looks like a normal NIC to the kernel
          └───┬───┘
              │  frames handed to / from userspace via /dev/net/tun
        ┌─────┴──────┐
        │  QEMU/KVM   │   the VM's virtual NIC lives here
        │  (the VM)   │
        └─────────────┘
```

The companion device is **TUN**. The difference is the layer:

- **TUN** carries **L3** — raw IP packets (no Ethernet header). Used by VPNs (WireGuard, OpenVPN): userspace gets IP packets to encrypt.
- **TAP** carries **L2** — full Ethernet frames. Used by VMs and anything that needs to emulate a real NIC with its own MAC.

---

## How it works

Both TUN and TAP are provided by the same kernel driver via the special file `/dev/net/tun`. A program opens it, issues an `ioctl` to create/attach `tap0`, and from then on:

- **Kernel → userspace:** any frame routed/bridged to `tap0` is readable by the program.
- **Userspace → kernel:** anything the program writes is injected as if it arrived on `tap0`.

The interface is otherwise normal — you can give it an IP, add it to a bridge, capture on it. The only difference from a veth is *who consumes the packets*. With a veth, the kernel moves frames to the peer interface. With a TAP, a userspace process is the peer.

{: .note }
> **TAP vs veth — who's on the other end?**
> Both look like ordinary interfaces to the kernel. The difference is the far end. A **veth**'s far end is *another interface* inside the kernel — packets stay in kernel space and pop out the peer. A **TAP**'s far end is a *userspace file descriptor* — packets leave the kernel and are consumed by a program (QEMU, a VPN, a packet generator). Use a veth to wire two kernel namespaces together; use a TAP when a userspace program must see and inject raw frames.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip tuntap add dev tap0 mode tap` | Create a TAP (L2) interface |
| `ip tuntap add dev tun0 mode tun` | Create a TUN (L3) interface |
| `ip tuntap add dev tap0 mode tap user $(whoami)` | Create owned by a non-root user |
| `ip tuntap show` | List TUN/TAP interfaces |
| `ip tuntap del dev tap0 mode tap` | Delete a TAP interface |

---

## Lab

You don't need a full VM. We'll create a TAP, attach it to a bridge, and use a tiny userspace reader to prove that frames cross the kernel↔userspace boundary.

### Step 1 — Create a TAP interface

```bash
$ sudo ip tuntap add dev tap0 mode tap
$ sudo ip link set tap0 up
$ ip tuntap show
tap0: tap persist
```

`persist` means the interface stays even when no program has it open (because we created it via `ip`, not by a process). A VM-created TAP usually disappears when the VM exits.

### Step 2 — Observe it behaves like a normal interface

```bash
$ ip link show tap0
N: tap0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN ...
    link/ether 6a:7b:8c:9d:ae:bf brd ff:ff:ff:ff:ff:ff
```

It has a MAC and can hold an IP, just like any NIC. The `state DOWN` until a process attaches and the link is brought fully up — TAP carrier depends on a userspace program being connected.

### Step 3 — Attach the TAP to a bridge (the VM-host pattern)

This mirrors how a hypervisor wires a VM into the host network:

```bash
$ sudo ip link add br0 type bridge
$ sudo ip link set br0 up
$ sudo ip link set tap0 master br0      # VM's NIC plugs into the host bridge
$ bridge link show
N: tap0 master br0 state forwarding ...
```

A real VM's traffic would now enter `tap0`, cross the bridge, and reach the rest of the host network. This is the standard KVM bridged-networking layout.

### Step 4 — Prove frames reach userspace

Open the TAP from a userspace program and read raw frames. A minimal Python reader:

```bash
$ sudo python3 - <<'PY'
import fcntl, struct, os
TUNSETIFF = 0x400454ca
IFF_TAP, IFF_NO_PI = 0x0002, 0x1000
f = os.open('/dev/net/tun', os.O_RDWR)
# attach to existing tap0
fcntl.ioctl(f, TUNSETIFF, struct.pack('16sH', b'tap0', IFF_TAP | IFF_NO_PI))
print("attached to tap0, waiting for a frame...")
frame = os.read(f, 2048)
print("got a frame of", len(frame), "bytes; first 14 bytes (Ethernet header):")
print(frame[:14].hex(' '))
PY
```

In another terminal, generate a frame toward the bridge/tap (e.g. `arping` or ping a fake address on the bridge subnet). The Python program prints the raw Ethernet header bytes — proof that the kernel handed a real L2 frame to userspace. That is exactly what QEMU does, except it forwards those bytes into the guest's emulated NIC.

### Step 5 — TUN vs TAP, by inspection

```bash
$ sudo ip tuntap add dev tun0 mode tun
$ ip tuntap show
tap0: tap persist
tun0: tun persist
```

`tun0` would deliver *IP packets* (no Ethernet header) to userspace — you'd see the IP header first, not a MAC header. That's why VPNs use TUN: they only care about routing/encrypting IP, not emulating Ethernet.

### Step 6 — Clean up

```bash
$ sudo ip link del br0
$ sudo ip tuntap del dev tap0 mode tap
$ sudo ip tuntap del dev tun0 mode tun
```

---

## Further Reading

| Topic | Link |
|---|---|
| TUN/TAP | [Wikipedia — TUN/TAP](https://en.wikipedia.org/wiki/TUN/TAP) |
| Universal TUN/TAP driver | [kernel.org — tuntap.txt](https://www.kernel.org/doc/Documentation/networking/tuntap.txt) |
| QEMU networking | [Wikipedia — QEMU](https://en.wikipedia.org/wiki/QEMU) |
| `ip-tuntap` | [man7.org — ip-tuntap(8)](https://man7.org/linux/man-pages/man8/ip-tuntap.8.html) |

---

## Checkpoint

**Q1. Why would a virtual machine use a TAP interface instead of a veth pair?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because a VM's NIC is emulated by a *userspace* program (QEMU/KVM), and a TAP is precisely the interface whose far end is a userspace file descriptor. The hypervisor opens the TAP, and every frame the kernel sends to it is delivered to QEMU, which feeds it into the guest's virtual Ethernet card; frames the guest sends come back out through the TAP into the host kernel. A veth, by contrast, has both ends inside the kernel — there's no hook for a userspace process to read and inject raw frames. Since the VM lives in userspace and needs to consume L2 frames with its own MAC, TAP is the right tool; veth is for wiring two kernel-side namespaces together.
</details>

---

**Q2. What is the difference between TUN and TAP, and which would a VPN like WireGuard use?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**TUN** operates at Layer 3: userspace receives and sends raw **IP packets** with no Ethernet header. **TAP** operates at Layer 2: userspace handles full **Ethernet frames** including MAC addresses. A VPN like WireGuard (or OpenVPN in tun mode) uses **TUN**, because it only needs the IP packets to encrypt and tunnel — it doesn't care about Ethernet framing or MAC addresses. TAP is used when you must emulate a real NIC (VMs) or bridge at L2. Rule of thumb: routing/encrypting IP → TUN; emulating an Ethernet device → TAP.
</details>

---

**Q3. Both a TAP and a veth look like ordinary interfaces to the kernel (they have a MAC, can hold an IP, can join a bridge). What is the one essential thing that distinguishes them?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
*Who consumes the packets on the far end.* A veth's far end is another interface inside the kernel — frames sent to one end emerge from the peer end, all within kernel space. A TAP's far end is a userspace program holding a file descriptor on `/dev/net/tun` — frames sent to the TAP leave the kernel and are read by that program, and anything the program writes is injected back as if it arrived on the interface. So veth = kernel-to-kernel cable; TAP = kernel-to-userspace bridge. Everything else about how they present to the stack is essentially the same.
</details>

---

## Homework

Create a TAP interface, give it an IP, and attach it to a bridge that also has a veth leading into a namespace. Without running any VM, write a short userspace program (or use a tool like `socat`/the Python snippet above) that reads from the TAP. From the namespace, send an ARP or ping toward the TAP's subnet and confirm your userspace program receives the raw frame. Explain each hop the frame took from the namespace to your program.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The frame's journey, hop by hop:

1. **Namespace generates the frame.** A ping/arping in the namespace produces an Ethernet frame on the namespace's veth end (e.g. an ARP broadcast "who has \<tap-ip\>").
2. **Across the veth into the host.** The frame exits the namespace veth and arrives on the host-side veth peer, which is a port on the bridge.
3. **Bridge floods/forwards.** The bridge receives the frame; since the destination is broadcast (ARP) or a MAC it has learned on the TAP port, it sends the frame out the `tap0` port.
4. **Kernel hands it to the TAP.** Delivering a frame to `tap0` means making it readable on the `/dev/net/tun` file descriptor that your userspace program holds.
5. **Userspace reads it.** Your program's `os.read()` returns the raw bytes — starting with the 14-byte Ethernet header (dst MAC, src MAC, EtherType), then the ARP/IP payload.

This is exactly the path a VM's inbound traffic takes, except step 5 ends in QEMU, which forwards the bytes into the guest's emulated NIC instead of printing them. The lesson: the TAP is the boundary where kernel-side L2 networking meets a userspace consumer.
</details>
