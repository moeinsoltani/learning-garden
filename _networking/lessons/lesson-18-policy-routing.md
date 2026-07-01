---
title: "Lesson 18 — Policy Routing & Multiple Tables"
nav_order: 18
parent: "Phase 5: Routing"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 18: Policy Routing & Multiple Routing Tables

## Concept

Ordinary routing decides where a packet goes based *only* on its **destination**. **Policy routing** lets you decide based on other things too — the **source** address, an interface, or a firewall **mark**. Two packets headed to the same destination can take completely different paths depending on where they came from.

The mechanism: there isn't just one routing table. There can be many (numbered 0–255), and a set of **rules** decides *which table* to consult for a given packet.

```
   packet arrives
        │
        ▼
   ┌─────────────────── ip rule list (checked in priority order) ───────────────┐
   │ prio 100:  from 192.168.1.0/24  → look up table 100                         │
   │ prio 200:  fwmark 0x1           → look up table 200                         │
   │ prio 32766: from all            → look up table main (the normal table)     │
   └────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
   chosen table's routes decide the next-hop
```

---

## How it works

The kernel keeps several built-in tables and lets you add your own:

| Table | Number | Purpose |
|---|---|---|
| `local` | 255 | Addresses the host owns (auto-managed; highest priority). |
| `main` | 254 | The normal table you see with `ip route show`. |
| `default` | 253 | Usually empty; last-resort. |
| custom | 1–252 | Yours to create for policy routing. |

A **rule** (`ip rule`) is a match-and-select: "if the packet matches X, use table Y." Rules are evaluated in **priority order** (lower number = checked first), and the first matching rule wins. The default ruleset already contains three rules pointing at `local`, `main`, and `default`.

You build policy routing by: (1) putting some routes in a custom table, and (2) adding a rule that sends the packets you care about to that table.

{: .note }
> **fwmark — routing by firewall decision**
> A *mark* is an integer the firewall (nftables/iptables) can stamp onto a packet's in-kernel metadata (`meta mark set 0x1`). It doesn't change the packet on the wire — it's an internal tag. `ip rule add fwmark 0x1 table 200` then routes marked packets via table 200. This is the glue between firewalling and routing: you classify packets with the firewall's rich matching, then route them by mark. It's how VPN split-tunneling and multi-WAN setups are built.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip rule list` | Show all routing policy rules in priority order |
| `ip rule add from <src> table <id> priority <n>` | Route by source address via a custom table |
| `ip rule add fwmark <mark> table <id>` | Route packets with a firewall mark via a table |
| `ip rule del <spec>` | Remove a rule |
| `ip route add <route> table <id>` | Add a route to a specific table |
| `ip route show table <id>` | Show one table's routes |
| `ip route get <dst> from <src>` | Resolve a route as if from a given source |

---

## Lab

We'll make traffic from two different source addresses on the same host take two different gateways to the same destination.

### Step 1 — Build a host with two uplinks

```bash
$ sudo ip netns add host
$ sudo ip netns add gwA
$ sudo ip netns add gwB

# host <-> gwA
$ sudo ip link add h-a type veth peer name a-h
$ sudo ip link set h-a netns host
$ sudo ip link set a-h netns gwA
# host <-> gwB
$ sudo ip link add h-b type veth peer name b-h
$ sudo ip link set h-b netns host
$ sudo ip link set b-h netns gwB

$ sudo ip netns exec host ip link set h-a up
$ sudo ip netns exec host ip addr add 10.0.10.1/24 dev h-a
$ sudo ip netns exec host ip link set h-b up
$ sudo ip netns exec host ip addr add 10.0.20.1/24 dev h-b

$ sudo ip netns exec gwA ip link set a-h up
$ sudo ip netns exec gwA ip addr add 10.0.10.254/24 dev a-h
$ sudo ip netns exec gwB ip link set b-h up
$ sudo ip netns exec gwB ip addr add 10.0.20.254/24 dev b-h
```

The host has two source addresses: `10.0.10.1` (toward gwA) and `10.0.20.1` (toward gwB).

### Step 2 — See the default rules

```bash
$ sudo ip netns exec host ip rule list
0:      from all lookup local
32766:  from all lookup main
32767:  from all lookup default
```

These three are always present. We'll insert higher-priority (lower-number) rules.

### Step 3 — Create two custom tables, one per uplink

```bash
# Table 100: default route via gwA
$ sudo ip netns exec host ip route add default via 10.0.10.254 table 100
# Table 200: default route via gwB
$ sudo ip netns exec host ip route add default via 10.0.20.254 table 200
```

### Step 4 — Add source-based rules

```bash
# Packets sourced from 10.0.10.1 → table 100 (via gwA)
$ sudo ip netns exec host ip rule add from 10.0.10.1 table 100 priority 100
# Packets sourced from 10.0.20.1 → table 200 (via gwB)
$ sudo ip netns exec host ip rule add from 10.0.20.1 table 200 priority 200

$ sudo ip netns exec host ip rule list
0:      from all lookup local
100:    from 10.0.10.1 lookup 100
200:    from 10.0.20.1 lookup 200
32766:  from all lookup main
...
```

### Step 5 — Prove the same destination takes different paths by source

```bash
# Same destination 8.8.8.8, but different source → different gateway
$ sudo ip netns exec host ip route get 8.8.8.8 from 10.0.10.1
8.8.8.8 from 10.0.10.1 via 10.0.10.254 dev h-a ...      # via gwA

$ sudo ip netns exec host ip route get 8.8.8.8 from 10.0.20.1
8.8.8.8 from 10.0.20.1 via 10.0.20.254 dev h-b ...      # via gwB
```

