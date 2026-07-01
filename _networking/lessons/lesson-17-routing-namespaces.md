---
title: "Lesson 17 — Routing Between Namespaces"
nav_order: 17
parent: "Phase 5: Routing"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 17: Routing Between Namespaces

## Concept

So far each namespace has been an endpoint. Now you'll make one act as a **router** — a box with a foot in two networks that forwards packets between them. This is the moment Linux stops being "a host" and becomes "the network."

```
   ns1                  router-ns                    ns2
  10.0.1.2/24      10.0.1.1/24   10.0.2.1/24      10.0.2.2/24
  ┌──────┐         ┌──────────────────────┐       ┌──────┐
  │ vethA│═════════│vethA'          vethB'│═══════│ vethB│
  └──────┘  net 1  └──────────────────────┘ net 2 └──────┘
       │                    │  │                       │
   default via          forwards between          default via
   10.0.1.1             the two subnets            10.0.2.1
```

ns1 and ns2 are in *different* subnets and have no direct link. The router namespace sits between them with an interface in each subnet. For traffic to cross, two things must be true: the router must **forward**, and each endpoint must know to send cross-subnet traffic **to the router**.

---

## How it works

A Linux host, by default, does **not** forward packets between its interfaces — it's an endpoint, not a router. A packet that arrives destined for somewhere else is dropped. You flip it into a router with one sysctl:

```bash
sysctl net.ipv4.ip_forward=1
```

With forwarding on, when a packet arrives whose destination isn't local, the router does a normal route lookup, decrements the TTL, rewrites the L2 MAC headers for the next hop, and sends it out the appropriate interface.

The endpoints need **default routes** (or specific routes) pointing at the router as their gateway. ns1 must know "to reach 10.0.2.0/24, send to 10.0.1.1." Without that, ns1 has no route to ns2's subnet and the packet never even leaves.

{: .note }
> **Forwarding is per-namespace**
> `net.ipv4.ip_forward` is a *namespaced* sysctl — setting it inside the router namespace affects only that namespace, not the host. This is why containers can be made into routers without turning the host into one. (More on namespaced vs global sysctls in Lesson 35.)

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `sysctl net.ipv4.ip_forward=1` | Enable IPv4 forwarding (turn the box into a router) |
| `ip netns exec <ns> sysctl -w net.ipv4.ip_forward=1` | Enable it inside a namespace |
| `ip route add default via <gw>` | Point an endpoint at its gateway |
| `ip route add <subnet> via <gw>` | Add a specific cross-subnet route |
| `cat /proc/sys/net/ipv4/ip_forward` | Check forwarding state (`1` = on) |

---

## Lab

We'll build the three-namespace router topology above and trace a packet across both hops.

### Step 1 — Create the namespaces and veth links

```bash
$ sudo ip netns add ns1
$ sudo ip netns add router
$ sudo ip netns add ns2

# Link ns1 <-> router
$ sudo ip link add vethA type veth peer name vethA-r
$ sudo ip link set vethA netns ns1
$ sudo ip link set vethA-r netns router

# Link router <-> ns2
$ sudo ip link add vethB-r type veth peer name vethB
$ sudo ip link set vethB-r netns router
$ sudo ip link set vethB netns ns2
```

### Step 2 — Address everything

```bash
# ns1 side — subnet 10.0.1.0/24
$ sudo ip netns exec ns1 ip link set vethA up
$ sudo ip netns exec ns1 ip addr add 10.0.1.2/24 dev vethA

# router side — one foot in each subnet
$ sudo ip netns exec router ip link set vethA-r up
$ sudo ip netns exec router ip addr add 10.0.1.1/24 dev vethA-r
$ sudo ip netns exec router ip link set vethB-r up
$ sudo ip netns exec router ip addr add 10.0.2.1/24 dev vethB-r

# ns2 side — subnet 10.0.2.0/24
$ sudo ip netns exec ns2 ip link set vethB up
$ sudo ip netns exec ns2 ip addr add 10.0.2.2/24 dev vethB
```

### Step 3 — Confirm it's broken before forwarding/routes

```bash
$ sudo ip netns exec ns1 ping -c 1 -W 1 10.0.2.2
# Network is unreachable — ns1 has no route to 10.0.2.0/24
```

### Step 4 — Give the endpoints default routes

```bash
$ sudo ip netns exec ns1 ip route add default via 10.0.1.1
$ sudo ip netns exec ns2 ip route add default via 10.0.2.1

# Still fails — the router isn't forwarding yet:
$ sudo ip netns exec ns1 ping -c 1 -W 1 10.0.2.2
# 100% packet loss
```

The packet now reaches the router (ns1 knows its gateway), but the router silently drops it.

### Step 5 — Enable forwarding on the router

```bash
$ sudo ip netns exec router sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1

$ sudo ip netns exec ns1 ping -c 2 10.0.2.2
64 bytes from 10.0.2.2: icmp_seq=1 ttl=63 time=0.10 ms
```

It works. Note `ttl=63` — the router **decremented the TTL** from 64 to 63 as it forwarded. That one-off decrement is the fingerprint of a routed (not bridged) hop.

