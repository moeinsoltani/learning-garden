---
title: "Lesson 24 — DHCP"
nav_order: 24
parent: "Phase 7: NAT & Conntrack"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 24: DHCP

## Concept

So far you've typed every IP address by hand. **DHCP** (Dynamic Host Configuration Protocol) automates that: a host boots with no address, shouts "does anyone configure me?", and a DHCP server hands it an IP, subnet mask, gateway, and DNS servers — a **lease** for a limited time.

The exchange is four messages, remembered as **DORA**:

```
   Client (no IP yet)                         Server
        │                                        │
        │── DISCOVER (broadcast) ───────────────►│   "Is there a DHCP server?"
        │◄──────────────── OFFER (broadcast) ────│   "Yes — here's 192.168.1.50"
        │── REQUEST (broadcast) ────────────────►│   "I'll take 192.168.1.50"
        │◄──────────────── ACK (broadcast) ──────│   "It's yours, lease 1h"
        ▼                                        ▼
   now configured with 192.168.1.50
```

---

## How it works

The crucial detail: when the client starts, it has **no IP address**, so it cannot send normal unicast IP packets. DISCOVER is sent as a **broadcast** — source `0.0.0.0`, destination `255.255.255.255` — so it reaches any DHCP server on the local segment without needing addresses. The server replies (also broadcast, since the client still has no usable address to unicast to). DHCP runs over **UDP**, server port **67**, client port **68**.

Because DISCOVER is a broadcast, it stays within one L2 broadcast domain — **it does not cross routers**. In real networks where the DHCP server is centralized and clients are on many subnets, a **DHCP relay** (relay agent) on each subnet catches the broadcast and forwards it as unicast to the server, then relays the reply back. This is the Linux equivalent of Cisco's `ip helper-address`.

{: .note }
> **Lease lifecycle**
> A lease isn't forever. The client tries to **renew** at ~50% of the lease time (T1) by unicasting a REQUEST directly to the server, and **rebinds** (broadcasts) at ~87.5% (T2) if the original server didn't answer. If the lease fully expires, the client must stop using the address and start over with DISCOVER. Lease files (e.g. `/var/lib/dhcp/dhclient.leases`) record what the client currently holds.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `dnsmasq --interface=<if> --dhcp-range=<lo>,<hi>,<lease>` | Run dnsmasq as a DHCP server |
| `dhclient <iface>` | Request a lease (ISC client) |
| `dhclient -r <iface>` | Release the current lease |
| `dhcpcd <iface>` | Alternative DHCP client |
| `dhcrelay -i <if> <server-ip>` | Run a DHCP relay agent |
| `tcpdump -n -i <if> port 67 or port 68` | Capture the DORA exchange |

---

## Lab

We'll run dnsmasq as a DHCP server in a router namespace and a dhclient in a client namespace, capturing the full DORA exchange.

### Step 1 — Two namespaces on one segment

```bash
$ sudo ip netns add server
$ sudo ip netns add client
$ sudo ip link add s-c type veth peer name c-s
$ sudo ip link set s-c netns server
$ sudo ip link set c-s netns client

# Server gets a static address; client starts with NONE
$ sudo ip netns exec server ip link set s-c up
$ sudo ip netns exec server ip addr add 192.168.50.1/24 dev s-c
$ sudo ip netns exec client ip link set c-s up
# (no address on the client — DHCP will provide it)
```

### Step 2 — Run dnsmasq as the DHCP server

```bash
$ sudo ip netns exec server dnsmasq \
    --no-daemon \
    --interface=s-c \
    --bind-interfaces \
    --dhcp-range=192.168.50.100,192.168.50.150,1h \
    --dhcp-option=3,192.168.50.1 \
    --log-dhcp &
# --dhcp-range: pool .100–.150, 1-hour leases
# --dhcp-option=3: advertise 192.168.50.1 as the default gateway
```

### Step 3 — Start a capture, then request a lease

```bash
# Terminal A — capture DHCP on the client side
$ sudo ip netns exec client tcpdump -n -i c-s port 67 or port 68 &

# Terminal B — ask for a lease
$ sudo ip netns exec client dhclient -v c-s
```

### Step 4 — Read the DORA exchange

The capture shows all four messages:

```
0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request, ... DHCP-Message: Discover
192.168.50.1.67 > 255.255.255.255.68: BOOTP/DHCP, Reply, ... DHCP-Message: Offer, ... yiaddr 192.168.50.100
0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request, ... DHCP-Message: Request, requested-ip 192.168.50.100
192.168.50.1.67 > 255.255.255.255.68: BOOTP/DHCP, Reply, ... DHCP-Message: ACK, yiaddr 192.168.50.100
```

Notice the source/destination addresses:
- DISCOVER and REQUEST come **from `0.0.0.0`** (client has no IP yet) **to `255.255.255.255`** (broadcast).
- The server's replies go to the broadcast address too, on UDP 67↔68.
- `yiaddr` ("your IP address") is the address being offered/assigned: `192.168.50.100`.

### Step 5 — Confirm the client got configured

```bash
$ sudo ip netns exec client ip addr show c-s
    inet 192.168.50.100/24 ...        # leased address now assigned
$ sudo ip netns exec client ip route show
default via 192.168.50.1 ...          # gateway came from dhcp-option 3
```

The client went from no configuration to a full IP + gateway, entirely via DHCP.

### Step 6 — Release the lease

```bash
$ sudo ip netns exec client dhclient -r c-s
# Sends a DHCPRELEASE; the server frees 192.168.50.100 back to the pool.
```