Identical destination, two different next-hops — chosen purely by source address. That's policy routing.

### Step 6 — (Optional) Route by firewall mark

```bash
# A rule that sends marked packets to table 200
$ sudo ip netns exec host ip rule add fwmark 0x1 table 200 priority 50

# In nftables you would mark packets, e.g.:
#   nft add rule inet mangle output ip daddr 1.1.1.1 meta mark set 0x1
# Then any packet to 1.1.1.1 (regardless of source) routes via gwB.
$ sudo ip netns exec host ip rule list | grep fwmark
50:     from all fwmark 0x1 lookup 200
```

Now routing depends on a firewall decision, not just addresses — the basis of VPN split-tunneling.

### Step 7 — Clean up

```bash
$ sudo ip netns delete host
$ sudo ip netns delete gwA
$ sudo ip netns delete gwB
```

---

## Further Reading

| Topic | Link |
|---|---|
| Policy-based routing | [Wikipedia — Policy-based routing](https://en.wikipedia.org/wiki/Policy-based_routing) |
| `ip-rule` | [man7.org — ip-rule(8)](https://man7.org/linux/man-pages/man8/ip-rule.8.html) |
| `ip-route` tables | [man7.org — ip-route(8)](https://man7.org/linux/man-pages/man8/ip-route.8.html) |
| Multi-homing | [Wikipedia — Multihoming](https://en.wikipedia.org/wiki/Multihoming) |

---

## Checkpoint

**Q1. Why would you want traffic from one source IP to take a different path than traffic from another source IP, even when both are going to the same destination?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Several real reasons:

- **Multi-homed / multi-WAN hosts:** a server with two uplinks (two ISPs) should send replies out the *same* uplink the request came in on. If a packet arrived on the ISP-A address, its reply must be sourced from that address and routed back through ISP-A's gateway — otherwise return traffic is asymmetric and may be dropped by reverse-path filtering or ISP egress filtering. Source-based policy routing enforces "reply via the link you came in on."
- **VPN split tunneling:** traffic from certain source apps/addresses goes through the VPN gateway while everything else uses the normal default route.
- **Bandwidth/cost separation:** route a particular subnet's or service's traffic over a cheaper or higher-capacity link.

Destination-only routing can't express any of these because the deciding factor isn't *where the packet is going* but *where it came from* (or how it was classified). Policy routing adds source/mark/interface as inputs to the decision.
</details>

---

**Q2. What is the relationship between `ip rule` and routing tables? Which is consulted first?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`ip rule` defines **which routing table** to consult for a given packet; the table then provides the actual routes. The kernel evaluates rules in **priority order** (lowest number first) and uses the first rule that matches the packet, looking up the route in that rule's table. So rules are consulted *before* tables — a rule selects the table, and the table selects the route. The default ruleset has `local` (priority 0), `main` (32766), and `default` (32767); you insert custom rules at lower priorities to override them for specific traffic. If a chosen table has no matching route, the kernel continues to the next matching rule (depending on rule type), eventually falling through to `main`.
</details>

---

**Q3. What does a firewall "mark" (fwmark) actually do to a packet, and why is it useful for routing?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A mark is an integer attached to the packet's *in-kernel metadata* — it is **not** written into the packet on the wire and never leaves the host; it's purely an internal tag. The firewall (nftables/iptables) sets it based on whatever rich criteria it can match (port, payload, connection state, DSCP, etc.). Then `ip rule add fwmark <value> table <id>` routes packets carrying that mark via a chosen table. This is powerful because it lets you separate *classification* (done by the firewall, which can match almost anything) from *routing* (done by policy rules). You can route by criteria far beyond what plain source/destination rules allow — e.g., "send all traffic the firewall identified as belonging to the VPN profile out the VPN gateway." It's the standard mechanism behind split tunneling and complex multi-path setups.
</details>

---

## Homework

Set up a host with two uplinks (gwA, gwB) as in the lab. Configure source-based policy routing so that source `10.0.10.1` uses gwA and `10.0.20.1` uses gwB. Then deliberately create the asymmetric-routing problem: make a packet *arrive* via gwA but force its *reply* out gwB (e.g., by removing the source rule). Observe with tcpdump on both gateways what happens, and explain why production multi-homed servers must use source-based policy routing to avoid this.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With correct source rules, a request arriving on the gwA link is replied to from `10.0.10.1` and routed back out gwA — symmetric, clean. When you remove the `from 10.0.10.1 table 100` rule, the reply falls through to the `main` table, which has only one default (say via gwA) — or, if you also skew main toward gwB, the reply for a connection that arrived via gwA now leaves via gwB.

tcpdump shows the asymmetry: the *request* appears on gwA's interface, but the *reply* appears on gwB's interface instead of gwA's. This breaks in the real world for two reasons:

1. **Reverse-path filtering (rp_filter):** a router/host configured with strict rp_filter drops packets whose source address wouldn't route back out the interface they arrived on. The asymmetric reply path can cause the return traffic to be discarded (you'll study rp_filter in Lesson 34).
2. **ISP egress / stateful middleboxes:** ISPs often drop packets whose source address doesn't belong to that link (BCP 38 anti-spoofing), and stateful firewalls/NAT on the gwB path never saw the inbound half of the connection, so they drop the reply as invalid.

The fix — and the reason multi-homed servers *require* source-based policy routing — is to bind each connection's replies to the same uplink the request came in on: a rule per source address, each pointing at a table whose default route uses the matching gateway. That guarantees symmetric paths and keeps stateful devices and anti-spoofing filters happy.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 19 — FRR Introduction →](lesson-19-frr-intro){: .btn .btn-primary }
