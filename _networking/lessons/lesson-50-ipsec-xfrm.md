---
title: "Lesson 50 — IPsec and the xfrm Framework"
nav_order: 50
parent: "Phase 16: Tunnels & VPNs"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 50: IPsec and the kernel xfrm framework

## Concept

[IPsec](https://en.wikipedia.org/wiki/IPsec) is the "enterprise standard" VPN: a suite built
into the IP layer itself that encrypts and authenticates packets. It's powerful and ubiquitous
(site-to-site VPNs, most corporate VPNs), but also famously complex — which is exactly why
WireGuard (next lesson) exists as a reaction to it. Learning IPsec first makes clear *what
WireGuard deliberately threw away*.

IPsec has two halves:

```
   ┌─────────────────────────────┐     ┌──────────────────────────────┐
   │ IKE  (userspace daemon)      │     │ ESP/AH  (kernel datapath)     │
   │ "negotiate keys & policy"    │ ──► │ "encrypt/authenticate packets"│
   │ strongSwan / libreswan       │     │ the xfrm subsystem            │
   └─────────────────────────────┘     └──────────────────────────────┘
        control plane                          data plane
```

A daemon negotiates *what* keys and parameters to use (IKE); the kernel then does the *per-packet*
crypto using those parameters (ESP).

---

## How it works

**ESP vs AH.** [ESP](https://en.wikipedia.org/wiki/IPsec) (Encapsulating Security Payload,
protocol 50) provides encryption + integrity — it's what everyone uses. AH (Authentication
Header) provides integrity only and is largely obsolete.

**Transport vs tunnel mode** — the distinction students must nail:

| Mode | What's protected | Outer IP header | Use |
|---|---|---|---|
| **Transport** | The payload only; original IP header kept | original | Host-to-host on the same network |
| **Tunnel** | The *entire* original packet, wrapped in a new IP header | new | Site-to-site gateways, "real" VPNs |

Transport mode is lean (no extra IP header) but exposes the original addresses and only works
end-host to end-host. **Tunnel mode** encapsulates the whole packet — so gateways can protect
traffic for *whole subnets* behind them, and the inner addresses are hidden. Almost every VPN you
think of as "IPsec" is tunnel mode.

