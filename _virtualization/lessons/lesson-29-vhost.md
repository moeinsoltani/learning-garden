---
title: "Lesson 29 — vhost: Moving the Datapath into the Kernel"
nav_order: 29
parent: "Phase 7: VirtIO & Paravirtualization"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 29: vhost — Moving the Datapath into the Kernel

## Concept

virtio (Lesson 27) already cut VM exits by batching through a shared ring. But there's
still a cost: the ring is serviced by **QEMU in userspace**. So every batch of packets
involves waking the QEMU process and copying data through it. **vhost** removes QEMU
from the *datapath* by letting the **kernel** service the virtqueue directly.

```
   PLAIN VIRTIO-NET                      VHOST-NET
   ────────────────                      ─────────
   guest ─ring─► QEMU (userspace) ─►      guest ─ring─► KERNEL vhost thread ─►
                 host kernel net                        host kernel net
   data crosses into userspace            datapath stays in the kernel;
   and back (exit + copy + wakeup)        QEMU only sets things up (control plane)
```

The guest still uses the *same* virtio-net driver — nothing changes inside the guest.
What changes is *who reads the ring*: a kernel worker (`vhost`) instead of the QEMU
process.

---

## How It Works

### The problem vhost solves

With plain virtio-net, when the guest notifies that packets are ready, control returns
to **QEMU** (userspace), which reads the virtqueue, copies the packets, and hands them
to the host kernel's network stack (the TAP device). That's a userspace round trip and
extra copies per batch — the QEMU process must be scheduled, touched, and switched
through for every burst of traffic.

### vhost-net: the kernel services the ring

**vhost-net** moves the virtqueue **datapath** into the kernel:

- A kernel thread (`vhost-<pid>`) is given direct access to the guest's virtqueue
  memory and the TAP device.
- When the guest posts packets and rings the doorbell (via an **eventfd**/irqfd), the
  kernel vhost thread — *not* QEMU — reads the ring and moves packets straight between
  the guest and the host network stack.
- The guest is interrupted on completion via **irqfd**, again without going through
  QEMU.

So the *data plane* (moving bytes) lives entirely in the kernel; **QEMU keeps only the
control plane**: it sets up the device, negotiates features, configures the vhost
kernel backend, and handles slow-path/config changes — but it's no longer in the hot
path of each packet.

### The general vhost model

The same idea generalizes:

- **vhost-net** — kernel datapath for virtio-net.
- **vhost-scsi** — kernel datapath for virtio-scsi (storage), bypassing QEMU's block
  layer for the data plane.
- The pattern: kernel worker threads own the virtqueue, signalled by eventfds, with
  QEMU relegated to setup/control.

### Why it's faster

It eliminates, per batch: the return to userspace, the QEMU process wakeup/scheduling,
and copies through QEMU. The result is higher throughput and lower latency, especially
for high packet rates — fewer context switches between kernel and the QEMU process.
You enable it with `vhost=on` on the netdev.

{: .note }
> **eventfd / irqfd — the signalling glue**
> vhost needs a way for guest↔kernel notifications to bypass QEMU. <strong>ioeventfd</strong>
> turns a guest doorbell write into a signal on an eventfd the vhost kernel thread waits
> on; <strong>irqfd</strong> lets the kernel inject the guest's completion interrupt
> directly. Because these are wired once at setup, the running datapath never needs QEMU
> to relay a notification — the kernel and guest signal each other directly. Keep this in
> mind for Lesson 30, where vhost-<em>user</em> uses the same eventfd signalling but with
> a different process owning the datapath.

---

## Lab

```bash
# 1. Enable vhost on a TAP-backed virtio-net NIC (TAP/bridge from Phase 8):
$ sudo qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -netdev tap,id=n0,ifname=tap0,script=no,downscript=no,vhost=on \
    -device virtio-net-pci,netdev=n0 \
    -drive file=disk.qcow2,if=virtio -nographic &

# 2. Confirm a vhost kernel thread now exists for this VM:
$ ps -eLf | grep -E 'vhost' | grep -v grep
root  ...  [vhost-12345]      ← kernel worker servicing the virtqueue datapath

# 3. Confirm the vhost device is in use:
$ ls -l /dev/vhost-net
crw------- 1 root root 10, 238 ... /dev/vhost-net
$ sudo lsof /dev/vhost-net 2>/dev/null | head    # QEMU holds it open

# 4. Measure the difference. Run iperf3 guest→host (recall Networking L33):
#    First WITHOUT vhost (vhost=off), then WITH (vhost=on), comparing throughput
#    and watching kvm_stat for userspace exits:
#   (guest)$ iperf3 -c <host-ip>
#   vhost=off:  ~X Gbit/s,  higher kvm_userspace_exit / more QEMU CPU
#   vhost=on:   ~higher Gbit/s, fewer userspace exits, datapath in kernel thread

# 5. See QEMU's CPU drop under load with vhost — the datapath left userspace:
$ top -H -p $(pgrep -f 'qemu-system' | head -1)   # QEMU threads' CPU during iperf
$ sudo kill %1 2>/dev/null
```

