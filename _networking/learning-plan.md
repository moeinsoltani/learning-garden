---
title: Learning Plan
nav_order: 2
nav_exclude: false
---

# Linux Networking Learning Plan

A bottom-up curriculum built on one idea: **network namespaces let you build any
network topology on a single Linux machine**. Starting from two isolated namespaces,
the phases add virtual interfaces, bridging, routing, firewalling, traffic control,
eBPF, tunnels & VPNs, data-center fabrics, and observability — every topic explored
with real commands and real packets.

{: .note }
> Before Lesson 1, prepare your VM with the [Lab Setup]({{ '/lab-setup.html' | relative_url }})
> page. Work the lessons in order — each builds on the ones before it.

---

## [Phase 1: Foundations — Namespaces & Basic IP Tools](lessons/phase-01-foundations.html)

### [Lesson 1: What a network namespace is](lessons/lesson-01-namespaces-intro.html)

**Goal:** Understand that a namespace is an independent networking universe.

**Topics:**
- What is isolated: interfaces, routes, ARP/neighbor tables, firewall rules, sockets
- What is shared with the host
- Why containers use namespaces

**Checkpoint:** Predict what happens if a process moves into another namespace.

---

### [Lesson 2: The "host is just another namespace" model](lessons/lesson-02-host-as-namespace.html)

**Goal:** Stop thinking of the host as special.

**Topics:**
- Initial namespace
- Additional namespaces
- Packet visibility across namespaces
- Why interfaces must connect namespaces

**Checkpoint:** Explain why a container cannot automatically see the host's interfaces.

---

### [Lesson 3: Build two isolated namespaces](lessons/lesson-03-build-namespaces.html)

**Goal:** Experience isolation directly.

**Topics:**
- `ip netns add`, `ip netns exec`, `ip netns list`
- Observe independent routing tables with `ip route`

**Lab:**

```
Host | ns1 | ns2
```

**Checkpoint:** Why can't ns1 ping ns2 yet?

---

### [Lesson 4: IP addressing fundamentals (addrs)](lessons/lesson-04-ip-addressing.html)

**Goal:** Master `ip addr` and understand how IP addresses are assigned to interfaces.

**Topics:**
- `ip link show` — interface state, MTU, MAC address
- `ip addr add/del/show` — CIDR notation, scope
- Primary vs secondary addresses and what happens when primary is deleted
- Address scopes: global, link, host
- IPv4 and IPv6 side by side

**Lab:** Assign IPv4 and IPv6 addresses inside namespaces. Delete the primary and observe.

**Checkpoint:** What is the difference between a primary and secondary address? What happens to secondaries when you delete the primary?

---

### [Lesson 5: Neighbor tables & ARP (neigh)](lessons/lesson-05-arp-neigh.html)

**Goal:** Understand how L3 addresses are resolved to L2 MACs.

**Topics:**
- ARP protocol mechanics (BROADCAST request, UNICAST reply)
- `ip neigh show/add/del/flush`
- ARP cache states: REACHABLE, STALE, DELAY, PROBE, FAILED, PERMANENT
- Gratuitous ARP and why it matters (IP takeover, failover)
- NDP (Neighbor Discovery Protocol) for IPv6

**Lab:** Manually add a static ARP entry. Observe ARP traffic with `tcpdump -e` to see MAC addresses.

**Checkpoint:** Why does `ping` fail even with a correct route if ARP resolution fails?

---

### [Lesson 6: tcpdump — your constant companion](lessons/lesson-06-tcpdump.html)

**Goal:** Learn packet capture as a diagnostic tool you will use in every single lab that follows.

**Topics:**
- `tcpdump -i <iface>`, `-n` (no DNS), `-e` (show MACs), `-vv`, `-w file.pcap`, `-r file.pcap`
- Filters: `host`, `port`, `tcp`, `udp`, `arp`, `icmp`
- Reading ARP, ICMP, TCP handshakes in output
- Capturing inside a namespace: `ip netns exec ns1 tcpdump -i any`
- Capturing both sides of a veth simultaneously (two terminals)

**Lab:** Capture and dissect a full ICMP echo exchange between two namespaces.

**Checkpoint:** Capture a TCP handshake. Identify the SYN, SYN-ACK, and ACK packets by flags.

---

## [Phase 2: Virtual Interfaces](lessons/phase-02-virtual-interfaces.html)

### [Lesson 7: veth pairs](lessons/lesson-07-veth-pairs.html)

**Goal:** Understand the virtual Ethernet cable — the fundamental building block of all Linux virtual networking.

**Topics:**
- `ip link add veth0 type veth peer name veth1`
- Packets entering one end exit the other
- Moving one end into a namespace: `ip link set veth1 netns ns1`
- Why veths always come in pairs

**Lab:**

```
ns1 <----veth----> ns2
```

**Checkpoint:** Trace a packet from ns1 to ns2. What does `tcpdump` show on each veth end?

---

### [Lesson 8: Loopback interface](lessons/lesson-08-loopback.html)

**Goal:** Understand why every namespace needs its own loopback.

**Topics:**
- What `lo` is and why applications bind to it
- Why new namespaces start with lo DOWN
- `ip link set lo up` inside a namespace
- Why 127.0.0.1 is not routed to the physical network

**Checkpoint:** Why does 127.0.0.1 work even when disconnected from all networks?

---

### [Lesson 9: Bond interfaces](lessons/lesson-09-bonding.html)

**Goal:** Combine multiple links for redundancy or throughput.

**Topics:**
- `ip link add bond0 type bond`
- Bond modes: `active-backup` (failover), `balance-rr` (round-robin), `802.3ad` (LACP)
- Adding slave interfaces: `ip link set eth1 master bond0`
- Monitoring slave state

**Checkpoint:** When would you choose `active-backup` over `802.3ad` LACP?

---

### [Lesson 10: Dummy interfaces](lessons/lesson-10-dummy.html)

**Goal:** Create a virtual IP endpoint that exists only in software.

**Topics:**
- `ip link add dummy0 type dummy`
- Use cases: stable loopback IPs for routing, testing, Kubernetes node IPs
- Difference from loopback

**Checkpoint:** Why create an interface that goes nowhere?

---

## [Phase 3: Layer-2 Networking](lessons/phase-03-layer2.html)

### [Lesson 11: Linux bridges](lessons/lesson-11-bridges.html)

**Goal:** Build a software switch inside the kernel.

