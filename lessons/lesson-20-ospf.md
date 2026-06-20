---
title: "Lesson 20 — OSPF with FRR"
nav_order: 20
parent: "Phase 6: Dynamic Routing"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 20: OSPF with FRR

## Concept

**OSPF** (Open Shortest Path First) is the most common interior routing protocol — the one that routes *within* a single organization or data center. It is a **link-state** protocol: every router builds a complete map of the network's topology, then independently runs the same shortest-path algorithm ([Dijkstra](https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm)) to compute its best route to every destination.

When a link fails, every router learns about it, updates its map, re-runs the algorithm, and converges on new paths — automatically, in seconds. That self-healing is what you'll watch happen in the lab.

```
   ns1 ── router-a ───── router-b ── ns2
              \             /
               \           /
                router-c
   Primary path: a → b. Kill a↔b, and OSPF reroutes a → c → b automatically.
```

---

## How it works

Key OSPF concepts:

- **Areas** — OSPF divides large networks into areas to limit the size of each router's map. **Area 0** is the backbone; all other areas connect to it. Small labs use a single area 0.
- **LSAs** (Link-State Advertisements) — the messages routers flood to describe their links. Every router collects all LSAs to build the identical topology database.
- **SPF / Dijkstra** — each router runs shortest-path-first on that database to compute routes. "Shortest" is by **cost**, which is derived from link bandwidth (lower cost = preferred).
- **Neighbors and adjacencies** — routers discover each other with **hello** packets and form adjacencies to exchange LSAs.
- **DR/BDR** — on multi-access segments (like a shared LAN), routers elect a **Designated Router** and **Backup DR** to reduce the number of adjacencies (so they don't all fully mesh).

The crucial property: because every router has the *same* complete map and runs the *same* algorithm, they all agree on loop-free paths. When the map changes (a link drops), they all recompute and re-converge.

{: .note }
> **Why "link-state" reroutes fast**
> A link-state protocol doesn't wait to be *told* new routes by neighbors (that's distance-vector behavior, which converges slowly). Instead, the failure is flooded as an LSA, every router updates its own map, and each independently recomputes. Convergence is bounded by flooding time + SPF computation, typically a few seconds. This is why OSPF (and IS-IS) dominate inside fast, self-healing networks.

---

## New Commands This Lesson

| vtysh command | What it does |
|---|---|
| `router ospf` | Enter OSPF configuration |
| `network 10.0.0.0/24 area 0` | Run OSPF on interfaces in this subnet, in area 0 |
| `show ip ospf neighbor` | Show OSPF adjacencies and their state |
| `show ip ospf database` | Show the link-state database (the topology map) |
| `show ip route ospf` | Show routes OSPF installed |
| `redistribute connected` | Advertise directly-connected routes into OSPF |

---

## Lab

We'll build the triangle topology: three routers (a, b, c) fully meshed, with ns1 hanging off router-a and ns2 off router-b. OSPF will find the path, and we'll kill the a↔b link to watch it reroute via c.

### Step 1 — Create namespaces and links

```bash
# Five namespaces
$ for n in ns1 ns2 ra rb rc; do sudo ip netns add $n; done

# Transit links between routers (each a /30)
# ra-rb: 10.0.12.0/30 | ra-rc: 10.0.13.0/30 | rb-rc: 10.0.23.0/30
# access links: ns1-ra 10.0.1.0/24 | ns2-rb 10.0.2.0/24
```

(Use the veth-pair pattern from Lesson 17 for each link — host-side ends moved into the respective router namespaces. The full script is long; the structure is what matters.)

Enable forwarding on all three routers:

```bash
$ for r in ra rb rc; do sudo ip netns exec $r sysctl -w net.ipv4.ip_forward=1; done
```

### Step 2 — Run FRR per namespace

FRR can run a separate instance per namespace. The simplest lab approach is to launch `zebra` and `ospfd` inside each router namespace, each with its own config/socket directory. For example:

```bash
$ sudo ip netns exec ra /usr/lib/frr/zebra  -d -i /run/frr-ra/zebra.pid  ...
$ sudo ip netns exec ra /usr/lib/frr/ospfd  -d -i /run/frr-ra/ospfd.pid  ...
```

