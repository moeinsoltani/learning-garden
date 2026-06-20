---
title: "Lesson 04 — IP Addressing Fundamentals"
nav_order: 4
parent: "Phase 1: Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 04: IP Addressing Fundamentals

## Concept

An IP address is not a property of a machine — it is a property of a **network interface**. One machine can have many interfaces, and each interface can have many addresses.

The `ip` command is the single tool you will use for everything address-related. It replaced the older `ifconfig` command and is the standard on all modern Linux systems.

```
  lo          127.0.0.1/8       (loopback — software only)
  eth0        10.0.0.5/24       (physical NIC — primary)
              10.0.0.6/24       (same NIC — secondary)
              2001:db8::1/64    (IPv6 — same NIC)
  eth1        10.0.1.1/24       (second physical NIC)
```

Every address lives on a specific interface, has a **prefix length** (the `/24` part), and has a **scope** that tells the kernel how far the address is meant to reach.

---

## How it works

When you run `ip addr add 10.0.0.5/24 dev eth0`, the kernel does three things at once:

1. Adds `10.0.0.5` to the **local routing table** — a hidden table of addresses the machine owns. Incoming packets are delivered to a local process only if their destination IP is in this table.
2. Creates a connected route in the main table: `10.0.0.0/24 dev eth0 proto kernel scope link` — any address in that subnet is directly reachable through this interface.
3. If this is the first address on the interface, it becomes the **primary** address. Any additional ones are **secondary**.

The primary/secondary distinction matters: if you delete the primary address, all secondary addresses on that interface are deleted too. This is a common footgun when you have multiple IPs on one interface.

{: .note }
> **Same subnet on two different interfaces**
> If you assign `10.0.0.5/24` to eth0 and `10.0.0.6/24` to eth1, the routing table gets two entries for the same subnet:
> ```
> 10.0.0.0/24 dev eth0 proto kernel scope link src 10.0.0.5
> 10.0.0.0/24 dev eth1 proto kernel scope link src 10.0.0.6
> ```
> The machine now owns both addresses (both appear in `ip route show table local`), so inbound packets to either are delivered locally. But outbound packets to the rest of the subnet (e.g., `10.0.0.100`) are ambiguous — the kernel has two equal routes and picks one unpredictably. This is a configuration mistake in normal network design. The correct solution for multi-homed setups is policy routing (Lesson 18), which lets you tell the kernel exactly which table to use per source address. Contrast this with the primary/secondary case: two addresses from the same subnet on *one* interface is fine, because there is only one route and no ambiguity.

{: .note }
> **Route ≠ ownership**
> The connected route `10.0.0.0/24 dev eth0` only tells the kernel *which interface to send packets out of*. It does not mean the machine accepts every address in that subnet as its own. If you send a packet to `10.0.0.6` and only `10.0.0.5` is assigned, the kernel routes the packet out eth0 and sends an ARP request for `10.0.0.6` — looking for another machine on that network that owns it. The local machine never accepts the packet for itself because `10.0.0.6` is not in its local table.
>
> You can inspect the local table directly:
> ```bash
> ip route show table local
> # local 10.0.0.5 dev eth0 proto kernel scope host src 10.0.0.5
> # Only .5 is listed — .6 is absent, so the machine won't claim it
> ```

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip link show` | List interfaces: name, state (UP/DOWN), MTU, MAC address |
| `ip addr show` | List all interfaces with their IP addresses |
| `ip addr show dev <iface>` | Show addresses on one interface only |
| `ip addr add <ip/prefix> dev <iface>` | Add an IP address to an interface |
| `ip addr del <ip/prefix> dev <iface>` | Remove an IP address from an interface |
| `ip -6 addr show` | Show IPv6 addresses only |
| `ip addr flush dev <iface>` | Remove all addresses from an interface |
| `ip route show table local` | Show the kernel's local address ownership table |

---

## Lab

### Step 1 — Create a namespace to experiment in

You will do all of this inside a throw-away namespace so you cannot accidentally break your host network.

```bash
$ sudo ip netns add lab
$ sudo ip netns exec lab ip link set lo up
```

### Step 2 — Examine the interface before any address

```bash
$ sudo ip netns exec lab ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

Key fields:

| Field | Meaning |
|---|---|
| `<LOOPBACK,UP,LOWER_UP>` | Flags: this interface is up and has carrier |
| `mtu 65536` | Maximum Transmission Unit — largest packet this interface accepts |
| `state UNKNOWN` | For loopback, state is always UNKNOWN (it has no real carrier concept) |
| `link/loopback 00:00:00:00:00:00` | MAC address — loopback uses all-zeros |

No IP addresses yet — `ip link show` only shows L2 (interface) information, not L3 (address) information.

### Step 3 — Add a primary IPv4 address

```bash
$ sudo ip netns exec lab ip addr add 192.168.10.1/24 dev lo
$ sudo ip netns exec lab ip addr show lo
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN ...
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet 192.168.10.1/24 scope global lo
       valid_lft forever preferred_lft forever
```

