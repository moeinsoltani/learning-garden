---
title: "Lesson 26 — Packet Filtering with nft"
nav_order: 26
parent: "Phase 8: nftables Firewall"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 26: Packet Filtering with nft

## Concept

Now you build a real **stateful firewall**. The key word is *stateful*: instead of judging each packet alone, the firewall uses connection tracking (Lesson 23) to know whether a packet belongs to a connection it already allowed. That single idea is what makes a firewall both secure and usable — you allow connections to *start* in the directions you want, and the replies are permitted automatically.

The canonical structure of a host firewall:

```
   chain input (policy DROP):              ← deny by default
     1. ct state established,related accept   ← let replies/related back in
     2. ct state invalid drop                 ← drop garbage early
     3. iif lo accept                         ← always trust loopback
     4. tcp dport 22 accept                   ← allow the services you offer
     5. (anything not matched) → DROP         ← the default policy
```

A **default-drop** policy with explicit allows is the secure pattern: only what you name gets through.

---

## How it works

- **`policy drop`** on a base chain means "if no rule says accept, drop it." This is *deny by default*. The opposite, `policy accept`, is *allow by default* and is generally insecure for an input chain.
- **`ct state established,related accept`** is the workhorse: it permits all return traffic for connections you already allowed (ESTABLISHED) and legitimately-spawned related connections (RELATED). Put it first — most packets in a busy system match it, and matching early is efficient.
- **`ct state invalid drop`** discards packets that don't correspond to any valid flow (spoofed, out-of-window, malformed). Dropping them early avoids odd behavior.
- **Rate limiting** (`limit rate 10/second`) caps how fast a rule matches — useful to blunt floods or limit logging.
- **Logging** (`log prefix "..."`) records packets to the kernel log for visibility, usually right before a drop.

Order matters: nftables evaluates rules top to bottom within a chain, and the first terminal verdict (`accept`/`drop`) wins. So you arrange rules cheapest-and-most-common first, specific allows next, and rely on the default policy to drop the rest.

{: .note }
> **Why allow established *before* anything else**
> In a typical workload the vast majority of packets are part of already-established connections (the data flowing on connections you set up). Matching `ct state established,related accept` as rule #1 means those packets take the shortest path through the chain and are accepted immediately, without being checked against every service rule. It's both a correctness rule (replies must get through) and a performance rule (short-circuit the common case).

---

## New Commands This Lesson

| Match / verdict | Meaning |
|---|---|
| `ct state established,related accept` | Allow return + related traffic |
| `ct state invalid drop` | Drop packets with no valid flow |
| `iif "lo" accept` / `iifname "eth0"` | Match input interface |
| `tcp dport 22 accept` | Match TCP destination port |
| `ip saddr 10.0.0.0/24` | Match source network |
| `limit rate 10/second burst 20 packets` | Rate-limit matches |
| `log prefix "DROP: " level warn` | Log matching packets |
| `policy drop` (on the chain) | Default-deny |

---

## Lab

We'll harden a "server" namespace: allow SSH and ICMP and established traffic, drop everything else, and watch the policy take effect.

### Step 1 — A server namespace reachable from a client

```bash
$ sudo ip netns add srv
$ sudo ip netns add cli
$ sudo ip link add s-c type veth peer name c-s
$ sudo ip link set s-c netns srv
$ sudo ip link set c-s netns cli
$ sudo ip netns exec srv ip link set lo up
$ sudo ip netns exec srv ip link set s-c up
$ sudo ip netns exec srv ip addr add 10.0.0.1/24 dev s-c
$ sudo ip netns exec cli ip link set c-s up
$ sudo ip netns exec cli ip addr add 10.0.0.2/24 dev c-s
```

### Step 2 — Baseline: everything works (no firewall yet)

```bash
$ sudo ip netns exec cli ping -c 1 10.0.0.1      # works
$ sudo ip netns exec srv nc -l -p 8080 &          # a "service" on 8080
$ sudo ip netns exec cli nc -z -v 10.0.0.1 8080   # connects
```

### Step 3 — Install a default-drop stateful firewall

