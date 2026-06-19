---
title: Learning Plan
nav_order: 2
nav_exclude: false
---

# Linux Networking Learning Plan

---

## Phase 1: Foundations — Namespaces & Basic IP Tools

### Lesson 1: What a network namespace is

**Goal:** Understand that a namespace is an independent networking universe.

**Topics:**
- What is isolated: interfaces, routes, ARP/neighbor tables, firewall rules, sockets
- What is shared with the host
- Why containers use namespaces

**Checkpoint:** Predict what happens if a process moves into another namespace.

---

### Lesson 2: The "host is just another namespace" model

**Goal:** Stop thinking of the host as special.

**Topics:**
- Initial namespace
- Additional namespaces
- Packet visibility across namespaces
- Why interfaces must connect namespaces

**Checkpoint:** Explain why a container cannot automatically see the host's interfaces.

---

### Lesson 3: Build two isolated namespaces

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

### Lesson 4: IP addressing fundamentals (addrs)

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

### Lesson 5: Neighbor tables & ARP (neigh)

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

### Lesson 6: tcpdump — your constant companion

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

## Phase 2: Virtual Interfaces

### Lesson 7: veth pairs

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

### Lesson 8: Loopback interface

**Goal:** Understand why every namespace needs its own loopback.

**Topics:**
- What `lo` is and why applications bind to it
- Why new namespaces start with lo DOWN
- `ip link set lo up` inside a namespace
- Why 127.0.0.1 is not routed to the physical network

**Checkpoint:** Why does 127.0.0.1 work even when disconnected from all networks?

---

### Lesson 9: Bond interfaces

**Goal:** Combine multiple links for redundancy or throughput.

**Topics:**
- `ip link add bond0 type bond`
- Bond modes: `active-backup` (failover), `balance-rr` (round-robin), `802.3ad` (LACP)
- Adding slave interfaces: `ip link set eth1 master bond0`
- Monitoring slave state

**Checkpoint:** When would you choose `active-backup` over `802.3ad` LACP?

---

### Lesson 10: Dummy interfaces

**Goal:** Create a virtual IP endpoint that exists only in software.

**Topics:**
- `ip link add dummy0 type dummy`
- Use cases: stable loopback IPs for routing, testing, Kubernetes node IPs
- Difference from loopback

**Checkpoint:** Why create an interface that goes nowhere?

---

## Phase 3: Layer-2 Networking

### Lesson 11: Linux bridges

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

### Lesson 12: VLAN interfaces

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

### Lesson 13: MACVLAN

**Goal:** Give a namespace its own MAC address and presence on the physical LAN.

**Topics:**
- `ip link add macvlan0 link eth0 type macvlan mode bridge`
- MACVLAN modes: `bridge`, `private`, `vepa`, `passthru`
- Host ↔ MACVLAN communication limitation (why they cannot talk to each other)
- MACVTAP: MACVLAN + TAP for VMs

**Checkpoint:** Why does a MACVLAN container appear as a separate machine on the LAN?

---

### Lesson 14: TAP interfaces

**Goal:** Understand interfaces that hand Ethernet frames to userspace (used by VMs).

**Topics:**
- TUN (L3, IP packets) vs TAP (L2, Ethernet frames)
- `ip tuntap add dev tap0 mode tap`
- Why QEMU/KVM attaches VM NICs to TAP interfaces
- TAP vs veth: who consumes the packets

**Checkpoint:** Why would a VM use TAP instead of a veth?

---

## Phase 4: Overlay Networks

### Lesson 15: VXLAN

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

## Phase 5: Layer-3 — Routing

### Lesson 16: Routing fundamentals (routes)

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

### Lesson 17: Routing between namespaces

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

### Lesson 18: Policy routing & multiple routing tables

**Goal:** Route based on source address or firewall mark, not just destination.

**Topics:**
- `ip rule add from 192.168.1.0/24 table 100 priority 100`
- Route tables: `main` (253), `local` (255), `default` (253)
- Marking packets with `nft` and routing by mark: `ip rule add fwmark 0x1 table 200`
- Use cases: multi-homed hosts, VPN split routing

**Checkpoint:** Why would you want traffic from one source IP to take a different path than another source IP going to the same destination?

---

## Phase 6: Dynamic Routing — FRR

### Lesson 19: FRR introduction

**Goal:** Replace static routes with a routing daemon that adapts to topology changes.

**Topics:**
- What FRR is: Free Range Routing, the open-source router suite (successor to Quagga)
- Daemons: `zebra` (RIB manager), `ospfd`, `bgpd`, `staticd`
- `vtysh`: the Cisco-style CLI
- How FRR installs routes into the kernel RIB via zebra
- Installing and starting FRR

**Checkpoint:** What problem does a dynamic routing protocol solve that static routes cannot?

---

### Lesson 20: OSPF with FRR

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

### Lesson 21: BGP with FRR

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

## Phase 7: NAT, Conntrack & DHCP

### Lesson 22: NAT with nftables

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

### Lesson 23: Conntrack

**Goal:** Understand the state machine that makes NAT and stateful firewalling possible.

**Topics:**
- `conntrack -L` (list connections), `conntrack -E` (event stream)
- Connection states: `NEW`, `ESTABLISHED`, `RELATED`, `INVALID`
- How conntrack ties reply packets back to the original flow
- Conntrack table limits: `nf_conntrack_max`

