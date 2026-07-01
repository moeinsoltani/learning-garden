---
title: "Lesson 58 — Large-Scale BGP"
nav_order: 58
parent: "Phase 17: Advanced Routing & Data-Center Fabrics"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 58: Large-Scale BGP — Route Reflectors, Communities, Policy

## Concept

In Lesson 21 you ran BGP between two autonomous systems. Inside a *single* AS, BGP (iBGP) has a
nasty rule: **iBGP does not re-advertise routes learned from one iBGP peer to another iBGP peer**
(to prevent loops, since iBGP has no AS-path defense within the AS). The consequence is brutal —
every iBGP router must peer with *every other* iBGP router: a **full mesh** of N(N-1)/2 sessions.

```
   Full iBGP mesh (4 routers = 6 sessions)     With a route reflector (4 = 3 sessions)
        R1───R2                                      R1     R2
        │ ╳  │                                         ╲   ╱
        R3───R4                                          RR
                                                       ╱   ╲
                                                     R3     R4
```

A **route reflector (RR)** is allowed to break the rule: it *reflects* routes between its clients,
so each router peers only with the RR. This is how a large AS scales BGP.

---

## How it works

**Route reflector.** Clients peer only with the RR(s). The RR readvertises ("reflects") a route
learned from one client to the others, adding originator-id and cluster-list attributes to detect
loops. This turns N(N-1)/2 sessions into roughly N. RRs are usually deployed in redundant pairs.

