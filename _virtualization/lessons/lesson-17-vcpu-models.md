---
title: "Lesson 17 — vCPU Models and Feature Flags"
nav_order: 17
parent: "Phase 5: CPU & Memory Configuration"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 17: vCPU Models and Feature Flags

## Concept

When KVM runs guest code natively, the guest *physically executes on your real
CPU*. So what CPU does the guest **think** it has? That's controlled by `-cpu`, and
it's a genuine trade-off between **performance** (expose everything the real CPU
can do) and **portability** (present a CPU that any host in your cluster can also
provide, so you can migrate).

```
   -cpu host           ── "you ARE this physical CPU, warts and all"
                          MAX performance, ZERO migration portability

   -cpu host-model     ── "the closest NAMED model + safe extra flags"
                          near-max perf, libvirt picks a portable baseline

   -cpu Haswell-noTSX   ── a specific NAMED model
                          predictable, portable to any host >= that model
```

The deeper a feature flag (AES-NI, AVX-512, …) the guest sees, the faster certain
workloads run — but only if *every* host the VM might land on also has it.

---

## How It Works

### The three strategies

- **`host-passthrough`** (`-cpu host`): expose the host CPU's *entire* feature set
  to the guest. The guest sees essentially the real CPU. **Best performance** —
  crypto, vector, and other instruction-set extensions are all available. **Worst
  portability**: the guest now depends on those exact features, so it can only
  migrate to an identical (or feature-superset) CPU.
- **`host-model`**: libvirt picks the *closest matching named model* the host
  supports and adds extra flags the host has, *resolving it to a concrete model at
  start time*. Near-passthrough performance with a more portable, describable CPU.
- **Named models** (e.g. `Haswell-noTSX`, `EPYC`, `Skylake-Server`): present a
  *specific, well-defined* CPU. The guest sees exactly that model's features on
  every host, so it migrates cleanly to any host that can provide at least that
  model. Predictable and the safest for heterogeneous clusters.

### CPU feature flags

Beyond the base model you can add/remove individual flags:

```
   -cpu Skylake-Server,+aes,-svm,+pcid
   #                    ^add AES-NI  ^drop nested-virt  ^add PCID
```

Common reasons: enable `aes` for crypto throughput, expose `vmx`/`svm` for nested
virtualization (Lesson 55), or *disable* a flag to keep a guest portable to hosts
that lack it.

### CPUID, microarchitecture levels, and mitigations

The guest learns its features via the **CPUID** instruction (which causes a VM exit
KVM filters — Lesson 10). `-cpu` ultimately programs what CPUID reports. Two modern
wrinkles:

- **Microarchitecture levels** (x86-64-v2/v3/v4): some distros now require a
  baseline (e.g. v3 needs AVX2). Your `-cpu` choice must satisfy the guest OS's
  required level.
- **Speculative-execution mitigations**: flags like `spec-ctrl`, `ssbd`,
  `md-clear` expose the CPU's Spectre/Meltdown mitigation capabilities to the guest
  so the guest OS can protect itself. Named models in current QEMU include the
  appropriate ones; passthrough inherits whatever the host has.

### The migration tension (the headline)

`host-passthrough` is the fastest but **the wrong choice for a migration cluster**:
a guest that booted seeing AVX-512 cannot be live-migrated to a host without
AVX-512 — the instruction would simply be missing mid-execution, and migration is
refused (or the guest crashes if forced). For clusters you choose a **named
baseline model** that every host supports — the "lowest common denominator" — so
any VM can run on any node. You trade a little peak performance for the freedom to
move VMs anywhere.

{: .note }
> **The baseline-CPU problem (revisited in Lesson 49)**
> A migration pool of mixed CPU generations must agree on a CPU model the *oldest*
> member can provide. Tools like <code>virsh cpu-baseline</code> compute a model that's
> a safe common denominator across hosts. Pin VMs to that, and any VM migrates to
> any host. Use host-passthrough only for VMs that never migrate (or pools of
> identical hardware).

---

## Lab

