---
title: "Lesson 73 — Network Observability & Telemetry"
nav_order: 73
parent: "Phase 20: Modern Transport & Observability"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 73: Network Observability & Telemetry

## Concept

Troubleshooting (Phase 15) is *reactive* — you dig in when something breaks. **Observability** is
*proactive*: continuously collecting metrics so you can see trends, alert before failure, and have
data already in hand when an incident hits. The three classic pillars are **metrics** (numbers over
time), **logs** (events), and **flow telemetry** (who-talked-to-whom). This lesson is about building
that continuous view of a network.

```
   counters (/proc/net, ss, ip -s)  → metrics over time → dashboards & alerts
   flow records (NetFlow/IPFIX/sFlow) → who talked to whom, how much
   correlate across the stack to localize "where did it go wrong?"
```

---

## How it works

**Counters everywhere.** The kernel exposes running counters you sample over time:
- `ip -s link` — per-interface RX/TX packets, bytes, **errors, drops** (rising drops = a problem).
- `/proc/net/snmp`, `/proc/net/netstat` — protocol-level counters (TCP retransmits, listen-queue
  overflows, out-of-order, etc.) — gold for spotting *why* TCP is unhappy.
- `ss -s` / `ss -ti` — socket summary and per-socket TCP internals (cwnd, RTT, retransmits — Lesson 62).
- `tc -s qdisc` — per-qdisc drops/backlog (Lesson 29), to catch shaping/queue issues.
- `nstat`, `sar`/`mpstat` — convenient deltas of those counters.

The technique is always *rate of change*: a counter's absolute value is meaningless; its **derivative**
(drops/sec, retransmits/sec) is the signal.

**Flow telemetry.** Counters tell you *how much*; flow records tell you *who and what*. Routers/switches
export **NetFlow / IPFIX** (per-flow records: 5-tuple, bytes, packets, timestamps) or **sFlow**
(sampled packets). A collector aggregates these to answer "what's consuming the uplink?" or "who's
talking to that host?" — without full packet capture.

**Exporting metrics.** Tools like Prometheus **node_exporter** scrape the kernel counters above and
expose them as time series; you graph them (Grafana) and alert on thresholds. This turns ad-hoc
`ip -s link` checks into a continuous, queryable history.

