---
title: "Lesson 34 — Network sysctl Tunables"
nav_order: 34
parent: "Phase 10: sysctl"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 34: Network sysctl Tunables

## Concept

**sysctl** exposes hundreds of kernel knobs as a virtual filesystem under `/proc/sys/`. The networking ones (under `net.`) control behaviors you've already touched — forwarding, ARP, buffers — plus many you haven't. Changing them takes effect *immediately*, with no reboot or recompile. Knowing the handful that matter is a core operational skill.

```
   /proc/sys/net/ipv4/ip_forward        ←→  sysctl net.ipv4.ip_forward
   /proc/sys/net/ipv4/conf/all/rp_filter ←→  sysctl net.ipv4.conf.all.rp_filter
        (the dotted name maps directly to the file path)
```

---

## How it works

A sysctl name like `net.ipv4.conf.all.rp_filter` maps to the file `/proc/sys/net/ipv4/conf/all/rp_filter`. You read it with `sysctl <name>` (or `cat`) and set it with `sysctl -w <name>=<value>`.

The most important networking tunables:

| Sysctl | Purpose |
|---|---|
| `net.ipv4.ip_forward` | Enable IPv4 routing/forwarding (Lesson 17). `1` = router. |
| `net.ipv6.conf.all.forwarding` | Same for IPv6. |
| `net.ipv4.conf.<if>.rp_filter` | **Reverse-path filter**: drop packets whose source wouldn't route back out the interface they came in on. `0`=off, `1`=strict, `2`=loose. |
| `net.ipv6.conf.<if>.accept_ra` | Accept IPv6 Router Advertisements (autoconfig). |
| `net.core.rmem_max` / `wmem_max` | Max socket receive/send buffer sizes (for high-BDP tuning). |
| `net.ipv4.tcp_congestion_control` | Which congestion-control algorithm (cubic, bbr, …). |
| `net.ipv4.ip_local_port_range` | The ephemeral source-port range for outbound connections. |
| `net.ipv4.neigh.default.gc_thresh1/2/3` | ARP cache size thresholds (when entries get garbage-collected). |

**Per-interface vs global:** many `conf` knobs exist both as `conf/all/<knob>` and `conf/<iface>/<knob>`. The effective value is usually a combination (for `rp_filter`, the **max** of `all` and the specific interface). So setting `all` doesn't always override a per-interface value — you often must set both.

