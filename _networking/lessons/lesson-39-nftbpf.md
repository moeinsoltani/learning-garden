---
title: "Lesson 39 — nftbpf: BPF in nftables"
nav_order: 39
parent: "Phase 12: nftables + BPF"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 39: nftbpf — Calling BPF Programs from nftables

## Concept

You now have two packet-processing toolkits. **nftables** (Phase 8) is great at *declarative policy*: "accept these ports, drop from these IPs, rate-limit this." **BPF** (Phase 11) is great at *arbitrary logic*: "match if bytes 40–48 of the payload equal this pattern." Each is awkward at the other's job. **nftbpf** bridges them — an nftables rule can call a **BPF program as a match expression**, so the BPF program decides "match / no match" and nftables decides what to *do* about it.

```
   packet → nftables chain
              │
              ├─ rule: meta bpf object-pinned /sys/fs/bpf/myprog  drop
              │        │
              │        ▼ nftables hands the skb to the BPF program
              │     BPF program inspects packet → returns a value
              │        │
              │   value > 0  → "match"  → apply the verdict (drop)
              │   value == 0 → "no match" → fall through to next rule
              ▼
```

The division of labor: **BPF answers a hard yes/no question about the packet; nftables owns the policy decision and the rest of the ruleset.** Think of the BPF program as a custom, programmable *match keyword* you've added to nftables' vocabulary.

---

## How it works

nftables exposes a `meta bpf` match that runs a BPF program of type **socket filter** (`BPF_PROG_TYPE_SOCKET_FILTER`) against the packet's `sk_buff`. The program returns an integer; nftables treats the return value as the match result — **non-zero means match** (continue applying the rule), **zero means no match** (the rule doesn't fire). Because it's a match, you can attach any normal nftables verdict (`drop`, `accept`, `jump`, counters, logging) to the rule.

The BPF program is typically **pinned** to the BPF filesystem first (Lesson 37), then referenced by path:

```
nft add rule inet filter input meta bpf object-pinned /sys/fs/bpf/myprog drop
```

**When to reach for this** instead of a plain nftables construct:

- A **named set** (Lesson 27) is the right tool for "is this IP/port in a list" — it's a hash lookup, fast and atomically updatable. Use a set for set membership.
- Use **nftbpf** when the match needs *logic nftables can't express*: deep/variable-offset payload inspection, parsing application data, computing something across multiple header fields, or stateful logic you've already written in BPF. The BPF program can do arbitrary (verified, bounded) computation that no combination of `nft` expressions could.

{: .note }
> **Match vs verdict — keep the split clear**
> The BPF program does **not** drop the packet itself in this model — it only returns "match / no match." The *drop/accept/jump* is the nftables verdict written on the rule. This keeps all the actual policy visible in `nft list ruleset`, with BPF supplying only the bits of matching logic that rules can't express. (Contrast XDP in Lesson 38, where the BPF program returns the verdict directly.)

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `bpftool prog load prog.o /sys/fs/bpf/myprog` | Load and pin a BPF program by path |
| `nft add rule ... meta bpf object-pinned <path> <verdict>` | Match with a pinned BPF program |
| `nft list ruleset` | Confirm the rule (and its BPF reference) is in place |
| `bpftool prog show pinned <path>` | Inspect the pinned program |

---

## Lab

We'll write a socket-filter BPF program that **matches UDP packets whose payload starts with a specific byte pattern**, then use it in an nftables rule to drop those packets — something a plain `nft` rule can't easily express for arbitrary payloads.

