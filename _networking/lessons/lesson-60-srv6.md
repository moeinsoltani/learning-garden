---
title: "Lesson 60 — Segment Routing & SRv6"
nav_order: 60
parent: "Phase 17: Advanced Routing & Data-Center Fabrics"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 60: Segment Routing & SRv6

## Concept

Normal routing is **destination-based**: every router independently decides the next hop from the
destination address, and the network core holds per-flow state for any traffic engineering. **Segment
Routing (SR)** flips this: the **source** encodes the path as an ordered list of *segments*
(waypoints/instructions) into the packet itself, and the core just follows the list. The core keeps
*no per-flow state* — the instructions ride in the packet.

```
   Destination routing:  packet → R1(decides) → R2(decides) → R3(decides) → dst
   Segment routing:      source writes [via R2, via R5, to dst] into the packet;
                         routers just pop/follow the list.
```

[SRv6](https://en.wikipedia.org/wiki/Segment_Routing) is the IPv6 flavor: each segment is a full
**IPv6 address (a SID)**, carried in an IPv6 extension header (SRH).

---

## How it works

**Segments / SIDs.** A *segment* is an instruction identified by an ID (SID). It can mean "route to
node N" (a waypoint) or "perform function F" (e.g. forward out a specific link, decapsulate, hand to
a VRF). The source builds a **segment list** — the ordered path/program.

**Two encodings.** SR-MPLS uses MPLS labels as SIDs (runs on existing MPLS cores). **SRv6** uses
IPv6 addresses as SIDs and a **Segment Routing Header (SRH)** as an IPv6 extension header; the
"active" segment is the IPv6 destination address, and a pointer advances through the list as each
segment is completed.

**Stateless traffic engineering.** Because the path lives in the packet, you can steer a flow along
a non-shortest path (avoid a congested link, pin latency-sensitive traffic to a low-delay route)
*without* configuring per-flow state in every core router — only the ingress node imposes the segment
list. This removes the scaling pain of older TE (RSVP-TE).

**On Linux.** The kernel's `seg6` support lets you impose an SRH on matching traffic with
`ip route ... encap seg6` and define local SID behaviors with `ip -6 route add <SID> encap seg6local`.

{: .note }
> **SID functions ("network programming")**
> SRv6's power is that a SID isn't just "go here" — it can encode a *behavior*: `End` (continue down
> the list), `End.X` (forward out a specific interface), `End.DT4/DT6` (decapsulate and look up in a
> table — used for VPNs). A segment list becomes a little program the packet carries, which is why
> SRv6 is described as "network programming."

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `sysctl net.ipv6.conf.all.seg6_enabled=1` | Enable SRv6 processing |
| `ip route add <dst> encap seg6 mode encap segs <SID1>,<SID2> dev <if>` | Steer traffic via a segment list |
| `ip -6 route add <SID> encap seg6local action End dev <if>` | Define a local SID behavior |
| `ip -6 route show` | See SRv6 routes/SIDs |
| `tcpdump -ni <if> ip6` | Observe the SRH (routing header type 4) |

---

## Lab

Steer traffic from a source through an explicit waypoint using SRv6, instead of the shortest path.

### Step 1 — Three IPv6 namespaces in a line: src — mid — dst, plus a direct src—dst link

```bash
$ for n in src mid dst; do sudo ip netns add $n; done
# Give src TWO paths to dst: a direct link, and an indirect one via mid.
# Assign IPv6 addrs; enable forwarding and seg6 on all.
$ for n in src mid dst; do sudo ip netns exec $n sysctl -w net.ipv6.conf.all.seg6_enabled=1; done
```

### Step 2 — Define a local SID on mid

```bash
# mid's SID 'fc00:bbbb::1' = End behavior (process and continue)
$ sudo ip netns exec mid ip -6 route add fc00:bbbb::1 encap seg6local action End dev <mid-if>
```

### Step 3 — On src, steer dst-bound traffic via mid's SID

```bash
$ sudo ip netns exec src ip route add <dst-addr> encap seg6 mode encap \
    segs fc00:bbbb::1 dev <src-if>
```

### Step 4 — Verify the path goes via mid, not the direct link