**Topics:**
- `ip link add br0 type bridge`
- Attaching ports: `ip link set veth0 master br0`
- MAC learning and the forwarding database: `bridge fdb show`
- `bridge link show`, `bridge vlan show`
- Difference between a bridge (L2 switch) and a router (L3 forwarder)

**Lab:**

```
ns1
  \
   br0 (host)
  /
ns2
```

**Checkpoint:** Why can two namespaces communicate through the bridge without any IP routing?

---

### [Lesson 12: VLAN interfaces](lessons/lesson-12-vlans.html)

**Goal:** Carry multiple isolated networks over one cable.

**Topics:**
- 802.1Q tagging: what the tag looks like in a frame
- `ip link add link eth0 name eth0.10 type vlan id 10`
- Trunk ports (carry multiple VLANs) vs access ports (one VLAN, untagged)
- Bridge VLAN filtering: `bridge vlan add dev veth0 vid 10 pvid untagged`

**Lab:**

```
ns1 (VLAN 10)   ns3 (VLAN 10)
ns2 (VLAN 20)
        |
      br0 (VLAN-aware bridge)
```

ns1 and ns3 can communicate. ns2 cannot reach them.

**Checkpoint:** Explain how one physical cable carries multiple isolated networks. Verify isolation with tcpdump.

---

### [Lesson 13: MACVLAN](lessons/lesson-13-macvlan.html)

**Goal:** Give a namespace its own MAC address and presence on the physical LAN.

**Topics:**
- `ip link add macvlan0 link eth0 type macvlan mode bridge`
- MACVLAN modes: `bridge`, `private`, `vepa`, `passthru`
- Host ↔ MACVLAN communication limitation (why they cannot talk to each other)
- MACVTAP: MACVLAN + TAP for VMs

**Checkpoint:** Why does a MACVLAN container appear as a separate machine on the LAN?

---

### [Lesson 14: TAP interfaces](lessons/lesson-14-tap.html)

**Goal:** Understand interfaces that hand Ethernet frames to userspace (used by VMs).

**Topics:**
- TUN (L3, IP packets) vs TAP (L2, Ethernet frames)
- `ip tuntap add dev tap0 mode tap`
- Why QEMU/KVM attaches VM NICs to TAP interfaces
- TAP vs veth: who consumes the packets

**Checkpoint:** Why would a VM use TAP instead of a veth?

---

## [Phase 4: Overlay Networks](lessons/phase-04-overlay.html)

### [Lesson 15: VXLAN](lessons/lesson-15-vxlan.html)

**Goal:** Stretch a Layer-2 network across a Layer-3 infrastructure.

**Topics:**
- VXLAN encapsulation: inner Ethernet frame wrapped in UDP (port 4789), with a 24-bit VNI
- `ip link add vxlan0 type vxlan id 42 dstport 4789 remote 10.0.0.2 dev eth0`
- Unicast VXLAN (point-to-point FDB entries) vs multicast VXLAN
- Populating the FDB manually: `bridge fdb append 00:00:00:00:00:00 dev vxlan0 dst 10.0.0.2`
- Why Kubernetes CNIs and data-center fabrics use VXLAN

**Lab:**

```
Host A (10.0.0.1)               Host B (10.0.0.2)
 ns1 (192.168.100.1)  <-VXLAN-> ns2 (192.168.100.2)
```

Capture the outer UDP packet with tcpdump. Identify the VXLAN header.

**Checkpoint:** How does a Layer-2 frame travel across a Layer-3 network? What wraps what?

---

## [Phase 5: Layer-3 — Routing](lessons/phase-05-routing.html)

### [Lesson 16: Routing fundamentals (routes)](lessons/lesson-16-routing-fundamentals.html)

**Goal:** Understand how the kernel decides where to send a packet.

**Topics:**
- `ip route show`, `ip route add`, `ip route del`, `ip route flush`
- Route types: `unicast`, `blackhole`, `unreachable`, `prohibit`
- Route metrics: lower wins
- Longest prefix match: `/32` beats `/24` beats `0.0.0.0/0`
- `ip route get 8.8.8.8` — ask the kernel which route it would use
- Multiple routing tables: `ip rule list`, `ip route show table 100`

**Lab:** Add a blackhole route. Observe ICMP unreachable vs silent drop.

**Checkpoint:** You have a `/24` route via gateway A and a `0.0.0.0/0` default via gateway B. Traffic to 10.0.1.5 goes where?

---

### [Lesson 17: Routing between namespaces](lessons/lesson-17-routing-namespaces.html)

**Goal:** Use a namespace as a router.

**Topics:**
- Enabling forwarding: `sysctl net.ipv4.ip_forward=1`
- Default gateways: `ip route add default via 10.0.1.1`
- Hop-by-hop packet trace with tcpdump on each interface

**Lab:**

```
ns1 (10.0.1.2/24) <--veth--> router-ns (10.0.1.1 / 10.0.2.1) <--veth--> ns2 (10.0.2.2/24)
```

**Checkpoint:** Trace the packet hop-by-hop. Capture on both veth pairs simultaneously (two tcpdump instances).

---

### [Lesson 18: Policy routing & multiple routing tables](lessons/lesson-18-policy-routing.html)

**Goal:** Route based on source address or firewall mark, not just destination.

**Topics:**
- `ip rule add from 192.168.1.0/24 table 100 priority 100`
- Route tables: `main` (253), `local` (255), `default` (253)
- Marking packets with `nft` and routing by mark: `ip rule add fwmark 0x1 table 200`
- Use cases: multi-homed hosts, VPN split routing

**Checkpoint:** Why would you want traffic from one source IP to take a different path than another source IP going to the same destination?

---

## [Phase 6: Dynamic Routing — FRR](lessons/phase-06-frr.html)

### [Lesson 19: FRR introduction](lessons/lesson-19-frr-intro.html)

**Goal:** Replace static routes with a routing daemon that adapts to topology changes.

**Topics:**
- What FRR is: Free Range Routing, the open-source router suite (successor to Quagga)
- Daemons: `zebra` (RIB manager), `ospfd`, `bgpd`, `staticd`
- `vtysh`: the Cisco-style CLI
- How FRR installs routes into the kernel RIB via zebra
- Installing and starting FRR

**Checkpoint:** What problem does a dynamic routing protocol solve that static routes cannot?

---

### [Lesson 20: OSPF with FRR](lessons/lesson-20-ospf.html)

**Goal:** Build a self-healing routed network inside namespaces.