**Security Associations (SAs) and the SPD.** An **SA** is a one-way agreement — keys, cipher, peer
— for a flow; you need two (one each direction). The **Security Policy Database (SPD)** says
*which* traffic must be protected by *which* SA ("traffic from 10.1.0.0/24 to 10.2.0.0/24 → use
this ESP tunnel SA"). In Linux these live in the **`xfrm`** subsystem, managed with `ip xfrm`.

**IKE** ([Internet Key Exchange](https://en.wikipedia.org/wiki/Internet_Key_Exchange), v2 today)
is the userspace negotiation: it authenticates the peers (PSK or certificates), runs Diffie-Hellman
for forward secrecy, and installs the resulting SAs into the kernel. [strongSwan](https://www.strongswan.org/)
is the common daemon.

{: .note }
> **Why IPsec feels heavy**
> Look at the moving parts: two protocols (IKE + ESP), two databases (SAD + SPD), transport vs
> tunnel modes, dozens of negotiable cipher/DH/lifetime parameters that *both ends must agree on*,
> NAT-traversal kludges (ESP-in-UDP on port 4500), and split control/data planes. A single
> mismatch ("phase 2 mismatch") and it silently fails. WireGuard's whole pitch is replacing all of
> this with ~4 lines of config and one cipher suite (Lesson 51).

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip xfrm state` | Show Security Associations (the SAD) |
| `ip xfrm policy` | Show Security Policies (the SPD) |
| `ip xfrm state add ...` | Manually install an SA (for the lab) |
| `swanctl --list-sas` | List SAs negotiated by strongSwan |
| `tcpdump -ni <if> esp` | Capture ESP-protected packets |

---

## Lab

We'll set up a **manual** ESP tunnel between two namespaces with `ip xfrm` (no IKE) so the SA/SPD
machinery is visible. Manual keys are for learning only — real deployments use IKE.

### Step 1 — Two namespaces (reuse the Lesson 48 underlay)

```bash
$ sudo ip netns add a; sudo ip netns add b
$ sudo ip link add a-b type veth peer name b-a
$ sudo ip link set a-b netns a; sudo ip link set b-a netns b
$ sudo ip netns exec a ip addr add 10.0.0.1/24 dev a-b; sudo ip netns exec a ip link set a-b up
$ sudo ip netns exec b ip addr add 10.0.0.2/24 dev b-a; sudo ip netns exec b ip link set b-a up
```

### Step 2 — Install matching SAs and policies (transport mode)

```bash
# Keys (hex) — same on both ends. KEY_E = encryption, KEY_A = auth.
$ E=0x000102030405060708090a0b0c0d0e0f
$ A=0x00112233445566778899aabbccddeeff00112233

# On a: outbound SA a→b and inbound SA b→a
$ sudo ip netns exec a ip xfrm state add src 10.0.0.1 dst 10.0.0.2 proto esp spi 0x1 \
    mode transport enc 'cbc(aes)' $E auth 'hmac(sha256)' $A
$ sudo ip netns exec a ip xfrm state add src 10.0.0.2 dst 10.0.0.1 proto esp spi 0x2 \
    mode transport enc 'cbc(aes)' $E auth 'hmac(sha256)' $A
$ sudo ip netns exec a ip xfrm policy add src 10.0.0.1 dst 10.0.0.2 dir out \
    tmpl proto esp mode transport
$ sudo ip netns exec a ip xfrm policy add src 10.0.0.2 dst 10.0.0.1 dir in \
    tmpl proto esp mode transport
# (mirror the same four commands in namespace b)
```

### Step 3 — Verify it's protected

```bash
$ sudo ip netns exec b tcpdump -ni b-a &
$ sudo ip netns exec a ping -c 2 10.0.0.2
# tcpdump shows ESP, not ICMP — the ping is encrypted:
# IP 10.0.0.1 > 10.0.0.2: ESP(spi=0x00000001,seq=0x1), length 136
$ sudo ip netns exec a ip xfrm state    # shows the two SAs with byte counters climbing
```

### Step 4 — Clean up

```bash
$ sudo ip netns delete a b
```

---

## Further Reading

| Topic | Link |
|---|---|
| IPsec | [Wikipedia — IPsec](https://en.wikipedia.org/wiki/IPsec) |
| IKE | [Wikipedia — Internet Key Exchange](https://en.wikipedia.org/wiki/Internet_Key_Exchange) |
| strongSwan | [strongswan.org](https://www.strongswan.org/) |
| `ip-xfrm` | [man7.org — ip-xfrm(8)](https://man7.org/linux/man-pages/man8/ip-xfrm.8.html) |

---

## Checkpoint

**Q1. What is the difference between IPsec transport mode and tunnel mode, and when do you use each?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Transport mode</strong> protects only the payload of the original packet and keeps the original IP header, so it works host-to-host between the two actual endpoints and adds no extra IP header. <strong>Tunnel mode</strong> encrypts and authenticates the <em>entire</em> original packet (header included) and wraps it in a brand-new outer IP header between gateways. You use transport mode for end-to-end protection between two specific hosts; you use tunnel mode for site-to-site VPNs where gateways protect traffic on behalf of whole subnets behind them — the original/inner addresses get hidden inside the new outer header. Nearly every "IPsec VPN" is tunnel mode.
</details>

---

**Q2. What is a Security Association, and why do you need two for a single bidirectional connection?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A Security Association (SA) is a one-way agreement bundling everything needed to protect a flow in one direction: the peer addresses, the protocol (ESP), the SPI that identifies it, the cipher and keys, and the lifetime. Because an SA is <em>unidirectional</em>, a normal two-way conversation needs <strong>two SAs</strong> — one for A→B and one for B→A — each with its own SPI and keys. The kernel stores them in the SAD (visible via <code>ip xfrm state</code>), while the SPD (<code>ip xfrm policy</code>) decides which traffic must use which SA.
</details>

---

**Q3. Name two concrete reasons IPsec is considered more complex to operate than WireGuard.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Any two of: (1) <strong>Split control/data plane</strong> — a separate userspace IKE daemon (strongSwan/libreswan) negotiates and installs kernel SAs, so there are two systems to configure and debug. (2) <strong>Large negotiated parameter space</strong> — both ends must agree on cipher, integrity algorithm, DH group, modes, and lifetimes across IKE "phase 1/phase 2"; a single mismatch fails silently. (3) <strong>Two databases and two modes</strong> (SAD + SPD, transport vs tunnel) to reason about. (4) <strong>NAT traversal bolt-on</strong> — ESP isn't TCP/UDP, so it needs ESP-in-UDP encapsulation (port 4500) to cross NAT. WireGuard collapses all of this into a single in-kernel module, one modern cipher suite, no negotiation, and UDP-by-default — which is the whole point of the next lesson.
</details>

---

## Homework

Install `strongSwan` and configure a real IKEv2 site-to-site tunnel (tunnel mode, PSK auth)
between two namespaces, replacing the manual `ip xfrm` keys from the lab. Use `swanctl --list-sas`
to watch the SAs get negotiated. Then explain what IKE did for you that you had to do by hand in
the manual lab, and why that matters for forward secrecy.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With strongSwan/IKEv2 you configure each end's identity, the PSK, and the traffic selectors
(which subnets to protect); starting the connection makes IKE authenticate both peers, run a
Diffie-Hellman exchange, and <em>automatically</em> install the matching inbound/outbound ESP SAs
into the kernel — the same SAs you typed by hand with <code>ip xfrm state add</code>. What IKE adds
beyond the manual lab: (1) <strong>peer authentication</strong> (PSK or certificate) so you know
who you're keying with; (2) <strong>automatic, fresh key generation via DH</strong> rather than
static hardcoded keys; and (3) <strong>periodic re-keying</strong> as SAs reach their lifetime.
That last two points give <strong>forward secrecy</strong>: because keys are ephemeral DH-derived
values that rotate and are never written down, a future compromise of the PSK/cert can't decrypt
previously captured traffic — impossible with the static manual keys, which would decrypt
everything ever sent if leaked.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 51 — WireGuard Fundamentals →](lesson-51-wireguard-fundamentals){: .btn .btn-primary }
