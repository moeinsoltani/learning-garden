---
title: "Lesson 69 — QUIC & HTTP/3"
nav_order: 69
parent: "Phase 20: Modern Transport & Observability"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 69: QUIC & HTTP/3

## Concept

[QUIC](https://en.wikipedia.org/wiki/QUIC) is a modern transport that replaces the TCP+TLS combo for
the web. Surprisingly, it runs **over UDP** — yet it's *connection-oriented*, reliable, and
encrypted, reimplementing in userspace what TCP does in the kernel, plus fixing TCP's structural
limits. HTTP/3 is simply HTTP carried over QUIC.

```
   HTTP/2 stack          HTTP/3 stack
   ┌─────────┐           ┌─────────┐
   │ HTTP/2  │           │ HTTP/3  │
   │  TLS    │           │  QUIC   │ ← reliability + TLS 1.3 + streams, all together
   │  TCP    │           │  UDP    │ ← thin, unreliable substrate
   └─────────┘           └─────────┘
```

---

## How it works

**Why UDP, not a new protocol next to TCP?** A genuinely new IP protocol (like SCTP) is undeployable
on today's internet — middleboxes, NATs, and firewalls only pass TCP and UDP, and OS TCP stacks live
in the kernel where they're slow to evolve. By building on **UDP**, QUIC traverses existing networks
*and* lives in **userspace**, so it can iterate quickly and ship with the application. UDP is just the
minimal "get a datagram there" layer; QUIC adds everything else on top.

**Built-in TLS 1.3.** Encryption isn't optional or layered separately — QUIC integrates TLS 1.3 into
the transport. The handshake establishes the connection and the keys together, so connection setup is
~1 RTT (or 0-RTT on resumption), beating TCP+TLS's separate handshakes.

**Streams without head-of-line blocking.** This is the headline fix. In HTTP/2-over-TCP, many
requests are multiplexed on one TCP connection — but TCP is a single ordered byte stream, so one lost
packet **stalls *all* multiplexed streams** until it's retransmitted (head-of-line blocking). QUIC
has **independent streams** with their own ordering: a lost packet only blocks *its* stream; the
others keep flowing.

**Connection migration.** A QUIC connection is identified by a **connection ID**, not the 4-tuple. So
if your IP changes (WiFi→cellular), the connection survives — like MPTCP/WireGuard roaming, but native
to the transport.

{: .note }
> **Hard to inspect**
> Because almost everything — headers, handshake, stream info — is encrypted, on-path middleboxes (and
> your tcpdump) can see far less than with TCP. Only minimal fields are visible. This is great for
> privacy and ossification-resistance, but it means debugging QUIC relies on endpoint logs
> (`SSLKEYLOGFILE` + Wireshark, or qlog) rather than passive capture.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `curl --http3 https://<host>` | Fetch a URL over HTTP/3/QUIC |
| `tcpdump -ni <if> udp port 443` | See QUIC traffic (UDP/443) |
| `SSLKEYLOGFILE=keys.log curl --http3 ...` | Log keys so Wireshark can decrypt QUIC |
| `ss -uanp` | UDP sockets (QUIC shows as UDP) |

---

## Lab

Observe QUIC on the wire and confirm it's UDP, then contrast its visibility with TLS-over-TCP.

### Step 1 — Fetch over HTTP/3 and capture

```bash
$ sudo tcpdump -ni any udp port 443 -c 10 &
$ curl -sI --http3 https://www.cloudflare.com | head -1
HTTP/3 200
# tcpdump shows UDP/443 datagrams — that's QUIC, not TCP.
```

### Step 2 — Compare to HTTP/2 (TCP)

```bash
$ sudo tcpdump -ni any tcp port 443 -c 10 &
$ curl -sI --http2 https://www.cloudflare.com | head -1
HTTP/2 200
# Now it's TCP/443; you can see the TCP handshake flags, unlike QUIC.
```

### Step 3 — Note what's visible

```bash
# In the QUIC capture, almost everything after the first packet is encrypted —
# you cannot read stream/headers passively. In the TCP+TLS capture you can at least
# see the TCP state machine and the unencrypted parts of the TLS handshake (e.g. SNI).
```

---

## Further Reading

| Topic | Link |
|---|---|
| QUIC | [Wikipedia — QUIC](https://en.wikipedia.org/wiki/QUIC) |
| HTTP/3 | [Wikipedia — HTTP/3](https://en.wikipedia.org/wiki/HTTP/3) |
| Head-of-line blocking | [Wikipedia — Head-of-line blocking](https://en.wikipedia.org/wiki/Head-of-line_blocking) |
| TLS 1.3 | Security & Identity track, Phase 3 |

---

## Checkpoint

**Q1. Why does QUIC run over UDP instead of being a new protocol alongside TCP?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Two reasons, both about deployability. (1) <strong>Middlebox reality</strong>: the internet's NATs,
firewalls, and load balancers only reliably pass TCP and UDP; a brand-new IP protocol (as SCTP
discovered) gets dropped, so it would be undeployable. UDP is universally allowed, so QUIC rides it to
traverse existing networks. (2) <strong>Evolvability</strong>: TCP lives in the kernel, so improving it
means waiting for OS upgrades across the world (ossification). Building on UDP lets QUIC run in
<strong>userspace</strong>, shipped and updated with the application (e.g. the browser), so it can
iterate rapidly. UDP provides just the minimal datagram delivery; QUIC re-implements reliability,
ordering, congestion control, and encryption on top.
</details>

---

**Q2. What is head-of-line blocking in HTTP/2-over-TCP, and how does QUIC eliminate it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
HTTP/2 multiplexes many requests over a single TCP connection. But TCP delivers one <em>ordered byte
stream</em>, so if a single packet is lost, TCP must wait for its retransmission before delivering any
later bytes — even bytes belonging to <em>other</em> multiplexed requests. One lost packet therefore
stalls <em>all</em> the streams: that's transport-level head-of-line blocking. QUIC eliminates it by
implementing <strong>independent streams</strong>, each with its own sequencing/flow control: a lost
packet only holds up the stream it belonged to, while the other streams continue to be delivered. So
loss affecting one response no longer freezes the others.
</details>

---

**Q3. How does QUIC survive a client's IP address changing, and what is it about QUIC that makes this possible?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
QUIC supports <strong>connection migration</strong>: a connection is identified by a
<strong>connection ID</strong> carried in the packets, not by the source/destination IP-and-port
4-tuple the way TCP is. So when the client's address changes (e.g. WiFi to cellular), packets with the
same connection ID are recognized as belonging to the existing connection, which continues without
resetting — the endpoints validate the new path and carry on. This is possible because QUIC deliberately
decoupled connection identity from the network 4-tuple (and because its encryption authenticates the
packets so migration can't be spoofed). It's conceptually like WireGuard's roaming or MPTCP's
path-independence, but native to the transport.
</details>

---

## Homework

Use `SSLKEYLOGFILE` to log QUIC's keys during a `curl --http3` fetch, open the capture in Wireshark,
and decrypt the QUIC streams. Then explain why this endpoint-cooperation step is required to inspect
QUIC at all, and what that implies for network operators who used to rely on passive monitoring of
TCP/TLS.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
By exporting the session keys to <code>SSLKEYLOGFILE</code> and loading them into Wireshark, you can
decrypt and read the QUIC streams (headers, HTTP/3 frames) that are otherwise opaque on the wire. This
endpoint cooperation is <em>required</em> because QUIC encrypts almost everything — not just the payload
but most of the transport metadata (stream IDs, frame types, even much of the handshake) — so a passive
observer with only the captured packets sees essentially ciphertext; the only way in is to obtain the
keys from one of the endpoints. The implication for operators: techniques that depended on
<strong>passively reading TCP/TLS</strong> — inferring performance from TCP state, seeing SNI for
policy/filtering, doing DPI — largely <strong>stop working with QUIC</strong>. Visibility shifts from
the middle of the network to the <strong>endpoints</strong> (server/client logs, qlog), which is by
design: QUIC's encryption resists both surveillance and the protocol ossification that middlebox
meddling caused. Operators must move to endpoint telemetry rather than in-path inspection.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 70 — TLS at the Packet Level →](lesson-70-tls){: .btn .btn-primary }
