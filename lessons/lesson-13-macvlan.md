---
title: "Lesson 13 — MACVLAN"
nav_order: 13
parent: "Phase 3: Layer-2 Networking"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 13: MACVLAN

## Concept

A **MACVLAN** interface gives a namespace its own **MAC address** directly on a physical LAN, sharing one physical NIC. Each MACVLAN looks like a separate physical machine to the rest of the network — its own MAC, its own IP, getting its own DHCP lease — even though several of them ride on a single parent NIC.

```
                 physical LAN (one switch port, one cable)
                              │
                          ┌───┴───┐
                          │ eth0  │  parent NIC (MAC AA)
                          └───┬───┘
              ┌───────────────┼───────────────┐
        ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐
        │ macvlan0  │   │ macvlan1  │   │ macvlan2  │
        │  MAC BB   │   │  MAC CC   │   │  MAC DD   │   each appears as a
        │ (ns1)     │   │ (ns2)     │   │ (ns3)     │   distinct host on the LAN
        └───────────┘   └───────────┘   └───────────┘
```

Contrast with a bridge: a bridge is a switch you wire veths into. A MACVLAN skips the bridge entirely — the parent NIC itself fans out into multiple MAC identities. It's lighter weight and gives containers a "real" presence on the physical network.

---

## How it works

The parent NIC receives frames for several MAC addresses and demultiplexes each to the right MACVLAN sub-interface by destination MAC. Outbound, each sub-interface stamps its own source MAC. The physical switch sees multiple MACs on one port and learns them all — to the switch, they're just several hosts.

MACVLAN has four **modes**:

| Mode | Behavior |
|---|---|
| `bridge` | Sub-interfaces on the same parent can talk to *each other* directly (the kernel shortcuts between them). Most common. |
| `private` | Sub-interfaces are fully isolated from each other, even on the same parent. |
| `vepa` | Frames between sub-interfaces are sent *up to the switch* and reflected back (needs a [VEPA](https://en.wikipedia.org/wiki/Virtual_Ethernet_Port_Aggregator)-capable switch). |
| `passthru` | One sub-interface takes over the whole parent — used for direct NIC handoff to a VM. |

{: .note }
> **The host ↔ MACVLAN blind spot**
> A surprising limitation: the **parent host cannot talk to its own MACVLAN children** (and vice versa) by default, even in `bridge` mode. Traffic from a MACVLAN goes *out* the physical NIC and is expected to come back via the switch — but the host's own stack and its children are short-circuited and never reach each other. Containers can reach the LAN and each other (in bridge mode), but not the host they run on. The standard workaround is to create an additional MACVLAN on the *host* side too, so the host joins the same fan-out. This trips up nearly everyone the first time.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip link add macvlan0 link eth0 type macvlan mode bridge` | Create a MACVLAN child on parent `eth0` |
| `ip link set macvlan0 netns ns1` | Move it into a namespace |
| `ip link set macvlan0 address <mac>` | Set a specific MAC (otherwise random) |
| `ip -d link show macvlan0` | Show MACVLAN mode and parent |
| `ip link add macvtap0 link eth0 type macvtap` | MACVTAP variant (L2 to userspace, for VMs) |

---

## Lab

Because MACVLAN needs a parent that's on a real L2 segment, we'll simulate a "physical LAN" with a bridge and a veth, then attach MACVLANs to the namespace-side interface.

### Step 1 — Build a fake LAN

```bash
# A bridge acts as our "switch"; a veth connects a host-side "NIC" to it.
$ sudo ip link add lanbr type bridge
$ sudo ip link set lanbr up
$ sudo ip link add eth-uplink type veth peer name eth-lan
$ sudo ip link set eth-lan master lanbr
$ sudo ip link set eth-lan up
$ sudo ip link set eth-uplink up      # this is our "physical NIC" / parent
```

### Step 2 — Create two MACVLAN children and move them into namespaces

```bash
$ sudo ip netns add ns1
$ sudo ip netns add ns2

$ sudo ip link add mvl1 link eth-uplink type macvlan mode bridge
$ sudo ip link add mvl2 link eth-uplink type macvlan mode bridge
$ sudo ip link set mvl1 netns ns1
$ sudo ip link set mvl2 netns ns2

$ sudo ip netns exec ns1 ip link set mvl1 up
$ sudo ip netns exec ns1 ip addr add 10.0.0.1/24 dev mvl1
$ sudo ip netns exec ns2 ip link set mvl2 up
$ sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev mvl2
```

### Step 3 — Each child has its own distinct MAC

```bash
$ sudo ip netns exec ns1 ip link show mvl1
N: mvl1@if...: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    link/ether b2:11:11:11:11:11 ...      ← unique MAC, different from parent
$ sudo ip netns exec ns2 ip link show mvl2
    link/ether c3:22:22:22:22:22 ...      ← another unique MAC
```

### Step 4 — Children talk to each other (bridge mode)

```bash
$ sudo ip netns exec ns1 ping -c 1 10.0.0.2
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.06 ms
```

In `bridge` mode the two MACVLANs on the same parent communicate directly. (In `private` mode this ping would fail.)

### Step 5 — Demonstrate the host blind spot

Give the *parent* `eth-uplink` an address and try to reach a child:

```bash
$ sudo ip addr add 10.0.0.254/24 dev eth-uplink
$ ping -c 1 -W 1 10.0.0.1
# 100% packet loss — the host cannot reach its own MACVLAN child
```

Even though `10.0.0.254` and `10.0.0.1` are "on the same wire," the host-to-child path is short-circuited. To fix it, the host needs its *own* MACVLAN on the same parent:

```bash
$ sudo ip addr del 10.0.0.254/24 dev eth-uplink
$ sudo ip link add mvl-host link eth-uplink type macvlan mode bridge
$ sudo ip link set mvl-host up
$ sudo ip addr add 10.0.0.254/24 dev mvl-host
$ ping -c 1 10.0.0.1
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.05 ms   # now it works
```

The host joined the MACVLAN fan-out as just another sibling, so it can reach the children.

### Step 6 — Clean up

```bash
$ sudo ip netns delete ns1
$ sudo ip netns delete ns2
$ sudo ip link del mvl-host
$ sudo ip link del eth-uplink
$ sudo ip link del lanbr
```

---

## Further Reading

| Topic | Link |
|---|---|
| MACVLAN / MACVTAP | [Wikipedia — MAC address](https://en.wikipedia.org/wiki/MAC_address) |
| Linux MACVLAN driver | [kernel.org — drivers/net/macvlan](https://www.kernel.org/doc/html/latest/networking/index.html) |
| VEPA | [Wikipedia — Virtual Ethernet Port Aggregator](https://en.wikipedia.org/wiki/Virtual_Ethernet_Port_Aggregator) |
| `ip-link` macvlan options | [man7.org — ip-link(8)](https://man7.org/linux/man-pages/man8/ip-link.8.html) |

---

## Checkpoint

**Q1. Why does a MACVLAN-based container appear as a completely separate machine on the LAN?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because it has its own unique MAC address on the physical segment, distinct from the parent NIC and from sibling MACVLANs. The container sends and receives frames stamped with its own MAC, so the physical switch learns that MAC on its port and treats it as just another host. The container gets its own IP (often its own DHCP lease, since the DHCP server sees a distinct MAC), and other devices on the LAN ARP for and reach it directly without any NAT or port mapping. From the network's point of view there's no indication it's sharing a NIC — it's indistinguishable from a separate physical machine plugged into the same switch.
</details>

---

**Q2. Two MACVLAN sub-interfaces on the same parent NIC can ping each other, but the host the parent NIC belongs to cannot ping either of them. Why, and how do you fix it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
This is the MACVLAN host↔child blind spot. The kernel short-circuits traffic between the host's own stack and its MACVLAN children: frames from a child are designed to go *out* the physical NIC toward the switch, and the direct host-to-child path is deliberately not connected. Sibling children in `bridge` mode *can* reach each other (the kernel shortcuts between them), but the host is excluded. The fix is to give the **host its own MACVLAN** on the same parent and assign the host's address there. The host then participates in the same fan-out as a sibling and can reach the children normally. This is a well-known gotcha — people assume "same wire = reachable," but the host is the one exception.
</details>

---

**Q3. When would you choose `private` mode over `bridge` mode for MACVLAN?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You'd choose `private` mode when sub-interfaces sharing a parent must **not** be able to talk to each other directly, even though they're on the same NIC — for example, a multi-tenant host where each container should only reach the upstream gateway/network and must be isolated from its neighbors for security. In `bridge` mode the kernel shortcuts sibling-to-sibling traffic, so containers can reach each other locally; in `private` mode that shortcut is removed, so a container can only communicate out through the physical network (and even reflected traffic is dropped). Pick `bridge` for convenience and local east-west traffic; pick `private` for strict isolation between co-located sub-interfaces.
</details>

---

## Homework

Set up three MACVLAN children on one parent in `bridge` mode (ns1, ns2, ns3). Confirm all three can ping each other. Then recreate them in `private` mode and re-test. Document which pings succeed and which fail in each mode, and explain the role of the parent NIC and switch in `private` mode.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Bridge mode:** all three children can ping each other. The kernel detects that the destination MAC belongs to a sibling MACVLAN on the same parent and shortcuts the frame directly between them without sending it out the wire. East-west traffic works locally.

**Private mode:** none of the children can ping each other. In `private` mode the kernel refuses to shortcut between siblings, and it also drops frames that would come back from the switch addressed to another local MACVLAN. Each child can still reach the *external* network (a real gateway or host beyond the switch), but sibling-to-sibling traffic is blocked in both directions. The parent NIC still demultiplexes inbound frames by MAC and stamps outbound source MACs, but the switch can't help reflect traffic between siblings because `private` explicitly discards such reflected frames (that reflection behavior is what `vepa` mode would *enable*, using a VEPA-capable switch). So the difference is entirely about whether local siblings are allowed to reach each other: `bridge` yes, `private` no.
</details>