**Topics:**
- OSPF concepts: areas, LSAs, SPF algorithm, DR/BDR election
- Configuring OSPF in vtysh: `router ospf`, `network 10.0.0.0/24 area 0`
- `show ip ospf neighbor`, `show ip route`
- Redistributing connected routes

**Lab:**

```
ns1 — router-a — router-b — ns2
              \             /
              router-c
```

Kill the link between router-a and router-b. Watch OSPF reconverge via router-c.

**Checkpoint:** Why does traffic reroute automatically when a link fails?

---

### [Lesson 21: BGP with FRR](lessons/lesson-21-bgp.html)

**Goal:** Understand the protocol that connects autonomous systems.

**Topics:**
- eBGP vs iBGP
- AS numbers and prefix advertisement
- BGP path attributes: AS_PATH, LOCAL_PREF, MED
- Simple two-AS lab with FRR

**Lab:**

```
AS 65001 (router-a) <--eBGP--> AS 65002 (router-b)
```

**Checkpoint:** What is the fundamental difference between OSPF (link-state) and BGP (path-vector)?

---

## [Phase 7: NAT, Conntrack & DHCP](lessons/phase-07-nat.html)

### [Lesson 22: NAT with nftables](lessons/lesson-22-nat.html)

**Goal:** Recreate a home router in namespaces.

**Topics:**
- SNAT: rewrite source IP on outbound packets (`masquerade`)
- DNAT: rewrite destination IP for port forwarding
- `nft add rule ip nat POSTROUTING oifname "eth0" masquerade`
- `nft add rule ip nat PREROUTING dport 80 dnat to 192.168.100.10`

**Lab:**

```
LAN namespace (192.168.100.x)
         |
  router-ns (eth0: 10.0.0.1 / eth1: 192.168.100.1)
         |
 "Internet" namespace (10.0.0.2)
```

**Checkpoint:** Explain why return traffic reaches the right host even though the router rewrote the source IP.

---

### [Lesson 23: Conntrack](lessons/lesson-23-conntrack.html)

**Goal:** Understand the state machine that makes NAT and stateful firewalling possible.

**Topics:**
- `conntrack -L` (list connections), `conntrack -E` (event stream)
- Connection states: `NEW`, `ESTABLISHED`, `RELATED`, `INVALID`
- How conntrack ties reply packets back to the original flow
- Conntrack table limits: `nf_conntrack_max`

**Lab:** Watch a TCP connection lifecycle in conntrack from SYN to FIN. Run `conntrack -E` while pinging and making HTTP requests.

**Checkpoint:** What state does conntrack assign to the reply packet of an outbound TCP SYN?

---

### [Lesson 24: DHCP](lessons/lesson-24-dhcp.html)

**Goal:** Automate IP address assignment.

**Topics:**
- DHCP message flow: DISCOVER → OFFER → REQUEST → ACK
- `dnsmasq` as a DHCP server inside a namespace
- `dhclient` or `dhcpcd` as the client
- DHCP lease files
- DHCP relay: why you need it when client and server are on different L2 segments
- `ip helper-address` equivalent in Linux (`dhcrelay`)

**Lab:** Run dnsmasq in router-ns, run dhclient in ns1. Capture the full DORA exchange with tcpdump.

**Checkpoint:** Why can't a DHCP DISCOVER cross a router without a relay agent? (Hint: what address does DISCOVER go to?)

---

## [Phase 8: Firewalling — nftables](lessons/phase-08-nftables.html)

### [Lesson 25: nftables architecture](lessons/lesson-25-nftables-arch.html)

**Goal:** Understand the structure of the modern Linux firewall.

**Topics:**
- Why nftables replaced iptables
- Tables, chains, rules, hooks
- Hook types and priorities: `prerouting` → `input`/`forward` → `postrouting`
- Address families: `ip`, `ip6`, `inet`, `netdev`, `bridge`
- `nft list ruleset`, `nft -f rules.nft`
- Rule syntax: `nft add rule inet filter input tcp dport 22 accept`

**Checkpoint:** What is a hook? Draw the path a packet takes through nftables from arrival to delivery.

---

### [Lesson 26: Packet filtering with nft](lessons/lesson-26-nft-filtering.html)

**Goal:** Build a real stateful firewall.

**Topics:**
- Stateful filtering: `ct state established,related accept`
- Drop INVALID packets: `ct state invalid drop`
- Rate limiting: `limit rate 10/second burst 20 packets`
- Logging: `log prefix "DROPPED: " level warn`
- Default drop policy on a chain

**Lab:** Harden router-ns from Lesson 17: allow only traffic for established connections to be forwarded, drop all else. Verify with tcpdump.

**Checkpoint:** Write a complete nftables ruleset that: allows SSH in, allows established/related traffic, drops everything else.

---

### [Lesson 27: nftables sets and verdict maps](lessons/lesson-27-nft-sets.html)

**Goal:** Write efficient multi-value rules without repeating yourself.

**Topics:**
- Anonymous sets: `tcp dport { 80, 443, 8080 } accept`
- Named sets: `nft add set inet filter blocked_ips { type ipv4_addr ; }`
- Interval sets for IP ranges: `type ipv4_addr ; flags interval ;`
- Verdict maps: `ip saddr vmap { 10.0.0.1 : accept, 10.0.0.2 : drop }`
- Updating named sets atomically without flushing rules

**Checkpoint:** Why is a named set faster than 1000 individual `ip saddr` rules?

---

### [Lesson 28: nftables flowtable (connection offload)](lessons/lesson-28-flowtable.html)

**Goal:** Bypass the full netfilter stack for established flows.

**Topics:**
- `nft add flowtable inet f { hook ingress priority 0 ; devices = { eth0, eth1 } ; }`
- Offloading established TCP/UDP to the fast path
- Performance tradeoffs: `conntrack -L` still works, some features unavailable

**Checkpoint:** What is the tradeoff of using a flowtable? What stops working?

---

## [Phase 9: Traffic Control — tc, qdisc, Filters](lessons/phase-09-tc.html)

### [Lesson 29: Traffic control model](lessons/lesson-29-tc-model.html)

**Goal:** Understand how the kernel queues and schedules outbound packets.

**Topics:**
- Egress vs ingress path (TC operates mostly on egress)
- What a qdisc is and where it sits in the packet path
- The default `pfifo_fast` qdisc
- `tc qdisc show dev eth0`
- Concepts: rate, burst, latency, backlog

