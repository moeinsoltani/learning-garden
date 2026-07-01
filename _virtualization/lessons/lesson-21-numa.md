---
title: "Lesson 21 — NUMA Awareness"
nav_order: 21
parent: "Phase 5: CPU & Memory Configuration"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 21: NUMA Awareness

## Concept

On a multi-socket server, memory is not uniformly fast to reach. Each CPU socket has
its **own** memory attached directly (its **NUMA node**); reaching another socket's
memory means crossing an interconnect — slower and higher-latency. This is
**NUMA**: Non-Uniform Memory Access.

```
   ┌──── NUMA node 0 ────┐   interconnect   ┌──── NUMA node 1 ────┐
   │  CPU socket 0       │◄───(slower)─────►│  CPU socket 1       │
   │  local RAM (fast)   │                  │  local RAM (fast)   │
   └─────────────────────┘                  └─────────────────────┘

   vCPU on node 0 reading node-0 RAM  = fast  (local)
   vCPU on node 0 reading node-1 RAM  = slow  (remote, crosses interconnect)
```

The classic VM performance cliff: a guest whose **vCPUs run on one node but whose
memory lives on another** — every memory access is remote. Confining a VM to a
single node (vCPUs *and* memory together) avoids it.

---

## How It Works

### Host NUMA topology

First, know your host's layout. `numactl -H` and `lscpu` show how many NUMA nodes
exist, which CPUs belong to each, and how much memory each node has, plus a
**distance matrix** (relative cost of node-to-node access). A single-socket desktop
has one node (NUMA is moot); a dual-socket server has two or more.

### Confining a guest to one node

For a guest that fits within one node's resources, pin **both** its vCPUs and its
memory to that node:

- **vCPU placement:** pin the vCPU threads to that node's physical CPUs (full
  treatment in Lesson 50). At the QEMU level you can also set `-numa` mappings; with
  libvirt you use `<vcpu placement='static' cpuset=...>` and `<numatune>`.
- **Memory placement:** bind the guest's memory allocation to that node so its pages
  come from local RAM. `numactl --membind=`, or QEMU's `host-nodes=`/`policy=bind`
  on the memory backend, or libvirt `<numatune><memory mode='strict' nodeset=.../>`.

When vCPUs and memory share a node, accesses stay local → fast.

### Exposing a virtual NUMA topology to large guests

A guest *bigger* than one host node can't be confined — it must span nodes. For
those, you expose a **virtual NUMA topology** to the guest (`-numa node,...`) that
*mirrors* the host's, and pin each virtual node to a corresponding host node. Now
the **guest's own** NUMA-aware scheduler and allocator keep each workload's threads
near their memory, because the guest can finally see the locality structure. Without
a virtual topology, the guest thinks all its memory is uniform and places threads
blindly.

### Automatic NUMA balancing and numad

- **Automatic NUMA balancing** (kernel `numa_balancing`) tries to migrate pages and
  tasks toward locality on its own. Helpful for unpinned, general workloads — but for
  a **pinned** VM it just adds overhead and fights your explicit placement, so you
  typically **disable** it for pinned guests (Lesson 51).
- **`numad`** is a userspace daemon that monitors and nudges processes toward NUMA
  locality automatically — an alternative to manual pinning for dynamic workloads.

{: .note }
> **Why straddling two nodes is so much slower**
> If half a VM's memory is remote, a large fraction of its memory accesses pay the
> interconnect latency penalty (often 1.5–2× the local latency, sometimes more) and
> contend for limited interconnect bandwidth. For memory-bound workloads this can cut
> throughput substantially and add jitter. Confining the VM to one node — or, for big
> VMs, exposing a matching virtual NUMA topology and pinning it — keeps the hot path
> local. NUMA placement is one of the highest-impact tuning levers for large guests.

---

## Lab

