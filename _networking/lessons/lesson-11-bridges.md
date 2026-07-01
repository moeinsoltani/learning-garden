---
title: "Lesson 11 — Linux Bridges"
nav_order: 11
parent: "Phase 3: Layer-2 Networking"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 11: Linux Bridges

## Concept

A **Linux bridge** is a software Ethernet switch that lives inside the kernel. You attach interfaces to it as **ports**, and it forwards frames between them based on MAC addresses — exactly like a physical switch. This is the device Docker's `docker0` and most container networks are built on.

```
     ns1                    ns2                    ns3
   ┌──────┐              ┌──────┐              ┌──────┐
   │ eth0 │              │ eth0 │              │ eth0 │
   └───┬──┘              └───┬──┘              └───┬──┘
       │ veth                │ veth                │ veth
   ┌───┴────────────────────┴────────────────────┴───┐
   │                      br0                          │  ← software switch
   │   learns: which MAC is behind which port          │
   └───────────────────────────────────────────────────┘
```

A bridge operates at **Layer 2** — it forwards by MAC address and does not look at or modify IP headers. Two namespaces on the same bridge in the same subnet talk to each other with **no routing at all**.

---

## How it works

A bridge does three things, just like a hardware switch:

1. **Learning.** When a frame arrives on a port, the bridge records "source MAC X is reachable via port P" in its **forwarding database** (FDB). It learns the topology by watching traffic.
2. **Forwarding.** When a frame's destination MAC is in the FDB, it's sent out only that one port (unicast). 
3. **Flooding.** When the destination MAC is unknown (or is broadcast/multicast), the frame is sent out *all* ports except the one it came in on. The reply teaches the bridge where that MAC lives, so future frames are forwarded, not flooded.

```
First frame ns1 → ns2 (dest MAC unknown):
   br0 floods to ns2 AND ns3
   ns2 replies → br0 learns "ns2's MAC is on port veth-ns2"
Next frame ns1 → ns2:
   br0 forwards directly out veth-ns2 only (no flood)
```

{: .note }
> **Bridge (L2) vs router (L3) — the core distinction**
> A **bridge** forwards Ethernet frames by MAC address within a single broadcast domain/subnet. It never changes the packet and needs no IP configuration to do its job. A **router** forwards IP packets *between* different subnets, decrements the TTL, and consults the routing table. Rule of thumb: same subnet → bridge (switching); different subnets → router. The two namespaces in this lab can talk with no gateway and no routes precisely because the bridge keeps them in one L2 domain.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip link add br0 type bridge` | Create a bridge |
| `ip link set <iface> master br0` | Attach an interface as a bridge port |
| `ip link set <iface> nomaster` | Detach a port from the bridge |
| `bridge link show` | List bridge ports and their states |
| `bridge fdb show` | Show the forwarding database (learned MAC→port table) |
| `bridge fdb show br br0` | FDB entries for one bridge |
| `ip -d link show br0` | Show bridge details (STP state, etc.) |

---

## Lab

We'll connect three namespaces to one bridge and watch MAC learning happen.

### Step 1 — Create the bridge

```bash
$ sudo ip link add br0 type bridge
$ sudo ip link set br0 up
```

### Step 2 — Create three namespaces, each with a veth into the bridge

```bash
for n in 1 2 3; do
  sudo ip netns add ns$n
  sudo ip link add veth-ns$n type veth peer name br-ns$n
  sudo ip link set veth-ns$n netns ns$n
  sudo ip link set br-ns$n master br0          # host end → bridge port
  sudo ip link set br-ns$n up
  sudo ip netns exec ns$n ip link set veth-ns$n up
  sudo ip netns exec ns$n ip addr add 10.0.0.$n/24 dev veth-ns$n
done
```

Now all three namespaces are on `10.0.0.0/24`, wired to `br0`. No router, no routes added.

### Step 3 — See the bridge ports

```bash
$ bridge link show
3: br-ns1@... master br0 state forwarding ...
5: br-ns2@... master br0 state forwarding ...
7: br-ns3@... master br0 state forwarding ...
```

Three ports, all `forwarding`.

### Step 4 — Watch the FDB before and after traffic

```bash
# Before any traffic — only the bridge's own/static entries
$ bridge fdb show br br0
# (mostly permanent local entries for the port MACs)

# Generate traffic
$ sudo ip netns exec ns1 ping -c 1 10.0.0.2

# Now the bridge has learned ns1 and ns2's MACs dynamically
$ bridge fdb show br br0 | grep -v permanent
<ns1-mac> dev br-ns1 master br0
<ns2-mac> dev br-ns2 master br0
```

The bridge learned which port each MAC lives behind by watching the ping. `ns3`'s MAC isn't there — it never sent anything.

### Step 5 — Prove forwarding is L2 (no routing involved)

```bash
$ sudo ip netns exec ns1 ip route show
10.0.0.0/24 dev veth-ns1 proto kernel scope link src 10.0.0.1
# Only the connected route. No gateway, no inter-subnet routing.

