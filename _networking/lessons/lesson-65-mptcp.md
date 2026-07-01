---
title: "Lesson 65 — Multipath TCP (MPTCP)"
nav_order: 65
parent: "Phase 18: High-Performance Networking"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 65: Multipath TCP (MPTCP)

## Concept

Ordinary TCP is single-path: one connection rides one source/destination pair, so if that path dies
(WiFi drops) or is slow, the connection suffers. **MPTCP** lets a single logical TCP connection use
**multiple paths at once** — e.g. WiFi *and* cellular — by splitting it into **subflows**, each a
real TCP connection over a different interface, presented to the app as one ordinary socket.

```
   Plain TCP:   app ── one TCP ── one path
   MPTCP:       app ── MPTCP ──┬─ subflow over WiFi
                               └─ subflow over LTE     (aggregate or fail over, app unaware)
```

The application sees a normal stream socket; the kernel manages the subflows underneath.

---

## How it works

**Subflows.** An MPTCP connection starts as a normal TCP handshake carrying the `MP_CAPABLE` option.
Once established, additional **subflows** can be opened over other addresses/interfaces (announced via
`ADD_ADDR`), each joining with `MP_JOIN`. Data is striped across subflows and reassembled in order
using a connection-level sequence number on top of each subflow's own TCP sequencing.

