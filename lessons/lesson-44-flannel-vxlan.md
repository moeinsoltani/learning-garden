---
title: "Lesson 44 — VXLAN-based CNI (Flannel)"
nav_order: 44
parent: "Phase 14: Container & Cloud"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 44: VXLAN-based CNI (Flannel)

## Concept

Lesson 43 promised a flat pod network where Pod A reaches Pod B by its real IP across nodes. But what if the underlying network (the "underlay") **doesn't know** how to route each node's pod CIDR — a typical cloud VPC, where you can't just inject pod routes? **Flannel's VXLAN backend** solves this by building an **overlay**: it takes the VXLAN you learned in Lesson 15 and uses it to stretch one flat L2/L3 pod network across nodes that only have plain IP connectivity to each other.

```
   Node A (192.168.1.10)                     Node B (192.168.1.20)
   ┌───────────────────────┐                 ┌───────────────────────┐
   │ pod 10.244.1.5         │                 │ pod 10.244.2.7         │
   │   └─ cni0 bridge       │                 │   └─ cni0 bridge       │
   │       └─ flannel.1 (VTEP)               │       └─ flannel.1 (VTEP)
   └───────────┬───────────┘                 └───────────┬───────────┘
               │  inner pod frame 10.244.1.5→10.244.2.7  │
               └── wrapped in UDP/VXLAN (VNI 1) ─────────┘
                   outer  192.168.1.10 → 192.168.1.20
                   (the underlay only ever sees node-IP UDP packets)
```

The underlay just carries ordinary UDP between node IPs; inside that UDP is the real pod-to-pod frame. To the pods, it's one flat network — exactly the Lesson 43 promise, delivered over an indifferent underlay.

---

## How it works

Each node runs a VXLAN device named **`flannel.1`** — the **VTEP** (VXLAN Tunnel Endpoint, Lesson 15). When a pod on Node A sends to a pod on Node B:

1. Node A routes the pod packet toward `flannel.1` (Flannel installs a route: "pod CIDR `10.244.2.0/24` is reachable via the overlay").
2. `flannel.1` needs two things to encapsulate: the **destination VTEP's MAC** (to build the inner Ethernet header) and the **destination node's IP** (the outer UDP destination). It gets the MAC from the **neighbor/ARP table** and the node IP from the **forwarding database (FDB)**.
3. The inner pod frame is wrapped in **UDP (VXLAN, VNI from config) → outer IP `NodeA → NodeB`** and sent over the underlay.
4. Node B's `flannel.1` **decapsulates**, recovering the original pod frame, and delivers it to the local pod via `cni0`.

**Who populates the FDB and neighbor entries?** This is the crux. There's no multicast and no flood-and-learn here. The **flannel daemon (`flanneld`) on each node watches the Kubernetes API server** for node objects. Each node publishes its pod CIDR, its node IP, and its `flannel.1` VTEP MAC as annotations. When `flanneld` sees a new/changed node, it programs that node's entries locally:

- an **FDB entry**: "VTEP MAC `M` is reachable at node IP `NodeB`" → `bridge fdb`
- a **neighbor entry**: "pod-CIDR-gateway / remote VTEP → MAC `M`" → `ip neigh`
- a **route**: "remote pod CIDR via `flannel.1`"

So the control plane (Kubernetes node objects) keeps the dataplane (FDB + neigh + routes) in sync as nodes join and leave. This is **unicast VXLAN with a controller-populated FDB** — precisely the manual `bridge fdb append` pattern from Lesson 15, but automated by `flanneld` from cluster state.

