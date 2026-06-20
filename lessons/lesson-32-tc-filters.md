---
title: "Lesson 32 — tc Filters"
nav_order: 32
parent: "Phase 9: Traffic Control"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 32: tc Filters

## Concept

An HTB hierarchy defines *buckets* of bandwidth; **filters** decide *which packets go in which bucket*. A filter is a classifier: it matches packet fields and assigns a `flowid` pointing at a leaf class. Without filters, your carefully-built class tree does nothing — everything falls into the default class.

```
   packet → [ tc filter ] → flowid 1:10  → HTB class 1:10 (HTTP, 5mbit)
                          → flowid 1:20  → HTB class 1:20 (bulk, 2mbit)
                          → (no match)   → default class
```

There are several filter *types*, trading flexibility for performance and hardware-offload capability.

---

## How it works

The three filter types you should know:

| Type | Matches on | Notes |
|---|---|---|
| **`u32`** | Raw bytes at offsets in the packet (IP/TCP header fields). | Maximally flexible, works everywhere, but you specify byte offsets/masks — low-level and fiddly. |
| **`flower`** | Structured L2/L3/L4 fields (src/dst IP, ports, protocol, VLAN, etc.) by name. | Cleaner syntax *and* **hardware-offloadable** — the NIC can do the classification. |
| **`bpf`** | Arbitrary logic in a compiled BPF program. | Most powerful; you write code for classification decisions. |

A `u32` filter for "TCP destination port 80":

```
tc filter add dev eth0 protocol ip parent 1: u32 \
    match ip dport 80 0xffff flowid 1:10
#         └ field    └value └mask  └ target class
```

The same with `flower`:

```
tc filter add dev eth0 protocol ip parent 1: flower \
    ip_proto tcp dst_port 80 action goto chain 1
```

`flower` reads almost like a firewall rule, and crucially it maps to the structured match capabilities of modern NICs, so the hardware can classify and steer packets without the CPU — essential at high speed.

{: .note }
> **Why hardware offload favors `flower` over `u32`**
> `u32` matches arbitrary byte offsets with masks — extremely general, but NIC hardware classifiers don't think in "byte 20, mask 0xffff." They think in *structured fields*: "this is the TCP destination port." `flower` expresses matches in exactly those structured terms (src/dst IP, L4 ports, VLAN, etc.), which map directly onto the NIC's flow-classification tables. So a `flower` filter can be **offloaded** to capable hardware (with `skip_sw`), letting the NIC steer or drop flows at line rate before the CPU is involved. `u32`'s generality is precisely what prevents clean hardware offload. Choose `flower` when you want speed and offload; `u32` when you need to match something `flower` can't express.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `tc filter add dev <if> protocol ip parent 1: u32 match ip dport 80 0xffff flowid 1:10` | u32: classify TCP/UDP dport 80 into class 1:10 |
| `tc filter add dev <if> protocol ip parent 1: u32 match ip src 10.0.0.5 flowid 1:20` | u32: classify by source IP |
| `tc filter add dev <if> protocol ip parent 1: flower ip_proto tcp dst_port 80 flowid 1:10` | flower: same match, structured syntax |
| `tc filter add dev <if> parent 1: bpf obj classifier.o sec tc flowid 1:10` | bpf: classify via a compiled program |
| `tc filter show dev <if>` | List filters on an interface |
| `tc -s filter show dev <if>` | Filter stats (match counts) |

---

## Lab

We'll reuse an HTB setup and classify two traffic types (two ports) into two classes, then verify with iperf3 that each lands in the right bucket.

### Step 1 — Veth + HTB hierarchy

```bash
$ sudo ip netns add a
$ sudo ip netns add b
$ sudo ip link add va type veth peer name vb
$ sudo ip link set va netns a
$ sudo ip link set vb netns b
$ sudo ip netns exec a ip link set va up
$ sudo ip netns exec a ip addr add 10.0.0.1/24 dev va
$ sudo ip netns exec b ip link set vb up
$ sudo ip netns exec b ip addr add 10.0.0.2/24 dev vb

# HTB: 10mbit parent, two leaves
$ sudo ip netns exec a tc qdisc add dev va root handle 1: htb default 20
$ sudo ip netns exec a tc class add dev va parent 1:  classid 1:1  htb rate 10mbit
$ sudo ip netns exec a tc class add dev va parent 1:1 classid 1:10 htb rate 8mbit ceil 10mbit
$ sudo ip netns exec a tc class add dev va parent 1:1 classid 1:20 htb rate 2mbit ceil 10mbit
```