**Expected result:** With `vhost=on` a `[vhost-<pid>]` kernel thread appears and
`/dev/vhost-net` is held open by QEMU. Under an iperf3 load, throughput is higher and
QEMU's own CPU usage / userspace exits are lower than with `vhost=off`, because the
packet datapath now runs in the kernel thread instead of the QEMU process.

---

## Further Reading

| Topic | Link |
|---|---|
| vhost-net / vhost architecture | [kernel.org — vhost](https://docs.kernel.org/driver-api/vhost.html) |
| vhost-net & macvtap (Red Hat) | [access.redhat.com — vhost-net](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_virtualization/) |
| `eventfd` | [man7.org — eventfd(2)](https://man7.org/linux/man-pages/man2/eventfd.2.html) |
| virtio | [Wikipedia — Virtio](https://en.wikipedia.org/wiki/Virtio) |
| Data plane vs control plane | [Wikipedia — Data plane](https://en.wikipedia.org/wiki/Data_plane) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. With vhost-net, which part of packet processing no longer requires switching into the QEMU process, and why does that speed things up?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The data plane — actually reading the virtqueue and moving packets between the guest and the host network stack — no longer goes through QEMU. A kernel vhost worker thread services the ring directly, signalled by eventfd/irqfd. This speeds things up because each batch of packets no longer requires returning to userspace, scheduling/waking the QEMU process, and copying data through it; eliminating those context switches and copies raises throughput and lowers latency, especially at high packet rates. QEMU keeps only the control plane (setup, feature negotiation, configuration).
</details>

---

**Q2. After enabling vhost, does anything change inside the guest? What changes on the host?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Nothing changes inside the guest — it uses the exact same virtio-net driver and virtqueue interface; vhost is transparent to it. On the host, a kernel worker thread (e.g. [vhost-<pid>]) is created to service the virtqueue datapath, QEMU opens /dev/vhost-net and configures the kernel backend, and the packet datapath now flows through that kernel thread rather than the QEMU userspace process. QEMU's role shrinks to control-plane setup.
</details>

---

**Q3. What are ioeventfd and irqfd, and why are they essential to keeping QEMU out of the datapath?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
ioeventfd turns a guest's doorbell/notification (a register write) into a signal on an eventfd that the kernel vhost thread waits on, so the guest can notify the kernel directly. irqfd lets the kernel inject the guest's completion interrupt directly. Both are wired up once at device setup. They're essential because they let guest↔kernel signalling happen without QEMU relaying each notification — so during the running datapath, the kernel and guest signal each other directly and QEMU never has to be scheduled in to pass messages, which is exactly what removes it from the hot path.
</details>

---

## Homework

Run an `iperf3` test from a guest to the host twice — once with `vhost=off` and once with `vhost=on` on the virtio-net netdev. Record throughput and, with `top -H` on the QEMU process, the CPU consumed by QEMU's threads during each test. Explain the relationship between QEMU's CPU usage and where the datapath runs in each case.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With vhost=off, throughput is lower and QEMU's threads consume significant CPU during the transfer, because the packet datapath runs in QEMU userspace — every batch returns to the QEMU process, which reads the ring and copies data. With vhost=on, throughput is higher and QEMU's own CPU usage during the transfer drops sharply; instead a kernel [vhost-<pid>] thread does the work. The relationship: QEMU CPU usage tracks how much of the datapath flows through the QEMU process. Moving the data plane into the kernel offloads it from QEMU (lower QEMU CPU) and removes userspace round-trips, yielding more throughput per CPU cycle.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 30 — vhost-user, vsock, and Shared-Memory Datapaths →](lesson-30-vhost-user-vsock){: .btn .btn-primary }