**Lab:** Watch a TCP connection lifecycle in conntrack from SYN to FIN. Run `conntrack -E` while pinging and making HTTP requests.

**Checkpoint:** What state does conntrack assign to the reply packet of an outbound TCP SYN?

---

### Lesson 24: DHCP

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

## Phase 8: Firewalling — nftables

### Lesson 25: nftables architecture

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

### Lesson 26: Packet filtering with nft

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

### Lesson 27: nftables sets and verdict maps

**Goal:** Write efficient multi-value rules without repeating yourself.

**Topics:**
- Anonymous sets: `tcp dport { 80, 443, 8080 } accept`
- Named sets: `nft add set inet filter blocked_ips { type ipv4_addr ; }`
- Interval sets for IP ranges: `type ipv4_addr ; flags interval ;`
- Verdict maps: `ip saddr vmap { 10.0.0.1 : accept, 10.0.0.2 : drop }`
- Updating named sets atomically without flushing rules

**Checkpoint:** Why is a named set faster than 1000 individual `ip saddr` rules?

---

### Lesson 28: nftables flowtable (connection offload)

**Goal:** Bypass the full netfilter stack for established flows.

**Topics:**
- `nft add flowtable inet f { hook ingress priority 0 ; devices = { eth0, eth1 } ; }`
- Offloading established TCP/UDP to the fast path
- Performance tradeoffs: `conntrack -L` still works, some features unavailable

**Checkpoint:** What is the tradeoff of using a flowtable? What stops working?

---

## Phase 9: Traffic Control — tc, qdisc, Filters

### Lesson 29: Traffic control model

**Goal:** Understand how the kernel queues and schedules outbound packets.

**Topics:**
- Egress vs ingress path (TC operates mostly on egress)
- What a qdisc is and where it sits in the packet path
- The default `pfifo_fast` qdisc
- `tc qdisc show dev eth0`
- Concepts: rate, burst, latency, backlog

**Checkpoint:** Draw where a qdisc sits relative to IP routing and the NIC driver.

---

### Lesson 30: Classless qdiscs — shaping and emulation

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

### Lesson 31: Classful qdiscs — HTB

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

### Lesson 32: tc filters

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

### Lesson 33: Bandwidth measurement

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

## Phase 10: Kernel Network Parameters — sysctl

### Lesson 34: Network sysctl tunables

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

### Lesson 35: sysctl persistence

**Goal:** Make sysctl changes survive reboots.

**Topics:**
- `/etc/sysctl.conf` and `/etc/sysctl.d/*.conf`
- `sysctl -p /etc/sysctl.d/10-forwarding.conf`
- Namespace-scoped sysctl (some are per-netns, some are host-global)
- Which sysctls are namespaced: `net.ipv4.ip_forward` yes, `net.core.rmem_max` no

**Checkpoint:** Why doesn't setting `net.ipv4.ip_forward=1` inside a container affect the host?

---

## Phase 11: eBPF & BPF Maps

### Lesson 36: eBPF fundamentals

**Goal:** Understand the programmable kernel dataplane.

**Topics:**
- What eBPF is: verified bytecode programs loaded into the kernel at runtime
- Program types: `XDP` (before skb), `TC` (after skb), socket filter, kprobe/tracepoint
- The verifier: why eBPF programs cannot crash the kernel (bounded loops, no null deref)
- `bpftool prog list`, `bpftool prog dump xlated id <N>`
- Build toolchain: `clang`, `libbpf`, BTF (BPF Type Format)

**Checkpoint:** What prevents an eBPF program from running an infinite loop?

---

### Lesson 37: BPF maps

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

### Lesson 38: XDP — Express Data Path

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

## Phase 12: nftables + BPF Integration

### Lesson 39: nftbpf — calling BPF programs from nftables

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

## Phase 13: Configuration Persistence

### Lesson 40: systemd-networkd

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

### Lesson 41: Other persistence approaches

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

## Phase 14: Container & Cloud Networking

### Lesson 42: Docker networking from first principles

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

### Lesson 43: Kubernetes networking

**Topics:**
- Pod networking model: every pod gets its own IP, no NAT between pods
- Pod namespaces: one network namespace per pod, shared by all containers in the pod
- CNI (Container Network Interface): the plugin interface called by kubelet on pod create/delete
- What a CNI plugin does: creates veth, assigns IP, sets up routes
- kube-proxy: how ClusterIP services are implemented (iptables or IPVS modes)
- eBPF dataplane: Cilium replaces kube-proxy with BPF maps

**Checkpoint:** Trace a packet from Pod A on Node 1 to Pod B on Node 2. Name every hop and transformation.

---

### Lesson 44: VXLAN-based CNI (Flannel)

**Goal:** See VXLAN from Phase 4 operating in a real Kubernetes context.

**Topics:**
- Flannel's VXLAN mode: one VTEP per node
- How Flannel populates FDB entries (watches kube-apiserver for node IP/MAC)
- `bridge fdb show dev flannel.1`
- The pod CIDR allocation per node

**Checkpoint:** What does Flannel's VXLAN FDB look like? Who populates it and how does it stay in sync?

---

## Phase 15: Troubleshooting & Diagnostics

### Lesson 45: tcpdump mastery

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

### Lesson 46: Full diagnostic toolchain

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

### Lesson 47: Network debugging methodology

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
