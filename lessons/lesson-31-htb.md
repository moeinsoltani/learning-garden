---
title: "Lesson 31 — Classful qdiscs: HTB"
nav_order: 31
parent: "Phase 9: Traffic Control"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 31: Classful qdiscs — HTB

## Concept

Classless qdiscs treat all traffic the same. **HTB** (Hierarchical Token Bucket) lets you carve a link's bandwidth into a *tree of classes*, each with its own guaranteed rate, and let classes **borrow** unused bandwidth from each other. This is how you say "web traffic gets at least 5 Mbit, backups get at least 2 Mbit, and whichever is idle, the other may use its share."

```
   root 1: (10 Mbit total)
    └── 1:1  parent (10 Mbit)
         ├── 1:10  HTTP   rate 5mbit  ceil 10mbit
         └── 1:20  bulk   rate 2mbit  ceil 10mbit
   Guarantees: HTTP ≥5, bulk ≥2. Ceil: either may burst to 10 if the other is idle.
```

The magic word is **borrowing**: `rate` is the *guarantee* (always available to that class), `ceil` is the *maximum* it may reach by borrowing unused capacity from its parent. Idle bandwidth isn't wasted.

---

## How it works

HTB is a tree of **classes**, identified by `major:minor` handles:

- The **root qdisc** (`handle 1:`) is the top.
- **Classes** hang off it (`classid 1:1`, `1:10`, `1:20`), forming a hierarchy. A parent class's rate bounds the sum its children can *guarantee*.
- Each class has:
  - **`rate`** — the guaranteed minimum bandwidth (assured even under contention).
  - **`ceil`** — the absolute maximum it can use, including borrowed bandwidth.
- **Borrowing:** when a class needs more than its `rate` and its siblings aren't using theirs, it can borrow from the parent up to its `ceil`. When siblings get busy, borrowing stops and each class falls back to its guarantee.

You also need **filters** (next lesson) to *classify* packets into the right leaf class — otherwise everything lands in the `default` class.

```
   tc qdisc add dev eth0 root handle 1: htb default 20
   tc class add dev eth0 parent 1:  classid 1:1  htb rate 10mbit
   tc class add dev eth0 parent 1:1 classid 1:10 htb rate 5mbit ceil 10mbit
   tc class add dev eth0 parent 1:1 classid 1:20 htb rate 2mbit ceil 10mbit
```