{: .note }
> **Overlay vs underlay — the one-sentence summary**
> The **underlay** is the real network between node IPs (your VPC/LAN); the **overlay** is the virtual pod network built on top with VXLAN. Flannel's job is to make the overlay look flat to pods while asking nothing of the underlay except plain IP reachability between nodes — which is why it works in clouds where you can't add pod routes.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip -d link show flannel.1` | Show the VXLAN VTEP device and its VNI |
| `bridge fdb show dev flannel.1` | The forwarding DB: VTEP MAC → remote node IP |
| `ip neigh show dev flannel.1` | Neighbor entries: remote pod/VTEP → MAC |
| `ip route` | Routes to remote pod CIDRs via `flannel.1` |
| `tcpdump -ni <uplink> udp port 8472` | Capture the encapsulated VXLAN traffic (Flannel default port 8472) |

---

## Lab

We'll build Flannel's per-node setup **by hand** between two namespaces acting as nodes, reproducing what `flanneld` would program. (This is Lesson 15's VXLAN lab, reframed as a CNI. Run on one host; the two "node" namespaces simulate two machines.)

### Step 1 — Two "nodes" with underlay connectivity

```bash
$ sudo ip netns add nodeA; sudo ip netns add nodeB
$ sudo ip link add ua netns nodeA type veth peer name ub netns nodeB
$ sudo ip netns exec nodeA ip addr add 192.168.1.10/24 dev ua
$ sudo ip netns exec nodeB ip addr add 192.168.1.20/24 dev ub
$ sudo ip netns exec nodeA ip link set ua up
$ sudo ip netns exec nodeB ip link set ub up
$ sudo ip netns exec nodeA ping -c1 192.168.1.20      # underlay works
```

### Step 2 — Create the VTEP (flannel.1) on each node

```bash
# VNI 1, VXLAN port 8472 (Flannel's default), no remote yet (FDB-driven)
$ sudo ip netns exec nodeA ip link add flannel.1 type vxlan id 1 dev ua dstport 8472 nolearning
$ sudo ip netns exec nodeB ip link add flannel.1 type vxlan id 1 dev ub dstport 8472 nolearning
$ sudo ip netns exec nodeA ip addr add 10.244.1.0/32 dev flannel.1
$ sudo ip netns exec nodeB ip addr add 10.244.2.0/32 dev flannel.1
$ sudo ip netns exec nodeA ip link set flannel.1 up
$ sudo ip netns exec nodeB ip link set flannel.1 up
```

### Step 3 — Note each VTEP's MAC (flanneld would publish these)

```bash
$ sudo ip netns exec nodeA ip -d link show flannel.1   # note MAC, e.g. MAC_A
$ sudo ip netns exec nodeB ip -d link show flannel.1   # note MAC, e.g. MAC_B
```

### Step 4 — Manually program what flanneld would (routes + neigh + FDB)

```bash
# On nodeA: reach nodeB's pod CIDR via the overlay, using nodeB's VTEP MAC at nodeB's IP
$ sudo ip netns exec nodeA ip route add 10.244.2.0/24 dev flannel.1
$ sudo ip netns exec nodeA ip neigh add 10.244.2.0 lladdr MAC_B dev flannel.1
$ sudo ip netns exec nodeA bridge fdb append MAC_B dev flannel.1 dst 192.168.1.20