{: .note }
> **Availability note**
> `meta bpf object-pinned` requires a kernel and nftables build with this match compiled in; it is less common than the rest of nftables. If your lab kernel lacks it, read the lab for the model and use the alternative (a TC-BPF program, Lesson 36's TC hook) noted in the homework.

### Step 1 — The match program

```c
// nft_match.c — socket filter: return 1 (match) if UDP payload starts with 0xDE 0xAD
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/udp.h>
#include <bpf/bpf_helpers.h>

SEC("socket")
int match_deadbeef(struct __sk_buff *skb) {
    void *data = (void *)(long)skb->data, *end = (void *)(long)skb->data_end;
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > end) return 0;
    if (eth->h_proto != __builtin_bswap16(ETH_P_IP)) return 0;
    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > end) return 0;
    if (ip->protocol != IPPROTO_UDP) return 0;
    struct udphdr *udp = (void *)ip + ip->ihl * 4;
    if ((void *)(udp + 1) > end) return 0;
    unsigned char *p = (void *)(udp + 1);
    if ((void *)(p + 2) > end) return 0;
    if (p[0] == 0xDE && p[1] == 0xAD) return 1;   // match
    return 0;                                     // no match
}
char _license[] SEC("license") = "GPL";
```

### Step 2 — Compile and pin

```bash
$ clang -O2 -g -target bpf -c nft_match.c -o nft_match.o
$ sudo mount -t bpf bpf /sys/fs/bpf 2>/dev/null   # ensure bpffs is mounted
$ sudo bpftool prog load nft_match.o /sys/fs/bpf/match_deadbeef type socket_filter
$ sudo bpftool prog show pinned /sys/fs/bpf/match_deadbeef
... socket_filter  name match_deadbeef ...
```

### Step 3 — Build a table/chain and the BPF-matched rule

```bash
$ sudo nft add table inet filter
$ sudo nft 'add chain inet filter input { type filter hook input priority 0 ; }'
$ sudo nft add rule inet filter input meta bpf object-pinned /sys/fs/bpf/match_deadbeef counter drop
$ sudo nft list ruleset
table inet filter {
    chain input {
        type filter hook input priority 0; policy accept;
        meta bpf object-pinned "/sys/fs/bpf/match_deadbeef" counter packets 0 bytes 0 drop
    }
}
```

### Step 4 — Test it

```bash
# Send a UDP packet whose payload starts 0xDE 0xAD → should be dropped (counter rises)
$ printf '\xde\xad\xbe\xef' | nc -u -w1 127.0.0.1 9999
$ sudo nft list ruleset | grep counter        # packets/bytes incremented

# Send a non-matching payload → passes (counter unchanged)
$ printf 'hello' | nc -u -w1 127.0.0.1 9999
```

### Step 5 — Clean up

```bash
$ sudo nft delete table inet filter
$ sudo rm -f /sys/fs/bpf/match_deadbeef
```

---

## Further Reading

| Topic | Link |
|---|---|
| nftables wiki | [netfilter.org — nftables wiki](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page) |
| nft meta expressions | [man7.org — nft(8)](https://man7.org/linux/man-pages/man8/nft.8.html) |
| BPF program types | [kernel.org — BPF program types](https://docs.kernel.org/bpf/libbpf/program_types.html) |
| socket filters (classic context) | [kernel.org — networking filter](https://docs.kernel.org/networking/filter.html) |
| `bpftool prog` | [man7.org — bpftool-prog(8)](https://man7.org/linux/man-pages/man8/bpftool-prog.8.html) |

---

## Checkpoint

**Q1. When would you use nftbpf instead of an nftables named set?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Use a **named set** when the match is **set membership** — "is this source IP / destination port / IP range in this list?" A set is a hash (or interval) lookup: fast, memory-efficient, and atomically updatable from userspace, and it needs no BPF at all. That covers the large majority of "match against many values" cases.

Reach for **nftbpf** only when the match requires **logic that nftables expressions cannot represent**: inspecting payload bytes at variable offsets, parsing application-layer data, computing a relationship across several header fields, or reusing stateful matching logic you've already implemented in BPF. nftbpf lets a verified BPF program perform arbitrary bounded computation and report a yes/no result that nftables then acts on. In short: **set = membership test; nftbpf = arbitrary programmable match.** If a set (or any built-in nft expression) can express it, prefer that — it's simpler and faster.
</details>

---

**Q2. In the nftbpf model, what decides the packet's fate — the BPF program or the nftables rule? Explain the split.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The **nftables rule** decides the packet's fate. The BPF program acts only as a **match expression**: it inspects the packet and returns an integer that nftables interprets as "match" (non-zero) or "no match" (zero). If it matches, nftables applies whatever **verdict** is written on that rule (`drop`, `accept`, `jump`, plus counters/logging); if not, the rule doesn't fire and evaluation continues. So the BPF program answers "*does this packet qualify?*" and nftables answers "*and what do we do with qualifying packets?*". This keeps the actual policy visible in `nft list ruleset`, with BPF contributing only the hard matching logic. (This contrasts with XDP, where the BPF program returns the verdict — `XDP_DROP` etc. — itself.)
</details>

---

## Homework

Suppose your kernel lacks the `meta bpf` match. Re-implement the same effect — drop UDP packets whose payload starts with `0xDE 0xAD` — using a **TC (sched_cls) BPF program** attached on the ingress qdisc instead (Lesson 36 introduced the TC hook). Sketch the attach commands (`tc qdisc add dev <if> clsact`, `tc filter add dev <if> ingress bpf ...`) and explain the trade-off versus the nftbpf approach: what do you gain, and what do you lose in terms of where the policy is expressed and how visible it is?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**TC-BPF approach.** Add a `clsact` qdisc and attach the program as an ingress BPF classifier that returns `TC_ACT_SHOT` (drop) when the payload matches and `TC_ACT_OK` otherwise:

```
tc qdisc add dev eth0 clsact
tc filter add dev eth0 ingress bpf da obj tc_match.o sec classifier
```

Here the BPF program both *matches and decides* (it returns the drop/accept action directly), similar to XDP but at the TC hook (after the skb exists, so richer context; works on egress too).

**Trade-offs vs nftbpf:**

- *You gain:* it works without the `meta bpf` match compiled into nftables/kernel; TC-BPF is widely supported; you get the full skb context and can act on egress as well as ingress; and you avoid depending on a relatively rare nftables feature.
- *You lose:* the **policy is no longer expressed in nftables**. With nftbpf, BPF supplied only the *match* and the drop verdict lived in the ruleset, so `nft list ruleset` showed the whole policy in one place alongside all your other firewall rules. With TC-BPF, the drop decision is buried inside the BPF program and the qdisc/filter config — it won't appear in `nft list ruleset`, so an operator auditing the firewall could miss it. You've traded **policy visibility and unified management** for **portability and independence from the nftbpf feature**. The general lesson: nftbpf keeps policy declarative and centralized while delegating only matching logic; TC/XDP-BPF move the whole decision into BPF, which is more capable and portable but less transparent to firewall tooling.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 40 — systemd-networkd →](lesson-40-systemd-networkd){: .btn .btn-primary }