{: .note }
> **rate vs ceil — guarantee vs ceiling**
> Think of `rate` as the floor you're *promised* and `ceil` as the ceiling you're *allowed* to reach. Under full contention every class gets at least its `rate` (the sum of child rates should not exceed the parent's rate). When there's slack — some class isn't using its share — other classes borrow it, expanding up toward their `ceil`. Setting `ceil = rate` makes a class a hard limit (no borrowing); setting `ceil` higher than `rate` enables borrowing. This dual knob is what makes HTB both *fair under load* and *efficient when idle*.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `tc qdisc add dev <if> root handle 1: htb default <minor>` | Create the HTB root; unclassified traffic → `default` class |
| `tc class add dev <if> parent 1: classid 1:1 htb rate 10mbit` | Add a parent class |
| `tc class add dev <if> parent 1:1 classid 1:10 htb rate 5mbit ceil 10mbit` | Add a leaf class with guarantee + ceiling |
| `tc -s class show dev <if>` | Show per-class stats (sent bytes, rate, borrows) |
| `tc qdisc add dev <if> parent 1:10 handle 10: fq_codel` | Attach a child qdisc to a leaf class |
| `tc class change dev <if> ... htb rate ...` | Adjust a class live |

---

## Lab

We'll shape a veth into two classes and watch them compete — one starving, then sharing, then borrowing.

### Step 1 — Veth between two namespaces

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
```

### Step 2 — Build the HTB hierarchy on a's egress

```bash
$ sudo ip netns exec a tc qdisc add dev va root handle 1: htb default 20
$ sudo ip netns exec a tc class add dev va parent 1:  classid 1:1  htb rate 10mbit
$ sudo ip netns exec a tc class add dev va parent 1:1 classid 1:10 htb rate 5mbit ceil 10mbit
$ sudo ip netns exec a tc class add dev va parent 1:1 classid 1:20 htb rate 2mbit ceil 10mbit
```

We'll classify port 5201 (iperf3) traffic into 1:10 and everything else into the default 1:20. Add a quick filter (full filter coverage is Lesson 32):

```bash
$ sudo ip netns exec a tc filter add dev va protocol ip parent 1: \
    u32 match ip dport 5201 0xffff flowid 1:10
```

### Step 3 — Single flow can borrow up to ceil

With only class 1:10 active and 1:20 idle, 1:10 borrows the idle bandwidth and reaches its `ceil` of 10mbit:

```bash
$ sudo ip netns exec b iperf3 -s -p 5201 &
$ sudo ip netns exec a iperf3 -c 10.0.0.2 -p 5201 -t 5
# ~10 Mbit/s — class 1:10 (guaranteed 5) borrowed the idle 5 from 1:20 up to ceil 10
```

This demonstrates the homework answer to "what happens to 1:20 when 1:10 is idle" in reverse: the active class expands into the idle class's share.

### Step 4 — Two competing flows fall back to guarantees

Now run a second flow on a different port (lands in default 1:20) at the same time:

```bash
# Terminal A: port 5201 -> class 1:10 (guaranteed 5mbit)
$ sudo ip netns exec b iperf3 -s -p 5201 &
$ sudo ip netns exec a iperf3 -c 10.0.0.2 -p 5201 -t 15 &

# Terminal B: port 5202 -> default class 1:20 (guaranteed 2mbit)
$ sudo ip netns exec b iperf3 -s -p 5202 &
$ sudo ip netns exec a iperf3 -c 10.0.0.2 -p 5202 -t 15 &
```

Under contention, each class is held to (at least) its guarantee and the link is shared roughly 5:2 in the proportion of their rates (totaling the 10mbit parent). Neither can borrow much because both are busy.

### Step 5 — Watch per-class stats and borrowing

```bash
$ sudo ip netns exec a tc -s class show dev va
class htb 1:10 ... rate 5Mbit ceil 10Mbit
  Sent X bytes ... 
  lended: ... borrowed: ...      # borrowed = tokens taken from parent
class htb 1:20 ... rate 2Mbit ceil 10Mbit
  Sent Y bytes ...
```

The `borrowed`/`lended` counters show bandwidth moving between classes — the heart of HTB.

### Step 6 — Clean up

```bash
$ sudo ip netns delete a b
```

---

## Further Reading

| Topic | Link |
|---|---|
| HTB | [man7.org — tc-htb(8)](https://man7.org/linux/man-pages/man8/tc-htb.8.html) |
| Hierarchical token bucket | [Wikipedia — Token bucket](https://en.wikipedia.org/wiki/Token_bucket) |
| Linux Advanced Routing & Traffic Control | [lartc.org](https://lartc.org/) |
| `tc` | [man7.org — tc(8)](https://man7.org/linux/man-pages/man8/tc.8.html) |

---

## Checkpoint

**Q1. In the hierarchy where 1:10 has `rate 5mbit ceil 10mbit` and 1:20 has `rate 2mbit ceil 10mbit` on a 10mbit parent, what happens to class 1:20's throughput when class 1:10 is idle?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
When 1:10 is idle, class 1:20 can **borrow** the unused bandwidth and burst up to its `ceil` of 10mbit — far above its 2mbit guarantee. HTB's whole point is that guarantees (`rate`) are reserved only when needed: if a class isn't using its share, that capacity isn't wasted; sibling classes may borrow it from the parent up to their own `ceil`. So with 1:10 silent, 1:20 effectively gets the full 10mbit link. The moment 1:10 becomes active again, borrowing is reclaimed: 1:10 gets its guaranteed 5mbit back, and 1:20 falls back toward its 2mbit guarantee (the two then share according to their rates, with any remaining slack still borrowable). This dynamic — guarantee under contention, borrow when idle — is exactly why HTB is preferred over hard per-class caps: it's both fair and efficient.
</details>

---

**Q2. What is the difference between a class's `rate` and its `ceil`?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`rate` is the class's **guaranteed minimum** bandwidth — the amount it is always assured of getting, even when every other class is fully busy and the link is saturated. The sum of sibling `rate`s should not exceed the parent's rate, so the guarantees are actually deliverable.

`ceil` is the class's **maximum** bandwidth — the hard upper bound it can reach by *borrowing* unused capacity from its parent/siblings when they're idle. A class may expand above its `rate` up to its `ceil` whenever there's slack to borrow.

So `rate` = the floor you're promised; `ceil` = the ceiling you're allowed to reach. If `ceil == rate`, the class can't borrow and behaves as a hard rate limit. If `ceil > rate`, the class gets its guarantee under contention but can opportunistically use more when the link is underutilized. Tuning these two numbers per class is how you express "guarantee this much, but let it burst to that much if capacity is free."
</details>

---

**Q3. Why do you also need tc *filters* with an HTB setup, and what happens to a packet that matches no filter?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
HTB defines the *class hierarchy* (the buckets of bandwidth), but it doesn't know *which packets belong in which class*. **Filters** are the classifier: they match packet attributes (port, IP, protocol, mark, etc.) and assign each packet a `flowid` pointing at a leaf class. Without filters, every packet would land in the same place and the multi-class structure would be pointless — you'd have buckets but no way to sort traffic into them.

A packet that matches **no filter** goes to the class named by the qdisc's **`default`** parameter (e.g., `htb default 20` sends unclassified traffic to class 1:20). This is why you always specify a default class: it's the catch-all for anything your filters didn't explicitly classify. If you set a `default` to a class that doesn't exist (or forget it), unclassified traffic can end up unshaped or dropped, which is a common misconfiguration. So the division of labor is: HTB classes = how bandwidth is partitioned; filters = how packets are sorted into those partitions; `default` = where the leftovers go.
</details>

---

## Homework

Build a three-class HTB hierarchy on a shaped link: `interactive` (rate 3mbit, ceil 10mbit), `bulk` (rate 5mbit, ceil 10mbit), and `default` (rate 2mbit, ceil 10mbit). Classify SSH (port 22) into interactive and a large transfer into bulk. Run them simultaneously and confirm interactive stays responsive (low latency, gets its guarantee) even while bulk saturates the link. Measure with ping (over the interactive class) during a bulk iperf3. Explain how HTB protects latency-sensitive traffic.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Setup: `htb default 30` root at, say, 10mbit; class `1:10 interactive rate 3mbit ceil 10mbit`, `1:20 bulk rate 5mbit ceil 10mbit`, `1:30 default rate 2mbit ceil 10mbit`. Filters: `dport 22` (or your interactive marker) → 1:10; the bulk transfer's port → 1:20. Attach `fq_codel` as the leaf qdisc on the interactive class to keep its own queue latency low.

**What you observe:** while bulk iperf3 saturates the link, ping/SSH over the interactive class stays responsive — low, stable RTT — because the interactive class is *guaranteed* its 3mbit and its packets are dequeued promptly rather than sitting behind the bulk flow's large backlog. Without HTB, the bulk transfer would fill a single shared queue and interactive packets would wait behind megabytes of bulk data (bufferbloat), spiking latency into hundreds of milliseconds.

**Why HTB protects latency-sensitive traffic:** HTB gives each class its **own queue and its own guaranteed service rate**. Even when bulk wants the whole link, the scheduler ensures the interactive class gets serviced at (at least) its guaranteed rate, so its small packets don't get stuck behind the bulk backlog — they're pulled from a separate queue on schedule. The bulk class can still *borrow* idle capacity (efficiency), but the instant interactive has traffic, its guarantee is honored (protection). Combining HTB (per-class bandwidth guarantee) with a low-latency leaf qdisc like fq_codel (keeps each class's queue short) is the standard recipe for "keep VoIP/SSH/gaming snappy while big downloads run." The key idea: isolation into separate queues with guaranteed rates prevents one heavy flow from monopolizing the queue and inflating everyone else's latency.
</details>
