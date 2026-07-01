---
title: "Lesson 48 — Tunneling Fundamentals"
nav_order: 48
parent: "Phase 16: Tunnels & VPNs"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 48: Tunneling Fundamentals — GRE, IPIP, SIT, FOU

## Concept

A **tunnel** carries one network's packets *inside* another network's packets. The kernel takes
your original packet (the **inner** packet), wraps it in a new header (the **outer** packet),
and ships it across the intervening network as if it were ordinary traffic. At the far end the
wrapper is stripped and the inner packet pops out, untouched. This is **encapsulation** — exactly
the idea you met with VXLAN in Phase 4, but here without any cryptography.

```
   Original packet (inner)                 What travels on the wire (outer)
   ┌───────────────┐                       ┌────────────┬───────────────┐
   │ IP: 10.0.0.2  │  ── encapsulate ──►   │ outer IP   │ IP: 10.0.0.2  │
   │   payload     │                       │ 203.0.113.x│   payload     │
   └───────────────┘                       └────────────┴───────────────┘
                                                  └ routed across the real network ┘
```

A VPN is just a tunnel with encryption and authentication added (Lessons 49–51). Understanding
the plain tunnel first means the crypto later is the *only* new idea.

---

## How it works

The kernel offers several encapsulation types, differing only in what the outer header looks like:

| Type | Outer header | Carries | Typical use |
|---|---|---|---|
| `ipip` | IPv4 | IPv4 | Simplest point-to-point IPv4-in-IPv4 |
| `sit`  | IPv4 | IPv6 | IPv6-over-IPv4 (transition) |
| `gre`  | IPv4 + GRE | IPv4/IPv6/multicast | General-purpose; carries more than one protocol |
| `gretap` | IPv4 + GRE | **Ethernet frames** | L2 over L3 (like VXLAN, unicast) |
| `fou`/`gue` | IPv4 + **UDP** | another tunnel | Wrap a tunnel in UDP so NAT/ECMP handle it |

[GRE](https://en.wikipedia.org/wiki/Generic_Routing_Encapsulation) is the workhorse because it
adds a small protocol-type field, so one tunnel can carry IPv4, IPv6, or even multicast. `ipip`
and `sit` are leaner but single-protocol.

{: .note }
> **Why UDP encapsulation (FOU/GUE) matters**
> Plain GRE/IPsec use their own IP protocol numbers (47, 50). Many NATs and load balancers only
> understand TCP/UDP and either drop or fail to balance other protocols. Wrapping the tunnel in
> **UDP** (Foo-over-UDP / Generic UDP Encapsulation) makes it look like ordinary traffic that NAT
> can translate and ECMP can hash across links. This is the same reason WireGuard is UDP-only —
> you'll see this theme again in NAT traversal (Lesson 53).

**The MTU problem.** Every wrapper adds bytes. If the inner packet is already 1500 bytes and you
add a 24-byte GRE+IP header, the 1524-byte result won't fit a 1500-byte link — it must fragment
or be dropped. So a tunnel **lowers the effective MTU** for traffic inside it. Either lower the
inner interface's MTU or rely on Path MTU Discovery (which breaks if ICMP is filtered — a classic
"tunnel works for small packets, hangs on large ones" bug).

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip link add <name> type gre local <A> remote <B>` | Create a GRE tunnel |
| `ip link add <name> type ipip local <A> remote <B>` | Create an IPIP tunnel |
| `ip tunnel show` | List configured tunnels |
| `ip link set <name> mtu 1400` | Lower the tunnel MTU |
| `tcpdump -ni <if> proto gre` | Capture GRE-encapsulated traffic |

---

## Lab

We'll build a GRE tunnel between two namespaces and watch the inner packet ride inside an outer one.

### Step 1 — Two namespaces with an "underlay" link

```bash
$ sudo ip netns add a
$ sudo ip netns add b
$ sudo ip link add a-b type veth peer name b-a
$ sudo ip link set a-b netns a
$ sudo ip link set b-a netns b
$ sudo ip netns exec a ip addr add 10.0.0.1/24 dev a-b
$ sudo ip netns exec b ip addr add 10.0.0.2/24 dev b-a
$ sudo ip netns exec a ip link set a-b up
$ sudo ip netns exec b ip link set b-a up
```

### Step 2 — Build a GRE tunnel over that underlay

```bash
# In namespace a
$ sudo ip netns exec a ip link add gre0 type gre local 10.0.0.1 remote 10.0.0.2
$ sudo ip netns exec a ip addr add 192.168.99.1/30 dev gre0
$ sudo ip netns exec a ip link set gre0 up