```bash
$ sudo ip netns exec srv nft -f - <<'EOF'
flush ruleset
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        ct state established,related accept   # replies/related first
        ct state invalid drop                 # drop garbage
        iif "lo" accept                       # trust loopback
        ip protocol icmp accept               # allow ping
        tcp dport 22 accept                   # allow SSH
        # everything else falls through to: policy drop
    }
}
EOF
```

### Step 4 — Verify the policy

```bash
# ICMP allowed:
$ sudo ip netns exec cli ping -c 1 10.0.0.1
64 bytes from 10.0.0.1 ...                  # works (icmp rule)

# Port 8080 now blocked (only 22 is allowed):
$ sudo ip netns exec srv nc -l -p 8080 &
$ sudo ip netns exec cli nc -z -w 2 -v 10.0.0.1 8080
# nc: connect to 10.0.0.1 port 8080 ... Connection timed out  ← dropped by policy
```

8080 is silently dropped because no rule accepts it and the policy is `drop`. SSH (22) and ICMP work because they're explicitly allowed.

### Step 5 — Prove established traffic still flows

The server can still make *outbound* connections and get replies, because of the established rule:

```bash
# From srv, connect OUT to cli (cli listening on 7000)
$ sudo ip netns exec cli nc -l -p 7000 &
$ sudo ip netns exec srv nc -w 2 10.0.0.2 7000
# Works: srv's outbound SYN leaves (output), the reply comes back and matches
# 'ct state established accept' in the input chain — no inbound rule for 7000 needed.
```

This is the payoff of stateful filtering: you didn't open port 7000 inbound, yet the reply traffic for the connection *you* initiated is allowed automatically.

### Step 6 — Add logging and rate limiting for dropped traffic

```bash
$ sudo ip netns exec srv nft add rule inet filter input \
    limit rate 5/second log prefix "FW-DROP: " level warn
# (place before the implicit policy drop; logs at most 5 dropped pkts/sec)

# Generate blocked traffic, then check the kernel log:
$ sudo ip netns exec cli nc -z -w 1 10.0.0.1 9999
$ sudo dmesg | tail | grep FW-DROP
... FW-DROP: IN=s-c ... PROTO=TCP ... DPT=9999 ...
```

The rate limit prevents a flood of blocked packets from filling your logs while still giving visibility.

### Step 7 — Clean up

```bash
$ sudo ip netns delete srv cli
```

---

## Further Reading

