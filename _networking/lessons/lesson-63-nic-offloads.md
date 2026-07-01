---
title: "Lesson 63 — NIC Offloads & Multiqueue Scaling"
nav_order: 63
parent: "Phase 18: High-Performance Networking"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 63: NIC Offloads & Multiqueue Scaling

## Concept

A single CPU core can't process 40 Gbit/s of small packets — there are too many per-packet operations.
Two families of techniques rescue throughput: **offloads** (let the NIC do bulk work so the CPU
handles fewer, larger units) and **multiqueue scaling** (spread packet processing across many cores).

```
   Without offload/scaling          With offload + multiqueue
   1 core ─ packet ─ packet ─ ...   NIC groups packets (GRO) ─┐
   (saturates, drops)               core0 ─ queue0            ├─ work spread across cores
                                    core1 ─ queue1            │
                                    core2 ─ queue2 ───────────┘
```

---

## How it works

**Offloads — fewer, bigger units.** The expensive part isn't moving bytes, it's the per-packet
overhead. Offloads batch or delegate it:

| Offload | What it does |
|---|---|
| **TSO** (TCP Segmentation Offload) | Hand the NIC one big buffer; the NIC chops it into MTU-sized segments on send |
| **GSO** | Same idea done in software just before the driver (works without NIC support) |
| **GRO** (Generic Receive Offload) | On receive, merge many small packets of a flow into one big one handed up the stack |
| **Checksum offload** | NIC computes/verifies checksums instead of the CPU |

GRO is the receive-side hero: the stack traverses *once* per merged super-packet instead of once per
packet. Check/toggle with `ethtool -k`.

**Multiqueue scaling — more cores.** Modern NICs have multiple hardware queues. The goal is to fan
incoming packets across CPUs so no single core is the bottleneck:

- **RSS** (Receive Side Scaling): the *NIC* hashes each packet's flow to one of its hardware queues,
  each wired to a different CPU's interrupt. Pure hardware, fastest.
- **RPS** (Receive Packet Steering): the *software* equivalent — the kernel hashes flows to CPUs in
  software, for NICs/queues without enough RSS. 
- **XPS** (Transmit Packet Steering): picks which TX queue a core uses, keeping a flow's transmit
  work on one CPU.

**IRQ affinity & NAPI.** Each queue raises interrupts; pinning a queue's IRQ to a specific core
(`/proc/irq/*/smp_affinity`) keeps its processing local (cache-friendly). **NAPI** reduces interrupt
storms by switching to *polling* under load — the NIC interrupts once, then the kernel polls a batch
of packets, instead of one interrupt per packet.

