---
title: "Lesson 62 — TCP Internals & Congestion Control"
nav_order: 62
parent: "Phase 18: High-Performance Networking"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 62: TCP Internals — Congestion Control & Flow Dynamics

## Concept

TCP throughput isn't set by your link speed alone — it's governed by two windows. The **receive
window** (flow control) stops a fast sender from overwhelming a slow receiver. The **congestion
window (cwnd)** (congestion control) stops senders from overwhelming the *network*. At any moment a
sender may have at most `min(rwnd, cwnd)` bytes in flight.

```
   bytes in flight ≤ min(receive window, congestion window)

   throughput ≈ window_size / RTT      ← the fundamental TCP equation
```

That equation explains everything: to go fast over a high-latency link you need a *big* window;
congestion control is the algorithm that grows and shrinks cwnd safely.

---

## How it works

**Window scaling.** The TCP header's window field is only 16 bits (max 64 KB), far too small for
modern bandwidth×delay products. The **window scale** option multiplies it so windows can reach
megabytes — without it, a 100 ms transcontinental link is capped at ~5 Mbit/s no matter the
bandwidth.

**Slow start & congestion avoidance.** A new connection doesn't know the network's capacity, so it
**slow-starts**: cwnd begins small and *doubles* each RTT until it hits a threshold or detects loss.
After that it enters **congestion avoidance** (additive increase). On loss it backs off — the
classic "AIMD" (additive-increase/multiplicative-decrease) sawtooth.