Two addresses now: `127.0.0.1/8` (added automatically when lo came up) and your new `192.168.10.1/24`.

{: .note }
> **Reading `ip addr show` output**
>
> | Field | Meaning |
> |---|---|
> | `inet` | This is an IPv4 address (`inet6` would be IPv6) |
> | `scope host` | Address is only meaningful on this machine — never routed |
> | `scope global` | Address is routable — meaningful to the outside world |
> | `scope link` | Address is only meaningful on this L2 segment (link-local) |
> | `valid_lft forever` | This address never expires (static assignment) |

### Step 4 — Add a secondary address on the same interface

```bash
$ sudo ip netns exec lab ip addr add 192.168.10.2/24 dev lo
$ sudo ip netns exec lab ip addr show lo
    inet 127.0.0.1/8 scope host lo
    inet 192.168.10.1/24 scope global lo
    inet 192.168.10.2/24 scope global secondary lo
#                                    ^^^^^^^^^
#                                    kernel labels it secondary
```

`192.168.10.1` is the primary because it was added first. `192.168.10.2` is secondary.

### Step 5 — Delete the primary address and observe the secondary

```bash
$ sudo ip netns exec lab ip addr del 192.168.10.1/24 dev lo
$ sudo ip netns exec lab ip addr show lo
    inet 127.0.0.1/8 scope host lo
    # 192.168.10.2 is gone too — deleted along with the primary
```

Both addresses disappeared. This is the primary/secondary footgun. If you need to keep secondary addresses alive, delete them first, then delete the primary — or use a different approach entirely (separate interfaces, or assign the secondary first).

### Step 6 — Add an IPv6 address

```bash
$ sudo ip netns exec lab ip addr add 2001:db8::1/64 dev lo
$ sudo ip netns exec lab ip -6 addr show lo
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN qlen 1000
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
    inet6 2001:db8::1/64 scope global
       valid_lft forever preferred_lft forever
```

IPv4 and IPv6 addresses coexist on the same interface. The `-6` flag limits `ip addr show` to IPv6 only.

{: .note }
> **`2001:db8::/32` is the documentation prefix** — the IPv6 equivalent of `192.0.2.0/24` or `10.0.0.0/8` for examples. It is globally routable in format but reserved for documentation and will never appear on the real internet. Safe to use in labs.

### Step 7 — Check what routes appeared automatically

```bash
$ sudo ip netns exec lab ip route show
192.168.10.0/24 dev lo proto kernel scope link src 192.168.10.1
# (after step 6, you will also see:)
$ sudo ip netns exec lab ip -6 route show
2001:db8::/64 dev lo proto kernel metric 256 pref medium
```

Every `ip addr add` created a connected route automatically. Every `ip addr del` removed it.

### Step 8 — Flush all addresses from an interface

```bash
$ sudo ip netns exec lab ip addr flush dev lo
$ sudo ip netns exec lab ip addr show lo
1: lo: <LOOPBACK,UP,LOWER_UP> ...
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    # no inet lines — all addresses gone
```

`flush` removes all addresses in one command, regardless of primary/secondary status.

### Step 9 — Clean up

```bash
$ sudo ip netns delete lab
```

---

## Address Scopes

Scope tells the kernel **how far an address is allowed to travel** — whether packets destined for it can be routed beyond the local machine or the local link.

| Scope | What it means | Example |
|---|---|---|
| `host` | Only valid on this machine. Never put on the wire, never forwarded. | `127.0.0.1/8` on lo |
| `link` | Only valid on this directly connected L2 segment. Not routed beyond the local switch. | `169.254.x.x/16` (link-local, assigned when DHCP fails) |
| `global` | Routable anywhere. This is what all normal addresses use. | Any address you manually assign |

Scope also appears on **routes**, not just addresses. The connected route the kernel auto-creates has `scope link`:

```
10.0.0.0/24 dev eth0 proto kernel scope link src 10.0.0.5
```

`scope link` on a route means "the destination is directly reachable on this link — no gateway needed." A route with `scope global` would mean a gateway is required.

`scope link` and ARP are two separate mechanisms that work together:

- **`scope link`** is a routing decision — the kernel sends the packet directly onto the interface without consulting a gateway. Routers are also expected to drop packets destined for `scope link` addresses rather than forwarding them.
- **ARP** is what makes direct delivery actually work — once the kernel decides to send directly out an interface, it still needs to know the destination's MAC address. ARP resolves that. ARP does not know about scope; it just maps IPs to MACs on whatever segment you ask it to.

The sequence when a packet hits a `scope link` route:

```
1. Kernel matches route → "scope link, send out eth0 directly"
2. Kernel checks ARP cache for the destination IP's MAC
   → hit:  sends packet immediately in an Ethernet frame to that MAC
   → miss: broadcasts ARP request ("who has <IP>?") on eth0
           waits for a reply
           sends the packet once the MAC is known
           if no reply comes → nobody on the link has that IP → packet dropped
```

