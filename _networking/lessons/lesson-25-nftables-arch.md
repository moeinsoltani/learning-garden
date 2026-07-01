---
title: "Lesson 25 — nftables Architecture"
nav_order: 25
parent: "Phase 8: nftables Firewall"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 25: nftables Architecture

## Concept

**nftables** is the modern packet-classification framework in the Linux kernel — the firewall. It replaced **iptables** (and its siblings ip6tables, arptables, ebtables) with a single, unified, more efficient tool. Before you write a single rule, you need the mental model: **tables → chains → rules**, with chains attached to **hooks** in the packet's journey through the kernel.

```
   packet arrives
        │
   ┌────▼─────┐   ┌──────────┐   ┌───────────┐   ┌───────────┐
   │prerouting│──►│  routing  │──►│   input    │──►│  local    │
   └──────────┘   │ decision  │   │ (to host)  │   │ process    │
        │         └─────┬─────┘   └───────────┘   └─────┬─────┘
        │               │                                │
        │          ┌────▼────┐                      ┌────▼─────┐
        │          │ forward  │─────────────────────►│postrouting│──► packet leaves
        │          │(transit) │                      └──────────┘
        │          └──────────┘
   (a packet for THIS host goes prerouting→input; a packet passing THROUGH
    goes prerouting→forward→postrouting; a packet FROM this host goes
    output→postrouting)
```

A **hook** is a well-defined point in this path where the kernel will call your chain to make a decision.

---

## How it works

The building blocks:

- **Table** — a namespace/container for chains and sets, scoped to an **address family**.
- **Chain** — an ordered list of rules. A *base chain* attaches to a hook with a type and priority; a *regular chain* is just a jump target.

{: .note }
> **Base chain vs. regular chain**
> A **base chain** is wired into the packet's path: it names a `hook`, a `type`, and a `priority` (and an optional `policy`), and the **kernel runs it automatically** when a packet reaches that hook. It's the *entry point* into your ruleset — every base chain in this lesson (`input`, `forward`, `output`) is one.
> A **regular chain** has *none* of `type`/`hook`/`priority`/`policy`. The kernel never calls it on its own — you reach it only with `jump` or `goto` from another chain. It's like a function: a reusable container of rules you call to keep big rulesets organized (e.g. one chain per service or zone). Example: `nft add chain inet filter ssh_rules` (no hook) then `... input tcp dport 22 jump ssh_rules`.
- **Rule** — a match (e.g. `tcp dport 22`) plus a verdict (`accept`, `drop`, `jump`, etc.).
- **Hook** — where in the path the base chain runs: `prerouting`, `input`, `forward`, `output`, `postrouting` (plus `ingress` for very early/XDP-like processing).
- **Priority** — when multiple base chains share a hook, lower priority numbers run first. NAT and filter chains use conventional priorities so they interleave correctly.

**Address families** decide which traffic the table sees:

| Family | Sees |
|---|---|
| `ip` | IPv4 only |
| `ip6` | IPv6 only |
| `inet` | Both IPv4 and IPv6 (the modern default — write rules once) |
| `arp` | ARP packets |
| `bridge` | Frames traversing a [bridge](lesson-11-bridges) (L2) |
| `netdev` | Earliest ingress, per-interface (used for XDP-like filtering) |

The flow a packet takes depends on its destination: **to this host** → prerouting → input; **through this host** → prerouting → forward → postrouting; **from this host** → output → postrouting. Knowing which hook a packet hits is the whole game — a rule in `input` never sees forwarded traffic, and vice versa.

{: .note }
> **Why nftables replaced iptables**
> iptables had separate tools and kernels paths for IPv4, IPv6, ARP, and bridging, with fixed built-in tables/chains and a rule-matching model that re-evaluated everything linearly. nftables unifies all of that into one framework with one syntax, adds first-class **sets and maps** (O(1) lookups instead of thousands of linear rules — Lesson 27), lets you create only the chains you need, supports atomic ruleset replacement, and uses a small virtual machine for matching. It's faster, more expressive, and less error-prone. iptables commands today are usually translated to nftables under the hood (`iptables-nft`).

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `nft list ruleset` | Show the entire ruleset |
| `nft add table inet filter` | Create a table in the `inet` family |
| `nft 'add chain inet filter input { type filter hook input priority 0 ; policy accept ; }'` | Create a base chain on the input hook |
| `nft add rule inet filter input tcp dport 22 accept` | Add a rule |
| `nft -f rules.nft` | Load a ruleset from a file (atomic) |
| `nft list table inet filter` | Show one table |
| `nft flush ruleset` | Wipe everything (careful!) |
| `nft monitor` | Watch ruleset changes live |

