---
title: "Lesson 30 — Classless qdiscs: Shaping & Emulation"
nav_order: 30
parent: "Phase 9: Traffic Control"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 30: Classless qdiscs — Shaping and Emulation

## Concept

Classless qdiscs apply one policy to all traffic on an interface. Two are indispensable:

- **`tbf`** (Token Bucket Filter) — a hard **rate limiter**. Use it to cap a link to, say, 1 Mbit/s.
- **`netem`** (Network Emulator) — adds **delay, loss, reordering, duplication, corruption** to emulate a bad/slow/distant network. This is how you test how software behaves on a satellite link, a lossy mobile connection, or a transatlantic path — without leaving your desk.

```
   netem delay 100ms loss 1%:
     every packet held ~100ms, 1 in 100 dropped
   ┌──────────────────────────────────────────┐
   │  send → [ +100ms delay,  1% dropped ]  →  │  looks like a slow, lossy WAN
   └──────────────────────────────────────────┘
```

`netem` in particular is one of the most useful tools in this whole curriculum — it lets you reproduce "it only breaks on slow connections" bugs deterministically.

---

## How it works

**TBF / token bucket:** imagine a bucket filling with tokens at a fixed `rate`. Sending a packet costs tokens proportional to its size. If the bucket has tokens, the packet goes immediately; if not, it waits until enough tokens accumulate. The `burst` parameter is the bucket's size — it sets how much data can go out *instantaneously* (using saved-up tokens) before the steady `rate` limit dominates. `latency` caps how long a packet may wait before being dropped.

```
   tc qdisc add dev eth0 root tbf rate 1mbit burst 32kbit latency 400ms
                                   │         │            │
                                   │         │            └ max queue wait before drop
                                   │         └ instantaneous allowance (bucket size)
                                   └ sustained rate
```

**netem:** intercepts packets on egress and applies impairments. `delay 100ms 10ms` means 100ms ± 10ms jitter. `loss 1%` drops 1% of packets randomly. You can combine them (`netem delay 100ms loss 1% reorder 25% 50%`) to model realistic conditions.

{: .note }
> **netem applies to egress — emulate both directions by configuring both ends**
> Because tc shapes egress, `netem delay 100ms` on one host's interface adds 100ms to packets *leaving* that host. The round-trip a `ping` measures is the sum of delays in *both* directions. If you add 100ms on only one side, RTT rises by ~100ms (one-way). To emulate a symmetric 100ms-each-way link (200ms RTT), apply `netem delay 100ms` on *each* end's egress. Forgetting this is the usual reason netem RTTs are "half what I expected."

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `tc qdisc add dev <if> root tbf rate 1mbit burst 32kbit latency 400ms` | Hard rate-limit to 1 Mbit/s |
| `tc qdisc add dev <if> root netem delay 100ms` | Add 100ms egress delay |
| `tc qdisc add dev <if> root netem delay 100ms 20ms` | Delay with ±20ms jitter |
| `tc qdisc add dev <if> root netem loss 5%` | Drop 5% of packets |
| `tc qdisc add dev <if> root netem delay 100ms loss 1% reorder 25% 50%` | Combined impairments |
| `tc qdisc change dev <if> root netem delay 200ms` | Modify an existing netem |
| `tc qdisc del dev <if> root` | Remove it |

---

## Lab

We'll use a veth pair between two namespaces and impair one direction, measuring with ping and observing throughput.

### Step 1 — Two namespaces on a veth

```bash
$ sudo ip netns add a
$ sudo ip netns add b
$ sudo ip link add va type veth peer name vb
$ sudo ip link set va netns a
$ sudo ip link set vb netns b
$ sudo ip netns exec a ip link set va up
$ sudo ip netns exec a ip addr add 10.0.0.1/24 dev va
$ sudo ip netns exec b ip link set vb up
$ sudo ip netns exec b ip addr add 10.0.0.2/24 dev vb
```

### Step 2 — Baseline RTT (no impairment)

```bash
$ sudo ip netns exec a ping -c 3 10.0.0.2
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.05 ms     # microseconds
```

### Step 3 — Add 100ms delay on a's egress

