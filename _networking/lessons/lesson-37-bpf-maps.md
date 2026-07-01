---
title: "Lesson 37 — BPF Maps"
nav_order: 37
parent: "Phase 11: eBPF"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 37: BPF Maps

## Concept

An eBPF program runs at a hook, sees one event, and exits. By itself it has **no memory** — the next packet starts from scratch, and userspace can't see anything it computed. A **BPF map** is the shared key/value store that fixes both problems: it persists state *between* invocations of a program, and it's the bridge between the **kernel** (the eBPF program) and **userspace** (your control plane).

```
   userspace (bpftool / your C program)
        │  read / write via syscalls
        ▼
   ┌──────────── BPF MAP (key → value) ────────────┐
   │   10.0.0.5 → 42      10.0.0.6 → 7    ...        │
   └────────────────────────────────────────────────┘
        ▲
        │  lookup / update from inside the kernel
   eBPF program (XDP, TC, kprobe, ...)
```

Think of a map as a **shared spreadsheet**: the eBPF program scribbles counts into it on every packet, and userspace reads the totals whenever it likes — neither has to stop the other. This is how an XDP packet counter, a blocklist, a load-balancer backend table, or a connection tracker all work.

---

## How it works

A map is created with a fixed **type**, **key size**, **value size**, and **max entries**. The eBPF program references the map and calls helpers like `bpf_map_lookup_elem()` and `bpf_map_update_elem()`; userspace uses the `bpf()` syscall (wrapped by libbpf or `bpftool`) to read and write the same map.

**Common map types:**

| Type | Shape | Typical use |
|---|---|---|
| `BPF_MAP_TYPE_HASH` | general key → value hash | blocklists, per-IP state |
| `BPF_MAP_TYPE_ARRAY` | integer index → value | fixed counters, config slots |
| `BPF_MAP_TYPE_LRU_HASH` | hash that evicts oldest | bounded caches (conntrack-style) |
| `BPF_MAP_TYPE_PERCPU_HASH` | one copy of the value **per CPU** | lock-free counters |
| `BPF_MAP_TYPE_RINGBUF` | streaming buffer to userspace | events/logs from kernel → user |

**Why PERCPU matters.** A plain `HASH` keeps a single value per key. If 16 CPUs all try to `+1` the same counter at once, the kernel must serialize those updates (an atomic op or lock) — contention that costs cycles on a hot path. A `PERCPU_HASH` gives every CPU its **own private copy** of the value, so each core increments *its* copy with **no locking and no cache-line bouncing**. When userspace reads the map, it gets one value per CPU and **sums them** to get the total. The trade-off: slightly more memory and you must remember to aggregate across CPUs when reading.

**Pinning.** A map normally lives only as long as something references it. **Pinning** it to the BPF filesystem (`/sys/fs/bpf/...`) gives it a persistent path, so it survives after the loading process exits and can be shared by other programs and tools by name.

{: .note }
> **Map vs program lifetime**
> A map exists while *any* reference to it exists — a loaded program that uses it, an open file descriptor, or a pin. Pinning to `/sys/fs/bpf` is how you keep a map (or program) alive independently of the process that created it, which is exactly what you need when userspace wants to keep reading counters after the loader exits.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `bpftool map list` | List all loaded maps |
| `bpftool map show id <N>` | Details of one map (type, key/value size, entries) |
| `bpftool map dump id <N>` | Dump every key/value in the map |
| `bpftool map lookup id <N> key <bytes>` | Read one entry |
| `bpftool map update id <N> key <bytes> value <bytes>` | Write one entry |
| `bpftool map pin id <N> /sys/fs/bpf/<name>` | Pin a map to the BPF filesystem |

---

## Lab

We'll write an XDP program that **counts received packets per source IPv4 address** in a `HASH` map, then read the counts from userspace with `bpftool`.

### Step 1 — The program with a map

```c
// xdp_count.c — count packets per source IPv4 address
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 1024);
    __type(key, __u32);     // source IP (network byte order)
    __type(value, __u64);   // packet count
} pkt_count SEC(".maps");

SEC("xdp")
int xdp_counter(struct xdp_md *ctx) {
    void *data     = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;

    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end) return XDP_PASS;   // bounds check
    if (eth->h_proto != __builtin_bswap16(ETH_P_IP)) return XDP_PASS;

    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end) return XDP_PASS;    // bounds check

    __u32 src = ip->saddr;
    __u64 *cnt = bpf_map_lookup_elem(&pkt_count, &src);
    if (cnt) {
        __sync_fetch_and_add(cnt, 1);   // existing key: atomically ++
    } else {
        __u64 one = 1;
        bpf_map_update_elem(&pkt_count, &src, &one, BPF_ANY);
    }
    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```

Note the two **bounds checks** against `data_end` — without them the verifier rejects the program (Lesson 36).

### Step 2 — Compile and load into a namespace

```bash
$ clang -O2 -g -target bpf -c xdp_count.c -o xdp_count.o
$ sudo ip netns add lab
$ sudo ip netns exec lab ip link add eth0 type dummy
$ sudo ip netns exec lab ip addr add 10.0.0.1/24 dev eth0
$ sudo ip netns exec lab ip link set eth0 up
$ sudo ip netns exec lab ip link set dev eth0 xdp obj xdp_count.o sec xdp
```

### Step 3 — Find the map and inspect it

```bash
$ sudo bpftool map list
N: hash  name pkt_count  flags 0x0
    key 4B  value 8B  max_entries 1024  memlock ...

$ sudo bpftool map show id N
N: hash  name pkt_count  key 4B  value 8B  max_entries 1024 ...
```

