---
title: "Lesson 07 — veth Pairs"
nav_order: 7
parent: "Phase 2: Virtual Interfaces"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 07: veth Pairs

## Concept

A **veth pair** (virtual Ethernet pair) is a software cable with two ends. Whatever you push into one end comes out the other. There is no way to have a half-cable: veths are always created in pairs, and deleting either end deletes both.

This single primitive is the foundation of *all* Linux virtual networking. Containers, bridges, and router topologies are all just veth pairs wired together in different patterns.

```
      ns1                              ns2
  ┌─────────┐                      ┌─────────┐
  │ veth-a  │ ═══════════════════  │ veth-b  │
  │10.0.0.1 │    virtual cable     │10.0.0.2 │
  └─────────┘                      └─────────┘
       │                                │
   packet in ───────────────────────► packet out
   (TX on veth-a)                  (RX on veth-b)
```

You used a veth pair already in Lessons 5 and 6 without explanation. Now you understand it: it is a back-to-back pair of interfaces where one end's transmit is the other end's receive.

---

## How it works

When you run `ip link add veth-a type veth peer name veth-b`, the kernel creates two interfaces joined internally. A frame transmitted on `veth-a` is immediately delivered to `veth-b`'s receive path, and vice versa — no NIC, no driver, no wire, just a memory handoff inside the kernel.

The power comes from **moving one end into a namespace**:

```bash
ip link set veth-b netns ns2
```

Now `veth-a` lives in one namespace (or the host) and `veth-b` lives in another. The cable crosses the namespace boundary. This is exactly how a container gets connectivity: one end stays on the host (often attached to a bridge), the other end moves into the container's namespace and is renamed `eth0`.

{: .note }
> **Both ends must be UP, and both need the link partner up**
> A veth is up only when *both* ends are administratively up. If `veth-a` is up but `veth-b` is down, `veth-a` will show `NO-CARRIER` — exactly like unplugging the far end of a real cable. This is a very common "why won't it ping" cause.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip link add <a> type veth peer name <b>` | Create a veth pair |
| `ip link set <b> netns <ns>` | Move one end into a namespace |
| `ip link set <iface> up` | Bring an interface up |
| `ip link del <a>` | Delete the pair (removing either end removes both) |
| `ethtool -S <iface>` | Show per-interface TX/RX statistics (verify the cable carries traffic) |

---

## Lab

### Step 1 — Create a pair in the host namespace

```bash
$ sudo ip link add veth-a type veth peer name veth-b
$ ip link show type veth
12: veth-b@veth-a: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 ... state DOWN
13: veth-a@veth-b: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 ... state DOWN
```

Note the `veth-b@veth-a` naming — the `@` shows you the peer. Both start `DOWN`.

### Step 2 — Move one end into a namespace

```bash
$ sudo ip netns add ns2
$ sudo ip link set veth-b netns ns2

# veth-b is now gone from the host:
$ ip link show veth-b
Device "veth-b" does not exist.

# It is now inside ns2:
$ sudo ip netns exec ns2 ip link show veth-b
12: veth-b@if13: <BROADCAST,MULTICAST> mtu 1500 ... state DOWN
```

The peer index `@if13` tells you the other end is interface index 13 — but in a different namespace, so it can't print the name.

### Step 3 — Bring both ends up and address them

```bash
# Host end
$ sudo ip link set veth-a up
$ sudo ip addr add 10.0.0.1/24 dev veth-a

# Namespace end
$ sudo ip netns exec ns2 ip link set veth-b up
$ sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth-b
```

Check carrier — now that both ends are up, no `NO-CARRIER`:

```bash
$ ip link show veth-a
13: veth-a@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> ... state UP
#                                       ^^^^^^^^   ^^^^^^^^
#                                       LOWER_UP = peer is up, carrier present
```

### Step 4 — Trace a packet across the cable

Open three terminals. Capture on **both ends simultaneously** to prove the packet traverses the cable.

```bash
# Terminal A — capture the host end
$ sudo tcpdump -n -i veth-a icmp

# Terminal B — capture the namespace end
$ sudo ip netns exec ns2 tcpdump -n -i veth-b icmp

# Terminal C — ping from host into the namespace
$ ping -c 1 10.0.0.2
```

Both captures show the same packet:

```
# Terminal A (veth-a):  10.0.0.1 > 10.0.0.2: ICMP echo request
#                       10.0.0.2 > 10.0.0.1: ICMP echo reply
# Terminal B (veth-b):  10.0.0.1 > 10.0.0.2: ICMP echo request   ← same packet, other end
#                       10.0.0.2 > 10.0.0.1: ICMP echo reply
```

The request leaves `veth-a` (TX) and appears on `veth-b` (RX). The reply makes the reverse trip. This is the literal meaning of "a packet entering one end exits the other."

### Step 5 — Prove the carrier dependency

```bash
# Bring the far end down
$ sudo ip netns exec ns2 ip link set veth-b down

