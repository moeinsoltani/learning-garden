---
title: "Lesson 59 — BGP EVPN & VXLAN Fabrics"
nav_order: 59
parent: "Phase 17: Advanced Routing & Data-Center Fabrics"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 59: BGP EVPN & VXLAN Fabrics

## Concept

In Lesson 15 you built VXLAN with *flood-and-learn*: to find where a MAC lives, a VTEP floods
unknown frames to every other VTEP and learns from replies. That doesn't scale — flooding grows
with the fabric, and there's no single source of truth. **EVPN** (Ethernet VPN) fixes this by
giving VXLAN a **control plane**: BGP distributes "MAC address X (and its IP) lives behind VTEP Y"
as routes, so VTEPs *know* where everything is without flooding.

```
   Flood-and-learn VXLAN              EVPN-controlled VXLAN
   "who has MAC X?" → flood all       BGP advertises: MAC X @ VTEP-2, MAC Y @ VTEP-3
   learn from the reply              every VTEP has the full map, no flooding
```

This is the standard modern data-center fabric: a **leaf-spine** IP underlay carrying VXLAN
overlays, with EVPN as the brain.

---

## How it works

**The underlay: leaf-spine.** Servers attach to *leaf* switches; every leaf connects to every
*spine*; spines connect nothing but leaves. Result: any leaf is exactly two hops from any other,
with ECMP (Lesson 58) across all spines. The underlay runs plain BGP/OSPF carrying only VTEP
loopback reachability.

**The overlay: VXLAN.** Tenant L2 segments are stretched across the IP underlay with VXLAN
(Lesson 15), each VTEP being a leaf.

