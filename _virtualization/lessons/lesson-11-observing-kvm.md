---
title: "Lesson 11 — Observing KVM at Runtime"
nav_order: 11
parent: "Phase 3: KVM Internals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 11: Observing KVM at Runtime

## Concept

VM exits are invisible — they happen millions of times with no log line. This
lesson gives you instruments to **make the invisible visible**, the same way the
networking track uses `tcpdump` to see packets. If you can't measure exits, you
can't tune them.

```
   ┌──────────────────────────────────────────────────────────┐
   │  Instruments, coarse → fine                               │
   │                                                          │
   │  kvm_stat        live exit counters by reason  (start here) │
   │  perf kvm stat   richer per-exit timing/aggregation        │
   │  /sys/kernel/debug/kvm/   raw debugfs counters             │
   │  ftrace / trace-cmd  kvm:* tracepoints — every single exit │
   └──────────────────────────────────────────────────────────┘
```

The workflow: start broad (`kvm_stat` to see *which* exit reason dominates), then
zoom in (tracepoints to see *exactly* what's causing it).

---

## How It Works

### `kvm_stat` — the first tool to reach for

`kvm_stat` reads KVM's debugfs counters and shows exits **by reason**, updating
live. It's the workhorse. `kvm_stat -1` prints a single snapshot (scriptable);
`kvm_stat -l` logs over time. The columns: total count, percentage, and current
rate per second. A spike in `mmio`/`pio`/`userspace_exit` means device I/O; a high
`halt`/`halt_poll` means an idle guest.

### `perf kvm stat` — timing, not just counting

`perf kvm stat record` / `report` aggregates exits with **timing** — it tells you
not just how many of each exit reason but how much *time* was spent in each. This
surfaces the difference between a frequent-but-cheap exit and a rare-but-expensive
one. `perf kvm top` gives a live `top`-like view of guest activity.

### `/sys/kernel/debug/kvm/` — the raw source

`kvm_stat` is really just a friendly reader of these debugfs files. You can `cat`
them directly: each VM appears as a subdirectory, with files like
`exits`, `mmio_exits`, `halt_exits`, `host_state_reload`, etc. Useful when you want
one specific counter without the whole UI, or to script against.

### `ftrace` / `trace-cmd` — every single exit

For the deepest view, KVM exposes **tracepoints** (`kvm:kvm_exit`,
`kvm:kvm_entry`, `kvm:kvm_mmio`, `kvm:kvm_pio`, …). With `trace-cmd record -e kvm`
you capture *every* exit with its reason code and guest RIP — invaluable for
diagnosing a pathological guest where one specific address keeps trapping.

{: .note }
> **Reading exit-reason histograms**
> The shape of the histogram tells the story. Dominated by HLT → idle guest, fine.
> Dominated by EPT violations that never settle → the guest keeps touching new
> memory or memory is being reclaimed under it (overcommit/KSM pressure, Phase 5).
> Dominated by MMIO/PIO/userspace exits → emulated devices that should be virtio
> (Lesson 10). High `host_state_reload` / external interrupts → noisy neighbor or
> too many timer interrupts. You read it like a doctor reads a chart.

---

## Lab

```bash
# Start a guest to observe (use virtio so you can contrast with emulated later):
$ qemu-system-x86_64 -accel kvm -m 1G -smp 2 -nographic \
    -drive file=guest.qcow2,if=virtio &

# 1. kvm_stat live — watch the reason breakdown (q to quit):
$ sudo kvm_stat
 Event              Total   %Total   CurAvg/s
 kvm_exit           20431    100.0       640
 kvm_entry          20410     ...        639
 kvm_halt            8800     ...        210     ← guest idling a lot
 kvm_mmio            210      ...          2
 kvm_userspace_exit 540       ...          6

# 2. Single snapshot for scripting:
$ sudo kvm_stat -1 | sort -k2 -nr | head
kvm_exit            20431
kvm_entry           20410
kvm_halt             8800
...

# 3. Raw debugfs (what kvm_stat reads):
$ sudo mount -t debugfs none /sys/kernel/debug 2>/dev/null
$ sudo cat /sys/kernel/debug/kvm/*/exits 2>/dev/null | head

# 4. perf timing view (sudo apt install linux-tools-$(uname -r)):
$ sudo perf kvm stat record -a -- sleep 5
$ sudo perf kvm stat report
Analyze events for all VMs...
VM-EXIT    Samples  Samples%   Time%   ...
HLT        8800     43.1%      90.2%      ← lots of idle time
MSR_WRITE  ...
EPT_VIOL   ...

# 5. Trace EVERY exit for 2 seconds with reason + guest RIP:
$ sudo trace-cmd record -e kvm:kvm_exit -- sleep 2
$ sudo trace-cmd report | head
   CPU 0: ... kvm_exit: reason MMIO rip 0xffffffff81... 
```

**Expected result:** `kvm_stat` and `perf kvm stat` both show exits dominated by
HLT for an idle guest. When you add load, MMIO/PIO/userspace counters rise. The
tracepoint capture shows individual exits with their guest instruction pointer.

---

## Further Reading

| Topic | Link |
|---|---|
| Tuning KVM / `kvm_stat` | [linux-kvm.org — Tuning KVM](https://www.linux-kvm.org/page/Tuning_KVM) |
| `perf` wiki | [perf.wiki.kernel.org](https://perf.wiki.kernel.org/index.php/Main_Page) |
| ftrace | [kernel.org — ftrace](https://docs.kernel.org/trace/ftrace.html) |
| `trace-cmd` | [man7.org — trace-cmd(1)](https://man7.org/linux/man-pages/man1/trace-cmd.1.html) |
| KVM tracepoints | [kernel.org — KVM](https://docs.kernel.org/virt/kvm/index.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. You see a huge count of MMIO exits in `kvm_stat`. What does that suggest about the guest's device configuration, and what would you change?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A flood of MMIO exits means the guest is constantly accessing memory-mapped registers of an emulated hardware device — usually a fully emulated NIC or disk controller — and each access traps to QEMU. The change is to switch the guest to paravirtualized virtio devices (virtio-net, virtio-blk/scsi), and enable vhost where applicable, so the datapath uses shared-memory rings handled in the kernel instead of per-register MMIO exits. That collapses tens of thousands of expensive exits into far fewer cheap notifications.
</details>

---

**Q2. What does `perf kvm stat` give you that `kvm_stat` does not?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`perf kvm stat` adds timing/aggregation: besides counting exits by reason it reports how much *time* was spent handling each exit reason. That distinguishes a frequent-but-cheap exit from a rare-but-expensive one — e.g. HLT might be 43% of samples but 90% of time. kvm_stat only shows counts and rates, so it can't tell you where the time actually goes.
</details>

---

**Q3. If you need to find the exact guest instruction address that keeps causing a specific exit, which tool do you use and why?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Use ftrace / trace-cmd on the KVM tracepoints (e.g. kvm:kvm_exit). Unlike the aggregate counters of kvm_stat/perf, tracepoints record every individual exit with its reason code and the guest RIP (instruction pointer), so you can pinpoint the exact address (and thus the code path / device) responsible for a pathological stream of exits.
</details>

---

## Homework

Start a guest with an *emulated* NIC (`-netdev user,id=n0 -device e1000,netdev=n0`) and run a `ping -f` flood inside it while watching `kvm_stat`. Then recreate the guest with `-device virtio-net-pci,netdev=n0` and repeat. Record the approximate `kvm_mmio`/`kvm_pio`/`userspace_exit` rates for each and explain the difference using Lesson 10's concepts.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The e1000 (emulated) NIC produces a high rate of MMIO/PIO and userspace exits during the ping flood — every packet involves register accesses that trap to QEMU's device model. The virtio-net NIC produces far fewer such exits because packets move through a shared virtqueue ring with batched notifications instead of per-register pokes (and with vhost the datapath stays in the kernel entirely). The difference is exit reduction: virtio replaces many expensive userspace round trips per packet with a small number of cheap notifications, which is why throughput and CPU efficiency improve dramatically.
</details>
