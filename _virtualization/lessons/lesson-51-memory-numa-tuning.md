---
title: "Lesson 51 — Memory and NUMA Tuning in Production"
nav_order: 51
parent: "Phase 13: Performance Tuning"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 51: Memory and NUMA Tuning in Production

## Concept

Lesson 21 introduced NUMA and Lesson 19 introduced hugepages. This lesson applies them
*together, in production*, for a pinned guest. The golden rule: **pin vCPUs and memory to
the same NUMA node, back them with hugepages, and stop the kernel from second-guessing
you.**

```
   THE COMPLETE FIX (single-node guest)
   ┌──────────── NUMA node 0 ────────────┐
   │  pinned vCPUs (Lesson 50)            │   <vcpupin cpuset=node0 cpus>
   │  memory bound to node 0  (strict)    │   <numatune mode=strict nodeset=0>
   │  backed by 1 GiB hugepages           │   <hugepages> + node-0 pool
   │  automatic NUMA balancing OFF         │   so the kernel won't migrate pages
   └──────────────────────────────────────┘
```

Half-doing this is the classic trap: you pin vCPUs to node 0 but leave memory unpinned, so
the guest's RAM ends up on node 1 — and every access is remote (Lesson 21).

---

## How It Works

### Hugepage pools sized for the guest set

For production you reserve hugepages at boot (so they're guaranteed and unfragmented,
Lesson 19), sized for *all* your VMs, on the *right node*:

- Reserve via kernel cmdline (`default_hugepagesz=1G hugepagesz=1G hugepages=32`) for
  **1 GiB** pages — best for large guests (fewest TLB entries).
- Per-node pools matter on NUMA: reserve hugepages on the node where the guest will live,
  so its huge-backed memory is local.
- libvirt: `<memoryBacking><hugepages><page size='1' unit='G' nodeset='0'/></hugepages>
  </memoryBacking>`.

### numatune and memnode — binding memory to a node

`<numatune>` controls *where guest memory comes from*:

```xml
   <numatune>
     <memory mode='strict' nodeset='0'/>          <!-- all guest RAM from node 0 -->
   </numatune>
```

- **`strict`** — memory *must* come from the nodeset; allocation fails rather than spill to
  another node. Use for pinned, performance-critical guests where remote memory is
  unacceptable.
- **`preferred`** — try the nodeset, but fall back to others if needed (softer).
- **`<memnode>`** — for multi-node (large) guests, bind each *virtual* NUMA node's memory
  to a specific host node (pairs with the virtual topology from Lesson 21).

Combined with vCPU pinning (Lesson 50) to the same node, every access is local.

### Disable automatic NUMA balancing for pinned guests

The kernel's **automatic NUMA balancing** migrates pages/tasks at runtime to chase
locality — helpful for *unpinned* workloads, but for a *pinned* guest it fights your
explicit placement and adds scanning/migration overhead for no benefit. Disable it:

```
   sysctl kernel.numa_balancing=0      # (or per the pinned-guest tuned profile)
```

Your manual placement is already optimal; let it stand.

### Avoid swap for VM hosts

A VM host should **not swap** guest memory: if QEMU's pages get swapped to disk, guest
memory access suddenly incurs disk latency — catastrophic and unpredictable for the guest.
Mitigations: don't overcommit memory on latency-sensitive hosts; use hugepages (hugetlbfs
pages are non-swappable by nature); set a low `vm.swappiness`; size RAM so you never need
swap. Hugepages neatly solve this for the guests that use them.

{: .note }
> **The "pinned vCPUs but unpinned memory" trap**
> If you pin a guest's vCPUs to node 0 but don't bind its memory, the kernel may allocate
> the guest's RAM on node 1 (e.g. node 0 was busy at allocation time). Now every memory
> access from the node-0 vCPUs crosses the interconnect to node-1 RAM — the exact NUMA
> penalty you were trying to avoid, and you may not notice because nothing "failed." The
> fix is to also bind memory with <code>&lt;numatune mode='strict' nodeset='0'/&gt;</code>
> (and back it with node-0 hugepages), so vCPUs and memory share the node and accesses stay
> local.

---

## Lab

```bash
# 1. Reserve 1 GiB hugepages at boot, sized for your guests (kernel cmdline):
#   GRUB_CMDLINE_LINUX_DEFAULT="... default_hugepagesz=1G hugepagesz=1G hugepages=16"
$ grep -i huge /proc/meminfo
HugePages_Total:      16
Hugepagesize:    1048576 kB        ← 1 GiB pages

# 2. Check per-NODE hugepage pools (reserve where the guest will run):
$ cat /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/nr_hugepages
16
$ cat /sys/devices/system/node/node1/hugepages/hugepages-1048576kB/nr_hugepages
0

# 3. Bind a guest's memory to node 0 (strict) + hugepages + vCPU pin (Lesson 50):
$ virsh edit bigvm
#   <memoryBacking>
#     <hugepages><page size='1' unit='G' nodeset='0'/></hugepages>
#   </memoryBacking>
#   <numatune>
#     <memory mode='strict' nodeset='0'/>
#   </numatune>
#   <cputune>
#     <vcpupin vcpu='0' cpuset='0'/> ... (node-0 cores)
#   </cputune>

# 4. Verify the guest's memory really landed on node 0:
$ numastat -p $(pgrep -f 'guest=bigvm')
                 Node 0          Node 1
Private        16000.00            0.00      ← all on node 0, as intended
Huge           16384.00            0.00

# 5. Disable automatic NUMA balancing for pinned guests:
$ cat /proc/sys/kernel/numa_balancing
1
$ sudo sysctl kernel.numa_balancing=0
kernel.numa_balancing = 0

# 6. Confirm no swapping pressure on the VM host:
$ free -h | grep Swap
$ cat /proc/sys/vm/swappiness        # keep low for VM hosts
```

