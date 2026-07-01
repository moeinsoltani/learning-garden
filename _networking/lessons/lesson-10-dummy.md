---
title: "Lesson 10 — Dummy Interfaces"
nav_order: 10
parent: "Phase 2: Virtual Interfaces"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 10: Dummy Interfaces

## Concept

A **dummy interface** is a virtual interface that is always up, has no peer, and goes nowhere. Anything you send *into* it is dropped — like `/dev/null` for packets. So why create one? Because of its **IP address**, not its ability to carry traffic.

A dummy gives you a stable, always-up IP that does not depend on any physical link being present or any cable being plugged in. It is the standard way to pin a reliable identity to a host.

```
   ┌──────────────┐
   │   dummy0      │   state: always UP
   │ 10.255.0.1/32 │   carries no traffic — but the IP is reachable
   └──────────────┘   via routing to other interfaces
          ⌧  packets sent OUT of dummy0 are discarded
```

---

## How it works

When you assign an address to a dummy interface and advertise/route it, other machines can reach that IP — the packets arrive on a *real* interface and get delivered locally because the address is owned by the host (it's in the local table, just like Lesson 4 explained). The dummy is just where the address "lives" so it isn't tied to the up/down state of `eth0`.

This matters because a normal interface's addresses become unreachable if that interface goes down. A dummy never goes down, so an address on it is a rock-solid endpoint.

Common real-world uses:

- **Stable router ID / loopback IP** for dynamic routing (OSPF/BGP). Routers advertise a `/32` on a dummy (or loopback) so the router is reachable by *that* IP regardless of which physical link is currently up.
- **Kubernetes / load-balancer VIPs** — a service IP that should exist independent of any NIC.
- **Anycast** — the same `/32` configured on many hosts; routing decides which one you reach.
- **Testing** — a guaranteed-present local address to bind services to.

{: .note }
> **Dummy vs loopback — what's the difference?**
> Both are software-only and always available. The differences: you can create *many* independent dummy interfaces (`dummy0`, `dummy1`, …) each with its own name, MAC, and addresses, whereas there is conventionally one `lo` per namespace. And loopback addresses (`127.0.0.0/8`) have **host scope** and are never routable off the machine, while a dummy holds **global-scope** addresses that *are* meant to be advertised and reached from elsewhere. Use `lo` for "never leaves this machine"; use a dummy for "a routable identity not tied to a physical NIC."

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip link add dummy0 type dummy` | Create a dummy interface |
| `ip link set dummy0 up` | Bring it up (then it stays up forever) |
| `ip addr add 10.255.0.1/32 dev dummy0` | Assign a stable address |
| `ip link del dummy0` | Remove it |

---

## Lab

### Step 1 — Create a dummy interface in a namespace

```bash
$ sudo ip netns add lab
$ sudo ip netns exec lab ip link set lo up
$ sudo ip netns exec lab ip link add dummy0 type dummy
$ sudo ip netns exec lab ip link set dummy0 up
$ sudo ip netns exec lab ip addr add 10.255.0.1/32 dev dummy0
```

### Step 2 — Confirm it is up and owns the address

```bash
$ sudo ip netns exec lab ip addr show dummy0
N: dummy0: <BROADCAST,NOARP,UP,LOWER_UP> mtu 1500 ... state UNKNOWN
    link/ether 1a:2b:3c:4d:5e:6f brd ff:ff:ff:ff:ff:ff
    inet 10.255.0.1/32 scope global dummy0
       valid_lft forever preferred_lft forever
```

Note `UP,LOWER_UP` — it has "carrier" with no peer. Unlike a veth, it never reports NO-CARRIER. Note also it has a real MAC (`link/ether`), unlike loopback's all-zeros.

### Step 3 — Prove it owns the address locally

```bash
$ sudo ip netns exec lab ping -c 1 10.255.0.1
64 bytes from 10.255.0.1: icmp_seq=1 ttl=64 time=0.04 ms

$ sudo ip netns exec lab ip route show table local | grep 10.255.0.1
local 10.255.0.1 dev dummy0 proto kernel scope host src 10.255.0.1
```

The host answers for `10.255.0.1` because it's in the local table — exactly the ownership mechanism from Lesson 4.

### Step 4 — Make it reachable from another namespace

Wire a veth between `lab` and a `client` namespace, then route to the dummy's `/32`.

```bash
$ sudo ip netns add client
$ sudo ip link add veth-lab type veth peer name veth-cli
$ sudo ip link set veth-lab netns lab
$ sudo ip link set veth-cli netns client

$ sudo ip netns exec lab ip link set veth-lab up
$ sudo ip netns exec lab ip addr add 192.168.1.1/24 dev veth-lab
$ sudo ip netns exec client ip link set veth-cli up
$ sudo ip netns exec client ip addr add 192.168.1.2/24 dev veth-cli

# client needs a route to reach the dummy's /32 via lab's veth address
$ sudo ip netns exec client ip route add 10.255.0.1/32 via 192.168.1.1

$ sudo ip netns exec client ping -c 1 10.255.0.1
64 bytes from 10.255.0.1: icmp_seq=1 ttl=64 time=0.06 ms
```

The dummy's address is now reachable from another namespace — even though no traffic ever flows *through* `dummy0`. The packet arrives on `veth-lab` and is delivered locally because `lab` owns `10.255.0.1`.

### Step 5 — Prove the dummy survives a "link" event

There's no physical link to flap, which is the whole point. Even if you bring the veth down:

```bash
$ sudo ip netns exec lab ip link set veth-lab down
$ sudo ip netns exec lab ip addr show dummy0
N: dummy0: <BROADCAST,NOARP,UP,LOWER_UP> ...   ← still UP, address still present
```

In a multi-link setup, the dummy IP would remain reachable via any *other* still-up path. That stability is exactly why routers use it for their router ID.

### Step 6 — Clean up

```bash
$ sudo ip netns delete lab
$ sudo ip netns delete client
```

---

## Further Reading

| Topic | Link |
|---|---|
| Dummy network interface | [Wikipedia — Dummy network interface](https://en.wikipedia.org/wiki/Network_interface_controller) |
| Anycast | [Wikipedia — Anycast](https://en.wikipedia.org/wiki/Anycast) |
| Router ID / loopback in routing | [Wikipedia — Router (computing)](https://en.wikipedia.org/wiki/Router_(computing)) |
| `ip-link` | [man7.org — ip-link(8)](https://man7.org/linux/man-pages/man8/ip-link.8.html) |

---

## Checkpoint

**Q1. A dummy interface drops every packet you send out of it. So why create an interface that goes nowhere?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You create it for the *address*, not the data path. A dummy gives you a stable, always-up, routable IP that isn't tied to any physical NIC's link state. Other machines reach that IP via normal routing — the packets arrive on a real interface and are delivered locally because the host owns the address. The value is that the address never disappears when a cable is unplugged or a NIC goes down, which makes it ideal for a router's ID, a service VIP, or an anycast address. The "goes nowhere" part is irrelevant; nobody sends traffic *through* it, they send traffic *to* the address it hosts.
</details>

---

**Q2. Both `lo` and a dummy interface are software-only and always available. Give two concrete differences and when each matters.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
1. **Scope / routability.** Loopback addresses (`127.0.0.0/8`, `::1`) have host scope and can never be routed off the machine. A dummy holds global-scope addresses meant to be advertised and reached from other hosts. So if you need a *reachable* stable IP, use a dummy; if you need a strictly-local-only address, use loopback.
2. **Multiplicity and identity.** You can create many independent dummy interfaces, each with its own name, MAC address, and set of addresses, whereas there is conventionally a single `lo`. Dummies have a real Ethernet MAC; loopback's MAC is all zeros. This matters when you need several distinct routable endpoints (e.g., multiple VIPs or anycast addresses) cleanly separated on one host.
</details>

---

**Q3. Why do routers running OSPF or BGP typically put their router ID / advertised loopback address on a dummy (or loopback) interface rather than on a physical link like eth0?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because a physical interface's address becomes unreachable the moment that interface goes down, and a router may have several physical links any of which could fail. If the router's identity lived on `eth0` and `eth0` died, the router would be unreachable by that IP even though it's still up and reachable through `eth1`. By placing the router ID on an always-up dummy/loopback and advertising it as a `/32`, the router stays reachable by that single stable address via *any* surviving physical path. The routing protocol picks whichever physical link is currently working to deliver packets to the `/32`. This decouples the router's identity from the health of any one link — exactly the stability a dummy provides.
</details>

---

## Homework

Create *two* dummy interfaces in one namespace with addresses `10.255.0.1/32` and `10.255.0.2/32`. From a client namespace connected by veth, add routes to both and ping each. Then delete `dummy0` (holding `.1`) and observe what happens to reachability of `.1` vs `.2`. Explain why the behavior differs from the primary/secondary address footgun you saw in Lesson 4.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Both `.1` and `.2` are reachable from the client while their dummies exist. When you delete `dummy0`, only `10.255.0.1` becomes unreachable (its interface and the local-table ownership entry are gone, and the client's route to it now leads nowhere). `10.255.0.2` on `dummy1` is completely unaffected and remains reachable.

This differs from the Lesson 4 primary/secondary footgun because there the two addresses lived on the *same* interface, so deleting the primary cascaded and removed the secondary. Here the addresses are on *separate* interfaces (`dummy0` and `dummy1`), so they're independent — deleting one interface has no effect on the other. The lesson: using separate dummy interfaces for separate VIPs avoids the cascade-delete behavior entirely, which is one reason production setups often give each VIP its own dummy.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 11 — Linux Bridges →](lesson-11-bridges){: .btn .btn-primary }
