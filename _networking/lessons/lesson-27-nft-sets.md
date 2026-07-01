---
title: "Lesson 27 — Sets and Verdict Maps"
nav_order: 27
parent: "Phase 8: nftables Firewall"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 27: nftables Sets and Verdict Maps

## Concept

Imagine blocking 10,000 IP addresses. With one rule per address, every packet would be checked against up to 10,000 rules linearly — slow and unwieldy. nftables solves this with **sets**: a single rule matches against a whole collection in roughly **constant time** (hash/tree lookup), no matter how many elements. **Verdict maps** go further: they map a key directly to an action, replacing a long chain of if/else rules with one lookup.

```
   Without sets:                    With a named set:
   ip saddr 1.1.1.1 drop            ip saddr @blocklist drop
   ip saddr 1.1.1.2 drop                       │
   ip saddr 1.1.1.3 drop            ┌───────────▼────────────┐
   ... 9997 more rules ...          │ blocklist = {1.1.1.1,   │
   (linear scan, slow)              │   1.1.1.2, ... 10000}   │  O(1) lookup
                                    └─────────────────────────┘
```

---

## How it works

- **Anonymous set** — inline, defined right in the rule: `tcp dport { 22, 80, 443 } accept`. Convenient for a fixed list, but you can't reference or update it later.
- **Named set** — a first-class object you create once and reference by `@name`. You can add/remove elements at runtime *without touching the rules*: `nft add element inet filter blocklist { 1.2.3.4 }`.
- **Interval set** (`flags interval`) — stores ranges/CIDRs efficiently: `type ipv4_addr ; flags interval ;` lets you put `10.0.0.0/8` or `192.168.1.10-192.168.1.20` as single elements.
- **Verdict map (vmap)** — maps keys to verdicts in one expression: `ip saddr vmap { 10.0.0.1 : accept, 10.0.0.2 : drop }`. The lookup *is* the decision; no rule chain needed. Maps can also target chains: `tcp dport vmap { 22 : jump ssh_chain, 80 : jump web_chain }`.

The performance win is algorithmic: a set/map uses a hash table or search tree, so lookup cost is independent of element count (O(1) or O(log n)), whereas individual rules are evaluated one by one (O(n)). For large lists this is the difference between microseconds and milliseconds per packet.

{: .note }
> **Atomic set updates — why they matter operationally**
> A named set can be updated *while the firewall runs* without reloading or flushing any rules: `nft add element ...` / `nft delete element ...` change only the set contents, atomically. This is how dynamic blocklists work — fail2ban, threat feeds, and rate-limiters add/remove offending IPs from a named set on the fly, and the existing `ip saddr @blocklist drop` rule instantly applies to the new contents. You never touch the ruleset structure, so there's no risk of a bad reload and no disruption to established traffic.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `tcp dport { 22, 80, 443 } accept` | Anonymous set (inline) |
| `nft add set inet filter blocklist '{ type ipv4_addr ; }'` | Create a named set |
| `nft add set inet filter nets '{ type ipv4_addr ; flags interval ; }'` | Interval set (for CIDRs/ranges) |
| `nft add element inet filter blocklist { 1.2.3.4 }` | Add an element at runtime |
| `nft delete element inet filter blocklist { 1.2.3.4 }` | Remove an element |
| `nft add rule inet filter input ip saddr @blocklist drop` | Reference a named set |
| `ip saddr vmap { 10.0.0.1 : accept, 10.0.0.2 : drop }` | Verdict map |
| `nft list set inet filter blocklist` | Show a set's contents |

---

## Lab

### Step 1 — A server namespace with a firewall table

```bash
$ sudo ip netns add srv
$ sudo ip netns add cli
$ sudo ip link add s-c type veth peer name c-s
$ sudo ip link set s-c netns srv
$ sudo ip link set c-s netns cli
$ sudo ip netns exec srv ip link set s-c up
$ sudo ip netns exec srv ip addr add 10.0.0.1/24 dev s-c
$ sudo ip netns exec cli ip link set c-s up
$ sudo ip netns exec cli ip addr add 10.0.0.2/24 dev c-s

$ sudo ip netns exec srv nft add table inet filter
$ sudo ip netns exec srv nft 'add chain inet filter input { type filter hook input priority 0 ; policy accept ; }'
```

