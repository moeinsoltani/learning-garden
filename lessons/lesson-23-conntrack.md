---
title: "Lesson 23 — Conntrack"
nav_order: 23
parent: "Phase 7: NAT & Conntrack"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 23: Conntrack

## Concept

**Conntrack** (connection tracking) is the kernel subsystem that remembers the *flows* passing through the machine. It is the memory that makes both NAT (previous lesson) and stateful firewalling (next phase) possible. Without it, every packet would be judged in isolation; with it, the kernel knows "this packet belongs to a connection I already approved."

```
   First packet (SYN) ──► conntrack creates an entry, state = NEW
   Reply (SYN-ACK)    ──► conntrack sees the flow confirmed, state = ESTABLISHED
   ...data...         ──► all matched to the same entry, state = ESTABLISHED
   A related flow     ──► (e.g. FTP data channel) state = RELATED
   Garbage / no flow  ──► state = INVALID
```

Conntrack tracks every connection through the box (whether NATed or just forwarded/filtered), storing each flow's addresses, ports, protocol, and **state**.

---

## How it works

For each new flow, conntrack records a tuple in both directions (original and reply) and assigns it a **state**:

| State | Meaning |
|---|---|
| `NEW` | First packet of a flow the kernel hasn't seen before. |
| `ESTABLISHED` | A reply has been seen — the flow is two-way and confirmed. |
| `RELATED` | A *new* connection that is logically part of an existing one (e.g. an FTP data transfer spawned by an FTP control channel, or an ICMP error about an existing flow). |
| `INVALID` | A packet that doesn't match any known flow and isn't a valid new connection — often spoofed, out-of-window, or corrupt. |

This state is what firewall rules key on. The canonical stateful-firewall line — `ct state established,related accept` — says "allow replies and related traffic for connections we already permitted," which is what lets you open *outbound* without manually allowing every *inbound* reply.

Conntrack ties replies to flows by matching the reply's tuple against the stored *reply-direction* tuple. For NAT, it additionally stores the translation so it can reverse it (Lesson 22).

{: .note }
> **The conntrack table has a size limit**
> Every tracked flow consumes a slot, capped by `net.netfilter.nf_conntrack_max`. Under a flood of connections (or a SYN flood), the table can fill, and new connections get dropped with `nf_conntrack: table full` in the logs. High-traffic routers and load balancers tune `nf_conntrack_max` and the per-state timeouts. This is also why connection-tracking has a real cost — the flowtable offload in Lesson 28 exists partly to skip it for established flows.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `conntrack -L` | List all tracked connections |
| `conntrack -E` | Stream conntrack events live (NEW/UPDATE/DESTROY) |
| `conntrack -L -p tcp` | List only TCP connections |
| `conntrack -D <filter>` | Delete matching entries |
| `conntrack -C` | Count current entries |
| `sysctl net.netfilter.nf_conntrack_max` | Show/set the table size limit |
| `sysctl net.netfilter.nf_conntrack_count` | Current number of tracked flows |

---

## Lab

We'll watch a TCP connection's full lifecycle move through conntrack states, from SYN to teardown.

### Step 1 — Two namespaces through a tracking router

Reuse the lan/router/net topology from Lesson 22 (with masquerade enabled), or a simpler two-namespace setup where the router forwards between them. Conntrack runs on the **router** (the box the traffic passes through).

For a minimal setup, the router just needs to be in the path and have nftables loaded (loading any nft ruleset, or just the presence of conntrack, enables tracking):

```bash
# Ensure conntrack is active on the router (a no-op filter table is enough)
$ sudo ip netns exec router nft add table inet filter
$ sudo ip netns exec router nft 'add chain inet filter forward { type filter hook forward priority 0 ; }'
$ sudo ip netns exec router nft add rule inet filter forward ct state established,related accept
$ sudo ip netns exec router nft add rule inet filter forward ct state new accept
```

### Step 2 — Stream conntrack events on the router

```bash
$ sudo ip netns exec router conntrack -E -p tcp &
```

### Step 3 — Make a TCP connection through the router