**Checkpoint:** Draw where a qdisc sits relative to IP routing and the NIC driver.

---

### [Lesson 30: Classless qdiscs — shaping and emulation](lessons/lesson-30-classless-qdisc.html)

**Goal:** Rate-limit and emulate network conditions without classifying traffic.

**Topics:**
- `tbf` (Token Bucket Filter): hard rate limit
  `tc qdisc add dev eth0 root tbf rate 1mbit burst 32kbit latency 400ms`
- `fq_codel`: fair queuing + active queue management (Linux default on modern kernels)
- `netem`: emulate delay, loss, reorder, corruption (invaluable for testing)
  `tc qdisc add dev eth0 root netem delay 100ms 10ms loss 1%`

**Lab:** Add 100ms delay and 5% packet loss to a veth. Observe with ping RTT and tcpdump. Measure throughput with iperf3 before and after.

**Checkpoint:** What is the difference between a traffic shaper and a scheduler?

---

### [Lesson 31: Classful qdiscs — HTB](lessons/lesson-31-htb.html)

**Goal:** Shape and prioritize different traffic flows independently.

**Topics:**
- HTB (Hierarchical Token Bucket): the standard classful qdisc
- Class hierarchy: root class → child classes
- `tc qdisc add dev eth0 root handle 1: htb default 30`
- `tc class add dev eth0 parent 1: classid 1:1 htb rate 10mbit`
- `tc class add dev eth0 parent 1:1 classid 1:10 htb rate 5mbit ceil 10mbit`
- Borrowing: a class can borrow unused bandwidth from its parent up to `ceil`

**Lab:**

```
root 1: (10mbit total)
 ├── 1:10 (guaranteed 5mbit, ceil 10mbit) → HTTP
 └── 1:20 (guaranteed 2mbit, ceil 10mbit) → bulk
```

**Checkpoint:** What happens to class 1:20's throughput when class 1:10 is idle?

---

### [Lesson 32: tc filters](lessons/lesson-32-tc-filters.html)

**Goal:** Classify packets into HTB classes so the right qdisc handles them.

**Topics:**
- `u32` filter: match on raw IP header fields
  `tc filter add dev eth0 protocol ip parent 1: u32 match ip dport 80 0xffff flowid 1:10`
- `flower` filter: match on L2/L3/L4 fields (hardware-offloadable)
  `tc filter add dev eth0 protocol ip parent 1: flower ip_proto tcp dst_port 80 action goto chain 1`
- BPF filter: arbitrary classification logic
  `tc filter add dev eth0 parent 1: bpf obj classifier.o sec tc action goto chain 1`
- `tc filter show dev eth0`

**Lab:** Classify HTTP traffic (port 80) into 1:10 and everything else into 1:20. Verify with iperf3 on both ports.

**Checkpoint:** What is the key advantage of `flower` over `u32` for hardware NIC offload?

---

### [Lesson 33: Bandwidth measurement](lessons/lesson-33-bandwidth.html)

**Goal:** Measure real throughput in your lab topology.

**Topics:**
- `iperf3 -s` / `iperf3 -c <host>` — TCP throughput
- `iperf3 -u -b 100M` — UDP throughput
- `iperf3 -R` — reverse direction (server sends to client)
- `ip -s link show dev eth0` — interface RX/TX counters
- `ss -ti` — per-socket TCP stats including throughput, retransmits, RTT

**Lab:** Baseline throughput through a veth. Apply HTB shaping. Measure again. Verify limits are enforced.

**Checkpoint:** Why does `iperf3 -R` test a different bottleneck than the default direction?

---

## [Phase 10: Kernel Network Parameters — sysctl](lessons/phase-10-sysctl.html)

### [Lesson 34: Network sysctl tunables](lessons/lesson-34-sysctl-tunables.html)

**Goal:** Control kernel networking behavior at runtime without recompiling.

**Topics:**
- `sysctl -a | grep ^net` — enumerate all network knobs
- Forwarding: `net.ipv4.ip_forward=1`, `net.ipv6.conf.all.forwarding=1`
- Reverse path filter: `net.ipv4.conf.all.rp_filter` (0=off, 1=strict, 2=loose)
- Router advertisement acceptance: `net.ipv6.conf.eth0.accept_ra=1`
- TCP socket buffers: `net.core.rmem_max`, `net.core.wmem_max`
- TCP congestion control: `net.ipv4.tcp_congestion_control`
- Local port range: `net.ipv4.ip_local_port_range`
- ARP cache thresholds: `net.ipv4.neigh.default.gc_thresh1/2/3`
- Per-interface vs global settings (`conf/all` vs `conf/eth0`)

**Lab:** Enable `rp_filter=1` in a router namespace. Route a packet asymmetrically and watch it drop. Switch to `rp_filter=0` and watch it pass.

**Checkpoint:** What does `rp_filter=1` do? When would you set it to `0` or `2`?

---

### [Lesson 35: sysctl persistence](lessons/lesson-35-sysctl-persistence.html)

**Goal:** Make sysctl changes survive reboots.

**Topics:**
- `/etc/sysctl.conf` and `/etc/sysctl.d/*.conf`
- `sysctl -p /etc/sysctl.d/10-forwarding.conf`
- Namespace-scoped sysctl (some are per-netns, some are host-global)
- Which sysctls are namespaced: `net.ipv4.ip_forward` yes, `net.core.rmem_max` no

**Checkpoint:** Why doesn't setting `net.ipv4.ip_forward=1` inside a container affect the host?

---

## [Phase 11: eBPF & BPF Maps](lessons/phase-11-ebpf.html)

### [Lesson 36: eBPF fundamentals](lessons/lesson-36-ebpf-fundamentals.html)

**Goal:** Understand the programmable kernel dataplane.

**Topics:**
- What eBPF is: verified bytecode programs loaded into the kernel at runtime
- Program types: `XDP` (before skb), `TC` (after skb), socket filter, kprobe/tracepoint
- The verifier: why eBPF programs cannot crash the kernel (bounded loops, no null deref)
- `bpftool prog list`, `bpftool prog dump xlated id <N>`
- Build toolchain: `clang`, `libbpf`, BTF (BPF Type Format)

**Checkpoint:** What prevents an eBPF program from running an infinite loop?

---

### [Lesson 37: BPF maps](lessons/lesson-37-bpf-maps.html)

**Goal:** Share state between eBPF programs and userspace (and between programs).

