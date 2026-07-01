---
title: "Lesson 12 — VLAN Interfaces"
nav_order: 12
parent: "Phase 3: Layer-2 Networking"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 12: VLAN Interfaces

## Concept

A **VLAN** (Virtual LAN) lets one physical cable carry many isolated Layer-2 networks. Each frame gets a small **tag** — a 12-bit VLAN ID (1–4094) inserted into the Ethernet header per the [802.1Q](https://en.wikipedia.org/wiki/IEEE_802.1Q) standard. Switches use the tag to keep traffic in VLAN 10 separate from VLAN 20, even though both ride the same wire.

```
   Untagged frame:   [ dst MAC | src MAC | EtherType | payload ]
   802.1Q tagged:    [ dst MAC | src MAC | 0x8100 | VID=10 | EtherType | payload ]
                                            └──── the VLAN tag ────┘
```

Two terms you must internalize:

- **Access port** — carries exactly one VLAN, untagged. The device behind it (a PC, a server) doesn't know VLANs exist; the switch adds/removes the tag.
- **Trunk port** — carries *many* VLANs, tagged. Used between switches (or switch↔router) to multiplex multiple VLANs over a single link.

---

## How it works

On Linux there are two distinct ways to deal with VLANs:

1. **VLAN sub-interfaces** — `eth0.10` is a virtual interface that tags everything it sends with VID 10 and only receives VID-10 frames from `eth0`. This is how a host participates in a trunk: each `eth0.N` is one VLAN.

2. **VLAN-aware bridge** — a single bridge that understands tags, assigning VLANs per-port (`bridge vlan add`). This is how a software switch implements access and trunk ports. Modern, flexible, and how container/VM hosts do it.

```
   ns1 (VLAN 10)    ns3 (VLAN 10)        ns2 (VLAN 20)
      │                 │                    │
   access            access               access
   port pvid 10      port pvid 10         port pvid 20
      └────────┬────────┘                    │
        ┌──────┴────────────────────────────┴──────┐
        │        br0  (VLAN-aware bridge)            │
        └────────────────────────────────────────────┘
   ns1 ↔ ns3 can talk (same VLAN).  ns2 is isolated (different VLAN).
```

{: .note }
> **`pvid` and `untagged` explained**
> On a VLAN-aware bridge port, `pvid` (Port VLAN ID) is the VLAN assigned to *untagged* frames arriving on that port — it's what makes a port an "access" port for that VLAN. `untagged` means egress frames for that VLAN leave *without* a tag (so the connected device, which doesn't speak VLANs, gets plain Ethernet). A trunk port instead has multiple VIDs and omits `untagged`/`pvid`, so frames keep their tags. The combination `pvid untagged` on a single VID is the canonical access-port configuration.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip link add link eth0 name eth0.10 type vlan id 10` | Create a VLAN sub-interface for VID 10 |
| `ip link add br0 type bridge vlan_filtering 1` | Create a VLAN-aware bridge |
| `bridge vlan add dev <port> vid 10 pvid untagged` | Make a port an access port for VLAN 10 |
| `bridge vlan add dev <port> vid 10` | Add VLAN 10 to a trunk port (tagged) |
| `bridge vlan show` | Show per-port VLAN membership |

---

## Lab

We'll build the VLAN-aware bridge topology above: ns1 and ns3 in VLAN 10, ns2 in VLAN 20, all on one bridge. ns1↔ns3 will work; ns2 will be isolated.

### Step 1 — Create a VLAN-aware bridge

```bash
$ sudo ip link add br0 type bridge vlan_filtering 1
$ sudo ip link set br0 up
```

### Step 2 — Wire three namespaces to the bridge

```bash
for n in 1 2 3; do
  sudo ip netns add ns$n
  sudo ip link add veth-ns$n type veth peer name br-ns$n
  sudo ip link set veth-ns$n netns ns$n
  sudo ip link set br-ns$n master br0
  sudo ip link set br-ns$n up
  sudo ip netns exec ns$n ip link set veth-ns$n up
done

# All three share subnet 10.0.0.0/24 (VLANs will provide the isolation)
sudo ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth-ns1
sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth-ns2
sudo ip netns exec ns3 ip addr add 10.0.0.3/24 dev veth-ns3
```

### Step 3 — Assign VLANs per port (access ports)

```bash
# ns1 and ns3 → VLAN 10 ; ns2 → VLAN 20
$ sudo bridge vlan add dev br-ns1 vid 10 pvid untagged
$ sudo bridge vlan add dev br-ns3 vid 10 pvid untagged
$ sudo bridge vlan add dev br-ns2 vid 20 pvid untagged

# Remove default VLAN 1 from these access ports to enforce isolation cleanly
$ sudo bridge vlan del dev br-ns1 vid 1
$ sudo bridge vlan del dev br-ns2 vid 1
$ sudo bridge vlan del dev br-ns3 vid 1
```

### Step 4 — Inspect VLAN membership

```bash
$ sudo bridge vlan show
port     vlan-id
br-ns1   10 PVID Egress Untagged
br-ns2   20 PVID Egress Untagged
br-ns3   10 PVID Egress Untagged
```

### Step 5 — Verify isolation

```bash
# Same VLAN (10) — works:
$ sudo ip netns exec ns1 ping -c 1 10.0.0.3
64 bytes from 10.0.0.3: icmp_seq=1 ttl=64 time=0.07 ms