{: .note }
> **rp_filter — the asymmetric-routing trap**
> Reverse-path filtering (RPF, [RFC 3704](https://en.wikipedia.org/wiki/Reverse-path_forwarding)) defends against IP spoofing: when a packet arrives, the kernel checks "if I had to *reply* to this source, would the reply go back out *this same* interface?" In **strict** mode (`1`), if not, the packet is dropped. That's great for security but **breaks legitimate asymmetric routing** — multi-homed hosts where traffic intentionally arrives on one link and leaves on another. **Loose** mode (`2`) relaxes the check: it only requires the source be reachable via *some* interface, not the same one. So: `1` for normal single-homed hosts (anti-spoof), `2` for asymmetric/multi-homed setups, `0` to disable entirely (rarely advisable). This is a top cause of "packets mysteriously dropped on a router."

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `sysctl <name>` | Read a value |
| `sysctl -w <name>=<value>` | Set a value (runtime, not persistent) |
| `sysctl -a` | Dump all sysctls |
| `sysctl -a | grep ^net` | Show all networking knobs |
| `cat /proc/sys/net/...` | Read directly via the proc file |
| `ip netns exec <ns> sysctl ...` | Read/set inside a namespace |

---

## Lab

We'll reproduce the classic `rp_filter` drop: route a packet asymmetrically through a router, watch strict RPF drop it, then loosen RPF and watch it pass.

### Step 1 — Build a multi-homed scenario

A router with two interfaces to a client that *replies on a different path* than packets arrive. The simplest demonstration: a router receiving a packet whose source address routes back out a *different* interface than it arrived on.

```bash
$ sudo ip netns add router
$ sudo ip netns add h1
$ sudo ip netns add h2

# router <-> h1 on 10.0.1.0/24, router <-> h2 on 10.0.2.0/24
# (use the veth pattern from Lesson 17)
$ sudo ip netns exec router sysctl -w net.ipv4.ip_forward=1
```

Now craft asymmetry: make the router's route back to h1's source go out the h2 interface (e.g., a deliberately wrong/asymmetric route), so a packet arriving from h1 on the h1-interface has a reverse path pointing at the h2-interface.

### Step 2 — Set strict rp_filter and watch the drop

```bash
$ sudo ip netns exec router sysctl -w net.ipv4.conf.all.rp_filter=1
$ sudo ip netns exec router sysctl -w net.ipv4.conf.default.rp_filter=1
# (also set the specific ingress interface, since effective = max(all, iface))
$ sudo ip netns exec router sysctl -w net.ipv4.conf.<h1-iface>.rp_filter=1

# Send the asymmetric packet from h1; capture on the router:
$ sudo ip netns exec router tcpdump -n -i <h1-iface> &
$ sudo ip netns exec h1 ping -c 1 -W 1 10.0.2.<h2>
# The request ARRIVES on the router's h1-iface but is DROPPED — strict RPF
# rejects it because the reverse path for h1's source isn't this interface.
```

### Step 3 — Confirm RPF is the culprit

```bash
$ sudo ip netns exec router cat /proc/net/stat/rt_cache 2>/dev/null
# or check the rp_filter drop counter:
$ sudo ip netns exec router nstat -az | grep -i rpfilter
# IpReversePathFilter   N   ← increments per RPF-dropped packet
```

The `IpReversePathFilter` counter rising is the smoking gun: the packet was dropped specifically by reverse-path filtering.

### Step 4 — Switch to loose mode and watch it pass

```bash
$ sudo ip netns exec router sysctl -w net.ipv4.conf.all.rp_filter=2
$ sudo ip netns exec router sysctl -w net.ipv4.conf.<h1-iface>.rp_filter=2

$ sudo ip netns exec h1 ping -c 1 10.0.2.<h2>
64 bytes ...        # now it passes — loose RPF only requires the source be
                    # reachable via SOME interface, not the same one
```

Loose mode (`2`) accepts the asymmetric packet because h1's source is reachable via *an* interface, even if not the ingress one. This is the correct setting for legitimate multi-homed/asymmetric topologies.

### Step 5 — Explore other tunables

```bash
# See the current congestion control and what's available
$ sysctl net.ipv4.tcp_congestion_control
$ sysctl net.ipv4.tcp_available_congestion_control

# The ephemeral port range used for outbound connections
$ sysctl net.ipv4.ip_local_port_range
net.ipv4.ip_local_port_range = 32768   60999

# Per-interface vs all (forwarding)
$ sysctl net.ipv4.conf.all.forwarding net.ipv4.conf.<if>.forwarding
```

### Step 6 — Clean up

```bash
$ sudo ip netns delete router h1 h2
```

---

## Further Reading

| Topic | Link |
|---|---|
| sysctl | [Wikipedia — sysctl](https://en.wikipedia.org/wiki/Sysctl) |
| Reverse-path forwarding | [Wikipedia — Reverse-path forwarding](https://en.wikipedia.org/wiki/Reverse-path_forwarding) |
| Kernel IP sysctl reference | [kernel.org — ip-sysctl](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt) |
| `sysctl` | [man7.org — sysctl(8)](https://man7.org/linux/man-pages/man8/sysctl.8.html) |

---

## Checkpoint

**Q1. What does `rp_filter=1` do? When would you set it to `0` or `2`?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`rp_filter=1` is **strict reverse-path filtering**: when a packet arrives, the kernel checks whether a reply to that packet's *source* address would be routed back out the *same* interface the packet arrived on. If not, the packet is dropped. This is an anti-spoofing defense — it discards packets whose source addresses are implausible for the interface they came in on (a common trait of spoofed traffic).

- **Set it to `1` (strict)** on normal single-homed hosts and routers where every source legitimately has a symmetric path — it's good, cheap anti-spoofing and is the recommended default there.
- **Set it to `2` (loose)** on multi-homed hosts or routers with intentional **asymmetric routing**, where traffic legitimately arrives on one interface but the return path uses another. Loose mode only checks that the source is reachable via *some* interface, not the same one, so it permits valid asymmetric flows while still dropping truly unroutable sources.
- **Set it to `0` (off)** only when you have a specific reason to disable the check entirely and another mechanism handles anti-spoofing — it's rarely advisable because you lose the protection.

The trap: strict RPF silently drops legitimate asymmetric traffic, so on multi-homed routers it's a frequent cause of mysterious packet loss, and the fix is usually `2`, not `0`.
</details>

---

**Q2. You set `net.ipv4.conf.all.rp_filter=2` but packets are still being dropped by strict RPF on `eth0`. Why might the `all` setting not be taking effect?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because for `rp_filter`, the **effective value is the maximum of `conf/all/rp_filter` and `conf/<iface>/rp_filter`**, not just the `all` value. If `net.ipv4.conf.eth0.rp_filter` is still `1` (strict), then `max(2, 1) = 2`... but the gotcha is the other direction: if `all` is `1` and you only set the interface to `2`, the max is still `1` (strict). More commonly, the per-interface value was left at strict (`1`) by a distro default while you only lowered `all`, and since the kernel takes the max, the stricter per-interface setting wins for that interface. The fix is to set **both** `conf/all/rp_filter` *and* the specific `conf/<iface>/rp_filter` (and often `conf/default` for interfaces created later) to the value you want. This per-interface-vs-all interaction — where the kernel combines them (max for rp_filter) rather than letting `all` override — is a classic source of confusion: changing only `all` doesn't guarantee the behavior you expect on a specific interface.
</details>

---

**Q3. Why does changing a sysctl with `sysctl -w` take effect immediately but not survive a reboot?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`sysctl -w` writes directly to the live kernel's in-memory parameter (via the `/proc/sys/` virtual filesystem), so the new value is active **instantly** — the kernel consults that memory location on the next relevant operation, no restart needed. But `/proc/sys` is *not* persistent storage; it's a window into the running kernel's RAM. On **reboot**, the kernel starts fresh and initializes every sysctl to its built-in default (or whatever the boot-time configuration specifies), discarding any runtime `-w` changes. To make a change **survive reboots**, you must record it in a configuration file that's applied during boot — `/etc/sysctl.conf` or a file in `/etc/sysctl.d/` — which the system reads and applies at startup (via `sysctl -p` / the systemd sysctl service). That's the subject of the next lesson: runtime change (`-w`) for "now," config file for "persist across reboots." The two-step model (apply live, then persist) is standard for kernel tuning.
</details>

---

## Homework

Investigate `net.ipv4.tcp_congestion_control`. Check the current algorithm and the available ones. If `bbr` is available, switch to it (`sysctl -w net.ipv4.tcp_congestion_control=bbr`) and run an `iperf3` test over a `netem`-impaired link (high delay + slight loss) before and after, comparing throughput. Explain at a high level why a different congestion-control algorithm can produce different throughput on the *same* lossy, high-latency link.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Procedure:** `sysctl net.ipv4.tcp_congestion_control` shows the current default (often `cubic`); `sysctl net.ipv4.tcp_available_congestion_control` lists options (cubic, reno, bbr, …). Set up a veth with `netem delay 80ms loss 1%` on both ends, run `iperf3` with `cubic`, then switch the sender to `bbr` and re-test.

**Typical result:** on a high-latency, lossy link, **BBR usually achieves noticeably higher throughput than CUBIC**, sometimes by a large margin.

**Why the algorithm changes throughput on the same link:** congestion-control algorithms differ in *how they interpret signals and decide their sending rate*:

- **Loss-based algorithms (CUBIC, Reno)** treat **packet loss as the primary congestion signal**. When they see loss, they cut the congestion window (Reno halves it; CUBIC reduces and regrows along a cubic curve). On a link with *non-congestion* loss (random corruption/drops, as on wireless or our netem 1% loss), they misread that loss as congestion and **needlessly throttle**, so throughput collapses well below the link's real capacity — especially when high RTT makes window recovery slow (the BDP problem from Lesson 30).

- **BBR (Bottleneck Bandwidth and RTT)** ignores loss as the main signal and instead **models the path** — it actively estimates the bottleneck *bandwidth* and the *minimum RTT*, then paces sending to match the bandwidth-delay product. Because it isn't spooked by random loss, it keeps the pipe fuller on lossy links and tolerates high RTT better.

So on the *same* physical link, the throughput you get depends on the algorithm's *philosophy*: loss-based controllers conflate random loss with congestion and back off too aggressively, while BBR's rate-based model sustains higher utilization. This is exactly why operators tune `tcp_congestion_control` (e.g., to BBR) for long-haul, lossy, or high-BDP paths — the link didn't change, but the sender's *strategy* did, and strategy determines how much of the link a TCP flow can actually use.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 35 — sysctl Persistence →](lesson-35-sysctl-persistence){: .btn .btn-primary }