### Step 6 — Trace the packet on both hops simultaneously

Open two captures inside the router, one per interface:

```bash
# Terminal A — the ns1-side interface
$ sudo ip netns exec router tcpdump -n -i vethA-r icmp

# Terminal B — the ns2-side interface
$ sudo ip netns exec router tcpdump -n -i vethB-r icmp

# Terminal C — ping
$ sudo ip netns exec ns1 ping -c 1 10.0.2.2
```

You'll see the *same* ICMP echo request appear first on `vethA-r` (arriving from ns1) and then on `vethB-r` (leaving toward ns2), and the reply make the reverse trip. This is a packet being routed hop-by-hop, visible on both legs of the router.

### Step 7 — Clean up

```bash
$ sudo ip netns delete ns1
$ sudo ip netns delete router
$ sudo ip netns delete ns2
```

---

## Further Reading

| Topic | Link |
|---|---|
| IP forwarding | [Wikipedia — IP forwarding](https://en.wikipedia.org/wiki/IP_forwarding) |
| Router (computing) | [Wikipedia — Router](https://en.wikipedia.org/wiki/Router_(computing)) |
| Time to live (TTL) | [Wikipedia — Time to live](https://en.wikipedia.org/wiki/Time_to_live) |
| `ip-sysctl` networking knobs | [kernel.org — ip-sysctl](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt) |

---

## Checkpoint

**Q1. After you give ns1 a default route via the router, ns1→ns2 still fails. The router has an address in both subnets. What single thing is still missing?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
IP forwarding is not enabled on the router. By default a Linux host treats packets destined for other hosts as not-its-problem and drops them — it acts as an endpoint, not a router. Even though ns1 correctly sends the packet to the router (it has the default route) and the router has a foot in both subnets, the router won't pass the packet from one interface to the other until `net.ipv4.ip_forward=1` is set in the router namespace. Once forwarding is enabled, the router does a route lookup, decrements the TTL, and forwards the packet to ns2.
</details>

---

**Q2. After routing works, you notice replies from ns2 arrive at ns1 with `ttl=63` instead of `ttl=64`. What does that tell you?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It tells you the packet was **routed through exactly one L3 hop** (the router namespace). Every router that forwards an IP packet decrements its TTL by one before sending it on — this prevents packets from looping forever. ns2 sends its reply with the default TTL of 64; the router decrements it to 63 as it forwards toward ns1. So a TTL one less than the origin's default is the signature of a single routed hop. If you saw `ttl=62` you'd know it passed through two routers. (A *bridged* hop, by contrast, leaves the TTL untouched, which is one way to tell switching from routing.)
</details>

---

**Q3. Why is it safe to turn a container/namespace into a router with `ip_forward=1` without affecting the host's forwarding behavior?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because `net.ipv4.ip_forward` is a **namespaced** sysctl: each network namespace has its own independent copy of the setting. Writing `1` to it inside the router namespace turns *that namespace* into a router but leaves the host's value (and every other namespace's) unchanged. This isolation is what lets container runtimes and CNI plugins build routers, NAT gateways, and forwarding paths inside containers without compromising the host's posture. (Not all sysctls are namespaced — some, like socket buffer sizes, are global; you'll explore which is which in Lesson 35.)
</details>

---

## Homework

Extend the topology to a **four**-namespace chain: `ns1 — routerA — routerB — ns2`, where routerA and routerB are each routers on a middle transit subnet. Make ns1 ping ns2 across *two* routed hops. Then capture at ns2 and confirm the TTL. Finally, break it deliberately by disabling forwarding on routerB and use `ip route get` / tcpdump to pinpoint exactly where the packet dies.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Topology: ns1 (`10.0.1.2`) — routerA (`10.0.1.1` / `10.0.9.1`) — [transit `10.0.9.0/24`] — routerB (`10.0.9.2` / `10.0.2.1`) — ns2 (`10.0.2.2`). Required config:

- Enable forwarding on **both** routerA and routerB.
- ns1 default via `10.0.1.1` (routerA); ns2 default via `10.0.2.1` (routerB).
- routerA needs a route to ns2's subnet: `ip route add 10.0.2.0/24 via 10.0.9.2`.
- routerB needs a route back to ns1's subnet: `ip route add 10.0.1.0/24 via 10.0.9.1`. (Return path matters — both directions need routes.)

When it works, an echo arriving at ns2 shows **ttl=62** (started at 64, decremented once by routerA and once by routerB — two routed hops).

To break and diagnose: disable forwarding on routerB (`sysctl -w net.ipv4.ip_forward=0`). The ping fails. Capturing on routerB's interfaces shows the echo request *arriving* on its transit-side interface (`10.0.9.2`) but **never leaving** the ns2-side interface — the packet dies inside routerB because forwarding is off. `ip route get 10.0.2.2` on routerB would still show a valid route (routing isn't the problem), which combined with "arrives but doesn't leave" isolates the fault to forwarding being disabled. This is the core diagnostic skill: capture on each hop's ingress and egress, find the interface where the packet last appears, and that node is where it's being dropped.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 18 — Policy Routing & Multiple Tables →](lesson-18-policy-routing){: .btn .btn-primary }
