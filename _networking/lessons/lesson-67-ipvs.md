---
title: "Lesson 67 — Layer-4 Load Balancing (IPVS/LVS)"
nav_order: 67
parent: "Phase 19: Network Services"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 67: Layer-4 Load Balancing — IPVS/LVS

## Concept

A **load balancer** presents one *virtual* address (a VIP) and spreads incoming connections across a
pool of backend **real servers**. **IPVS** (IP Virtual Server, the kernel half of LVS) is a Layer-4
load balancer built into the Linux kernel: it makes forwarding decisions per *connection* (by IP and
port), not by inspecting HTTP — fast, but protocol-agnostic.

```
                          ┌─► real server 1
   client ─► VIP (IPVS) ──┼─► real server 2     IPVS picks a backend per new connection
                          └─► real server 3     and pins that connection to it
```

This is the mechanism behind kube-proxy's IPVS mode (Lesson 43).

---

## How it works

**Scheduling algorithms** decide which backend a *new* connection goes to:

| Scheduler | Behavior |
|---|---|
| `rr` | Round-robin — next backend each time |
| `wrr` | Weighted round-robin — bigger servers get more |
| `lc` | Least-connection — fewest active connections wins |
| `wlc` | Weighted least-connection |
| `sh` | Source hashing — same client → same backend (stickiness) |

**Three forwarding modes** — *how* the packet reaches the backend, which determines how the **reply**
travels:

| Mode | How | Return path |
|---|---|---|
| **NAT** (masq) | LB rewrites dest IP to the backend; reply must come *back through the LB* to be un-NATed | through the LB (bottleneck) |
| **DR** (Direct Routing) | LB rewrites only the *MAC*; backend shares the VIP (on `lo`) and replies **directly to the client** | bypasses the LB |
| **TUN** (tunnel) | LB encapsulates to the backend (can be remote); backend replies directly | bypasses the LB |

**Why DR scales.** In NAT mode *every* packet — request *and* reply — traverses the load balancer, so
the LB's bandwidth caps the whole service (and replies are usually far bigger than requests). In
**Direct Routing**, the LB only touches the small inbound request (rewriting the destination MAC to
the chosen backend); the backend, which holds the VIP on its loopback, sends the large reply
**straight to the client**, never returning through the LB. The LB handles only inbound, so it scales
far higher.

**Persistence.** IPVS can pin a client to the same backend for a timeout (needed for stateful apps),
the L4 analogue of session affinity.

