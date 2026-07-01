---
title: "Lesson 05 — Neighbor Tables & ARP"
nav_order: 5
parent: "Phase 1: Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 05: Neighbor Tables & ARP

## Concept

A route tells the kernel *which interface* to send a packet out of. But sending a packet on Ethernet requires a **MAC address** — a 6-byte hardware identifier burned into the network card. IP addresses are L3; MAC addresses are L2. Something has to bridge the gap.

That something is **ARP** — the Address Resolution Protocol. Before the kernel can build an Ethernet frame destined for `10.0.0.2`, it needs to know "what is the MAC address of the machine that owns `10.0.0.2`?"

```
You want to send a packet to 10.0.0.2

  Kernel checks ARP cache:
  ┌─────────────────────────────────────────────┐
  │  10.0.0.2   →   aa:bb:cc:dd:ee:ff   REACHABLE │
  └─────────────────────────────────────────────┘
        hit → wrap packet in Ethernet frame, send it
        miss → broadcast ARP request first

ARP Request (broadcast — every machine on the segment sees it):
  "Who has 10.0.0.2? Tell 10.0.0.1"
  src IP:  10.0.0.1    src MAC: my MAC
  dst IP:  10.0.0.2    dst MAC: ff:ff:ff:ff:ff:ff  ← broadcast

ARP Reply (unicast — only the target responds):
  "10.0.0.2 is at aa:bb:cc:dd:ee:ff"
  src IP:  10.0.0.2    src MAC: aa:bb:cc:dd:ee:ff
  dst IP:  10.0.0.1    dst MAC: my MAC
```

The kernel stores the result in the **neighbor table** (also called the ARP cache). The `ip neigh` command manages this table.

---

## How it works

### When ARP fires

ARP only runs for addresses the kernel will deliver **directly** — that is, destinations matched by a `scope link` route. If the destination is behind a gateway, the kernel ARPs for the gateway's MAC, not the final destination's.

```
Sending to 10.0.0.2 (/24 on same interface) → ARP for 10.0.0.2 directly
Sending to 8.8.8.8 (matched by default route via 10.0.0.1) → ARP for 10.0.0.1 (the gateway)
```

### ARP cache states

Every entry in the neighbor table has a state:

| State | Meaning |
|---|---|
| `REACHABLE` | Entry is fresh — confirmed recently. Used without re-checking. |
| `STALE` | Entry exists but confirmation timer expired. Still used, but kernel will re-confirm before the next send. |
| `DELAY` | Waiting a short grace period before probing (gives TCP a chance to confirm reachability itself). |
| `PROBE` | Sending unicast probes to the cached MAC to re-confirm. |
| `FAILED` | Probes sent, no reply. Kernel won't use this entry. Packets are dropped. |
| `PERMANENT` | Manually set static entry. Never expires, never probed. |

