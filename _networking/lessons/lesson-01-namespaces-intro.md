---
title: "Lesson 01 — What a Network Namespace Is"
nav_order: 1
parent: "Phase 1: Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 01: What a Network Namespace Is

## Concept

Imagine your Linux machine's networking as a single room. Every network interface
(`eth0`, `lo`, etc.), every route, every firewall rule, every socket — all of it
lives in that one room. Any process on the machine can see and use all of it.

A **network namespace** is a second, completely separate room. It has its own
interfaces, its own routing table, its own firewall rules, its own sockets.
A process running inside that room sees only what is in that room — nothing from
the original room, and nothing from any other room.

```
┌─────────────────────────────────────┐   ┌─────────────────────────────────┐
│         Default namespace           │   │         Namespace: ns1          │
│                                     │   │                                 │
│  interfaces:  eth0, lo              │   │  interfaces:  lo (only)         │
│  routes:      10.0.0.0/24 via eth0  │   │  routes:      (none)            │
│  firewall:    (your iptables rules) │   │  firewall:    (empty)           │
│  sockets:     nginx on :80          │   │  sockets:     (none)            │
└─────────────────────────────────────┘   └─────────────────────────────────┘
         different rooms — completely isolated
```

This is exactly how containers work. Docker gives each container its own network
namespace. Inside the container, it looks like the container owns the whole machine's
networking — but it is actually just looking at its own small room.

---

## What Is Isolated (per namespace)

Each network namespace has its own independent copy of:

