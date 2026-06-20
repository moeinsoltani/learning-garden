---
title: "Lesson 47 — Network Debugging Methodology"
nav_order: 47
parent: "Phase 15: Troubleshooting"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 47: Network Debugging Methodology

## Concept

This is the capstone. You have all the tools (Lesson 46); now you need the **discipline** that turns tools into reliable diagnosis. The entire methodology fits in one rule:

> **Never guess. Work layer by layer, from the bottom up, and let the packet tell you where it stops.**

```
   Bad debugging:  "Maybe it's DNS? ...the firewall? ...let me restart it..."
                   (random, untestable, often makes things worse)

   Good debugging: L1 up? → L2 resolving? → L3 routed? → return path?
                   → NAT? → firewall? → BPF? → forwarding?
                   (ordered, each step a yes/no, stop at the first NO)
```

Working **bottom-up** matters because lower layers are prerequisites for higher ones: there's no point investigating a TCP handshake if the interface is down or ARP isn't resolving. You verify each layer is sound *before* suspecting the one above it, so you never waste time debugging a symptom whose cause is two layers down.

---

## How it works

**The ordered question list.** Ask these in sequence; the first one that answers "no/wrong" is where (or near where) your fault lives:

1. **Is the interface up?** `ip link show` — `state UP` *and* `LOWER_UP` (carrier). No link, nothing else matters.
2. **Is an IP assigned, correctly?** `ip addr show` — right address, **right prefix** (a wrong mask silently makes peers "off-link").
3. **Is L2 reachable?** `ip neigh show` / `arping` / tcpdump for ARP — does the next hop resolve to a MAC (`REACHABLE`), or `FAILED`?
4. **Is L3 reachable?** `ping` — do ICMP echo *replies* come back?
5. **Is the route correct?** `ip route get <dst>` — right gateway, right interface, sane source IP?
6. **Is routing symmetric?** Check the **return path** with `ip route get` *from the other side*. Asymmetric routing + `rp_filter` silently drops packets.
7. **Is NAT correct?** `conntrack -L` and tcpdump on **both** sides of the NAT — is the translation entry present and applied?
8. **Is the firewall dropping it?** `nft list ruleset` — look for DROP/REJECT; add a **temporary `log` rule** to catch it in the act.
9. **Is an eBPF program intercepting?** `ip link show` (xdp), `bpftool net show`.
10. **Is forwarding enabled?** `sysctl net.ipv4.ip_forward` — on anything acting as a router.

**Two principles that prevent most wasted hours:**

- **Bisect the path.** Don't check ten hops linearly — capture in the *middle* and ask "did the packet get this far?" That halves the search space each time (this is the both-ends capture from Lesson 45, generalized).
- **Change one thing at a time, and verify.** After each fix, re-test and confirm *that specific thing* improved before moving on. Shotgun changes make it impossible to know what fixed (or broke) what — and can stack new faults on top of old ones.

{: .note }
> **Don't stop at the first fault**
> A fix that makes *one* symptom disappear (ping works!) doesn't mean the system is healthy — there may be a second, independent fault one layer up (a firewall rule, a missing service). Walk the **whole** ordered list even after finding something, and treat "ping works but the app doesn't" as a signal that you've fixed a *lower* layer and the real target is *higher*. Methodical completeness is what separates "it works now (somehow)" from "I understand exactly what was wrong."

---

## A reusable diagnosis template

Write your findings down — it forces rigor and produces a record:

```
Symptom:        <what fails, exactly — "curl 10.0.2.7:80 from pod-a times out">
Expected:       <what should happen>
L1 link:        ip link show     → [good/fault: ...]
L2 addr:        ip addr show      → [good/fault: ...]
L2 neigh:       ip neigh show     → [good/fault: ...]
L3 ping:        ping <dst>        → [good/fault: ...]
L3 route (fwd): ip route get <dst>→ [good/fault: ...]
L3 route (ret): ip route get <src> (from other side) → [good/fault: ...]
NAT:            conntrack -L      → [good/fault: ...]
Firewall:       nft list ruleset  → [good/fault: ...]
eBPF:           bpftool net show  → [good/fault: ...]
Forwarding:     sysctl ip_forward → [good/fault: ...]
Packet truth:   tcpdump (both ends) → [where it stops]
Root cause:     <the layer that failed and why>
Fix:            <the one change> → re-test → [confirmed]
```

---

## Lab

We'll practice the methodology on a topology with a **single hidden fault**, diagnosing strictly in order.

### Step 1 — Build a router topology and inject one fault