# In namespace b
$ sudo ip netns exec b ip link add gre0 type gre local 10.0.0.2 remote 10.0.0.1
$ sudo ip netns exec b ip addr add 192.168.99.2/30 dev gre0
$ sudo ip netns exec b ip link set gre0 up
```

### Step 3 — Send inner traffic and capture the outer

```bash
# Capture the underlay while pinging the tunnel address
$ sudo ip netns exec b tcpdump -ni b-a proto gre &
$ sudo ip netns exec a ping -c 2 192.168.99.2
```

Expected: the ping (inner addresses `192.168.99.x`) succeeds, while tcpdump on the underlay shows
**GRE** packets between the *outer* addresses `10.0.0.1 → 10.0.0.2`:

```
IP 10.0.0.1 > 10.0.0.2: GREv0, ... IP 192.168.99.1 > 192.168.99.2: ICMP echo request
```

The single capture line shows the wrapping: outer `10.0.0.x` carrying inner `192.168.99.x`.

### Step 4 — See the MTU effect

```bash
$ sudo ip netns exec a ip link show gre0 | grep mtu
# mtu 1476  ← 1500 underlay minus the 24-byte GRE+IP overhead
```

### Step 5 — Clean up

```bash
$ sudo ip netns delete a b
```

---

## Further Reading

| Topic | Link |
|---|---|
| GRE | [Wikipedia — Generic Routing Encapsulation](https://en.wikipedia.org/wiki/Generic_Routing_Encapsulation) |
| Tunneling protocol | [Wikipedia — Tunneling protocol](https://en.wikipedia.org/wiki/Tunneling_protocol) |
| IP-in-IP | [Wikipedia — IP in IP](https://en.wikipedia.org/wiki/IP_in_IP) |
| MTU | [Wikipedia — Maximum transmission unit](https://en.wikipedia.org/wiki/Maximum_transmission_unit) |
| `ip-link` (tunnel types) | [man7.org — ip-link(8)](https://man7.org/linux/man-pages/man8/ip-link.8.html) |

---

## Checkpoint

**Q1. What is the difference between the inner and outer packet in a tunnel, and which addresses does the underlay network route on?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The <strong>inner packet</strong> is your original traffic (e.g. source/dest <code>192.168.99.x</code>), and the <strong>outer packet</strong> is the wrapper the kernel adds (source/dest = the tunnel endpoints, e.g. <code>10.0.0.x</code>). The underlay network only sees and routes on the <strong>outer</strong> addresses — it has no idea there's another packet inside. The inner addresses only become meaningful again after the far endpoint strips the wrapper. This is why a tunnel can carry "private" addressing across a network that knows nothing about it.
</details>

---

**Q2. Why does a tunnel lower the effective MTU, and what symptom does ignoring it produce?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Each encapsulation layer prepends a new header (GRE+IP ≈ 24 bytes, UDP-wrapped more). The outer packet is therefore larger than the inner one, so the largest inner packet that still fits the underlay's MTU is reduced (1500 → 1476 for GRE). If you ignore it, small packets (pings, TCP handshakes) work fine, but large packets exceed the path MTU and must fragment or get dropped. With Path MTU Discovery relying on ICMP "fragmentation needed" messages — which firewalls often filter — the result is the classic hang: SSH connects but a large paste freezes, web pages start loading then stall. The fix is lowering the tunnel/inner MTU (or MSS clamping) so packets fit with overhead included.
</details>

---

**Q3. Why might you wrap a tunnel inside UDP (FOU/GUE) instead of using GRE or ESP directly?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
GRE (IP protocol 47) and IPsec ESP (protocol 50) are not TCP or UDP, and a lot of middleboxes — home NAT routers, cloud load balancers, ECMP hashing — only understand TCP/UDP. They may drop the unknown protocol, fail to NAT it (no ports to translate), or pin the whole flow to a single path because they can't hash it. Encapsulating the tunnel in UDP gives it real source/destination ports, so NAT can translate it and ECMP can spread it across links, and it traverses restrictive networks that only allow "normal" traffic. This is the same design reason WireGuard chose to be UDP-only, and it foreshadows the NAT-traversal problem in Lesson 53.
</details>

---

## Homework

Build an **IPIP** tunnel (instead of GRE) between the same two namespaces, then try to send a
*multicast* ping (`ping 224.0.0.1`) and an IPv6 packet through it. Note what works and what
doesn't, then switch the tunnel to GRE and retry. Explain the difference you observe.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With <strong>IPIP</strong>, unicast IPv4 works, but multicast and IPv6 do not: <code>ipip</code> encapsulates only unicast IPv4-in-IPv4 — it has no protocol-type field to distinguish other payloads, and multicast isn't carried. (IPv6 specifically needs a <code>sit</code> tunnel.) When you switch to <strong>GRE</strong>, multicast and IPv6 start working, because GRE includes a 16-bit protocol-type field in its header that says "the inner payload is IPv4 / IPv6 / a multicast frame," so one GRE tunnel can multiplex several protocols. The lesson: <code>ipip</code>/<code>sit</code> are minimal single-purpose tunnels, while GRE trades a few extra header bytes for generality — which is exactly why routing protocols that rely on multicast (e.g. OSPF) are run over GRE, not IPIP.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 49 — VPN Cryptography Building Blocks →](lesson-49-vpn-crypto){: .btn .btn-primary }
