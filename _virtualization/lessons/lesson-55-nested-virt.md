---
title: "Lesson 55 — Nested Virtualization"
nav_order: 55
parent: "Phase 14: Nested Virt & Security"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 55: Nested Virtualization

## Concept

**Nested virtualization** is running a hypervisor *inside* a VM — KVM inside KVM. It's how
you run VMs on a cloud instance, test hypervisors, build CI runners that spin up VMs, and
(very likely) how *this whole curriculum* runs if your "host" is itself a VM.

```
   L0  ── the physical host (real hardware, real VT-x/AMD-V)
    │  runs KVM
   L1  ── a guest that ITSELF runs KVM  (needs vmx/svm EXPOSED to it)
    │  runs KVM
   L2  ── a guest running inside L1   ("nested" guest)
```

The trick: L1 normally *doesn't* see the CPU's virtualization extensions (vmx/svm), so it
can't be a hypervisor. Nested virtualization makes L0's KVM **expose vmx/svm to L1**, so L1
can use them to run L2.

---

## How It Works

### Enabling nesting on L0

Two parts:

1. **Turn on the module parameter** on the physical host (Lesson 9):

   ```
   # Intel:  options kvm_intel nested=1     (/sys/module/kvm_intel/parameters/nested → Y)
   # AMD:    options kvm_amd   nested=1
   ```

2. **Expose vmx/svm to the L1 guest's virtual CPU.** The L1 VM must be given a CPU model
   that includes the virtualization feature — easiest with `-cpu host` (passthrough,
   Lesson 17), or by explicitly adding the feature (`-cpu ...,+vmx` / `+svm`). In libvirt:
   `<cpu mode='host-passthrough'/>`.

Then inside L1, `grep vmx /proc/cpuinfo` succeeds, `/dev/kvm` is usable, and L1 can run L2.

### How it works under the hood (briefly)

The CPU only has *one* level of hardware virtualization (one VMX root/non-root, one set of
VMCS). When L1 (itself a guest) tries to run its own guest L2 using VMX, L0's KVM must
**emulate the VMX instructions** L1 issues and **shadow** L1's VMCS into a real VMCS the
hardware understands — merging two levels of control into the one the CPU provides.
Similarly, nested **EPT** shadows two levels of address translation. The hardware runs L2
directly when possible, but L0 mediates all the virtualization *control* operations L1
performs.

### Why it's slower