```bash
# Lesson 17 topology: ns1 — router — ns2, but we'll "forget" to enable forwarding
$ sudo ip netns add ns1; sudo ip netns add ns2; sudo ip netns add rtr
$ sudo ip link add a netns ns1 type veth peer name ra netns rtr
$ sudo ip link add b netns ns2 type veth peer name rb netns rtr
$ sudo ip netns exec ns1 ip addr add 10.0.1.2/24 dev a
$ sudo ip netns exec rtr ip addr add 10.0.1.1/24 dev ra
$ sudo ip netns exec ns2 ip addr add 10.0.2.2/24 dev b
$ sudo ip netns exec rtr ip addr add 10.0.2.1/24 dev rb
$ for p in "ns1 a" "ns2 b" "rtr ra" "rtr rb"; do set -- $p; sudo ip netns exec $1 ip link set $2 up; done
$ for n in ns1 ns2 rtr; do sudo ip netns exec $n ip link set lo up; done
$ sudo ip netns exec ns1 ip route add default via 10.0.1.1
$ sudo ip netns exec ns2 ip route add default via 10.0.2.1
# NOTE: we intentionally did NOT run: sysctl -w net.ipv4.ip_forward=1 in rtr
```

### Step 2 — Symptom

```bash
$ sudo ip netns exec ns1 ping -c2 10.0.2.2     # FAILS (100% loss)
```

### Step 3 — Diagnose in order

```bash
# 1 link
$ sudo ip netns exec ns1 ip link show a              # UP, LOWER_UP → good
# 2 addr
$ sudo ip netns exec ns1 ip addr show a              # 10.0.1.2/24 → good
# 3 neigh (to the gateway)
$ sudo ip netns exec ns1 ping -c1 10.0.1.1           # gateway reachable → good (L1-L3 local fine)
# 4/5 route
$ sudo ip netns exec ns1 ip route get 10.0.2.2       # via 10.0.1.1 dev a → good
# bisect: does the packet reach the router's far interface?
$ sudo ip netns exec rtr tcpdump -ni rb icmp &       # capture on the ns2-facing iface
$ sudo ip netns exec ns1 ping -c2 10.0.2.2
#   → NOTHING on rb. Packet arrives at the router (ra) but is NOT forwarded out rb.
$ sudo ip netns exec rtr tcpdump -ni ra icmp         # confirms: echo request arrives on ra
# 10 forwarding
$ sudo ip netns exec rtr sysctl net.ipv4.ip_forward  # = 0  ← THE FAULT
```

### Step 4 — Fix one thing, verify

```bash
$ sudo ip netns exec rtr sysctl -w net.ipv4.ip_forward=1
$ sudo ip netns exec ns1 ping -c2 10.0.2.2           # now works
```

The bisect capture (packet *in* on `ra`, *not out* on `rb`) localized it to "the router isn't forwarding," and the forwarding sysctl confirmed exactly why.

### Step 5 — Clean up

```bash
$ sudo ip netns delete ns1; sudo ip netns delete ns2; sudo ip netns delete rtr
```

---

## Further Reading