---

## Lab

We'll build a minimal but complete ruleset in a namespace and observe which hook each kind of traffic hits.

### Step 1 — A namespace to experiment safely

```bash
$ sudo ip netns add fw
$ sudo ip netns exec fw ip link set lo up
$ sudo ip netns exec fw ip link add eth0 type dummy
$ sudo ip netns exec fw ip link set eth0 up
$ sudo ip netns exec fw ip addr add 10.0.0.1/24 dev eth0
```

### Step 2 — Create a table and the three filter base chains

```bash
$ sudo ip netns exec fw nft add table inet filter
$ sudo ip netns exec fw nft 'add chain inet filter input   { type filter hook input   priority 0 ; policy accept ; }'
$ sudo ip netns exec fw nft 'add chain inet filter forward { type filter hook forward priority 0 ; policy accept ; }'
$ sudo ip netns exec fw nft 'add chain inet filter output  { type filter hook output  priority 0 ; policy accept ; }'
```

Each base chain names a `hook` and a `priority`. `policy accept` is the default verdict if no rule matches.

{: .note }
> **The word "filter" appears twice — they mean different things**
> In `add chain inet filter input { type filter hook input ... }`:
> - The **first `filter`** is the **table name** — an arbitrary label you picked (it could be `myfirewall`). By convention the filtering table is just called `filter`.
> - **`type filter`** is the **chain type** — a real keyword telling the kernel *what kind of work* this chain does. nftables has three chain types: **`filter`** (accept/drop/reject decisions — the common case), **`nat`** (address/port translation — Lesson 22), and **`route`** (re-trigger routing for locally-generated packets, like the old `mangle` OUTPUT). So `type filter` declares "this chain makes filtering verdicts."
>
> Only **base chains** have a `type`; regular chains don't.

### Step 3 — Inspect the structure

```bash
$ sudo ip netns exec fw nft list ruleset
table inet filter {
    chain input   { type filter hook input   priority filter; policy accept; }
    chain forward { type filter hook forward priority filter; policy accept; }
    chain output  { type filter hook output  priority filter; policy accept; }
}
```

### Step 4 — Add a counting rule to see which hook fires

```bash
# Count packets the host receives for itself (input hook)
$ sudo ip netns exec fw nft add rule inet filter input counter

$ sudo ip netns exec fw ping -c 3 10.0.0.1     # ping our own address
$ sudo ip netns exec fw nft list chain inet filter input
chain input {
    type filter hook input priority filter; policy accept;
    counter packets 3 bytes 252      # the 3 echo replies hit the input hook
}
```

The counter proves traffic *to* the host traverses the **input** chain. Loopback-bound traffic to your own address comes back in via input.

### Step 5 — Demonstrate input vs output separation

```bash
$ sudo ip netns exec fw nft add rule inet filter output counter
$ sudo ip netns exec fw ping -c 2 10.0.0.1
$ sudo ip netns exec fw nft list table inet filter | grep counter
# input  counter ...   ← echo replies arriving
# output counter ...   ← echo requests leaving
```

Outbound packets hit `output`; inbound hit `input`. A rule in one never sees the other's traffic — the core reason you must put a rule on the *right* hook.

### Step 6 — Load a ruleset from a file (atomic replace)

```bash
$ cat > /tmp/rules.nft <<'EOF'
flush ruleset
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        iif "lo" accept
        ip protocol icmp accept
    }
}
EOF
$ sudo ip netns exec fw nft -f /tmp/rules.nft
```

`nft -f` applies the whole file **atomically** — either all of it loads or none does. This is how production firewalls are managed: one declarative file, version-controlled, loaded as a unit.

### Step 7 — Clean up

```bash
$ sudo ip netns delete fw
```

---

## Further Reading

