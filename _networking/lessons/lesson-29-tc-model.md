---
title: "Lesson 29 — Traffic Control Model"
nav_order: 29
parent: "Phase 9: Traffic Control"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 29: Traffic Control Model

## Concept

When the kernel has finished deciding *where* a packet goes (routing) and *whether* it's allowed (firewall), there's one more stage before it hits the wire: **traffic control** (tc). This is where packets are **queued, scheduled, shaped, and prioritized** on their way out an interface. tc is how you rate-limit a link, give one kind of traffic priority over another, or emulate a slow/lossy network for testing.

The central object is the **qdisc** (queueing discipline) — the algorithm that decides *the order and timing* in which queued packets are sent.

```
   IP routing decides the egress interface
            │
            ▼
   ┌──────────────────────────┐
   │   qdisc (queueing disc.)  │  ← packets wait here; the qdisc picks
   │   on the egress interface │     who goes next and when
   └────────────┬─────────────┘
                ▼
            NIC driver → wire
```

tc operates almost entirely on **egress** (outbound). The kernel has rich control over what *it* sends; it has far less control over what *arrives* (ingress), which is why shaping is fundamentally an egress activity.

---

## How it works

Every interface has a **root qdisc**. By default it's something simple like `pfifo_fast` (a basic priority FIFO) or, on modern kernels, `fq_codel`. When the network stack wants to transmit a packet, it doesn't go straight to the NIC — it's **enqueued** into the qdisc. The driver then **dequeues** packets from the qdisc when it's ready to send. The qdisc's algorithm decides the dequeue order and rate.

Key concepts you'll use throughout the phase:

| Concept | Meaning |
|---|---|
| **rate** | The sustained speed the qdisc allows packets out (e.g. 1mbit). |
| **burst** | How much data can be sent instantaneously before the rate limit kicks in (a token bucket allowance). |
| **latency / limit** | How long / how many packets the qdisc will hold before dropping (queue depth). |
| **backlog** | Packets currently waiting in the queue. |

Two families of qdisc:

- **Classless** — a single algorithm applied to all traffic on the interface (e.g. `tbf`, `netem`, `fq_codel`). Simple: one queue, one policy. (Lesson 30)
- **Classful** — contains a *hierarchy of classes*, each potentially with its own qdisc, so you can treat different traffic differently (e.g. `htb`). Powerful: many queues, per-class policy. (Lesson 31)

{: .note }
> **Why ingress shaping is hard (and "policing" vs "shaping")**
> By the time a packet *arrives*, the bandwidth was already consumed on the wire — you can't un-send it. So you can't truly *shape* (smoothly delay) incoming traffic; you can only **police** it (drop/mark packets that exceed a rate, hoping the sender backs off, as TCP does). Real shaping — buffering and releasing packets at a controlled rate — only works on egress, where the kernel owns the timing. This is why almost all tc work happens on the egress side, and why "shape your downstream" usually means shaping the *other* end's egress.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `tc qdisc show dev <iface>` | Show the qdisc(s) on an interface |
| `tc -s qdisc show dev <iface>` | Show qdisc stats (sent bytes, drops, backlog) |
| `tc qdisc add dev <iface> root <qdisc>` | Set the root qdisc |
| `tc qdisc del dev <iface> root` | Remove the root qdisc (back to default) |
| `tc qdisc replace dev <iface> root <qdisc>` | Replace the root qdisc atomically |
| `ip -s link show dev <iface>` | Interface-level TX/RX counters |

---

## Lab

This lesson is about *seeing* the model; Lessons 30–32 do the active shaping. Here you inspect and understand the default qdisc.

### Step 1 — A namespace with a veth to observe

```bash
$ sudo ip netns add lab
$ sudo ip netns exec lab ip link set lo up
$ sudo ip netns exec lab ip link add eth0 type dummy
$ sudo ip netns exec lab ip link set eth0 up
$ sudo ip netns exec lab ip addr add 10.0.0.1/24 dev eth0
```

### Step 2 — Look at the default qdisc

```bash
$ sudo ip netns exec lab tc qdisc show dev eth0
qdisc noqueue 0: root refcnt 2
# (dummy/virtual interfaces often default to 'noqueue'; physical NICs
#  typically show 'fq_codel' or 'pfifo_fast')
```

On a real interface you'd see something like:

```
qdisc fq_codel 0: root refcnt 2 limit 10240p flows 1024 quantum 1514 ...
```

`fq_codel` = fair queuing with controlled delay, the modern default that fights bufferbloat.

### Step 3 — See the stats view

```bash
$ sudo ip netns exec lab tc -s qdisc show dev eth0
qdisc ... 
 Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0)
 backlog 0b 0p requeues 0
```

The key fields:
- `Sent` — total bytes/packets the qdisc has released.
- `dropped` — packets discarded (queue full, or policed).
- `backlog` — how much is waiting right now.
- `overlimits` — times the rate limit was hit (relevant once you add shaping).

### Step 4 — Swap the root qdisc and observe the change

```bash
# Replace with a simple priority FIFO
$ sudo ip netns exec lab tc qdisc replace dev eth0 root pfifo_fast
$ sudo ip netns exec lab tc qdisc show dev eth0
qdisc pfifo_fast 0: root refcnt 2 bands 3 priomap ...
```

`pfifo_fast` has 3 priority **bands**: higher-priority packets (by their ToS/DSCP) are dequeued before lower ones. This is the simplest form of prioritization — and a preview of why classful qdiscs exist.

### Step 5 — Where the qdisc sits, proven by counters

```bash
$ sudo ip netns exec lab ip -s link show dev eth0
# TX: bytes/packets counted AFTER the qdisc releases them to the driver.
```