**Topics:**
- Map types: `BPF_MAP_TYPE_HASH`, `ARRAY`, `LRU_HASH`, `PERCPU_HASH`, `RINGBUF`
- `bpftool map dump id <N>`
- Reading and writing maps from userspace (libbpf in C, or `bpftool`)
- Pinning maps to the BPF filesystem: `bpftool map pin id N /sys/fs/bpf/mymap`
- PERCPU maps: why per-CPU avoids spinlocks for packet counters

**Lab:** Write a minimal XDP program that counts packets per source IP using a `BPF_MAP_TYPE_HASH`. Read the counts from userspace with `bpftool map dump`.

**Checkpoint:** Why is `BPF_MAP_TYPE_PERCPU_HASH` faster than `BPF_MAP_TYPE_HASH` for packet counters?

---

### [Lesson 38: XDP — Express Data Path](lessons/lesson-38-xdp.html)

**Goal:** Process packets at the earliest possible kernel hook, before skb allocation.

**Topics:**
- XDP hook: runs in the driver's receive path, no skb yet
- Return codes: `XDP_DROP`, `XDP_PASS`, `XDP_TX` (hairpin), `XDP_REDIRECT`
- Native XDP (driver support) vs generic XDP (works everywhere, slower)
- Attaching: `ip link set dev eth0 xdp obj prog.o sec xdp`
- `bpftool net show`

**Lab:** Write an XDP program that drops packets from any IP in a BPF_MAP_TYPE_HASH blocklist. Add and remove IPs from userspace without reloading the program.

**Checkpoint:** What is the performance advantage of `XDP_DROP` over nftables `drop`?

---

## [Phase 12: nftables + BPF Integration](lessons/phase-12-nftbpf.html)

### [Lesson 39: nftbpf — calling BPF programs from nftables](lessons/lesson-39-nftbpf.html)

**Goal:** Combine nftables policy with BPF logic for cases neither can handle alone.

**Topics:**
- How nftables can invoke a pinned BPF program as a match expression
- `nft add rule inet filter input meta bpf object-pinned /sys/fs/bpf/myprog`
- The BPF program receives the `sk_buff` and returns a verdict
- Return values: continue (match), break (no match)
- Use case: complex string matching, ML-based classification, stateful logic in BPF with policy decisions in nftables

**Lab:** Write a BPF program that matches packets containing a specific byte pattern in the payload. Use it as a match in nftables to drop those packets.

**Checkpoint:** When would you use nftbpf instead of a nftables named set?

---

## [Phase 13: Configuration Persistence](lessons/phase-13-persistence.html)

### [Lesson 40: systemd-networkd](lessons/lesson-40-systemd-networkd.html)

**Goal:** Persist the entire network configuration from all previous phases in a maintainable, declarative format.

**Topics:**
- File types: `.link` (udev rename rules), `.netdev` (create virtual device), `.network` (configure device)
- `/etc/systemd/network/` — files processed in lexicographic order
- Configuring in `.network`: `[Address]`, `[Route]`, `[RoutingPolicyRule]`
- Configuring in `.netdev`: bridge, vlan, bond, vxlan, dummy, veth
- `networkctl status`, `networkctl list`
- `systemd-networkd-wait-online` for boot ordering

**Lab:** Recreate the bridge + VLAN + two veth namespace topology from Phase 3 using `.netdev` and `.network` files. Verify it comes back correctly after `systemctl restart systemd-networkd`.

**Checkpoint:** What is the difference between a `.netdev` and a `.network` file?

---

### [Lesson 41: Other persistence approaches](lessons/lesson-41-persistence-alternatives.html)

**Goal:** Know what else exists and when to use it.

**Topics:**
- `netplan` (Ubuntu): YAML abstraction that renders to systemd-networkd or NetworkManager
- `/etc/network/interfaces` (Debian): the legacy approach
- NetworkManager (`nmcli`, `nmtui`): desktop-oriented, good for dynamic environments
- nftables persistence: `nft -f /etc/nftables.conf`, `systemctl enable nftables`
- sysctl persistence: `/etc/sysctl.d/`
- Persisting tc rules with `tc-cache` or a systemd service unit

**Checkpoint:** Why would you prefer systemd-networkd over NetworkManager on a headless server?

---

## [Phase 14: Container & Cloud Networking](lessons/phase-14-container-cloud.html)

### [Lesson 42: Docker networking from first principles](lessons/lesson-42-docker-networking.html)

**Goal:** Build Docker's default bridge network manually without using Docker.

**Build manually:**

```
namespace (container)
       |
      veth1 (inside container, renamed eth0)
       |
      veth0 (host end, attached to bridge)
       |
    docker0 (bridge, 172.17.0.1)
       |
  iptables/nft MASQUERADE (SNAT for outbound)
       |
    host eth0 (physical NIC)
```

Then inspect a real Docker container: `docker network inspect bridge` and compare.

**Checkpoint:** Reconstruct Docker's default bridge network from scratch. Verify a process in the namespace can reach the internet.

---

### [Lesson 43: Kubernetes networking](lessons/lesson-43-kubernetes-networking.html)

**Topics:**
- Pod networking model: every pod gets its own IP, no NAT between pods
- Pod namespaces: one network namespace per pod, shared by all containers in the pod
- CNI (Container Network Interface): the plugin interface called by kubelet on pod create/delete
- What a CNI plugin does: creates veth, assigns IP, sets up routes
- kube-proxy: how ClusterIP services are implemented (iptables or IPVS modes)
- eBPF dataplane: Cilium replaces kube-proxy with BPF maps

**Checkpoint:** Trace a packet from Pod A on Node 1 to Pod B on Node 2. Name every hop and transformation.

---

### [Lesson 44: VXLAN-based CNI (Flannel)](lessons/lesson-44-flannel-vxlan.html)

**Goal:** See VXLAN from Phase 4 operating in a real Kubernetes context.

**Topics:**
- Flannel's VXLAN mode: one VTEP per node
- How Flannel populates FDB entries (watches kube-apiserver for node IP/MAC)
- `bridge fdb show dev flannel.1`
- The pod CIDR allocation per node

**Checkpoint:** What does Flannel's VXLAN FDB look like? Who populates it and how does it stay in sync?

---

## [Phase 15: Troubleshooting & Diagnostics](lessons/phase-15-troubleshooting.html)

### [Lesson 45: tcpdump mastery](lessons/lesson-45-tcpdump-mastery.html)

**Goal:** Read network traffic at expert level.

