---
title: "Lesson 16 — Routing Fundamentals"
nav_order: 16
parent: "Phase 5: Routing"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 16: Routing Fundamentals

## Concept

A **route** answers one question: "given a destination IP, which interface do I send the packet out of, and to which next-hop?" The kernel keeps a table of routes, and for every packet it performs a lookup to decide. Understanding that lookup is the heart of L3 networking.

```
   Packet to 8.8.8.8 — kernel scans the routing table for the BEST match:

   10.0.0.0/24    dev eth0    scope link            (covers 10.0.0.x)
   192.168.0.0/16 via 10.0.0.9                       (covers 192.168.x.x)
   0.0.0.0/0      via 10.0.0.1  dev eth0  (default)  (covers EVERYTHING)
                  └────────────────────────────────┘
   8.8.8.8 matches only the default → send via gateway 10.0.0.1 out eth0
```

The selection rule is **longest prefix match**: the most *specific* route that contains the destination wins. A `/32` (single host) beats a `/24` beats the `0.0.0.0/0` default.

---

## How it works

For each packet the kernel finds every route whose prefix contains the destination, then picks the winner by these tiebreakers, in order:

1. **Longest prefix** — more specific wins. `/32` > `/24` > `/16` > `/0`.
2. **Metric** — among equal-length matches, **lower metric wins** (metric = "cost"/preference).
3. Other factors (table priority via policy rules — Lesson 18).

Route types you can install:

| Type | Effect |
|---|---|
| `unicast` (default) | Normal forwarding to a next-hop or out an interface. |
| `blackhole` | Silently drop matching packets. No error returned to the sender. |
| `unreachable` | Drop and send back ICMP "destination unreachable." |
| `prohibit` | Drop and send back ICMP "administratively prohibited." |

The single most useful diagnostic command is `ip route get <dst>` — it asks the kernel to *run the lookup* and tell you exactly which route it would use, including the source address it would pick. Never guess routing; ask the kernel.

{: .note }
> **`scope link` vs `via` (gateway)**
> A route with `scope link` (e.g. the auto-created connected route) means the destination is *directly reachable* on that interface — the kernel ARPs for the destination itself. A route with `via <gateway>` means "not directly reachable; hand the packet to this next-hop router" — the kernel ARPs for the *gateway's* MAC, not the destination's. This connects back to Lesson 5: routing decides *who* to ARP for.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip route show` | Show the main routing table |
| `ip route add <prefix> via <gw>` | Add a route via a gateway |
| `ip route add <prefix> dev <iface>` | Add a route out an interface (directly connected) |
| `ip route add default via <gw>` | Add the default route (`0.0.0.0/0`) |
| `ip route del <prefix>` | Delete a route |
| `ip route get <dst>` | Ask the kernel which route it would use for `<dst>` |
| `ip route add blackhole <prefix>` | Silently drop traffic to a prefix |
| `ip rule list` | Show routing policy rules (which table is consulted) |
| `ip route show table <id>` | Show a specific routing table |

---

## Lab

### Step 1 — Set up a namespace with two routes to compare

```bash
$ sudo ip netns add lab
$ sudo ip netns exec lab ip link set lo up
$ sudo ip netns exec lab ip link add eth0 type dummy
$ sudo ip netns exec lab ip link set eth0 up
$ sudo ip netns exec lab ip addr add 10.0.0.5/24 dev eth0
```

Adding the address auto-created the connected route. Check it:

```bash
$ sudo ip netns exec lab ip route show
10.0.0.0/24 dev eth0 proto kernel scope link src 10.0.0.5
```

### Step 2 — Add a default route and a more-specific route

```bash
# Default: everything not otherwise matched goes via 10.0.0.1
$ sudo ip netns exec lab ip route add default via 10.0.0.1

# More specific: 10.0.1.0/24 goes via a different gateway 10.0.0.9
$ sudo ip netns exec lab ip route add 10.0.1.0/24 via 10.0.0.9

$ sudo ip netns exec lab ip route show
default via 10.0.0.1 dev eth0
10.0.0.0/24 dev eth0 proto kernel scope link src 10.0.0.5
10.0.1.0/24 via 10.0.0.9 dev eth0
```

### Step 3 — Ask the kernel to resolve specific destinations

```bash
# Inside the connected subnet → directly reachable (scope link)
$ sudo ip netns exec lab ip route get 10.0.0.50
10.0.0.50 dev eth0 src 10.0.0.5 ...

# Matches the /24 specific route → via 10.0.0.9
$ sudo ip netns exec lab ip route get 10.0.1.5
10.0.1.5 via 10.0.0.9 dev eth0 src 10.0.0.5 ...

# Matches nothing specific → default via 10.0.0.1
$ sudo ip netns exec lab ip route get 8.8.8.8
8.8.8.8 via 10.0.0.1 dev eth0 src 10.0.0.5 ...
```

Three destinations, three different decisions — all explained by longest-prefix match.

### Step 4 — Demonstrate longest prefix match with an overlapping /32

```bash
# Add a host route (/32) that overrides everything for one IP
$ sudo ip netns exec lab ip route add 8.8.8.8/32 via 10.0.0.9

