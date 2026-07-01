---
title: "Lesson 38 — XDP: Express Data Path"
nav_order: 38
parent: "Phase 11: eBPF"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 38: XDP — Express Data Path

## Concept

**XDP** (eXpress Data Path) is the **earliest** hook in the Linux receive path. Your eBPF program runs **inside the NIC driver**, the instant a packet's bytes land in the receive ring — *before* the kernel allocates a socket buffer (`skb`), before routing, before netfilter, before anything. That's what makes it the fastest place to drop, redirect, or rewrite a packet.

```
   wire → NIC → [ driver RX ]
                     │
                  ┌──┴── XDP program runs here ── returns a verdict
                  │
                  ▼
   XDP_DROP   → free the page, packet never existed       (cheapest possible drop)
   XDP_PASS   → build skb, continue up the normal stack    (routing, nft, sockets...)
   XDP_TX     → send it back out the same NIC ("hairpin")
   XDP_REDIRECT → send it out another NIC or to a socket
```

Compare with everything you've learned: an `nft drop` happens *after* the skb is built and the packet has traveled some way up the stack. `XDP_DROP` happens before the skb even exists — so a machine under a packet flood can discard millions of packets per second per core while barely touching CPU. This is the basis of modern DDoS mitigation and software load balancers (Facebook's Katran, Cilium).

---

## How it works

The XDP program receives a tiny context, `struct xdp_md`, exposing `data` and `data_end` — raw pointers to the start and end of the packet. You parse headers manually, **bounds-checking every access** against `data_end` (the verifier insists), then return a verdict.

**Verdicts:**

| Return code | Effect |
|---|---|
| `XDP_DROP` | Discard immediately, free the buffer. No skb, no trace upstream. |
| `XDP_PASS` | Hand the packet to the normal stack as usual. |
| `XDP_TX` | Transmit back out the **same** interface (e.g., a reflector or simple LB). |
| `XDP_REDIRECT` | Send out a **different** interface or into an AF_XDP socket. |
| `XDP_ABORTED` | Error path — drop and fire a tracepoint (for debugging). |

**Native vs generic XDP.** *Native* XDP runs in the NIC driver's own RX routine — maximum speed, but requires driver support. *Generic* (a.k.a. SKB-mode) XDP runs a bit later, in the stack, after the skb is allocated — it works on **any** interface (including `veth` and `dummy`) but loses much of the speed advantage. It exists so you can develop and test XDP anywhere, then deploy on hardware that supports native mode. `veth` and many virtual devices only do generic mode (veth also supports native XDP in recent kernels).

**Attaching.** `ip link set dev eth0 xdp obj prog.o sec xdp` loads native (falls back / errors depending on driver); `xdpgeneric` forces generic mode; `xdp off` detaches. `bpftool net show` lists what's attached where.

{: .note }
> **Why XDP_DROP beats nft drop for floods**
> `XDP_DROP` frees the packet's buffer while it's still just bytes in the RX ring — no `skb` is allocated, and none of the routing/conntrack/netfilter machinery runs. An `nft drop` only fires *after* the kernel has built the skb and walked part of the stack to reach the relevant hook, so every dropped packet still cost a memory allocation and stack traversal. Under a high-rate flood that per-packet setup cost is exactly what saturates the CPU — moving the drop to XDP removes it.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip link set dev <if> xdp obj prog.o sec xdp` | Attach XDP (native if supported) |
| `ip link set dev <if> xdpgeneric obj prog.o sec xdp` | Force generic (SKB-mode) XDP |
| `ip link set dev <if> xdp off` | Detach XDP |
| `bpftool net show` | Show programs attached to network hooks |
| `ip link show <if>` | The `prog/xdp id N` field shows an attached program |

---

## Lab

We'll build an XDP **blocklist drop**: drop any IPv4 packet whose source is in a `HASH` map, and add/remove IPs from userspace live. (We use generic XDP on a `veth` so it runs anywhere.)

### Step 1 — The program

```c
// xdp_block.c — drop packets whose source IP is in the blocklist map
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key, __u32);     // source IP
    __type(value, __u8);    // any value = "blocked"
} blocklist SEC(".maps");

SEC("xdp")
int xdp_block(struct xdp_md *ctx) {
    void *data = (void *)(long)ctx->data, *end = (void *)(long)ctx->data_end;
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > end) return XDP_PASS;
    if (eth->h_proto != __builtin_bswap16(ETH_P_IP)) return XDP_PASS;
    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > end) return XDP_PASS;

    __u32 src = ip->saddr;
    if (bpf_map_lookup_elem(&blocklist, &src))
        return XDP_DROP;     // blocked source: discard at the driver
    return XDP_PASS;
}
char _license[] SEC("license") = "GPL";
```

### Step 2 — Build a two-namespace veth lab

```bash
$ clang -O2 -g -target bpf -c xdp_block.c -o xdp_block.o
$ sudo ip netns add a; sudo ip netns add b
$ sudo ip link add veth-a netns a type veth peer name veth-b netns b
$ sudo ip netns exec a ip addr add 10.0.0.1/24 dev veth-a
$ sudo ip netns exec b ip addr add 10.0.0.2/24 dev veth-b
$ sudo ip netns exec a ip link set veth-a up
$ sudo ip netns exec b ip link set veth-b up
$ sudo ip netns exec a ping -c2 10.0.0.2          # works
```

### Step 3 — Attach XDP on b's interface (generic mode)

```bash
$ sudo ip netns exec b ip link set dev veth-b xdpgeneric obj xdp_block.o sec xdp
$ sudo ip netns exec b ip link show veth-b
... prog/xdp id N ...
$ sudo ip netns exec b bpftool net show
xdp:
veth-b(...) generic id N
```

Blocklist is empty, so traffic still flows:

```bash
$ sudo ip netns exec a ping -c2 10.0.0.2          # still works
```

### Step 4 — Block 10.0.0.1 from userspace (no reload)

```bash
# find the map id
$ sudo bpftool map list | grep blocklist
M: hash  name blocklist ...