### Step 2 — Anonymous set: allow several ports in one rule

```bash
$ sudo ip netns exec srv nft add rule inet filter input tcp dport { 22, 80, 443 } accept
$ sudo ip netns exec srv nft list chain inet filter input
chain input {
    ...
    tcp dport { 22, 80, 443 } accept     # one rule, three ports
}
```

### Step 3 — Named set: a runtime-updatable blocklist

```bash
$ sudo ip netns exec srv nft 'add set inet filter blocklist { type ipv4_addr ; }'
$ sudo ip netns exec srv nft add rule inet filter input ip saddr @blocklist drop

# Initially empty — client can ping:
$ sudo ip netns exec cli ping -c 1 10.0.0.1            # works

# Block the client at runtime, WITHOUT changing any rule:
$ sudo ip netns exec srv nft add element inet filter blocklist { 10.0.0.2 }
$ sudo ip netns exec cli ping -c 1 -W 1 10.0.0.1       # now blocked

# Unblock just as easily:
$ sudo ip netns exec srv nft delete element inet filter blocklist { 10.0.0.2 }
$ sudo ip netns exec cli ping -c 1 10.0.0.1            # works again
```

The `ip saddr @blocklist drop` rule never changed — only the set's contents did, atomically and live.

### Step 4 — Interval set for CIDR ranges

```bash
$ sudo ip netns exec srv nft 'add set inet filter badnets { type ipv4_addr ; flags interval ; }'
$ sudo ip netns exec srv nft add element inet filter badnets { 10.0.0.0/29 }
$ sudo ip netns exec srv nft add rule inet filter input ip saddr @badnets drop
# Now any source in 10.0.0.0–10.0.0.7 is dropped with a single element + rule.
```

`flags interval` is required to store CIDRs/ranges; without it the set only holds exact addresses.

### Step 5 — Verdict map: per-source decisions in one lookup

```bash
$ sudo ip netns exec srv nft add rule inet filter input \
    ip saddr vmap { 10.0.0.2 : accept, 10.0.0.3 : drop }
# 10.0.0.2 is accepted, 10.0.0.3 is dropped — no separate rules, just a map lookup.
```

A vmap can also dispatch to chains, e.g. `tcp dport vmap { 22 : jump ssh, 80 : jump web }` routes each port to its own handling chain — a clean alternative to a long if/else ladder.

### Step 6 — Inspect the sets

```bash
$ sudo ip netns exec srv nft list set inet filter blocklist
set blocklist {
    type ipv4_addr
    elements = { }      # (currently empty)
}
```

### Step 7 — Clean up

```bash
$ sudo ip netns delete srv cli
```

---

## Further Reading

