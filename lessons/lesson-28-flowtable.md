---
title: "Lesson 28 — Flowtable Offload"
nav_order: 28
parent: "Phase 8: nftables Firewall"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 28: nftables Flowtable (Connection Offload)

## Concept

Every packet of every connection normally climbs the full netfilter stack — routing, conntrack lookup, all your firewall chains — on its way through a router. For the *first* packets of a connection that's fine, but for a long-running flow (a big download, a video stream) re-evaluating the entire ruleset on every single packet is wasted work. A **flowtable** lets the kernel recognize an already-established, already-approved flow and shove its packets onto a **fast path** that skips most of that processing.

```
   Normal (slow) path — every packet:
     NIC → prerouting → conntrack → routing → forward chain → postrouting → NIC

   Flowtable (fast) path — after the flow is offloaded:
     NIC → [flowtable lookup: known flow!] → straight out the other NIC
              (skips the firewall chains and full routing for this packet)
```

The first packets take the slow path (so the firewall *does* inspect and approve the connection); once established, the flow is added to the flowtable and subsequent packets are forwarded almost directly.

---

## How it works

You declare a **flowtable** bound to a hook (ingress) and a set of devices, then add a rule that *offloads* established flows into it:

```
table inet filter {
    flowtable f {
        hook ingress priority 0
        devices = { eth0, eth1 }
    }
    chain forward {
        type filter hook forward priority 0; policy drop;
        ct state established,related accept
        ct state established flow add @f      # offload established flows to the fast path
        ...
    }
}
```

Once a flow is in the flowtable, its packets are matched at the **ingress** hook (very early, before the normal forward processing) and forwarded directly between the two devices, bypassing the conntrack re-evaluation and the forward chain. The connection is still *tracked* — `conntrack -L` still shows it — but per-packet rule evaluation is skipped.

There's a hardware angle too: on supported NICs, the flowtable can offload flows **to the NIC itself** (`flags offload`), so packets are forwarded in hardware and never even reach the CPU. That's how Linux software routers reach very high throughput.

{: .note }
> **The tradeoff: what stops working on the fast path**
> Offloading skips most of the stack, so anything that needs to see every packet no longer applies to offloaded flows. That means: per-packet **filtering changes** mid-flow won't affect an already-offloaded connection, **traffic shaping/QoS (tc qdiscs)** and **accounting/counters** in the forward chain won't see those packets, and features like **NAT helpers or deep inspection** can't act on them. Connection *tracking* still works (the entry exists and times out normally), and the slow path still handles connection setup, RELATED traffic, and teardown. The rule of thumb: offload only flows you've fully decided to allow and don't need to police or shape per-packet. You trade per-packet control for speed.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `flowtable f { hook ingress priority 0 ; devices = { eth0, eth1 } ; }` | Declare a flowtable on two devices |
| `ct state established flow add @f` | Offload established flows into the flowtable |
| `flowtable f { ... flags offload ; ... }` | Enable hardware offload (NIC-dependent) |
| `nft list flowtables` | Show configured flowtables |
| `nft list ruleset` | See the flowtable + offload rule together |
| `conntrack -L` | Confirm offloaded flows are still tracked |

---

## Lab

We'll set up a forwarding router between two namespaces, add a flowtable, and confirm that established flows get offloaded while still being tracked.

### Step 1 — Router between two namespaces

```bash
$ sudo ip netns add ns1
$ sudo ip netns add router
$ sudo ip netns add ns2
# Wire ns1<->router (10.0.1.0/24) and router<->ns2 (10.0.2.0/24) as in Lesson 17,
# enable forwarding on the router, and add default routes on ns1/ns2.
$ sudo ip netns exec router sysctl -w net.ipv4.ip_forward=1
```

(Use the Lesson 17 script: `vethA`/`vethA-r` for ns1↔router, `vethB-r`/`vethB` for router↔ns2, addresses `10.0.1.x` / `10.0.2.x`.)

### Step 2 — Baseline forwarding works

```bash
$ sudo ip netns exec ns1 ping -c 1 10.0.2.2     # works (slow path)
```

### Step 3 — Install a flowtable and offload rule on the router

```bash
$ sudo ip netns exec router nft -f - <<'EOF'
table inet filter {
    flowtable f {
        hook ingress priority 0
        devices = { vethA-r, vethB-r }
    }
    chain forward {
        type filter hook forward priority 0; policy accept;
        ct state established,related accept
        ip protocol { tcp, udp } flow add @f
    }
}
EOF
$ sudo ip netns exec router nft list flowtables
table inet filter {
    flowtable f { hook ingress priority filter; devices = { vethA-r, vethB-r } }
}
```

### Step 4 — Generate a sustained flow and observe offload

```bash
# A long-lived TCP flow (e.g. iperf3 or a big nc transfer) ns1 -> ns2
$ sudo ip netns exec ns2 iperf3 -s &
$ sudo ip netns exec ns1 iperf3 -c 10.0.2.2 -t 20 &

# While it runs, confirm the flow is OFFLOADED but still TRACKED:
$ sudo ip netns exec router conntrack -L -p tcp
tcp 6 ESTABLISHED ... [OFFLOAD] ...
#                       ^^^^^^^^
#   the [OFFLOAD] flag means this flow is on the flowtable fast path
```

The `[OFFLOAD]` marker in conntrack confirms the flow's packets are now taking the fast path through the ingress flowtable rather than the full forward chain — yet the connection is still tracked (the entry exists, with state ESTABLISHED).

### Step 5 — Demonstrate the tradeoff (counters miss offloaded packets)

