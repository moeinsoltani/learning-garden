---
title: "Lesson 68 — High Availability (VRRP & keepalived)"
nav_order: 68
parent: "Phase 19: Network Services"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 68: High Availability — VRRP & keepalived

## Concept

A default gateway or service IP is a single point of failure: if that box dies, everything pointing
at it breaks. **VRRP** (Virtual Router Redundancy Protocol) makes a **virtual IP (VIP)** float
between two or more routers, so if the active one fails, a backup takes the VIP over — clients keep
using the same address, unaware anything happened.

```
   Normal:    clients ─► VIP 10.0.0.1  (held by MASTER = router A)
                                        router B = BACKUP, idle, watching

   A fails:   clients ─► VIP 10.0.0.1  (now held by router B, promoted to MASTER)
```

`keepalived` is the standard Linux daemon that implements VRRP (and ties in health checks).

---

## How it works

**Election & advertisements.** VRRP routers in a group share a **virtual router ID (VRID)** and a
**priority**. The highest-priority router becomes **MASTER** and owns the VIP; it sends periodic
**VRRP advertisements** (multicast) saying "I'm alive and master." Backups listen; if advertisements
stop arriving for a few intervals, the highest-priority backup concludes the master is dead, promotes
itself, and claims the VIP.

**Claiming the VIP = gratuitous ARP.** When a backup takes over, it must redirect L2 traffic to its
own MAC. It sends a **gratuitous ARP** (Lesson 5): an unsolicited ARP announcing "VIP 10.0.0.1 is now
at *my* MAC." Switches update their forwarding tables and clients update their ARP caches, so frames
for the VIP now go to the new master. Without this, traffic would keep going to the dead box's MAC
until ARP entries aged out.

