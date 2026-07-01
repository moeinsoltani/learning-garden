---
title: "Lesson 61 — Anycast"
nav_order: 61
parent: "Phase 17: Advanced Routing & Data-Center Fabrics"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 61: Anycast

## Concept

**Anycast** is "one address, many origins." The *same* IP is announced from multiple locations, and
the routing system delivers each client to the *nearest* one (by routing metric). No client config,
no load balancer — the network itself picks the closest instance.

```
        announce 192.0.2.1                announce 192.0.2.1
   ┌──────────────────────┐         ┌──────────────────────┐
   │  London server       │         │  Tokyo server         │
   └─────────┬────────────┘         └──────────┬───────────┘
             │  both advertise the SAME /24 into routing       │
   client in Europe ─────► London      client in Asia ─────► Tokyo
   (routing sends each to the topologically nearest instance)
```

This is how DNS root servers, public resolvers (`8.8.8.8`, `1.1.1.1`), and CDNs scale globally and
absorb attacks.

---

## How it works

**Same prefix, many announcers.** Each site runs a router that announces the anycast prefix into
BGP (globally) or an IGP (within an AS). Routing's normal "best path" selection then naturally sends
each client to whichever announcement is closest from its vantage point.

**Failure handling = route withdrawal.** If a site goes down, its router stops announcing the prefix
(or a health check withdraws it). Routing reconverges and clients are drawn to the next-nearest
site. No DNS change, no client timeout on a dead IP — the address simply moves with the routing
table.

**The catch: per-flow stability isn't guaranteed.** Anycast routes *packets*, and a routing change
(link flap, reconvergence, ECMP rehash) can shift a flow from one instance to a *different* instance
mid-connection. For **stateless** request/response (a DNS query, a TLS-terminated HTTP request)
that's fine — any instance can answer. For **long-lived stateful** connections, landing on a server
that has no idea about your session breaks it. That's why anycast is paired with stateless protocols,
or backed by shared state / consistent steering at the edge.

{: .note }
> **Anycast vs a load balancer**
> A load balancer is one front-end IP distributing to many backends *behind it* (Lesson 67). Anycast
> distributes the front-end IP *itself* across the planet via routing — there's no single box in the
> path, so it scales without a chokepoint and provides DDoS dispersion (an attack on the IP is split
> across every site). They're complementary: anycast to the nearest site, load-balance within it.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip addr add <anycast-ip>/32 dev lo` | Bind the shared anycast address (commonly on loopback) |
| `vtysh -c 'network <prefix>'` | Announce the anycast prefix from each site (FRR) |
| `ip route get <anycast-ip>` | See which instance the routing table selects |
| `show ip bgp <prefix>` | See multiple origins for the same prefix |

---

## Lab

Two "sites" announce the same anycast address; a client reaches whichever is closer, and you watch
failover when one withdraws.

### Step 1 — Topology: client — router — {siteA, siteB}, both sites bind the same IP

```bash
$ for n in client r siteA siteB; do sudo ip netns add $n; done
# wire client--r, r--siteA, r--siteB; make r's path to siteA cheaper than to siteB.
$ sudo ip netns exec siteA ip addr add 192.0.2.1/32 dev lo
$ sudo ip netns exec siteB ip addr add 192.0.2.1/32 dev lo
$ sudo ip netns exec siteA ip link set lo up; sudo ip netns exec siteB ip link set lo up
```

### Step 2 — Each site announces 192.0.2.1/32 to r (FRR/BGP or static)

```bash
# Both originate the same /32; r prefers siteA (lower metric).
$ sudo ip netns exec r ip route get 192.0.2.1
192.0.2.1 via <siteA> ...        # nearest instance selected
```

### Step 3 — Reach the anycast service and confirm which site answers

```bash
$ sudo ip netns exec siteA nc -l -p 80 &      # only siteA should get the connection
$ sudo ip netns exec client sh -c 'echo hi | nc 192.0.2.1 80'   # served by siteA
```

### Step 4 — Withdraw siteA and watch failover

