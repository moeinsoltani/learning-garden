---
title: "Lesson 03 — Build Two Isolated Namespaces"
nav_order: 3
parent: "Phase 1: Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 03: Build Two Isolated Namespaces

## Concept

The previous two lessons were conceptual. This one is entirely hands-on. You will
build two namespaces, assign IP addresses inside them, and then try — and fail —
to make them talk to each other. That failure is the lesson.

```
 Host (initial namespace)
 ┌────────────────────────────────────────────────────┐
 │                                                    │
 │   ┌──────────────┐        ┌──────────────┐         │
 │   │     ns1      │        │     ns2      │         │
 │   │              │        │              │         │
 │   │ lo: 10.0.1.1 │   ??   │ lo: 10.0.2.1 │         │
 │   │              │ ─────▶ │              │         │
 │   └──────────────┘  fail  └──────────────┘         │
 │                                                    │
 └────────────────────────────────────────────────────┘
```

By the end you will have a concrete, visceral answer to: *why can't ns1 ping ns2?*
That question is the setup for Lesson 7, where you will wire them together.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip netns add <name>` | Create a new named network namespace |
| `ip netns exec <name> <cmd>` | Run a command inside a namespace |
| `ip netns list` | List all named namespaces |
| `ip netns delete <name>` | Delete a namespace |
| `ip link set lo up` | Bring the loopback interface up |
| `ip addr add <ip/prefix> dev <iface>` | Assign an IP address to an interface |
| `ip addr show` | Show IP addresses on all interfaces |
| `ip route show` | Show the routing table |
| `ping -c2 <ip>` | Send 2 ICMP echo requests |

---

## Lab

Work through this step by step. Do not skip ahead.

### Step 1 — Create the namespaces

```bash
$ sudo ip netns add ns1
$ sudo ip netns add ns2
$ ip netns list
ns2
ns1
```

### Step 2 — Bring up loopback in each namespace

New namespaces start with `lo` in the DOWN state. Bring it up in both.

```bash
$ sudo ip netns exec ns1 ip link set lo up
$ sudo ip netns exec ns2 ip link set lo up
```

Verify:

```bash
$ sudo ip netns exec ns1 ip link show lo
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
#                  ^^^^^^^^^^
#                  lo is now UP
```

{: .note }
> **What is `lo`?**
> `lo` is the [loopback interface](https://en.wikipedia.org/wiki/Loopback) — a
> virtual interface that exists entirely in software with no hardware behind it.
> Packets sent into `lo` are immediately delivered back to `lo` in the same
> namespace. They never leave the machine and never touch a physical NIC.
> Every namespace gets one automatically when it is created.
>
> You can assign any IP address to `lo` — it is just an interface like any other.
> The key property: any IP you assign to `lo` is always reachable from within
> the same namespace, because the packet loops back instantly. This makes `lo`
> useful for testing (stable address that is always up) and for giving a namespace
> a predictable IP to work with before any real interfaces are connected.

### Step 3 — Assign IP addresses

Give each namespace a unique IP on its loopback interface.

```bash
$ sudo ip netns exec ns1 ip addr add 10.0.1.1/24 dev lo
$ sudo ip netns exec ns2 ip addr add 10.0.2.1/24 dev lo
```

Verify ns1 has its address:

```bash
$ sudo ip netns exec ns1 ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
    inet 127.0.0.1/8 scope host lo
    inet 10.0.1.1/24 scope global lo
```

Two addresses on lo: the automatic `127.0.0.1/8` (assigned when you brought lo up)
and your new `10.0.1.1/24`.

{: .note }
> **What does adding an IP to `lo` actually mean?**
> It tells the kernel two things: (1) this namespace owns this IP — deliver any
> incoming packet for it here; (2) create a connected route for the subnet
> automatically. It does NOT mean traffic can leave `lo` to reach other namespaces.
> `lo` loops everything back internally. So `10.0.1.1` is reachable from inside
> ns1, but completely unreachable from ns2 or the host.
>
> Try it — ping your own loopback IP from inside the namespace. It will always succeed:
> ```bash
> sudo ip netns exec ns1 ping -c2 10.0.1.1   # succeeds — loops back
> sudo ip netns exec ns1 ping -c2 10.0.2.1   # fails — different namespace
> ```

### Step 4 — Inspect the routing tables

Each namespace has its own routing table, built automatically from the addresses
you assigned.

```bash
$ sudo ip netns exec ns1 ip route show
10.0.1.0/24 dev lo proto kernel scope link src 10.0.1.1

