---
title: "Lesson 54 — tuned and Host Profiles"
nav_order: 54
parent: "Phase 13: Performance Tuning"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 54: tuned and Host Profiles

## Concept

Lessons 50–53 tuned things by hand. **`tuned`** is a daemon that applies a bundle of sane
performance settings — a **profile** — in one command, so you don't hand-set a dozen
knobs per machine. There are paired profiles for the two roles: **`virtual-host`** (for the
hypervisor) and **`virtual-guest`** (inside a VM).

```
   tuned profile = a curated bundle of knobs:
   ┌─────────────────────────────────────────────┐
   │ CPU governor        (performance vs powersave)│
   │ transparent hugepages                         │
   │ VM dirty ratios     (writeback behavior)      │
   │ scheduler / sched_* knobs                     │
   │ disk readahead, I/O scheduler                 │
   │ network buffer tunables                        │
   └─────────────────────────────────────────────┘
   pick a role → tuned sets all of them appropriately
```

It's the "good defaults, fast" layer: apply a profile, then override only what your
specific workload needs.

---

## How It Works

### What tuned does

`tuned` ships a set of named profiles, each a collection of system settings tuned for a
purpose (throughput, latency, power saving, virtualization roles). Applying a profile sets
the **CPU frequency governor**, **transparent hugepage** policy, **VM dirty ratios** (how
aggressively the kernel writes back dirty page-cache pages), **scheduler** knobs, **I/O
scheduler / readahead**, and **network** sysctls — all at once, consistently.

### The two virtualization profiles