**Expected result:** `numastat` confirms the guest's memory (including hugepages) is
entirely on node 0, matching its pinned vCPUs; automatic NUMA balancing is off; and the
host isn't swapping guest memory.

---

## Further Reading

| Topic | Link |
|---|---|
| libvirt NUMA tuning (`<numatune>`) | [libvirt.org — NUMA tuning](https://libvirt.org/formatdomain.html#numa-node-tuning) |
| libvirt memory backing / hugepages | [libvirt.org — Memory backing](https://libvirt.org/formatdomain.html#memory-backing) |
| HugeTLB pages | [kernel.org — HugeTLB](https://docs.kernel.org/admin-guide/mm/hugetlbpage.html) |
| Automatic NUMA balancing | [kernel.org — NUMA memory policy](https://docs.kernel.org/admin-guide/mm/numa_memory_policy.html) |
| `numastat` | [man7.org — numastat(8)](https://man7.org/linux/man-pages/man8/numastat.8.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. You pinned vCPUs to node 0 but left memory unpinned. What can go wrong, and what completes the fix?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The kernel may allocate the guest's RAM on a different node (e.g. node 1), so the node-0 vCPUs make remote memory accesses across the interconnect for everything — the full NUMA penalty (higher latency, less bandwidth), and you might not notice because nothing errored. The fix is to also bind the guest's memory to node 0, e.g. <code>&lt;numatune&gt;&lt;memory mode='strict' nodeset='0'/&gt;&lt;/numatune&gt;</code>, ideally backed by node-0 hugepages. With vCPUs and memory on the same node, accesses stay local. (You'd also disable automatic NUMA balancing so it doesn't migrate pages against your placement.)
</details>

---

**Q2. Why disable automatic NUMA balancing for a pinned guest?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Automatic NUMA balancing migrates pages and tasks at runtime to improve locality, which helps unpinned workloads. But a pinned guest already has vCPUs and memory deliberately placed for optimal locality, so the balancer can only fight that placement — moving pages/tasks against your intent — while adding continuous scanning and migration overhead for zero benefit. Disabling it (kernel.numa_balancing=0) removes that overhead and keeps your explicit, already-optimal placement stable.
</details>

---

**Q3. Why should a VM host avoid swapping guest memory, and how do hugepages help?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
If the host swaps out QEMU's pages, a guest memory access that should hit RAM instead triggers a disk read to swap back in — adding huge, unpredictable latency to the guest, which is catastrophic for performance and especially for latency-sensitive workloads. Hugepages help because hugetlbfs pages are non-swappable by nature: memory backed by explicit hugepages is reserved and pinned in RAM, so it can never be paged out. That guarantees the guest's huge-backed memory always has RAM latency. (Other mitigations: don't overcommit on such hosts, keep swappiness low, size RAM to never need swap.)
</details>

---

## Homework

Configure a guest with `<numatune mode='strict' nodeset='0'>`, node-0 hugepages, and vCPUs pinned to node-0 cores. Use `numastat -p <qemu-pid>` to prove all its memory is on node 0. Then temporarily remove the `<numatune>` binding (leaving vCPUs pinned), restart, and re-check `numastat`. Explain any difference and why the strict binding mattered. (If your host is single-node, explain what you'd expect on a dual-node host.)

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With strict numatune + node-0 hugepages + node-0 vCPU pinning, numastat shows the guest's Private and Huge memory entirely on Node 0 and ~0 on Node 1 — local to its vCPUs. After removing the numatune binding (vCPUs still pinned to node 0), on a dual-node host the guest's memory can be allocated partly or wholly on node 1 (wherever the kernel found free pages at allocation time), so numastat shows memory split across nodes or sitting on node 1 — meaning the node-0 vCPUs now make remote accesses, degrading bandwidth/latency. The strict binding mattered because it forces all guest RAM to come from node 0 (failing rather than spilling), guaranteeing locality with the pinned vCPUs. On a single-node host there's only Node 0, so both cases look identical and NUMA binding is a no-op — the distinction only appears with two or more populated nodes.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 52 — Storage and I/O Tuning →](lesson-52-storage-io-tuning){: .btn .btn-primary }
