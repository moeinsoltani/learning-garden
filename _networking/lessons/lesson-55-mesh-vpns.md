---
title: "Lesson 55 — Mesh VPNs and Coordination Planes"
nav_order: 55
parent: "Phase 16: Tunnels & VPNs"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 55: Mesh VPNs and Coordination/Control Planes

## Concept

A single WireGuard tunnel is point-to-point. Real deployments have dozens or thousands of nodes
that should all reach each other. Configuring that by hand is impossible — and this is where the
**control plane / data plane split** becomes the central idea.

```
   CONTROL PLANE  (coordination server)      DATA PLANE  (the encrypted mesh)
   ┌──────────────────────────────┐
   │ • who are the nodes?          │          n1 ◄══════► n2
   │ • their public keys           │           ╲          ╱
   │ • their endpoints (from STUN) │            ╲        ╱     direct WireGuard
   │ • who may talk to whom (ACL)  │             ╲      ╱      tunnels, peer-to-peer
   │ • DNS names                   │              ╲    ╱       (encrypted data never
   └──────────────────────────────┘               n3          touches the control plane)
        distributes config                    peers talk directly
```

The coordination server never sees your data — it only distributes the *information peers need to
build tunnels themselves*.

---

## How it works

**The N² problem.** A full mesh of N nodes has N(N-1)/2 tunnels; manually maintaining every node's
config as nodes join/leave/roam is hopeless. The control plane automates it: each node reports its
public key and STUN-discovered endpoint, and the server tells every other node what it needs to
peer with it.