| Topic | Link |
|---|---|
| nftables sets | [wiki.nftables.org — Sets](https://wiki.nftables.org/wiki-nftables/index.php/Sets) |
| nftables maps / vmaps | [wiki.nftables.org — Maps](https://wiki.nftables.org/wiki-nftables/index.php/Maps) |
| Hash table | [Wikipedia — Hash table](https://en.wikipedia.org/wiki/Hash_table) |
| fail2ban (uses dynamic sets) | [Wikipedia — Fail2ban](https://en.wikipedia.org/wiki/Fail2ban) |

---

## Checkpoint

**Q1. Why is a named set faster than 1000 individual `ip saddr` rules?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because of how matching scales. With 1000 individual `ip saddr X drop` rules, the kernel evaluates them **linearly** — a packet may be checked against up to 1000 rules one after another before a verdict is reached. That's O(n): cost grows with the number of rules. A **named set** stores all 1000 addresses in a single data structure (a hash table for exact matches, or a tree for intervals), and the single rule `ip saddr @blocklist drop` does **one lookup** into that structure. Hash/tree lookups are O(1) or O(log n) — essentially independent of how many elements the set holds. So matching against 1000 addresses costs about the same as matching against 5. For large lists this is the difference between scanning a thousand rules per packet and doing a single constant-time lookup, which is dramatically faster and also keeps the ruleset small and readable.
</details>

---

**Q2. What is the practical difference between an anonymous set and a named set?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An **anonymous set** is defined inline inside a rule, e.g. `tcp dport { 22, 80, 443 } accept`. It's convenient and efficient for matching, but it has no name and no independent existence — you can't reference it elsewhere, and you can't add or remove elements without rewriting the rule itself. It's best for a *fixed* list known at rule-creation time.

A **named set** is a first-class object you create separately (`nft add set ... blocklist ...`) and reference by `@name` in rules. Its big advantage is that its contents can be **updated at runtime** — `nft add element` / `nft delete element` — atomically and without touching any rules. This makes named sets the right choice for anything *dynamic*: blocklists fed by fail2ban or threat intelligence, allowlists that change as hosts come and go, etc. Summary: anonymous = inline, static, throwaway; named = reusable, runtime-mutable, the basis of dynamic firewalling.
</details>

---

**Q3. How does a verdict map (vmap) differ from a plain set, and when would you reach for one?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A plain **set** is a collection of *keys* you test membership against — a rule like `ip saddr @blocklist drop` asks "is the source in this set?" and applies one fixed verdict (`drop`) to all matches. A **verdict map (vmap)** maps each *key to its own verdict (or chain target)*: `ip saddr vmap { 10.0.0.1 : accept, 10.0.0.2 : drop, 10.0.0.3 : jump special }`. The lookup itself yields the action, so a single expression replaces what would otherwise be many separate rules each with a different outcome.

You reach for a vmap when you need **different actions for different key values**, especially as a clean dispatch table — for example, routing each destination port to its own handling chain (`tcp dport vmap { 22 : jump ssh, 80 : jump web, 443 : jump web }`) instead of writing a long if/else ladder of jump rules. It's both more efficient (one constant-time lookup vs. sequential rule evaluation) and more readable (the policy is a table). Use a set when the verdict is uniform across all members; use a vmap when the verdict depends on which member matched.
</details>

---

## Homework

Build a firewall that uses: (1) an anonymous set to allow a group of service ports, (2) a named interval set for an allowlist of trusted CIDR ranges, and (3) a verdict map that sends traffic from different source subnets to different named chains (e.g. a `trusted` chain that's permissive and a `guest` chain that's restrictive). Then simulate updating the trusted-CIDR set at runtime (add and remove a range) and confirm the change takes effect immediately with no rule reload. Explain why this architecture scales better than a flat list of rules.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A sketch combining all three:

```
table inet filter {
    set trusted_nets { type ipv4_addr ; flags interval ; }   # named interval set

    chain trusted {                      # permissive handling
        ct state established,related accept
        accept
    }
    chain guest {                        # restrictive handling
        ct state established,related accept
        tcp dport { 80, 443 } accept     # anonymous set: only web out
        drop
    }
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        iif "lo" accept
        ip saddr @trusted_nets jump trusted          # interval-set membership → trusted
        ip saddr vmap { 192.168.10.0/24 : jump guest } # vmap dispatch by subnet
    }
}
```

**Runtime update (no reload):**
```
nft add element inet filter trusted_nets { 10.50.0.0/24 }   # instantly trusted
nft delete element inet filter trusted_nets { 10.50.0.0/24 } # instantly removed
```
The `jump trusted` rule references `@trusted_nets` by name, so changing the set's contents immediately changes which sources are trusted — no rule is added, removed, or reloaded, and established connections are undisturbed.

**Why it scales better than a flat rule list:**
1. **Algorithmic matching.** Membership tests against sets are O(1)/O(log n) lookups regardless of how many CIDRs or ports are in them, versus O(n) linear evaluation of one rule per entry. As the lists grow to thousands of entries, set-based matching stays fast while a flat list degrades linearly.
2. **Separation of policy from data.** The *structure* (which chains exist, what each does) is stable and reviewable, while the *data* (which IPs are trusted/blocked) lives in sets you mutate atomically at runtime. Operationally this means dynamic tools can update memberships continuously without ever risking a bad ruleset reload.
3. **Readability and maintainability.** A vmap dispatch table and a handful of named chains express "trusted vs guest" intent far more clearly than hundreds of interleaved per-IP rules, making the policy easier to audit and less error-prone.

This set/map/chain decomposition is exactly how large production firewalls and tools like fail2ban stay both fast and manageable at scale.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 28 — Flowtable Offload →](lesson-28-flowtable){: .btn .btn-primary }