```bash
$ sudo ip netns exec a tc qdisc add dev va root netem delay 100ms
$ sudo ip netns exec a ping -c 3 10.0.0.2
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=100 ms       # +100ms one way
```

RTT jumped to ~100ms because the echo *request* is delayed 100ms on a's egress (the reply path is still fast). To get a symmetric 200ms RTT, also add delay on b's egress:

```bash
$ sudo ip netns exec b tc qdisc add dev vb root netem delay 100ms
$ sudo ip netns exec a ping -c 3 10.0.0.2
64 bytes from 10.0.0.2: time=200 ms                          # 100ms each way
```

### Step 4 — Add jitter and packet loss

```bash
$ sudo ip netns exec a tc qdisc change dev va root netem delay 100ms 20ms loss 10%
$ sudo ip netns exec a ping -c 10 10.0.0.2
# Times vary (80–120ms jitter); ~1 in 10 pings shows as lost:
# 10 packets transmitted, 9 received, 10% packet loss
```

You've reproduced a flaky mobile-style link deterministically.

### Step 5 — Hard rate limit with TBF

Remove netem and apply a rate cap, then measure throughput with iperf3:

```bash
$ sudo ip netns exec a tc qdisc del dev va root
$ sudo ip netns exec b tc qdisc del dev vb root

# Baseline throughput (veth is very fast)
$ sudo ip netns exec b iperf3 -s &
$ sudo ip netns exec a iperf3 -c 10.0.0.2 -t 5
# ... several Gbits/sec ...

# Now cap a's egress to 10 Mbit
$ sudo ip netns exec a tc qdisc add dev va root tbf rate 10mbit burst 32kbit latency 400ms
$ sudo ip netns exec a iperf3 -c 10.0.0.2 -t 5
# ... ~10 Mbits/sec — the cap is enforced ...
```

The throughput drops to the configured rate. TBF turned a multi-gigabit virtual link into a 10 Mbit one.

### Step 6 — Inspect the qdisc stats

```bash
$ sudo ip netns exec a tc -s qdisc show dev va
qdisc tbf ... rate 10Mbit burst ...
 Sent 6250000 bytes ... (dropped 0, overlimits N ...)
#                                    ^^^^^^^^^^^^ times the rate limit engaged
```

`overlimits` rising confirms the rate limiter is actively throttling.

### Step 7 — Clean up

```bash
$ sudo ip netns delete a b
```

---

## Further Reading