```bash
# Add a counter to the forward chain
$ sudo ip netns exec router nft add rule inet filter forward counter
# Run another sustained transfer, then check the counter:
$ sudo ip netns exec router nft list chain inet filter forward | grep counter
# The counter sees the FIRST few packets (slow path, before offload) but NOT the
# bulk of the offloaded flow — because offloaded packets skip the forward chain.
```

This is the tradeoff made visible: per-packet processing in the forward chain (counting, here; shaping/QoS in general) doesn't apply to offloaded packets.

### Step 6 — Clean up

```bash
$ sudo kill %1 %2 2>/dev/null
$ sudo ip netns delete ns1 router ns2
```

---

## Further Reading

| Topic | Link |
|---|---|
| nftables flowtables | [wiki.nftables.org — Flowtables](https://wiki.nftables.org/wiki-nftables/index.php/Flowtables) |
| Netfilter flow offload | [kernel.org — netfilter](https://docs.kernel.org/networking/nf_flowtable.html) |
| Connection tracking | [Wikipedia — Stateful firewall](https://en.wikipedia.org/wiki/Stateful_firewall) |
| Fast path / slow path | [Wikipedia — Fast path](https://en.wikipedia.org/wiki/Fast_path) |

---

## Checkpoint

**Q1. What is the tradeoff of using a flowtable? What stops working for offloaded flows?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The tradeoff is **speed in exchange for per-packet control**. A flowtable forwards established flows on a fast path at the ingress hook, skipping the normal routing and forward-chain processing — which is exactly why it's fast, but also why anything that needs to inspect or act on *every packet* no longer applies to those offloaded flows:

- **Per-packet filtering** beyond the initial connection setup: rule changes mid-flow won't affect an already-offloaded connection, and the forward chain doesn't see its packets.
- **Traffic shaping / QoS** (tc qdiscs) and **accounting / counters** in the forward path miss the offloaded packets.
- **Deep packet inspection, NAT helpers**, and other features that require seeing the full packet stream can't operate on the fast path.

What *still* works: the connection is still **tracked** (conntrack entry exists, ages out normally, shows `[OFFLOAD]`), and the **slow path still handles connection setup, RELATED connections, and teardown** — only the steady-state established packets are offloaded. The guidance: only offload flows you've already fully decided to permit and don't need to shape or inspect per-packet.
</details>

---

**Q2. The first packets of a connection take the slow path even with a flowtable configured. Why is that important rather than a flaw?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It's essential to security and correctness. The flowtable only offloads flows once they're **established** — meaning the connection's initial packets (the SYN, the handshake) first traverse the *full* netfilter stack, where your firewall rules actually inspect and **decide whether to allow the connection at all**. If the very first packet were offloaded, the firewall would never get a chance to evaluate the new connection, and unwanted connections could slip straight through on the fast path. By design, setup goes through the slow path so policy is enforced at connection birth; only *after* the connection has been approved and established does its bulk traffic get offloaded. So the slow-path-for-setup behavior isn't a limitation — it's what lets you have both a real firewall (full inspection of new connections) and high throughput (fast path for the already-vetted bulk data). It cleanly separates "should this connection exist?" (decided once, on the slow path) from "forward these packets fast" (the offloaded steady state).
</details>

---

**Q3. After offloading, `conntrack -L` still shows the connection (with an `[OFFLOAD]` flag). What does that tell you about the relationship between connection tracking and the flowtable fast path?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It tells you that **offloading does not remove the connection from conntrack — it just changes how the connection's packets are forwarded.** The conntrack entry still exists, still has its state (ESTABLISHED), and still ages out by its normal timeouts; the `[OFFLOAD]` flag simply marks that this flow's steady-state packets are being handled on the flowtable fast path instead of climbing the full stack. So the two systems are complementary: conntrack remains the *authority on which flows exist and their state*, while the flowtable is an *optimization for forwarding the packets of flows conntrack already knows are established*. This is why teardown still works (when FINs/RSTs or a timeout end the flow, the slow path/conntrack handles it and the flow leaves the flowtable), and why you can still *see* and account for connections at the flow level even though you can't process their individual packets per-rule anymore. The flowtable rides on top of conntrack; it doesn't replace it.
</details>

---

## Homework

Set up the forwarding router with a flowtable. Run a sustained transfer and confirm the `[OFFLOAD]` flag appears in conntrack. Then attempt to apply a `tc netem` delay (you'll learn `tc` properly in Phase 9, but try `tc qdisc add dev <iface> root netem delay 100ms`) on the router's egress interface and measure whether the offloaded flow is affected. Explain your observation in terms of where the flowtable fast path sits relative to the qdisc.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Observation:** Adding `netem delay 100ms` to the router's egress interface has **little or no effect on the already-offloaded flow's latency** (or affects it inconsistently), whereas a non-offloaded flow through the same interface would clearly show the added 100ms RTT.

**Why, in terms of path placement:** the flowtable fast path operates at the **ingress** hook and forwards packets very early — for offloaded flows, packets are shunted from the incoming device toward the outgoing device while bypassing much of the normal stack processing. The standard egress **qdisc** (where `tc netem` lives) sits later in the conventional egress path that offloaded packets are designed to skip. So packets riding the flowtable fast path don't get queued/delayed by the egress qdisc the way slow-path packets do. This is a concrete instance of the Lesson tradeoff: **traffic shaping/QoS doesn't reliably apply to offloaded flows** because offload's entire purpose is to skip the per-packet processing stages — including the qdisc — for established connections.

The practical lesson: if you need to shape, delay, count, or otherwise police a flow per-packet, **don't offload it** (exclude it from the `flow add @f` rule). Offload and per-packet traffic control are mutually exclusive for a given flow, because they live on opposite sides of the fast-path/slow-path split. (Exact behavior can vary by kernel version and whether software or hardware offload is in use, but the principle — fast path skips the qdisc — holds.)
</details>