```bash
# Listener on the far side
$ sudo ip netns exec net nc -l -p 9000 &
# Connect from the LAN
$ sudo ip netns exec lan nc 10.0.0.2 9000
# (type a line, then Ctrl-D to close cleanly)
```

### Step 4 — Read the lifecycle in the event stream

The `conntrack -E` output shows the flow's progression:

```
[NEW] tcp ... SYN_SENT       src=192.168.100.10 dst=10.0.0.2 sport=... dport=9000
[UPDATE] tcp ... SYN_RECV    ...                              # SYN-ACK seen
[UPDATE] tcp ... ESTABLISHED ...                              # handshake complete
[UPDATE] tcp ... FIN_WAIT    ...                              # one side closed
[UPDATE] tcp ... CLOSE       ...
[DESTROY] tcp ... ...                                         # entry removed
```

Map this to what you learned in Lesson 6: the `NEW`/`SYN_SENT` corresponds to the SYN; `ESTABLISHED` appears once the SYN-ACK + ACK complete the handshake; the `FIN_WAIT`/`CLOSE`/`DESTROY` track the FIN teardown. Conntrack is watching the very TCP flags you captured earlier and deriving connection state from them.

### Step 5 — Snapshot the table mid-connection

Start a connection and leave it open, then:

```bash
$ sudo ip netns exec router conntrack -L -p tcp
tcp 6 ESTABLISHED src=192.168.100.10 dst=10.0.0.2 sport=43xxx dport=9000 \
    src=10.0.0.2 dst=192.168.100.10 sport=9000 dport=43xxx [ASSURED] ...
```

Both directions are stored. `[ASSURED]` means the flow saw traffic both ways and won't be evicted early under pressure.

### Step 6 — Observe the table count and limit

```bash
$ sudo ip netns exec router sysctl net.netfilter.nf_conntrack_count
net.netfilter.nf_conntrack_count = 3
$ sudo ip netns exec router sysctl net.netfilter.nf_conntrack_max
net.netfilter.nf_conntrack_max = 262144
```

Generate many short connections in a loop and watch `nf_conntrack_count` rise, then fall as flows time out.

### Step 7 — Clean up

```bash
$ sudo kill %1 2>/dev/null   # stop conntrack -E
$ sudo ip netns delete lan router net
```

---

## Further Reading