```bash
# 1. Inspect the HOST NUMA topology:
$ numactl -H
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7
node 0 size: 64000 MB
node 1 cpus: 8 9 10 11 12 13 14 15
node 1 size: 64000 MB
node distances:
node   0   1
  0:  10  21        ← '10' = local, '21' = remote (≈2.1× cost)
  1:  21  10

# (single-node host? then NUMA tuning is a no-op — you'll see "1 nodes (0)")

# 2. Run a guest CONFINED to node 0: vCPUs on node-0 CPUs, memory from node 0.
$ numactl --cpunodebind=0 --membind=0 -- \
    qemu-system-x86_64 -accel kvm -m 16G -smp 8 -cpu host \
    -drive file=disk.qcow2,if=virtio -nographic &

# 3. Verify the QEMU process's memory came from node 0:
$ numastat -p $(pgrep -f 'name lab21\|qemu-system' | head -1) 2>/dev/null | head
                           Node 0          Node 1
Huge                          ...             ...
Heap                          ...             ...
Private                  16000.00            0.00     ← all on node 0, good

# 4. For a guest LARGER than one node, expose a matching virtual NUMA topology:
$ qemu-system-x86_64 -accel kvm -m 32G -cpu host \
    -smp 16,sockets=2,cores=8,threads=1 \
    -object memory-backend-ram,id=m0,size=16G,host-nodes=0,policy=bind \
    -object memory-backend-ram,id=m1,size=16G,host-nodes=1,policy=bind \
    -numa node,nodeid=0,cpus=0-7,memdev=m0 \
    -numa node,nodeid=1,cpus=8-15,memdev=m1 \
    -drive file=disk.qcow2,if=virtio -nographic &
#   (guest)$ numactl -H   # the guest now sees 2 NUMA nodes mirroring the host

# 5. Automatic NUMA balancing status (disable for pinned guests, Lesson 51):
$ cat /proc/sys/kernel/numa_balancing
1
$ kill %1 %2 2>/dev/null
```

**Expected result:** `numactl -H` reveals your node count and the remote-access cost
in the distance matrix. A guest launched under `numactl --cpunodebind=0 --membind=0`
has all its memory on node 0 (`numastat`), and a large guest given a matching
`-numa` topology shows two nodes inside the guest.

---

## Further Reading

| Topic | Link |
|---|---|
| Non-uniform memory access | [Wikipedia — Non-uniform memory access](https://en.wikipedia.org/wiki/Non-uniform_memory_access) |
| `numactl` | [man7.org — numactl(8)](https://man7.org/linux/man-pages/man8/numactl.8.html) |
| `numastat` | [man7.org — numastat(8)](https://man7.org/linux/man-pages/man8/numastat.8.html) |
| libvirt NUMA tuning | [libvirt.org — NUMA tuning](https://libvirt.org/formatdomain.html#numa-node-tuning) |
| Automatic NUMA balancing | [kernel.org — NUMA](https://docs.kernel.org/admin-guide/mm/numa_memory_policy.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why can a VM whose vCPUs and memory straddle two NUMA nodes be much slower than one confined to a single node?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
On a NUMA system each socket has local memory it reaches quickly; reaching another node's memory crosses an interconnect with higher latency (often ~1.5–2× or more) and limited bandwidth. If a VM's vCPUs run on one node but much of its memory lives on another, a large fraction of memory accesses are remote — paying that latency penalty and contending for interconnect bandwidth — which for memory-bound workloads sharply cuts throughput and adds jitter. Confining vCPUs and memory to the same node keeps accesses local and fast.
</details>

---

**Q2. For a guest too large to fit in one host NUMA node, what do you do instead of confining it, and why does that help?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You expose a *virtual* NUMA topology to the guest (e.g. via -numa) that mirrors the host's nodes, pinning each virtual node's vCPUs and memory to the corresponding host node. This helps because the guest's own NUMA-aware scheduler and allocator can then see the locality structure and keep each workload's threads near their memory. Without a virtual topology the guest believes all memory is uniform and places threads blindly, causing many remote accesses despite the underlying hardware being NUMA.
</details>

---

**Q3. Why do you typically disable automatic NUMA balancing for a pinned VM?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Automatic NUMA balancing migrates pages and tasks at runtime trying to improve locality, which helps unpinned, general workloads. But for a VM you've explicitly pinned (vCPUs and memory placed deliberately on chosen nodes), the balancer fights your placement — it may move pages/tasks against your intent and adds scanning/migration overhead for no benefit, since locality is already optimal. So you disable it for pinned guests to avoid the overhead and keep your manual placement stable.
</details>

---

## Homework

Run `numactl -H` on your host. If it reports more than one node, launch a guest under `numactl --cpunodebind=0 --membind=0` and confirm with `numastat -p <qemu-pid>` that its memory is on node 0. If your host has only one node, explain what that means for NUMA tuning and what hardware change would make NUMA relevant.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
On a multi-node host, the numactl-confined guest's numastat shows essentially all its Private memory on Node 0 and ~0 on other nodes — proof it's local. On a single-node host, numactl -H reports "1 nodes (0)" and all memory is equidistant, so NUMA tuning is a no-op: there's no remote memory to avoid and pinning to "node 0" is trivially satisfied. NUMA becomes relevant on multi-socket servers (or some large single-socket chips with multiple internal memory domains), where adding a second populated CPU socket creates a second node and thus the local-vs-remote distinction worth tuning for.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 22 — Disk Image Formats and qemu-img →](lesson-22-image-formats-qemu-img){: .btn .btn-primary }