| Topic | Link |
|---|---|
| `ping` | [man7.org — ping(8)](https://man7.org/linux/man-pages/man8/ping.8.html) |
| `ip route get` | [man7.org — ip-route(8)](https://man7.org/linux/man-pages/man8/ip-route.8.html) |
| Reverse path filtering | [kernel.org — IP sysctl (rp_filter)](https://docs.kernel.org/networking/ip-sysctl.html) |
| `mtr` (path diagnosis) | [man7.org — mtr(8)](https://man7.org/linux/man-pages/man8/mtr.8.html) |
| `traceroute` | [man7.org — traceroute(8)](https://man7.org/linux/man-pages/man8/traceroute.8.html) |

---

## Checkpoint

**Q1. Why debug from the bottom layer up, instead of starting with the application symptom?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because **lower layers are prerequisites for higher ones**, so a failure low down *causes* symptoms high up that look like higher-layer problems. If the interface is down, ARP can't resolve; if ARP fails, IP can't reach the next hop; if routing is wrong, TCP never connects. Starting at the application ("the HTTP request hangs") and poking at HTTP/DNS wastes time chasing a *symptom* whose *cause* is two layers below. By verifying each layer is sound from the bottom up, you either find the fault at its true layer or definitively rule that layer out before moving up — so when you finally reach the application, you *know* everything beneath it is healthy and the problem really is there. It converts a vague top-down hunt into an ordered process where the first failing check is at (or very near) the root cause.
</details>

---

**Q2. Diagnose the broken topology in the lab: name every check you ran and what it told you.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>

1. **`ip link show a` (ns1)** → UP/LOWER_UP → **good**: L1/L2 link is fine.
2. **`ip addr show a` (ns1)** → `10.0.1.2/24` → **good**: correct address and prefix.
3. **`ping 10.0.1.1` (gateway)** → succeeds → **good**: ns1↔router is fully working through L3, so the problem is *beyond* the first hop (ARP, local routing, and the near link are all fine).
4. **`ip route get 10.0.2.2` (ns1)** → via `10.0.1.1 dev a` → **good**: ns1 is correctly sending off-subnet traffic to the router.
5. **Bisect with tcpdump on the router:** capture on **`rb`** (ns2-facing) during a ping → **nothing arrives** there; capture on **`ra`** (ns1-facing) → the **echo request *does* arrive**. This localizes the fault precisely: the packet **reaches the router but is not forwarded out the other interface**.
6. **`sysctl net.ipv4.ip_forward` (router)** → **`0`** → **the root cause**: the router isn't acting as a router because IP forwarding is disabled, so it accepts the packet on `ra` and drops it instead of forwarding to `rb`.

**Fix:** `sysctl -w net.ipv4.ip_forward=1` on the router, then re-test → ping succeeds. The decisive evidence was the bisect capture (in on `ra`, not out on `rb`), which pointed straight at forwarding before I even checked the sysctl.
</details>

---

**Q3. What is "asymmetric routing," and why can it cause a connection to fail even when both the forward route and the firewall look correct?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Asymmetric routing** is when a packet's **reply takes a different path than the request** — the forward and return directions traverse different interfaces/routers. It causes silent failures for two reasons that the forward-path checks won't reveal:

1. **Reverse-path filtering (`rp_filter`).** With strict `rp_filter=1` (Lesson 34), the kernel drops an incoming packet if the route back to its *source* doesn't point out the interface the packet *arrived on*. In an asymmetric setup, reply (or request) packets arrive on an interface that isn't the one the return route would use, so they're **silently dropped** — even though your forward route and firewall rules are perfectly correct.
2. **Stateful firewall / conntrack mismatch.** A stateful rule expects to see *both* directions of a flow on the same node. If the return traffic bypasses the box that tracked the outbound side, the reply looks like it has **no matching conntrack entry** (or is `INVALID`) and gets dropped by an `established,related`-based ruleset.

That's why step 6 of the methodology explicitly checks the **return path** with `ip route get` *from the other end*: a connection needs *both* directions to work, and a forward-only check can look entirely healthy while the reverse path is quietly broken. Fixes include making routing symmetric, relaxing `rp_filter` to loose mode (`2`) or off (`0`) where appropriate, or ensuring both directions pass through the same stateful node.
</details>

---

## Homework

Have someone (or a script) build you a **fresh broken lab** without telling you the fault, and diagnose it from scratch using the diagnosis template — **timing yourself**. Fill in every row (good/fault) in order, identify the root cause from the *first* failing check plus a confirming tcpdump, apply a single fix, and verify. Then write a short retrospective: which check found it, how many checks you ran before that, and whether any guess you were tempted to make would have been wrong.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
There's no single "correct" fault here — the point is the **process**, and a good answer demonstrates it regardless of what the injected fault was. A strong write-up shows:

- **The template filled in top-to-bottom**, each row marked good/fault, stopping the *investigation's conclusion* at the first failing layer (though you still sanity-check higher layers after fixing, per Q in Lesson 46's spirit).
- **A bisecting tcpdump** that turned "it fails somewhere" into "the packet reaches X but not Y," pinning the fault to a specific layer/hop rather than a guess.
- **Root cause stated as a layer + reason** (e.g. "L2: `ip neigh` FAILED because the static ARP entry pointed at the wrong MAC," or "L3 return path: `rp_filter=1` dropped the asymmetric reply," or "forwarding disabled," or "input `drop` rule").
- **One fix, then re-test and confirm** — not a flurry of changes.
- **Retrospective honesty:** e.g. "I was tempted to blame DNS/the application, but the bottom-up walk showed ARP never resolved — the app guess would have wasted 20 minutes." The lesson to articulate: the methodical order found it in N checks, and the tempting shortcut guess would have been **wrong**, which is exactly why "never guess, go layer by layer" is the rule. Faster *next* time comes from trusting the order, not from skipping it.

**Congratulations — that's the whole curriculum.** From "what is a namespace" to diagnosing an arbitrary broken topology under time pressure, you've built every primitive by hand and can now reason about Docker, Kubernetes, overlays, firewalls, and eBPF as compositions of things you understand. The methodology in this lesson is what you'll actually reach for at 2 a.m. when something's down: never guess, work the layers, let the packet tell you where it stops.
</details>
