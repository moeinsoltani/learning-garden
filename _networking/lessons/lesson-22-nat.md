---
title: "Lesson 22 — NAT with nftables"
nav_order: 22
parent: "Phase 7: NAT & Conntrack"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 22: NAT with nftables

## Concept

**NAT** (Network Address Translation) rewrites the IP addresses (and often ports) in packets as they pass through a router. It's how your home gets by with one public IP while many devices share it: the router rewrites every outbound packet's *source* to its own public address, and rewrites the *replies* back to the right internal device.

```
   LAN (private)                router                 "Internet"
   192.168.100.10  ──src:192.168.100.10──►  [SNAT]  ──src:10.0.0.1──►  server
                   ◄──dst:192.168.100.10──   [un-NAT] ◄──dst:10.0.0.1──
                          the router rewrites src on the way out,
                          and restores dst on the way back
```

Two directions of NAT:

- **SNAT** (Source NAT) — rewrite the **source** address of outbound packets. `masquerade` is the dynamic form (use whatever the outbound interface's current IP is). This is how a LAN shares one public IP.
- **DNAT** (Destination NAT) — rewrite the **destination** of inbound packets. This is **port forwarding**: "traffic to my public IP:80 → send to internal server 192.168.100.10:80."

---

## How it works

NAT lives in **nftables** `nat`-type chains hooked at specific points in the packet path:

- **prerouting** hook — runs *before* the routing decision. DNAT happens here, so the rewritten destination influences how the packet is routed.
- **postrouting** hook — runs *after* routing, just before the packet leaves. SNAT/masquerade happens here, so the source is rewritten on the way out.

The magic that makes return traffic work is **connection tracking** (conntrack — next lesson). When the router SNATs an outbound packet, conntrack records the original tuple (who really sent it). When the reply comes back addressed to the router's public IP, conntrack matches it to that flow and *reverses* the translation, restoring the real internal destination. You only write the rule for one direction; conntrack handles the reverse automatically.

{: .note }
> **masquerade vs snat**
> `snat to <fixed-ip>` rewrites the source to a specific address — use when the router's public IP is static and known. `masquerade` looks up the outbound interface's current address at send time — use when the IP is dynamic (DHCP/PPP uplinks). Masquerade is slightly more expensive (per-packet lookup) but survives IP changes, which is why home routers use it.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `nft add table ip nat` | Create a NAT table |
| `nft add chain ip nat postrouting { type nat hook postrouting priority 100 ; }` | Create the SNAT chain |
| `nft add chain ip nat prerouting { type nat hook prerouting priority -100 ; }` | Create the DNAT chain |
| `nft add rule ip nat postrouting oifname "<if>" masquerade` | Masquerade outbound traffic |
| `nft add rule ip nat prerouting tcp dport 80 dnat to 192.168.100.10` | Port-forward inbound :80 |
| `nft list ruleset` | Show the full ruleset |
| `conntrack -L` | List tracked connections and their NAT translations |

---

## Lab

We'll build the classic three-namespace home-router topology: a LAN host behind a router that masquerades to an "internet" host.

### Step 1 — Build the topology

```bash
$ sudo ip netns add lan
$ sudo ip netns add router
$ sudo ip netns add net      # the "internet"

# lan <-> router  (private subnet 192.168.100.0/24)
$ sudo ip link add lan-r type veth peer name r-lan
$ sudo ip link set lan-r netns lan
$ sudo ip link set r-lan netns router
# router <-> net  ("public" subnet 10.0.0.0/24)
$ sudo ip link add r-net type veth peer name net-r
$ sudo ip link set r-net netns router
$ sudo ip link set net-r netns net

# Addresses
$ sudo ip netns exec lan ip link set lan-r up
$ sudo ip netns exec lan ip addr add 192.168.100.10/24 dev lan-r
$ sudo ip netns exec lan ip route add default via 192.168.100.1

$ sudo ip netns exec router ip link set r-lan up
$ sudo ip netns exec router ip addr add 192.168.100.1/24 dev r-lan
$ sudo ip netns exec router ip link set r-net up
$ sudo ip netns exec router ip addr add 10.0.0.1/24 dev r-net
$ sudo ip netns exec router sysctl -w net.ipv4.ip_forward=1

$ sudo ip netns exec net ip link set net-r up
$ sudo ip netns exec net ip addr add 10.0.0.2/24 dev net-r
```

### Step 2 — Show that without NAT, replies can't get back

The "internet" host (`net`) has **no route** to the private `192.168.100.0/24` — that's the whole point of private addresses.

```bash
$ sudo ip netns exec lan ping -c 1 -W 1 10.0.0.2
# Request reaches net (router forwards it), but net's reply is addressed
# to 192.168.100.10, which net can't route back → ping fails.
```

You can confirm by capturing on `net`: the request arrives with source `192.168.100.10`, and `net` tries to reply but has nowhere to send it.

### Step 3 — Add masquerade (SNAT) on the router

```bash
$ sudo ip netns exec router nft add table ip nat
$ sudo ip netns exec router nft 'add chain ip nat postrouting { type nat hook postrouting priority 100 ; }'
$ sudo ip netns exec router nft add rule ip nat postrouting oifname "r-net" masquerade
```

Now retry:

```bash
$ sudo ip netns exec lan ping -c 2 10.0.0.2
64 bytes from 10.0.0.2: icmp_seq=1 ttl=63 time=0.1 ms      # works!
```

### Step 4 — Watch the translation on the wire

Capture on the "public" side while pinging:

```bash
$ sudo ip netns exec net tcpdump -n -i net-r icmp &
$ sudo ip netns exec lan ping -c 1 10.0.0.2
# net sees:  IP 10.0.0.1 > 10.0.0.2: ICMP echo request
#                ^^^^^^^^
#   source is the ROUTER's public IP, NOT 192.168.100.10 — SNAT rewrote it
```

From `net`'s perspective, the router itself is talking — it never sees the private address. The reply goes to `10.0.0.1`, and conntrack on the router restores the destination to `192.168.100.10` before forwarding it back to the LAN.

### Step 5 — See the conntrack translation entry

```bash
$ sudo ip netns exec router conntrack -L
icmp 1 ... src=192.168.100.10 dst=10.0.0.2 ... src=10.0.0.2 dst=10.0.0.1 ...
#         └─ original direction ──────────┘ └─ reply direction (NATed) ───┘
```

The entry stores both the original tuple and the translated reply tuple — this is the table conntrack uses to un-NAT returning packets.

### Step 6 — Add DNAT (port forwarding)

Make the router forward inbound TCP :8080 to the LAN host:

```bash
$ sudo ip netns exec router nft 'add chain ip nat prerouting { type nat hook prerouting priority -100 ; }'
$ sudo ip netns exec router nft add rule ip nat prerouting iifname "r-net" tcp dport 8080 dnat to 192.168.100.10:8080

# On the LAN host, listen:
$ sudo ip netns exec lan nc -l -p 8080 &
# From the "internet", connect to the ROUTER's public IP:
$ sudo ip netns exec net nc 10.0.0.1 8080
# The connection lands on 192.168.100.10 — DNAT rewrote the destination.
```

### Step 7 — Clean up

```bash
$ sudo ip netns delete lan
$ sudo ip netns delete router
$ sudo ip netns delete net
```

---

## Further Reading

| Topic | Link |
|---|---|
| Network address translation | [Wikipedia — NAT](https://en.wikipedia.org/wiki/Network_address_translation) |
| Port forwarding / DNAT | [Wikipedia — Port forwarding](https://en.wikipedia.org/wiki/Port_forwarding) |
| nftables NAT | [nftables wiki — NAT](https://wiki.nftables.org/wiki-nftables/index.php/Performing_Network_Address_Translation_(NAT)) |
| `nft` | [man7.org — nft(8)](https://man7.org/linux/man-pages/man8/nft.8.html) |

---

## Checkpoint

**Q1. Explain why return traffic reaches the right internal host even though the router rewrote the source IP on the way out.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because of **connection tracking**. When the router SNATs/masquerades an outbound packet, it records the flow in the conntrack table: the *original* tuple (real source `192.168.100.10`, its port, destination, etc.) and the *translated* tuple it rewrote it to (source = router's public IP and a chosen port). The external host replies to the router's public IP. When that reply arrives, conntrack looks it up, finds the matching flow, and **reverses the translation** — rewriting the destination from the router's public IP back to `192.168.100.10` (and the original port) — before forwarding it onto the LAN. You only wrote a rule for the outbound direction; conntrack automatically handles the return direction by remembering each flow's original addresses. Without conntrack, the router would have no way to know which internal host a given reply belongs to, since many hosts share the one public IP.
</details>

---

**Q2. What is the difference between SNAT and DNAT, and at which hook does each occur?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**SNAT (Source NAT)** rewrites the *source* address of packets, typically on the way *out* of a network — it's how a private LAN shares one public IP (`masquerade` is its dynamic form). SNAT happens at the **postrouting** hook, after the routing decision, just before the packet leaves the router, so the routing was based on the real addresses and only the source is changed on egress.

**DNAT (Destination NAT)** rewrites the *destination* address of packets, typically on the way *in* — it's how port forwarding sends traffic for the router's public IP to an internal server. DNAT happens at the **prerouting** hook, *before* the routing decision, so that the new (internal) destination is what the router then routes toward. The placement is essential: DNAT must run before routing so the packet gets routed to the real target, while SNAT runs after routing so the egress source reflects the router's address.
</details>

---

**Q3. When would you use `masquerade` instead of `snat to <ip>`?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Use `masquerade` when the router's outbound (public) IP address is **dynamic** or not known in advance — for example, a home/WAN uplink that gets its address via DHCP or PPPoE and may change. `masquerade` looks up the outbound interface's current address at the moment each packet is sent, so it automatically uses whatever the IP currently is, and keeps working across IP changes (it even flushes affected conntrack entries when the interface goes down). Use `snat to <fixed-ip>` when the public IP is **static and known**: it's slightly more efficient because it doesn't have to look up the interface address per packet, and it lets you map to a specific address (or pool) explicitly. So: dynamic uplink → masquerade; static, known address → snat. Functionally both do source NAT; masquerade is just the convenient, address-agnostic variant.
</details>

---

## Homework

Extend the lab with a *second* LAN host (`192.168.100.20`). Have both LAN hosts open connections to the same "internet" server on the same destination port simultaneously. Examine `conntrack -L` on the router. Explain how the router keeps the two flows separate when both are masqueraded behind the same public IP — and what would happen if both happened to pick the same source port.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With both LAN hosts (`.10` and `.20`) connecting to the same server:port behind one public IP, the router creates **two distinct conntrack entries**, one per flow. Each entry records the original source (`.10:portA` vs `.20:portB`) and the translated source (router's public IP : some port). The reply packets are distinguished by the **full 5-tuple** — protocol, source IP, source port, destination IP, destination port. Even though both share the router's public IP and the same destination, their *source ports* (as seen by the server) differ, so each reply maps back to exactly one conntrack entry and thus one internal host. This is why NAT is often called PAT (Port Address Translation): it multiplexes many internal hosts onto one IP by using the port number as the disambiguator.

If both internal hosts happened to choose the **same** source port for their connection to the same server:port, there would be a collision — two flows that, after naive SNAT, would have identical translated 5-tuples, making their replies ambiguous. NAT handles this with **port translation**: the router detects the conflict and rewrites one flow's source port to a different, unused value (port remapping) so the translated tuples stay unique. You'd see this in `conntrack -L` as the translated source port differing from the original. The takeaway: NAT guarantees each flow has a unique translated tuple by rewriting ports when necessary, which is what lets hundreds of devices share a single public address without their replies getting mixed up.
</details>