**The control plane: BGP EVPN.** EVPN is an address family in BGP that carries *MAC/IP information*
as routes. The important [EVPN route types](https://en.wikipedia.org/wiki/Ethernet_VPN):

| Type | Carries | Replaces |
|---|---|---|
| Type 2 | A host's MAC (and optionally IP) behind a VTEP | MAC flooding + ARP flooding |
| Type 3 | "I am a VTEP for VNI N" (inclusive multicast) | VTEP auto-discovery |
| Type 5 | An IP prefix behind a VTEP | Inter-subnet routing in the fabric |

**Auto-discovery.** Type-3 routes let VTEPs find each other automatically (no manual `bridge fdb
append` like Lesson 15). Type-2 routes mean a host's location is *advertised* the moment it appears,
so other VTEPs populate their forwarding tables from BGP rather than from flooding. ARP can even be
suppressed because the IP↔MAC binding is already in EVPN.

{: .note }
> **Why this is better at scale**
> Flood-and-learn wastes bandwidth (every unknown is flooded fabric-wide) and is slow to converge.
> EVPN turns L2 learning into a routing problem: BGP — which already scales to the whole internet —
> distributes endpoint locations explicitly, supports policy, and converges fast. FRR's `bgpd`
> implements EVPN, so you can build the whole thing on Linux.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `address-family l2vpn evpn` | Enable the EVPN address family in FRR `router bgp` |
| `advertise-all-vni` | Advertise all local VNIs into EVPN |
| `show bgp l2vpn evpn` | View EVPN routes (type 2/3/5) |
| `show evpn mac vni <id>` | MACs learned via EVPN for a VNI |
| `bridge fdb show dev vxlan0` | See the FDB now populated by the control plane |

---

## Lab

Build a two-VTEP EVPN fabric in namespaces (conceptually leaf1/leaf2 over a spine), and watch a
host's MAC appear as an EVPN Type-2 route instead of being flooded.

### Step 1 — Underlay (spine + two leaves) with BGP loopback reachability

```bash
$ for n in spine leaf1 leaf2; do sudo ip netns add $n; done
# wire leaf1--spine and leaf2--spine; give each a /32 loopback; run FRR eBGP underlay
# so leaf1 and leaf2 can reach each other's loopbacks (the VTEP IPs).
```

### Step 2 — VXLAN VTEP + bridge on each leaf

```bash
$ sudo ip netns exec leaf1 ip link add vxlan10 type vxlan id 10 dstport 4789 \
    local <leaf1-lo> nolearning
$ sudo ip netns exec leaf1 ip link add br10 type bridge
$ sudo ip netns exec leaf1 ip link set vxlan10 master br10
# (mirror on leaf2; 'nolearning' because EVPN, not flooding, populates the FDB)
```

### Step 3 — Turn on EVPN in BGP

```bash
$ sudo ip netns exec leaf1 vtysh -c 'configure terminal' \
    -c 'router bgp 65001' -c 'address-family l2vpn evpn' -c 'advertise-all-vni'
# same on leaf2
```

### Step 4 — Verify control-plane learning

```bash
# Put a host (veth) into br10 on leaf1, send a frame, then on leaf2:
$ sudo ip netns exec leaf2 vtysh -c 'show bgp l2vpn evpn'
# A Type-2 route for the host's MAC appears, next-hop = leaf1's VTEP IP.
$ sudo ip netns exec leaf2 bridge fdb show dev vxlan10
# leaf1's host MAC is installed pointing at leaf1 — learned from BGP, not flooding.
```

### Step 5 — Clean up

```bash
$ sudo ip netns delete spine leaf1 leaf2
```

---

## Further Reading

| Topic | Link |
|---|---|
| EVPN | [Wikipedia — Ethernet VPN](https://en.wikipedia.org/wiki/Ethernet_VPN) |
| VXLAN | [Lesson 15 — VXLAN](lesson-15-vxlan) |
| Leaf-spine | [Wikipedia — Clos network](https://en.wikipedia.org/wiki/Clos_network) |
| FRR EVPN | [docs.frrouting.org](https://docs.frrouting.org/en/latest/evpn.html) |

---

## Checkpoint

**Q1. How does EVPN replace VXLAN's data-plane MAC flooding, and why is that better at scale?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Plain VXLAN learns MAC locations by <em>flood-and-learn</em>: an unknown destination MAC is flooded
to every VTEP, and the reply teaches everyone where it lives. EVPN adds a <strong>control plane</strong>
— it carries host MAC/IP locations as <strong>BGP routes</strong> (Type-2), so each VTEP is explicitly
told "MAC X is behind VTEP Y" and populates its forwarding table from BGP instead of from flooding
(and Type-3 routes auto-discover the VTEPs). It's better at scale because flooding wastes bandwidth
fabric-wide and converges slowly, while turning L2 learning into a routing problem reuses BGP's proven
scalability, fast convergence, and policy controls; ARP can even be suppressed since the IP↔MAC binding
is already advertised.
</details>

---

**Q2. What is a leaf-spine topology, and what property makes it the standard data-center underlay?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
In leaf-spine, servers connect to <em>leaf</em> switches, every leaf connects to every <em>spine</em>,
and spines interconnect only leaves (a two-tier Clos network). The key property is <strong>uniform,
predictable any-to-any bandwidth</strong>: every leaf is exactly two hops (leaf→spine→leaf) from every
other, and there are as many equal-cost paths between leaves as there are spines, so ECMP spreads load
evenly and adding a spine increases bisection bandwidth. This regularity (no oversubscribed
"distribution layer," no spanning-tree blocking) is why it replaced the old access/aggregation/core
design for data centers, and EVPN/VXLAN ride on top of it.
</details>

---

**Q3. What do EVPN Type-2 and Type-3 routes each accomplish?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Type-2</strong> routes advertise a specific host's <strong>MAC address (optionally with its
IP)</strong> and the VTEP it sits behind — this is what replaces data-plane MAC flooding and ARP
flooding, letting every VTEP learn endpoint locations (and IP↔MAC bindings) from BGP. <strong>Type-3</strong>
(inclusive multicast) routes advertise "<strong>I am a VTEP participating in VNI N</strong>," which
provides <strong>VTEP auto-discovery</strong> and sets up how broadcast/unknown/multicast traffic for
that VNI is delivered — replacing the manual VTEP peer configuration (e.g. <code>bridge fdb append</code>)
you did in Lesson 15. (Type-5 adds IP-prefix routes for routing between subnets in the fabric.)
</details>

---

## Homework

Extend the lab to two VNIs (e.g. 10 and 20) on the same pair of leaves and confirm that EVPN keeps
the tenants isolated — a host in VNI 10 cannot reach a host in VNI 20 even though both ride the same
underlay. Explain, in terms of EVPN route attributes, how the fabric enforces that separation.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With two VNIs configured, each leaf advertises its hosts' Type-2 routes <em>tagged with the VNI</em>
(carried via the route's MPLS/VNI label and route-target community), and each VTEP only imports EVPN
routes whose <strong>route-target</strong> matches the VNIs it participates in. So a VNI-10 host's
MAC is only installed into other leaves' VNI-10 bridge/FDB, never into VNI-20's, and the VXLAN
encapsulation uses different VNI numbers (10 vs 20) on the wire. A VNI-10 host therefore has no
forwarding entry for, and no shared broadcast domain with, any VNI-20 host — traffic between them is
simply not bridgeable and would require explicit inter-subnet routing (a Type-5 route plus a routing
policy) to cross. The isolation is enforced by the VNI label plus route-target import filtering, which
is EVPN's multi-tenancy mechanism: one physical fabric, many cryptographically-separate L2 domains.
</details>