(In practice many people use the `vrf`/`netns` integration or a tool like [mininet](https://en.wikipedia.org/wiki/Mininet) / containerlab to script this. The conceptual config below is what each router needs.)

### Step 3 — Configure OSPF on each router (vtysh)

On **router-a**:

```
configure terminal
router ospf
  network 10.0.1.0/24 area 0     # access link to ns1
  network 10.0.12.0/30 area 0    # transit to rb
  network 10.0.13.0/30 area 0    # transit to rc
```

On **router-b**:

```
router ospf
  network 10.0.2.0/24 area 0
  network 10.0.12.0/30 area 0
  network 10.0.23.0/30 area 0
```

On **router-c**:

```
router ospf
  network 10.0.13.0/30 area 0
  network 10.0.23.0/30 area 0
```

### Step 4 — Verify adjacencies formed

```
ra# show ip ospf neighbor
Neighbor ID    State    Address       Interface
10.0.12.2      Full     10.0.12.2     <ra-rb link>
10.0.13.2      Full     10.0.13.2     <ra-rc link>
```

`Full` state means the adjacency is fully established and LSAs have been exchanged. Each router now has the complete map.

### Step 5 — Check the computed route and path

```
ra# show ip route ospf
O>* 10.0.2.0/24 [110/20] via 10.0.12.2, <ra-rb link>
#                  ^^^ admin distance 110 (OSPF), cost 20
```

The direct a→b path is chosen because it's lowest cost. Confirm end-to-end:

```bash
$ sudo ip netns exec ns1 ping -c 1 10.0.2.<ns2>
64 bytes ... ttl=62      # two routed hops: ra then rb
```

### Step 6 — Break the a↔b link and watch reconvergence

```bash
# Kill the direct link between router-a and router-b
$ sudo ip netns exec ra ip link set <ra-rb-iface> down
```

Within a few seconds:

```
ra# show ip route ospf
O>* 10.0.2.0/24 [110/30] via 10.0.13.2, <ra-rc link>
#                  cost is now 30 (longer path a→c→b)
```

OSPF noticed the failure (the neighbor went down, an LSA was flooded), every router recomputed, and traffic now flows **a → c → b**. The ping keeps working after a brief blip:

```bash
$ sudo ip netns exec ns1 ping 10.0.2.<ns2>
64 bytes ... ttl=61      # now THREE hops (ra → rc → rb), TTL one lower
```

The TTL dropping from 62 to 61 confirms the path got one hop longer. Bring the link back up and OSPF reverts to the shorter path.

### Step 7 — Clean up

```bash
$ for n in ns1 ns2 ra rb rc; do sudo ip netns delete $n; done
```

---

## Further Reading

| Topic | Link |
|---|---|
| OSPF | [Wikipedia — Open Shortest Path First](https://en.wikipedia.org/wiki/Open_Shortest_Path_First) |
| Link-state routing | [Wikipedia — Link-state routing protocol](https://en.wikipedia.org/wiki/Link-state_routing_protocol) |
| Dijkstra's algorithm | [Wikipedia — Dijkstra's algorithm](https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm) |
| FRR OSPF docs | [docs.frrouting.org — OSPFv2](https://docs.frrouting.org/en/latest/ospfd.html) |

---

## Checkpoint

**Q1. Why does traffic reroute automatically when a link fails in an OSPF network?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because OSPF is a link-state protocol where every router maintains a complete, identical map of the network topology. When a link fails, the routers adjacent to it detect the loss (the OSPF neighbor stops responding to hello packets / the interface goes down) and **flood a new LSA** describing the change. Every router in the area receives that LSA, updates its topology database, and independently re-runs the shortest-path (Dijkstra) algorithm on the new map. Since the failed link is no longer in the map, the algorithm naturally computes the next-best loop-free path around the failure, and each router installs the new route. No human acts and no central controller is needed — the rerouting is an emergent result of every router recomputing from the same updated map. Convergence takes only the time to flood the LSA plus recompute, typically a few seconds.
</details>

---

**Q2. What does it mean that OSPF is a "link-state" protocol, and how does that differ from "distance-vector"?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
In a **link-state** protocol like OSPF, each router learns the *entire topology* — every router and link in the area — by flooding and collecting LSAs, then independently computes shortest paths with Dijkstra's algorithm on that full map. Routers share *topology*, not routes, and each one calculates its own routes.

In a **distance-vector** protocol (like RIP), routers don't see the whole map. Each one only knows "to reach network X, my neighbor says it costs N, so via that neighbor it costs N+1." They share *their route tables* (distances and directions) with neighbors and trust them. This is simpler but converges more slowly and is prone to loops/count-to-infinity problems during failures.

The practical upshot: link-state converges faster and more reliably because every router has full visibility and recomputes from ground truth, whereas distance-vector relies on iterative gossip of summarized distances. (BGP, covered next, is a third type — *path-vector* — which shares full AS-paths to avoid loops at internet scale.)
</details>

---

**Q3. After the a↔b link fails and traffic reroutes via c, the end-to-end TTL at the destination drops by one. Why?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because the new path has one more routed hop than the old one. Originally ns1→ns2 went ns1 → router-a → router-b → ns2, which is two routers decrementing the TTL (64 → 62 at the destination). After the a↔b link fails, the path becomes ns1 → router-a → router-c → router-b → ns2 — now *three* routers forward the packet, each decrementing the TTL by one, so it arrives at 61 instead of 62. The TTL is effectively a hop counter, so a TTL one lower than before is direct evidence that OSPF rerouted onto a longer (one-extra-hop) path. This is a handy way to confirm reconvergence happened and to see the new path length without tracing every interface.
</details>

---

## Homework

In the triangle topology, give the a↔c and c↔b links a deliberately *high* OSPF cost (e.g. `ip ospf cost 1000` on those interfaces) while leaving a↔b at default cost. Predict which path OSPF chooses normally, and which it chooses after a↔b fails. Then verify with `show ip route ospf` and explain how cost influences SPF's decision.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Normally:** OSPF chooses the direct **a → b** path. Its total cost (sum of link costs along the path) is far lower than the a → c → b path, which now traverses two high-cost (1000) links. SPF/Dijkstra always selects the lowest *total cost* path, so the direct link wins decisively.

**After a↔b fails:** the direct path no longer exists, so OSPF is forced onto the only remaining route, **a → c → b**, despite its high cost (≈2000+). There's no cheaper alternative, so SPF selects it. Connectivity is preserved, just over the expensive path.

How cost drives the decision: each interface has an OSPF cost (by default inversely proportional to bandwidth; here we set some manually). SPF computes the shortest path as the one minimizing the *sum* of interface costs from source to destination. Raising a link's cost makes any path using it less attractive, so OSPF avoids it whenever a lower-total-cost alternative exists — but it will still use a high-cost link if it's the only way to reach the destination. This is exactly how operators steer traffic onto preferred links (set low cost) and keep others as backups (set high cost) without removing them: the high-cost links sit idle until the preferred path fails, then automatically take over.
</details>
