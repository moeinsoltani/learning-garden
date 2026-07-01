---
title: "Lesson 64 — Kernel Bypass (AF_XDP & DPDK)"
nav_order: 64
parent: "Phase 18: High-Performance Networking"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 64: Kernel Bypass — AF_XDP & DPDK

## Concept

Even with offloads (Lesson 63), the kernel network stack does a lot per packet: allocate an `skb`,
walk protocol layers, copy to userspace. For the most demanding workloads (millions of packets/sec
on a single box — routers, firewalls, packet generators), that overhead is the bottleneck.
**Kernel bypass** delivers packets to userspace *without* the normal stack — sometimes without the
kernel touching them at all.

```
   Normal path                   AF_XDP                         DPDK
   NIC → driver → skb → stack    NIC → XDP → AF_XDP socket      NIC → poll-mode driver
       → socket → app            → app (zero-copy)              → app (kernel not in datapath)
```

---

## How it works

**Why bypass.** The kernel's generality costs CPU cycles: per-packet `skb` allocation, generic
protocol processing you may not need, and copies. If your app *is* the network function, you'd rather
get raw frames straight from the NIC and do exactly what you need.

**AF_XDP (in-kernel fast path).** An [AF_XDP](https://www.kernel.org/doc/html/latest/networking/af_xdp.html)
socket pairs with an XDP program (Lesson 38). The XDP hook runs in the driver, before any `skb`, and
**redirects** chosen packets into a userspace-mapped ring buffer — **zero-copy** when the driver
supports it. The kernel still owns the NIC (so other traffic flows normally), but selected flows
reach userspace with minimal overhead. It's the "kernel-cooperative" bypass: fast *and* you keep the
kernel's NIC management, BPF safety, and the ability to pass non-bypassed traffic up the normal stack.

**DPDK (full bypass).** [DPDK](https://en.wikipedia.org/wiki/Data_Plane_Development_Kit) takes the
NIC *away* from the kernel entirely: a userspace **poll-mode driver** binds the device, and the app
busy-polls for packets (no interrupts, no kernel in the datapath). Maximum performance, but the NIC
is dedicated to DPDK, it burns a core spinning, and you must reimplement any networking you need
(ARP, IP, TCP) in userspace. Uses **hugepages** (Lesson from virtualization track) for big,
TLB-friendly packet buffers.

{: .note }
> **The trade-off in one line**
> Bypass trades the kernel's *generality and safety* for *raw speed*. AF_XDP keeps most of the kernel
> (cooperative, safer, selective); DPDK discards it (dedicated NIC, busy-poll, you rebuild the stack).
> Reach for bypass only when the kernel stack is provably your bottleneck — for the vast majority of
> services, offloads + multiqueue (Lesson 63) are enough.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip link set <if> xdp obj prog.o sec xdp` | Attach an XDP program (the AF_XDP redirect step) |
| `bpftool net show` | See attached XDP programs |
| `dpdk-devbind.py --status` | Show which NICs are bound to kernel vs DPDK drivers |
| `dpdk-testpmd` | DPDK's packet-forwarding test app |
| `xdpdump` / `xdp-loader` (xdp-tools) | Inspect/load XDP for AF_XDP setups |

---

## Lab

Use the XDP toolchain you met in Lesson 38 to see the redirect path that AF_XDP relies on. (A full
zero-copy AF_XDP app needs C/libbpf; here we confirm the mechanism.)

### Step 1 — Confirm XDP is available on a veth (generic XDP works anywhere)

```bash
$ sudo ip netns add x
$ sudo ip netns exec x ip link add v0 type veth peer name v1
$ sudo ip netns exec x ip link set v0 up
```

### Step 2 — Attach a minimal XDP program and observe it

```bash
# Reuse the XDP_PASS/counter program from Lesson 38:
$ sudo ip netns exec x ip link set v0 xdp obj xdp_pass.o sec xdp
$ sudo ip netns exec x bpftool net show
xdp:
v0(2) generic id 42
```

### Step 3 — Inspect a real NIC's driver binding (host, not namespace)

```bash
$ dpdk-devbind.py --status 2>/dev/null | head
# Network devices using kernel driver:    ← normal
# Network devices using DPDK-compatible driver:  ← bound to DPDK (kernel can't see them)
```

### Step 4 — Detach

```bash
$ sudo ip netns exec x ip link set v0 xdp off
$ sudo ip netns delete x
```

---

## Further Reading

| Topic | Link |
|---|---|
| AF_XDP | [kernel.org — AF_XDP](https://www.kernel.org/doc/html/latest/networking/af_xdp.html) |
| DPDK | [Wikipedia — Data Plane Development Kit](https://en.wikipedia.org/wiki/Data_Plane_Development_Kit) |
| XDP | [Lesson 38 — XDP, Express Data Path](lesson-38-xdp) |
| eBPF | [Lesson 36 — eBPF fundamentals](lesson-36-ebpf-fundamentals) |

---

## Checkpoint

**Q1. What does kernel bypass buy you, and what do you give up?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It buys <strong>raw packet-processing speed</strong> — by skipping the kernel's per-packet overhead
(skb allocation, generic protocol processing, copies and interrupts), a single box can handle millions
more packets per second. What you give up is the kernel's <strong>generality and safety</strong>: you
lose the ready-made TCP/IP stack, sockets, firewall, and routing for the bypassed traffic and must
provide whatever you need yourself; with full bypass (DPDK) you also dedicate the NIC, burn a CPU core
busy-polling, and lose the kernel's isolation/protection on that path. So bypass is worthwhile only
when the kernel stack is the proven bottleneck for a packet-centric workload; ordinary services are
better served by offloads and multiqueue scaling.
</details>

---

**Q2. How does AF_XDP differ from DPDK in how much of the kernel it keeps?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>AF_XDP</strong> is cooperative: the kernel still owns and manages the NIC, but an XDP program
in the driver redirects <em>selected</em> packets into a userspace ring (zero-copy where supported)
before any skb is built. Non-selected traffic still flows up the normal kernel stack, and you keep BPF
verification and kernel NIC management. <strong>DPDK</strong> is full bypass: it unbinds the NIC from
the kernel entirely and drives it from a userspace poll-mode driver, so the kernel isn't in the
datapath at all — fastest, but the device is dedicated to DPDK, a core spins polling, and you must
reimplement any networking (ARP/IP/TCP) yourself. In short, AF_XDP keeps most of the kernel and is
selective; DPDK discards it and is total.
</details>

---

**Q3. Why does DPDK use a poll-mode driver and hugepages?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Poll-mode</strong>: at very high packet rates, interrupt-driven reception becomes a bottleneck
(an interrupt per packet, or even per batch, costs context switches and latency jitter). A poll-mode
driver instead has a dedicated core <em>busy-poll</em> the NIC's descriptor rings continuously, so
packets are picked up immediately with no interrupt overhead — trading a fully-consumed CPU core for
deterministic, maximal throughput. <strong>Hugepages</strong>: DPDK allocates large packet-buffer
pools, and using 2 MB/1 GB hugepages instead of 4 KB pages drastically reduces TLB misses (one TLB
entry covers far more memory) and avoids page-table walk overhead on every buffer access, plus
guarantees physically contiguous memory for DMA. Both choices remove kernel/hardware overheads that
would otherwise cap packets-per-second.
</details>

---

## Homework

Compare conceptually (and, if you have hardware, empirically) the maximum small-packet (64-byte)
forwarding rate of: (a) the normal kernel stack, (b) an XDP program doing `XDP_TX`/redirect, and
(c) DPDK testpmd. Predict the ordering and explain *where* each design spends or saves cycles per
packet. Then state which you'd actually choose for a typical web service and why.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Expected ordering, slowest to fastest: <strong>(a) kernel stack &lt; (b) XDP &lt; (c) DPDK</strong>.
(a) The normal stack pays the most per packet — interrupt handling, skb allocation/free, full protocol
traversal, and copies — so 64-byte line rate on fast NICs overwhelms it. (b) XDP runs in the driver
<em>before</em> skb allocation and can transmit/redirect a packet with a tiny BPF program, eliminating
the skb and stack-traversal cost while still being interrupt/NAPI-driven and kernel-managed — a large
jump in pps. (c) DPDK removes even the kernel and interrupts: a poll-mode driver on a dedicated core
processes packets from hugepage buffers with essentially no per-packet OS overhead, hitting the highest
pps (often true line rate for 64-byte frames). For a <strong>typical web service</strong>, though, you'd
choose <strong>none of the bypass options</strong> — you'd use the normal kernel stack with offloads
and multiqueue scaling (Lesson 63), because a web service is rarely packet-rate-bound, needs the
kernel's TCP/TLS/sockets/firewall, and benefits from the kernel's generality and safety. Bypass is for
dedicated packet-movers (routers, firewalls, load balancers, DDoS scrubbers), not application servers.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 65 — Multipath TCP (MPTCP) →](lesson-65-mptcp){: .btn .btn-primary }