$ sudo ip netns exec ns1 ping -c 1 10.0.0.3
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=0.08 ms   # works!
```

ns1 reaches ns3 with only a connected route — because they share a broadcast domain and the bridge switches the frame at L2. The kernel ARPs for `10.0.0.3`, the bridge floods the ARP, ns3 answers, done.

### Step 6 — Observe flooding with tcpdump

```bash
# Capture on ns3's interface, then ping a brand-new (unknown) MAC from ns1
$ sudo ip netns exec ns3 tcpdump -e -n -i veth-ns3 arp &
$ sudo ip netns exec ns1 ip neigh flush all
$ sudo ip netns exec ns1 ping -c 1 10.0.0.2
```

ns3 sees the *broadcast ARP request* (`who-has 10.0.0.2`) even though it's destined for ns2 — because broadcast frames are flooded to every port. ns3 does *not* see the unicast ARP reply or the ICMP, because those go only to the learned port.

### Step 7 — Clean up

```bash
$ sudo ip link del br0
$ for n in 1 2 3; do sudo ip netns delete ns$n; done
```

---

## Further Reading

| Topic | Link |
|---|---|
| Network bridging | [Wikipedia — Bridging (networking)](https://en.wikipedia.org/wiki/Bridging_(networking)) |
| MAC address / forwarding | [Wikipedia — Forwarding information base](https://en.wikipedia.org/wiki/Forwarding_information_base) |
| `bridge` command | [man7.org — bridge(8)](https://man7.org/linux/man-pages/man8/bridge.8.html) |
| Spanning Tree Protocol | [Wikipedia — Spanning Tree Protocol](https://en.wikipedia.org/wiki/Spanning_Tree_Protocol) |

---

## Checkpoint

**Q1. Why can two namespaces on the same bridge communicate without any IP routing configured?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because they're in the same subnet and the same Layer-2 broadcast domain. A bridge forwards Ethernet frames by MAC address; it doesn't route by IP. When ns1 wants to reach ns2 (same subnet), its connected route says "directly reachable on this interface," it ARPs for ns2's MAC (the bridge floods the request, ns2 answers), and then sends the frame. The bridge switches that frame straight to ns2's port. No gateway, no routing table lookup across subnets, and no TTL decrement is involved — it's pure L2 switching. Routing only becomes necessary when crossing between *different* subnets.
</details>

---

**Q2. The first packet to a never-seen-before destination MAC is flooded to all ports, but subsequent packets are not. What changed, and where is that information stored?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The bridge *learned* the destination's location. When the first frame triggered a reply (or any frame from the destination crossed the bridge), the bridge recorded "this source MAC is reachable via this port" in its forwarding database (FDB), visible with `bridge fdb show`. Once that MAC→port mapping exists, future frames for it are forwarded out only that single port (unicast) instead of being flooded to every port. Flooding is the fallback for unknown destinations and for broadcast/multicast; learning turns subsequent traffic into efficient targeted forwarding. FDB entries age out after a timeout if the MAC goes quiet, after which the bridge would flood again.
</details>

---

**Q3. Explain the difference between what a bridge does and what a router does, using the words "MAC," "IP," "subnet," and "TTL."**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A **bridge** forwards Ethernet frames by **MAC** address within a single **subnet** / broadcast domain. It does not inspect or modify the IP header and does not touch the **TTL** — the frame passes through transparently. A **router** forwards **IP** packets *between* different **subnets**: it looks up the destination IP in its routing table, decrements the **TTL** (dropping the packet if it hits zero), rewrites the L2 MAC addresses for the next hop, and forwards. So the bridge keeps you inside one subnet at L2; the router moves you between subnets at L3 and leaves a TTL "fingerprint" of having been routed.
</details>

---

## Homework

Put two namespaces on `br0` in subnet `10.0.0.0/24`, and a third namespace on the *same bridge* but with an address in a *different* subnet, `192.168.0.5/24`. Can ns1 (`10.0.0.1`) reach the third namespace? Capture ARP to explain exactly why or why not, and state what would be required to fix it.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
ns1 (`10.0.0.1`) cannot reach the third namespace (`192.168.0.5`) even though they share the same bridge. The bridge would happily carry the frames at L2 — that's not the problem. The problem is L3: when ns1 tries to reach `192.168.0.5`, its routing table has no route to `192.168.0.0/24` (only its connected `10.0.0.0/24`), so the kernel either rejects the packet outright or, if a default route existed, would send it to a gateway rather than ARPing for `192.168.0.5` directly. Even if you forced an ARP, the third namespace is in a different subnet and wouldn't consider itself directly reachable from `10.0.0.1` either.

The bridge being a pure L2 device means "same wire" but not "same subnet." To connect the two subnets you need a **router**: a namespace (or the host) with an address in each subnet and `ip_forward` enabled, plus routes/default gateways so each side knows to send cross-subnet traffic to the router. A bridge alone cannot join two different IP subnets — that's a routing job (Phase 5).
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 12 — VLAN Interfaces →](lesson-12-vlans){: .btn .btn-primary }