| Topic | Link |
|---|---|
| Token bucket | [Wikipedia — Token bucket](https://en.wikipedia.org/wiki/Token_bucket) |
| netem | [man7.org — tc-netem(8)](https://man7.org/linux/man-pages/man8/tc-netem.8.html) |
| TBF | [man7.org — tc-tbf(8)](https://man7.org/linux/man-pages/man8/tc-tbf.8.html) |
| Traffic shaping | [Wikipedia — Traffic shaping](https://en.wikipedia.org/wiki/Traffic_shaping) |

---

## Checkpoint

**Q1. What is the difference between a traffic shaper and a scheduler?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A **shaper** controls the *rate and timing* at which packets leave — it delays packets (buffering them) so that the output conforms to a target bandwidth, smoothing bursts down to a sustained rate. TBF is a pure shaper: it holds packets until tokens are available, so the egress never exceeds the configured rate. Shaping answers "how fast may traffic go out?"

A **scheduler** controls the *order* in which queued packets are sent — given multiple packets waiting, which goes next? `pfifo_fast`'s priority bands and `fq_codel`'s fair-queuing are schedulers: they decide sequencing/fairness among packets, not the absolute rate. Scheduling answers "who goes first?"

The two often work together (a classful setup schedules *between* classes and shapes *within* them), but conceptually shaping = rate/timing control via delay, scheduling = ordering/fairness control via dequeue selection. A shaper can add latency to enforce a rate; a scheduler reorders without necessarily limiting total throughput.
</details>

---

**Q2. You add `netem delay 100ms` to one end of a veth and ping across it, but RTT only rises by ~100ms, not 200ms. Is netem broken?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
No, it's working correctly. `netem` shapes **egress** (outbound) traffic only. Adding `delay 100ms` on one end delays packets *leaving* that end — so the echo *request* (going out that interface) is delayed by 100ms, but the echo *reply* travels back over the *other* direction, which has no delay configured. The round-trip therefore includes only one 100ms delay, giving ~100ms RTT, not 200ms. To emulate a link with 100ms latency in *each* direction (a 200ms RTT), you must apply `netem delay 100ms` on **both ends' egress** interfaces. The "missing" 100ms isn't a bug — it's a consequence of tc being an egress mechanism, so each direction's latency must be configured on the side that *sends* in that direction. This is the single most common netem surprise.
</details>

---

**Q3. In `tbf rate 1mbit burst 32kbit`, what does `burst` control, and what happens if you set it too small?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`burst` is the size of the token bucket — it controls how much data can be sent **instantaneously** (in a quick burst) using accumulated tokens, before the steady `rate` limit takes over. A larger burst lets short spikes of traffic pass through at full speed (smoothing brief bursts), while still enforcing the long-term average `rate`.

If you set `burst` **too small**, the bucket can't hold enough tokens to send even a single full-rate quantum of data at the configured rate, and TBF becomes unable to actually achieve the target rate — throughput ends up *lower* than `rate` because the limiter is constantly starved for tokens and serializes packets too aggressively. There's a minimum sensible burst tied to the rate and the kernel timer resolution: roughly, burst must be at least `rate / HZ` (enough bytes to cover one timer tick at the target rate). Too-small burst is a classic mistake that makes people think "TBF can't hit the rate I asked for." The fix is to size burst appropriately for the rate (higher rates need larger bursts). Conversely, too *large* a burst lets longer spikes exceed the intended smoothing, so it's a balance: big enough to reach the rate, small enough to actually shape.
</details>

---

## Homework

Build a veth between two namespaces. Use `netem` to emulate a realistic transatlantic link: ~80ms each-way delay with ±10ms jitter and 0.5% loss. Measure RTT with ping and throughput with iperf3 (TCP). Then *without changing netem*, observe how TCP throughput responds to the latency and loss. Explain why high latency alone reduces TCP throughput even with no packet loss (hint: bandwidth-delay product and the TCP window).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Setup: apply `tc qdisc add dev <if> root netem delay 80ms 10ms loss 0.5%` on **both** ends' egress (so RTT ≈ 160ms with jitter). ping confirms ~160ms RTT with variation; occasional losses appear.

**Observations:** TCP throughput drops far below the link's raw capacity, and adding the 0.5% loss reduces it further and makes it more variable.

**Why latency alone limits TCP throughput (no loss needed):** TCP can only have so much data "in flight" (sent but not yet acknowledged) at once — bounded by the **receive/congestion window**. Throughput is governed by the **bandwidth-delay product (BDP)**: the maximum achievable throughput ≈ window size ÷ RTT. With a 160ms RTT, even a modest window means each window's worth of data takes a full round-trip to be acknowledged before more can be sent. If the window is, say, 64 KB and RTT is 160ms, max throughput ≈ 64 KB / 0.16 s ≈ 400 KB/s ≈ 3.2 Mbit/s — regardless of how fast the underlying link is. To fill a high-bandwidth, high-latency "long fat pipe," you need a *large* window (which is why TCP window scaling exists); without enough window, the connection spends most of its time idle, waiting for ACKs to come back across the long RTT.

**Why loss makes it worse:** TCP interprets loss as congestion and **halves its congestion window** (then slowly grows it again — the classic sawtooth). With a long RTT, recovering/growing the window back takes many slow round-trips, so even a small loss rate keeps the average window — and thus throughput — well below the link capacity. The combination of high RTT (slows window growth and ACK feedback) and loss (repeatedly shrinks the window) is exactly why real long-distance, lossy links deliver a fraction of their nominal bandwidth to a single TCP flow — and exactly the scenario `netem` lets you reproduce and study deterministically. The takeaway: throughput is not just "link speed"; for TCP it's fundamentally `window / RTT`, which is why latency is a first-class performance factor, not just a nuisance.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 31 — Classful qdiscs: HTB →](lesson-31-htb){: .btn .btn-primary }