- **`virtual-host`** — for the **hypervisor**. It favors throughput and predictable VM
  performance: a performance-oriented CPU governor (don't downclock under VM load), dirty
  ratios tuned so heavy guest I/O doesn't cause writeback storms, and settings that suit
  running many guest processes. It assumes *this machine runs VMs*.
- **`virtual-guest`** — for **inside a VM**. It accounts for the guest being virtualized:
  e.g. adjusted dirty ratios and readahead (the host also caches), a swappiness suited to a
  guest, and disabling host-only tunables that don't make sense in a guest. It assumes
  *this machine IS a VM*.

They're **different because host and guest sit at different layers**: the host manages
physical hardware and many guests (so it tunes governors, NUMA, big-picture I/O); the guest
runs atop virtual hardware and shouldn't fight the host's caching/scheduling (so it tunes
for cooperating with the layer beneath it). Pair them: `virtual-host` on the hypervisor,
`virtual-guest` in each VM.

### Applying and inspecting

```
   tuned-adm list                 # available profiles
   tuned-adm active               # current profile
   tuned-adm profile virtual-host # apply (host)
   tuned-adm recommend            # what tuned suggests for this machine
   tuned-adm verify               # confirm settings match the profile
```

### When to override the profile

A profile is a *starting point*, not gospel. Override when your workload has specific needs
the generic profile doesn't capture:

- A **latency-critical** guest may need the manual pinning/isolation from Lesson 50 on top
  of (or instead of) the throughput-oriented `virtual-host` defaults.
- A host with **hugepage-backed** guests (Lesson 51) may want THP disabled (explicit
  hugepages instead), which you set regardless of the profile.
- You can create a **custom child profile** that inherits a stock one and overrides only
  specific knobs, keeping the good defaults while changing what matters.

{: .note }
> **What category of settings virtual-host adjusts, and why host/guest differ**
> <code>virtual-host</code> adjusts system-wide performance knobs aimed at running VMs well:
> the CPU frequency governor (keep clocks high under load), VM dirty-page writeback ratios
> (so heavy guest I/O doesn't trigger writeback stalls), scheduler and NUMA-related settings,
> and I/O/network tunables. Host and guest profiles differ because they operate at different
> layers: the host owns the physical hardware and orchestrates many guests, so it tunes
> governors, memory writeback, and big-picture I/O; the guest runs on virtual hardware atop
> that host and should <em>cooperate</em> with it (avoid double-caching/readahead conflicts,
> use guest-appropriate swappiness) rather than try to manage hardware it doesn't really own.

---

## Lab

```bash
# (Install: sudo apt install tuned   /   sudo dnf install tuned)

# 1. See available profiles and the current one:
$ tuned-adm list | grep -E 'virtual|throughput|latency'
- virtual-host       - Optimize for running KVM guests
- virtual-guest      - Optimize for running inside a virtual machine
- throughput-performance
- latency-performance
$ tuned-adm active
Current active profile: balanced

# 2. On the HYPERVISOR, apply the host profile:
$ sudo tuned-adm profile virtual-host
$ tuned-adm active
Current active profile: virtual-host

# 3. See what it changed — e.g. CPU governor and dirty ratios:
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
performance                       ← governor bumped up
$ sysctl vm.dirty_ratio vm.dirty_background_ratio
vm.dirty_ratio = 40
vm.dirty_background_ratio = 10

# 4. INSIDE a guest, apply the guest profile:
#   (guest)$ sudo tuned-adm profile virtual-guest
#   (guest)$ tuned-adm active
#   Current active profile: virtual-guest

# 5. Let tuned recommend and verify:
$ tuned-adm recommend
virtual-host
$ tuned-adm verify
Verification succeeded, current system settings match the preset profile.

# 6. Create a custom child profile that inherits virtual-host but disables THP
#    (for a host using explicit hugepages, Lesson 51):
$ sudo mkdir -p /etc/tuned/myhost
$ sudo tee /etc/tuned/myhost/tuned.conf >/dev/null <<'EOF'
[main]
include=virtual-host
[vm]
transparent_hugepages=never
EOF
$ sudo tuned-adm profile myhost
```

**Expected result:** `tuned-adm profile virtual-host` applies a bundle of host-appropriate
settings in one step (visible as a performance governor, adjusted dirty ratios, etc.), and
`virtual-guest` does the guest-appropriate equivalent. A custom child profile lets you keep
those defaults while overriding a single knob (e.g. THP).

---

## Further Reading

| Topic | Link |
|---|---|
| tuned | [tuned-project.org](https://tuned-project.org/) |
| tuned profiles | [tuned — Profiles](https://github.com/redhat-performance/tuned) |
| `tuned-adm` | [man7.org — tuned-adm(8)](https://man7.org/linux/man-pages/man8/tuned-adm.8.html) |
| CPU frequency scaling / governors | [kernel.org — CPUFreq governors](https://docs.kernel.org/admin-guide/pm/cpufreq.html) |
| Dirty page writeback | [kernel.org — sysctl/vm](https://docs.kernel.org/admin-guide/sysctl/vm.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What category of settings does the `virtual-host` tuned profile adjust, and why are host and guest profiles different?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
virtual-host adjusts system-wide performance knobs for running VMs: the CPU frequency governor (keep clocks high under load), VM dirty-page writeback ratios (so heavy guest I/O doesn't cause writeback stalls), scheduler/NUMA-related settings, and I/O and network tunables — all at once. Host and guest profiles differ because they sit at different layers: the host owns the physical hardware and orchestrates many guests, so it tunes governors, memory writeback, and big-picture I/O; the guest runs on virtual hardware atop the host and should cooperate with it (avoid double-caching/readahead conflicts, use guest-appropriate swappiness) rather than manage hardware it doesn't truly control. Hence virtual-host on the hypervisor and virtual-guest inside the VM.
</details>

---

**Q2. What is the advantage of using a tuned profile instead of hand-setting individual knobs?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A tuned profile applies a curated, consistent bundle of many settings (governor, THP, dirty ratios, scheduler, I/O scheduler/readahead, network sysctls) in one command, so you get sensible role-appropriate defaults without manually finding and setting a dozen knobs per machine — and consistently across many machines. It's also reversible/manageable (switch profiles, verify with tuned-adm verify) and reduces the chance of forgetting or mis-setting a knob. You then override only what your specific workload needs, rather than building the whole configuration by hand.
</details>

---

**Q3. When would you override or extend a stock profile, and how can you do that cleanly?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Override when your workload has needs the generic profile doesn't cover — e.g. a latency-critical guest needing manual CPU pinning/isolation (Lesson 50) beyond throughput defaults, or a host using explicit hugepages where you want THP disabled (Lesson 51) regardless of the profile. The clean way is to create a custom child profile that <code>include=</code>s the stock profile and overrides only the specific knobs you care about (e.g. transparent_hugepages=never), then activate it. This keeps all the good inherited defaults while changing just what matters, rather than abandoning the profile or editing it in place.
</details>

---

## Homework

On a host, run `tuned-adm active` and `tuned-adm recommend`, then apply `virtual-host` and observe what changed (CPU governor, `vm.dirty_ratio`). Inside a guest, apply `virtual-guest`. Then create a child profile that inherits `virtual-host` but overrides one knob relevant to your setup (e.g. disable THP if you use explicit hugepages). Explain why your override is appropriate for your specific workload.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Before applying, tuned-adm active likely shows "balanced"; tuned-adm recommend suggests virtual-host on a hypervisor. After <code>tuned-adm profile virtual-host</code>, the CPU governor switches to performance and vm.dirty_ratio/background ratios change to VM-friendly values; inside the guest, virtual-guest sets guest-appropriate defaults. A reasonable override: create /etc/tuned/myhost inheriting virtual-host with transparent_hugepages=never. This is appropriate if your guests are backed by *explicit* hugepages (Lesson 51): you don't want the kernel spending effort on transparent hugepage merging/khugepaged for memory that's already reserved as explicit hugepages — THP would add jitter and wouldn't apply to hugetlbfs-backed guest memory anyway. The child profile keeps all of virtual-host's good throughput defaults while disabling just the one knob that conflicts with your explicit-hugepage strategy.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 55 — Nested Virtualization →](lesson-55-nested-virt){: .btn .btn-primary }
