---
title: "Lesson 21 — BGP with FRR"
nav_order: 21
parent: "Phase 6: Dynamic Routing"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 21: BGP with FRR

## Concept

OSPF routes *within* one organization. **BGP** (Border Gateway Protocol) routes *between* organizations — it is the protocol that glues the entire internet together. Each independently-administered network is an **Autonomous System** (AS) with a number, and BGP exchanges reachability ("I can reach these prefixes, and here's the path") between ASes.

```
        AS 65001                          AS 65002
   ┌──────────────────┐   eBGP peering  ┌──────────────────┐
   │  router-a        │═════════════════│  router-b         │
   │  advertises      │                 │  advertises        │
   │  198.51.100.0/24 │                 │  203.0.113.0/24    │
   └──────────────────┘                 └──────────────────┘
   "To reach 203.0.113.0/24, the AS_PATH is: 65002"
```

Where OSPF is **link-state** (everyone shares the full map), BGP is **path-vector**: each route carries the explicit list of ASes it traversed (the **AS_PATH**), which is how BGP detects and avoids loops at global scale without anyone holding a full map.

---

## How it works

- **AS numbers** identify each network. Public ASNs are assigned by registries; private ones (64512–65534, plus the 32-bit range) are for labs and internal use.
- **eBGP vs iBGP:**
  - **eBGP** (external) — peering *between* different ASes. This is the typical "two companies connect" case.
  - **iBGP** (internal) — peering *within* one AS, used to carry external routes across a large network.
- **Prefix advertisement** — a router announces the networks it can reach with the `network` statement (or by redistribution).
- **Path attributes** — metadata attached to each route that drives best-path selection:

| Attribute | Meaning |
|---|---|
| `AS_PATH` | The sequence of ASes the route passed through. Shorter is preferred; also used for loop detection (a router rejects routes already containing its own ASN). |
| `LOCAL_PREF` | Local preference within an AS — higher wins. The primary knob for choosing your *outbound* path. |
| `MED` (Multi-Exit Discriminator) | A hint to a neighbor AS about which entry point to prefer — lower wins. |
| `NEXT_HOP` | The IP to forward to for this prefix. |

BGP's best-path algorithm checks these in a defined order (LOCAL_PREF, then AS_PATH length, then MED, etc.) to pick one winner per prefix.

{: .note }
> **Why the internet needs path-vector, not link-state**
> OSPF requires every router to hold the full topology — fine for one organization, impossible for the entire internet (hundreds of thousands of networks, all distrustful of each other). BGP scales by sharing only *reachability plus the AS_PATH*, not the internal topology of each AS. The AS_PATH gives loop-freedom (reject anything containing your own ASN) and lets each AS apply its own *policy* (who to prefer, who to avoid) — something link-state's "shortest path wins" can't express. The internet is a policy network, and BGP is a policy protocol.

---

## New Commands This Lesson

| vtysh command | What it does |
|---|---|
| `router bgp <asn>` | Enter BGP config for your AS |
| `neighbor <ip> remote-as <asn>` | Define a BGP peer and its AS |
| `network <prefix>` | Advertise a prefix into BGP |
| `show ip bgp` | Show the BGP table (all known prefixes + AS_PATH) |
| `show ip bgp summary` | Show peers and session state |
| `show ip bgp neighbors <ip>` | Detailed peer info |

---

## Lab

Two namespaces, each a router in its own AS, peering via eBGP. Each advertises a prefix; we verify the other learns it with the correct AS_PATH.

### Step 1 — Two router namespaces on a shared link

```bash
$ sudo ip netns add asA
$ sudo ip netns add asB
$ sudo ip link add a-b type veth peer name b-a
$ sudo ip link set a-b netns asA
$ sudo ip link set b-a netns asB

$ sudo ip netns exec asA ip link set a-b up
$ sudo ip netns exec asA ip addr add 10.0.0.1/30 dev a-b
$ sudo ip netns exec asB ip link set b-a up
$ sudo ip netns exec asB ip addr add 10.0.0.2/30 dev b-a

# Give each a "local network" to advertise (on dummy interfaces)
$ sudo ip netns exec asA ip link add lan type dummy
$ sudo ip netns exec asA ip link set lan up
$ sudo ip netns exec asA ip addr add 198.51.100.1/24 dev lan
$ sudo ip netns exec asB ip link add lan type dummy
$ sudo ip netns exec asB ip link set lan up
$ sudo ip netns exec asB ip addr add 203.0.113.1/24 dev lan
```

