---
title: "Lesson 30 — vhost-user, vsock, and Shared-Memory Datapaths"
nav_order: 30
parent: "Phase 7: VirtIO & Paravirtualization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 30: vhost-user, vsock, and Shared-Memory Datapaths

## Concept

Lesson 29 moved the datapath from QEMU userspace *into the kernel* (vhost). But what
if the fastest packet processor isn't the kernel either — it's a *specialized
userspace program* like DPDK or SPDK that polls NICs in tight loops, bypassing the
kernel entirely? **vhost-user** moves the datapath to **another userspace process**.
And **virtio-vsock** gives host↔guest communication with no network at all.

```
   vhost-net        datapath in the KERNEL
        guest ─ring─► [kernel vhost thread] ─► kernel net stack

   vhost-user       datapath in ANOTHER USERSPACE PROCESS
        guest ─ring─► [DPDK/OVS-DPDK/SPDK process] ─► NIC (kernel bypass)
                      shared memory + eventfd signalling

   virtio-vsock     host ↔ guest sockets, NO networking
        guest app ◄══ AF_VSOCK socket ══► host app
```

The throughline: the virtqueue + eventfd signalling mechanism is reusable — you can
point the datapath at the kernel, at a kernel-bypass userspace dataplane, or use the
same plumbing for a pure host/guest message channel.

---

## How It Works

### vhost-user — datapath in a separate userspace process

In vhost-user, the device's datapath is handled by a **different userspace process**
than QEMU — typically a high-performance packet/storage engine:

- **DPDK / OVS-DPDK** — userspace networking that polls the NIC in busy loops,
  bypassing the kernel network stack for maximum packet rate.
- **SPDK** — the storage equivalent for NVMe/block.

The mechanics:

- Guest RAM is made **shared** (`memory-backend-file,share=on` on `/dev/shm` or
  hugetlbfs) so the external process can map the virtqueues directly.
- QEMU and the dataplane process exchange setup over a **vhost-user socket** (a Unix
  socket): which memory regions, which virtqueues, which eventfds.
- After setup, the dataplane process reads/writes the rings in shared memory and uses
  **eventfd/irqfd** (same as vhost) to signal the guest — QEMU is out of the datapath
  entirely, *and so is the kernel*.

**What problem does this solve that in-kernel vhost can't?** The kernel network stack
itself is overhead at very high packet rates (millions of pps). A kernel-bypass
userspace dataplane (DPDK) with poll-mode drivers and hugepage buffers can go faster
than the kernel can — but it lives in userspace, so vhost-*net* (kernel) can't host
it. vhost-user lets that specialized userspace engine own the virtio datapath. It's
the backbone of high-performance NFV/telco and fast virtual switches.

### Why shared memory + eventfd

For *any* external entity to service a virtqueue without going through QEMU, two things
are needed: **access to the ring memory** (hence shared guest memory) and a **way to be
notified / to notify** (hence eventfd doorbells and irqfd interrupts). vhost-net uses
these between guest and kernel; vhost-user uses the same primitives between guest and a
sibling userspace process. Same idea, different owner.

### virtio-vsock — host/guest comms without networking

Sometimes you just want a host program and a guest program to talk — for a guest agent,
a control channel, log shipping — **without** configuring IPs, routes, or firewalls.
**virtio-vsock** provides exactly that: a socket address family (**AF_VSOCK**) where
each VM has a **context ID (CID)** and you connect by `(CID, port)`. No NIC, no IP
stack involved; it rides a virtqueue directly.

- Reliable, like TCP, but addressed by CID instead of IP.
- Survives the guest having no/misconfigured networking.
- Used by guest agents, Firecracker's API/metrics, and Kata Containers.

### virtio-fs vs 9p (revisited)

For *filesystem* sharing (vs raw sockets), **virtio-fs** uses a shared-memory (DAX)
datapath for near-native file access, while **9p** tunnels a filesystem protocol over
virtio with more overhead (Lesson 28). virtio-fs is the modern, faster choice.

{: .note }
> **The progression in one sentence**
> Plain virtio = datapath in QEMU userspace; vhost = datapath in the kernel; vhost-user
> = datapath in a specialized sibling userspace process (kernel bypass). Each step moves
> the bytes to wherever they can flow fastest for a given workload, reusing the same
> virtqueue + eventfd machinery. virtio-vsock then reuses that machinery not for device
> emulation but as a direct host↔guest pipe.

---

## Lab

