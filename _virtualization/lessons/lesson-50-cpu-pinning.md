---
title: "Lesson 50 — CPU Pinning and Thread Placement"
nav_order: 50
parent: "Phase 13: Performance Tuning"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 50: CPU Pinning and Thread Placement

## Concept

By default the host scheduler moves vCPU threads around freely (Lesson 18). For
latency-sensitive guests (real-time, telco, trading, audio) that movement causes
**jitter** — unpredictable latency spikes. **Pinning** nails each vCPU thread to a
specific physical core so it stays put, and **thread placement** keeps QEMU's *other*
threads off the vCPU cores so they don't interfere.

```
   FLOATING (default)                   PINNED
   ──────────────────                   ──────
   vCPU0 ↔ any core (migrates)           vCPU0 → pCPU2   (fixed)
   vCPU1 ↔ any core                      vCPU1 → pCPU3
   emulator thread ↔ any core            emulator/iothreads → pCPU0,1 (separate!)
   → cache thrash, jitter                → stable caches, predictable latency
```

A VM isn't just vCPU threads — it also has an **emulator thread** (the QEMU main loop /
device model) and **iothreads** (disk). If those share cores with the vCPUs, they steal
cycles right when the guest needs them. Good placement separates them.

---

## How It Works

### vCPU → pCPU pinning

Bind each vCPU thread to a dedicated physical CPU:

- **virsh:** `virsh vcpupin <dom> <vcpu> <pcpu>` (live), or `<cputune><vcpupin
  vcpu='0' cpuset='2'/></cputune>` in the XML (persistent).
- The vCPU now runs *only* on that core — no migration, so its L1/L2 cache and TLB stay
  warm and latency is stable.

### Isolating the cores from the host

Pinning a vCPU to a core isn't enough if the *host kernel* still schedules other tasks
there. To give a vCPU a truly dedicated core, isolate it from the host scheduler via
kernel cmdline:

- **`isolcpus=2,3`** — the host scheduler won't place normal tasks on these CPUs.
- **`nohz_full=2,3`** — disable the periodic scheduler tick on them (no timer interrupts
  when one task runs) → fewer interruptions for a CPU-bound real-time vCPU.
- **`rcu_nocbs=2,3`** — offload RCU callback processing off these cores so RCU work
  doesn't interrupt the guest.

Together these turn cores 2,3 into quiet, dedicated lanes for pinned vCPUs.

### Placing the emulator and iothreads OFF the vCPU cores

The QEMU **emulator thread** (main loop, device emulation, timers) and **iothreads**
(Lesson 52) do real work that, if run on a vCPU's core, *preempts the guest's real-time
thread* at the worst moment. So you pin them to *different* cores:

- `<cputune><emulatorpin cpuset='0-1'/></cputune>` — emulator thread on the housekeeping
  cores.
- `<cputune><iothreadpin iothread='1' cpuset='0-1'/></cputune>` — iothreads likewise.

The vCPUs get cores 2–N (isolated); the "overhead" threads get cores 0–1 (housekeeping).
Now nothing on a vCPU core competes with the vCPU.

### Hyperthread siblings and noisy neighbors

If you told the guest `threads=2` (Lesson 18), the topology is only real if you pin sibling
vCPUs to actual HT siblings of one physical core. Also, two unrelated VMs (or host tasks)
on the *same physical core's HT siblings* contend for the core's execution units — a
"noisy neighbor." For latency-critical guests you either give them whole physical cores
(both siblings, or disable the sibling) so no one else shares their execution resources.

{: .note }
> **Why pin the emulator thread to different cores than the vCPUs (for a real-time guest)**
> A real-time guest needs its vCPU to respond within a tight deadline. The QEMU emulator
> thread and iothreads do device/IO work asynchronously; if they're scheduled on the same
> core as the vCPU, they preempt the vCPU exactly when an interrupt or device event fires —
> injecting latency precisely when the guest can least afford it. Pinning them to separate
> "housekeeping" cores means the vCPU's core is never stolen for emulation/IO work, so the
> guest's worst-case latency stays bounded. Isolation (isolcpus/nohz_full/rcu_nocbs) on the
> vCPU cores completes the picture by keeping the host kernel off them too.

---

## Lab