$ sudo ip netns exec lab ip route get 8.8.8.8
8.8.8.8 via 10.0.0.9 dev eth0 ...     # the /32 beat the default /0
```

The `/32` is the most specific possible route, so it wins over the `0.0.0.0/0` default.

### Step 5 — Compare blackhole vs unreachable

```bash
# Blackhole: silent drop
$ sudo ip netns exec lab ip route add blackhole 172.16.0.0/16
$ sudo ip netns exec lab ip route get 172.16.0.1
RTNETLINK answers: Invalid argument
# (kernel reports the route is a blackhole; a real send is silently dropped)

# Unreachable: drop + ICMP error
$ sudo ip netns exec lab ip route add unreachable 172.17.0.0/16
$ sudo ip netns exec lab ping -c 1 172.17.0.1
ping: ... Destination unreachable
```

`blackhole` drops with no feedback (good for silently absorbing traffic, e.g. anti-DDoS). `unreachable`/`prohibit` actively reject with an ICMP error, so the sender learns immediately.

### Step 6 — Clean up

```bash
$ sudo ip netns delete lab
```

---

## Further Reading

| Topic | Link |
|---|---|
| IP routing | [Wikipedia — IP routing](https://en.wikipedia.org/wiki/IP_routing) |
| Longest prefix match | [Wikipedia — Longest prefix match](https://en.wikipedia.org/wiki/Longest_prefix_match) |
| Default route | [Wikipedia — Default route](https://en.wikipedia.org/wiki/Default_route) |
| `ip-route` | [man7.org — ip-route(8)](https://man7.org/linux/man-pages/man8/ip-route.8.html) |

---

## Checkpoint

**Q1. You have a `/24` route for `10.0.1.0/24` via gateway A, and a `0.0.0.0/0` default via gateway B. Traffic to `10.0.1.5` goes where, and why?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It goes via **gateway A**. Both routes "contain" `10.0.1.5` — the default (`0.0.0.0/0`) contains every address, and the `/24` contains `10.0.1.0`–`10.0.1.255`. The kernel uses **longest prefix match**: the more specific route wins. A `/24` is far more specific than a `/0`, so the `10.0.1.0/24 via A` route is selected and the packet is sent to gateway A. The default route only handles destinations that match *nothing* more specific. You can confirm with `ip route get 10.0.1.5`, which shows the chosen route and next-hop.
</details>

---

**Q2. What is the difference between a `blackhole` route and an `unreachable` route?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Both discard matching packets, but they differ in what feedback they give the sender. A **blackhole** route silently drops the packet — no ICMP error is returned, so the sender just sees a timeout and never learns why. An **unreachable** route also drops the packet but sends back an ICMP "destination unreachable" message, so the sender immediately knows the destination can't be reached (and a `prohibit` route returns "administratively prohibited" instead). Use `blackhole` when you want to quietly absorb traffic without revealing anything (e.g., dropping a DDoS source); use `unreachable`/`prohibit` when you want callers to fail fast with a clear error rather than hanging until timeout.
</details>

---

**Q3. Why is `ip route get 8.8.8.8` more trustworthy than reading `ip route show` and reasoning about it yourself?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because `ip route get` makes the **kernel** perform the actual route lookup and report the real decision — the exact route chosen, the next-hop, the output interface, and even the source address it would select. Reading `ip route show` and reasoning manually is error-prone: you might overlook longest-prefix-match subtleties, metrics, multiple routing tables, or policy rules (Lesson 18) that change which table is even consulted. `ip route get` accounts for all of that automatically and shows you ground truth. The discipline "never guess routing; ask the kernel" exists precisely because manual reasoning misses edge cases that `ip route get` resolves correctly every time.
</details>

---

## Homework

In a namespace, set up these routes and then predict (before running `ip route get`) where each destination goes. Verify your predictions.

```
default via 10.0.0.1
10.0.0.0/24 dev eth0 (connected)
10.0.0.128/25 via 10.0.0.2
10.0.0.200/32 via 10.0.0.3
```

Destinations to resolve: `10.0.0.50`, `10.0.0.130`, `10.0.0.200`, `203.0.113.1`.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Apply longest-prefix match to each:

- **`10.0.0.50`** → matches the connected `10.0.0.0/24` (scope link, directly reachable on eth0). It does *not* fall in the `/25` (which is `.128`–`.255`). So it's delivered directly via eth0, no gateway. Most specific match: `/24`.
- **`10.0.0.130`** → falls in `10.0.0.128/25` (`.128`–`.255`). Both the `/24` and the `/25` contain it, but `/25` is longer/more specific, so it goes **via 10.0.0.2**.
- **`10.0.0.200`** → contained by `/24`, `/25`, *and* the host route `10.0.0.200/32`. The `/32` is the most specific possible, so it goes **via 10.0.0.3**, overriding the `/25` and `/24`.
- **`203.0.113.1`** → matches no specific route, so it falls to the `0.0.0.0/0` default and goes **via 10.0.0.1**.

The pattern: as the destination's matching prefix gets longer, a more specific route can override the broader ones — `/32` beats `/25` beats `/24` beats the default `/0`.
</details>