**Loss-based vs model-based control.**
[CUBIC](https://en.wikipedia.org/wiki/CUBIC_TCP) (the Linux default) treats **packet loss** as the
congestion signal — it fills buffers until something drops, then backs off. That interacts badly
with oversized buffers (**bufferbloat**): the buffer fills, latency balloons, and only then does
CUBIC react. [BBR](https://en.wikipedia.org/wiki/TCP_congestion_control#TCP_BBR) instead **models**
the path's bottleneck bandwidth and round-trip propagation time and paces packets to that estimate,
so it achieves high throughput *without* filling buffers — much better latency under load and on
lossy links where CUBIC misreads loss as congestion.

{: .note }
> **Pacing**
> Modern TCP (especially BBR) doesn't dump a whole window at once; it **paces** packets evenly across
> the RTT. Bursts cause transient queue spikes and loss; pacing smooths them, which is why pacing and
> BBR go together (and why `fq` qdisc, Lesson 30, is recommended with BBR).

**Reading it live.** `ss -ti` exposes cwnd, current RTT, retransmits, and the chosen congestion
control per socket — the single best window into why a connection is slow.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ss -ti` | Per-socket cwnd, RTT, retransmits, congestion algorithm |
| `sysctl net.ipv4.tcp_congestion_control` | Show/set the default congestion control |
| `sysctl net.ipv4.tcp_available_congestion_control` | List available algorithms |
| `sysctl net.ipv4.tcp_window_scaling` | Window scaling on/off |
| `tc qdisc add dev <if> root fq` | Enable fair-queue pacing (pairs with BBR) |

---

## Lab

Watch slow start, then compare CUBIC vs BBR under emulated latency + loss (netem, Lesson 30).

### Step 1 — Two namespaces with an iperf3 path (reuse Lesson 33 setup)

```bash
$ sudo ip netns add c; sudo ip netns add s
# veth c<->s, addresses, both up; run iperf3 -s in s.
```

### Step 2 — Add latency and a little loss, baseline with CUBIC

```bash
$ sudo ip netns exec c tc qdisc add dev <c-if> root netem delay 100ms loss 1%
$ sudo ip netns exec c sysctl -w net.ipv4.tcp_congestion_control=cubic
$ sudo ip netns exec c iperf3 -c <s-ip> -t 10
# Note the throughput and, in another terminal: ss -ti dst <s-ip>  (watch cwnd sawtooth)
```

### Step 3 — Switch to BBR and re-measure

```bash
$ sudo modprobe tcp_bbr
$ sudo ip netns exec c sysctl -w net.ipv4.tcp_congestion_control=bbr
$ sudo ip netns exec c tc qdisc replace dev <c-if> root fq   # pacing for BBR
$ sudo ip netns exec c iperf3 -c <s-ip> -t 10
# On a lossy, high-RTT path BBR typically sustains much higher throughput than CUBIC.
```

### Step 4 — Clean up

```bash
$ sudo ip netns delete c s
```

---

## Further Reading

| Topic | Link |
|---|---|
| TCP congestion control | [Wikipedia — TCP congestion control](https://en.wikipedia.org/wiki/TCP_congestion_control) |
| CUBIC | [Wikipedia — CUBIC TCP](https://en.wikipedia.org/wiki/CUBIC_TCP) |
| Bufferbloat | [Wikipedia — Bufferbloat](https://en.wikipedia.org/wiki/Bufferbloat) |
| `ss` | [man7.org — ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html) |

---

## Checkpoint

**Q1. Why can a fast link still give slow TCP throughput over a long-distance path, and what option fixes it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because TCP throughput ≈ window / RTT: the amount of data in flight is capped by the window, so over a
high-latency path you need a large window to keep the pipe full. The classic limiter is that TCP's
header window field is only 16 bits (max 64 KB); on a 100 ms link, 64 KB / 0.1 s ≈ 5 Mbit/s regardless
of the link's actual bandwidth. The fix is the <strong>window scaling</strong> option, which applies a
multiplier so the effective window can reach megabytes, matching the bandwidth×delay product. Without
adequate window (and buffers), the connection is RTT-bound, not bandwidth-bound.
</details>

---

**Q2. What is the core difference between CUBIC and BBR, and when does BBR win?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>CUBIC</strong> is loss-based: it grows its window until the network drops a packet, treating
that loss as the congestion signal, then backs off. <strong>BBR</strong> is model-based: it actively
estimates the path's bottleneck bandwidth and minimum RTT and paces sending to that model, aiming to
keep the pipe full <em>without</em> filling buffers. BBR wins (1) on paths with <strong>bufferbloat</strong>,
because CUBIC fills the oversized buffer and inflates latency before reacting, while BBR keeps queues
short; and (2) on <strong>lossy</strong> paths (e.g. wireless, long-haul), where random loss isn't
real congestion — CUBIC misreads it and needlessly slashes its window, whereas BBR keeps going. On a
clean, well-buffered LAN the two are similar.
</details>

---

**Q3. What does `ss -ti` tell you, and why is it the first tool to reach for on a slow connection?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<code>ss -ti</code> dumps per-socket TCP internals: the congestion window (cwnd), the smoothed RTT,
retransmission counts, the negotiated MSS, the active congestion-control algorithm, pacing rate, and
more. It's the first tool because it turns "the connection is slow" into a measured diagnosis: a tiny
cwnd with high retransmits points at loss/congestion; a large RTT with a full window points at a
latency-bound path needing bigger windows/buffers; a low pacing rate or unexpected algorithm points at
config. Instead of guessing, you read the actual state the kernel is using to drive the connection.
</details>

---

## Homework

On the lossy 100 ms path, sweep the loss rate (0.1%, 1%, 5%) and measure throughput under CUBIC vs
BBR at each, recording cwnd from `ss -ti`. Plot or tabulate the results and explain the crossover:
at what point does treating loss as congestion become the wrong assumption, and why does BBR's model
hold up?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You should see CUBIC's throughput <strong>collapse as loss rises</strong> while BBR degrades far more
gently, with the gap widening at higher loss. The reason: CUBIC interprets <em>every</em> loss as a
congestion signal and multiplicatively cuts cwnd, so on a path with non-congestive (random) loss it
spends most of its time with a needlessly small window — and the higher the loss rate, the more often
it's knocked down, so cwnd (visible in <code>ss -ti</code>) stays small and throughput craters. BBR
ignores loss as its primary signal; it estimates bottleneck bandwidth and min-RTT and paces to that
model, so random drops don't make it abandon a correct capacity estimate. The "crossover" is wherever
the link's loss stops being a reliable proxy for congestion — on a clean link loss ≈ congestion and
CUBIC is fine, but once loss is dominated by the medium (wireless, long-haul) the loss-equals-congestion
assumption is simply wrong and BBR's measured model wins.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 63 — NIC Offloads & Multiqueue Scaling →](lesson-63-nic-offloads){: .btn .btn-primary }