### Step 2 — Classify port 5201 into the fast class with u32

```bash
$ sudo ip netns exec a tc filter add dev va protocol ip parent 1: \
    u32 match ip dport 5201 0xffff flowid 1:10
# Everything else (no match) → default class 1:20
```

### Step 3 — Verify the classification

Run two iperf3 flows: port 5201 (→ 1:10, 8mbit) and port 5202 (→ default 1:20, 2mbit), simultaneously.

```bash
$ sudo ip netns exec b iperf3 -s -p 5201 &
$ sudo ip netns exec b iperf3 -s -p 5202 &
$ sudo ip netns exec a iperf3 -c 10.0.0.2 -p 5201 -t 15 &   # should get ~8mbit
$ sudo ip netns exec a iperf3 -c 10.0.0.2 -p 5202 -t 15 &   # should get ~2mbit
```

Under contention the port-5201 flow gets ~8 Mbit (class 1:10) and the port-5202 flow ~2 Mbit (default 1:20). The filter steered each flow into the right bucket.

### Step 4 — Confirm with filter stats

```bash
$ sudo ip netns exec a tc -s filter show dev va
filter parent 1: protocol ip pref ... u32 ...
  match ... dport 5201 ...
  ... (packet/byte counts that prove this filter is matching the 5201 flow)
$ sudo ip netns exec a tc -s class show dev va
class htb 1:10 ... Sent <large> bytes ...     # the classified flow
class htb 1:20 ... Sent <small> bytes ...     # the default flow
```

The per-class `Sent` counters confirm the split: most bytes flowed through 1:10 (the classified flow), the rest through the default.

### Step 5 — (Optional) Same match with flower

```bash
# Remove the u32 filter and express it as flower instead
$ sudo ip netns exec a tc filter del dev va parent 1:
$ sudo ip netns exec a tc filter add dev va protocol ip parent 1: \
    flower ip_proto tcp dst_port 5201 flowid 1:10
```

Re-run the test — same result, but the rule is expressed in structured, offload-friendly terms.

### Step 6 — Clean up

```bash
$ sudo ip netns delete a b
```

---

## Further Reading