$ sudo ip netns exec ns2 ip route show
10.0.2.0/24 dev lo proto kernel scope link src 10.0.2.1
```

{: .note }
> **Reading `ip route` output**
> Each line is one route. Breaking down `10.0.1.0/24 dev lo proto kernel scope link src 10.0.1.1`:
>
> | Field | Meaning |
> |---|---|
> | `10.0.1.0/24` | Destination: match packets going to any IP in this subnet |
> | `dev lo` | Send them out through the `lo` interface |
> | `proto kernel` | The kernel added this route automatically when you assigned the IP |
> | `scope link` | Destination is directly reachable — no gateway needed |
> | `src 10.0.1.1` | Use this as the source IP when sending to this network |
>
> Routing will be covered in depth in Phase 5. For now, just notice that each
> namespace has a completely separate table with no knowledge of the other.

Neither routing table knows anything about the other namespace. ns1 has a route
for `10.0.1.0/24` only. ns2 has a route for `10.0.2.0/24` only.

Compare with the host routing table:

```bash
$ ip route show
# Your host's routes — completely separate, does not include 10.0.1.0/24 or 10.0.2.0/24
```

### Step 5 — Try to ping across namespaces (it will fail)

Try to reach ns2's IP from inside ns1:

```bash
$ sudo ip netns exec ns1 ping -c2 10.0.2.1
ping: connect: Network is unreachable
```

Try to reach ns1 from the host:

```bash
$ ping -c2 10.0.1.1
ping: connect: Network is unreachable
```

Both fail. There is no path between these namespaces.

### Step 6 — Confirm the host cannot see inside the namespaces

```bash
$ ip addr show
# eth0, lo — the host's interfaces
# 10.0.1.1 and 10.0.2.1 are nowhere here — they exist only inside their namespaces
```

### Step 7 — Clean up

```bash
$ sudo ip netns delete ns1
$ sudo ip netns delete ns2
$ ip netns list
# (empty)
```

---

## Why the Ping Failed

The error `Network is unreachable` means the kernel looked in its routing table
and found no route to the destination. From inside ns1, the routing table only
knows about `10.0.1.0/24`. It has never heard of `10.0.2.0/24` — that route
lives in ns2's table, which is a completely separate kernel object.

```
ns1 routing table          ns2 routing table
─────────────────          ─────────────────
10.0.1.0/24 dev lo         10.0.2.0/24 dev lo
(nothing else)             (nothing else)
```

Even if ns1 somehow knew the route, there is no interface to send the packet
through. ns1 has only `lo`, which loops packets back to itself — it cannot
transmit to another namespace.

To connect two namespaces you need a **virtual Ethernet pair** (veth) — a virtual
cable with one end in each namespace. That is exactly what Lesson 07 covers.

---

## Further Reading

| Topic | Link |
|---|---|
| `ip-address` — manage IP addresses | [man7.org — ip-address(8)](https://man7.org/linux/man-pages/man8/ip-address.8.html) |
| `ip-route` — manage routing tables | [man7.org — ip-route(8)](https://man7.org/linux/man-pages/man8/ip-route.8.html) |
| `ip-netns` — manage namespaces | [man7.org — ip-netns(8)](https://man7.org/linux/man-pages/man8/ip-netns.8.html) |
| ICMP and ping | [Wikipedia — Internet Control Message Protocol](https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol) |
| Loopback interface | [Wikipedia — Loopback](https://en.wikipedia.org/wiki/Loopback) |
| `ping` command | [man7.org — ping(8)](https://man7.org/linux/man-pages/man8/ping.8.html) |

---

## Checkpoint

**Q1. You assigned `10.0.1.1/24` to ns1's loopback. The routing table inside ns1
then showed `10.0.1.0/24 dev lo proto kernel`. Where did that route come from —
you didn't add it manually.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The kernel adds it automatically. Whenever you assign an IP address with a prefix length (e.g. /24) to an interface, the kernel creates a "connected route" for the entire subnet — it assumes that any address in that subnet is reachable directly through that interface. The `proto kernel` label means the kernel generated it, not a user or routing daemon.
</details>

---

**Q2. Why did `ping 10.0.2.1` from inside ns1 return "Network is unreachable"
rather than just timing out?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
"Network is unreachable" is returned immediately by the local kernel — it looked in ns1's routing table, found no route to 10.0.2.0/24, and rejected the packet before it was even sent. A timeout would mean the packet was sent but no reply came back. "Unreachable" means it never left the kernel at all.
</details>

---

**Q3. What would you need to add to make ns1 and ns2 able to ping each other?
You don't need to know the exact commands — reason from what you know.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Two things: a virtual interface connecting the two namespaces (so there is a physical path for packets to travel), and a route in each namespace pointing to the other's subnet via that interface. Right now ns1 has no interface that leads anywhere except back to itself. The virtual interface used for this is called a veth pair — covered in Lesson 07.
</details>

---

## Homework

Without deleting them, run both namespaces at the same time and use `lsns -t net`
to list all three network namespaces (host, ns1, ns2). Identify which PID belongs
to each namespace and explain why the host namespace shows a much higher NPROCS count.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The host (initial) namespace shows a high NPROCS because every process on the system that hasn't been moved into another namespace lives there — systemd, sshd, your shell, cron, and everything else. ns1 and ns2 each show NPROCS of 1 or 0 because no long-running process lives inside them; they exist as persistent objects (pinned via /var/run/netns/) but are currently empty.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 04 — IP Addressing Fundamentals →](lesson-04-ip-addressing){: .btn .btn-primary }