| Topic | Link |
|---|---|
| nftables | [Wikipedia — nftables](https://en.wikipedia.org/wiki/Nftables) |
| netfilter hooks | [Wikipedia — netfilter](https://en.wikipedia.org/wiki/Netfilter) |
| nftables wiki | [wiki.nftables.org](https://wiki.nftables.org/) |
| `nft` | [man7.org — nft(8)](https://man7.org/linux/man-pages/man8/nft.8.html) |

---

## Checkpoint

**Q1. What is a hook? Describe the path a packet takes through nftables from arrival to delivery, for (a) a packet destined for this host and (b) a packet being forwarded through it.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A **hook** is a defined point in the kernel's packet-processing path where netfilter will invoke any base chains attached there to make a filtering/NAT decision. The five main hooks are prerouting, input, forward, output, and postrouting.

**(a) Packet destined for this host:** it enters at **prerouting** (before the routing decision), the kernel routes it and decides it's for a local address, so it goes to the **input** hook, and if accepted is delivered to the local process. So: prerouting → (routing) → input → local delivery.

**(b) Packet being forwarded (transit):** it enters at **prerouting**, the routing decision determines it's destined elsewhere (not local), so instead of input it goes to the **forward** hook, and if accepted continues to **postrouting** before leaving the box. So: prerouting → (routing) → forward → postrouting → out.

(For completeness, a packet *originating* from this host's own processes starts at **output**, then **postrouting**.) The practical consequence: a rule placed in `input` only sees traffic *to* the host, and a rule in `forward` only sees traffic *through* it — putting a rule on the wrong hook is a common reason a firewall "doesn't work."
</details>

---

**Q2. What does the `inet` address family give you that separate `ip` and `ip6` tables don't?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The `inet` family lets a single table and its chains handle **both IPv4 and IPv6** traffic, so you write your rules once instead of maintaining two parallel rulesets (one in an `ip` table for v4, one in an `ip6` table for v6). This avoids the classic mistake of securing IPv4 while leaving IPv6 wide open (or vice versa) because someone forgot to duplicate a rule. Within an `inet` chain you can still write family-specific matches when needed (e.g., `ip saddr` vs `ip6 saddr`), but protocol-agnostic logic (port numbers, conntrack state, interface matching) applies to both address families automatically. It's the modern recommended default precisely because most policy ("allow SSH, allow established, drop the rest") is identical for v4 and v6, and `inet` keeps them in sync by construction.
</details>

---

**Q3. Why is loading a ruleset with `nft -f rules.nft` (atomic) safer than running many individual `nft add rule` commands?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`nft -f` applies the entire file as a single **atomic transaction**: the kernel either commits the whole new ruleset or, if any line has an error, rejects all of it and leaves the existing ruleset untouched. There is never an intermediate moment where the firewall is half-configured.

Running many separate `nft add rule` commands has two hazards: (1) **transient insecure states** — between commands the firewall is partially built, so for a window it might be dropping legitimate traffic or, worse, allowing traffic a later rule was supposed to block; and (2) **partial failure** — if command 7 of 12 fails, you're left with an inconsistent, half-applied policy and have to manually reason about what state you're in. The atomic file approach is also declarative and version-controllable: the file *is* the complete intended policy, reviewable as a unit and reproducible. This is why production firewalls are managed as a single `.nft` file loaded atomically (often via `systemctl` and `/etc/nftables.conf`), not as a sequence of imperative commands.
</details>

---

## Homework

In a namespace, create an `inet filter` table with `input`, `forward`, and `output` base chains, each with a `counter` rule. Generate three kinds of traffic: (a) ping your own local address, (b) have the namespace act as a router and forward a packet between two veths, and (c) make an outbound connection. Predict which counter(s) increment for each, then verify. Explain any surprises (especially around loopback and locally-destined traffic).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Predicted and observed:

- **(a) Ping your own local address (e.g. `10.0.0.1` assigned to the host):** both the **output** and **input** counters increment. The echo *request* you send leaves via the output hook, and because the destination is a local address, the kernel loops it back and the echo *reply* (and even the request's local delivery) arrives via the input hook. People are often surprised that pinging your *own* address touches input — but locally-destined traffic genuinely traverses the input hook on the way to the local socket. The forward counter stays at zero (nothing is transiting).

- **(b) Forwarding between two veths:** only the **forward** counter increments. A packet that arrives on one interface and is routed out another (with `ip_forward=1`) takes prerouting → forward → postrouting and never touches input or output, because it's neither destined for nor originated by this host. This is the key separation: transit traffic = forward only.

- **(c) Outbound connection from the namespace itself:** the **output** counter increments for the packets you send, and the **input** counter increments for the replies coming back. Locally-generated traffic uses output → postrouting on the way out, and returning packets use prerouting → input on the way back.

The big takeaway / surprise: **input and output are about *this host* as an endpoint**, while **forward is about *transit*** — and traffic to your own local addresses (including loopback) does pass through input/output, not forward. Choosing the correct hook for a rule depends entirely on whether you're filtering traffic to/from the host or through it.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 26 — Packet Filtering with nft →](lesson-26-nft-filtering){: .btn .btn-primary }