**What the coordination server does (and doesn't).** It is a *control plane*: it holds the node
registry (public keys, endpoints, names), the access policy, and helps with signaling for NAT
traversal (Lesson 53). It does **not** carry or decrypt user traffic — peers build direct WireGuard
tunnels (or fall back to relays, Lesson 54) and the actual data is end-to-end encrypted between
them. Compromising the coordinator could let an attacker re-map identities, but not silently read
existing traffic, because it never holds session keys.

**Access control (ACLs) and zero trust.** Being *on* the mesh doesn't mean you can reach
*everything*. Policy rules ("group:dev may reach tag:db on 5432") are distributed from the control
plane and enforced at each node. This is the **zero-trust** model: identity-and-policy decide every
connection, not network location. (You'll meet this again in the Security & Identity track, Phase 6.)

**Identity.** Nodes and users are tied to real identities — typically via SSO/OIDC — so the mesh
knows *who* a node belongs to, and keys can be rotated or revoked when someone leaves.

**Convenience layers built on top:**
- **Split DNS / magic naming:** the control plane assigns names so you reach `db` instead of
  `100.x.y.z`, resolving only mesh names through the overlay.
- **Subnet routers:** a node advertises a route to a whole subnet behind it, so the mesh reaches
  non-mesh devices (printers, legacy servers).
- **Exit nodes:** a node advertises `0.0.0.0/0` (Lesson 51 homework) so others route all internet
  traffic through it.

{: .note }
> **Open-source meshes to study**
> The pattern above is implemented by several open tools you can run yourself: **Nebula**
> (Slack's, certificate-based identity), **innernet** (CIDR-based, WireGuard), and **Headscale**
> (an open-source coordination server speaking the Tailscale protocol). Reading one end-to-end is
> the fastest way to internalize the control/data-plane split.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `wg show all` | Inspect every WireGuard interface/peer the data plane built |
| `wg set wg0 peer <K> endpoint <ip:port>` | What a control plane does for you per peer |
| `ip route show table <n>` | Subnet-router / exit-node routes the mesh installs |
| `resolvectl status` | Split-DNS configuration for the overlay |

---

## Lab

Build a 3-node full mesh *by hand* to feel the N² problem the control plane removes, then reason
about what would automate it.

### Step 1 — Three namespaces with keys

```bash
$ for n in n1 n2 n3; do sudo ip netns add $n; wg genkey | tee $n.key | wg pubkey > $n.pub; done
# (give each a veth to a shared 'lan' bridge so they share an underlay — see Lesson 11)
```

### Step 2 — On each node, add the OTHER two as peers

```bash
# n1 must peer with n2 AND n3; n2 with n1 AND n3; n3 with n1 AND n2.
$ sudo ip netns exec n1 ip link add wg0 type wireguard
$ sudo ip netns exec n1 wg set wg0 private-key n1.key listen-port 51820 \
    peer "$(cat n2.pub)" allowed-ips 10.8.0.2/32 endpoint <n2-underlay>:51820 \
    peer "$(cat n3.pub)" allowed-ips 10.8.0.3/32 endpoint <n3-underlay>:51820
$ sudo ip netns exec n1 ip addr add 10.8.0.1/24 dev wg0
$ sudo ip netns exec n1 ip link set wg0 up
# ...repeat the analogous two-peer config for n2 and n3...
```

### Step 3 — Verify the mesh

```bash
$ sudo ip netns exec n1 ping -c1 10.8.0.2 && sudo ip netns exec n1 ping -c1 10.8.0.3
$ sudo ip netns exec n1 wg show
```

### Step 4 — Count the work, then imagine automation

With 3 nodes you wrote 3×2 = 6 peer entries. At 10 nodes that's 90; at 100, ~9900. A coordination
server replaces all of it: each node registers its pubkey+endpoint once, and the server pushes the
right peer list to everyone, updating it as nodes join, leave, or roam.

### Step 5 — Clean up

```bash
$ sudo ip netns delete n1 n2 n3
```

---

## Further Reading

| Topic | Link |
|---|---|
| Mesh networking | [Wikipedia — Mesh networking](https://en.wikipedia.org/wiki/Mesh_networking) |
| Control plane vs data plane | [Wikipedia — Control plane](https://en.wikipedia.org/wiki/Control_plane) |
| Zero-trust model | [Wikipedia — Zero trust security model](https://en.wikipedia.org/wiki/Zero_trust_security_model) |
| Nebula | [github.com/slackhq/nebula](https://github.com/slackhq/nebula) |
| Headscale | [github.com/juanfont/headscale](https://github.com/juanfont/headscale) |

---

## Checkpoint

**Q1. In a mesh VPN, what does the control plane handle versus the data plane?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The <strong>control plane</strong> (coordination server) handles <em>metadata and policy</em>: the
node registry (public keys, names), each node's current endpoint (often STUN-discovered), the access
policy/ACLs, signaling to help peers do NAT traversal, and conveniences like DNS naming. The
<strong>data plane</strong> is the actual encrypted traffic, which flows <em>directly between
peers</em> over WireGuard tunnels (or via a relay when direct fails). Critically, user data never
passes through (or is readable by) the control plane — the server only distributes the information
peers need to build their own tunnels, while the bytes themselves stay end-to-end encrypted between
the nodes.
</details>

---

**Q2. Why is a full manual mesh impractical at scale, and how does a coordination server fix it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A full mesh of N nodes needs N(N-1)/2 tunnels, and every node's config must list every other node's
public key and current endpoint. With nodes constantly joining, leaving, and roaming (changing IPs),
keeping all those configs correct by hand scales quadratically and is unmanageable beyond a handful
of nodes (10 nodes ≈ 90 peer entries; 100 ≈ ~9900). A coordination server fixes it by having each
node <strong>register once</strong> (its public key and discovered endpoint) and then
<strong>pushing the right peer list to everyone automatically</strong>, updating it in real time as
membership and endpoints change. Each node only talks to the server about config; it builds its data
tunnels itself.
</details>

---

**Q3. The mesh enforces "being connected ≠ being allowed." What model is this, and why does it matter?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
This is the <strong>zero-trust</strong> model: trust is based on verified <em>identity and policy</em>
for every connection, not on network location. Even after a node joins the encrypted mesh, ACLs
distributed by the control plane and enforced at each node decide which nodes/ports it may actually
reach (e.g. "dev group may reach the database on 5432, nothing else"). It matters because the old
"perimeter" model — inside the network = trusted — fails badly: one compromised node could roam the
whole flat network. Zero trust contains blast radius by making each connection prove it's authorized,
which is exactly the philosophy the Security & Identity track's Phase 6 develops further.
</details>

---

## Homework

Stand up an open-source coordination server (e.g. **Headscale**) on one node and join two other
nodes to it, letting it configure the WireGuard peering automatically. Compare the per-node effort
to the manual mesh you built in the lab, then write down which of the control-plane responsibilities
from this lesson you saw the server perform (registry, endpoint distribution, ACLs, DNS).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With Headscale, each node runs a client that authenticates to the server once and then receives its
entire peer configuration automatically — you never hand-write peer public keys, endpoints, or
AllowedIPs. Compared to the manual lab (where every node needed an explicit entry for every other
node), the per-node effort becomes constant instead of growing with mesh size: join once, and the
server keeps your peer list current as others come and go. The control-plane responsibilities you
can observe it performing: the <strong>node registry</strong> (it tracks each node's public key and
identity), <strong>endpoint distribution</strong> (it learns and shares each node's reachable
address, enabling direct tunnels or relay fallback), <strong>ACL policy</strong> (it distributes
rules controlling who may reach whom), and <strong>DNS / naming</strong> (it assigns names so you can
reach nodes by name over the overlay). The data, meanwhile, still flows directly peer-to-peer and
end-to-end encrypted — the server never carries it.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 56 — TLS-based VPNs (OpenVPN) →](lesson-56-tls-vpns){: .btn .btn-primary }
