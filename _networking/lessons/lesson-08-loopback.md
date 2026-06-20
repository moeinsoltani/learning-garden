---
title: "Lesson 08 — Loopback Interface"
nav_order: 8
parent: "Phase 2: Virtual Interfaces"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 08: Loopback Interface

## Concept

The **loopback interface** (`lo`) is a virtual interface that talks only to itself. A packet sent to a loopback address never touches a NIC, never reaches the wire — it is turned around inside the kernel and delivered straight back to the local machine. `127.0.0.1` is its famous IPv4 address; `::1` is the IPv6 one.

```
   application sends to 127.0.0.1:8080
            │
            ▼
   ┌─────────────────┐
   │   lo interface   │   packet goes "out"... and immediately comes back "in"
   │  (the U-turn)    │   never reaches any physical interface
   └─────────────────┘
            │
            ▼
   application listening on 127.0.0.1:8080 receives it
```

Every namespace gets its own `lo`. They are independent — `127.0.0.1` inside ns1 is a completely different interface from `127.0.0.1` inside ns2 or on the host.

---

## How it works

Loopback is how processes on the same machine talk to each other over the network stack without any hardware. A database listening on `127.0.0.1:5432` is reachable by other local processes but invisible to the outside world, because `127.0.0.0/8` has **host scope** — the kernel will never route it onto a physical interface (covered in Lesson 4).

There are two things that surprise beginners:

1. **A fresh namespace starts with `lo` DOWN.** When you create a namespace, it gets a loopback interface, but it is administratively down. Until you run `ip link set lo up`, even `ping 127.0.0.1` fails inside that namespace.

2. **`127.0.0.1` works with no network at all.** Pull every cable, disable Wi-Fi — `127.0.0.1` still responds. It depends on nothing external because the packet never leaves the kernel.

{: .note }
> **Why applications bind to `127.0.0.1` vs `0.0.0.0`**
> Binding a service to `127.0.0.1` means "only local processes can reach me" — a security boundary. Binding to `0.0.0.0` means "any interface, any source" — reachable from the network. The difference between a database exposed to the internet and one that isn't is often exactly this one address. Loopback is the safe default for anything that should stay local.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip link set lo up` | Bring the loopback interface up (required in new namespaces) |
| `ip addr show lo` | Show loopback addresses (`127.0.0.1/8`, `::1/128`) |
| `ss -tlnp` | List listening TCP sockets — see which bind to `127.0.0.1` vs `0.0.0.0` |
| `ping 127.0.0.1` | Confirm the local stack is alive |

---

## Lab

### Step 1 — Create a namespace and observe lo is DOWN

```bash
$ sudo ip netns add lab
$ sudo ip netns exec lab ip link show lo
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
#         ^^^^^^^^                          ^^^^^^^^^^
#   no UP/LOWER_UP flags                    state DOWN
```

### Step 2 — Confirm loopback is broken while DOWN

```bash
$ sudo ip netns exec lab ping -c 1 -W 1 127.0.0.1
ping: connect: Network is unreachable
```

Even talking to yourself fails — because the interface that does the U-turn is down.

### Step 3 — Bring lo up and retry

```bash
$ sudo ip netns exec lab ip link set lo up
$ sudo ip netns exec lab ip addr show lo
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ... state UNKNOWN
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host

$ sudo ip netns exec lab ping -c 2 127.0.0.1
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.03 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.02 ms
```

Bringing `lo` up automatically restored `127.0.0.1/8` and `::1/128`, and now loopback works.

### Step 4 — Prove loopback traffic never hits a real interface

Start a listener bound to loopback and connect to it, capturing on `lo`:

```bash
# Terminal A — listener bound to 127.0.0.1
$ sudo ip netns exec lab nc -l 127.0.0.1 -p 9000

# Terminal B — capture on lo
$ sudo ip netns exec lab tcpdump -n -i lo port 9000

# Terminal C — connect
$ sudo ip netns exec lab nc 127.0.0.1 9000
```

The handshake appears on `lo` — and *only* on `lo`. If you also captured on any other interface, you'd see nothing. The packets never left the loopback path.