```bash
# 1. See what named CPU models THIS QEMU/host can offer:
$ qemu-system-x86_64 -cpu help | head -30
Available CPUs:
  ...
  x86 Haswell-noTSX     ...
  x86 Skylake-Server    ...
  x86 EPYC              ...
  x86 host              (only with KVM)
  ...

# 2. Boot with full passthrough (max perf, no portability):
$ qemu-system-x86_64 -accel kvm -cpu host -m 2G -smp 2 \
    -drive file=disk.qcow2,if=virtio -nographic &

# 3. Inside the guest, see the CPU it believes it has:
#    (guest)$ lscpu | grep 'Model name'
#    (guest)$ grep -o aes /proc/cpuinfo | head -1   # AES-NI exposed?

# 4. Boot with a NAMED model + explicit flags (portable + AES):
$ qemu-system-x86_64 -accel kvm -cpu Skylake-Server,+aes -m 2G -smp 2 \
    -drive file=disk.qcow2,if=virtio -nographic &

# 5. With libvirt later, compute a safe cluster baseline across hosts:
$ virsh cpu-baseline host-cpu-a.xml host-cpu-b.xml 2>/dev/null
# (prints a <cpu> model that both hosts can provide)

# 6. Check which CPUID-exposed flags KVM can pass through on this host:
$ qemu-system-x86_64 -accel kvm -cpu host -nographic -S \
    -qmp stdio <<<'{"execute":"qmp_capabilities"}' 2>/dev/null | head -1
$ kill %1 %2 2>/dev/null
```

**Expected result:** `-cpu help` lists named models; a passthrough guest reports
the real CPU model and all its flags (e.g. `aes`), while a named-model guest reports
exactly that model's features — predictable across hosts.

---

## Further Reading

| Topic | Link |
|---|---|
| QEMU CPU models (x86) | [qemu.org — CPU models](https://www.qemu.org/docs/master/system/i386/cpu.html) |
| libvirt CPU model/topology | [libvirt.org — CPU model and topology](https://libvirt.org/formatdomain.html#cpu-model-and-topology) |
| CPUID | [Wikipedia — CPUID](https://en.wikipedia.org/wiki/CPUID) |
| x86-64 microarchitecture levels | [Wikipedia — x86-64 (levels)](https://en.wikipedia.org/wiki/X86-64#Microarchitecture_levels) |
| Transient execution / Spectre mitigations | [Wikipedia — Spectre (security vulnerability)](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why might `host-passthrough` give the best performance but be the wrong choice for a migration cluster?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
host-passthrough exposes the host CPU's entire feature set to the guest, so all instruction-set extensions (AES-NI, AVX-512, etc.) are available and workloads run at full native speed. But the guest then depends on those exact features. To live-migrate, the destination CPU must provide everything the guest was booted with; if a target host lacks, say, AVX-512, that instruction would vanish mid-execution, so migration is refused (or the guest crashes if forced). In a heterogeneous cluster you instead pin a named baseline model every host supports, trading a little peak performance for the ability to move VMs anywhere.
</details>

---

**Q2. What does `host-model` do differently from `host-passthrough`?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
host-passthrough literally hands the guest the host's full CPU feature set ("you are this physical CPU"). host-model instead resolves, at VM start time, to the closest matching *named* CPU model the host supports, plus extra safe flags the host has. The result is a concrete, describable CPU definition with near-passthrough performance but more portability — because it's expressed as a known model that other hosts can be checked against, rather than an opaque copy of one specific physical CPU.
</details>

---

**Q3. The guest learns its CPU features via the CPUID instruction. How does `-cpu` relate to CPUID, and what happens at the hardware level when the guest runs CPUID?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`-cpu` ultimately programs what the guest sees when it executes CPUID — the model and feature flags reported. When the guest runs CPUID, it causes a VM exit; KVM intercepts it and returns the filtered feature set defined by the chosen CPU model (masking out features you didn't expose, adding mitigation flags as configured). So CPUID is the guest's discovery mechanism and -cpu is how you control its answers, letting you present anything from the full real CPU (host) to a constrained named baseline.
</details>

---

## Homework

On your host, run `qemu-system-x86_64 -cpu help` and pick a named model older than your physical CPU (e.g. `Nehalem` or `Haswell-noTSX`). Boot a guest with `-cpu host` and again with that older named model, and in each guest compare whether `aes` and `avx2` appear in `/proc/cpuinfo`. Explain why a cluster of mixed-generation hosts would standardize on the older model.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With <code>-cpu host</code> the guest sees the full modern feature set (likely both aes and avx2). With an older named model like Nehalem the guest sees only that model's features — aes may be absent and avx2 almost certainly is. A mixed-generation cluster standardizes on the older model because it's the lowest common denominator every host can provide: a VM booted with only those features can be live-migrated to any node, old or new, without an instruction suddenly being unavailable. Newer hosts simply don't expose their extra features to that VM, sacrificing some peak performance for universal mobility.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 18 — CPU Topology →](lesson-18-cpu-topology){: .btn .btn-primary }