# Different VLAN (10 → 20) — fails, even though same subnet & same bridge:
$ sudo ip netns exec ns1 ping -c 1 -W 1 10.0.0.2
# 100% packet loss
```

ns1 and ns3 communicate because they share VLAN 10. ns2 is unreachable despite being on the same subnet and same physical bridge — the VLAN tag keeps it in a separate broadcast domain.

### Step 6 — Confirm with tcpdump that ns2 never sees VLAN-10 traffic

```bash
$ sudo ip netns exec ns2 tcpdump -e -n -i veth-ns2 &
$ sudo ip netns exec ns1 ip neigh flush all
$ sudo ip netns exec ns1 ping -c 1 10.0.0.3
# ns2's capture stays silent — the broadcast ARP for VLAN 10 is NOT flooded
# into VLAN 20. The bridge confines flooding to ports in the same VLAN.
```

This is the proof: even broadcast traffic (ARP) is contained within its VLAN. The single bridge behaves as two independent switches.

### Step 7 — Clean up

```bash
$ sudo ip link del br0
$ for n in 1 2 3; do sudo ip netns delete ns$n; done
```

---

## Further Reading

| Topic | Link |
|---|---|
| IEEE 802.1Q VLAN tagging | [Wikipedia — IEEE 802.1Q](https://en.wikipedia.org/wiki/IEEE_802.1Q) |
| Virtual LAN | [Wikipedia — VLAN](https://en.wikipedia.org/wiki/VLAN) |
| `bridge` VLAN filtering | [man7.org — bridge(8)](https://man7.org/linux/man-pages/man8/bridge.8.html) |
| Trunking | [Wikipedia — Trunking](https://en.wikipedia.org/wiki/Trunking) |

---

## Checkpoint

**Q1. Explain how one physical cable can carry multiple isolated networks simultaneously.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Each Ethernet frame on the cable carries a small 802.1Q **VLAN tag** — a 12-bit VLAN ID inserted into the header. Switches and VLAN-aware endpoints read this tag and treat frames with different IDs as belonging to different, isolated Layer-2 networks. A switch will only forward and flood a frame among ports that belong to that frame's VLAN, so VLAN 10 traffic never reaches VLAN 20 ports even though all the frames physically traverse the same wire. The cable is shared at the physical layer; the tag enforces logical separation at L2. This is called a *trunk* when the link carries multiple tagged VLANs at once.
</details>

---

**Q2. What is the difference between an access port and a trunk port?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An **access port** belongs to a single VLAN and handles **untagged** frames — the connected device (a PC, server, or container) is unaware of VLANs; the switch adds the VLAN tag on ingress and strips it on egress (configured on Linux with `pvid untagged`). A **trunk port** carries **multiple VLANs**, each frame keeping its tag, and is used between switches or between a switch and a router/host that participates in several VLANs at once. In short: access = one VLAN, untagged, edge devices; trunk = many VLANs, tagged, infrastructure links.
</details>

---

**Q3. ns1 (VLAN 10) and ns2 (VLAN 20) are on the same bridge and the same IP subnet `10.0.0.0/24`. ns1 cannot ping ns2. Is this a routing problem? Explain.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
No, it's not a routing problem — it's L2 isolation by VLAN. ns1 and ns2 are in the same subnet, so ns1 *believes* ns2 is directly reachable and ARPs for it. But the VLAN-aware bridge confines that broadcast ARP to VLAN 10 ports only; it never floods into VLAN 20, so ns2 never hears the request and never replies. Without an ARP reply, ns1 has no MAC to send to, and the ping fails at L2. Routing isn't involved at all (same subnet, no gateway). To let them communicate you'd either put them in the same VLAN, or — if they genuinely should be different networks — give them different subnets and route between the VLANs with an L3 device (router-on-a-stick / inter-VLAN routing). The VLAN tag created two separate broadcast domains on one physical bridge.
</details>

---

## Homework

Reconfigure the lab so that one extra "router" namespace is connected to the bridge via a **trunk port** carrying both VLAN 10 and VLAN 20. Inside the router namespace, create two VLAN sub-interfaces (`eth0.10`, `eth0.20`), give each an address in a *different* subnet, enable forwarding, and route between the VLANs so ns1 (VLAN 10) can finally reach ns2 (VLAN 20). Sketch the config. (This is "router-on-a-stick.")

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The trick is that the VLANs must become *different subnets* and a router bridges them at L3. Outline:

1. Put VLAN 10 on subnet `10.0.10.0/24` and VLAN 20 on `10.0.20.0/24` (re-address ns1/ns3 and ns2 accordingly). VLANs map to subnets.
2. Add a router namespace with a veth to the bridge, and make that bridge port a **trunk**: `bridge vlan add dev br-router vid 10` and `bridge vlan add dev br-router vid 20` (tagged, no `pvid untagged`).
3. Inside the router, create sub-interfaces on the trunk: `ip link add link eth0 name eth0.10 type vlan id 10` and `eth0.20 type vlan id 20`. Give `eth0.10` address `10.0.10.254/24` and `eth0.20` address `10.0.20.254/24` — these are the gateways for each VLAN.
4. Enable forwarding in the router: `sysctl net.ipv4.ip_forward=1`.
5. Point each namespace's default route at its VLAN gateway: ns1 → `10.0.10.254`, ns2 → `10.0.20.254`.

Now ns1→ns2 goes: ns1 sends to its gateway (router's `eth0.10`, tagged VLAN 10 on the trunk), the router routes it to the `10.0.20.0/24` interface, and it leaves as `eth0.20` (tagged VLAN 20) to ns2. One physical link (the trunk) carries both VLANs; the router moves packets between them at L3. This is the classic "router-on-a-stick" design, and it shows the only correct way to connect two VLANs is *routing*, not bridging.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 13 — MACVLAN →](lesson-13-macvlan){: .btn .btn-primary }