```bash
# Bring down siteA's announcement (or the link); routing reconverges to siteB:
$ sudo ip netns exec siteA ip link set lo down
$ sudo ip netns exec r ip route get 192.0.2.1
192.0.2.1 via <siteB> ...        # the SAME address now resolves to siteB
```

### Step 5 — Clean up

```bash
$ sudo ip netns delete client r siteA siteB
```

---

## Further Reading

| Topic | Link |
|---|---|
| Anycast | [Wikipedia — Anycast](https://en.wikipedia.org/wiki/Anycast) |
| DNS root server anycast | [Wikipedia — Root name server](https://en.wikipedia.org/wiki/Root_name_server) |
| BGP | [Lesson 21 — BGP with FRR](lesson-21-bgp) |
| ECMP/Clos | [Wikipedia — Clos network](https://en.wikipedia.org/wiki/Clos_network) |

---

## Checkpoint

**Q1. How does anycast deliver a client to the "nearest" server without any client configuration or load balancer?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The same IP prefix is announced into the routing system from multiple sites. Routing's normal best-path
selection — which already prefers the topologically closest route — independently delivers each client
to whichever announcement is nearest from <em>its</em> location. There's no client setup (everyone just
uses the one address) and no box in the path choosing a backend; the <strong>routing table itself</strong>
is the distribution mechanism. A client in Europe follows the European announcement, one in Asia follows
the Asian announcement, all to the identical IP.
</details>

---

**Q2. What happens to clients when an anycast site fails, and why is recovery automatic?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
When a site fails, it (or its health check) <strong>stops announcing the anycast prefix</strong>, so
that route is withdrawn from the routing system. Routing <strong>reconverges</strong> and the clients
that were going to the failed site are now drawn to the next-nearest site that still announces the
prefix. Recovery is automatic because the address isn't tied to one machine — it "lives" wherever it's
being announced, so removing one announcer simply shifts traffic to the others, with no DNS change and
no clients stuck timing out against a dead IP. The convergence time is essentially the routing
protocol's reconvergence time.
</details>

---

**Q3. Why is anycast great for stateless services but risky for long-lived stateful connections?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Anycast routes <em>packets</em> to the nearest instance, but "nearest" can change mid-connection: a
link flap, routing reconvergence, or ECMP rehash can redirect a flow to a <em>different</em> instance.
For <strong>stateless</strong> services (a DNS query, a single TLS-terminated HTTP request) that's
harmless — any instance can handle any request independently. For <strong>long-lived stateful</strong>
connections, being shifted to an instance that knows nothing about your TCP/TLS/session state breaks
the connection. So anycast pairs naturally with stateless request/response protocols; to use it for
stateful traffic you need shared/replicated state across sites or edge steering that keeps a flow
pinned to one instance.
</details>

---

## Homework

Make the two sites equidistant from the client (equal routing cost) and enable ECMP on the router.
Observe how flows are distributed, then deliberately flap one link and explain what happens to an
in-progress TCP connection versus a series of independent DNS-style requests. Tie your answer back to
why CDNs terminate TLS at the edge.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With equal-cost paths and ECMP, the router hashes flows across both sites — each 5-tuple is pinned to
one site for its lifetime, so different connections land on different sites but a single flow normally
stays put. When you <strong>flap a link</strong>, the ECMP set changes and the hash is recomputed, so
some existing flows get <strong>rehashed to the other site</strong>. An in-progress <strong>TCP
connection</strong> that moves lands on a server with no record of it and is reset/broken (it must
reconnect). A series of independent <strong>DNS-style requests</strong>, by contrast, is unaffected —
each request is self-contained, so whichever site answers is fine. This is exactly why CDNs and edge
networks <strong>terminate TLS/HTTP at the anycast edge</strong>: they keep the anycast-facing work
stateless (or per-request), absorbing routing churn gracefully, and only then hand off to stable,
unicast-addressed backends (or shared state) for anything long-lived — so a reconvergence never kills a
user's session.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 62 — TCP Internals & Congestion Control →](lesson-62-tcp-internals){: .btn .btn-primary }
