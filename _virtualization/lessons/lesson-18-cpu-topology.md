---
title: "Lesson 18 — CPU Topology"
nav_order: 18
parent: "Phase 5: CPU & Memory Configuration"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 18: CPU Topology

## Concept

`-smp 8` gives a guest eight vCPUs — but *how* those eight are arranged matters.
Are they 8 separate sockets? 1 socket with 4 cores × 2 threads? 2 sockets × 4
cores? The **topology** changes how the guest's scheduler behaves, how it counts
licenses, and how it reasons about cache and memory locality.

```
   -smp 8                          ── 8 vCPUs, default layout (8 sockets!)

   -smp 8,sockets=1,cores=4,threads=2
        ┌─────────── socket 0 ───────────┐
        │ core0[t0 t1] core1[t0 t1]       │   ← threads share a core's
        │ core2[t0 t1] core3[t0 t1]       │     execution units (SMT)
        └─────────────────────────────────┘
```

The guest *trusts* what you tell it. Present a nonsensical topology and the guest's
scheduler makes bad placement decisions; present an honest one and it optimizes
correctly.

---

## How It Works

### The `-smp` breakdown

```
   -smp cpus=8,sockets=1,cores=4,threads=2
   #     │       │         │        └ SMT threads per core
   #     │       │         └ cores per socket
   #     │       └ number of CPU sockets
   #     └ total vCPUs  (must equal sockets × cores × threads)
```

If you give just `-smp 8`, QEMU historically defaulted to **8 sockets × 1 core × 1
thread** — which is unusual hardware and can confuse guests (and licensing). It's
good practice to spell out a realistic topology.

### Why the guest cares

- **Scheduler decisions:** the guest OS scheduler uses topology to decide where to
  place threads. It will avoid putting two busy threads on *SMT siblings of the same
  core* (they share execution units) and prefer spreading across cores; it groups
  cooperating threads by shared cache. A wrong topology defeats this.
- **Cache/locality assumptions:** the guest assumes cores in a socket share an L3
  cache and that crossing sockets is costlier — which feeds NUMA-aware placement
  (Lesson 21).
- **Licensing:** lots of commercial software (databases, Windows editions) is
  licensed **per socket**. Presenting 8 sockets instead of 1 socket × 8 cores can
  multiply your license cost — a real-world reason topology matters.

### Overcommitting vCPUs

You can assign more total vCPUs across all guests than you have physical cores
(**overcommit**). The host scheduler time-slices physical cores among vCPU threads.
Modest overcommit is fine for bursty/idle guests; heavy overcommit causes
**vCPU steal time** (a vCPU is runnable but waiting for a physical core), which the
guest sees as mysterious slowness. A common starting guideline is to keep the
vCPU:pCPU ratio low for latency-sensitive guests (near 1:1) and allow higher ratios
only for idle/bursty workloads.

### Toward pinning

By default any vCPU thread can run on any physical core, and the host scheduler
moves them around. For latency-sensitive guests you **pin** vCPUs to specific
physical cores so they don't migrate and don't share cores with noisy neighbors.
That's the full subject of Lesson 50 (Phase 13); here just know the default is
"float freely."

{: .note }
> **SMT siblings and the topology lie**
> If you tell the guest <code>threads=2</code>, it believes pairs of vCPUs are SMT
> siblings sharing one physical core, and schedules accordingly (avoiding piling two
> hot threads onto sibling vCPUs). But unless you also *pin* those sibling vCPUs to
> real sibling hyperthreads, the mapping is fiction — the host may place them on
> unrelated cores. For the guest's topology assumptions to be true, topology and
> pinning must agree (Lesson 50).

---

## Lab

```bash
# 1. Boot with an explicit, realistic topology:
$ qemu-system-x86_64 -accel kvm -m 4G \
    -smp cpus=8,sockets=1,cores=4,threads=2 \
    -cpu host -drive file=disk.qcow2,if=virtio -nographic &

# 2. Inside the guest, confirm the topology it sees:
#    (guest)$ lscpu
#    CPU(s):                8
#    Thread(s) per core:    2
#    Core(s) per socket:    4
#    Socket(s):             1
#    (guest)$ lscpu -e        # per-CPU: which socket/core/thread each vCPU is

# 3. Contrast: boot with the bare default and see the odd 8-socket layout:
$ qemu-system-x86_64 -accel kvm -m 4G -smp 8 -cpu host \
    -drive file=disk.qcow2,if=virtio -nographic &
#    (guest)$ lscpu | grep -E 'Socket|Core|Thread'
#    Socket(s): 8   Core(s) per socket: 1   Thread(s) per core: 1   ← unusual!

# 4. Observe overcommit on the HOST: total guest vCPUs vs physical CPUs.
$ nproc                       # physical/logical CPUs on the host
8
$ ps -eLf | grep -c 'CPU .*/KVM'   # count vCPU threads across all running guests

# 5. Watch steal time appear in a guest under host CPU pressure:
#    (guest)$ vmstat 1        # the 'st' column = vCPU steal time
$ kill %1 %2 2>/dev/null
```