The interface TX counters reflect what the qdisc *dequeued and sent*. If a qdisc is shaping/dropping, the qdisc `Sent`/`dropped` stats and the interface TX counters together tell you what got through vs. held back.

### Step 6 — Reset and clean up

```bash
$ sudo ip netns exec lab tc qdisc del dev eth0 root
$ sudo ip netns delete lab
```

---

## Further Reading

| Topic | Link |
|---|---|
| Linux traffic control | [Wikipedia — tc (Linux)](https://en.wikipedia.org/wiki/Tc_(Linux)) |
| Network scheduler / qdisc | [Wikipedia — Network scheduler](https://en.wikipedia.org/wiki/Network_scheduler) |
| Bufferbloat & fq_codel | [Wikipedia — Bufferbloat](https://en.wikipedia.org/wiki/Bufferbloat) |
| `tc` | [man7.org — tc(8)](https://man7.org/linux/man-pages/man8/tc.8.html) |

---

## Checkpoint

**Q1. Draw (in words) where a qdisc sits relative to IP routing and the NIC driver, and explain what it does there.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The order on the egress path is: **IP routing → qdisc → NIC driver → wire.** First, IP routing decides *which* interface the packet leaves by. Then, instead of going straight to the hardware, the packet is **enqueued into that interface's root qdisc**. The qdisc holds packets in one or more queues and runs an algorithm that decides the *order and timing* in which they're released. When the NIC driver is ready to transmit, it **dequeues** packets from the qdisc. So the qdisc is the buffering-and-scheduling stage between the routing decision and the hardware: it's where rate limiting, prioritization, fair queuing, and network emulation happen. The NIC driver only ever sends what the qdisc hands it, which is exactly why tc can control outbound bandwidth and ordering — it controls the gate between the stack and the wire.
</details>

---

**Q2. Why does traffic control operate mostly on egress rather than ingress?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because the kernel only fully controls the *timing* of packets it is about to send. On **egress**, packets are buffered in the qdisc and the kernel decides exactly when to release each one — so it can smoothly shape, delay, reorder, and prioritize them before they hit the wire. On **ingress**, by the time a packet arrives, the upstream bandwidth has *already been spent* transmitting it; the kernel can't retroactively slow down something already received. The best it can do for inbound traffic is **police** it — drop or mark packets that exceed a target rate — and rely on the sender (e.g., TCP congestion control) to slow down in response. That's coarser and lossier than true shaping. Real, smooth rate control requires owning the transmit timing, which you only have on egress. This is why "shaping the download" practically means shaping the *sender's* egress, and why nearly all tc configuration targets the outbound path.
</details>

---

**Q3. What is the difference between a classless and a classful qdisc?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A **classless** qdisc applies a *single* queueing algorithm to *all* traffic on the interface — there's one policy and no way to treat different flows differently within it. Examples: `tbf` (a flat rate limit), `netem` (uniform delay/loss emulation), `fq_codel` (fair queuing for everything). They're simple to configure and ideal when you want one uniform behavior.

A **classful** qdisc contains a *hierarchy of classes*, and you can attach different child qdiscs and policies to each class, plus **filters** that sort packets into the right class. This lets you give different traffic different treatment — e.g., guarantee 5mbit to web traffic and 2mbit to backups, with borrowing of unused bandwidth between them. The canonical example is `htb` (Hierarchical Token Bucket, Lesson 31). 

In short: classless = one queue, one rule, for all traffic; classful = a tree of classes with per-class rules and a classifier to direct packets into them. You reach for classful when you need to *prioritize or partition* bandwidth among categories of traffic; classless suffices when one uniform policy is enough.
</details>

---

## Homework

On a real interface (or a veth pair carrying real traffic), inspect the default qdisc with `tc -s qdisc show`. Generate sustained traffic (a ping flood or a file transfer) and watch the `Sent`, `backlog`, and `dropped` counters change. Then replace the root qdisc with `pfifo_fast` and again with `fq_codel`, and describe what each counter tells you and how the two qdiscs differ in intent.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Reading the counters under load:**
- `Sent` climbs steadily as the qdisc releases packets to the driver — it's the cumulative throughput that made it out.
- `backlog` shows the *instantaneous* queue occupancy: it rises when packets arrive faster than they can be sent (the link or a rate limit is the bottleneck) and falls when the queue drains. A persistently high backlog indicates congestion/bufferbloat.
- `dropped` increments when the queue overflows its limit (or, with shaping, when policing discards excess) — packets the qdisc had to discard rather than send.

**pfifo_fast:** a simple 3-band priority FIFO. Its intent is minimal: send packets roughly first-in-first-out, but let higher-priority (by ToS/DSCP) packets jump ahead of lower-priority ones across its three bands. It does *no* active queue management — under sustained overload its single FIFO per band can fill up and induce large latency (bufferbloat) before it finally drops. It's fast and cheap but naive about delay.

**fq_codel:** fair queuing + CoDel (Controlled Delay). Its intent is to keep latency low *and* share bandwidth fairly. It hashes flows into many sub-queues so one heavy flow can't starve others (fairness), and the CoDel algorithm actively drops/marks packets when their *sojourn time* in the queue grows too long, signaling senders to slow down *before* the queue bloats. Under the same load you'll typically see a much smaller, better-controlled `backlog` and more proactive `dropped` counts that keep latency bounded — versus pfifo_fast letting the queue grow large.

The contrast captures the evolution of qdisc design: pfifo_fast optimizes for simplicity and priority ordering; fq_codel optimizes for fair, low-latency behavior under load, which is why it's the modern default. Both are *classless* — they apply one policy to all traffic — which sets up the need for *classful* HTB when you want per-category bandwidth control.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 30 — Classless qdiscs: Shaping & Emulation →](lesson-30-classless-qdisc){: .btn .btn-primary }