**BGP communities** ([RFC 1997](https://en.wikipedia.org/wiki/Border_Gateway_Protocol#Communities))
are tags attached to routes — arbitrary 32-bit values like `65001:100` — that carry no meaning to
the protocol but let operators implement policy consistently: "tag customer routes `65001:100`,
then everywhere apply local-preference based on that tag." They decouple *marking* a route from
*acting* on it.

**Route-maps and prefix-lists** are the policy language: a prefix-list matches which prefixes, a
route-map matches (on prefix-list, community, AS-path) and sets attributes (local-pref, MED,
communities) or permits/denies. This is how you steer traffic, prefer one provider, or filter what
you announce.

**ECMP** (equal-cost multipath): when BGP has multiple equally-good paths, the kernel can install
several next-hops and hash flows across them — load-sharing across links (relevant to the
leaf-spine fabric in Lesson 59).

{: .note }
> **Confederations (the other scaling option)**
> Instead of route reflectors, an AS can be split into sub-ASes that speak eBGP to each other
> (a confederation). It's less common than RRs today but solves the same full-mesh problem; you'll
> see it mentioned in large-ISP designs.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `vtysh -c 'show ip bgp summary'` | BGP neighbor states |
| `neighbor <peer> route-reflector-client` | Make a peer an RR client (in `router bgp`) |
| `bgp community ...` / `set community 65001:100` | Tag routes in a route-map |
| `show ip bgp <prefix>` | Inspect a prefix's attributes (communities, originator) |
| `show ip route <prefix>` | See ECMP next-hops installed in the kernel |

---

## Lab

Build a small iBGP setup with one route reflector and confirm clients learn each other's routes
*without* peering directly. Uses FRR (Lesson 19) in namespaces.

### Step 1 — Three routers in AS 65010: RR + two clients

```bash
$ for n in rr c1 c2; do sudo ip netns add $n; done
# wire rr--c1 and rr--c2 (NO c1--c2 link); give each loopback/addresses; start FRR.
```

### Step 2 — Configure the RR

```bash
$ sudo ip netns exec rr vtysh -c 'configure terminal' \
    -c 'router bgp 65010' \
    -c 'neighbor 10.0.1.2 remote-as 65010' \
    -c 'neighbor 10.0.1.2 route-reflector-client' \
    -c 'neighbor 10.0.2.2 remote-as 65010' \
    -c 'neighbor 10.0.2.2 route-reflector-client'
```

### Step 3 — Clients peer ONLY with the RR and originate a prefix

```bash
$ sudo ip netns exec c1 vtysh -c 'configure terminal' \
    -c 'router bgp 65010' -c 'neighbor 10.0.1.1 remote-as 65010' \
    -c 'network 192.0.2.0/24'
# c2 similar, originating 198.51.100.0/24
```

### Step 4 — Verify c1 learns c2's route via the RR

```bash
$ sudo ip netns exec c1 vtysh -c 'show ip bgp 198.51.100.0/24'
# Shows the prefix learned via the RR, with Originator-ID = c2 — even though c1 and c2 never peered.
```

### Step 5 — Clean up

```bash
$ sudo ip netns delete rr c1 c2
```

---

## Further Reading

| Topic | Link |
|---|---|
| BGP | [Wikipedia — Border Gateway Protocol](https://en.wikipedia.org/wiki/Border_Gateway_Protocol) |
| Route reflector | [Wikipedia — Route reflector](https://en.wikipedia.org/wiki/Route_reflector) |
| BGP communities | [Wikipedia — BGP communities](https://en.wikipedia.org/wiki/Border_Gateway_Protocol#Communities) |
| FRR BGP | [docs.frrouting.org](https://docs.frrouting.org/en/latest/bgp.html) |

---

## Checkpoint

**Q1. Why does iBGP require a full mesh by default, and how does a route reflector remove that requirement?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
iBGP has no within-AS loop protection via AS-path (the AS number doesn't change inside the AS), so
to prevent loops the rule is: a router must <em>not</em> re-advertise a route learned from one iBGP
peer to another iBGP peer. The only way every router then hears every route is for all of them to
peer directly — a full mesh of N(N-1)/2 sessions, which explodes with size. A <strong>route
reflector</strong> is explicitly permitted to reflect routes between its clients, using
originator-id and cluster-list attributes for loop detection. Clients peer only with the RR, so
sessions drop from ~N² to ~N, and the RR distributes every client's routes to the others.
</details>

---

**Q2. What problem do BGP communities solve, and how do they differ from a route-map's action?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
BGP communities are arbitrary tags (e.g. <code>65001:100</code>) attached to routes that the protocol
itself ignores but operators agree to interpret. They <strong>decouple marking a route from acting on
it</strong>: you can tag routes at ingress according to where/how they were learned ("customer route,"
"prefer-not"), then apply consistent policy anywhere downstream based purely on the tag, without each
location re-deriving the classification. A <strong>route-map</strong> is the mechanism that does both
sides — it <em>matches</em> (on prefix-list, community, AS-path, …) and <em>sets</em> attributes
(local-pref, MED, or adds/removes communities) or permits/denies. So communities are the data/labels;
route-maps are the if-then policy logic that reads and writes them.
</details>

---

**Q3. You have two equal-cost BGP paths to a destination. What is ECMP, and what does the kernel do?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
ECMP (Equal-Cost Multi-Path) means that when BGP (or any routing source) yields multiple paths of
equal cost to the same prefix, the router installs <em>several</em> next-hops for it rather than
picking one. The kernel then <strong>hashes each flow</strong> (typically on the 5-tuple) to a
next-hop, spreading traffic across the links while keeping any single flow on one path (so it doesn't
reorder). This gives load-sharing and redundancy — if one next-hop fails, the others continue. It's
the foundation of the leaf-spine data-center fabric in Lesson 59, where every leaf has equal-cost
paths up to multiple spines.
</details>

---

## Homework

Add a route-map on the RR that sets `local-preference 200` on routes carrying community `65010:100`,
and have c1 tag its `192.0.2.0/24` announcement with that community. Verify on c2 that the route
arrives with the higher local-preference. Explain how this pattern lets an operator implement
"prefer routes from site A" as policy without touching every router.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
c1 attaches <code>65010:100</code> to its <code>192.0.2.0/24</code> announcement; the RR's route-map
matches that community and sets <code>local-preference 200</code>; c2 then sees the prefix with
local-pref 200 (higher than the default 100), so BGP's best-path selection prefers it. The point of
the pattern: the <strong>marking</strong> (community) is applied once at the source, and the
<strong>policy</strong> (raise local-pref for that tag) is applied in one place (the RR / a shared
route-map), so "prefer routes from site A" is expressed as a single rule rather than being hand-coded
on every router. Change the policy in one spot and it applies fleet-wide; add a new site and you just
tag it with the agreed community. This separation of tagging from action is exactly why large networks
run almost entirely on community-driven route-maps.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 59 — BGP EVPN & VXLAN Fabrics →](lesson-59-evpn){: .btn .btn-primary }