### Step 4 — Generate traffic and read counts

```bash
# generate a few packets toward the dummy iface's subnet
$ sudo ip netns exec lab ping -c 3 10.0.0.2 || true

$ sudo bpftool map dump id N
key: 01 00 00 0a   value: 03 00 00 00 00 00 00 00
#     ^ 10.0.0.1 (little-endian bytes)  ^ count = 3
```

`bpftool` prints raw bytes. The key `01 00 00 0a` is `10.0.0.1` in network byte order; the value `03 00...` is the little-endian count `3`.

### Step 5 — Pin the map so it outlives the loader

```bash
$ sudo bpftool map pin id N /sys/fs/bpf/pkt_count
$ ls -l /sys/fs/bpf/pkt_count
$ sudo bpftool map dump pinned /sys/fs/bpf/pkt_count    # read by path now
```

### Step 6 — Clean up

```bash
$ sudo ip netns exec lab ip link set dev eth0 xdp off
$ sudo rm -f /sys/fs/bpf/pkt_count
$ sudo ip netns delete lab
```

---

## Further Reading

| Topic | Link |
|---|---|
| BPF maps | [kernel.org — BPF maps](https://docs.kernel.org/bpf/maps.html) |
| `bpf()` syscall | [man7.org — bpf(2)](https://man7.org/linux/man-pages/man2/bpf.2.html) |
| `bpftool map` | [man7.org — bpftool-map(8)](https://man7.org/linux/man-pages/man8/bpftool-map.8.html) |
| BPF filesystem / pinning | [kernel.org — BPF design Q&A](https://docs.kernel.org/bpf/bpf_design_QA.html) |
| eBPF overview | [ebpf.io — what is eBPF](https://ebpf.io/what-is-ebpf/) |

---

## Checkpoint

**Q1. Why is `BPF_MAP_TYPE_PERCPU_HASH` faster than `BPF_MAP_TYPE_HASH` for packet counters?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A plain `HASH` stores a **single shared value** per key. On a multi-core machine, many CPUs process packets in parallel, and if they all increment the *same* counter they must synchronize — via an atomic operation or a lock — to avoid lost updates. That synchronization serializes the updates and bounces the value's cache line between cores, which is expensive on a per-packet hot path.

A `PERCPU_HASH` gives **each CPU its own private copy** of the value for a key. Every core increments *its own* copy with **no locking and no cross-core cache contention** — each copy lives in that core's cache. The cost is paid only at read time: userspace gets one value per CPU and must **sum them** to obtain the true total (and it uses a bit more memory). For a counter that is written constantly and read rarely, trading a cheap aggregation-on-read for lock-free writes is a big win.
</details>

---

**Q2. An eBPF program computes per-IP counts, but when you stop the loader the counts vanish. What did you forget, and how do you fix it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You forgot to **pin** the map. A map lives only as long as something references it — the loaded program, an open file descriptor, or a pin. When the loader process exits and the program is detached, the last reference disappears and the kernel frees the map, taking your counts with it. **Pinning** it to the BPF filesystem (`bpftool map pin id N /sys/fs/bpf/pkt_count`) creates a persistent reference at a path, so the map (and its data) survives the loader exiting and can be read later by any tool via that path. (Pinning the *program* similarly keeps it loaded.)
</details>

---

**Q3. What role does a BPF map play that a local variable inside the eBPF program cannot?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A local variable lives on the program's stack and is **gone the instant the program returns** — each invocation (each packet/event) starts fresh, and userspace can never see it. A map provides the two things a local can't: (1) **persistence across invocations** — state computed for one packet is still there for the next; and (2) a **shared channel between kernel and userspace** (and between multiple eBPF programs) — userspace can read and write the same key/value store via the `bpf()` syscall while the program runs. Anything that must accumulate over time (counters, caches, conntrack state) or be configured/observed from outside (blocklists, backend tables) must live in a map.
</details>

---

## Homework

Extend the lab: instead of just counting, turn the map into a **blocklist**. Add a second `HASH` map keyed by source IP whose presence means "drop." In the XDP program, look up the source IP in the blocklist first and `return XDP_DROP` if found, otherwise continue counting. Then, *from userspace only* (`bpftool map update`), add and remove an IP from the blocklist while the program keeps running, and confirm with `ping` that traffic from that IP is dropped and restored — without ever reloading the program. Explain why this "update the map, not the program" pattern is the core idea behind dynamic eBPF tooling.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The program stays loaded and unchanged; the **policy lives in the map**. Adding an IP with `bpftool map update id <blocklist> key <ip-bytes> value 01` makes the very next packet from that IP hit the `bpf_map_lookup_elem` → non-NULL → `XDP_DROP` path, so `ping` from it immediately fails. Deleting the key (`bpftool map delete ...`) restores it just as fast. No recompile, no detach/reattach, no dropped packets during the change.

This is the central pattern of production eBPF systems (Cilium, load balancers, DDoS scrubbers): the **dataplane** (the verified, JIT-compiled program) is fixed and fast, while the **control plane** (userspace) steers behavior by updating maps. Map updates are cheap, atomic per entry, and take effect on the next packet, so you get dynamic, hot-reconfigurable policy without ever touching the kernel-side code path. Separating "fast fixed logic" from "frequently changing data" is what makes eBPF both safe and operationally practical.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 38 — XDP: Express Data Path →](lesson-38-xdp){: .btn .btn-primary }