{: .note }
> **L4 vs L7**
> IPVS is L4: it balances *connections* by IP/port and can't see URLs, cookies, or hostnames. An L7
> proxy (HAProxy, Envoy, nginx) terminates the connection and balances *requests* with full HTTP
> visibility — more features, more overhead. They're often layered: L4 to spread load across L7
> proxies, L7 for smart routing.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ipvsadm -A -t <VIP:port> -s rr` | Add a virtual service with a scheduler |
| `ipvsadm -a -t <VIP:port> -r <real> -m` | Add a real server (`-m` NAT, `-g` DR, `-i` TUN) |
| `ipvsadm -L -n` | List services/backends and connection counts |
| `ipvsadm -L -n --stats` | Per-backend traffic stats |
| `ipvsadm -e ... -w <n>` | Change a backend's weight |

---

## Lab

Build a NAT-mode IPVS service across two backends in namespaces and watch round-robin distribute
connections.

### Step 1 — A director + two backends

```bash
$ sudo ip netns add lb; sudo ip netns add b1; sudo ip netns add b2
# wire lb to a client side and to b1/b2; enable ip_forward on lb. b1=10.0.0.11, b2=10.0.0.12.
$ sudo ip netns exec b1 sh -c 'echo b1 > /tmp/i; busybox httpd -p 80 -h /tmp' 2>/dev/null
$ sudo ip netns exec b2 sh -c 'echo b2 > /tmp/i; busybox httpd -p 80 -h /tmp' 2>/dev/null
```

### Step 2 — Define the virtual service and add backends (NAT mode)

```bash
$ sudo ip netns exec lb ipvsadm -A -t 10.0.0.1:80 -s rr
$ sudo ip netns exec lb ipvsadm -a -t 10.0.0.1:80 -r 10.0.0.11:80 -m
$ sudo ip netns exec lb ipvsadm -a -t 10.0.0.1:80 -r 10.0.0.12:80 -m
```

### Step 3 — Hit the VIP repeatedly and watch round-robin

```bash
$ for i in 1 2 3 4; do sudo ip netns exec client curl -s http://10.0.0.1/i; done
b1
b2
b1
b2            # connections alternate between backends
$ sudo ip netns exec lb ipvsadm -L -n      # ActiveConn split across the two reals
```

### Step 4 — Clean up

```bash
$ sudo ip netns delete lb b1 b2
```

---

## Further Reading

| Topic | Link |
|---|---|
| LVS / IPVS | [Wikipedia — Linux Virtual Server](https://en.wikipedia.org/wiki/Linux_Virtual_Server) |
| Load balancing | [Wikipedia — Load balancing (computing)](https://en.wikipedia.org/wiki/Load_balancing_(computing)) |
| Kubernetes networking | [Lesson 43 — Kubernetes networking](lesson-43-kubernetes-networking) |
| `ipvsadm` | [man7.org — ipvsadm(8)](https://man7.org/linux/man-pages/man8/ipvsadm.8.html) |

---

## Checkpoint

**Q1. Why does Direct Routing (DR) mode scale better than NAT mode for return traffic?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
In <strong>NAT mode</strong>, the load balancer rewrites the destination IP to the chosen backend, so
the backend's reply is addressed back to the LB, which must un-NAT it and forward it to the client —
meaning <em>every</em> packet in both directions passes through the LB. Since replies (e.g. web content)
are usually much larger than requests, the LB's bandwidth becomes the bottleneck for the whole service.
In <strong>Direct Routing mode</strong>, the LB only rewrites the destination <em>MAC</em> of the inbound
request to the backend (the backend holds the VIP on its loopback), and the backend replies
<strong>directly to the client</strong>, bypassing the LB entirely. The LB therefore handles only the
small inbound request traffic, not the large outbound replies, so it can sustain far more throughput
with the same hardware.
</details>

---

**Q2. IPVS is a Layer-4 load balancer. What can it not do that an L7 proxy can?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
IPVS balances at L4 — it decides per <em>connection</em> using IP addresses and ports, and forwards
packets without understanding the application protocol. So it <strong>cannot</strong> route based on
HTTP details: it can't see the URL/path, the Host header (so no name-based virtual hosting), cookies
(so no cookie-based session affinity beyond source-IP stickiness), methods, or content; it can't
terminate TLS, rewrite headers, retry failed HTTP requests, or do content-based routing. An L7 proxy
(HAProxy/Envoy/nginx) terminates the connection and sees the full request, enabling all of that — at
the cost of more per-request overhead. The trade-off is raw speed/simplicity (L4/IPVS) versus
application-aware features (L7).
</details>

---

**Q3. What is persistence (session affinity) in a load balancer, and when do you need it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Persistence pins a given client to the <strong>same backend</strong> across multiple connections for a
configured timeout (in IPVS, via the persistence option or the source-hashing scheduler). You need it
when the backend holds <strong>per-client state that isn't shared</strong> across the pool — e.g. an
in-memory session, a shopping cart, or a TLS session cached on one server — so sending the client's
next connection to a different backend would lose that state and break the experience. The better
long-term fix is to make backends stateless (shared session store), but persistence is the pragmatic
mechanism when they're not. The trade-off is less even load distribution, since clients stick rather
than being freely rebalanced.
</details>

---

## Homework

Reconfigure the lab to use weighted round-robin with backend b1 given weight 3 and b2 weight 1, then
send a batch of requests and confirm the ~3:1 split with `ipvsadm -L -n --stats`. Then reason about
why DR mode would be impractical to demonstrate in this simple namespace setup (hint: what must the
backends do with the VIP, and what ARP problem appears?).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With <code>-w 3</code> on b1 and <code>-w 1</code> on b2 under <code>wrr</code>, roughly three out of
every four connections go to b1 and one to b2, which <code>ipvsadm -L -n --stats</code> confirms in the
per-backend counters. DR mode is awkward to demo here because Direct Routing requires <strong>each
backend to own the VIP itself</strong> (configured on its loopback) so it can accept packets destined
to the VIP and reply directly to the client. That creates the classic <strong>ARP problem</strong>:
multiple machines on the same L2 segment now have the same VIP, so they would all answer ARP requests
for it, causing conflicts and sending client traffic to the wrong host. Real DR deployments must
<strong>suppress ARP for the VIP on the backends</strong> (e.g. <code>arp_ignore</code>/<code>arp_announce</code>
sysctls so only the director answers ARP for the VIP). Setting that up correctly across namespaces is
fiddly and easy to get wrong, which is why the simple lab uses NAT mode — but the ARP-suppression
requirement is exactly the operational subtlety that makes DR powerful yet tricky in production.
</details>
