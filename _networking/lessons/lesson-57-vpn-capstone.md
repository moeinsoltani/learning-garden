---
title: "Lesson 57 — Capstone: An Encrypted Mesh Across NAT"
nav_order: 57
parent: "Phase 16: Tunnels & VPNs"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 57: Capstone — An Encrypted Mesh Across NAT

## Concept

This lesson ties Phase 16 together. You'll build three nodes where two sit behind simulated NAT,
have them establish **direct** WireGuard tunnels via hole punching, and then force one NAT to be
symmetric so traffic must **fall back to a relay** — observing the whole decision in packets.

```
   node A ── NAT-A ─┐                       ┌─ NAT-B ── node B
                    ├──  "internet" + relay ─┤
                    │   (coordination/STUN)  │
                 node C (public, doubles as relay/coordinator)

   Phase 1: A and B hole-punch → direct WireGuard
   Phase 2: make NAT-B symmetric → punch fails → A↔B via relay on C
```

Everything you learned — encapsulation, crypto, WireGuard, NAT traversal, relays, coordination —
shows up here at once.

---

## How it works

This is integration, not new theory. The pieces and where they came from:

- **Encrypted data plane:** WireGuard tunnels (Lessons 51–52) carry all node-to-node traffic.
- **NAT:** each edge node is behind an nftables masquerade router (Lesson 22), so it has no inbound
  reachability.
- **Discovery + signaling:** node C plays STUN + coordinator (Lessons 53, 55): each edge node
  learns its external mapping and exchanges it.
- **Direct path:** simultaneous sends punch holes (Lesson 53) so A and B build a direct tunnel.
- **Fallback:** when NAT-B becomes symmetric, the punch fails and traffic relays through C
  (Lesson 54), still end-to-end encrypted.

The whole point: a robust mesh **always succeeds**, preferring direct and degrading to relay, and you
can *see* which path is in use from latency and captures.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `wg show wg0 endpoints` | See whether a peer's endpoint is a direct address or the relay |
| `ping <peer>` (watch RTT) | Direct path = low RTT; relayed = higher (extra hop) |
| `conntrack -L -p udp` (on routers) | Confirm the punched mapping vs relay session |
| `tcpdump -ni <if>` (on C) | Watch traffic appear/disappear on the relay |

---

## Lab

### Step 1 — Build the topology

```bash
# Public node / relay / coordinator
$ sudo ip netns add C
# Two NATed edges: each is lanX behind routerX (masquerade), routers attach to C's segment.
# Reuse Lesson 22's lan/router/internet pattern twice, with C as the shared 'internet'.
$ sudo ip netns add lanA; sudo ip netns add rA
$ sudo ip netns add lanB; sudo ip netns add rB
# wire lanA--rA--C and lanB--rB--C; enable ip_forward + masquerade on rA and rB.
```

### Step 2 — WireGuard keys and interfaces on A, B, C

```bash
$ for n in A B C; do wg genkey | tee $n.key | wg pubkey > $n.pub; done
# Give each a wg0 with inner addrs 10.8.0.1 (A), .2 (B), .3 (C); peers added below.
```

### Step 3 — Phase 1: direct via hole punching

```bash
# C tells A and B each other's external mapping (router's masqueraded IP:port).
# Both edges set the peer endpoint to that mapping and send simultaneously:
$ sudo ip netns exec lanA ping -c1 10.8.0.2 &
$ sudo ip netns exec lanB ping -c1 10.8.0.1 &
# After the punch, the tunnel is direct:
$ sudo ip netns exec lanA wg show wg0 endpoints     # shows B's router mapping (direct)
$ sudo ip netns exec lanA ping -c3 10.8.0.2         # low RTT, traffic NOT seen on C
```

### Step 4 — Phase 2: force symmetric NAT and watch the fallback

```bash
# Make rB symmetric (fully randomized source ports per destination):
$ sudo ip netns exec rB nft add rule ip nat postrouting oifname "rB-C" masquerade random
# Re-establish: punch now fails; the mesh routes A↔B through C as a relay.
$ sudo ip netns exec C tcpdump -ni <C-if> udp &     # now you SEE the relayed traffic on C
$ sudo ip netns exec lanA ping -c3 10.8.0.2         # higher RTT (extra hop via C)
$ sudo ip netns exec lanA wg show wg0 endpoints     # endpoint is now C (the relay)
```

### Step 5 — Clean up

```bash
$ sudo ip netns delete C lanA rA lanB rB
```

---

## Further Reading