The normal lifecycle of a dynamic entry: `REACHABLE → STALE → DELAY → PROBE → REACHABLE` (if host responds) or `FAILED` (if it doesn't).

### Gratuitous ARP

A **gratuitous ARP** is an ARP reply sent without a prior request — a machine announces its own IP-to-MAC mapping to the whole segment without being asked. Any machine that receives it will update its ARP cache.

Use cases:
- **IP failover / HA clusters**: when a backup node takes over a virtual IP (e.g. with [keepalived](https://en.wikipedia.org/wiki/Keepalived) or [VRRP](https://en.wikipedia.org/wiki/Virtual_Router_Redundancy_Protocol)), it sends a gratuitous ARP so every other machine on the segment stops sending to the failed node's MAC and starts sending to the new one. Without it, traffic keeps going to the old MAC until every machine's ARP cache entry expires (which can take minutes).
- **Duplicate IP detection**: a machine sends a gratuitous ARP for its own IP when it comes online. If another machine replies, there is a conflict.

### NDP — the IPv6 equivalent

IPv6 has no ARP. It uses **Neighbor Discovery Protocol** ([NDP](https://en.wikipedia.org/wiki/Neighbor_Discovery_Protocol)), which is built on [ICMPv6](https://en.wikipedia.org/wiki/ICMPv6):

- **Neighbor Solicitation** (ICMPv6 type 135) — the multicast equivalent of the ARP request. It goes to a **solicited-node multicast address** derived from the target IP, not a flat broadcast. Only the target machine is in that multicast group, so far fewer machines are interrupted.
- **Neighbor Advertisement** (ICMPv6 type 136) — the unicast reply, equivalent to the ARP reply.

The kernel stores IPv6 neighbors in the same neighbor table — `ip neigh show` displays both IPv4 and IPv6 entries.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip neigh show` | Show the full ARP / neighbor table |
| `ip neigh show dev <iface>` | Show neighbor entries on one interface only |
| `ip neigh add <ip> lladdr <mac> dev <iface> nud permanent` | Add a static (permanent) ARP entry |
| `ip neigh del <ip> dev <iface>` | Delete a specific ARP entry |
| `ip neigh flush dev <iface>` | Flush all dynamic entries on an interface |
| `ip neigh flush all` | Flush all dynamic entries on all interfaces |
| `tcpdump -e -n -i <iface> arp` | Capture ARP packets and show MAC addresses (`-e`) |
| `arping -I <iface> <ip>` | Send ARP requests and display replies (like ping at L2) |

{: .note }
> **`lladdr` means link-layer address**
> `lladdr` is the generic term the kernel uses for a MAC address in the context of neighbor entries. `ll` = link-layer, the L2 address bound to the hardware interface.

---

## Lab

This lab uses a **veth pair** — a virtual Ethernet cable with two ends. Veth pairs are covered fully in Lesson 7; here you only need to know that packets entering one end exit the other, just like a real cable. It is the simplest way to create two interfaces on the same L2 segment inside a single machine.

### Step 1 — Build the topology

```bash
# Two namespaces
$ sudo ip netns add ns1
$ sudo ip netns add ns2

# One veth pair (virtual cable)
$ sudo ip link add veth-ns1 type veth peer name veth-ns2

# Move one end into each namespace
$ sudo ip link set veth-ns1 netns ns1
$ sudo ip link set veth-ns2 netns ns2

# Bring up interfaces and assign IPs
$ sudo ip netns exec ns1 ip link set veth-ns1 up
$ sudo ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth-ns1

$ sudo ip netns exec ns2 ip link set veth-ns2 up
$ sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth-ns2
```

### Step 2 — Verify the ARP table is empty

```bash
$ sudo ip netns exec ns1 ip neigh show
# (no output — cache is empty)
```

No traffic has happened yet, so ns1 has never asked about ns2 and has no entries.

### Step 3 — Trigger ARP by pinging

```bash
$ sudo ip netns exec ns1 ping -c 1 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.065 ms
```

Now check the neighbor table:

```bash
$ sudo ip netns exec ns1 ip neigh show
10.0.0.2 dev veth-ns1 lladdr <ns2-mac> REACHABLE
```

The entry appeared automatically. The kernel sent an ARP request before the first ping packet and stored the reply.

### Step 4 — Observe ARP traffic with tcpdump

Flush the cache so ARP fires again, then capture it:

```bash
# Flush the cache
$ sudo ip netns exec ns1 ip neigh flush dev veth-ns1

# Start capturing ARP on ns1's interface (runs in background)
$ sudo ip netns exec ns1 tcpdump -e -n -i veth-ns1 arp &

# Ping to trigger ARP
$ sudo ip netns exec ns1 ping -c 1 10.0.0.2
```

You will see two ARP packets:

```
# ARP request (broadcast)
<ns1-mac> > ff:ff:ff:ff:ff:ff  ARP, Request who-has 10.0.0.2 tell 10.0.0.1

# ARP reply (unicast)
<ns2-mac> > <ns1-mac>          ARP, Reply 10.0.0.2 is-at <ns2-mac>
```

Key observations:
- The request goes to `ff:ff:ff:ff:ff:ff` — the Ethernet broadcast address. Every device on the segment receives it.
- The reply is unicast — only ns1 receives it.
- The `-e` flag on tcpdump is what reveals the MAC addresses (`<ns1-mac>`, `<ns2-mac>`).

Kill the background tcpdump when done:

```bash
$ kill %1
```

### Step 5 — Add a static (permanent) ARP entry

First, get ns2's MAC address:

```bash
$ sudo ip netns exec ns2 ip link show veth-ns2
# Look for the "link/ether xx:xx:xx:xx:xx:xx" line
# Copy that MAC — you need it for the next command
```

Flush the dynamic cache and add a permanent entry:

```bash
$ sudo ip netns exec ns1 ip neigh flush dev veth-ns1
$ sudo ip netns exec ns1 ip neigh add 10.0.0.2 lladdr <ns2-mac> dev veth-ns1 nud permanent
$ sudo ip netns exec ns1 ip neigh show
10.0.0.2 dev veth-ns1 lladdr <ns2-mac> PERMANENT
```

Now capture ARP while pinging — you will see **no ARP requests**:

```bash
$ sudo ip netns exec ns1 tcpdump -e -n -i veth-ns1 arp &
$ sudo ip netns exec ns1 ping -c 2 10.0.0.2
# (ICMP packets appear but no ARP — the kernel used the static entry directly)
$ kill %1
```

A `PERMANENT` entry is never probed and never expires. The kernel uses it as-is, forever.

### Step 6 — Observe FAILED state

Try to reach an IP that doesn't exist on the segment:

```bash
$ sudo ip netns exec ns1 ping -c 1 -W 2 10.0.0.99
# ping: connect: No route to host  — or simply times out
```

Check the neighbor table:

```bash
$ sudo ip netns exec ns1 ip neigh show
10.0.0.99 dev veth-ns1  FAILED
```

The kernel tried ARP (`Who has 10.0.0.99?`), got no reply, and marked the entry `FAILED`. Subsequent packets to `10.0.0.99` are dropped immediately without re-sending ARP (until the failed entry expires).

### Step 7 — Clean up

```bash
$ sudo ip netns delete ns1
$ sudo ip netns delete ns2
# The veth pair is deleted automatically when its namespace is deleted
```

---

## Further Reading

| Topic | Link |
|---|---|
| Address Resolution Protocol (ARP) | [Wikipedia — ARP](https://en.wikipedia.org/wiki/Address_Resolution_Protocol) |
| `ip-neighbour` — manage ARP entries | [man7.org — ip-neighbour(8)](https://man7.org/linux/man-pages/man8/ip-neighbour.8.html) |
| Neighbor Discovery Protocol (NDP) | [Wikipedia — NDP](https://en.wikipedia.org/wiki/Neighbor_Discovery_Protocol) |
| Gratuitous ARP | [Wikipedia — ARP announcements](https://en.wikipedia.org/wiki/Address_Resolution_Protocol#ARP_announcements) |
| Virtual Router Redundancy Protocol (VRRP) | [Wikipedia — VRRP](https://en.wikipedia.org/wiki/Virtual_Router_Redundancy_Protocol) |
| ICMPv6 | [Wikipedia — ICMPv6](https://en.wikipedia.org/wiki/ICMPv6) |
| `tcpdump` | [man7.org — tcpdump(8)](https://man7.org/linux/man-pages/man8/tcpdump.8.html) |

---

## Checkpoint

**Q1. You have a correct route for `10.0.0.2` — `ip route get 10.0.0.2` shows it will go out `veth-ns1` directly. But `ping 10.0.0.2` fails immediately. ARP resolution is also failing. Why does `ping` fail even though the route is correct?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A route only tells the kernel which interface to send the packet out of. To actually transmit the packet, the kernel must wrap it in an Ethernet frame — and that frame requires the destination's MAC address. If ARP fails (nobody replies to "who has 10.0.0.2?"), the kernel has no MAC to put in the frame and drops the packet. The route is correct, but the L2 delivery mechanism is broken. IP and Ethernet are separate layers; the route resolves L3, but ARP must resolve L2 before any packet can leave the wire.
</details>

---

**Q2. What is the difference between a `STALE` ARP entry and a `FAILED` one? Can the kernel still deliver packets to a `STALE` address?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`STALE` means the entry is old — the kernel has a MAC address cached but hasn't confirmed it recently (the reachability timer expired). The kernel **will still use the STALE entry** to send packets; it just also schedules a probe to re-confirm it. The MAC in the cache is probably still valid.

`FAILED` means the kernel actively tried to re-confirm the entry (sent unicast probes) and got no response. The entry is known-bad. The kernel **drops packets** to a `FAILED` address rather than using the stale MAC.

STALE = "I have an answer, but I should double-check it." FAILED = "I tried to double-check and nobody answered."
</details>

---

**Q3. A high-availability pair of servers shares a virtual IP (`10.0.0.100`). The primary server fails. The backup takes over and brings up `10.0.0.100` on its own interface. But clients keep sending traffic to the dead primary for the next two minutes. What is missing, and what would fix it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The clients have a cached ARP entry mapping `10.0.0.100` to the primary's MAC address. When the backup takes over, the clients don't know — they keep sending Ethernet frames to the primary's (now silent) MAC. The frames reach nobody.

What is missing: a **gratuitous ARP**. When the backup brings up `10.0.0.100`, it should immediately broadcast an ARP announcement: "10.0.0.100 is at <backup-MAC>." Every client on the segment will update its ARP cache on receipt. Traffic shifts to the backup within milliseconds, not minutes. VRRP and keepalived both send gratuitous ARPs as part of failover for exactly this reason.
</details>

---

## Homework

Build the same two-namespace veth topology from the lab (ns1 at `10.0.0.1/24`, ns2 at `10.0.0.2/24`).

1. Ping once so ARP populates the cache.
2. Bring `veth-ns2` **down** in ns2: `sudo ip netns exec ns2 ip link set veth-ns2 down`
3. Watch what happens to the ARP entry in ns1 over the next 30–60 seconds. Check `ip neigh show` every 10 seconds.
4. Try pinging `10.0.0.2` from ns1 while the link is down.

Record the state transitions you observe and explain why each one occurred.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
After you bring veth-ns2 down and stop sending traffic, the REACHABLE entry in ns1 ages out:

1. `REACHABLE` — entry is fresh right after the first ping.
2. `STALE` — the reachability confirmation timer expires (typically ~30 seconds). The kernel still has the MAC.
3. `DELAY` — ns1 wants to re-confirm. It waits a grace period (5 seconds) to see if a higher-layer protocol (like TCP) can confirm reachability without sending an ARP probe.
4. `PROBE` — ns1 sends unicast ARP probes to the cached MAC. Since veth-ns2 is down, no reply arrives.
5. `FAILED` — after 3 probes with no response, the entry is marked FAILED.

While the entry is STALE or DELAY, pings may still appear to work momentarily (the kernel optimistically uses the cached MAC while probing). Once the entry reaches FAILED, pings stop and the kernel reports no route to host or the packets are silently dropped.

Bringing veth-ns2 back up does not automatically re-trigger ARP. You either wait for the FAILED entry to expire and then ping again, or flush the entry manually with `ip neigh flush dev veth-ns1` and ping.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 06 — tcpdump, Your Constant Companion →](lesson-06-tcpdump){: .btn .btn-primary }