# The host end immediately loses carrier:
$ ip link show veth-a
13: veth-a@if12: <BROADCAST,MULTICAST,UP> ... state LOWERLAYERDOWN
#   note: LOWER_UP is GONE — NO-CARRIER condition

# Pings now fail:
$ ping -c 1 -W 1 10.0.0.2
# 100% packet loss — the cable's far end is unplugged
```

Bring it back up and pings resume. This models a real unplugged cable exactly.

### Step 6 — Clean up

```bash
$ sudo ip netns delete ns2     # deletes veth-b (and thus veth-a) automatically
$ ip link show type veth        # veth-a is gone too
```

Deleting a namespace deletes every interface inside it. Because a veth pair cannot exist with only one end, removing `veth-b` removes `veth-a`.

---

## Further Reading

| Topic | Link |
|---|---|
| `veth` — virtual Ethernet device | [man7.org — veth(4)](https://man7.org/linux/man-pages/man4/veth.4.html) |
| `ip-link` — manage interfaces | [man7.org — ip-link(8)](https://man7.org/linux/man-pages/man8/ip-link.8.html) |
| Virtual Ethernet / tunneling overview | [Wikipedia — Tun/Tap](https://en.wikipedia.org/wiki/TUN/TAP) |

---

## Checkpoint

**Q1. Why does a veth always come in a pair? What happens to `veth-a` if you delete `veth-b`?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A veth models a point-to-point cable, and a cable inherently has two ends. The kernel implements it as two interfaces whose transmit and receive paths are cross-connected: TX on one is RX on the other. A single end would have nothing to hand its packets to. Because the two are inseparable halves of one object, deleting `veth-b` automatically deletes `veth-a` — you cannot have a one-ended cable.
</details>

---

**Q2. You bring up `veth-a` and assign it an IP, but `ip link show veth-a` reports `NO-CARRIER` and pings fail. The far end `veth-b` was moved into a namespace. What is the most likely cause?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`veth-b` (the far end) is still `DOWN`. A veth has carrier only when *both* ends are administratively up — exactly like a physical cable needs a powered device on each side. You brought `veth-a` up but forgot to run `ip netns exec ns2 ip link set veth-b up`. Until the far end is up, the near end shows `NO-CARRIER` / `LOWERLAYERDOWN` and drops all traffic.
</details>

---

**Q3. When a container starts, the runtime creates a veth pair, keeps one end on the host, and moves the other into the container's namespace, renaming it `eth0`. Why this design instead of giving the container direct access to the host's physical NIC?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Namespaces isolate the network stack — a container should not see or control the host's physical NIC, its addresses, or its routes. The veth pair provides a controlled, dedicated link across the namespace boundary: the container gets its own `eth0` (one veth end) with its own IP and routes, while the host keeps the other end to wire into a bridge, apply NAT, or enforce firewall rules. It gives connectivity without sharing the physical device, which preserves isolation and lets the host mediate every packet in and out of the container.
</details>

---

## Homework

Build a chain of three namespaces connected by two veth pairs:

```
ns1 (10.0.0.1/24) <--veth--> ns2 (10.0.0.2/24, 10.0.1.2/24) <--veth--> ns3 (10.0.1.3/24)
```

ns2 sits in the middle with one interface on each subnet. From ns1, try to ping ns3 (`10.0.1.3`). It will fail. Explain *why* it fails, and what single thing ns2 would need to do to make it work (you'll build this fully in Phase 5).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The ping fails for two reasons that compound:

1. **ns1 has no route to `10.0.1.0/24`.** ns1 only has a connected route for `10.0.0.0/24`. A packet to `10.0.1.3` matches no route, so it's rejected before it even leaves ns1 — unless ns1 has a default route pointing at ns2 (`10.0.0.2`).

2. **ns2 is not forwarding.** Even if ns1 sends the packet to ns2 as a gateway, ns2 will not pass a packet from one interface to another unless `net.ipv4.ip_forward=1` is set. By default a Linux host is an endpoint, not a router, and silently drops transit packets.

To make it work, ns2 needs to act as a router: enable `sysctl net.ipv4.ip_forward=1`, and ns1/ns3 each need a route to the far subnet via ns2. That is exactly the routing-between-namespaces lab in Lesson 17. A veth cable provides connectivity; it does not provide routing.
</details>
