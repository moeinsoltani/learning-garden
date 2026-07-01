---
title: "Lesson 54 — Relays and Fallback Paths"
nav_order: 54
parent: "Phase 16: Tunnels & VPNs"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 54: Relays and Fallback Paths

## Concept

NAT traversal (Lesson 53) sometimes fails — symmetric NATs, restrictive firewalls that block UDP
entirely, captive networks that only allow HTTPS. A serious mesh VPN can never say "sorry, no
connection." The escape hatch is a **relay**: a server on the public internet that *both* peers can
reach with ordinary outbound connections, which forwards their traffic for them.

```
   Direct (preferred):   A ───────────────► B          (low latency, no middleman)

   Relayed (fallback):   A ──► [ Relay ] ◄── B         (always works; both reach it outbound)
                              forwards bytes
```

The key property: a relay guarantees connectivity by relying only on **outbound** reachability,
which essentially every network permits.

---

## How it works

**Why outbound always works.** Even the most locked-down NAT/firewall lets internal hosts make
outbound connections (that's how anyone browses the web). A relay exploits this: both peers open an
outbound session *to the relay*, and the relay shuttles packets between those two sessions. No
inbound port, no hole punching, no NAT cooperation required.

**Surviving hostile networks: relay over HTTPS/443.** Some networks block UDP and all non-web
ports. So relays commonly speak **TLS over TCP port 443**, indistinguishable from normal HTTPS
traffic. If you can load a web page, you can reach the relay. (This is why such relays are a
last-resort fallback for connectivity, not performance.)

**End-to-end encryption is preserved.** Crucially, the relay forwards the peers' *already
WireGuard-encrypted* packets. It never holds the session keys and cannot read the traffic — it sees
only ciphertext and the two peers' relay-session addresses. So using a relay does **not** weaken
confidentiality; it only adds a forwarding hop. A relay must be untrusted-by-design.

```
   A: [ inner data | WG-encrypted ] ──TLS──► Relay ──TLS──► B
                     └ relay can't read this; it only forwards the blob ┘
```

**The cost: latency and bandwidth.** A relayed path is longer (A→relay→B instead of A→B) and the
relay operator pays for the bandwidth. So a mesh treats relaying as a **fallback**, not a default.

{: .note }
> **Upgrade from relayed to direct, mid-session**
> Good meshes start relayed (instant connectivity, zero wait) while *simultaneously* attempting NAT
> traversal in the background. The moment a direct path is confirmed, traffic is migrated off the
> relay onto the direct path with no interruption — the connection "gets faster" a second or two
> after it starts. Latency to peers dropping suddenly is the visible sign of this hand-off.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `socat TCP-LISTEN:443,fork TCP:<peer>:<port>` | Crude TCP relay (forwards a stream) |
| `ss -tnp` | See established outbound relay sessions |
| `ping <peer>` (watch RTT) | Detect the relayed→direct hand-off (RTT drops) |
| `tcpdump -ni <if> port 443` | Confirm traffic riding the HTTPS-port relay |

---

## Lab

Demonstrate the relay principle with `socat`: two peers that can't reach each other directly both
connect *outbound* to a relay, which forwards between them.

### Step 1 — Three namespaces: A, R (relay), B — A and B can each reach R but not each other

```bash
$ sudo ip netns add A; sudo ip netns add R; sudo ip netns add B
# veths: A--R and B--R (note: no A--B link exists at all)
$ sudo ip link add A-R type veth peer name R-A
$ sudo ip link add B-R type veth peer name R-B
$ sudo ip link set A-R netns A; sudo ip link set R-A netns R
$ sudo ip link set B-R netns B; sudo ip link set R-B netns R
$ sudo ip netns exec A ip addr add 10.1.0.1/24 dev A-R; sudo ip netns exec A ip link set A-R up
$ sudo ip netns exec R ip addr add 10.1.0.2/24 dev R-A; sudo ip netns exec R ip link set R-A up
$ sudo ip netns exec R ip addr add 10.2.0.2/24 dev R-B; sudo ip netns exec R ip link set R-B up
$ sudo ip netns exec B ip addr add 10.2.0.1/24 dev B-R; sudo ip netns exec B ip link set B-R up
```

### Step 2 — Confirm A and B cannot reach each other directly

```bash
$ sudo ip netns exec A ping -c1 10.2.0.1     # fails: no route/link to B
```

### Step 3 — Run a relay on R and connect both peers outbound

```bash
# B listens for the relayed payload; R forwards A's connection to B
$ sudo ip netns exec B nc -l -p 9000 &
$ sudo ip netns exec R socat TCP-LISTEN:443,fork TCP:10.2.0.1:9000 &
$ sudo ip netns exec A sh -c 'echo "hello via relay" | nc 10.1.0.2 443'
# B receives "hello via relay" — forwarded by R, though A and B share no link
```

### Step 4 — Clean up

```bash
$ sudo ip netns delete A R B
```

---

## Further Reading

| Topic | Link |
|---|---|
| TURN (the standardized relay) | [Wikipedia — TURN](https://en.wikipedia.org/wiki/Traversal_Using_Relays_around_NAT) |
| NAT traversal | [Wikipedia — NAT traversal](https://en.wikipedia.org/wiki/NAT_traversal) |
| TLS (relay transport) | [Wikipedia — Transport Layer Security](https://en.wikipedia.org/wiki/Transport_Layer_Security) |
| `socat` | [man7.org — socat(1)](https://man7.org/linux/man-pages/man1/socat.1.html) |

---

## Checkpoint

**Q1. Why does a relay reliably connect two peers when direct NAT traversal fails?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because a relay depends only on <strong>outbound</strong> reachability, which virtually every
network permits (it's how normal web browsing works). Both peers open an outbound connection <em>to
the relay</em> — neither needs a public IP, an open inbound port, or any NAT cooperation — and the
relay forwards packets between the two outbound sessions. Direct traversal can fail (symmetric NAT,
UDP blocked, restrictive firewall), but "can this host make an outbound connection to a public
server" is almost always yes, so the relay is a dependable fallback. To survive networks that block
everything but web traffic, relays often run over TLS on port 443 so they look like ordinary HTTPS.
</details>

---

**Q2. Does relaying traffic let the relay operator read it? Explain.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
No. The relay forwards the peers' <strong>already end-to-end-encrypted</strong> packets (e.g.
WireGuard ciphertext). It never possesses the peers' session keys, so it sees only opaque encrypted
blobs plus the two relay-session addresses — it can observe <em>that</em> two peers are communicating
and how much, but not <em>what</em>. This is a deliberate design property: the relay is untrusted by
construction, which is what makes it safe to operate a shared relay fleet that many tenants route
through. Confidentiality and integrity come from the peers' own encryption, independent of the
transport, so adding a relay hop doesn't weaken them.
</details>

---

**Q3. Why do mesh VPNs treat relaying as a fallback and try to "upgrade" to a direct path?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A relayed path is strictly worse on performance: it's longer (A→relay→B instead of A→B), adding
latency, and it consumes the relay operator's bandwidth, which costs money and can bottleneck. So
meshes use the relay only to <em>guarantee</em> connectivity. A good implementation starts relayed —
so the connection works instantly with no waiting — while running NAT traversal in the background;
as soon as a direct peer-to-peer path is confirmed, it migrates the traffic onto it seamlessly, and
you observe the round-trip time drop. You get the best of both: immediate connectivity from the
relay and, shortly after, the low latency of a direct path.
</details>

---

## Homework

Extend the relay lab so that A's payload to B is itself encrypted *before* it reaches the relay
(e.g. run the earlier WireGuard tunnel's traffic, or just `openssl enc` the data) and verify with
`tcpdump` on R that the relay only ever sees ciphertext. Then argue why this property is what lets a
provider run one shared relay fleet for many untrusting customers.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Capturing on R shows only the encrypted blob passing through — the plaintext never appears on the
relay, because A encrypted it end-to-end before sending and only B holds the key to decrypt.
Therefore the relay is a pure forwarder of opaque bytes. This is exactly what makes a <strong>shared,
multi-tenant relay fleet</strong> safe: since the relay can't read or tamper with anyone's traffic
(any modification would break the AEAD tag and be rejected by the recipient), customers don't have
to trust the relay operator with their data — they only trust it to forward packets. The trust
boundary is the peers' own keys, not the relay. That decoupling lets one provider operate global
relays that thousands of independent users route through without exposing anyone's content to the
provider or to each other.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 55 — Mesh VPNs and Coordination Planes →](lesson-55-mesh-vpns){: .btn .btn-primary }