**Expected result:** With an explicit topology the guest's `lscpu` reports 1 socket
× 4 cores × 2 threads; with bare `-smp 8` it reports the unusual 8-socket layout.
Under overcommit, the guest's `vmstat` `st` column rises — the visible symptom of
too many vCPUs chasing too few physical cores.

---

## Further Reading

| Topic | Link |
|---|---|
| QEMU `-smp` / topology | [qemu.org — Invocation (-smp)](https://www.qemu.org/docs/master/system/invocation.html) |
| Simultaneous multithreading (SMT) | [Wikipedia — Simultaneous multithreading](https://en.wikipedia.org/wiki/Simultaneous_multithreading) |
| `lscpu` | [man7.org — lscpu(1)](https://man7.org/linux/man-pages/man1/lscpu.1.html) |
| CPU steal time | [Wikipedia — CPU time (steal)](https://en.wikipedia.org/wiki/CPU_time) |
| libvirt vCPU topology | [libvirt.org — CPU topology](https://libvirt.org/formatdomain.html#cpu-model-and-topology) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. A guest with 8 vCPUs presented as 8 sockets behaves differently from 1 socket × 4 cores × 2 threads. Why might the guest care?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The guest's scheduler and subsystems make decisions based on the topology it's told. With 8 sockets it assumes 8 independent CPUs with no shared cache/SMT, so it won't optimize cache locality or avoid SMT-sibling contention; it may also assume a multi-NUMA layout. With 1 socket × 4 cores × 2 threads it knows cores share an L3 cache and that thread pairs are SMT siblings sharing execution units, so it spreads hot threads across cores and co-locates cooperating ones. Additionally, per-socket-licensed software would count 8 sockets and could cost far more. So the same 8 vCPUs yield different scheduling, locality, and licensing behavior.
</details>

---

**Q2. What is vCPU overcommit, and what symptom does excessive overcommit produce inside a guest?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Overcommit is assigning more total vCPUs across all guests than the host has physical cores; the host scheduler time-slices physical cores among the vCPU threads. Excessive overcommit produces vCPU steal time — periods where a guest's vCPU is runnable but can't get a physical core because others are using it. Inside the guest this appears as the "st" column in vmstat/top rising and manifests as unexplained slowness/latency even though the guest itself looks idle-ish.
</details>

---

**Q3. You set `threads=2` so the guest thinks vCPUs come in SMT-sibling pairs. Why is that only "true" if you also do something else?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Telling the guest threads=2 only changes what topology the guest *believes*; it doesn't force the host to place those sibling vCPUs on actual hyperthread siblings of one physical core. Without pinning, the host scheduler may run the two "sibling" vCPUs on entirely unrelated cores, so the guest's SMT-aware scheduling decisions are based on a fiction. To make the reported topology real you must also pin the sibling vCPU threads to the corresponding physical sibling hyperthreads (Lesson 50), so topology and actual placement agree.
</details>

---

## Homework

Boot a guest with `-smp cpus=4,sockets=2,cores=2,threads=1` and another with `-smp cpus=4,sockets=1,cores=2,threads=2`. In each guest, run `lscpu -e` and note how the 4 vCPUs map to sockets/cores/threads. Explain which layout a per-socket-licensed database would prefer, and which layout better reflects a single modern physical CPU.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The first layout shows 2 sockets, each 2 cores, 1 thread (4 independent cores across two sockets). The second shows 1 socket, 2 cores, 2 threads each (one physical CPU with SMT). A per-socket-licensed database prefers the second (1 socket) because it pays for one socket instead of two. The second also better reflects a single modern physical CPU, which is typically one socket with multiple cores and hyperthreading — so the guest's cache-locality and SMT-aware scheduling assumptions match reality (provided you also pin appropriately).
</details>