**Topics:**
- Complex filter expressions: `tcp[tcpflags] & tcp-syn != 0`, `not port 22`
- `-X` flag: hex + ASCII payload dump
- `-G 60 -W 5`: rotating capture files
- `tshark -r file.pcap -T fields -e ip.src -e tcp.dstport` for scripted analysis
- Loading a `.pcap` in Wireshark for visual stream following
- Comparing captures at the sender vs receiver to localize drops

**Lab:** Debug a broken NAT setup using only tcpdump. No guessing allowed — every conclusion must come from a packet.

**Checkpoint:** Write a tcpdump filter that captures only TCP SYN packets destined for port 443.

---

### [Lesson 46: Full diagnostic toolchain](lessons/lesson-46-diagnostic-toolchain.html)

**Goal:** Develop a systematic process for diagnosing any networking failure.

**Tools and what each answers:**

| Tool | Question |
|------|----------|
| `ip link show` | Is the interface up? Does it have carrier? |
| `ip addr show` | Is an IP address assigned? |
| `ip neigh show` | Is ARP resolving to the right MAC? |
| `ip route get <dst>` | Which route does the kernel select? |
| `bridge fdb show` | Is L2 forwarding learning the MAC? |
| `nft list ruleset` | Is a firewall rule dropping the packet? |
| `conntrack -L` | Is the connection being tracked? Is state valid? |
| `ss -tulpn` | Is the service actually listening? |
| `tcpdump` | Where does the packet stop appearing? |
| `bpftool prog list` | Is an XDP or TC BPF program intercepting packets? |
| `sysctl net.ipv4.ip_forward` | Is forwarding enabled on the router? |

**Lab:** A broken namespace topology is given. Find and fix the fault using only the tools above.

**Checkpoint:** Diagnose the broken topology. Name every check you ran and what it told you.

---

### [Lesson 47: Network debugging methodology](lessons/lesson-47-debugging-methodology.html)

**Goal:** Never guess. Work layer by layer.

**Questions to ask in order:**

1. **Is the interface up?** `ip link show` — check `state UP` and `LOWER_UP`
2. **Is an IP assigned?** `ip addr show` — correct address, correct prefix?
3. **Is L2 reachable?** `arping` or `tcpdump` for ARP — is a reply coming back?
4. **Is L3 reachable?** `ping` — do ICMP echo replies come back?
5. **Is the route correct?** `ip route get <dst>` — right gateway, right interface?
6. **Is routing symmetric?** Check the return path too — `ip route get` from the other side
7. **Is NAT correct?** `conntrack -L` — is the translation entry there? `tcpdump` on both sides of the NAT
8. **Is the firewall dropping it?** `nft list ruleset` — look for DROP/REJECT rules. Add logging temporarily
9. **Is an eBPF program intercepting?** `ip link show` for xdp, `bpftool net show`
10. **Is forwarding enabled?** `sysctl net.ipv4.ip_forward`

**Checkpoint:** Given a fresh broken lab, diagnose it from scratch using only this methodology. Time yourself.

---

## [Phase 16: Tunnels & VPNs](lessons/phase-16-tunnels-vpns.html)

### [Lesson 48: Tunneling fundamentals — GRE, IPIP, SIT, FOU](lessons/lesson-48-tunneling-fundamentals.html)

**Goal:** Understand encapsulation without encryption — the substrate every VPN sits on.

**Topics:**
- Outer vs inner packet; MTU and the overhead problem
- `ip tunnel` / `ip link add type gre|ipip|sit`; point-to-point vs multipoint (mGRE)
- FOU/GUE: tunneling inside UDP and why UDP encapsulation matters
- When you'd use a plain tunnel vs an overlay (VXLAN) vs a VPN

**Lab:** GRE tunnel between two namespaces; tcpdump the outer + inner headers.

**Checkpoint:** Why does a tunnel lower the effective MTU, and what breaks if you ignore it?

---

### [Lesson 49: VPN cryptography building blocks](lessons/lesson-49-vpn-crypto.html)

**Goal:** Acquire the crypto vocabulary every VPN protocol assumes.

**Topics:**
- Symmetric vs asymmetric crypto; Diffie-Hellman key exchange
- AEAD ciphers (ChaCha20-Poly1305, AES-GCM); what "authenticated" buys you
- PKI/certificates vs pre-shared keys; the CIA triad (confidentiality/integrity/authenticity)
- Threat model: what a VPN does and does *not* protect against

**Checkpoint:** Why is encryption without authentication dangerous? What attack does it allow?

**Cross-reference:** This is a deliberately thin, WireGuard-focused primer so the VPN phase stands on its own. The full treatment of cryptography lives in the **Security & Identity** track, Phase 1 (Cryptography Foundations).

---

### [Lesson 50: IPsec and the kernel xfrm framework](lessons/lesson-50-ipsec-xfrm.html)

**Goal:** Learn the "enterprise standard" VPN and the kernel machinery behind it.

**Topics:**
- ESP vs AH; transport mode vs tunnel mode
- Security Associations and the SPD; the `xfrm` subsystem
- IKEv2 key negotiation; strongSwan as the userspace daemon
- Why IPsec is powerful but operationally heavy

**Lab:** Site-to-site ESP tunnel in tunnel mode between two namespaces; verify with `ip xfrm state`.

**Checkpoint:** What's the difference between transport and tunnel mode, and when do you use each?

---

### [Lesson 51: WireGuard fundamentals](lessons/lesson-51-wireguard-fundamentals.html)

**Goal:** Master the modern, minimal VPN protocol.

**Topics:**
- Key pairs and **crypto-key routing** (the AllowedIPs model)
- The Noise-based handshake; UDP-only, stateless-by-design philosophy
- `wg`, `wg-quick`, config files; roaming endpoints
- Why WireGuard deliberately drops IPsec's flexibility

**Lab:** Two-peer WireGuard tunnel; inspect with `wg show`; capture the encrypted UDP.

**Checkpoint:** How does WireGuard decide which peer a given destination IP belongs to?

---

### [Lesson 52: WireGuard internals & userspace datapaths](lessons/lesson-52-wireguard-internals.html)

**Goal:** See how the same protocol runs in-kernel vs in userspace.

**Topics:**
- Kernel module vs a userspace implementation over a TUN device (Lesson 14 payoff)
- Userspace TCP/IP stacks (e.g. `netstack`) and when you'd embed one
- The handshake/cookie anti-DoS mechanism; key rotation
- Performance levers: UDP batching (GSO/GRO), offloads