### Step 2 — Run zebra + bgpd in each namespace

Launch FRR's `zebra` and `bgpd` inside each router namespace (as in Lesson 20), then configure via vtysh.

### Step 3 — Configure eBGP on AS 65001 (router-a)

```
configure terminal
router bgp 65001
  neighbor 10.0.0.2 remote-as 65002     # peer is in a DIFFERENT AS → eBGP
  network 198.51.100.0/24                # advertise my prefix
```

### Step 4 — Configure eBGP on AS 65002 (router-b)

```
router bgp 65002
  neighbor 10.0.0.1 remote-as 65001
  network 203.0.113.0/24
```

### Step 5 — Verify the session came up

```
asA# show ip bgp summary
Neighbor        V   AS   MsgRcvd MsgSent  State/PfxRcd
10.0.0.2        4 65002      10      10             1
#                                              ^^ 1 prefix received = session UP
```

A numeric `State/PfxRcd` (not "Active"/"Idle") means the session is established and that many prefixes were received.

### Step 6 — Inspect the learned route and its AS_PATH

```
asA# show ip bgp
   Network          Next Hop      Path
*> 198.51.100.0/24  0.0.0.0       (locally originated)
*> 203.0.113.0/24   10.0.0.2      65002 i
#                                  ^^^^^ AS_PATH: learned from AS 65002
```

router-a learned `203.0.113.0/24` with AS_PATH `65002` — exactly one AS away. The `*>` marks it as the best, selected route. Confirm it reached the kernel:

```bash
$ sudo ip netns exec asA ip route show | grep 203.0.113
203.0.113.0/24 via 10.0.0.2 dev a-b proto bgp ...
```

### Step 7 — Prove end-to-end reachability across ASes

```bash
$ sudo ip netns exec asA ping -c 1 -I 198.51.100.1 203.0.113.1
64 bytes from 203.0.113.1 ...
```

AS 65001's network can reach AS 65002's network, using a route learned entirely via BGP.

### Step 8 — Clean up

```bash
$ sudo ip netns delete asA
$ sudo ip netns delete asB
```

---

## Further Reading