Because L0 must emulate/shadow L1's virtualization operations, every VM exit that L2 causes
may bounce through *two* hypervisors (L2→L1, and L1's handling may trap to L0), and VMCS/EPT
shadowing adds overhead. CPU-bound L2 work that avoids exits runs near-native (the hardware
executes L2 directly), but exit-heavy or virtualization-control-heavy workloads pay a
compounding tax. Nesting is great for *functionality* (testing, CI, training) and fine for
light workloads, but it's not how you'd run production performance-critical nested guests.

### Use cases and limitations

- **Use cases:** developing/testing hypervisors, CI that needs to create VMs, training
  labs (like this curriculum), running a different hypervisor inside a cloud VM.
- **Limitations:** slower (above); migration of an L1 that's actively running L2 guests is
  fragile/often unsupported; some advanced features may not be available nested; AMD and
  Intel nesting maturity has varied over time.

{: .note }
> **Why your "host" might already be L1**
> If you're doing this curriculum on a cloud VM or a VM on your laptop, your "host" is
> itself a guest (L1). For <code>/dev/kvm</code> to work there and let you run lab VMs (L2),
> the outer hypervisor (L0) must have nested virtualization enabled AND expose vmx/svm to
> your VM. If <code>grep -E 'vmx|svm' /proc/cpuinfo</code> is empty inside your VM, nesting
> isn't enabled on L0 — your labs will fall back to slow TCG. The fix lives on L0 (the
> physical host or cloud provider's setting), not inside your VM.

---

## Lab

```bash
# ===== On L0 (the physical host) =====
# 1. Enable nesting on the vendor module (Lesson 9). Intel shown; AMD = kvm_amd:
$ cat /sys/module/kvm_intel/parameters/nested
N
$ sudo modprobe -r kvm_intel && sudo modprobe kvm_intel nested=1   # (no VMs running)
$ cat /sys/module/kvm_intel/parameters/nested
Y
# Persist:
$ echo "options kvm_intel nested=1" | sudo tee /etc/modprobe.d/kvm.conf

# 2. Boot the L1 guest WITH vmx/svm exposed (host-passthrough is simplest):
$ qemu-system-x86_64 -accel kvm -cpu host -m 8G -smp 4 \
    -drive file=L1.qcow2,if=virtio -nographic &

# ===== Inside L1 (the nested hypervisor) =====
# 3. Confirm the virtualization extension is now visible:
#   (L1)$ grep -oE 'vmx|svm' /proc/cpuinfo | sort -u
#   vmx                            ← exposed! L1 can be a hypervisor
#   (L1)$ ls -l /dev/kvm           ← usable
#   (L1)$ kvm-ok
#   KVM acceleration can be used

# 4. Run an L2 guest INSIDE L1, accelerated by L1's KVM:
#   (L1)$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
#           -drive file=L2.qcow2,if=virtio -nographic
#   (L2 boots; you are now three levels deep: L0 > L1 > L2)

# 5. Verify L2 really used KVM (not TCG) — check from L1:
#   (L1)$ sudo kvm_stat -1 | head    # exit counters confirm hardware accel in use
$ kill %1 2>/dev/null
```

**Expected result:** After enabling `nested=1` on L0 and giving L1 a `-cpu host` CPU, the
`vmx`/`svm` flag appears inside L1 and `/dev/kvm` works there — so L1 can run an L2 guest
with KVM acceleration. Without nesting, L1 would lack the flag and L2 would fall back to
TCG.

---

## Further Reading

| Topic | Link |
|---|---|
| KVM nested virtualization | [kernel.org — Nested VMX](https://docs.kernel.org/virt/kvm/x86/nested-vmx.html) |
| Nested virtualization (concept) | [Wikipedia — Nested virtualization](https://en.wikipedia.org/wiki/Virtual_machine#Nested_virtualization) |
| KVM module params (Lesson 9) | [KVM kernel modules and parameters]({{ '/virtualization/lessons/lesson-09-kvm-modules-params.html' | relative_url }}) |
| CPU models (Lesson 17) | [vCPU models]({{ '/virtualization/lessons/lesson-17-vcpu-models.html' | relative_url }}) |
| Host readiness (Lesson 7) | [Checking and enabling virtualization]({{ '/virtualization/lessons/lesson-07-host-readiness.html' | relative_url }}) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What host setting must change to let a guest itself run KVM, and why is nested virtualization inherently slower than a single level?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
On the physical host (L0) you must enable the nested module parameter (kvm_intel nested=1 / kvm_amd nested=1) AND expose the virtualization feature (vmx/svm) to the L1 guest's CPU (e.g. -cpu host). Then L1 sees vmx/svm and can use /dev/kvm. It's inherently slower because the CPU provides only one hardware virtualization level: L0's KVM must emulate the VMX/SVM instructions L1 issues and shadow L1's VMCS/EPT into the single real structures the hardware understands. So L2's virtualization-control operations and exits can bounce through two hypervisors with extra shadowing overhead. Pure CPU-bound L2 code that avoids exits still runs near-native, but exit-heavy/control-heavy work pays a compounding tax.
</details>

---

**Q2. Inside an L1 guest, `grep vmx /proc/cpuinfo` returns nothing. What does that tell you, and where is the fix?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It tells you the virtualization extension isn't exposed to L1, so L1 can't act as a hypervisor — /dev/kvm won't work and any L2 guest will fall back to slow TCG emulation. The fix is on L0 (the outer host/cloud provider), not inside L1: L0 must have nested virtualization enabled (kvm_intel/kvm_amd nested=1) and must give the L1 VM a CPU that exposes vmx/svm (e.g. -cpu host or +vmx/+svm, libvirt host-passthrough). You can't enable it from within L1; it's the outer hypervisor's configuration.
</details>

---

**Q3. Give two legitimate use cases for nested virtualization.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Examples: (1) developing/testing hypervisors or virtualization software without dedicated bare-metal hardware; (2) CI/CD runners that need to create and run VMs as part of their jobs; (3) training/lab environments (like this curriculum) where students run VMs inside a provided VM; (4) running a different hypervisor or a VM-based sandbox inside a cloud instance. Any two of these are valid — all rely on giving a guest the ability to itself run hardware-accelerated VMs.
</details>

---

## Homework

On your host (L0), check `/sys/module/kvm_intel/parameters/nested` (or kvm_amd). If it's N, enable it and persist it. Boot an L1 VM with `-cpu host`, and inside it confirm `vmx`/`svm` appears and `kvm-ok` passes. Then boot a tiny L2 guest inside L1. Finally, explain how you'd tell whether L2 is using L1's KVM acceleration versus falling back to TCG, and why nesting matters for running this curriculum's labs.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
After enabling nested=1 on L0 and giving L1 -cpu host, inside L1 <code>grep -oE 'vmx|svm' /proc/cpuinfo</code> returns the flag and <code>kvm-ok</code> reports KVM can be used; an L2 guest then boots with -accel kvm. To tell whether L2 uses KVM vs TCG: check that L2 was launched with -accel kvm (and didn't error/fall back), confirm /dev/kvm was usable in L1, and run <code>kvm_stat</code> in L1 while L2 runs — nonzero exit counters mean hardware-accelerated KVM is active for L2 (TCG would show no KVM exits and L2 would be conspicuously slow to boot). Nesting matters for this curriculum because if your "host" is itself a VM, you need L0 nesting + exposed vmx/svm for /dev/kvm to work, so your lab VMs run with KVM acceleration instead of crawling under TCG.
</details>
