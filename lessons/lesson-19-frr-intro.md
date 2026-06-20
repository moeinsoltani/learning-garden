---
title: "Lesson 19 — FRR Introduction"
nav_order: 19
parent: "Phase 6: Dynamic Routing"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 19: FRR Introduction

## Concept

Everything you've routed so far was **static** — you typed every route by hand. That works for a handful of namespaces, but real networks have dozens of routers and links that fail. A **dynamic routing protocol** lets routers *talk to each other*, discover the topology automatically, and recompute routes the instant something changes.

**FRR** (Free Range Routing) is the leading open-source router suite on Linux — the successor to Quagga, used in production by major cloud and networking vendors. It turns a Linux box (or namespace) into a full-featured router speaking OSPF, BGP, IS-IS, RIP, and more.

```
   Static routing:                 Dynamic routing (FRR):
   you → type each route           routers → exchange reachability
   link dies → you fix it          link dies → protocol reroutes in seconds
   doesn't scale                   scales to thousands of prefixes
```

---

## How it works

FRR is not one program but a set of cooperating **daemons**, coordinated by a core process:

| Daemon | Role |
|---|---|
| `zebra` | The core. Manages the kernel routing table (the RIB — Routing Information Base). All other daemons hand their best routes to zebra, which installs them into the kernel. |
| `ospfd` | Speaks OSPF (link-state IGP). |
| `bgpd` | Speaks BGP (path-vector, inter-AS). |
| `staticd` | Manages static routes within FRR's config. |
| `ripd`, `isisd`, … | Other protocols. |

The flow: each protocol daemon learns routes from its peers, picks its best ones, and offers them to **zebra**. Zebra arbitrates between protocols and installs the winners into the **kernel RIB** — the same table you've been editing with `ip route`. So FRR ultimately drives the very routing table you already understand; it just populates it automatically.

```
   ospfd ─┐
   bgpd ──┼──► zebra ──► kernel routing table (what `ip route show` displays)
   staticd┘     (RIB manager / arbiter)
```

You configure FRR through **vtysh**, a Cisco/IOS-style CLI. If you've seen `configure terminal`, `router ospf`, `show ip route` — that's vtysh. It feels like configuring a hardware router.

{: .note }
> **RIB vs FIB**
> The **RIB** (Routing Information Base) is the full set of candidate routes a router knows, including alternatives. The **FIB** (Forwarding Information Base) is the subset actually used to forward packets — the "winners." In FRR, the protocol daemons build their RIBs, zebra selects best routes and pushes them to the kernel, and the kernel's forwarding table is the FIB. `show ip route` in vtysh shows FRR's RIB; `ip route show` shows what got installed (the FIB).

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `sudo apt install frr frr-pythontools` | Install FRR (Debian/Ubuntu) |
| `vtysh` | Enter FRR's integrated CLI |
| `show running-config` | (in vtysh) Show current FRR configuration |
| `show daemons` | (in vtysh) Which FRR daemons are running |
| `configure terminal` | (in vtysh) Enter config mode |
| `show ip route` | (in vtysh) Show FRR's RIB |
| `/etc/frr/daemons` | File that enables/disables individual daemons |

---

## Lab

This lesson is about understanding FRR's architecture; Lessons 20–21 do the protocol labs. Here you install FRR and explore its structure.

### Step 1 — Install FRR

```bash
$ sudo apt update && sudo apt install -y frr frr-pythontools
$ frr --version
FRRouting 8.x ...
```

### Step 2 — Enable the daemons you'll use

FRR ships with most daemons disabled. Edit `/etc/frr/daemons` and turn on the ones you need:

```bash
# /etc/frr/daemons
zebra=yes
ospfd=yes
bgpd=yes
staticd=yes
```

Then restart and confirm:

```bash
$ sudo systemctl restart frr
$ sudo systemctl status frr
   Active: active (running)
```

### Step 3 — Enter vtysh and look around

```bash
$ sudo vtysh

router# show daemons
zebra ospfd bgpd staticd

router# show ip route
Codes: K - kernel route, C - connected, S - static, O - OSPF, B - BGP
C>* 127.0.0.0/8 is directly connected, lo
...
```

The codes letter on each route shows *which source* installed it: `C` = connected, `S` = static, `O` = OSPF, `B` = BGP, `K` = kernel. This is zebra's arbitration made visible.

### Step 4 — Add a static route the FRR way and see it reach the kernel

```bash
router# configure terminal
router(config)# ip route 203.0.113.0/24 10.0.0.1
router(config)# exit
router# show ip route static
S>* 203.0.113.0/24 [1/0] via 10.0.0.1, ...
router# exit
```

Now confirm zebra installed it into the *actual kernel table*:

```bash
$ ip route show | grep 203.0.113
203.0.113.0/24 via 10.0.0.1 dev ... proto 196
#                                    ^^^^^^^^^
#   proto 196 (or "zebra"/"static") = installed by FRR, not by hand
```

The `proto` field shows the route came from FRR's zebra, not from `ip route add`. This is the whole point: you configured in vtysh, and zebra pushed it into the kernel FIB.