{: .note }
> **Keep a flow on one CPU**
> The recurring theme is *locality*: hash a flow consistently to one queue → one IRQ → one core, so
> its data stays in that core's cache and packets don't reorder. RSS/RPS choose the core; IRQ affinity
> and XPS keep the rest of that flow's work on the same core. Spreading *flows* across cores scales;
> splitting a *single flow* across cores hurts (cache misses, reordering).

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ethtool -k <if>` | Show offload settings (gso, gro, tso, rx/tx checksum) |
| `ethtool -K <if> gro off` | Toggle an offload (for testing) |
| `ethtool -l <if>` / `-L <if> combined N` | Show / set number of NIC queues |
| `cat /proc/interrupts` | See per-queue, per-CPU interrupt counts |
| `echo <mask> > /proc/irq/<n>/smp_affinity` | Pin a queue's IRQ to a CPU |

---

## Lab

Inspect offloads and observe GRO merging packets; veth supports GRO so this works in namespaces.

### Step 1 — Look at offload state

```bash
$ ethtool -k eth0 | grep -E 'generic-receive|generic-segmentation|tcp-segmentation|rx-checksum'
generic-receive-offload: on
generic-segmentation-offload: on
tcp-segmentation-offload: on
rx-checksumming: on
```

### Step 2 — See GRO merging on receive

```bash
# On a veth receiver, capture: with GRO on, tcpdump shows large "super-segments"
$ sudo ip netns exec s tcpdump -ni <s-if> 'tcp' &
$ sudo ip netns exec c iperf3 -c <s-ip> -t 5
# Packet lengths much larger than the MTU appear (e.g. len 64240) — GRO coalesced them.
```

### Step 3 — Turn GRO off and watch packets shrink

```bash
$ sudo ip netns exec s ethtool -K <s-if> gro off
$ sudo ip netns exec c iperf3 -c <s-ip> -t 5
# Now tcpdump shows MTU-sized packets again, and the receiver's CPU works harder per byte.
```

### Step 4 — Check queue/IRQ spread on a real multiqueue NIC

```bash
$ ethtool -l eth0          # Combined: N queues
$ grep eth0 /proc/interrupts   # one line per queue, counts spread across CPU columns (RSS at work)
```

### Step 5 — Restore

```bash
$ sudo ip netns exec s ethtool -K <s-if> gro on
```

---

## Further Reading

| Topic | Link |
|---|---|
| Large send / segmentation offload | [Wikipedia — Large send offload](https://en.wikipedia.org/wiki/Large_send_offload) |
| Receive side scaling & RPS | [kernel.org — scaling.rst](https://www.kernel.org/doc/Documentation/networking/scaling.rst) |
| New API (NAPI) | [Wikipedia — New API](https://en.wikipedia.org/wiki/New_API) |
| `ethtool` | [man7.org — ethtool(8)](https://man7.org/linux/man-pages/man8/ethtool.8.html) |

---

## Checkpoint

**Q1. What does GRO do, and why does it improve receive-side throughput?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
GRO (Generic Receive Offload) <strong>merges many small incoming packets belonging to the same flow
into one large packet</strong> before handing it up the network stack. It helps because the dominant
cost on receive is the per-packet overhead of traversing the stack (protocol processing, bookkeeping,
function calls) — by coalescing, say, 40 MTU-sized packets into one ~64 KB super-packet, the stack does
that work <em>once</em> instead of 40 times, slashing CPU per byte and freeing the core to handle more
traffic. You can see it in tcpdump as received "packets" far larger than the MTU.
</details>

---

**Q2. What is the difference between RSS and RPS, and what problem do both solve?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Both spread received-packet processing <strong>across multiple CPU cores</strong> so a single core
isn't the bottleneck at high packet rates. <strong>RSS</strong> (Receive Side Scaling) does it in
<em>hardware</em>: the NIC hashes each packet's flow to one of its hardware receive queues, each wired
to a different CPU's interrupt — fast and zero CPU cost to distribute. <strong>RPS</strong> (Receive
Packet Steering) is the <em>software</em> equivalent: the kernel hashes flows to CPUs after reception,
used when the NIC lacks enough hardware queues/RSS. Both hash <em>per flow</em> so all packets of a
connection land on the same core (preserving cache locality and ordering); they differ only in whether
the NIC or the kernel does the steering.
</details>

---

**Q3. Why is "keep a flow on one CPU" the guiding principle, rather than splitting one flow across cores?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because scaling comes from running <em>many</em> flows in parallel on different cores, while splitting a
<em>single</em> flow across cores is actively harmful. If one connection's packets are processed on
several cores, you get (1) <strong>cache thrashing</strong> — the socket/TCP state bounces between
cores' caches instead of staying hot on one; and (2) <strong>packet reordering</strong> — different
cores finish at different times, and reordered segments make TCP think there's loss, triggering
spurious retransmits and shrinking the window. So RSS/RPS hash <em>per flow</em> to pin each connection
to one core, IRQ affinity keeps that queue's interrupt on the same core, and XPS keeps its transmit
side there too. Distribute across flows, never within a flow.
</details>

---

## Homework

Measure the CPU cost of GRO: run `iperf3` with GRO on and off (Step 2/3) while watching per-core
utilization (`mpstat -P ALL 1` or `top`). Record throughput and the softirq CPU% in each case, then
explain why disabling a "mere optimization" can turn a CPU-bound receiver from line-rate into a
bottleneck.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With GRO <strong>on</strong>, you should see high throughput at relatively low softirq CPU; with GRO
<strong>off</strong>, throughput drops (or the link only stays full by pegging a core) and softirq CPU%
on the receiving core climbs sharply. The reason: GRO isn't a "mere" optimization — it changes the
<em>number of times</em> the stack runs. Off, the kernel processes every MTU-sized packet individually,
so at high packet-per-second rates the per-packet overhead (protocol handling, allocations, function
calls) saturates a single core's softirq budget and that core becomes the ceiling — extra bandwidth
can't be used because the CPU can't keep up with the packet rate. On, dozens of packets are coalesced
into one traversal, cutting per-byte CPU by roughly the coalescing factor and letting the same core
sustain far higher throughput. This is why on fast NICs offloads are essential, not optional, and why
benchmarks always state whether they were enabled.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 64 — Kernel Bypass (AF_XDP & DPDK) →](lesson-64-kernel-bypass){: .btn .btn-primary }
