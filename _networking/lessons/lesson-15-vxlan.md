---
title: "Lesson 15 — VXLAN"
nav_order: 15
parent: "Phase 4: Overlay Networks"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 15: VXLAN

## Concept

**VXLAN** (Virtual eXtensible LAN) lets you build a virtual Layer-2 network that spans multiple hosts connected only by Layer-3 (routed IP) infrastructure. It does this by **encapsulation**: it wraps each inner Ethernet frame inside a UDP packet and ships it across the IP network to the right destination host, which unwraps it and delivers the original frame.

This is the foundation of Kubernetes pod networking (Flannel, Calico VXLAN mode) and data-center fabrics: pods/VMs on different physical nodes *think* they're on one big flat L2 network, while the underlying packets are just ordinary routable UDP.

```
   Host A (10.0.0.1)                              Host B (10.0.0.2)
   ┌─────────────────┐      L3 routed network     ┌─────────────────┐
   │ ns1             │       (could be many        │ ns2             │
   │ 192.168.100.1   │        router hops)         │ 192.168.100.2   │
   │      │          │                             │      │          │
   │   vxlan0 ───────┼─── UDP/4789 encapsulation ──┼──── vxlan0      │
   └─────────────────┘                             └─────────────────┘
        the two namespaces believe they share one L2 segment
        even though A and B are only connected via routed IP
```

---

## How it works

A VXLAN interface is a **VTEP** — VXLAN Tunnel EndPoint. When a frame needs to reach a remote host, the VTEP:

1. Takes the original inner Ethernet frame.
2. Prepends a **VXLAN header** containing a 24-bit **VNI** (VXLAN Network Identifier — the "which virtual network" tag, up to ~16 million vs VLAN's 4094).
3. Wraps that in **UDP** (destination port **4789**), then in an outer IP header addressed to the remote host's real IP, then an outer Ethernet header.
4. Sends it onto the physical/underlay network like any normal packet.

```
   [ outer Eth | outer IP (A→B) | UDP dport 4789 | VXLAN (VNI 42) | INNER Eth frame ]
     └────────────────── added by the VTEP ──────────────────┘   └─ original frame ─┘
```

The receiving VTEP strips all the outer headers and delivers the inner frame locally. The inner hosts never see any of the wrapping.

**How does the VTEP know which host to send to?** Two approaches:

- **Multicast VXLAN** — VTEPs join an IP multicast group; unknown/broadcast frames are flooded via multicast. Needs multicast support in the underlay (often unavailable).
- **Unicast VXLAN with manual/learned FDB** — you populate the VXLAN forwarding database with "inner MAC → remote VTEP IP" entries (or a controller does it, as Flannel does by watching the Kubernetes API). This is what's used in practice.

{: .note }
> **Why the MTU shrinks**
> Encapsulation adds overhead — the VXLAN+UDP+IP+Ethernet outer headers total **50 bytes** (IPv4 underlay). If the underlay MTU is 1500, the inner interface must use a smaller MTU (typically 1450) or large inner frames will need fragmentation or be dropped. MTU mismatches are the single most common cause of "VXLAN works for ping but breaks for big transfers." Always account for the 50-byte overhead.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip link add vxlan0 type vxlan id 42 dstport 4789 remote <ip> dev <iface>` | Create a point-to-point VXLAN VTEP |
| `ip link add vxlan0 type vxlan id 42 dstport 4789 dev <iface>` | Create a VTEP with no fixed remote (use FDB) |
| `bridge fdb append <mac> dev vxlan0 dst <remote-ip>` | Add a forwarding entry: reach this MAC via this remote VTEP |
| `bridge fdb show dev vxlan0` | Show the VXLAN forwarding database |
| `ip -d link show vxlan0` | Show VXLAN details (VNI, dstport, remote) |

---

## Lab

We'll simulate two hosts as two namespaces (`hostA`, `hostB`) connected by a routed "underlay," then build a VXLAN between them so two *inner* namespaces share an overlay L2 network.

To keep it runnable on one machine, we put each "host" in its own netns and connect them with a veth that represents the L3 underlay.

### Step 1 — Build the underlay (two "hosts" on a routed link)

```bash
$ sudo ip netns add hostA
$ sudo ip netns add hostB
$ sudo ip link add ulA type veth peer name ulB
$ sudo ip link set ulA netns hostA
$ sudo ip link set ulB netns hostB

$ sudo ip netns exec hostA ip link set ulA up
$ sudo ip netns exec hostA ip addr add 10.0.0.1/24 dev ulA
$ sudo ip netns exec hostB ip link set ulB up
$ sudo ip netns exec hostB ip addr add 10.0.0.2/24 dev ulB

# Confirm underlay connectivity
$ sudo ip netns exec hostA ping -c 1 10.0.0.2
```

### Step 2 — Create a VXLAN VTEP on each host

```bash
# Host A: VNI 42, peer is Host B's underlay IP
$ sudo ip netns exec hostA ip link add vxlan0 type vxlan id 42 \
      dstport 4789 remote 10.0.0.2 dev ulA
$ sudo ip netns exec hostA ip link set vxlan0 up
$ sudo ip netns exec hostA ip addr add 192.168.100.1/24 dev vxlan0

# Host B: mirror image, peer is Host A
$ sudo ip netns exec hostB ip link add vxlan0 type vxlan id 42 \
      dstport 4789 remote 10.0.0.1 dev ulB
$ sudo ip netns exec hostB ip link set vxlan0 up
$ sudo ip netns exec hostB ip addr add 192.168.100.2/24 dev vxlan0
```

We used the simple `remote` form (point-to-point), so each VTEP already knows its single peer.

### Step 3 — Ping across the overlay

```bash
$ sudo ip netns exec hostA ping -c 2 192.168.100.2
64 bytes from 192.168.100.2: icmp_seq=1 ttl=64 time=0.12 ms
```

`192.168.100.1` and `.2` appear to be on the same L2 segment, but the packets actually traveled as UDP across the `10.0.0.0/24` underlay.

### Step 4 — Capture the encapsulation

This is the key observation. Capture on the **underlay** interface while pinging the overlay:

```bash
# Capture the UNDERLAY in hostA
$ sudo ip netns exec hostA tcpdump -n -i ulA udp port 4789 &

$ sudo ip netns exec hostA ping -c 1 192.168.100.2
```

You'll see the outer packet — VXLAN over UDP:

```
10.0.0.1.xxxxx > 10.0.0.2.4789: VXLAN, flags [I] (0x08), vni 42
   IP 192.168.100.1 > 192.168.100.2: ICMP echo request ...
└─ OUTER: real host IPs, UDP/4789  ──┘ └─ INNER: the overlay frame ─┘
```

The outer header carries the real host IPs and UDP port 4789; tcpdump decodes the VXLAN header (VNI 42) and then the *inner* ICMP. That nested structure — IP packet inside UDP inside IP — is the whole point of an overlay.

### Step 5 — Inspect the VXLAN FDB

```bash
$ sudo ip netns exec hostA bridge fdb show dev vxlan0
00:00:00:00:00:00 dst 10.0.0.2 self permanent
# the all-zeros entry = "flood unknown/broadcast frames to 10.0.0.2"
```

With the `remote` form, the kernel auto-installed a flood entry pointing at the peer. In a multi-host setup you'd manually `bridge fdb append <remote-mac> dev vxlan0 dst <remote-host-ip>` for each peer (this is exactly what a CNI controller automates).

### Step 6 — Clean up

```bash
$ sudo ip netns delete hostA
$ sudo ip netns delete hostB
```

---

## Further Reading

| Topic | Link |
|---|---|
| VXLAN | [Wikipedia — Virtual Extensible LAN](https://en.wikipedia.org/wiki/Virtual_Extensible_LAN) |
| VXLAN RFC 7348 | [Wikipedia — VXLAN (RFC 7348)](https://en.wikipedia.org/wiki/Virtual_Extensible_LAN) |
| Overlay network | [Wikipedia — Overlay network](https://en.wikipedia.org/wiki/Overlay_network) |
| Linux VXLAN driver | [kernel.org — vxlan](https://www.kernel.org/doc/Documentation/networking/vxlan.txt) |
| Tunneling protocol | [Wikipedia — Tunneling protocol](https://en.wikipedia.org/wiki/Tunneling_protocol) |

---

## Checkpoint

**Q1. How does a Layer-2 frame travel across a Layer-3 (routed) network? What wraps what?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The L2 frame is **encapsulated** — wrapped inside higher-layer headers that the L3 network can route. Working from the inside out: the original inner Ethernet frame is prepended with a VXLAN header (carrying the 24-bit VNI), which is placed inside a UDP datagram (destination port 4789), which is placed inside an outer IP header addressed to the *destination host's* real IP, which is finally placed inside an outer Ethernet header for the first hop. The routed underlay sees only an ordinary UDP/IP packet and forwards it hop by hop using normal routing. At the far end, the receiving VTEP strips all the outer layers and delivers the original inner frame locally. So: inner Ethernet ⊂ VXLAN ⊂ UDP ⊂ outer IP ⊂ outer Ethernet. The L2 network is "virtual" because it only exists as payload inside L3 packets.
</details>

---

**Q2. What is the VNI, and how does it compare to a VLAN ID?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The VNI (VXLAN Network Identifier) is a **24-bit** field in the VXLAN header that identifies which virtual network a frame belongs to — analogous to a VLAN ID, but much larger. A VLAN ID is only 12 bits, giving ~4094 usable networks; a VNI's 24 bits give ~16 million. Beyond scale, the crucial difference is reach: a VLAN tag only separates networks within a single L2 broadcast domain (one switched network), whereas a VNI separates overlay networks that can span an arbitrarily large *routed* L3 infrastructure via encapsulation. So VXLAN/VNI is what you use when you've outgrown VLANs either in number (need more than 4094 segments, common in multi-tenant clouds) or in span (need one logical L2 across many routed hosts).
</details>

---

**Q3. A VXLAN overlay works fine for `ping` but large file transfers stall or fail. What's the most likely cause and the fix?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An **MTU mismatch** caused by encapsulation overhead. VXLAN adds ~50 bytes of outer headers (VXLAN + UDP + outer IP + outer Ethernet) to every frame. If the inner/overlay interface still uses a 1500-byte MTU while the underlay is also 1500, a full-size inner packet becomes ~1550 bytes on the wire — too big — so it gets fragmented or dropped. Small packets like ICMP echoes fit fine, which is why ping "works" while bulk TCP (which uses full-size segments) stalls. The fix is to lower the inner/overlay MTU to account for the overhead (e.g., set the VXLAN interface MTU to 1450), or raise the underlay MTU (jumbo frames) so the encapsulated packets still fit. This is the single most common VXLAN gotcha.
</details>

---

## Homework

Extend the lab to **three** hosts (hostA, hostB, hostC) on one underlay subnet, all sharing VNI 42. Instead of the simple `remote` form, create each VTEP with no fixed remote and manually populate the VXLAN FDB so every host can reach every other. Then capture the underlay while one host sends a *broadcast* (e.g. an ARP) and explain how the frame reaches both other hosts.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Setup outline: create each VTEP with `ip link add vxlan0 type vxlan id 42 dstport 4789 dev <ul>` (no `remote`). Then, on each host, add an all-zeros flood entry for *each* peer:

```
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst <peerB-underlay-ip>
bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst <peerC-underlay-ip>
```

The all-zeros MAC entry means "for unknown-unicast and broadcast/multicast frames, replicate to this remote VTEP." With two such entries, a broadcast frame is sent (head-end replicated) to *both* peers.

When hostA sends an ARP broadcast on the overlay: its VTEP sees a broadcast destination, looks up the all-zeros FDB entries, and **replicates** the encapsulated frame once per peer — one UDP/4789 packet to hostB's underlay IP and another to hostC's. The underlay capture shows two separate outer UDP packets leaving hostA (unicast to B and to C), each containing the same inner ARP. Each receiving VTEP decapsulates and floods the ARP onto its local overlay. Specific unicast replies then get learned into the FDB as `<mac> dst <host-ip>` entries, so subsequent unicast traffic goes to exactly one peer instead of being replicated. This head-end replication is how unicast VXLAN emulates L2 broadcast without underlay multicast — and it's exactly what a CNI like Flannel automates by watching the cluster's node list.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 16 — Routing Fundamentals →](lesson-16-routing-fundamentals){: .btn .btn-primary }