**Checkpoint:** What are the trade-offs of a userspace WireGuard datapath vs the kernel module?

---

### [Lesson 53: NAT traversal — STUN, ICE, UDP hole punching](lessons/lesson-53-nat-traversal.html)

**Goal:** Solve the hardest problem in peer-to-peer VPNs — connecting two boxes both behind NAT.

**Topics:**
- NAT behavior classes (full-cone → address/port-restricted → symmetric)
- STUN for discovering your public mapping; ICE-style candidate gathering
- UDP hole punching; port prediction and the birthday-paradox trick for symmetric NAT
- Why this problem is adversarial and underdocumented in the wild

**Lab:** Simulate two NATs with namespaces + masquerade; demonstrate a successful hole punch.

**Checkpoint:** Why can two symmetric-NAT peers fail to hole-punch, and what's the fallback?

---

### [Lesson 54: Relays and fallback paths](lessons/lesson-54-relays.html)

**Goal:** Guarantee connectivity when a direct path is impossible.

**Topics:**
- Relay servers: forwarding encrypted traffic when hole punching fails
- Relaying over HTTPS/443 to survive restrictive firewalls
- Latency/cost trade-offs; upgrading from relayed to direct mid-session
- End-to-end encryption so the relay can't read traffic

**Checkpoint:** Why must the relay never be able to decrypt the traffic it forwards?

---

### [Lesson 55: Mesh VPNs and coordination/control planes](lessons/lesson-55-mesh-vpns.html)

**Goal:** Move from point-to-point tunnels to a self-configuring mesh.

**Topics:**
- Full mesh vs hub-and-spoke; the scaling problem of N² tunnels
- A coordination/control server: key distribution, peer discovery, node registry
- Policy-based access control (ACLs) and the zero-trust model
- Identity (SSO/OIDC), split-DNS for the overlay, subnet routers & exit nodes
- Open-source mesh orchestrators to study (Nebula, innernet, Headscale)

**Lab:** Stand up a small WireGuard mesh driven by a coordination tool; add/remove a node.

**Checkpoint:** What does the control plane handle vs the data plane in a mesh VPN?

---

### [Lesson 56: TLS-based VPNs (OpenVPN) — the contrast](lessons/lesson-56-tls-vpns.html)

**Goal:** Round out the protocol landscape and understand the trade-offs.

**Topics:**
- TLS/DTLS tunnels; userspace TUN; cert-based auth
- TCP-mode vs UDP-mode and the "TCP-over-TCP meltdown"
- Where TLS VPNs still win (firewall traversal, mature ecosystem)

**Checkpoint:** Why is running a VPN over TCP usually worse than over UDP?

---

### [Lesson 57: Capstone — an encrypted mesh across NAT](lessons/lesson-57-vpn-capstone.html)

**Goal:** Integrate the whole phase.

**Lab:** Three nodes, two behind simulated NAT. They hole-punch into direct WireGuard tunnels; when one NAT is made symmetric, traffic falls back to a relay. Diagnose with `wg show` + tcpdump.

**Checkpoint:** Trace a packet from node A to node B in both the direct and relayed cases. Name every transformation.

---

## [Phase 17: Advanced Routing & Data-Center Fabrics](lessons/phase-17-advanced-routing.html)

### [Lesson 58: Large-scale BGP — route reflectors, communities, policy](lessons/lesson-58-bgp-scale.html)

**Goal:** Run BGP at data-center/provider scale.

**Topics:**
- The iBGP full-mesh scaling problem → route reflectors
- BGP communities for tagging and policy
- route-maps / prefix-lists; ECMP; confederations (briefly)

**Lab:** Multi-router BGP with a route reflector in FRR.

**Checkpoint:** Why does a route reflector exist, and what scaling problem does it solve?

---

### [Lesson 59: BGP EVPN & VXLAN fabrics](lessons/lesson-59-evpn.html)

**Goal:** Replace flood-and-learn overlays with a BGP control plane.

**Topics:**
- Control-plane MAC learning vs the flood-and-learn VXLAN of Lesson 15
- EVPN route types (MAC/IP, IMET); VTEP auto-discovery
- The leaf-spine fabric model; integration with FRR's `bgpd`

**Checkpoint:** How does EVPN replace VXLAN's data-plane MAC flooding, and why is that better at scale?

---

### [Lesson 60: Segment Routing & SRv6](lessons/lesson-60-srv6.html)

**Goal:** Push path decisions to the source and remove state from the core.

**Topics:**
- Source routing reborn; SR-MPLS vs SRv6
- The segment list / SID
- `seg6` in the Linux kernel (`ip -6 route ... encap seg6`)
- Traffic-engineering use cases

**Checkpoint:** How does segment routing move path decisions to the source, and what state does it remove from the core?

---

### [Lesson 61: Anycast](lessons/lesson-61-anycast.html)

**Goal:** Serve one address from many locations.

**Topics:**
- One address, many origins; BGP/IGP-based anycast
- How DNS roots and CDNs use it
- Health-checked withdrawal; failure semantics (no per-flow guarantees)

**Checkpoint:** Why is anycast great for stateless services but risky for long-lived stateful connections?

---

## [Phase 18: High-Performance Networking](lessons/phase-18-performance.html)

### [Lesson 62: TCP internals — congestion control & flow dynamics](lessons/lesson-62-tcp-internals.html)

**Goal:** Understand what governs a TCP connection's throughput.

**Topics:**
- Sliding window & window scaling
- Slow start / congestion avoidance; CUBIC vs **BBR**
- Bufferbloat; pacing
- Reading `ss -ti` (cwnd, RTT, retransmits)

**Checkpoint:** What problem does BBR solve that loss-based congestion control (CUBIC) does not?

---

### [Lesson 63: NIC offloads & multiqueue scaling](lessons/lesson-63-nic-offloads.html)

**Goal:** Scale packet processing across CPU cores and the NIC.

**Topics:**
- GRO/GSO/TSO/checksum offloads (`ethtool -k`)
- RSS / RPS / XPS
- IRQ affinity & NAPI; ring buffers
- Why a single CPU caps throughput

**Checkpoint:** How do RSS and RPS spread packet processing across cores, and how do they differ?

---

### [Lesson 64: Kernel bypass — AF_XDP & DPDK](lessons/lesson-64-kernel-bypass.html)

**Goal:** Move packets without the kernel network stack.