# key = 10.0.0.1 in network byte order = 0a 00 00 01 → bytes "0a 00 00 01"
$ sudo bpftool map update id M key 0x0a 0x00 0x00 0x01 value 0x01
$ sudo ip netns exec a ping -c2 -W1 10.0.0.2       # now 100% loss — dropped at XDP
```

### Step 5 — Unblock and confirm

```bash
$ sudo bpftool map delete id M key 0x0a 0x00 0x00 0x01
$ sudo ip netns exec a ping -c2 10.0.0.2           # works again
```

The program never reloaded — only the map changed.

### Step 6 — Clean up

```bash
$ sudo ip netns exec b ip link set dev veth-b xdpgeneric off
$ sudo ip netns delete a; sudo ip netns delete b
```

---

## Further Reading

| Topic | Link |
|---|---|
| XDP | [Wikipedia — Express Data Path](https://en.wikipedia.org/wiki/Express_Data_Path) |
| XDP overview | [kernel.org — XDP](https://docs.kernel.org/networking/xdp.html) |
| XDP tutorial | [xdp-project tutorial](https://github.com/xdp-project/xdp-tutorial) |
| `bpftool net` | [man7.org — bpftool-net(8)](https://man7.org/linux/man-pages/man8/bpftool-net.8.html) |
| AF_XDP sockets | [kernel.org — AF_XDP](https://docs.kernel.org/networking/af_xdp.html) |

---

## Checkpoint

**Q1. What is the performance advantage of `XDP_DROP` over an nftables `drop` rule?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`XDP_DROP` runs in the NIC driver's receive path **before the kernel allocates a socket buffer (`skb`)** and before any of the networking stack (routing, conntrack, netfilter) executes. Dropping there is almost free — the packet's buffer is simply recycled. An nftables `drop`, by contrast, only fires once the packet has reached a netfilter hook, which means the kernel has *already* built the skb and walked part of the stack to get there; that per-packet allocation and traversal cost is paid even for packets you immediately discard. Under a high-rate flood, that setup cost is precisely what exhausts CPU, so moving the drop all the way forward to XDP lets a single core discard millions of packets per second — which is why DDoS mitigation uses XDP.
</details>

---

**Q2. What is the difference between native and generic XDP, and why does generic mode exist?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Native XDP** runs inside the **NIC driver's** RX routine, at the earliest possible point, giving the full performance benefit — but it requires the driver to implement XDP support. **Generic (SKB-mode) XDP** runs later, in the generic networking stack *after* the `skb` has been allocated; it's slower (you've lost the "before skb" advantage) but it works on **any** interface, including virtual ones like `veth` and `dummy` regardless of driver support. Generic mode exists so you can **develop and test** XDP programs anywhere — on a laptop, in namespaces, on virtual interfaces — and then deploy the same program in native mode on hardware that supports it. It's a portability/development fallback, not a production performance target.
</details>

---

**Q3. Name the four main XDP verdicts and what each does.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>

- **`XDP_DROP`** — discard the packet immediately and recycle its buffer; no skb is built and nothing upstream ever sees it. The cheapest possible drop.
- **`XDP_PASS`** — let the packet continue into the normal kernel stack (skb allocation, routing, netfilter, sockets) as if XDP weren't there.
- **`XDP_TX`** — bounce the packet back out the **same** interface it arrived on (a "hairpin"), useful for reflectors or simple in-place load balancing.
- **`XDP_REDIRECT`** — send the packet out a **different** interface, or into an AF_XDP socket for userspace processing.

(There's also `XDP_ABORTED`, an error verdict that drops and fires a tracepoint for debugging.)
</details>

---

## Homework

Extend the blocklist program to also **count** how many packets it dropped per blocked source IP (combine Lesson 37's per-key counting with this lesson's drop). Then write a one-line shell loop that polls the count map with `bpftool map dump` once a second while you flood traffic from a blocked IP, and watch the drop counter climb — all while the program stays loaded. Finally, explain why a real DDoS-mitigation system structures itself exactly this way: fixed fast-path program + map-driven blocklist + map-based telemetry.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You'd add a second map (e.g. a `PERCPU_HASH` keyed by source IP, value = drop count) and, on the `XDP_DROP` branch, increment that key before returning. Polling it with a `while true; do bpftool map dump id <counts>; sleep 1; done` loop shows the per-IP drop totals rising in real time as you flood, with no reload.

A production DDoS scrubber is built this way because the three concerns have very different change rates and constraints: (1) the **fast path** — parse + lookup + verdict — must be tiny, verified, and JIT-compiled so it runs at line rate per core; you don't want to touch it often. (2) The **policy** (which IPs to drop) changes constantly as attacks shift, so it lives in a **map** that the control plane updates atomically per entry, taking effect on the next packet with zero downtime. (3) **Telemetry** (what got dropped, how much) also lives in **maps** (per-CPU for lock-free counting) that userspace samples on its own schedule without disturbing the dataplane. Separating fixed-fast-logic from frequently-changing-data from observe-on-read counters is what lets the system be simultaneously fast, dynamically reconfigurable, and observable — the same control-plane/dataplane split behind Cilium and Katran.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 39 — nftbpf: BPF in nftables →](lesson-39-nftbpf){: .btn .btn-primary }