```bash
# ---- virtio-vsock: host <-> guest with no networking ----
# 1. Give the guest a vsock device with a context ID (guest-cid must be unique, >=3):
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -device vhost-vsock-pci,guest-cid=42 \
    -drive file=disk.qcow2,if=virtio -nographic &

# 2. On the HOST, listen on a vsock port (needs socat with VSOCK or a small tool):
$ socat - VSOCK-LISTEN:1234                 # waits for the guest

# 3. In the GUEST, connect to the host (CID 2 = the host) on that port:
#   (guest)$ socat - VSOCK-CONNECT:2:1234
#   type a line in the guest; it appears on the host — no IP, no NIC involved.

# 4. Confirm the vsock module/CID in the guest:
#   (guest)$ lsmod | grep vsock
#   (guest)$ cat /sys/module/vmw_vsock_virtio_transport/...  # CID 42 in use

# ---- vhost-user (conceptual setup with DPDK/OVS-DPDK) ----
# 5. Guest memory must be SHARED so the dataplane process can map the rings:
$ qemu-system-x86_64 -accel kvm -m 2G \
    -object memory-backend-file,id=mem,size=2G,mem-path=/dev/hugepages,share=on \
    -machine q35,memory-backend=mem \
    -chardev socket,id=c0,path=/var/run/vhost-user-net.sock \
    -netdev vhost-user,id=n0,chardev=c0 \
    -device virtio-net-pci,netdev=n0 \
    -drive file=disk.qcow2,if=virtio -nographic &
# The dataplane (e.g. OVS-DPDK) creates/owns /var/run/vhost-user-net.sock and
# services the virtqueues from shared memory — QEMU isn't in the datapath.
$ kill %1 %2 2>/dev/null
```

**Expected result:** Over virtio-vsock you exchange data between a host process and a
guest process addressed only by CID/port — with no IP networking configured at all. The
vhost-user setup shows the required pieces: shared guest memory + a Unix socket the
external dataplane connects to.

---

## Further Reading

| Topic | Link |
|---|---|
| vhost-user protocol | [qemu.org — vhost-user](https://www.qemu.org/docs/master/interop/vhost-user.html) |
| DPDK | [Wikipedia — Data Plane Development Kit](https://en.wikipedia.org/wiki/Data_Plane_Development_Kit) |
| virtio-vsock / AF_VSOCK | [man7.org — vsock(7)](https://man7.org/linux/man-pages/man7/vsock.7.html) |
| SPDK | [spdk.io](https://spdk.io/) |
| Kernel bypass | [Wikipedia — Kernel bypass](https://en.wikipedia.org/wiki/Kernel_bypass) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. vhost-net moves the datapath to the kernel; vhost-user moves it to a userspace process. What problem is vhost-user solving that the in-kernel version can't?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
At very high packet rates (millions of pps), the kernel network stack itself becomes the bottleneck. Specialized kernel-bypass userspace dataplanes like DPDK/OVS-DPDK use poll-mode drivers and hugepage buffers to process packets faster than the kernel stack can — but they run in userspace, so vhost-net (which keeps the datapath in the kernel) can't host them. vhost-user lets that high-performance sibling userspace process own the virtio datapath (mapping the guest's shared-memory virtqueues and signalling via eventfd), achieving throughput neither QEMU-userspace virtio nor in-kernel vhost can match. It's the basis of high-performance NFV/virtual switching.
</details>

---

**Q2. Why does vhost-user require the guest's memory to be "shared" (e.g. memory-backend-file with share=on)?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because the datapath is handled by a *separate* userspace process (DPDK/OVS-DPDK/SPDK) that must read and write the guest's virtqueues directly. For that process to map the same physical memory as the guest's rings and buffers, the guest memory must be backed by a shareable mapping (memory-backend-file on /dev/shm or hugetlbfs with share=on). Anonymous, private guest memory couldn't be mapped by the external process. The shared memory plus a vhost-user Unix socket (for setup) and eventfds (for signalling) are what let the sibling process service the device without QEMU in the datapath.
</details>

---

**Q3. What is virtio-vsock, and when would you use it instead of giving the guest a normal network interface?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
virtio-vsock is a host↔guest communication channel using the AF_VSOCK socket family, where each VM has a context ID (CID) and you connect by (CID, port) over a virtqueue — no NIC, IP, routing, or firewall involved. You use it when you want reliable host/guest messaging independent of guest networking: guest agents, control/metrics channels, log shipping, or sandboxes where the guest has no (or deliberately no) network. It works even if the guest's networking is unconfigured or absent, which is why Firecracker and Kata Containers use it for their control planes.
</details>

---

## Homework

Set up a virtio-vsock device on a guest with `guest-cid=42`. Use `socat VSOCK-LISTEN:9000` on the host and `socat VSOCK-CONNECT:2:9000` in the guest (CID 2 = host) to exchange a message both directions. Then explain why a guest agent (like the QEMU guest agent) might prefer vsock over a TCP socket on a virtual NIC for its control channel.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You'll see text typed on one side appear on the other, with no IP addressing — only CID/port. A guest agent prefers vsock for its control channel because it works regardless of the guest's network configuration: there's no dependency on the guest having a correctly configured NIC, IP, routes, DNS, or firewall rules (which the agent may need to fix in the first place), and it can't be accidentally exposed to or reached from the external network. It's a private, always-available host↔guest pipe addressed by CID, so management/control traffic stays reliable and isolated even when guest networking is broken or intentionally absent.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 31 — User-Mode Networking (SLIRP) →](lesson-31-slirp){: .btn .btn-primary }