```bash
# 1. See the current (floating) vCPU placement:
$ virsh vcpuinfo web01
VCPU:           0
CPU:            5            ← currently on pCPU5, but can migrate anywhere
CPU Affinity:   yyyyyyyy      ← allowed on all CPUs (not pinned)

# 2. Pin vCPUs to dedicated physical cores (live):
$ virsh vcpupin web01 0 2
$ virsh vcpupin web01 1 3
$ virsh vcpuinfo web01 | grep -A1 'VCPU:\|Affinity'
CPU Affinity:   --y-----      ← vCPU0 now only on pCPU2

# 3. Persist pinning + place emulator/iothreads OFF the vCPU cores (XML):
$ virsh edit web01
#   <cputune>
#     <vcpupin vcpu='0' cpuset='2'/>
#     <vcpupin vcpu='1' cpuset='3'/>
#     <emulatorpin cpuset='0-1'/>
#     <iothreadpin iothread='1' cpuset='0-1'/>
#   </cputune>
#   <iothreads>1</iothreads>

# 4. Isolate the vCPU cores from the host scheduler (kernel cmdline) — needs reboot:
#   /etc/default/grub:
#   GRUB_CMDLINE_LINUX_DEFAULT="... isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3"
$ cat /sys/devices/system/cpu/isolated      # after reboot: 2-3
2-3

# 5. Verify the emulator thread is NOT on a vCPU core:
$ ps -To pid,tid,comm,psr -p $(pgrep -f 'guest=web01')
  PID   TID COMMAND          PSR
12345 12347 CPU 0/KVM          2     ← vCPU0 on core 2 (pinned)
12345 12348 CPU 1/KVM          3     ← vCPU1 on core 3
12345 12346 qemu-system-x86    0     ← emulator on core 0 (separate!)

# 6. Confirm reduced jitter: run a latency test (e.g. cyclictest) in the guest
#    before vs after pinning+isolation, comparing max latency.
#   (guest)$ cyclictest -p99 -t1 -n     # max latency should drop and stabilize
```

**Expected result:** After pinning, `vcpuinfo` shows each vCPU restricted to one physical
core, the emulator thread runs on a *different* (housekeeping) core, and the vCPU cores
appear under `/sys/.../cpu/isolated`. A latency test in the guest shows lower, more stable
worst-case latency.

---

## Further Reading

| Topic | Link |
|---|---|
| libvirt CPU tuning (`<cputune>`) | [libvirt.org — CPU tuning](https://libvirt.org/formatdomain.html#cpu-tuning) |
| `isolcpus` / nohz_full / rcu_nocbs | [kernel.org — Kernel parameters](https://docs.kernel.org/admin-guide/kernel-parameters.html) |
| `virsh vcpupin` | [libvirt.org — virsh](https://libvirt.org/manpages/virsh.html) |
| CPU affinity | [man7.org — sched_setaffinity(2)](https://man7.org/linux/man-pages/man2/sched_setaffinity.2.html) |
| Real-time tuning (KVM-RT) | [Wikipedia — Real-time computing](https://en.wikipedia.org/wiki/Real-time_computing) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why pin the *emulator* thread to different cores than the vCPU threads for a real-time guest?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A real-time guest needs its vCPU to meet tight latency deadlines. QEMU's emulator thread (main loop, device emulation, timers) and iothreads do asynchronous work; if they run on the same core as a vCPU, they preempt that vCPU exactly when a device event or interrupt fires — injecting latency at the worst possible moment. Pinning the emulator/iothreads to separate "housekeeping" cores ensures the vCPU's core is never stolen for emulation/IO, keeping the guest's worst-case latency bounded and predictable. (Isolating the vCPU cores with isolcpus/nohz_full/rcu_nocbs further keeps the host kernel off them.)
</details>

---

**Q2. What do `isolcpus`, `nohz_full`, and `rcu_nocbs` each contribute to a dedicated-core setup?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
isolcpus tells the host scheduler not to place normal tasks on the listed CPUs, so they're reserved for your pinned vCPUs. nohz_full disables the periodic scheduler tick on those CPUs when a single task is running, removing regular timer-interrupt interruptions for a CPU-bound real-time vCPU. rcu_nocbs offloads RCU callback processing off those cores so RCU housekeeping doesn't interrupt the guest. Together they turn the chosen cores into quiet, dedicated lanes where a pinned vCPU runs with minimal host-induced interruptions.
</details>

---

**Q3. What is a "noisy neighbor" in the context of hyperthread siblings, and how do you avoid it for a latency-critical guest?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Hyperthread (SMT) siblings share one physical core's execution units. A "noisy neighbor" is another VM or host task running on the sibling thread of a core that a latency-critical vCPU uses — it contends for those shared execution resources, causing unpredictable slowdowns/jitter for your guest even though it's "pinned." You avoid it by giving the latency-critical guest whole physical cores: pin it to both HT siblings of each core (and don't place anything else there), or disable the sibling thread, so no other workload shares its core's execution units.
</details>

---

## Homework

Pin a 2-vCPU guest's vCPUs to two physical cores and its emulator thread to two *different* cores via `<cputune>`. Verify with `ps -To pid,tid,comm,psr` that the vCPU threads and emulator thread are on the cores you chose. Then run a latency-sensitive workload (e.g. `cyclictest`) in the guest before and after, and report whether max latency improved. Explain the result in terms of cache warmth and preemption.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
After pinning, ps shows the "CPU N/KVM" vCPU threads on your chosen vCPU cores and the qemu-system emulator thread on the separate housekeeping cores (the PSR column). cyclictest's maximum latency typically drops and becomes more consistent after pinning+separation. The improvement comes from two effects: (1) cache warmth — a pinned vCPU stays on one core, so its L1/L2 cache and TLB aren't invalidated by migrations, reducing miss-driven latency; and (2) reduced preemption — with the emulator/iothreads on different cores and the vCPU cores isolated, nothing competes for or steals the vCPU's core, so the guest's real-time thread isn't interrupted at critical moments. The floating default suffered jitter from migrations and from overhead threads occasionally landing on the vCPU's core.
</details>
