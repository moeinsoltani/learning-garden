---
title: "Lesson 09 — Bond Interfaces"
nav_order: 9
parent: "Phase 2: Virtual Interfaces"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 09: Bond Interfaces

## Concept

A **bond** combines several physical (or virtual) links into one logical interface. To everything above it, the bond looks like a single NIC with one IP and one MAC. Below it, the kernel spreads or fails-over traffic across the member links — called **slaves**.

Two reasons to bond:

- **Redundancy** — if one cable or switch dies, traffic keeps flowing over the others.
- **Throughput** — combine the bandwidth of multiple links (with caveats).

```
            bond0  (one logical interface — 10.0.0.5/24)
          ╱   │   ╲
       eth1  eth2  eth3        ← slaves (member links)
         │     │     │
      switch A    switch B     ← spread across switches for redundancy
```

---

## How it works

The behavior depends on the **bonding mode**. The three you must know:

| Mode | Name | Behavior | When to use |
|---|---|---|---|
| `active-backup` (1) | Failover | One slave active, the rest idle. On failure, a backup takes over. | Redundancy where the switch can't do link aggregation. Works with *any* switch. |
| `balance-rr` (0) | Round-robin | Packets sent across slaves in turn. Can boost single-flow throughput. | Throughput, but risks packet reordering; needs matching switch config. |
| `802.3ad` (4) | LACP | Negotiates an aggregated link group with the switch via [LACP](https://en.wikipedia.org/wiki/Link_aggregation#Link_Aggregation_Control_Protocol). Hashes flows across slaves. | The "proper" throughput+redundancy mode, but **requires switch support and matching config**. |

The key distinction: `active-backup` needs nothing from the network — the switch doesn't even know bonding is happening. `802.3ad` (LACP) is a negotiated protocol; both the host and the switch must agree, or the link won't come up correctly.

{: .note }
> **Why round-robin rarely multiplies throughput for one connection**
> A single TCP flow striped packet-by-packet across links arrives out of order, and TCP treats reordering like loss — it backs off. So `balance-rr` often *hurts* a single connection. LACP avoids this by hashing each *flow* to one link: a single connection stays on one slave (so no reordering), but many connections spread across all slaves. You get aggregate throughput across many flows, not a faster single flow.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip link add bond0 type bond mode active-backup` | Create a bond with a given mode |
| `ip link set <slave> master bond0` | Enslave an interface to the bond (slave must be DOWN first) |
| `ip link set <slave> nomaster` | Remove a slave from the bond |
| `cat /proc/net/bonding/bond0` | Show bond status: mode, active slave, per-slave link state |
| `ip -d link show bond0` | Show bond details including mode |

---

## Lab

We'll build a bond over two veth pairs so you can experiment without physical NICs. The "far" ends go into a peer namespace acting as the switch side.

### Step 1 — Create the slaves (two veth pairs)

```bash
$ sudo ip netns add peer
$ sudo ip link add eth1 type veth peer name p1
$ sudo ip link add eth2 type veth peer name p2
$ sudo ip link set p1 netns peer
$ sudo ip link set p2 netns peer
$ sudo ip netns exec peer ip link set p1 up
$ sudo ip netns exec peer ip link set p2 up
```

### Step 2 — Create the bond and enslave the interfaces

Slaves must be **down** before enslaving:

```bash
$ sudo ip link add bond0 type bond mode active-backup
$ sudo ip link set eth1 down
$ sudo ip link set eth2 down
$ sudo ip link set eth1 master bond0
$ sudo ip link set eth2 master bond0
$ sudo ip link set bond0 up
$ sudo ip link set eth1 up
$ sudo ip link set eth2 up
$ sudo ip addr add 10.0.0.5/24 dev bond0
```

### Step 3 — Inspect bond status

```bash
$ cat /proc/net/bonding/bond0
Bonding Mode: fault-tolerance (active-backup)
Currently Active Slave: eth1
MII Status: up
...
Slave Interface: eth1
MII Status: up
Slave Interface: eth2
MII Status: up
```

`Currently Active Slave: eth1` — only eth1 carries traffic; eth2 is a hot standby.

### Step 4 — Simulate a link failure and watch failover

```bash
# Knock out the active slave
$ sudo ip link set eth1 down

$ cat /proc/net/bonding/bond0
Currently Active Slave: eth2      ← failover happened automatically
Slave Interface: eth1
MII Status: down
Slave Interface: eth2
MII Status: up
```

The bond switched to eth2 with no change to `bond0`'s IP or MAC. Anything using `10.0.0.5` never noticed.

### Step 5 — Inspect the mode and members declaratively

```bash
$ ip -d link show bond0
... bond mode active-backup ...
$ ip link show master bond0
# lists eth1 and eth2 as members
```

### Step 6 — Clean up

```bash
$ sudo ip link del bond0
$ sudo ip link del eth1   # removes eth1/p1 pair
$ sudo ip link del eth2   # removes eth2/p2 pair
$ sudo ip netns delete peer
```

---

## Further Reading

| Topic | Link |
|---|---|
| Link aggregation | [Wikipedia — Link aggregation](https://en.wikipedia.org/wiki/Link_aggregation) |
| LACP (802.3ad) | [Wikipedia — Link Aggregation Control Protocol](https://en.wikipedia.org/wiki/Link_aggregation#Link_Aggregation_Control_Protocol) |
| Linux bonding driver | [kernel.org — bonding.txt](https://www.kernel.org/doc/Documentation/networking/bonding.txt) |
| `ip-link` bond options | [man7.org — ip-link(8)](https://man7.org/linux/man-pages/man8/ip-link.8.html) |

---

## Checkpoint

**Q1. When would you choose `active-backup` over `802.3ad` (LACP)?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Choose `active-backup` when you need redundancy but cannot rely on switch cooperation — for example, when the two links go to two *different* switches (so no single switch can negotiate an aggregation group), or when the switch simply doesn't support LACP, or when you want the simplest possible setup that "just works" with any switch. `active-backup` is purely host-side: the switch doesn't even know bonding exists. `802.3ad` requires the switch to support LACP and be configured with a matching aggregation group on the same switch; in exchange it gives you both redundancy *and* aggregate throughput across multiple flows. So: pick `active-backup` for maximum compatibility and cross-switch redundancy; pick `802.3ad` when you control the switch and want bandwidth aggregation too.
</details>

---

**Q2. You set up a `balance-rr` bond hoping to double the speed of a single large file transfer, but throughput barely improves and you see TCP retransmits. Why?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`balance-rr` stripes individual packets of a single flow across both slaves. The two links have slightly different latencies, so packets arrive out of order at the receiver. TCP interprets out-of-order delivery much like packet loss: it triggers duplicate ACKs and congestion backoff, and may retransmit. The result is that a single TCP connection gains little and can even slow down. Round-robin helps aggregate throughput only for traffic that tolerates reordering or for many parallel flows. For a single connection you'd want it pinned to one link (as LACP does by hashing the flow to one slave).
</details>

---

**Q3. After failover from eth1 to eth2, the bond's IP and MAC are unchanged and existing connections survive. Why doesn't the rest of the network need to relearn anything?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The bond presents a single stable identity to the upper layers: one IP and one MAC that belong to `bond0`, not to any individual slave. When eth1 fails and eth2 takes over, traffic simply egresses through a different physical port, but the source MAC and IP in the frames are still the bond's. From the peer's perspective nothing about the host changed. (In `active-backup`, the driver may emit a gratuitous ARP so the upstream switch updates which physical port the bond's MAC now appears on, but the MAC/IP themselves don't change.) Because the identity is constant, ARP caches, routes, and established TCP connections all remain valid across the failover.
</details>

---

## Homework

Create an `active-backup` bond with two slaves and start a continuous ping through it (`ping 10.0.0.x`). While the ping runs, bring the active slave down, then a few seconds later bring it back up and bring the *other* slave down. Observe the ping output and `/proc/net/bonding/bond0` throughout. How many pings (if any) are lost during each failover, and why?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
During each failover you typically lose zero to a small handful of pings. The exact number depends on the link-monitoring interval (`miimon`) — the bond only notices a slave is down when its link-state monitor detects the carrier drop, and only then promotes the backup. With MII monitoring at the default interval, the gap is usually under a second, so a 1-per-second ping might lose one packet or none. Once the active slave is restored you can take the other one down with no loss, because there's always at least one healthy slave carrying `bond0`'s traffic. `/proc/net/bonding/bond0` shows `Currently Active Slave` flipping between eth1 and eth2 at each transition while the bond's IP/MAC stay put. The takeaway: failover is fast but not perfectly instantaneous — it's bounded by how quickly link failure is detected, which is why `miimon` tuning matters in HA setups.
</details>