```bash
$ sudo ip netns exec mid tcpdump -ni <mid-if> ip6 &
$ sudo ip netns exec src ping6 -c2 <dst-addr>
# tcpdump on mid sees the traffic (with an SRH / routing header), proving it detoured via the waypoint
# even though a direct src→dst link exists.
```

### Step 5 — Clean up

```bash
$ sudo ip netns delete src mid dst
```

---

## Further Reading

| Topic | Link |
|---|---|
| Segment routing | [Wikipedia — Segment routing](https://en.wikipedia.org/wiki/Segment_Routing) |
| Source routing | [Wikipedia — Source routing](https://en.wikipedia.org/wiki/Source_routing) |
| Linux SRv6 | [segment-routing.org](https://www.segment-routing.org/) |
| `ip-route` (encap) | [man7.org — ip-route(8)](https://man7.org/linux/man-pages/man8/ip-route.8.html) |

---

## Checkpoint

**Q1. How does segment routing move path decisions to the source, and what state does it remove from the network core?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
In segment routing the <strong>ingress/source node encodes the entire path as an ordered list of
segments (SIDs) into the packet</strong> — e.g. "via waypoint A, via waypoint B, to destination."
Core routers don't make traffic-engineering decisions; they simply forward toward the current active
segment and advance the list as each is completed. This removes <strong>per-flow traffic-engineering
state from the core</strong>: traditional TE (like RSVP-TE) had to install and maintain state for
each engineered path in every router along it, which scales poorly. With SR, only the packet carries
the path, so the core stays stateless and just follows instructions — you can steer arbitrarily many
flows along arbitrary paths by changing only the ingress, not the core.
</details>

---

**Q2. In SRv6, what is a SID and where does the segment list travel?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
In SRv6 a <strong>SID (Segment Identifier) is a full IPv6 address</strong> that identifies a segment
— either a waypoint ("route to this node") or a function/behavior ("forward out interface X,"
"decapsulate and look up in table T"). The ordered <strong>segment list travels in the IPv6 Segment
Routing Header (SRH)</strong>, an IPv6 extension header. The currently active segment is placed in
the packet's IPv6 <em>destination address</em> field, and a pointer in the SRH advances through the
list so the next SID becomes the destination as each segment completes. (The MPLS variant, SR-MPLS,
instead encodes SIDs as a stack of MPLS labels.)
</details>

---

**Q3. Give a concrete situation where steering traffic along a non-shortest path with SR is valuable.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Any case where the shortest path isn't the <em>best</em> path for a particular flow. Examples:
(1) <strong>Latency-sensitive traffic</strong> — pin voice/trading traffic onto a known low-delay
link even if it's not the IGP shortest path. (2) <strong>Congestion avoidance / traffic engineering</strong>
— route a bulk flow around a congested core link to balance utilization. (3) <strong>Service chaining</strong>
— force traffic through a firewall or DPI box (a SID with a function) before delivery. (4)
<strong>SLA separation</strong> — premium customers' traffic over a premium path. SR makes all of
these a matter of writing a segment list at the ingress, with no per-flow state in the core, which is
why providers adopted it for traffic engineering.
</details>

---

## Homework

Add a second waypoint so the segment list on `src` becomes two SIDs (via mid1, then mid2, then dst).
Capture on each waypoint and confirm the packet visits them in order, watching the SRH's
segments-left pointer decrement. Explain what the destination address field of the IPv6 header
contains at each hop.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With a two-SID list, captures show the packet visiting mid1 then mid2 then dst, in order. The
mechanism: the IPv6 <strong>destination address always holds the <em>current</em> active segment</strong>,
and the SRH carries the full list plus a <strong>"segments left" pointer</strong>. At the source the
destination = mid1's SID (segments-left = 2). When mid1 processes its End SID, it decrements
segments-left to 1 and rewrites the destination address to the next SID (mid2). mid2 does the same,
decrementing to 0 and setting the destination to the final dst address, so the last hop is plain IPv6
forwarding to dst. So each waypoint sees itself as the destination while it's the active segment, then
"hands off" by advancing the pointer and rewriting the destination — the segment list is consumed
front-to-back, exactly like stepping through a program's instructions.
</details>