### Step 7 — Clean up

```bash
$ sudo kill %1 %2 2>/dev/null
$ sudo ip netns delete server client
```

---

## Further Reading

| Topic | Link |
|---|---|
| DHCP | [Wikipedia — Dynamic Host Configuration Protocol](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) |
| DHCP relay / helper | [Wikipedia — DHCP (relay agents)](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol#DHCP_relaying) |
| dnsmasq | [Wikipedia — dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq) |
| `dhclient` | [man7.org — dhclient(8)](https://man7.org/linux/man-pages/man8/dhclient.8.html) |

---

## Checkpoint

**Q1. Why can't a DHCP DISCOVER cross a router without a relay agent? (Hint: what address does DISCOVER go to?)**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because DISCOVER is sent to the **broadcast address `255.255.255.255`** (from source `0.0.0.0`, since the client has no IP yet). Routers do not forward broadcasts — a broadcast is confined to the single L2 broadcast domain it originated in, by design (otherwise broadcasts would flood the entire internet). So a DHCP server on a *different* subnet never hears the DISCOVER; the broadcast dies at the first router.

A **DHCP relay agent** solves this: it runs on the client's subnet, catches the broadcast DISCOVER, and re-sends it as a **unicast** packet directly to the known DHCP server's IP (recording which subnet/interface it came from so the server picks the right pool). The server's reply is unicast back to the relay, which then broadcasts it onto the client's segment. This is exactly what Cisco's `ip helper-address` does and what Linux's `dhcrelay` provides. Without a relay, the DHCP server must be on the same L2 segment as the clients.
</details>

---

**Q2. List the four DHCP messages in order and state, for each, whether the client or server sends it and what it accomplishes.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The DORA exchange:

1. **DISCOVER** — *client → broadcast.* The client, having no address, broadcasts to find any available DHCP server(s) on the segment.
2. **OFFER** — *server → client.* Each server that hears the DISCOVER responds with a proposed lease (an available IP plus parameters like subnet mask, gateway, DNS). There may be multiple offers if multiple servers exist.
3. **REQUEST** — *client → broadcast.* The client picks one offer and broadcasts a request for that specific address. It's broadcast (not unicast) so the *other* servers also learn their offers were declined and can reclaim them.
4. **ACK** — *server → client.* The chosen server confirms the assignment and commits the lease (with its duration). The client may now configure and use the address. (If the address became unavailable, the server sends NAK instead, and the client restarts.)

Mnemonic: **D**iscover, **O**ffer, **R**equest, **A**ck → DORA.
</details>

---

**Q3. A lease is for 1 hour. At what points does the client try to renew, and what happens if every renewal attempt fails?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The client uses two timers within the lease:

- **T1 (~50% of the lease, ~30 min here):** the client tries to **renew** by sending a REQUEST *unicast* directly to the server that granted the lease. If the server ACKs, the lease is extended and the timers reset. This is the normal, quiet renewal that keeps a stable address.
- **T2 (~87.5%, ~52.5 min here):** if renewal didn't succeed (the original server is unreachable), the client **rebinds** — it *broadcasts* a REQUEST so that *any* DHCP server on the segment can extend or take over the lease.

If even rebinding fails and the lease reaches **100% (1 hour)**, it **expires**: the client must immediately stop using the address, deconfigure the interface, and start the whole process over from DISCOVER. This staged renew→rebind→expire design lets a client keep its address through brief server outages (it has the whole second half of the lease to recover) while guaranteeing it never keeps using an address it no longer has a valid lease for.
</details>

---

## Homework

Build a two-subnet topology where the DHCP server is on subnet A and the client is on subnet B, separated by a router. Confirm the client *cannot* get a lease directly. Then run `dhcrelay` on the router (on subnet B's interface, pointing at the server) and confirm the client now gets a lease. Capture on both the client side and the server side, and explain how the packets differ (broadcast vs unicast, and the `giaddr` field).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Without the relay:** the client on subnet B broadcasts DISCOVER, but the router doesn't forward broadcasts, so the DHCP server on subnet A never sees it. The client times out and gets no lease — proving DHCP is confined to one broadcast domain.

**With `dhcrelay` on the router:** the relay listens on subnet B's interface. When it hears the client's broadcast DISCOVER, it:
1. Fills in the **`giaddr`** (gateway/relay IP address) field of the DHCP packet with the address of the interface it received the broadcast on (e.g., subnet B's router interface). This tells the server which subnet the client is on, so the server selects the correct address pool.
2. **Unicasts** the DISCOVER directly to the configured DHCP server's IP on subnet A.

The server replies **unicast to the relay** (the `giaddr`), and the relay forwards the OFFER/ACK back onto subnet B toward the client (broadcast on that segment).

**Capture differences:**
- *Client side (subnet B):* you see ordinary broadcast DORA — `0.0.0.0 → 255.255.255.255` for client messages, with `giaddr` set in the packets the relay puts back onto the wire.
- *Server side (subnet A):* you see **unicast** packets between the relay's IP and the server's IP, and crucially the `giaddr` field is populated with subnet B's relay-interface address. The server uses `giaddr` both to choose the pool and to know where to send the reply.

The lesson: the relay bridges the broadcast gap by converting the client's local broadcast into a unicast conversation with a remote server, using `giaddr` as the "this client lives on this subnet" marker. This is how one central DHCP server can serve hundreds of subnets it isn't directly attached to.
</details>