# On nodeB: the mirror image
$ sudo ip netns exec nodeB ip route add 10.244.1.0/24 dev flannel.1
$ sudo ip netns exec nodeB ip neigh add 10.244.1.0 lladdr MAC_A dev flannel.1
$ sudo ip netns exec nodeB bridge fdb append MAC_A dev flannel.1 dst 192.168.1.10
```

### Step 5 — Inspect the tables (what `flanneld` keeps in sync)

```bash
$ sudo ip netns exec nodeA bridge fdb show dev flannel.1
MAC_B dst 192.168.1.20 self permanent
$ sudo ip netns exec nodeA ip neigh show dev flannel.1
10.244.2.0 lladdr MAC_B PERMANENT
$ sudo ip netns exec nodeA ip route | grep flannel
10.244.2.0/24 dev flannel.1 ...
```

### Step 6 — Send overlay traffic and capture the encapsulation

```bash
# Capture the OUTER packet on the underlay while pinging across the overlay
$ sudo ip netns exec nodeB tcpdump -ni ub udp port 8472 &
$ sudo ip netns exec nodeA ping -c2 10.244.2.0
# tcpdump shows OUTER 192.168.1.10 → 192.168.1.20, UDP/8472, VXLAN VNI 1, INNER 10.244.1.0 → 10.244.2.0
```

### Step 7 — Clean up

```bash
$ sudo ip netns delete nodeA; sudo ip netns delete nodeB
```

---

## Further Reading

| Topic | Link |
|---|---|
| Flannel | [github.com/flannel-io/flannel](https://github.com/flannel-io/flannel) |
| Flannel VXLAN backend | [flannel — VXLAN backend docs](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md) |
| VXLAN | [Wikipedia — VXLAN](https://en.wikipedia.org/wiki/Virtual_Extensible_LAN) |
| `bridge fdb` | [man7.org — bridge(8)](https://man7.org/linux/man-pages/man8/bridge.8.html) |
| VXLAN device | [man7.org — ip-link(8)](https://man7.org/linux/man-pages/man8/ip-link.8.html) |

---

## Checkpoint

**Q1. What does Flannel's VXLAN FDB look like, who populates it, and how does it stay in sync?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The FDB on each node's `flannel.1` device maps **each remote node's VTEP MAC → that node's underlay IP** — entries like `MAC_B dst 192.168.1.20` (`bridge fdb show dev flannel.1`). This tells the local VTEP "to reach the VTEP with MAC_B, send the encapsulating UDP packet to node IP 192.168.1.20." Alongside it sit **neighbor entries** (remote pod-CIDR/VTEP → MAC) and **routes** (remote pod CIDR via `flannel.1`).

It's populated by **`flanneld`**, the Flannel daemon on each node, which **watches the Kubernetes API server** for node objects. Each node publishes its pod CIDR, node IP, and VTEP MAC (as node annotations). When `flanneld` observes a node added/changed/removed, it correspondingly **adds/updates/deletes** the FDB, neighbor, and route entries on the local node. So the cluster control plane (node objects in the API server) is the source of truth, and `flanneld` continuously reconciles the dataplane to match — no multicast, no flood-and-learn; it's **unicast VXLAN with a controller-populated FDB**.
</details>

---

**Q2. Why does Flannel use an overlay at all, instead of just routing each node's pod CIDR directly (like Calico can)?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Direct routing requires the **underlay network to know how to route each node's pod CIDR** — i.e., you must be able to tell the physical/cloud routers "pod CIDR 10.244.2.0/24 lives behind node B." That's possible on a network you control (inject static routes, or run BGP as Calico does), but in many environments — especially **cloud VPCs** — you **can't add arbitrary routes** for non-VPC ranges, and the fabric will drop or refuse to route packets with unknown pod source/destination IPs. An **overlay (VXLAN)** sidesteps this entirely: it **encapsulates** pod traffic inside ordinary UDP between **node IPs**, which the underlay already knows how to route. The underlay never sees pod IPs — only node-to-node UDP — so Flannel works on top of *any* network that provides plain IP connectivity between nodes, requiring nothing of it. The trade-off is encapsulation overhead (extra headers, slightly lower MTU/throughput) versus the simplicity and portability of "works anywhere."
</details>

---

**Q3. In the captured packet, what is the "inner" and what is the "outer," and which IPs appear in each?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The **inner** packet is the original **pod-to-pod** frame: source = sending pod's IP, destination = receiving pod's IP (e.g., `10.244.1.5 → 10.244.2.7`), with the pods' VTEP MACs as the inner Ethernet addresses. The **outer** packet is the **encapsulation** the VTEP adds: an IP header with source = **sending node's** underlay IP and destination = **receiving node's** underlay IP (e.g., `192.168.1.10 → 192.168.1.20`), a **UDP** header (Flannel's default VXLAN port **8472**), and the **VXLAN header** carrying the **VNI**. So the underlay only ever sees node-IP UDP traffic (outer); the pod IPs are hidden inside (inner). Encapsulation = "wrap the pod frame in node-to-node UDP"; decapsulation on the far node strips the outer headers to recover the inner pod frame. This is exactly the VXLAN structure from Lesson 15, now carrying pod traffic.
</details>

---

## Homework

Add a **second pod CIDR entry** to your manual setup — pretend a third node (`nodeC`, `192.168.1.30`, pod CIDR `10.244.3.0/24`, VTEP MAC `MAC_C`) joined the cluster. Write the exact `ip route`, `ip neigh`, and `bridge fdb` commands `flanneld` on `nodeA` would run to integrate it, and the commands it would run if that node were later **removed**. Then explain why this watch-and-reconcile model scales better than a flood-and-learn (multicast) VXLAN for a large, churning cluster.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**On node join (flanneld on nodeA programs):**

```
ip route add 10.244.3.0/24 dev flannel.1
ip neigh add  10.244.3.0 lladdr MAC_C dev flannel.1
bridge fdb append MAC_C dev flannel.1 dst 192.168.1.30
```

**On node removal (flanneld on nodeA reverses):**

```
bridge fdb del MAC_C dev flannel.1 dst 192.168.1.30
ip neigh del 10.244.3.0 dev flannel.1
ip route del 10.244.3.0/24 dev flannel.1
```

`flanneld` triggers these automatically when it sees the node object appear/disappear in the API server.

**Why watch-and-reconcile beats flood-and-learn at scale:** A multicast/flood-and-learn VXLAN discovers remote MACs by **broadcasting unknown-destination frames to a multicast group** and learning replies — which means (1) it depends on the underlay supporting **multicast** (many clouds don't), and (2) every node receives flooded traffic for unknown destinations and BUM (broadcast/unknown-unicast/multicast) frames, generating overhead that grows with cluster size and churn. The **controller-populated unicast** model instead has each node learn the *exact* set of `{node IP, VTEP MAC, pod CIDR}` tuples from the **single source of truth (the API server)** and program **precise unicast** FDB/neigh/route entries — no flooding, no multicast dependency, and updates are **targeted** (one delta per node change) rather than discovered by broadcasting. For a large cluster with pods and nodes constantly coming and going, pushing exact deltas from a central registry is far less traffic and far more predictable than flooding the fabric to relearn state. It also fails more cleanly: entries reflect declared cluster state, not whatever happened to be overheard.
</details>