**Two goals: failover vs aggregation.**
- **Failover/resilience:** keep a backup subflow ready; if the active path dies, traffic continues on
  the other with no broken connection (the phone walking out of WiFi range onto LTE — the download
  doesn't drop).
- **Aggregation/bandwidth:** use multiple paths simultaneously to exceed any single path's throughput
  (subject to a path manager and congestion control that's fair to single-path flows sharing a
  bottleneck).

**Mobility/roaming.** Because the connection isn't bound to one 5-tuple, MPTCP survives address
changes gracefully — conceptually similar to WireGuard's roaming (Lesson 51), but at the transport
layer for plain TCP.

**On Linux.** Modern kernels have native MPTCP. You enable it per-socket (or system-wide via sysctl)
and manage subflow creation with `ip mptcp` (endpoints + path-manager limits). Existing apps can be
opted in with `mptcpize run`.

{: .note }
> **Middlebox reality**
> MPTCP options can be stripped by NATs/firewalls that don't understand them; MPTCP is designed to
> <em>fall back</em> to plain TCP transparently if the extra options don't survive the path. This
> "fail safe to TCP" design is why it can be deployed on the real internet at all.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `sysctl net.mptcp.enabled=1` | Enable MPTCP system-wide |
| `ip mptcp endpoint add <ip> dev <if> signal` | Announce an extra address as a subflow endpoint |
| `ip mptcp limits set subflow N add_addr_accepted M` | Path-manager limits |
| `ip mptcp endpoint show` | List MPTCP endpoints |
| `mptcpize run <program>` | Run an unmodified app with MPTCP |
| `ss -tiM` | Show MPTCP connection/subflow info |

---

## Lab

Make a connection use two paths between two namespaces and watch a subflow take over when one path
fails.

### Step 1 — Two namespaces joined by TWO veth links (two paths)

```bash
$ sudo ip netns add c; sudo ip netns add s
$ for i in 1 2; do sudo ip link add c$i type veth peer name s$i; \
    sudo ip link set c$i netns c; sudo ip link set s$i netns s; done
# Address each link on a different subnet; bring all up. Enable MPTCP on both:
$ for n in c s; do sudo ip netns exec $n sysctl -w net.mptcp.enabled=1; done
```

### Step 2 — Announce the second address as an MPTCP endpoint

```bash
$ sudo ip netns exec c ip mptcp limits set subflow 2 add_addr_accepted 2
$ sudo ip netns exec s ip mptcp limits set subflow 2 add_addr_accepted 2
$ sudo ip netns exec c ip mptcp endpoint add <c-second-ip> dev c2 subflow
```

### Step 3 — Start an MPTCP transfer and confirm two subflows

```bash
$ sudo ip netns exec s mptcpize run iperf3 -s &
$ sudo ip netns exec c mptcpize run iperf3 -c <s-first-ip> -t 20 &
$ sudo ip netns exec c ss -tiM       # shows the connection with two subflows
```

### Step 4 — Kill one path mid-transfer

```bash
$ sudo ip netns exec c ip link set c1 down     # drop the primary path
# The transfer continues over c2 instead of failing — MPTCP failed over.
```

### Step 5 — Clean up

```bash
$ sudo ip netns delete c s
```

---

## Further Reading

| Topic | Link |
|---|---|
| Multipath TCP | [Wikipedia — Multipath TCP](https://en.wikipedia.org/wiki/Multipath_TCP) |
| Linux MPTCP | [mptcp.dev](https://www.mptcp.dev/) |
| TCP | [Lesson 62 — TCP internals](lesson-62-tcp-internals) |
| `ip-mptcp` | [man7.org — ip-mptcp(8)](https://man7.org/linux/man-pages/man8/ip-mptcp.8.html) |

---

## Checkpoint

**Q1. How does MPTCP keep a connection alive when one underlying path disappears?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An MPTCP connection isn't tied to a single path/5-tuple — it's split into multiple <strong>subflows</strong>,
each a real TCP connection over a different interface/address, all carrying data for one logical
connection (reassembled via a connection-level sequence number). If the active path fails, the data
that wasn't acknowledged is simply sent over a surviving subflow, and the application — which sees only
one ordinary stream socket — never observes a broken connection. So a phone moving from WiFi to
cellular continues its download over the LTE subflow rather than resetting, because the connection's
identity lives above any one path.
</details>

---

**Q2. What are MPTCP's two main use cases, and how do they differ?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Failover/resilience</strong>: keep one or more backup subflows so the connection survives a
path going down or degrading (mobility, redundancy) — the goal is <em>never drop the connection</em>,
not necessarily more speed. <strong>Aggregation/bandwidth</strong>: actively use multiple paths at once
to get throughput greater than any single path (e.g. WiFi + LTE combined). They differ in intent: one
optimizes for continuity (a standby path that activates on failure), the other for capacity (striping
data across paths simultaneously). Aggregation is trickier because the path manager and congestion
control must remain fair to ordinary single-path flows sharing a bottleneck.
</details>

---

**Q3. Why must MPTCP be able to fall back to plain TCP, and how is that possible?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because MPTCP rides in <strong>TCP options</strong> (MP_CAPABLE, MP_JOIN, ADD_ADDR), and real-world
middleboxes — NATs, firewalls, proxies — may strip options they don't understand or rewrite the stream.
If MPTCP couldn't cope with that, connections through such paths would simply break, making it
undeployable on the internet. It's possible to fall back because MPTCP begins as a normal TCP handshake
that merely <em>offers</em> MP_CAPABLE; if the option doesn't survive end-to-end (the peer doesn't echo
it, or it's stripped), both ends just proceed as a standard single-path TCP connection. The multipath
capability is purely additive, so absence of support degrades gracefully to ordinary TCP rather than
failing.
</details>

---

## Homework

Set up MPTCP in **aggregation** mode (both paths active, with `tc` shaping each veth to, say, 10 Mbit)
and measure aggregate `iperf3` throughput across the two subflows versus a single-path TCP run. Then
shape one path to be much lossier than the other and observe how the data splits. Explain what the MPTCP
scheduler is trying to balance.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With both subflows active over two 10 Mbit paths, aggregate throughput should approach ~20 Mbit —
noticeably more than a single-path run capped near 10 Mbit — demonstrating bandwidth aggregation. When
you make one path lossy/slow, you'll see the scheduler send <strong>proportionally more data over the
good path</strong> and back off on the bad one, so total throughput is dominated by the healthy subflow
rather than being dragged down to the bad path's rate. What the MPTCP scheduler balances: it must place
each segment on the subflow that minimizes overall completion time (favoring lower-RTT, lower-loss,
higher-cwnd paths) while keeping data reassemblable in order at the receiver, <em>and</em> remain fair —
its coupled congestion control ensures the combined subflows don't grab more than their share at a
shared bottleneck (so two MPTCP subflows over one link don't beat a single competing TCP flow). In
short: maximize useful aggregate throughput, avoid head-of-line stalls from a slow path, and stay fair
to other traffic.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 66 — DNS Deep Dive →](lesson-66-dns){: .btn .btn-primary }
