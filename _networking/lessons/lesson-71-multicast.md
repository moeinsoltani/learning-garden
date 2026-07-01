---
title: "Lesson 71 — Multicast & IGMP"
nav_order: 71
parent: "Phase 20: Modern Transport & Observability"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 71: Multicast & IGMP

## Concept

**Unicast** is one-to-one, **broadcast** is one-to-all-on-the-segment. **Multicast** is
one-to-*many-who-asked*: a sender transmits a single stream to a **group address**, and the network
delivers a copy only to the hosts that have *joined* that group. One send, many receivers, no
duplicate transmissions by the source.

```
   Unicast: send N copies        Multicast: send 1 copy; network replicates only toward joiners
   src ─► A                        src ─► [group 239.1.1.1] ─┬─► A (joined)
   src ─► B                                                  ├─► B (joined)
   src ─► C                                                  └─  C (not joined → no copy)
```

Used for IPTV, market-data feeds, and service discovery (mDNS) — efficient one-to-many on a LAN.

---

## How it works

**Group addresses.** Multicast uses the `224.0.0.0/4` IPv4 range (`ff00::/8` for IPv6). A host
"tunes in" by **joining** a group; it then receives anything sent to that group address.

**IGMP — telling the network who wants what.** [IGMP](https://en.wikipedia.org/wiki/Internet_Group_Management_Protocol)
(Internet Group Management Protocol) is how hosts announce group membership to their local router/switch.
A host sends an IGMP **join** (membership report) for a group; the router periodically sends IGMP
**queries**, and hosts still interested respond. When no one reports, the router stops forwarding that
group. (IPv6 uses MLD, the equivalent.)

**Switches: IGMP snooping.** A dumb switch would flood multicast to every port (like broadcast). With
**IGMP snooping**, the switch *watches* the IGMP join/leave messages and learns which ports have
members, so it forwards each group only out the ports that asked — turning multicast back into the
efficient thing it's supposed to be. This is the answer to "how does a switch know which ports to send
a multicast stream to."

**Routing multicast** between segments needs a multicast routing protocol (PIM); within one L2 segment,
IGMP + snooping is enough.

{: .note }
> **Why multicast is rare on the public internet**
> Multicast needs every router on the path to maintain group state and run PIM, and there's no
> inter-provider business/trust model for carrying someone else's multicast — so ISPs generally don't.
> It thrives on <em>controlled</em> networks (enterprise LANs, data centers, IPTV access networks)
> where one operator controls the whole path. On the internet, one-to-many is done with unicast + CDNs
> instead.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip maddr show` | Show multicast group memberships per interface |
| `ip maddr add <group> dev <if>` | Join a multicast group on an interface |
| `socat - UDP4-DATAGRAM:239.1.1.1:5000` | Send to / receive from a multicast group |
| `tcpdump -ni <if> 'multicast'` | Capture multicast traffic |
| `bridge -d link show` / `cat /sys/class/net/<br>/bridge/multicast_snooping` | IGMP snooping state on a bridge |

---

## Lab

Two namespaces join a group; a sender's single stream reaches both, and you observe IGMP.

### Step 1 — Three namespaces on a shared bridge (Lesson 11)

```bash
$ sudo ip netns add tx; sudo ip netns add rx1; sudo ip netns add rx2
# attach all to bridge br0; addresses 10.0.0.1 (tx), .2 (rx1), .3 (rx2).
```

### Step 2 — Receivers join group 239.1.1.1 and listen

```bash
$ sudo ip netns exec rx1 socat -u UDP4-RECVFROM:5000,ip-add-membership=239.1.1.1:10.0.0.2,fork - &
$ sudo ip netns exec rx2 socat -u UDP4-RECVFROM:5000,ip-add-membership=239.1.1.1:10.0.0.3,fork - &
$ sudo ip netns exec rx1 ip maddr show     # 239.1.1.1 listed on rx1's interface
```

### Step 3 — Watch IGMP, then send ONE stream

```bash
$ sudo ip netns exec tx tcpdump -ni tx-br igmp &       # see membership reports/queries
$ echo "live stream packet" | sudo ip netns exec tx socat - UDP4-DATAGRAM:239.1.1.1:5000
# BOTH rx1 and rx2 print the line, from a single send by tx.
```

### Step 4 — Observe snooping behavior

```bash
$ cat /sys/class/net/br0/bridge/multicast_snooping     # 1 = snooping on (forward only to members)
```

### Step 5 — Clean up

```bash
$ sudo ip netns delete tx rx1 rx2
```

---

## Further Reading

| Topic | Link |
|---|---|
| Multicast | [Wikipedia — Multicast](https://en.wikipedia.org/wiki/Multicast) |
| IGMP | [Wikipedia — Internet Group Management Protocol](https://en.wikipedia.org/wiki/Internet_Group_Management_Protocol) |
| IGMP snooping | [Wikipedia — IGMP snooping](https://en.wikipedia.org/wiki/IGMP_snooping) |
| `ip-maddress` | [man7.org — ip-maddress(8)](https://man7.org/linux/man-pages/man8/ip-maddress.8.html) |

---

## Checkpoint

**Q1. How does multicast differ from broadcast, and why is it more efficient than sending unicast to each receiver?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Broadcast</strong> goes to <em>every</em> host on the segment whether they want it or not;
<strong>multicast</strong> goes only to hosts that have explicitly <em>joined</em> the group, so
uninterested hosts aren't bothered. Compared to <strong>unicast-to-each</strong>, multicast is more
efficient because the <em>source sends a single copy</em> to the group address and the <em>network</em>
replicates it only where needed (down branches that have members), rather than the source transmitting
N separate copies and consuming N× the upstream bandwidth. So for one-to-many (IPTV, market data),
multicast keeps source and backbone bandwidth roughly constant regardless of the number of receivers.
</details>

---

**Q2. What role does IGMP play, and what would happen on a switch without IGMP snooping?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>IGMP</strong> is how hosts tell their local router/switch which multicast groups they want to
receive: hosts send membership reports (joins) for a group, routers periodically query to see who's
still interested, and forwarding for a group stops when no one reports. Without <strong>IGMP
snooping</strong>, a switch has no idea which ports have members, so it treats multicast like broadcast
and <strong>floods the stream out every port</strong> — wasting bandwidth on all the non-member ports
and defeating the efficiency of multicast. With snooping, the switch watches the IGMP join/leave traffic,
learns which ports have members, and forwards each group only to those ports. So IGMP is the signaling,
and snooping is the switch using that signaling to forward selectively.
</details>

---

**Q3. Why is multicast common on enterprise/data-center networks but rare on the public internet?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Multicast requires every router along the delivery path to maintain per-group state and run a multicast
routing protocol (PIM), and crucially there's <strong>no inter-provider model</strong> — no agreed
business arrangement or trust for one ISP to carry another's multicast groups across the internet. So
public providers generally don't enable it end-to-end. It works well only where a <strong>single operator
controls the whole path</strong> — enterprise LANs, data centers, and IPTV access networks — and can
provision the group state and routing consistently. On the open internet, the same one-to-many need is
met instead by <strong>unicast plus CDNs</strong> (edge replication), which fit the internet's
per-connection, multi-provider economics.
</details>

---

## Homework

Turn IGMP snooping off on the bridge (`echo 0 > /sys/class/net/br0/bridge/multicast_snooping`) and
re-run the lab while capturing on a *third*, non-member receiver's port. Compare what that non-member
sees with snooping on vs off, and explain the bandwidth implication at scale.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With snooping <strong>on</strong>, the non-member's port receives <em>none</em> of the 239.1.1.1
stream — the bridge forwards it only to ports that issued IGMP joins. With snooping <strong>off</strong>,
the bridge can't tell members from non-members and <strong>floods the multicast to every port</strong>,
so the non-member captures the full stream even though it never joined. The bandwidth implication at
scale: without snooping, a single high-rate multicast feed (e.g. a video stream) is blasted out of every
switch port, so every host and every link carries it regardless of interest — on a busy segment with
many groups this can saturate links and overwhelm hosts' NICs/CPUs with unwanted traffic. Snooping
confines each stream to the branches that asked for it, which is what makes multicast scale on real
networks; turning it off effectively degrades multicast back into broadcast.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 72 — Time Synchronization (NTP & PTP) →](lesson-72-time-sync){: .btn .btn-primary }