| Thing | Example |
|---|---|
| Network interfaces | `eth0`, `lo`, `veth0` |
| IP addresses | 10.0.0.1 on eth0 |
| Routing table | `default via 10.0.0.1` |
| [ARP](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) / neighbor table | which MAC owns 10.0.0.2 |
| Firewall rules | nftables / iptables rulesets |
| [Sockets](https://en.wikipedia.org/wiki/Network_socket) | which process is listening on :80 |

{: .note }
> **What is a socket?**
> A socket is the kernel object that connects a process to the network. When a
> program (say, a web server) wants to receive traffic, it asks the kernel for a
> socket on a specific port — e.g. port 80. The kernel creates a mailbox: any
> packet arriving for that port gets delivered to that socket, and the process
> reads from it.
>
> ```
>  Process (nginx)
>       |
>    [socket]   ← kernel object, lives inside a specific namespace
>       |
>  Network stack
>       |
>     eth0
> ```
>
> Because sockets are per-namespace, two processes can each bind to port 80
> without conflict — one inside `ns1`, one on the host. The kernel knows which
> namespace each socket belongs to and routes packets accordingly.
> Run `ss -tulpn` on your VM to list all open sockets.

If you bind a process to port 80 inside `ns1`, it does **not** conflict with
a process listening on port 80 in the default namespace. They are in separate rooms.

---

## What Is NOT Isolated (shared with the host)

Network namespaces only isolate networking. The following are still shared:

- **The kernel itself** — same kernel, same kernel version
- **The filesystem** — unless you also use a mount namespace (containers do both)
- **Processes** — a process in ns1 is still visible in the host's process table
  (unless you also use a PID namespace)
- **CPU and memory** — no isolation at all unless you add [cgroups](https://en.wikipedia.org/wiki/Cgroups)

This is why a "network namespace" alone does not make a container. Docker combines
network namespaces with PID namespaces, mount namespaces, and cgroups.

---

## How It Works

The kernel maintains a list of network namespaces. Every process has a pointer to
one of them. When the process calls `socket()`, `bind()`, `sendmsg()`, or reads
the routing table, the kernel uses that process's namespace — not some global state.

When you create a new namespace with `ip netns add ns1`, the kernel:
1. Allocates a new, empty networking context
2. Gives it a loopback interface (`lo`) — but lo starts DOWN and has no IP

When you run a command inside ns1 with `ip netns exec ns1 <command>`, the kernel
temporarily switches that process's network namespace pointer to ns1 before running
the command.

---

## Lab

You need a Linux machine for this (your VM, not WSL2 for now).
Open a terminal and run these commands. Lines starting with `$` are commands to run.
Lines starting with `#` are comments explaining what you are about to see.

```bash
# See what namespaces exist right now (just the default one)
$ ip netns list
# (no output — the default namespace is not listed here)

# Create a new namespace called ns1
$ sudo ip netns add ns1

# List again
$ ip netns list
ns1

# Look inside ns1 — what interfaces does it have?
$ sudo ip netns exec ns1 ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

# Notice: lo is DOWN and has no IP address. The namespace is empty.

# Compare with the default namespace
$ ip link show
# You will see eth0 (or ens3 or similar), lo, and anything else on your VM

# Look at the routing table inside ns1
$ sudo ip netns exec ns1 ip route show
# (no output — ns1 has no routes at all)

# Compare with the default namespace routing table
$ ip route show
# You will see a default route and your local subnet routes

# Clean up
$ sudo ip netns delete ns1
```

**Expected result:** ns1 has only a DOWN loopback and zero routes.
The default namespace has interfaces and routes. They do not see each other.

---

## Further Reading

| Topic | Link |
|---|---|
| Network namespaces (Linux man page) | [man7.org — network_namespaces(7)](https://man7.org/linux/man-pages/man7/network_namespaces.7.html) |
| `ip netns` command reference | [man7.org — ip-netns(8)](https://man7.org/linux/man-pages/man8/ip-netns.8.html) |
| What is a network socket | [Wikipedia — Network socket](https://en.wikipedia.org/wiki/Network_socket) |
| ARP protocol | [Wikipedia — Address Resolution Protocol](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) |
| Linux cgroups | [Wikipedia — cgroups](https://en.wikipedia.org/wiki/Cgroups) |
| OS-level virtualisation (containers) | [Wikipedia — OS-level virtualisation](https://en.wikipedia.org/wiki/OS-level_virtualization) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What exactly is isolated when you create a network namespace? List at least four things.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Network interfaces (eth0, lo, veth0, etc.), IP addresses assigned to those interfaces, the routing table, the ARP/neighbor table (which MAC address owns which IP), firewall rules (nftables/iptables rulesets), and sockets (which process is listening on which port). All of these are per-namespace — each namespace has its own independent copy.
</details>

---

**Q2. A web server is listening on port 443 inside namespace `ns1`. Can another process on the host (default namespace) connect to it on `127.0.0.1:443`? Why or why not?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
No. The socket bound to port 443 lives inside ns1's networking context. The host's 127.0.0.1 is the loopback address of the default namespace — a completely separate room. A process in the default namespace has no visibility into ns1's sockets at all. To connect across namespaces you need a virtual interface linking the two (covered in a later lesson).
</details>

---

**Q3. You move a process into namespace `ns1`. The process calls `open("/etc/hosts")`. Does it read the host's `/etc/hosts` or ns1's `/etc/hosts`? Explain why.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It reads the host's /etc/hosts. Network namespaces only isolate networking — the filesystem is not touched. To give a namespace its own filesystem you need a separate mount namespace as well. This is exactly what Docker does: it combines a network namespace, a mount namespace, a PID namespace, and cgroups. A network namespace alone is not a container.
</details>

---

**Q4. A new namespace is created. What is the state of its loopback interface? Why does this matter?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The loopback interface (lo) exists but is DOWN with no IP address assigned. This matters because many applications try to connect to 127.0.0.1 — to reach themselves or other local services. If lo is down those connections fail immediately, even though no external network is involved. You must explicitly run `ip link set lo up` inside a new namespace before using it.
</details>

---

## Homework

Do this before Lesson 2.

1. Create a namespace called `myns`
2. Inside `myns`, bring up the loopback interface: `sudo ip netns exec myns ip link set lo up`
3. Check that lo now has an IP: `sudo ip netns exec myns ip addr show lo` — you should see `127.0.0.1/8`
4. Delete `myns`

Write one sentence below about what bringing `lo` up does and why it was not up by default.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Running `ip link set lo up` activates the loopback interface and causes the kernel to automatically assign 127.0.0.1/8 to it. It is not up by default because the kernel creates namespaces in a completely blank state — it makes no assumptions about what the namespace will be used for, so nothing is configured until you explicitly ask for it.
</details>
