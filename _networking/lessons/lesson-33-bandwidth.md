---
title: "Lesson 33 — Bandwidth Measurement"
nav_order: 33
parent: "Phase 9: Traffic Control"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 33: Bandwidth Measurement

## Concept

You've been shaping traffic — but how do you *measure* whether the shaping actually works? This lesson covers the measurement toolkit: **`iperf3`** for active throughput tests, **`ip -s link`** for interface counters, and **`ss -ti`** for per-socket TCP internals (RTT, retransmits, congestion window). Measurement turns "I think it's limited to 10 Mbit" into "it is, here's the number."

```
   iperf3 -s  (server)  ◄──────── test traffic ────────  iperf3 -c <server>
                                                          (client generates load)
        reports: throughput, retransmits, jitter (UDP), per-interval rates
```

The golden workflow: **baseline → apply change → measure again → confirm.** Never assume a shaping rule works; prove it with numbers.

---

## How it works

**iperf3** runs a server (`-s`) and a client (`-c`). The client floods the server (or vice versa) and both report throughput. Key modes:

| Flag | What it tests |
|---|---|
| `iperf3 -c <host>` | TCP throughput, client → server (the default direction) |
| `iperf3 -R` | **Reverse** — server → client (tests the *other* direction's bottleneck) |
| `iperf3 -u -b 100M` | UDP at a target rate (measures loss/jitter, not just throughput) |
| `iperf3 -P 4` | 4 parallel streams (saturate links a single flow can't) |
| `iperf3 -t 30` | Run for 30 seconds |

**Why direction matters:** TCP throughput can be limited by different things in each direction — the sender's congestion window, the receiver's advertised window, the path's shaping, or asymmetric link rates. `iperf3 -R` reverses who sends, so it stresses the *return* path. A link shaped to 10 Mbit *out* but 100 Mbit *in* will show wildly different numbers forward vs reverse.

**ss -ti** exposes the kernel's per-socket TCP state — the actual congestion window (`cwnd`), measured RTT, retransmits, and delivery rate. This is how you see *why* throughput is what it is, not just *what* it is.

{: .note }
> **Interface counters vs application throughput**
> `iperf3` reports *application-level* goodput (payload delivered). `ip -s link show dev <if>` reports *all* bytes/packets crossing the interface, including headers, retransmissions, ARP, and other traffic. They won't match exactly — header overhead and retransmits mean wire bytes exceed application goodput. Use iperf3 to measure useful throughput; use `ip -s link` to see total wire activity and spot drops/errors (the `dropped`/`errors` columns). Both together localize whether a shortfall is shaping, loss, or overhead.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `iperf3 -s` | Run as server |
| `iperf3 -c <host>` | Run as client (TCP, forward direction) |
| `iperf3 -c <host> -R` | Reverse direction (server sends) |
| `iperf3 -c <host> -u -b 50M` | UDP test at 50 Mbit target |
| `iperf3 -c <host> -P 4 -t 20` | 4 parallel streams, 20 seconds |
| `ip -s link show dev <if>` | Interface RX/TX byte/packet/error/drop counters |
| `ss -ti` | Per-socket TCP info: cwnd, rtt, retransmits, rate |

---

## Lab

We'll baseline a veth, apply HTB shaping, and measure again — confirming the limit is enforced.

### Step 1 — Veth between two namespaces

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

### Step 2 — Baseline throughput

```bash
$ sudo ip netns exec b iperf3 -s &
$ sudo ip netns exec a iperf3 -c 10.0.0.2 -t 5
[ ID] Interval   Transfer   Bitrate
[  5] 0.00-5.00  XX GBytes  YY Gbits/sec      # veth is very fast — multi-Gbit
```

Record this baseline.

### Step 3 — Apply a 20 Mbit shaper and re-measure

```bash
$ sudo ip netns exec a tc qdisc add dev va root tbf rate 20mbit burst 32kbit latency 400ms
$ sudo ip netns exec a iperf3 -c 10.0.0.2 -t 5
[  5] 0.00-5.00  ~12 MBytes  ~20 Mbits/sec    # capped at the shaped rate
```

The number dropped to ~20 Mbit — the shaping is *proven*, not assumed.

### Step 4 — Test the reverse direction

The shaper is on a's egress (forward direction). Reverse the test:

```bash
$ sudo ip netns exec a iperf3 -c 10.0.0.2 -R -t 5
[  5] 0.00-5.00  XX GBytes  YY Gbits/sec       # FAST again!
```

Reverse is fast because b → a traffic egresses b's interface, which has *no* shaper. This proves shaping is directional: the cap only applies to the direction whose egress you shaped. To limit both directions, shape both `va` and `vb`.

### Step 5 — Inspect per-socket TCP internals during a test

```bash
# While an iperf3 runs, in another terminal:
$ sudo ip netns exec a ss -ti
ESTAB ... 
   cubic wscale:7,7 rto:204 rtt:0.1/0.05 cwnd:10 ... 
   send YYbps ... retrans:0/0 ...
#  └ congestion control  └ measured RTT   └ congestion window  └ retransmits
```

`cwnd`, `rtt`, and `retrans` together explain throughput: a small cwnd or high retrans means the connection isn't filling the pipe. Under a shaper you'll see cwnd stabilize where rate × RTT allows.

### Step 6 — Confirm with interface counters

```bash
$ sudo ip netns exec a ip -s link show dev va
... TX:  bytes  packets  errors  dropped ...
        <wire bytes — higher than iperf3 goodput due to headers/retrans>
```

Compare TX bytes (wire) to iperf3's reported transfer (goodput). The gap is overhead. The `dropped` column reveals if the shaper queue overflowed.

### Step 7 — Clean up

```bash
$ sudo ip netns delete a b
```

---

## Further Reading

| Topic | Link |
|---|---|
| iperf | [Wikipedia — iperf](https://en.wikipedia.org/wiki/Iperf) |
| iperf3 docs | [software.es.net — iperf3](https://software.es.net/iperf/) |
| `ss` socket statistics | [man7.org — ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html) |
| TCP congestion window | [Wikipedia — TCP congestion control](https://en.wikipedia.org/wiki/TCP_congestion_control) |

---

## Checkpoint

**Q1. Why does `iperf3 -R` test a different bottleneck than the default direction?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`iperf3 -R` reverses which side *sends* the data: instead of the client pushing to the server (the default), the **server sends to the client**. This matters because the data now travels the **opposite direction** over the path, and each direction can have a different bottleneck:

- **Shaping/qdiscs are per-direction (egress).** A link shaped to 10 Mbit on one host's egress only limits traffic *leaving* that host; the return direction egresses the *other* host and may be unshaped or shaped differently. So forward and reverse tests can show very different numbers.
- **Asymmetric link rates** (common on consumer internet: fast download, slow upload) mean the forward and reverse capacities genuinely differ.
- **Windowing** can differ if the two ends have different buffer/window settings.

So the default test measures the bottleneck in the client→server direction, while `-R` measures the server→client direction's bottleneck. Testing both is how you discover asymmetry — e.g., "uploads are capped at 5 Mbit but downloads hit 100 Mbit" — which a single-direction test would completely miss.
</details>

---

**Q2. After applying a TBF shaper to an interface's egress, `iperf3` forward shows the expected limit but `iperf3 -R` shows full speed. Is the shaper broken?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
No — this is exactly correct behavior, and it demonstrates that **tc shaping is directional (egress-only)**. The TBF shaper you added limits traffic *leaving* the interface it's attached to. In the forward `iperf3` direction, the client's data egresses that shaped interface, so it's capped at the configured rate. In the reverse direction (`-R`), the data is sent by the *other* host and egresses the *other* interface, which has no shaper — so it runs at full speed. The shaper isn't broken; it's only ever affecting one direction because that's all egress shaping can do. If you want to limit both directions, you must apply a shaper to **both** interfaces' egress (one on each end). This is a frequent point of confusion: people add one shaper and expect bidirectional limiting, then are surprised that downloads (or the reverse test) are unaffected. The fix is to shape each direction at the point where that direction's traffic egresses.
</details>

---

**Q3. `iperf3` reports 18 Mbit/s of throughput, but `ip -s link show` reports more bytes transmitted than that over the same interval. Why don't they match, and which should you trust for "useful throughput"?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
They measure different things. **`iperf3` reports application-level goodput** — the actual payload data delivered to the application, excluding protocol overhead. **`ip -s link` counts every byte on the wire**, which includes: TCP/IP/Ethernet **headers** on every packet, **retransmitted** segments (sent more than once but counted each time), ACKs, and any other traffic crossing the interface (ARP, control packets). All that overhead means the wire byte count is legitimately *higher* than the application goodput — there's no contradiction. A rough rule: header overhead alone adds a few percent; significant divergence plus a rising `dropped`/`errors`/retransmit count signals loss or inefficiency.

For **"useful throughput"** — how much real data your application moves — **trust `iperf3`'s goodput number**. Use `ip -s link` to understand *total wire activity*, to spot drops/errors, and to see overhead, but not as the measure of useful application throughput. The two are complementary: iperf3 answers "how much real work got done," `ip -s link` answers "how much actually went over the wire and was any of it lost."
</details>

---

## Homework

Baseline a veth link with `iperf3`. Then apply an HTB hierarchy with a 30 Mbit ceiling and verify the cap with `iperf3` (both forward and `-R`). While a shaped transfer runs, capture `ss -ti` output a few times and record `cwnd`, `rtt`, and `retrans`. Explain how the congestion window relates to the shaped rate and RTT, and what you'd expect `cwnd` to settle near (hint: bandwidth-delay product).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Procedure:** baseline shows multi-Gbit (veth). After adding HTB with a 30 Mbit ceil on a's egress, the forward `iperf3` reports ~30 Mbit (cap enforced); `iperf3 -R` shows full speed unless you also shape b's egress (directionality, as in Q2). During the shaped transfer, `ss -ti` shows the live TCP state.

**What you observe and why:** the shaper introduces queueing, which raises the measured `rtt` compared to the unshaped baseline (packets wait in the TBF/HTB queue). The `cwnd` (congestion window) grows until the connection is delivering data at the shaped rate, then stabilizes. `retrans` stays low if the queue is sized well (latency parameter generous), but rises if the queue overflows and drops packets.

**cwnd, rate, and RTT — the relationship:** TCP throughput ≈ `cwnd / RTT` (the window's worth of data is sent each round-trip). For a connection limited to a shaped rate `R`, the congestion window will settle near the **bandwidth-delay product**:

```
cwnd ≈ R × RTT
```

That is, the amount of data "in flight" needed to keep the shaped pipe full equals the rate times the round-trip time. Concretely, at 30 Mbit (~3.75 MB/s) with an RTT of, say, 20ms (inflated by the shaper's queue), the BDP is about 3.75 MB/s × 0.02 s ≈ 75 KB — so cwnd should hover around ~75 KB worth of segments. If cwnd is *smaller* than the BDP, throughput falls short of the shaped rate (the pipe isn't kept full); if the sender tries to push cwnd much *larger*, the excess just piles up in the shaper's queue, inflating RTT further (and eventually causing drops that pull cwnd back down). So the shaped rate and the resulting RTT together pin where cwnd stabilizes — a concrete, observable instance of the `throughput = window / RTT` law you met with netem in Lesson 30. The measurement skill: read `cwnd`, `rtt`, and `retrans` from `ss -ti` to *explain* a throughput number rather than just observe it.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 34 — Network sysctl Tunables →](lesson-34-sysctl-tunables){: .btn .btn-primary }
