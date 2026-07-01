---
title: "Lesson 46 — Full Diagnostic Toolchain"
nav_order: 46
parent: "Phase 15: Troubleshooting"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 46: Full Diagnostic Toolchain

## Concept

You now know a lot of commands. The skill this lesson builds is knowing **which tool answers which question** — so that when something is broken, you don't flail, you *interrogate the stack*. Every diagnostic command answers exactly one question about one layer. Network debugging is just asking those questions in order until one of them gives a wrong answer.

```
   Layer            Question                        Tool
   ─────────────────────────────────────────────────────────────────
   L1/L2 link       Is the interface up? carrier?   ip link show
   L3 address       Is an IP assigned, right mask?  ip addr show
   L2 resolution    Is ARP resolving to a MAC?      ip neigh show
   L3 routing       Which route is chosen?          ip route get <dst>
   L2 forwarding    Is the bridge learning MACs?    bridge fdb show
   filtering        Is a rule dropping it?          nft list ruleset
   NAT/state        Is the flow tracked? valid?     conntrack -L
   service          Is anything listening?          ss -tulpn
   the packet       Where does it stop appearing?   tcpdump
   eBPF             Is XDP/TC intercepting?         bpftool net show
   forwarding knob  Is routing enabled?             sysctl net.ipv4.ip_forward
```

A tool isn't useful because it's clever — it's useful because it **definitively answers one question** so you can rule that layer in or out and move on. Memorize the question each one answers.

---

## How it works

The toolchain, with the precise question each tool resolves:

| Tool | The question it answers |
|---|---|
| `ip link show` | Is the interface **up** and does it have **carrier** (`state UP`, `LOWER_UP`)? |
| `ip addr show` | Is an **IP address assigned**, with the **right prefix**? |
| `ip neigh show` | Is **ARP/NDP resolving** the next hop to a MAC (`REACHABLE`), or `FAILED`? |
| `ip route get <dst>` | **Which route/gateway/interface** will the kernel actually use for this destination? |
| `bridge fdb show` | Is the **bridge learning** the destination MAC (L2 forwarding)? |
| `nft list ruleset` | Is a **firewall rule** dropping/rejecting the packet? |
| `conntrack -L` | Is the connection **being tracked**, and is its **state valid** (vs INVALID)? |
| `ss -tulpn` | Is the **service actually listening** on the expected address:port? |
| `tcpdump` | **Where does the packet stop appearing** along the path? |
| `bpftool net show` | Is an **XDP or TC BPF program** intercepting packets? |
| `sysctl net.ipv4.ip_forward` | Is **forwarding enabled** on a box that's supposed to route? |

Two complementary styles of question:

- **State queries** (`ip link/addr/neigh/route`, `ss`, `conntrack -L`, `nft list`, `bpftool`) ask "what is the configured/current state?" — fast, and they often find the fault outright (no IP, wrong route, drop rule, nothing listening).
- **Observation** (`tcpdump`) asks "what is *actually happening* to the packets?" — the ground truth that confirms or refutes what the state queries implied. Use state queries to form a hypothesis, tcpdump to prove it.