**Health checks (keepalived's value-add).** Plain VRRP only detects whether the *router* is up. But
the router can be alive while the *service* behind it is broken. keepalived adds **health checks**
(`vrrp_script`): probe the service (TCP connect, HTTP, custom script), and on failure **lower this
node's priority** so a healthy backup wins the election. This makes failover service-aware, not just
host-aware. keepalived also integrates with IPVS (Lesson 67) to health-check load-balancer backends.

{: .note }
> **Split-brain**
> If the two nodes can't hear each other's advertisements (e.g. the link between them fails, but both
> are otherwise up), <em>both</em> may declare themselves master and claim the VIP — "split-brain,"
> causing duplicate IPs and chaos. Mitigations: redundant communication paths, a tie-breaker/quorum,
> and preferring `nopreempt` so a recovered node doesn't flap the VIP back and forth.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `keepalived -f <conf> -n` | Run keepalived in the foreground with a config |
| `ip addr show` | See which node currently holds the VIP |
| `tcpdump -ni <if> vrrp` | Watch VRRP advertisements (protocol 112) |
| `arping -U -I <if> <vip>` | Send a gratuitous ARP manually (what failover does) |

---

## Lab

Two routers share a VIP via keepalived; kill the master and watch the backup take over with a
gratuitous ARP.

### Step 1 — Two routers and a client on one segment (a bridge, Lesson 11)

```bash
$ sudo ip netns add r1; sudo ip netns add r2; sudo ip netns add cli
# attach all three to a shared bridge 'br0'; r1=10.0.0.2, r2=10.0.0.3, cli=10.0.0.10; VIP=10.0.0.1
```

### Step 2 — keepalived config on r1 (MASTER, priority 150)

```bash
$ cat > /tmp/r1.conf <<'EOF'
vrrp_instance V1 {
    state MASTER
    interface r1-br
    virtual_router_id 51
    priority 150
    advert_int 1
    virtual_ipaddress { 10.0.0.1/24 }
}
EOF
$ sudo ip netns exec r1 keepalived -f /tmp/r1.conf -n &
```

### Step 3 — keepalived config on r2 (BACKUP, priority 100), then verify

```bash
# r2.conf identical but: state BACKUP, interface r2-br, priority 100
$ sudo ip netns exec r2 keepalived -f /tmp/r2.conf -n &
$ sudo ip netns exec r1 ip addr show r1-br | grep 10.0.0.1   # r1 holds the VIP
$ sudo ip netns exec cli ping -c2 10.0.0.1                   # reachable via r1
```

### Step 4 — Kill the master and watch failover

```bash
$ sudo ip netns exec cli tcpdump -ni cli-br arp &      # watch for the gratuitous ARP
$ sudo pkill -f r1.conf                                 # master "dies"
# Within a couple of advert intervals: r2 logs MASTER, sends a gratuitous ARP for 10.0.0.1,
$ sudo ip netns exec r2 ip addr show r2-br | grep 10.0.0.1   # r2 now holds the VIP
$ sudo ip netns exec cli ping -c2 10.0.0.1                   # still works, now via r2
```

### Step 5 — Clean up

```bash
$ sudo pkill keepalived; sudo ip netns delete r1 r2 cli
```

---

## Further Reading

| Topic | Link |
|---|---|
| VRRP | [Wikipedia — Virtual Router Redundancy Protocol](https://en.wikipedia.org/wiki/Virtual_Router_Redundancy_Protocol) |
| keepalived | [keepalived.org](https://www.keepalived.org/) |
| Gratuitous ARP | [Lesson 5 — Neighbor tables & ARP](lesson-05-arp-neigh) |
| High availability | [Wikipedia — High availability](https://en.wikipedia.org/wiki/High_availability) |

---

## Checkpoint

**Q1. When a VRRP backup takes over the VIP, what does it send on the network, and why is it necessary?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It sends a <strong>gratuitous ARP</strong> — an unsolicited ARP announcement stating that the VIP now
maps to <em>its own</em> MAC address. It's necessary because the VIP's IP didn't change, but the MAC
behind it did: switches have the VIP's old MAC in their forwarding tables and clients have it in their
ARP caches, so without intervention frames for the VIP would keep being delivered to the dead master's
MAC until those entries aged out (causing an outage). The gratuitous ARP forces switches and clients to
update immediately to the new master's MAC, so traffic for the VIP redirects to the promoted node right
away. (This is the same gratuitous-ARP mechanism from Lesson 5, used here for IP takeover.)
</details>

---

**Q2. Plain VRRP keeps the VIP on whichever router is "up." What does keepalived's health checking add, and why does it matter?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Plain VRRP only fails over when the <em>router itself</em> stops sending advertisements — i.e. it
detects host death, not service death. A node can be perfectly alive (still mastering the VIP) while the
<em>service</em> it fronts has crashed, so traffic keeps flowing to a black hole. keepalived adds
<strong>health checks</strong> (vrrp_script): it actively probes the real service (TCP/HTTP/custom), and
on failure <strong>lowers that node's VRRP priority</strong>, causing a healthy backup to win the
election and take the VIP. This makes failover <strong>service-aware</strong> rather than merely
host-aware, which matters because users care that the service works, not that some box is pingable.
</details>

---

**Q3. What is split-brain in a VRRP pair, and how is it mitigated?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Split-brain is when <strong>both nodes believe they are master and both claim the VIP simultaneously</strong>.
It happens when they can't hear each other's VRRP advertisements — typically the communication path
between them fails while both nodes are otherwise up and healthy — so each concludes the other is dead
and promotes itself. The result is a duplicated IP/MAC on the network, ARP confusion, and erratic
traffic. Mitigations: provide <strong>redundant communication paths</strong> between the nodes (so a
single link loss doesn't isolate them), use a <strong>quorum/tie-breaker</strong> or fencing so only one
can win, and configure sensible preemption behavior (e.g. <code>nopreempt</code>) so a recovered node
doesn't repeatedly seize the VIP back and cause flapping. The root cause is always a failure of the
nodes to coordinate, so the fixes center on making that coordination resilient.
</details>

---

## Homework

Add a `vrrp_script` to the keepalived configs that checks whether a local web server (`busybox httpd`)
is responding, with `weight -60` on failure. Start the web server on the master, confirm it holds the
VIP, then stop *only the web server* (leaving keepalived and the host running) and verify the VIP
fails over to the backup. Explain why this failover wouldn't happen with plain VRRP.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With a <code>vrrp_script</code> probing the local web server and <code>weight -60</code>, keepalived
periodically runs the check on the master. While the web server is up, the master keeps its full
priority (150) and holds the VIP. When you stop <em>only the web server</em>, the host and keepalived
are still alive — so plain VRRP would see nothing wrong and keep the VIP exactly where it is, leaving
clients pointed at a node whose service is dead. But the health check now fails, so keepalived subtracts
60 from the master's priority (150 → 90), dropping it below the backup's 100; the backup wins the
election, claims the VIP, and sends a gratuitous ARP, so traffic moves to the node whose web server
actually works. This demonstrates the core point: VRRP alone fails over on <em>host</em> failure, while
keepalived's health checks let you fail over on <em>service</em> failure by dynamically adjusting
priority.
</details>