### Step 5 — See the daemon processes

```bash
$ ps -ef | grep -E 'zebra|ospfd|bgpd' | grep -v grep
... /usr/lib/frr/zebra ...
... /usr/lib/frr/ospfd ...
... /usr/lib/frr/bgpd ...
```

Each protocol is a separate process; zebra is the one touching the kernel table.

### Step 6 — Clean up

```bash
router# configure terminal
router(config)# no ip route 203.0.113.0/24 10.0.0.1
router(config)# exit
```

(Leave FRR installed — Lessons 20 and 21 use it.)

---

## Further Reading

| Topic | Link |
|---|---|
| FRRouting | [Wikipedia — FRRouting](https://en.wikipedia.org/wiki/FRRouting) |
| FRR official docs | [docs.frrouting.org](https://docs.frrouting.org/) |
| Quagga (predecessor) | [Wikipedia — Quagga](https://en.wikipedia.org/wiki/Quagga_(software)) |
| Routing table / RIB | [Wikipedia — Routing table](https://en.wikipedia.org/wiki/Routing_table) |

---

## Checkpoint

**Q1. What problem does a dynamic routing protocol solve that static routes cannot?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Static routes are fixed: a human enters each one, and they don't change when the network does. A dynamic routing protocol solves two things static routes can't:

1. **Automatic adaptation to failure.** When a link or router goes down, neighboring routers detect it and recompute paths within seconds, rerouting traffic around the failure with no human intervention. Static routes just keep pointing at the dead path until someone fixes them manually.
2. **Scale.** In a network with many routers and prefixes, configuring (and keeping consistent) every route by hand is infeasible and error-prone. A protocol distributes reachability information automatically, so each router learns the whole topology without anyone typing thousands of routes.

In short, dynamic routing gives you self-healing and scalability; static routing gives you simplicity for small, stable topologies.
</details>

---

**Q2. In FRR, what is the role of `zebra` versus the protocol daemons like `ospfd` and `bgpd`?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The protocol daemons (`ospfd`, `bgpd`, etc.) each *speak their protocol* to peers, learn routes, and compute their own best paths. But they don't touch the kernel directly. Instead they hand their chosen routes to **zebra**, the core RIB-manager daemon. Zebra's job is arbitration and installation: when multiple sources offer routes to the same prefix, zebra picks the winner (based on administrative distance/preference) and installs it into the kernel's forwarding table — the same table `ip route show` displays. So the protocol daemons are the "brains" that learn routes, and zebra is the single "hand" that writes the final decisions into the kernel. This separation lets many protocols coexist while only one component owns the kernel table.
</details>

---

**Q3. You configure a route in vtysh and then run `ip route show` on the Linux host. How can you tell the route came from FRR rather than being added manually with `ip route add`?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
By the route's **`proto` field**. Routes installed by FRR's zebra are tagged with a protocol identifier (shown as `proto zebra`, `proto static`/`196`, `proto ospf`, `proto bgp`, etc.) rather than the `proto static`/`proto boot` you'd get from a manual `ip route add` — and notably *not* `proto kernel` (which is for auto-created connected routes). Each route-origin stamps its source into this field, so `ip route show` reveals provenance. This is the kernel side of zebra's work: vtysh configuration becomes real kernel routes, identifiable by who installed them. (In vtysh itself, `show ip route` shows a per-route code letter — `O` for OSPF, `B` for BGP, `S` for static — conveying the same information.)
</details>

---

## Homework

With FRR installed, enable `zebra` and `staticd` only (leave ospfd/bgpd off for now). In vtysh, add three static routes and a blackhole route. Then compare the output of `show ip route` (in vtysh) with `ip route show` (on the host). Note every difference you can find and explain what each tells you about the RIB-vs-FIB distinction.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Key things you should observe:

- **Code letters / proto tags.** vtysh's `show ip route` prefixes each route with a code (`S` static, `C` connected, `K` kernel) and shows administrative distance/metric in brackets like `[1/0]`. `ip route show` instead shows a `proto` field (`static`/`zebra`/`kernel`) — the kernel's record of who installed it. They're describing the same routes from two viewpoints.
- **Selected vs candidate.** In vtysh, a `>` marks the route zebra *selected* as best (FIB candidate) and `*` marks it as FIB-installed. Routes that lost arbitration appear in the RIB (`show ip route`) but are **not** present in `ip route show`, because only the winners get pushed to the kernel FIB. This is the RIB/FIB distinction made concrete: FRR may know several routes to a prefix (RIB), but only the chosen one reaches the kernel forwarding table (FIB).
- **Blackhole.** The blackhole route appears in both, but on the host it shows as a `blackhole` route type in `ip route show`, confirming zebra translated FRR's config into the kernel's native route type.
- **Connected routes.** These show as `C` in FRR and `proto kernel` on the host — auto-derived from interface addresses, present in both because they're always selected.

The overall lesson: vtysh shows FRR's full routing knowledge (the RIB, including alternatives and selection markers), while `ip route show` shows only the forwarding outcome (the FIB) that zebra actually programmed into the kernel.
</details>