| Topic | Link |
|---|---|
| tc filters / classifiers | [man7.org — tc(8)](https://man7.org/linux/man-pages/man8/tc.8.html) |
| u32 classifier | [man7.org — tc-u32(8)](https://man7.org/linux/man-pages/man8/tc-u32.8.html) |
| flower classifier | [man7.org — tc-flower(8)](https://man7.org/linux/man-pages/man8/tc-flower.8.html) |
| Hardware offload (switchdev) | [kernel.org — switchdev](https://docs.kernel.org/networking/switchdev.html) |

---

## Checkpoint

**Q1. What is the key advantage of `flower` over `u32` for hardware NIC offload?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`flower` expresses matches in terms of **structured, named packet fields** — source/destination IP, L4 ports, protocol, VLAN ID, and so on — which is exactly how modern NIC hardware classification engines are organized. That structural correspondence means a `flower` filter can be **offloaded to the NIC**, so the hardware classifies (and steers, drops, or marks) packets at line rate before the CPU ever touches them. `u32`, by contrast, matches **arbitrary byte offsets and masks** in the raw packet — maximally general, but NIC classifiers don't operate on "byte 22 with mask 0xffff"; they operate on recognized fields. That generality is precisely what prevents `u32` from mapping cleanly onto hardware tables, so it generally can't be offloaded and runs in software on the CPU. So for high-throughput scenarios where you want the NIC to do the work, `flower` is the right choice; `u32` remains useful when you must match something `flower` can't express, at the cost of staying in software.
</details>

---

**Q2. In an HTB setup, what is the relationship between a filter's `flowid` and an HTB class?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A filter's **`flowid`** is the *target* — it names the HTB leaf class that matching packets should be placed into. The filter does the *matching* (e.g., "TCP dport 5201"), and the `flowid 1:10` says "send packets that match this into class 1:10." So the two work together: HTB defines the class tree and each class's bandwidth (`rate`/`ceil`), while filters connect real packets to those classes by setting their `flowid` to the appropriate `classid`. A packet's journey is: filter evaluates it → assigns `flowid 1:10` → packet is enqueued in HTB class 1:10 → that class's shaping/scheduling applies. If a `flowid` points at a class that doesn't exist, classification fails; if no filter matches, the packet goes to the qdisc's `default` class. Essentially, `flowid` is the wire that plugs a classifier's decision into a specific bandwidth bucket.
</details>

---

**Q3. When would you reach for a `bpf` filter instead of `u32` or `flower`?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You reach for a **`bpf`** filter when your classification logic is too complex or stateful to express as a simple field/offset match. `u32` and `flower` are *declarative* matchers — they compare specific header fields against values/masks. A `bpf` classifier lets you run **arbitrary compiled program logic** on each packet: you can parse deep into payloads, combine many conditions, maintain state in BPF maps (e.g., per-flow counters, dynamic bl/allowlists), do custom hashing, or implement classification decisions that no fixed field-match could capture. So use `bpf` for things like application-layer-aware classification, decisions based on accumulated state, or logic that would require an unwieldy pile of `u32`/`flower` rules. The trade-off is complexity (you must write, compile, and load a BPF program, and reason about the verifier's constraints) versus the simplicity of declarative matchers. Rule of thumb: `flower` for clean, offloadable field matches; `u32` for odd field matches `flower` can't express; `bpf` when you genuinely need *programmable* classification beyond field comparisons.
</details>

---

## Homework

Build an HTB hierarchy with three classes and use filters to classify by **source IP** rather than port: traffic from `10.0.0.10` → high-priority class, from `10.0.0.20` → bulk class, everything else → default. Write the filters with `u32`, then rewrite them with `flower`. Run simultaneous transfers from each source and confirm the bandwidth split. Compare the two filter syntaxes and explain which you'd deploy on a high-throughput production router and why.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**u32 version** (matching source IP via raw header field):
```
tc filter add dev <if> protocol ip parent 1: u32 match ip src 10.0.0.10/32 flowid 1:10
tc filter add dev <if> protocol ip parent 1: u32 match ip src 10.0.0.20/32 flowid 1:20
# unmatched → default class
```

**flower version** (same intent, structured):
```
tc filter add dev <if> protocol ip parent 1: flower src_ip 10.0.0.10 flowid 1:10
tc filter add dev <if> protocol ip parent 1: flower src_ip 10.0.0.20 flowid 1:20
```

**Verification:** run transfers sourced from each address simultaneously; the per-class `tc -s class show` counters confirm `10.0.0.10`'s traffic flowed through the high-priority class and `10.0.0.20`'s through bulk, each held to its class's rate under contention, with the rest in default.

**Comparison and production choice:**
- **Readability/maintainability:** `flower` wins clearly — `src_ip 10.0.0.10` is self-documenting, whereas `u32 match ip src ...` is lower-level and error-prone (especially for more complex matches involving offsets/masks).
- **Hardware offload:** `flower` is offloadable to capable NICs (with `skip_sw`/`skip_hw` controls), so on a high-throughput router the classification can run **in the NIC hardware at line rate**, freeing the CPU. `u32` generally runs in software on the CPU, which becomes a bottleneck at high packet rates.
- **Flexibility:** `u32` can match arbitrary fields `flower` doesn't model, so it remains the fallback for unusual matches.

**For a high-throughput production router I'd deploy `flower`** — it's clearer to maintain and, critically, it can be offloaded to hardware, keeping CPU usage low and throughput high. I'd only drop to `u32` (or `bpf`) for matches `flower` can't express. The general principle: prefer the most *structured* classifier your match allows, because structure is what enables both human readability and hardware acceleration.
</details>
