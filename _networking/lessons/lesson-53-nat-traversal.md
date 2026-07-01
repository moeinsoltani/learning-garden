---
title: "Lesson 53 — NAT Traversal"
nav_order: 53
parent: "Phase 16: Tunnels & VPNs"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 53: NAT Traversal — STUN, ICE, UDP Hole Punching

## Concept

You now have an encrypted tunnel (WireGuard). But there's a brutal real-world problem: **both peers
are usually behind NAT** (Lesson 22), with no public IP and no open inbound port. Neither can simply
"connect to" the other. Making two NATed boxes talk *directly* is the hardest problem in
peer-to-peer VPNs — and the thing that distinguishes a toy tunnel from a real mesh.

```
   Peer A                NAT-A          Internet          NAT-B               Peer B
   10.0.0.5  ──►  [203.0.113.7:40000]   ←──?──►   [198.51.100.9:50000]  ◄──  10.0.0.9
       (no public IP)                                          (no public IP)
   Nobody can initiate inbound to either side. How do they meet in the middle?
```

The trick is **UDP hole punching**: get *both* sides to send outbound at the same time so each
NAT creates a mapping that lets the other's packets back in.

---

## How it works

**Step 1 — Discover your public mapping (STUN).** A peer behind NAT doesn't know what public
IP:port its NAT assigned it. It asks a [STUN](https://en.wikipedia.org/wiki/STUN) server on the
public internet "what address did my packet come from?" The reply reveals the peer's *external*
mapping (e.g. `203.0.113.7:40000`). Both peers do this and exchange their discovered mappings
through a **signaling channel** (a coordination server — Lesson 55).

**Step 2 — Punch the holes.** Each peer now sends UDP packets *to the other's discovered mapping*.
The first packets out of NAT-A create an outbound mapping for "A→B"; likewise NAT-B for "B→A".
Because both opened outbound at nearly the same moment, each NAT now has state that permits the
other side's incoming packets. The hole is punched; direct traffic flows.

```
   A ──► B's mapping   (opens A's NAT outbound; B's packet not allowed yet)
   B ──► A's mapping   (opens B's NAT outbound; now A's earlier mapping lets it in)
   ✓ both NATs have matching state → direct UDP path established
```

**NAT behavior decides whether this works.** NATs differ in how they assign external mappings:

| NAT type | Mapping behavior | Hole punching |
|---|---|---|
| Full-cone | One external port for any destination | Easiest |
| Address/port-restricted | Same port, but only accepts from contacted peers | Works with simultaneous send |
| **Symmetric** | A *different* external port per destination | Hard — the port you told B isn't the one used toward B |

**Symmetric NAT** is the villain: it picks a fresh external port for each distinct destination, so
the mapping a peer learned from the STUN server (toward the STUN server) is **not** the port the
NAT will use toward the other peer. The peer is trying to reach the wrong port.

{: .note }
> **The birthday-paradox trick**
> Against symmetric NAT, some implementations spray packets across *many* candidate ports while the
> peer does the same. By the birthday paradox, a surprisingly small number of guesses on each side
> gives a high chance that one pair of ports collides and connects. It's probabilistic, noisy, and
> not always allowed by the NAT — which is exactly why a relay fallback (Lesson 54) must exist.

**ICE** ([Interactive Connectivity Establishment](https://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment))
is the standardized framework that ties this together: gather *candidates* (local address, STUN-derived
"server-reflexive" address, relayed address), exchange them, and systematically test pairs until one
works — preferring direct over relayed. WebRTC uses ICE; mesh VPNs implement their own ICE-like logic.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `stun <server>` / `stunclient <server>` | Query a STUN server for your mapping |
| `nc -u -p <port> <ip> <port>` | Manually send UDP to test a punched path |
| `conntrack -L -p udp` | Watch the NAT mapping a punch creates (Lesson 23) |
| `tcpdump -ni <if> udp port <p>` | Observe the simultaneous outbound sends |

---

## Lab

Simulate two NATs with namespaces and demonstrate a hole punch. Topology: two "client" namespaces,
each behind a "router" namespace doing masquerade (Lesson 22), both connected to a shared "internet"
namespace.

### Step 1 — Build two NATed clients (sketch)

```bash
# For each side: client → router(masquerade) → internet bridge
# Reuse the NAT topology from Lesson 22 twice (lan1/router1, lan2/router2),
# joined by a common 'inet' namespace acting as the public segment + STUN host.
$ sudo ip netns add inet
# ... create router1/lan1 and router2/lan2, masquerade on each router's inet side ...
```

### Step 2 — Discover each client's external mapping

```bash
# From lan1, send a UDP packet to the 'inet' host and read the source it sees:
$ sudo ip netns exec inet tcpdump -ni <inet-if> udp port 3478 &
$ sudo ip netns exec lan1 nc -u -p 40000 <inet-ip> 3478   # type something
# tcpdump shows the packet's source = router1's external IP:port → lan1's mapping
# repeat from lan2 to learn router2's mapping
```

### Step 3 — Punch: both clients send to the other's mapping at once

```bash
# Tell each side the other's discovered external IP:port, then both send:
$ sudo ip netns exec lan1 nc -u -p 40000 <router2-ext-ip> <router2-ext-port> &
$ sudo ip netns exec lan2 nc -u -p 40000 <router1-ext-ip> <router1-ext-port> &
# Once both outbound mappings exist, the two nc's exchange text directly — no relay.
```

### Step 4 — Confirm the mapping on the routers

```bash
$ sudo ip netns exec router1 conntrack -L -p udp
udp ... src=<lan1> dst=<router2-ext> ... [ASSURED]   # the punched hole
```

### Step 5 — Clean up

```bash
$ sudo ip netns delete inet lan1 router1 lan2 router2
```

---

## Further Reading

| Topic | Link |
|---|---|
| NAT traversal | [Wikipedia — NAT traversal](https://en.wikipedia.org/wiki/NAT_traversal) |
| STUN | [Wikipedia — STUN](https://en.wikipedia.org/wiki/STUN) |
| ICE | [Wikipedia — Interactive Connectivity Establishment](https://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment) |
| UDP hole punching | [Wikipedia — UDP hole punching](https://en.wikipedia.org/wiki/UDP_hole_punching) |
| Network address translation | [Wikipedia — NAT](https://en.wikipedia.org/wiki/Network_address_translation) |

---

## Checkpoint

**Q1. Why can't two peers both behind NAT simply connect to each other, and how does UDP hole punching get around it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Neither peer has a public IP or an open inbound port — their NATs only create mappings in response to <em>outbound</em> traffic and drop unsolicited inbound packets, so a "connect to the other" attempt arrives at a NAT with no matching mapping and is discarded. UDP hole punching solves it by making <strong>both sides send outbound roughly simultaneously</strong> to each other's discovered public mapping. A's outbound packet makes NAT-A create a mapping permitting return traffic from B's address; B's outbound does the same on NAT-B. Once both mappings exist, each NAT now treats the other's incoming packets as replies to its own outbound flow and lets them through — a direct path is established without either side being publicly reachable beforehand.
</details>

---

**Q2. What does a STUN server do, and why is it necessary before hole punching?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A peer behind NAT doesn't know what public IP:port its NAT will present to the outside world — it only knows its private address. A <strong>STUN</strong> server sits on the public internet and, when the peer sends it a packet, replies with the <em>source address it observed</em> — i.e. the peer's external NAT mapping (the "server-reflexive" address). This is necessary because hole punching requires each peer to tell the other where to aim, and that target is the external mapping, not the private address. Both peers STUN to learn their own mappings, exchange them via a signaling/coordination channel, and then punch toward each other's discovered address.
</details>

---

**Q3. Why does symmetric NAT break ordinary hole punching, and what are the options when it does?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A <strong>symmetric NAT</strong> assigns a <em>different external port for every distinct destination</em>. So the mapping a peer learns from the STUN server is the port the NAT uses <em>toward the STUN server</em> — but when that peer later sends toward the other peer, the NAT picks a <em>new, different</em> port. The address the peers exchanged is therefore wrong for their direct path, and packets hit a closed door. Options: (1) the <strong>birthday-paradox / port-prediction</strong> trick — both sides spray many candidate ports hoping one pair collides (probabilistic, sometimes blocked); and (2) when direct punching fails, fall back to a <strong>relay</strong> (Lesson 54) that both peers can reach outbound, which forwards their encrypted traffic. A robust mesh always keeps the relay as a guaranteed fallback because some NATs simply cannot be traversed directly.
</details>

---

## Homework

Modify your lab's two routers so that *one* of them behaves like a symmetric NAT (hint: nftables
masquerade with `random` / fully-random port allocation, versus the default which tries to preserve
the source port). Re-run the hole-punch attempt and observe it fail. Then describe precisely *why*
the previously-exchanged mapping no longer works, and what your mesh would have to do next.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With the symmetric-style masquerade (fully randomized source ports per destination), the hole punch
fails: the external port that the symmetric side advertised (learned by sending to the STUN/inet
host) is <em>not</em> the port its NAT allocates when it later sends toward the other peer. So when
the non-symmetric peer aims at the advertised port, the symmetric NAT has no inbound mapping on that
port (it opened a different one toward the peer), and the packets are dropped; meanwhile the symmetric
side's packets arrive at the other peer from an unexpected source port. The exchanged candidate is
stale by construction. What the mesh must do next: (1) try <strong>port prediction / birthday-paradox
spraying</strong> across many ports to get a lucky collision, and if that fails (or the NAT forbids
it), (2) <strong>fall back to a relay</strong> — both peers connect outbound to a mutually reachable
relay server that forwards their still-end-to-end-encrypted packets. This is exactly why the next
lesson exists: a production mesh can never assume a direct path is achievable.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 54 — Relays and Fallback Paths →](lesson-54-relays){: .btn .btn-primary }