**Topics:**
- Why bypass the stack; AF_XDP sockets and zero-copy
- XDP redirect into userspace (ties to Phase 11)
- DPDK poll-mode drivers and hugepages
- Trade-offs vs in-kernel processing

**Checkpoint:** What does kernel bypass buy you, and what do you give up?

---

### [Lesson 65: Multipath TCP (MPTCP)](lessons/lesson-65-mptcp.html)

**Goal:** Run one TCP connection across multiple paths.

**Topics:**
- Subflows; the Linux MPTCP implementation (`mptcpize`, `ip mptcp`)
- Failover vs aggregation
- Relevance to mobility/roaming

**Checkpoint:** How does MPTCP keep a connection alive when one underlying path disappears?

---

## [Phase 19: Network Services — DNS, Load Balancing & HA](lessons/phase-19-services.html)

### [Lesson 66: DNS deep dive](lessons/lesson-66-dns.html)

**Goal:** Understand name resolution end to end.

**Topics:**
- Recursive vs authoritative; the resolution walk
- `systemd-resolved`; `/etc/resolv.conf` & nsswitch
- Split DNS / per-domain routing; DoT/DoH; caching & TTLs (extends Lesson 24)

**Checkpoint:** Trace a full recursive resolution from stub resolver to authoritative answer.

---

### [Lesson 67: Layer-4 load balancing — IPVS/LVS](lessons/lesson-67-ipvs.html)

**Goal:** Distribute connections across backends in the kernel.

**Topics:**
- `ipvsadm`; scheduling algorithms (rr, wrr, lc)
- DR vs NAT vs tunnel modes
- Connection persistence; why kube-proxy IPVS mode uses this (ties to Lesson 43)

**Checkpoint:** Why does Direct Routing mode scale better than NAT mode for return traffic?

---

### [Lesson 68: High availability — VRRP & keepalived](lessons/lesson-68-vrrp-keepalived.html)

**Goal:** Make a service IP survive a node failure.

**Topics:**
- Virtual IP failover; VRRP master/backup election
- `keepalived` config; health checks
- Gratuitous ARP on failover (callback to Lesson 5); split-brain risks

**Checkpoint:** What does VRRP send on failover so traffic redirects to the new master, and why?

---

## [Phase 20: Modern Transport & Observability](lessons/phase-20-modern-transport.html)

### [Lesson 69: QUIC & HTTP/3](lessons/lesson-69-quic.html)

**Goal:** Understand the transport replacing TCP for the web.

**Topics:**
- Connection-oriented over UDP (callback to Lesson 23)
- Built-in TLS 1.3; stream multiplexing without head-of-line blocking
- Connection migration; why it's hard to inspect

**Checkpoint:** Why does QUIC run over UDP instead of being a new protocol next to TCP?

---

### [Lesson 70: TLS at the packet level](lessons/lesson-70-tls.html)

**Goal:** Read and reason about a TLS connection on the wire.

**Topics:**
- Handshake walkthrough; SNI (and ESNI/ECH)
- What's encrypted vs visible on the wire
- Cert validation; reading a TLS handshake in tcpdump/tshark

**Checkpoint:** What can a passive observer still learn about a TLS connection despite encryption?

**Cross-reference:** This lesson stays at the wire/observability level (what's on the packet, SNI visibility). The TLS *protocol* internals — handshake mechanics, cipher negotiation, key exchange — are covered in depth in the **Security & Identity** track, Phase 3 (TLS & SSL).

---

### [Lesson 71: Multicast & IGMP](lessons/lesson-71-multicast.html)

**Goal:** Deliver one stream to many receivers efficiently.

**Topics:**
- Group addressing; IGMP join/leave
- Multicast routing basics; snooping on bridges
- When multicast is used (and why it's rare on the public internet)

**Checkpoint:** How does a switch know which ports to forward a multicast stream to?

---

### [Lesson 72: Time synchronization — NTP & PTP](lessons/lesson-72-time-sync.html)

**Goal:** Keep clocks aligned and know when precision matters.

**Topics:**
- Why clocks drift and why it matters (logs, TLS, distributed systems)
- NTP hierarchy/strata; `chrony`
- PTP (hardware timestamping) for sub-microsecond sync

**Checkpoint:** When is NTP insufficient and PTP required?

---

### [Lesson 73: Network observability & telemetry](lessons/lesson-73-observability.html)

**Goal:** Build a systematic view of a running network.

**Topics:**
- `/proc/net` and `ss` deep dive
- Flow telemetry (NetFlow/IPFIX/sFlow)
- Interface & qdisc stats; exporting metrics (node_exporter)
- Correlating drops across the stack

**Checkpoint:** Given "intermittent slowness," which counters do you check first and why?

---

## Recommended Learning Order

Do not skip ahead. Each step depends on the previous ones.

1. Namespaces (`ip netns`)
2. `ip link`, `ip addr` — address management
3. `ip neigh`, ARP — neighbor resolution
4. `tcpdump` — start using it now, never stop
5. veth pairs
6. Loopback
7. Bridges
8. VLAN interfaces
9. MACVLAN, TAP
10. VXLAN
11. `ip route` — routing fundamentals
12. Routing between namespaces (enable `ip_forward`)
13. Policy routing, multiple route tables
14. FRR — OSPF
15. FRR — BGP
16. NAT with nftables
17. Conntrack
18. DHCP
19. nftables — filtering and chains
20. nftables — sets and verdict maps
21. nftables flowtable
22. `sysctl` tunables
23. Traffic control — qdisc model
24. HTB + tc filters
25. Bandwidth measurement with iperf3
26. eBPF fundamentals
27. BPF maps
28. XDP programs
29. nftbpf integration
30. systemd-networkd persistence
31. Docker networking reconstruction
32. Kubernetes networking + CNI
33. Full troubleshooting methodology
34. Tunneling fundamentals (GRE, IPIP, FOU)
35. VPN cryptography building blocks
36. IPsec & the xfrm framework
37. WireGuard — fundamentals and internals
38. NAT traversal and relays
39. Mesh VPNs & coordination planes
40. Advanced BGP — route reflectors, EVPN
41. Segment routing (SRv6) and anycast
42. TCP internals & high-performance tuning (offloads, scaling)
43. Kernel bypass (AF_XDP/DPDK) and MPTCP
44. DNS deep dive, load balancing (IPVS), HA (VRRP)
45. QUIC/HTTP3, TLS, multicast, time sync, observability