| Topic | Link |
|---|---|
| Stateful firewall / connection tracking | [Wikipedia — Stateful firewall](https://en.wikipedia.org/wiki/Stateful_firewall) |
| netfilter conntrack | [Wikipedia — netfilter](https://en.wikipedia.org/wiki/Netfilter#Connection_tracking) |
| `conntrack` tool | [man7.org — conntrack(8)](https://man7.org/linux/man-pages/man8/conntrack.8.html) |
| TCP state machine | [Wikipedia — TCP state diagram](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Protocol_operation) |

---

## Checkpoint

**Q1. What state does conntrack assign to the reply packet (SYN-ACK) of an outbound TCP SYN?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The reply is matched as **ESTABLISHED**. Here's the sequence: the outbound SYN is the first packet of a flow conntrack hasn't seen, so it creates a NEW entry. When the SYN-ACK comes back, conntrack matches it to that existing entry's reply-direction tuple — this is *return traffic for a known flow*, so it's classified as ESTABLISHED (not NEW). This is exactly what makes the stateful-firewall idiom `ct state established,related accept` work: you allow the outbound NEW connection once, and all the return packets (starting with the SYN-ACK) are automatically permitted as ESTABLISHED without needing a separate inbound rule. The very first packet a *side* sees in the reply direction already counts as part of the established connection because conntrack created the entry from the original SYN.
</details>

---

**Q2. What is the difference between the `RELATED` and `ESTABLISHED` states? Give an example of RELATED.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**ESTABLISHED** means a packet belongs to an existing tracked connection — it matches a flow conntrack already knows, in either direction. **RELATED** means a packet starts a *new* connection (a different 5-tuple) that is logically associated with an existing one, so conntrack treats it as expected rather than unsolicited.

Classic RELATED examples:
- **FTP active mode:** the control connection on port 21 negotiates a *separate* data connection on another port. Conntrack's FTP helper recognizes the negotiation and marks the incoming data connection as RELATED to the control flow, so a firewall can allow it without a permanently open inbound rule.
- **ICMP errors:** an ICMP "destination unreachable" or "time exceeded" generated in response to one of your flows is RELATED to that flow.

The distinction matters for firewalling: `ct state established,related accept` permits both the return traffic of known connections (ESTABLISHED) *and* these legitimately-spawned auxiliary connections (RELATED), while still dropping truly unsolicited NEW inbound traffic.
</details>

---

**Q3. Your router under heavy load starts logging `nf_conntrack: table full, dropping packet` and new connections fail while existing ones are fine. What's happening and what are two ways to address it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The conntrack table has hit its capacity limit (`net.netfilter.nf_conntrack_max`). Every tracked flow occupies a slot; once the table is full, the kernel can't create entries for *new* connections, so it drops their first packets — which is why new connections fail while already-tracked (established) ones keep working. This commonly happens under connection floods, SYN floods, or simply very high legitimate connection rates (busy proxies, load balancers, NAT gateways).

Two ways to address it:
1. **Raise the limit / size the hash table.** Increase `nf_conntrack_max` (and the corresponding hash bucket count) to accommodate the real connection volume, assuming the box has memory for it — each entry costs a few hundred bytes.
2. **Reduce how long entries live / how many you create.** Lower the per-state timeouts (especially for half-open and TIME_WAIT-like states) so dead flows are evicted faster, and/or *don't track* traffic that doesn't need it using a `notrack` rule in the `raw` prerouting hook (e.g., bulk traffic that needs no NAT or stateful filtering). For very high throughput, offloading established flows with a flowtable (Lesson 28) also relieves pressure.

The root insight: connection tracking is powerful but finite, so on high-traffic devices its capacity and timeouts are tuning parameters, not set-and-forget defaults.
</details>

---

## Homework

Run `conntrack -E` on the router and, in another terminal, generate three different kinds of traffic through it: (a) a normal TCP connection that completes cleanly, (b) a UDP exchange (e.g. a DNS query), and (c) a ping. For each, note what conntrack states and timeouts you observe. Explain why UDP and ICMP — which have no handshake — can still be "tracked," and how conntrack decides one of these flows is over.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**(a) TCP:** you see the full state machine — NEW/SYN_SENT → SYN_RECV → ESTABLISHED → (FIN handling) CLOSE → DESTROY. TCP is the easy case because the protocol itself has explicit connection setup and teardown flags, so conntrack reads SYN/SYN-ACK/FIN/RST and follows along precisely. ESTABLISHED entries get a long timeout (hours by default) once `[ASSURED]`.

**(b) UDP:** UDP has no handshake — it's connectionless. Conntrack still creates an entry from the first packet (a "flow" defined by the 5-tuple) in an unreplied state, and marks it ESTABLISHED once it sees a packet in the reply direction. Since there's no FIN to signal the end, conntrack relies purely on a **timeout**: if no packet matches the flow for the UDP timeout (short, e.g. ~30s, or longer once it's seen as a stream), the entry is destroyed. So "tracking" UDP just means remembering the tuple and aging it out by inactivity.

**(c) ICMP:** ping is tracked by matching echo *requests* to echo *replies* using the ICMP id/sequence as a pseudo-connection key. Conntrack creates a short-lived entry for the request and considers it replied when the matching echo reply returns; it too ages out on a short timeout since there's no explicit close.

The key insight: conntrack doesn't require the protocol to have a real "connection." For connectionless protocols it *synthesizes* a flow from the packet tuple and decides the flow is over by **inactivity timeout** rather than by protocol teardown. TCP gets precise state from its flags; UDP and ICMP get best-effort flows bounded by timers. This is why UDP-heavy workloads can still exhaust the conntrack table — every distinct UDP tuple is an entry, even with no "connection" in the formal sense.
</details>