| Topic | Link |
|---|---|
| Stateful firewall | [Wikipedia — Stateful firewall](https://en.wikipedia.org/wiki/Stateful_firewall) |
| nftables stateful rules | [wiki.nftables.org — Conntrack](https://wiki.nftables.org/wiki-nftables/index.php/Matching_connection_tracking_stateful_metainformation) |
| Default-deny | [Wikipedia — Default deny](https://en.wikipedia.org/wiki/Principle_of_least_privilege) |
| `nft` | [man7.org — nft(8)](https://man7.org/linux/man-pages/man8/nft.8.html) |

---

## Checkpoint

**Q1. Write a complete nftables ruleset for a host that: allows incoming SSH, allows established/related traffic, allows loopback, drops invalid packets, and drops everything else. Explain the order of the rules.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>

```
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        ct state established,related accept   # 1. return traffic for allowed conns
        ct state invalid drop                 # 2. drop packets with no valid flow
        iif "lo" accept                       # 3. trust loopback
        tcp dport 22 accept                   # 4. allow inbound SSH
        # 5. anything else → dropped by 'policy drop'
    }
}
```

**Order rationale:**
1. `established,related accept` goes **first** because the bulk of packets belong to ongoing connections; matching them immediately is both correct (replies must get through) and efficient (short-circuits the rest of the chain).
2. `invalid drop` next, to discard malformed/spoofed packets early before they can match anything else.
3. `iif lo accept` so the host can always talk to itself (many local services depend on loopback).
4. The specific service allow (`tcp dport 22`) for new inbound SSH connections.
5. The chain's `policy drop` catches everything not explicitly accepted — this is the deny-by-default backbone that makes the firewall secure.

The principle: most-common/cheapest matches first, explicit allows next, default-deny last.
</details>

---

**Q2. With a default-drop input policy and only `tcp dport 22` allowed inbound, the host can still browse the web (make outbound HTTPS connections) and receive the responses. How, given there's no rule allowing inbound port 443 traffic?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because of the **`ct state established,related accept`** rule combined with connection tracking. When the host initiates an outbound HTTPS connection, that traffic leaves via the *output* chain (which isn't default-drop here, or has its own permissive policy), and conntrack records the new flow. The web server's responses come back to the host and hit the *input* chain — but they're classified by conntrack as **ESTABLISHED** (return traffic for a connection the host started), so the `established,related accept` rule permits them. No inbound rule for port 443 is needed because the firewall isn't allowing *new inbound* connections to 443; it's allowing *replies to outbound* connections the host itself opened. This is the entire value of statefulness: you control which direction connections may be *initiated* (here, outbound freely, inbound only SSH), and the return halves are handled automatically by tracking state, rather than having to poke holes for every possible reply port.
</details>

---

**Q3. Why drop `ct state invalid` packets, and why place that rule near the top of the chain?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`ct state invalid` packets are ones conntrack can't associate with any valid connection and that aren't a legitimate NEW connection either — for example, TCP segments arriving outside the valid window, packets for a flow that was never properly established, malformed packets, or certain spoofed/injected packets. Letting them through can cause subtle problems: they can confuse stateful logic, be used in some evasion or injection attacks, or trigger unexpected responses. Dropping them is a clean default.

You place the rule **near the top** (right after the established/related accept) so that invalid packets are discarded *before* they have any chance to match a later, more permissive rule by coincidence (e.g., an invalid TCP packet that happens to have dport 22 shouldn't be accepted by the SSH rule). Filtering them out early ensures the rest of the chain only evaluates packets that are at least part of a coherent flow or a genuine new connection, making the policy both safer and easier to reason about.
</details>

---

## Homework

Harden the router from Lesson 17 (the `ns1 — router — ns2` topology) so that the **forward** chain only permits *established/related* traffic plus *new* connections initiated from ns1 toward ns2 — but blocks new connections initiated from ns2 toward ns1. Verify with tcpdump and connection attempts in both directions. Explain how this asymmetric policy (one side can initiate, the other can't) is exactly how a home firewall protects a LAN.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The forward-chain policy on the router:

```
table inet filter {
    chain forward {
        type filter hook forward priority 0; policy drop;

        ct state established,related accept          # replies both ways
        iifname "<ns1-side>" ct state new accept     # ns1 may START connections
        # new connections from ns2 are NOT accepted → dropped by policy
    }
}
```

**Behavior to verify:**
- **ns1 → ns2 new connection:** succeeds. The SYN matches the `iifname ns1-side ct state new accept` rule, conntrack creates the flow, and ns2's replies come back as ESTABLISHED (accepted by rule 1). tcpdump shows the full handshake.
- **ns2 → ns1 new connection:** fails. ns2's SYN arrives on the ns2-side interface, doesn't match the "new from ns1" rule, isn't established, so it's dropped by `policy drop`. tcpdump on the router's ns1-side interface shows the SYN never being forwarded.

**Why this is exactly a home firewall:** your home router runs precisely this asymmetric policy. Devices on your LAN (the "ns1" side) can freely *initiate* connections out to the internet, and the replies flow back because they're ESTABLISHED. But hosts on the internet (the "ns2" side) **cannot initiate** new connections inward to your LAN — those unsolicited SYNs are dropped. Combined with NAT (Lesson 22), this gives the default home-network security model: outbound allowed, inbound blocked unless it's a reply to something you started (or an explicit port-forward you configured). The stateful `established,related accept` plus a directional `new` rule is the whole mechanism — there's no magic, just connection tracking applied asymmetrically by which interface a NEW connection arrives on.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 27 — Sets and Verdict Maps →](lesson-27-nft-sets){: .btn .btn-primary }