### Step 5 — See the security boundary with `ss`

```bash
$ sudo ip netns exec lab ss -tlnp
State   Recv-Q  Send-Q  Local Address:Port   Peer Address:Port
LISTEN  0       1       127.0.0.1:9000        0.0.0.0:*
#                       ^^^^^^^^^
#       bound to loopback only — unreachable from any other host
```

If the listener had bound to `0.0.0.0:9000` instead, the Local Address would show `0.0.0.0:9000`, meaning it accepts connections from any interface.

### Step 6 — Clean up

```bash
$ sudo ip netns delete lab
```

---

## Further Reading

| Topic | Link |
|---|---|
| Localhost / loopback | [Wikipedia — localhost](https://en.wikipedia.org/wiki/Localhost) |
| Loopback address `127.0.0.0/8` (RFC 1122) | [Wikipedia — Loopback](https://en.wikipedia.org/wiki/Loopback#Virtual_loopback_interface) |
| `ss` — socket statistics | [man7.org — ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html) |
| `ip-link` | [man7.org — ip-link(8)](https://man7.org/linux/man-pages/man8/ip-link.8.html) |

---

## Checkpoint

**Q1. Why does `ping 127.0.0.1` succeed even when the machine is physically disconnected from every network?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because a loopback packet never leaves the kernel. When you send to `127.0.0.1`, the kernel recognizes a host-scope loopback address, performs an internal U-turn on the `lo` interface, and delivers the packet straight back to the local stack. No NIC, driver, cable, or link partner is involved, so the state of physical connectivity is irrelevant. `127.0.0.1` is reachable as long as the loopback interface is up — which has nothing to do with the outside world.
</details>

---

**Q2. You create a new namespace, assign IPs to a veth, but a program trying to bind to `127.0.0.1` inside it fails with "Cannot assign requested address." What did you forget?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You forgot to bring up the loopback interface: `ip netns exec <ns> ip link set lo up`. A fresh namespace's `lo` starts DOWN, and while it's down the `127.0.0.1/8` address isn't usable, so any process trying to bind to `127.0.0.1` fails. Bringing `lo` up restores `127.0.0.1` and `::1`. This is a classic gotcha — people set up veths and external addresses but forget that loopback also needs to be explicitly enabled in a new namespace.
</details>

---

**Q3. A teammate ran a database in a container bound to `0.0.0.0:5432` and it ended up reachable from the internet. How would binding to `127.0.0.1:5432` have changed that, and what's the trade-off?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Binding to `127.0.0.1:5432` makes the database reachable only by processes inside that same network namespace, because loopback traffic never leaves the stack. It would not have been exposed to the network at all, so the internet exposure could not happen. The trade-off: nothing *outside* the namespace can reach it either — not other containers, not the host, not legitimate remote clients. If other components genuinely need access, you bind to a specific routable address (not `0.0.0.0`) and control reachability with firewall rules, rather than binding to all interfaces and hoping a firewall is in place.
</details>

---

## Homework

Inside a namespace, bring `lo` up and add a *second* loopback-range address: `ip addr add 127.0.0.2/8 dev lo`. Start a listener on `127.0.0.2:7000` and connect to it from the same namespace. Then, from a *different* namespace connected via veth, try to reach `127.0.0.2:7000`. Explain the results.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Within the same namespace, connecting to `127.0.0.2:7000` works — the entire `127.0.0.0/8` block is loopback, so `127.0.0.2` does the same internal U-turn as `127.0.0.1`. You can bind services to distinct `127.x.x.x` addresses to run several "localhost-only" services without port collisions.

From a different namespace over a veth, the connection fails completely. `127.0.0.0/8` has **host scope** — the kernel never routes it onto a real interface and never accepts it arriving from one. Each namespace has its own independent loopback; `127.0.0.2` in one namespace is unrelated to `127.0.0.2` in another. Loopback addresses are fundamentally non-routable and machine-local (here, namespace-local), which is exactly why they're a safe isolation boundary.
</details>