| Topic | Link |
|---|---|
| BGP | [Wikipedia — Border Gateway Protocol](https://en.wikipedia.org/wiki/Border_Gateway_Protocol) |
| Autonomous system | [Wikipedia — Autonomous system (Internet)](https://en.wikipedia.org/wiki/Autonomous_system_(Internet)) |
| Path-vector routing | [Wikipedia — Path-vector routing protocol](https://en.wikipedia.org/wiki/Path-vector_routing_protocol) |
| FRR BGP docs | [docs.frrouting.org — BGP](https://docs.frrouting.org/en/latest/bgp.html) |

---

## Checkpoint

**Q1. What is the fundamental difference between OSPF (link-state) and BGP (path-vector)?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**OSPF (link-state):** every router learns the *entire topology* of the area (all routers and links) by flooding LSAs, then independently computes shortest paths with Dijkstra. It optimizes for a single global metric (cost) and assumes all participants trust each other and want the mathematically shortest path. This works *within* one administrative domain.

**BGP (path-vector):** routers exchange *reachability plus the explicit AS_PATH* (the list of autonomous systems a route traversed), not full topology. Each AS makes *policy*-based decisions about which routes to prefer, accept, or reject — there's no single shared metric and no requirement to share internal structure. The AS_PATH provides loop detection (reject routes already containing your ASN) and scalability across hundreds of thousands of mutually-distrustful networks.

In short: OSPF shares topology and computes shortest paths inside one org; BGP shares paths and applies policy between orgs. OSPF answers "what's the fastest route here?"; BGP answers "whose routes do I accept and prefer, and how do I avoid loops at internet scale?"
</details>

---

**Q2. What is the difference between eBGP and iBGP?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**eBGP (external BGP)** is a peering session between routers in *different* autonomous systems — for example, your network connecting to your ISP, or two ISPs interconnecting. It's how reachability crosses AS boundaries, and it's where most BGP policy is applied.

**iBGP (internal BGP)** is a peering session between routers *within the same* AS. Its job is to carry externally-learned (eBGP) routes across a large internal network so every border/edge router in the AS knows about external prefixes consistently.

Key behavioral differences: eBGP prepends the local ASN to the AS_PATH (so paths grow as they cross ASes) and decrements/handles next-hop differently; iBGP does *not* prepend the ASN (you're staying in the same AS) and, by default, doesn't re-advertise iBGP-learned routes to other iBGP peers (the split-horizon rule), which is why large ASes need a full iBGP mesh or route reflectors. Simplified: eBGP = between ASes (the internet's seams), iBGP = within one AS (distributing external knowledge internally).
</details>

---

**Q3. router-a in AS 65001 learns `203.0.113.0/24` with AS_PATH `65002`. A few weeks later it sees the same prefix with AS_PATH `65003 65004 65002`. Which does BGP prefer and why? What does the AS_PATH also protect against?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
BGP prefers the route with AS_PATH `65002` — the **shorter** AS_PATH. After LOCAL_PREF (which isn't set here, so it's equal), the next tiebreaker in BGP's best-path algorithm is AS_PATH length, and shorter wins because it represents fewer AS hops to the destination. The path `65003 65004 65002` traverses three ASes versus one, so it's less preferred and would only be used if the direct one disappeared.

The AS_PATH also protects against **routing loops**. Because every AS prepends its own number as a route propagates, a router can detect a loop by checking whether its *own* ASN already appears in the AS_PATH of a received route — if it does, the route looped back, and the router rejects it. This is path-vector's loop-prevention mechanism, and it's what lets BGP stay loop-free across the whole internet without any router needing a global topology map (the way link-state protocols do).
</details>

---

## Homework

Build a **three-AS** chain: AS 65001 (router-a) — AS 65002 (router-b) — AS 65003 (router-c), where only adjacent ASes peer (a↔b and b↔c, but **not** a↔c). Advertise a prefix from AS 65003 and confirm AS 65001 learns it. Examine the AS_PATH at router-a. Then use `LOCAL_PREF` or a second path to influence which way traffic flows, and explain how policy attributes let an operator override "shortest AS_PATH."

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Reachability and AS_PATH:** AS 65003 advertises, say, `203.0.113.0/24`. It reaches AS 65002 with AS_PATH `65003`, and AS 65002 re-advertises it to AS 65001 prepending its own ASN, so router-a sees `203.0.113.0/24` with AS_PATH **`65002 65003`** — read right-to-left, the route originated in 65003 and passed through 65002. router-a installs it via router-b (its only path), and end-to-end ping works even though 65001 and 65003 never peer directly. This shows BGP carrying reachability transitively across ASes, with each hop recorded in the path.

**Policy override:** suppose you add a second, longer path to the same prefix (e.g., via an extra AS). By default BGP would pick the shorter AS_PATH. But if you set a higher **LOCAL_PREF** on the *longer* path at router-a (LOCAL_PREF is checked *before* AS_PATH length in the best-path algorithm), router-a will prefer that longer path despite the extra AS hop. This is the essence of BGP being a *policy* protocol: operators routinely steer traffic for cost, contractual, or performance reasons by setting attributes like LOCAL_PREF (for outbound preference), AS_PATH prepending (to make a path look longer/less attractive to others), and MED (to hint inbound preference to a neighbor). Unlike OSPF, where the shortest metric always wins, BGP lets human policy override the "obvious" shortest path — because on the internet, the best path is often a business decision, not just a topological one.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 22 — NAT with nftables →](lesson-22-nat){: .btn .btn-primary }