You can see scope on every address in `ip addr show`:

```
inet 127.0.0.1/8 scope host lo         ← never leaves this machine
inet 169.254.1.1/16 scope link eth0    ← never leaves this switch
inet 10.0.0.5/24 scope global eth0     ← routable anywhere
```

{: .note }
> **How does the kernel decide which scope to assign?**
> It uses the address range. Certain ranges are reserved by IANA standards and the kernel has built-in rules for them:
>
> | Address range | Scope | Standard |
> |---|---|---|
> | `127.0.0.0/8` | `host` | [RFC 1122](https://en.wikipedia.org/wiki/RFC_1122) — loopback |
> | `169.254.0.0/16` | `link` | [RFC 3927](https://en.wikipedia.org/wiki/Link-local_address) — IPv4 link-local |
> | `fe80::/10` (IPv6) | `link` | [RFC 4291](https://en.wikipedia.org/wiki/IPv6_address#Link-local_addresses) — IPv6 link-local |
> | `::1/128` (IPv6) | `host` | IPv6 loopback |
> | Everything else | `global` | No reservation — assumed routable |
>
> `10.0.0.5` gets `global` simply because `10.0.0.0/8` has no special reservation — it is private but still routable within networks that use it. `169.254.1.1` gets `link` because its range is an IANA-reserved link-local block that must never be routed.
>
> You can override the scope manually if needed: `ip addr add 10.0.0.5/24 dev eth0 scope link` — but this is rarely done in practice.

---

## Further Reading

| Topic | Link |
|---|---|
| `ip-address` — manage IP addresses | [man7.org — ip-address(8)](https://man7.org/linux/man-pages/man8/ip-address.8.html) |
| `ip-link` — manage network interfaces | [man7.org — ip-link(8)](https://man7.org/linux/man-pages/man8/ip-link.8.html) |
| CIDR notation | [Wikipedia — Classless Inter-Domain Routing](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) |
| IPv6 addressing | [Wikipedia — IPv6 address](https://en.wikipedia.org/wiki/IPv6_address) |
| Link-local address | [Wikipedia — Link-local address](https://en.wikipedia.org/wiki/Link-local_address) |
| IPv4 link-local (RFC 3927) | [Wikipedia — RFC 3927](https://en.wikipedia.org/wiki/Link-local_address#IPv4) |

---

## Checkpoint

**Q1. What is the difference between `ip link show` and `ip addr show`? What information does each one give you that the other does not?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`ip link show` shows Layer 2 information: interface name, state (UP/DOWN), MTU, and MAC address. It tells you nothing about IP addresses. `ip addr show` shows everything `ip link show` does, plus all Layer 3 addresses (IPv4 and IPv6) assigned to each interface, along with their prefix lengths and scopes. Use `ip link show` to check if an interface exists and is up; use `ip addr show` to check if it has the right IP address.
</details>

---

**Q2. You add `10.0.0.1/24` to eth0, then add `10.0.0.2/24` to eth0. Then you delete `10.0.0.1/24`. What happens to `10.0.0.2/24`?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`10.0.0.2/24` is deleted automatically along with `10.0.0.1/24`. The first address added becomes the primary; every subsequent address on the same interface is secondary. When you delete the primary, the kernel removes all secondary addresses on that interface too. To avoid this, delete the secondary addresses first, then delete the primary.
</details>

---

**Q3. After you run `ip addr add 10.1.2.1/24 dev eth0`, you immediately run `ip route show` and see a new route `10.1.2.0/24 dev eth0 proto kernel scope link`. You never ran `ip route add`. Where did that route come from?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The kernel adds it automatically whenever you assign an address with a prefix length. The logic is: if eth0 owns 10.1.2.1 and the prefix is /24, then any address in 10.1.2.0/24 must be directly reachable through eth0 — no gateway needed. The `proto kernel` label confirms the kernel generated it, not a user or routing daemon. Deleting the address also removes this route.
</details>

---

## Homework

Inside a namespace, add the address `10.0.0.1/24` to lo. Then add `10.0.0.50/24` to lo (it will appear as secondary). Now add a *different subnet* address: `172.16.0.1/16` to lo.

Run `ip addr show lo` and `ip route show`. Then delete `10.0.0.1/24` (the primary for the 10.0.0.0/24 subnet).

Answer: which addresses survived? Which routes survived? Explain why each one behaved the way it did.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
After deleting 10.0.0.1/24:
- `10.0.0.50/24` is gone — it was a secondary on the same interface under the same primary, so it was deleted with the primary.
- `172.16.0.1/16` survives — it is on a completely different subnet. Primary/secondary relationships are scoped to addresses in the same subnet on the same interface. 172.16.0.1 is unaffected by deleting something in 10.0.0.0/24.

Routes: the 10.0.0.0/24 connected route is gone (the addresses it was based on are gone). The 172.16.0.0/16 connected route remains (the address backing it is still there).
</details>