| Topic | Link |
|---|---|
| Whole phase recap | Lessons 48–56 |
| NAT traversal | [Wikipedia — NAT traversal](https://en.wikipedia.org/wiki/NAT_traversal) |
| WireGuard | [wireguard.com](https://www.wireguard.com/) |
| Zero-trust networking | [Wikipedia — Zero trust security model](https://en.wikipedia.org/wiki/Zero_trust_security_model) |

---

## Checkpoint

**Q1. Trace a packet from node A to node B in the DIRECT case. Name every transformation it undergoes.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) The application packet leaves A's stack addressed to B's <em>inner</em> overlay address
(<code>10.8.0.2</code>). (2) WireGuard on A consults crypto-key routing, sees <code>10.8.0.2</code>
belongs to peer B, and <strong>encrypts</strong> it (ChaCha20-Poly1305) into a UDP packet aimed at
B's discovered external mapping. (3) That UDP packet leaves lanA and hits router rA, which
<strong>SNATs/masquerades</strong> the source to rA's public IP:port (the mapping the hole punch
opened). (4) It crosses the "internet" segment directly to rB's public mapping — <em>not</em> via C.
(5) rB <strong>un-NATs</strong> (reverses the translation via conntrack) and delivers it to lanB.
(6) WireGuard on B verifies the AEAD tag, <strong>decrypts</strong>, checks the inner source is in
A's AllowedIPs, and hands the original packet to B's stack. Low RTT, and C never sees the traffic.
</details>

---

**Q2. In the RELAYED case, what changes about the path, and why is confidentiality still preserved?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The path gains a hop: instead of rA→rB directly, A's WireGuard packets go A→C and C forwards them
C→B (and vice versa), because the symmetric NAT made the direct punch impossible. The peer endpoint
in <code>wg show</code> now points at C, and RTT rises due to the extra leg; you can also <em>see</em>
the traffic on C with tcpdump (whereas in the direct case you couldn't). Confidentiality is still
preserved because C only forwards the <strong>WireGuard-encrypted</strong> packets — it never holds
A's or B's session keys, so it sees ciphertext and the relay-session addresses but cannot read or
tamper with the contents (any change would fail the AEAD tag at B). The relay changes the
<em>transport path</em>, not the <em>end-to-end encryption</em>.
</details>

---

**Q3. Summarize the decision logic a robust mesh uses to choose between direct and relayed paths.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A robust mesh <strong>prefers a direct peer-to-peer path but guarantees connectivity via relay</strong>.
On connect it typically starts <em>relayed</em> for instant connectivity while simultaneously running
NAT traversal in the background: each peer STUNs to learn its external mapping, exchanges candidates
through the coordination server, and attempts UDP hole punching (with port-prediction tricks against
symmetric NAT). If a direct path is confirmed, it migrates traffic onto it (RTT drops). If traversal
is impossible — symmetric NAT, UDP blocked — it stays on the relay, which works because both peers can
always reach it outbound. The result is "always connected, as fast as the network allows," and which
mode is active is observable from the peer endpoint and the latency.
</details>

---

## Homework

Add a fourth node D that is a **subnet router**: it advertises a route to a non-mesh subnet behind it
(e.g. a namespace with no WireGuard at all), so the other mesh nodes can reach that subnet through D.
Configure the AllowedIPs and routes to make it work, then explain how a subnet router extends the
mesh to devices that can't run WireGuard themselves — and what the security trade-off is.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
To make D a subnet router you (1) put the non-mesh subnet (say <code>192.168.50.0/24</code>) into
D's peer entry on the <em>other</em> nodes' <code>AllowedIPs</code>, so crypto-key routing sends
traffic for that subnet into D's tunnel; (2) add a route on the other nodes pointing that subnet at
<code>wg0</code>; and (3) enable <code>ip_forward</code> on D and (usually) masquerade or route so D
relays between its WireGuard interface and the local subnet. Now a mesh node sending to
<code>192.168.50.x</code> has the packet encrypted to D, decrypted by D, and forwarded onto the plain
subnet — and replies return the same way. This lets the mesh reach <strong>devices that can't run
WireGuard</strong> (printers, appliances, legacy servers, whole on-prem networks) by using D as a
gateway. The security trade-off: D becomes a <strong>trusted bridge</strong> into the encrypted mesh
— traffic to/from that subnet is only protected up to D, not end-to-end, and a compromise of D (or
of any device on the exposed subnet) widens the mesh's attack surface. So subnet routes should be
tightly scoped with ACLs (zero trust, Lesson 55) rather than exposing entire networks blindly.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 58 — Large-Scale BGP →](lesson-58-bgp-scale){: .btn .btn-primary }