{: .note }
> **`ip route get` is the most underused command in the list**
> Instead of reading the whole routing table and reasoning about longest-prefix match yourself, **ask the kernel**: `ip route get 8.8.8.8` returns the exact route, gateway, source IP, and outgoing interface it *would* use for that destination. It collapses "which of my routes wins?" into one authoritative answer — run it from *both* ends to check the **return path** too.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ss -tulpn` | TCP/UDP listening sockets + owning process |
| `ss -ti` | Per-connection TCP internals (rtt, retransmits, cwnd) |
| `conntrack -L` | List tracked connections and their states |
| `ip route get <dst>` | The exact route the kernel selects for a destination |
| `bpftool net show` | BPF programs attached to network hooks |
| `ip -s link show <dev>` | Interface counters (RX/TX, drops, errors) |

---

## Lab

We'll run the full battery of checks against a **healthy** namespace pair first (so you know what "good" looks like), then in the homework you'll apply them to a broken one.

### Step 1 — Build a known-good pair

```bash
$ sudo ip netns add x; sudo ip netns add y
$ sudo ip link add vx netns x type veth peer name vy netns y
$ sudo ip netns exec x ip addr add 10.1.1.1/24 dev vx
$ sudo ip netns exec y ip addr add 10.1.1.2/24 dev vy
$ sudo ip netns exec x ip link set vx up; sudo ip netns exec x ip link set lo up
$ sudo ip netns exec y ip link set vy up; sudo ip netns exec y ip link set lo up
```

### Step 2 — Walk the toolchain (what a healthy answer looks like)

```bash
$ sudo ip netns exec x ip link show vx          # state UP, LOWER_UP
$ sudo ip netns exec x ip addr show vx          # 10.1.1.1/24
$ sudo ip netns exec x ip route get 10.1.1.2    # via dev vx, src 10.1.1.1
$ sudo ip netns exec x ping -c1 10.1.1.2        # works
$ sudo ip netns exec x ip neigh show            # 10.1.1.2 ... REACHABLE
```

### Step 3 — Is a service listening?

```bash
$ sudo ip netns exec y python3 -m http.server 8080 &
$ sudo ip netns exec y ss -tulpn                # LISTEN 0.0.0.0:8080 ... python3
$ sudo ip netns exec x curl -s 10.1.1.2:8080 >/dev/null && echo OK
```

### Step 4 — Confirm with the packet view

```bash
$ sudo ip netns exec y tcpdump -ni vy port 8080 &
$ sudo ip netns exec x curl -s 10.1.1.2:8080 >/dev/null
# capture shows the SYN/SYN-ACK/ACK + data — ground truth that traffic flows
```

### Step 5 — Check counters and any BPF hooks

```bash
$ sudo ip netns exec y ip -s link show vy        # RX/TX packets, 0 drops/errors
$ sudo ip netns exec y bpftool net show          # (empty here — nothing intercepting)
```

### Step 6 — Clean up

```bash
$ sudo kill %1 2>/dev/null
$ sudo ip netns delete x; sudo ip netns delete y
```

---

## Further Reading

| Topic | Link |
|---|---|
| `ss` | [man7.org — ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html) |
| `ip route` | [man7.org — ip-route(8)](https://man7.org/linux/man-pages/man8/ip-route.8.html) |
| `conntrack` | [conntrack-tools](https://conntrack-tools.netfilter.org/) |
| `bridge` | [man7.org — bridge(8)](https://man7.org/linux/man-pages/man8/bridge.8.html) |
| `bpftool net` | [man7.org — bpftool-net(8)](https://man7.org/linux/man-pages/man8/bpftool-net.8.html) |

---

## Checkpoint

**Q1. For each tool, state the one question it answers: `ip route get`, `ss -tulpn`, `conntrack -L`, `nft list ruleset`, `ip neigh show`.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>

- **`ip route get <dst>`** — *Which route does the kernel actually select for this destination?* (exact gateway, source IP, and outgoing interface — the authoritative answer, no manual longest-prefix reasoning).
- **`ss -tulpn`** — *Is a service actually listening, on what address:port, and which process owns it?* (rules out "nothing's there to answer").
- **`conntrack -L`** — *Is this connection being tracked, and what state is it in?* (is there a NAT/state entry, is it `ESTABLISHED` vs `INVALID`).
- **`nft list ruleset`** — *Is a firewall rule dropping/rejecting/translating the packet?* (the full active rule set with hooks and verdicts).
- **`ip neigh show`** — *Is the next hop's L3 address resolving to a MAC?* (`REACHABLE` good, `FAILED`/missing means ARP/NDP is the problem).
</details>

---

**Q2. A packet leaves host A but the application on host B never receives it, yet `tcpdump` on B *does* show the packet arriving. Which tools do you check next, and why?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The packet reaching B's interface but not the application means it's being lost **between the wire and the socket on B** — so the link/route/ARP layers are fine and you focus on B's *local input path*:

1. **`nft list ruleset`** — is an **input firewall rule** dropping or rejecting it before it reaches the socket? (Most common cause.) Temporarily add a `log` rule to confirm.
2. **`ss -tulpn`** — is the application **actually listening** on the **expected address and port**? (e.g. bound to `127.0.0.1` instead of `0.0.0.0`, or a different port — the packet arrives but nothing is listening where it landed.)
3. **`conntrack -L`** — is the flow marked **INVALID** (so a stateful rule drops it), or is there a NAT mismatch?
4. **`bpftool net show`** / **`ip link show`** — is an **XDP/TC BPF** program dropping it before the stack delivers it?
5. **`ip -s link show`** — are **RX drops** climbing (buffer/qdisc/socket overflow)?

The logic: tcpdump proving arrival rules out *transit* problems and points you squarely at **B's local delivery path** — firewall input, listening socket, conntrack state, or a BPF hook — which is exactly the short list above.
</details>

---

**Q3. Why is `tcpdump` described as "ground truth" while the `ip`/`ss`/`nft` commands give "hypotheses"?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The `ip`/`ss`/`nft`/`conntrack`/`bpftool` commands report **configured or recorded state** — what the routing table says, what rules exist, what's supposed to be listening. That tells you what *should* happen, which is a strong **hypothesis** about behavior, but configuration and reality can diverge (a rule you misread, an interface in an odd state, an MTU/asymmetric-routing issue the config doesn't reveal). **`tcpdump` observes the actual packets** as they pass — it shows what is *really happening* on the wire, independent of what any config claims. So the productive workflow is: use the state queries to form a hypothesis ("this drop rule should be blocking it"), then use tcpdump to **prove or disprove** it ("the packet does/doesn't actually arrive / does/doesn't get a reply"). Configuration suggests; packets confirm. When the two disagree, the packets win.
</details>

---

## Homework

Build a deliberately broken namespace pair with **two simultaneous faults** — for example: (1) the receiver's interface has the **wrong subnet mask** so the peer is off-link, and (2) a **drop rule** in the receiver's input chain. Then debug it **strictly in the order of the toolchain table** (link → addr → neigh → route → fdb → nft → conntrack → ss → tcpdump), writing down for each tool whether its answer was "good" or "this is a fault." Show how the methodical order finds *both* faults instead of stopping at the first.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Setup (illustrative):** peers `10.2.2.1/24` and `10.2.2.2`, but assign the second one `10.2.2.2/30` (wrong mask, so `.1` may fall outside its on-link range depending on addressing — use a mask that makes the peer off-link, e.g. give one side a `/30` that excludes the other), and add `nft ... input ... drop` on the receiver.

**Walking the toolchain in order:**

1. `ip link show` → **good** (state UP, LOWER_UP on both).
2. `ip addr show` → **fault #1 found**: the receiver shows `/30` (or otherwise wrong prefix); the peer's address isn't in the on-link subnet. Note it, **but keep going** — there may be more.
3. `ip neigh show` → likely `FAILED`/incomplete for the peer, *consistent with* the wrong mask (off-link, ARP can't resolve directly). Symptom, not a new root cause.
4. `ip route get <peer>` → shows no on-link route / routes via an unexpected path — corroborates the addressing fault.
5. **Fix #1** (correct the mask to `/24`), re-test: now `ip neigh` resolves `REACHABLE` and ICMP *should* work — but a TCP service still fails. Don't declare victory.
6. `bridge fdb show` → n/a here (no bridge) → good.
7. `nft list ruleset` → **fault #2 found**: a `drop` rule in the input chain. (Add a temporary `log` rule, or use tcpdump in the next step, to confirm packets arrive but get dropped.)
8. `conntrack -L` → flow shows up but no reply / INVALID, consistent with the drop.
9. `ss -tulpn` → service *is* listening (rules out "nothing there"), so the loss is the firewall, not a missing socket.
10. `tcpdump` on the receiver → **ground truth**: SYNs arrive but get no response → confirms the input drop, not transit.
    **Fix #2** (remove/adjust the drop rule) → connection succeeds.

**The point:** had I stopped at the **first** fault (the mask), I'd have "fixed it," seen ping work, and still had a broken service — then been confused. Walking the **entire** layered sequence and recording good/fault at each step surfaces **both** independent problems. Methodical order beats guessing precisely because it doesn't let one found fault hide another.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 47 — Network Debugging Methodology →](lesson-47-debugging-methodology){: .btn .btn-primary }
