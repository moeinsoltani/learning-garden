---
title: "Lesson 56 — TLS-based VPNs (OpenVPN)"
nav_order: 56
parent: "Phase 16: Tunnels & VPNs"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 56: TLS-based VPNs (OpenVPN) — the contrast

## Concept

You've seen IPsec (kernel, complex) and WireGuard (kernel/userspace, minimal). The third major
family is the **TLS-based VPN**, exemplified by [OpenVPN](https://en.wikipedia.org/wiki/OpenVPN):
it builds the tunnel on top of the same TLS handshake that secures HTTPS, running in userspace over
a TUN/TAP device. It's older and slower than WireGuard, but it taught the industry a lot and is
still everywhere — partly because it can disguise itself as web traffic.

```
   OpenVPN datapath
   ┌─────────────┐
   │ app         │
   ├─────────────┤
   │ TUN/TAP dev │  ← kernel hands packets to userspace
   ├─────────────┤
   │ openvpn proc│  ← wraps them in a TLS session (cert-authenticated)
   └─────────────┘  ← sends over UDP (good) or TCP (fallback)
```

---

## How it works

**TLS for authentication and keying.** OpenVPN uses a full TLS handshake (Phase 20 / Security
track) with **X.509 certificates** to authenticate the peers and negotiate session keys, then
encrypts the tunneled packets with the derived keys. So it inherits TLS's mature, flexible
ecosystem: certificate authorities, revocation, cipher negotiation, even username/password +
cert combos.

**UDP mode vs TCP mode.** OpenVPN can run over UDP (default, preferred) or TCP. TCP mode exists to
traverse networks that only allow TCP/443 — but it introduces the **TCP-over-TCP meltdown**.

{: .note }
> **The TCP-over-TCP problem**
> If you tunnel TCP *inside* a TCP-mode VPN, you now have two reliability layers stacked. When the
> underlying network drops a packet, *both* the outer tunnel TCP and the inner application TCP try
> to retransmit and back off — their timers fight each other, retransmissions pile on
> retransmissions, and throughput collapses under loss. This is why VPNs strongly prefer UDP
> transport: the tunnel should be an unreliable pipe and let the inner TCP handle reliability once.
> WireGuard and IPsec are UDP for exactly this reason; OpenVPN's TCP mode is a last resort.

**Where TLS VPNs still win.** Despite being slower than WireGuard, OpenVPN (and commercial
TLS-VPN/SSL-VPN products) remain useful because: (1) running over **TCP/443 with TLS** makes the
traffic look like ordinary HTTPS, slipping through restrictive firewalls and DPI that block UDP and
WireGuard; (2) **certificate-based PKI** integrates with existing enterprise identity; and (3) a
huge, mature ecosystem of clients and management tooling.

**Userspace cost.** Like userspace WireGuard (Lesson 52), OpenVPN's datapath crosses the
kernel/userspace boundary per packet through the TUN device, so it's CPU-bound and slower than an
in-kernel tunnel. (Newer OpenVPN can use a kernel data-channel offload, DCO, to recover some of this.)

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `openvpn --config <file>` | Start an OpenVPN client/server from a config |
| `openvpn --genkey ...` / easy-rsa | Generate keys / a cert PKI for OpenVPN |
| `ip -d link show tun0` | Confirm the TUN device OpenVPN created |
| `openssl s_client -connect host:443` | Inspect the TLS handshake the tunnel rides on |

---

## Lab

Stand up a minimal OpenVPN server and client between two namespaces (point-to-point static-key mode
keeps the lab short; production uses certs).

### Step 1 — Two namespaces sharing an underlay (as in Lesson 48)

```bash
$ sudo ip netns add srv; sudo ip netns add cli
# veth srv<->cli, srv=10.0.0.1/24, cli=10.0.0.2/24, both up (see Lesson 48)
```

### Step 2 — A shared static key (lab only)

```bash
$ openvpn --genkey secret static.key       # one shared key, copied to both sides
```

### Step 3 — Server side, then client side

```bash
# Server (in srv): UDP, tun, give itself 10.9.0.1 talking to 10.9.0.2
$ sudo ip netns exec srv openvpn --dev tun0 --ifconfig 10.9.0.1 10.9.0.2 \
    --secret static.key --proto udp --lport 1194 --rport 1194 --remote 10.0.0.2 &
# Client (in cli): mirror it
$ sudo ip netns exec cli openvpn --dev tun0 --ifconfig 10.9.0.2 10.9.0.1 \
    --secret static.key --proto udp --lport 1194 --rport 1194 --remote 10.0.0.1 &
```

### Step 4 — Verify the tunnel

```bash
$ sudo ip netns exec cli ping -c2 10.9.0.1      # over the OpenVPN tunnel
$ sudo ip netns exec srv tcpdump -ni srv-cli udp port 1194   # encrypted UDP, payload opaque
```

### Step 5 — Clean up

```bash
$ sudo kill %1 %2 2>/dev/null; sudo ip netns delete srv cli
```

---

## Further Reading

| Topic | Link |
|---|---|
| OpenVPN | [Wikipedia — OpenVPN](https://en.wikipedia.org/wiki/OpenVPN) |
| TLS | [Wikipedia — Transport Layer Security](https://en.wikipedia.org/wiki/Transport_Layer_Security) |
| TCP-over-TCP meltdown | [sites.inka.de — Why TCP over TCP is a bad idea](http://sites.inka.de/bigred/devel/tcp-tcp.html) |
| TUN/TAP | [Lesson 14 — TAP interfaces](lesson-14-tap) |

---

## Checkpoint

**Q1. What does OpenVPN use TLS for, and how does that differ from WireGuard's approach to keys and authentication?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
OpenVPN uses a full <strong>TLS handshake with X.509 certificates</strong> to authenticate the peers
and negotiate the session keys, then encrypts tunneled packets with those keys — the same machinery
that secures HTTPS, including a CA-based PKI and cipher negotiation. WireGuard, by contrast, has
<em>no</em> TLS and no certificates: peers are identified directly by their static public keys
(configured out-of-band), and the handshake is the fixed Noise pattern with hard-coded primitives
and nothing to negotiate. So OpenVPN inherits TLS's flexible, mature-but-heavier ecosystem
(certificates, revocation, negotiation), while WireGuard trades all that flexibility for a tiny,
opinionated, fast design.
</details>

---

**Q2. What is the TCP-over-TCP meltdown, and why does it make VPNs prefer UDP transport?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
If a VPN's transport is TCP and you tunnel TCP traffic through it, you stack <strong>two reliability
layers</strong>. On packet loss, both the outer tunnel TCP and the inner application TCP independently
detect the loss, retransmit, and back off — and their retransmission timers interfere: the outer
layer re-sends data the inner layer is also re-sending, queues build up, and under any real loss the
throughput collapses (the "meltdown"). The fix is to make the tunnel an <strong>unreliable UDP
pipe</strong> so only the inner TCP handles reliability, exactly once. That's why WireGuard and IPsec
use UDP and OpenVPN defaults to UDP — TCP mode is reserved for getting through networks that block
everything except TCP/443.
</details>

---

**Q3. Given WireGuard is faster and simpler, name a real situation where a TLS-based VPN like OpenVPN is still the better choice.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
When you need to traverse a <strong>hostile/restrictive network that blocks UDP and recognizable
WireGuard traffic</strong> — e.g. a corporate guest network, hotel WiFi, or a censored network that
only permits TCP/443. OpenVPN (or an SSL-VPN) running over <strong>TLS on TCP 443</strong> looks
essentially like ordinary HTTPS, so it slips through deep-packet-inspection and port filtering that
would drop WireGuard's UDP outright. (Other valid answers: you need to plug into an existing
<strong>certificate/PKI-based enterprise identity</strong> system, or you require the mature client
and management tooling ecosystem.) The trade-off is lower performance — accepted in exchange for
connectivity where WireGuard simply can't get out.
</details>

---

## Homework

Switch your OpenVPN lab from `--proto udp` to `--proto tcp`, then use `tc netem` (Lesson 30) to add
2% packet loss on the underlay and compare throughput (`iperf3`, Lesson 33) of a TCP download
through the UDP tunnel vs the TCP tunnel. Explain the result in terms of the TCP-over-TCP meltdown.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Under added loss, the <strong>UDP-tunnel</strong> case keeps reasonable throughput, while the
<strong>TCP-tunnel</strong> case degrades far more — often dramatically — for the same loss rate.
Reason: with the UDP tunnel, lost packets are handled <em>once</em> by the inner application's TCP,
which backs off and retransmits normally as it's designed to. With the TCP tunnel, every lost packet
triggers retransmission and congestion back-off in <em>both</em> the outer tunnel TCP and the inner
application TCP simultaneously; the outer layer faithfully redelivers bytes the inner layer has
already given up on and re-queued, so retransmissions compound, the outer send buffer bloats, RTT and
timers balloon, and effective throughput collapses. This is the TCP-over-TCP meltdown in action and
the concrete reason VPN transports should be unreliable (UDP), leaving reliability to the single
inner TCP.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 57 — Capstone: An Encrypted Mesh Across NAT →](lesson-57-vpn-capstone){: .btn .btn-primary }
