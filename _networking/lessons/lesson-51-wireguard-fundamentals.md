---
title: "Lesson 51 — WireGuard Fundamentals"
nav_order: 51
parent: "Phase 16: Tunnels & VPNs"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 51: WireGuard Fundamentals

## Concept

[WireGuard](https://en.wikipedia.org/wiki/WireGuard) is the modern, minimal VPN: a single in-kernel
module, one fixed modern cipher suite, no negotiation, and a configuration so small it fits on a
postcard. Where IPsec (Lesson 50) is a flexible suite of protocols, WireGuard is a single opinionated
protocol. It's the data-plane foundation of most contemporary mesh VPNs.

The whole model is built on one idea: **crypto-key routing**. Each peer is identified by its
**public key**, and each peer is associated with a set of **AllowedIPs**. The public key *is* the
identity, and AllowedIPs answers both routing questions at once:

```
   Outbound: "which peer should this packet go to?"
       → the peer whose AllowedIPs contains the destination address

   Inbound:  "is this decrypted packet allowed from this peer?"
       → yes only if its source address is in that peer's AllowedIPs
```

```
   me (pubkey Ma)                          peer (pubkey Mb)
   AllowedIPs for peer: 10.8.0.2/32        AllowedIPs for me: 10.8.0.1/32
        │  encrypt to Mb, send over UDP          │
        └───────────────  UDP  ─────────────────►│ decrypt, check source ∈ AllowedIPs
```

---

## How it works

**One handshake, Noise-based.** WireGuard's handshake uses the
[Noise protocol framework](https://en.wikipedia.org/wiki/Noise_Protocol_Framework) (the *IK*
pattern) with fixed primitives: **Curve25519** for key exchange, **ChaCha20-Poly1305** for AEAD,
**BLAKE2s** for hashing. There is nothing to negotiate — both ends already agree because the
algorithms are baked in. The handshake establishes ephemeral session keys (forward secrecy) and
is re-run roughly every 2 minutes.

**UDP-only, stateless-feeling.** All traffic is UDP (default nothing — you pick a `ListenPort`).
There's no connection state to set up the way TCP or IKE has; a peer that hears a valid handshake
just responds. Endpoints can **roam** — if a peer's IP changes (laptop moves WiFi→LTE), the next
authenticated packet updates its endpoint automatically. This roaming is why WireGuard underlies
mobile mesh VPNs.

**Crypto-key routing replaces policy.** There's no SPD, no firewall-style match list. The
AllowedIPs of each peer simultaneously define the VPN's internal routing table *and* its inbound
access control. A packet for `10.8.0.2` is encrypted to whichever peer lists `10.8.0.2` in
AllowedIPs; a decrypted packet is dropped unless its source falls within the sending peer's
AllowedIPs. Set AllowedIPs to `0.0.0.0/0` and that peer becomes your default-route exit node.

{: .note }
> **Why this is so much smaller than IPsec**
> No phase 1/phase 2, no cipher negotiation, no SAD/SPD, no transport-vs-tunnel choice. Config is
> just: my private key, my listen port, and for each peer its public key + endpoint + AllowedIPs.
> The kernel module is ~4000 lines vs IPsec's tens of thousands. Less to configure means less to
> misconfigure — the security argument is partly *simplicity*.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `wg genkey` / `wg pubkey` | Generate a private key / derive its public key |
| `ip link add wg0 type wireguard` | Create a WireGuard interface |
| `wg set wg0 private-key <file> listen-port <n>` | Configure the local end |
| `wg set wg0 peer <PUBKEY> allowed-ips <cidr> endpoint <ip:port>` | Add a peer |
| `wg show` | Show peers, handshakes, transfer, endpoints |
| `wg-quick up <conf>` | Bring up a tunnel from a config file |

---

## Lab

Build a WireGuard tunnel between two namespaces and inspect it with `wg show`.

### Step 1 — Underlay + keys

```bash
$ sudo ip netns add a; sudo ip netns add b
$ sudo ip link add a-b type veth peer name b-a
$ sudo ip link set a-b netns a; sudo ip link set b-a netns b
$ sudo ip netns exec a ip addr add 10.0.0.1/24 dev a-b; sudo ip netns exec a ip link set a-b up
$ sudo ip netns exec b ip addr add 10.0.0.2/24 dev b-a; sudo ip netns exec b ip link set b-a up

$ wg genkey | tee a.key | wg pubkey > a.pub
$ wg genkey | tee b.key | wg pubkey > b.pub
```

### Step 2 — Create the WireGuard interfaces

```bash
# Namespace a: inner address 10.8.0.1, peer b's pubkey + endpoint
$ sudo ip netns exec a ip link add wg0 type wireguard
$ sudo ip netns exec a wg set wg0 private-key a.key listen-port 51820 \
    peer "$(cat b.pub)" allowed-ips 10.8.0.2/32 endpoint 10.0.0.2:51820
$ sudo ip netns exec a ip addr add 10.8.0.1/24 dev wg0
$ sudo ip netns exec a ip link set wg0 up

# Namespace b: mirror it
$ sudo ip netns exec b ip link add wg0 type wireguard
$ sudo ip netns exec b wg set wg0 private-key b.key listen-port 51820 \
    peer "$(cat a.pub)" allowed-ips 10.8.0.1/32 endpoint 10.0.0.1:51820
$ sudo ip netns exec b ip addr add 10.8.0.2/24 dev wg0
$ sudo ip netns exec b ip link set wg0 up
```

### Step 3 — Bring up the tunnel and inspect

```bash
$ sudo ip netns exec a ping -c 2 10.8.0.2     # triggers the handshake
$ sudo ip netns exec a wg show
interface: wg0
  public key: <Ma>
  listening port: 51820
peer: <Mb>
  endpoint: 10.0.0.2:51820
  allowed ips: 10.8.0.2/32
  latest handshake: 2 seconds ago
  transfer: 296 B received, 360 B sent
```

The capture of the underlay shows only encrypted UDP on port 51820 — the ICMP is invisible.

### Step 4 — Clean up

```bash
$ sudo ip netns delete a b
```

---

## Further Reading

| Topic | Link |
|---|---|
| WireGuard | [Wikipedia — WireGuard](https://en.wikipedia.org/wiki/WireGuard) |
| Official site / whitepaper | [wireguard.com](https://www.wireguard.com/) |
| Noise protocol framework | [Wikipedia — Noise Protocol Framework](https://en.wikipedia.org/wiki/Noise_Protocol_Framework) |
| `wg` | [man7.org — wg(8)](https://man7.org/linux/man-pages/man8/wg.8.html) |
| `wg-quick` | [man7.org — wg-quick(8)](https://man7.org/linux/man-pages/man8/wg-quick.8.html) |

---

## Checkpoint

**Q1. WireGuard has no routing table of its own and no firewall rules for the tunnel. How does it decide which peer to send a packet to, and whether to accept a decrypted packet?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Through <strong>crypto-key routing</strong>, driven by each peer's <strong>AllowedIPs</strong>. Outbound: the kernel looks at the packet's destination address and encrypts it to the peer whose AllowedIPs range contains that address (and sends it to that peer's endpoint). Inbound: after decrypting a packet from a given peer, WireGuard checks the packet's <em>source</em> address against that same peer's AllowedIPs and drops it unless it falls within the allowed range. So AllowedIPs is simultaneously the VPN's internal routing table (which peer owns which addresses) and its access-control list (a peer may only send traffic from addresses it's allowed to use). The peer itself is identified by its public key.
</details>

---

**Q2. Why does WireGuard have nothing to "negotiate" during its handshake, and what's the upside?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
WireGuard hard-codes its cryptographic primitives — Curve25519 for key exchange, ChaCha20-Poly1305 for AEAD, BLAKE2s for hashing — so both ends already agree by construction; there's no cipher/DH/lifetime negotiation like IPsec's IKE. The upsides: (1) the handshake is a single fixed exchange (the Noise IK pattern), faster and simpler; (2) there's no downgrade attack surface, since an attacker can't negotiate you onto a weaker algorithm; and (3) far less to misconfigure. The trade-off is no per-connection algorithm agility — but WireGuard's answer is "if a primitive is broken, ship a new version" rather than carry negotiation complexity forever.
</details>

---

**Q3. A laptop running WireGuard moves from WiFi to cellular and its public IP changes. Why does the tunnel keep working without reconfiguration?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because of WireGuard's <strong>roaming</strong> behavior. A peer isn't pinned to a fixed source IP; it's identified by its public key. When the laptop sends an <em>authenticated</em> packet from its new IP/port, the receiving peer verifies it cryptographically and then updates that peer's stored <code>endpoint</code> to the new address automatically. Since WireGuard is UDP and stateless-feeling (no TCP-style connection to re-establish), traffic simply continues to the updated endpoint. As long as at least one side knows how to reach the other to (re)start, the handshake resumes and the session keys roll over. This is what makes WireGuard suitable for mobile devices and mesh VPNs where endpoints move constantly.
</details>

---

## Homework

Convert the lab into `wg-quick` config files (`/etc/wireguard/wg0.conf` style with `[Interface]`
and `[Peer]` sections) and bring the tunnel up with `wg-quick up`. Then set one peer's
`AllowedIPs` to `0.0.0.0/0` and explain what changes about that peer's routing — and why this is
how an "exit node" / full-tunnel VPN is built.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A <code>wg-quick</code> config has an <code>[Interface]</code> section (PrivateKey, Address,
ListenPort) and one <code>[Peer]</code> section per peer (PublicKey, Endpoint, AllowedIPs);
<code>wg-quick up wg0</code> creates the interface, applies the keys, assigns the address, and —
crucially — installs <em>routes</em> for each peer's AllowedIPs automatically. When you set a peer's
<code>AllowedIPs = 0.0.0.0/0</code>, that peer now "owns" the entire IPv4 space in crypto-key
routing, so <code>wg-quick</code> routes <em>all</em> your traffic into the tunnel toward that peer.
That peer becomes a full-tunnel <strong>exit node</strong>: every packet you send is encrypted to it
and it forwards/NATs the traffic onward to the internet, so your effective public IP becomes the
exit node's. (<code>wg-quick</code> also adds a fwmark + policy-routing rule — Lesson 18 — so the
encrypted WireGuard packets themselves don't get caught in that default route.) This is exactly how
a commercial "connect and route everything through us" VPN, or a Tailscale-style exit node, is
constructed.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 52 — WireGuard Internals & Userspace Datapaths →](lesson-52-wireguard-internals){: .btn .btn-primary }