{: .note }
> **Correlate across the stack to localize**
> The power move is combining pillars: interface drops (`ip -s link`) + TCP retransmits
> (`/proc/net/netstat`) + qdisc backlog (`tc -s qdisc`) + flow records together tell you *where* loss
> happens (NIC? queue? a specific flow saturating a link?). One metric is a hint; correlated metrics
> localize the fault — the same layer-by-layer discipline as Lesson 47, but from always-on data.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip -s link show <if>` | Interface counters incl. errors/drops |
| `nstat -az` | Snapshot/delta of all kernel net counters |
| `ss -s` | Socket summary (totals by state/protocol) |
| `tc -s qdisc show dev <if>` | Per-qdisc sent/dropped/backlog |
| `cat /proc/net/netstat` / `/proc/net/snmp` | Protocol counters (TCP retrans, drops…) |

---

## Lab

Build the reflex of watching counter *deltas* while inducing a problem, then correlating across layers.

### Step 1 — Baseline interface and TCP counters

```bash
$ sudo ip netns add c; sudo ip netns add s
# veth c<->s, addresses, iperf3 -s in s.
$ sudo ip netns exec c ip -s link show <c-if>      # note baseline RX/TX, drops=0
$ sudo ip netns exec c nstat -az | grep -i retrans # baseline TCP retransmits
```

### Step 2 — Induce loss + queueing, then watch the counters move

```bash
$ sudo ip netns exec c tc qdisc add dev <c-if> root netem loss 5% limit 100
$ sudo ip netns exec c iperf3 -c <s-ip> -t 10 &
# While it runs, sample repeatedly and watch the DELTAS:
$ watch -n1 'ip netns exec c tc -s qdisc show dev <c-if>; ip netns exec c nstat | grep -i retrans'
# tc shows 'dropped' climbing; nstat shows TcpRetransSegs climbing — the loss made visible two ways.
```

### Step 3 — Correlate

```bash
$ sudo ip netns exec c ss -ti dst <s-ip>     # cwnd small, retrans high → matches the qdisc drops
# Conclusion built from data: drops at the qdisc → TCP retransmits → reduced cwnd → low throughput.
```

### Step 4 — Clean up

```bash
$ sudo ip netns delete c s
```

---

## Further Reading

| Topic | Link |
|---|---|
| Observability | [Wikipedia — Observability (software)](https://en.wikipedia.org/wiki/Observability_(software)) |
| NetFlow | [Wikipedia — NetFlow](https://en.wikipedia.org/wiki/NetFlow) |
| sFlow | [Wikipedia — sFlow](https://en.wikipedia.org/wiki/SFlow) |
| Debugging methodology | [Lesson 47 — Network debugging methodology](lesson-47-debugging-methodology) |

---

## Checkpoint

**Q1. Why is the *rate of change* of a counter more useful than its absolute value?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Kernel counters are cumulative totals since boot/reset, so an absolute number like "1,200,000 packets"
or "5,000 drops" tells you nothing on its own — you don't know whether those drops happened years ago or
in the last second. The <strong>rate of change</strong> (the derivative: drops/sec, retransmits/sec,
bytes/sec) is what reveals current behavior: drops climbing <em>now</em> means a problem happening
<em>now</em>. That's why observability is built on sampling counters over time and graphing/altering on
their deltas, and why tools like <code>nstat</code> and node_exporter focus on differences between
samples rather than raw totals.
</details>

---

**Q2. What does flow telemetry (NetFlow/IPFIX/sFlow) give you that interface counters do not?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Interface counters tell you <strong>how much</strong> traffic an interface carried (and how many errors/
drops), but not <em>who</em> or <em>what</em>. <strong>Flow telemetry</strong> exports per-flow records —
the 5-tuple (src/dst IP and port, protocol), byte/packet counts, and timestamps (NetFlow/IPFIX), or
sampled packets (sFlow) — so you can answer questions like "which hosts/applications are consuming the
uplink," "who is talking to this server," or "what changed when traffic spiked." It gives the
<strong>composition</strong> of traffic and the conversations, without the cost and volume of full packet
capture. So counters quantify, flow records attribute.
</details>

---

**Q3. You see low throughput. Describe how correlating counters across layers localizes the cause.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You combine signals from different layers instead of trusting one: check <strong>interface counters</strong>
(<code>ip -s link</code>) for RX/TX errors and drops (NIC/driver/cabling issues); check <strong>qdisc
stats</strong> (<code>tc -s qdisc</code>) for drops/backlog (shaping or queue overflow); check
<strong>protocol counters</strong> (<code>/proc/net/netstat</code>, <code>nstat</code>) for TCP
retransmits, out-of-order, or listen-queue overflows; and check <strong>per-socket</strong> state
(<code>ss -ti</code>) for a small cwnd and high RTT. If, say, qdisc drops and TCP retransmits both climb
together and the socket shows a collapsed cwnd, you've localized it: the queue is dropping packets, TCP
is retransmitting and backing off, and that's why throughput is low — not a NIC error and not the
application. Each counter is a hint; together they pinpoint <em>which</em> layer is losing traffic, which
is the whole point of correlated, always-on observability (the data-driven version of Lesson 47's
layer-by-layer method).
</details>

---

## Homework

Set up `node_exporter` (or a simple script sampling `nstat`/`ip -s link` every second into a file) and
capture a few minutes of data spanning an `iperf3` run with netem loss applied partway through. Identify,
purely from the recorded metrics, the exact moment loss was introduced and the chain of effects.
Explain how having this data *before* an incident changes troubleshooting.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
In the recorded series you can pinpoint the moment netem was applied as the timestamp where
<strong>qdisc drops and TCP retransmits start rising together</strong>, immediately followed by a
<strong>drop in throughput</strong> (bytes/sec) and, in per-socket data, a <strong>shrinking cwnd and
rising RTT</strong>. The chain reads cleanly off the metrics: introduce loss → qdisc/interface drops
climb → TCP detects loss and retransmits → congestion control cuts cwnd → goodput falls. Having this data
<strong>before</strong> an incident transforms troubleshooting from reactive guesswork into evidence:
you can see <em>when</em> behavior changed (correlate with a deploy/config change), <em>what</em> changed
first (root vs. symptom — drops preceded the throughput dip, so the loss is causal), and <em>where</em>
in the stack it originated — without having to reproduce the problem live. Continuous telemetry also
enables alerting so you're notified as drops begin rather than after users complain, and it provides the
historical baseline that makes "is this normal?" answerable. In short, observability turns the
layer-by-layer method of Lesson 47 into something you can run on data you already have.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Back to the learning plan →](../learning-plan.html){: .btn .btn-primary }

*🎉 You've reached the end of this track.*
