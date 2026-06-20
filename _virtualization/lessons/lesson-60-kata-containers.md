---
title: "Lesson 60 — VM-Isolated Containers and the VM-vs-Container Question"
nav_order: 60
parent: "Phase 15: Lightweight VMs & Ecosystem"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 60: VM-Isolated Containers and the VM-vs-Container Question

## Concept

Containers (network namespaces, cgroups — recall **Networking Lesson 1**) are fast and dense
but share the **host kernel**: a kernel exploit escapes the container. VMs have a separate
kernel and strong isolation but are heavier. **Kata Containers** asks: why choose? It wraps
each container in its own **microVM** (Phase 15), giving container ergonomics with VM-grade
isolation.

```
   THE ISOLATION SPECTRUM (weaker/denser  →  stronger/heavier)
   ┌─────────────┬──────────────┬───────────────┬──────────────┬──────────────┐
   │ namespaces  │  gVisor       │   Kata        │  full VM     │ separate     │
   │ (runc)      │ (userspace    │ (container in │              │ physical     │
   │ shared      │  kernel)      │  a microVM)   │ own kernel   │ host         │
   │ kernel      │ syscalls      │ own kernel,   │ + full       │ air-gapped   │
   │             │ intercepted   │ tiny VM       │ device model │              │
   └─────────────┴──────────────┴───────────────┴──────────────┴──────────────┘
```

---

## How It Works

### Plain containers (runc) — shared kernel

A standard OCI container (run by **runc**) is just host processes isolated with namespaces
and cgroups; **all containers share the one host kernel**. That's why they're lightweight
(no second kernel to boot) and dense — but the kernel is a **shared attack surface**: a
single kernel vulnerability or container-escape bug can compromise the host and every other
container. Fine for trusted workloads; risky for untrusted/multi-tenant code.

### Kata Containers — each container in a microVM

**Kata** implements the OCI runtime interface (drop-in for Docker/Kubernetes) but, instead
of running the container as host processes, it **launches a lightweight microVM** (QEMU
microvm / Cloud Hypervisor / Firecracker) with its **own guest kernel**, and runs the
container *inside* that VM. To the orchestrator it looks like a normal container; underneath
it's a VM. It uses the building blocks you've learned: a microVM (Lesson 59), virtio-fs
(Lesson 28) to share the container rootfs, vsock (Lesson 30) for the agent, and minimal
devices.

- **Security property gained:** a real **hardware-virtualization boundary** (its own kernel
  + VT-x/AMD-V isolation). A kernel exploit inside a Kata container hits the *guest* kernel,
  not the host — to reach the host an attacker must *also* break out of the VM. That's
  dramatically stronger isolation than namespaces alone.
- **Cost:** more overhead than runc — each container carries a guest kernel and VM
  (more memory, slower start than a plain namespace, some I/O overhead), though microVM tech
  keeps this far smaller than a full VM.

### gVisor — a contrasting approach

**gVisor** takes a different route to stronger-than-runc isolation: it runs a **userspace
kernel** that intercepts the container's syscalls and services most of them itself (in
userspace), only forwarding a limited, sanitized set to the host kernel. So it shrinks the
host kernel surface *without* a VM — but pays a syscall-interception performance cost and has
compatibility limits (some syscalls/features unsupported). Kata uses a real VM boundary;
gVisor uses a userspace-kernel boundary.

### Choosing: density vs isolation vs compatibility

| Option | Isolation | Density/overhead | Compatibility |
|---|---|---|---|
| runc (namespaces) | weakest (shared kernel) | highest density, lowest overhead | full (it's just the host kernel) |
| gVisor | medium (userspace kernel) | low-ish overhead | limited (some syscalls unsupported) |
| Kata (microVM) | strong (own kernel + HW VM) | higher overhead than runc | high (real Linux kernel inside) |
| full VM | strongest (in software stack) | heaviest | full |

Pick based on the threat: trusted workloads → runc (density); untrusted/multi-tenant code →
Kata or gVisor (isolation); maximum isolation/legacy → full VM or separate host.

{: .note }
> **What Kata buys over runc, and what it costs**
> Over a plain runc container, Kata adds a real hardware-virtualization boundary: each
> container runs in its own microVM with its own guest kernel, so a kernel-level exploit or
> container escape is contained to that VM's kernel — to compromise the host the attacker must
> additionally break out of the VM (a much harder, separate barrier). That's the key security
> gain for untrusted/multi-tenant workloads. The cost is overhead: each container now carries
> a guest kernel and VM, using more memory and starting slower than a namespace-only container
> (microVM tech keeps this modest but nonzero), and there's some I/O overhead from the virtio
> boundary. You trade a bit of density/speed for VM-grade isolation.

---

## Lab

```bash
# 1. Plain runc container shares the HOST kernel — prove it:
$ docker run --rm alpine uname -r
6.8.0-111-generic          ← same as the host kernel:
$ uname -r
6.8.0-111-generic          ← identical! the container has no kernel of its own

# 2. Install Kata and run a container with the kata runtime (own guest kernel):
$ docker run --rm --runtime io.containerd.kata.v2 alpine uname -r
5.19.2                      ← DIFFERENT! a kata container has its OWN guest kernel
# (the exact version is Kata's bundled guest kernel, not the host's)

# 3. See that a Kata container is actually backed by a VM process on the host:
$ ps aux | grep -E 'qemu|cloud-hypervisor|firecracker' | grep -v grep
# A VMM process exists per Kata container — proof it's a microVM, not just namespaces.

# 4. In Kubernetes, select Kata via a RuntimeClass (drop-in isolation per pod):
$ cat <<'EOF' | kubectl apply -f -
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata: { name: kata }
handler: kata
EOF
# then a Pod with  spec.runtimeClassName: kata  runs inside a microVM.

# 5. (Contrast) gVisor uses a userspace kernel, not a VM — same uname trick:
$ docker run --rm --runtime runsc alpine dmesg 2>&1 | head -1
#   gVisor's "kernel" (Sentry) responds — syscalls are intercepted in userspace.

# 6. Compare startup/overhead: time a runc vs a kata 'true' container:
$ time docker run --rm alpine true
$ time docker run --rm --runtime io.containerd.kata.v2 alpine true
# kata is slower to start (boots a microVM) but far more isolated.
```

**Expected result:** A runc container reports the *same* kernel version as the host (shared
kernel); a Kata container reports a *different* kernel (its own, inside a microVM), with a
VMM process backing it. Kata starts slower than runc but provides a hardware-VM isolation
boundary.

---

## Further Reading

| Topic | Link |
|---|---|
| Networking Lesson 1 (namespaces) | [What a network namespace is]({{ '/networking/lessons/lesson-01-namespaces-intro.html' | relative_url }}) |
| Kata Containers | [katacontainers.io](https://katacontainers.io/) |
| gVisor | [Wikipedia — gVisor](https://en.wikipedia.org/wiki/GVisor) |
| OS-level virtualization (containers) | [Wikipedia — OS-level virtualization](https://en.wikipedia.org/wiki/OS-level_virtualization) |
| runc / OCI runtime | [opencontainers.org](https://opencontainers.org/) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Kata runs each container in its own tiny VM. What security property does that buy over a plain runc container, and what does it cost?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It buys a real hardware-virtualization isolation boundary: each container runs in its own microVM with its own guest kernel, so a kernel exploit or container-escape inside a Kata container is contained to that VM's guest kernel — to reach the host, an attacker must additionally break out of the VM (a much harder, separate barrier). A plain runc container shares the single host kernel, so one kernel/escape bug can compromise the host and all other containers. The cost is overhead: each Kata container carries a guest kernel and VM, using more memory and starting slower than a namespace-only container, plus some I/O overhead from the virtio boundary — i.e. you trade density/speed for VM-grade isolation.
</details>

---

**Q2. Why is a plain runc container's shared kernel both its efficiency advantage and its security weakness?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Efficiency: because all runc containers are just host processes isolated by namespaces/cgroups sharing the one host kernel, there's no second kernel to boot or memory to dedicate per container — so they start instantly and pack densely. Security weakness: that same shared kernel is a common attack surface — a single kernel vulnerability or container-escape bug lets a malicious container break out and compromise the host and every other container, because there's no isolation boundary below the namespace level. The shared kernel is simultaneously what makes containers cheap and what makes them weaker than VMs for untrusted workloads.
</details>

---

**Q3. How does gVisor's approach to stronger isolation differ from Kata's?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Kata uses a real virtual machine boundary: it runs the container inside a microVM with its own guest kernel and hardware (VT-x/AMD-V) isolation. gVisor instead runs a userspace kernel (the Sentry) that intercepts the container's syscalls and services most of them in userspace, forwarding only a limited, sanitized subset to the host kernel — shrinking the host kernel attack surface without booting a VM. So Kata's boundary is hardware virtualization (strong isolation, microVM overhead), while gVisor's is a userspace-kernel syscall-interception layer (no VM, but a syscall-interception performance cost and some compatibility limitations because not all syscalls/features are supported).
</details>

---

## Homework

Run `uname -r` inside (a) a runc container and (b) a Kata container, comparing both to the host. Confirm runc matches the host kernel and Kata does not. Then check `ps aux` on the host for a VMM process while the Kata container runs. Write a short paragraph placing runc, gVisor, Kata, and a full VM on the isolation-vs-density spectrum and stating which you'd choose to run untrusted multi-tenant code, and why.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
runc's uname -r matches the host kernel (shared kernel, no isolation boundary below namespaces); Kata's uname -r shows a different kernel version (its own guest kernel inside a microVM), and ps aux shows a qemu/cloud-hypervisor/firecracker VMM process backing the Kata container — proof it's a real VM. On the spectrum from densest/weakest to strongest/heaviest: runc (shared kernel, max density, weakest isolation) → gVisor (userspace kernel intercepting syscalls, medium isolation, low-ish overhead, some compatibility limits) → Kata (container in a microVM with its own kernel + hardware VM boundary, strong isolation, more overhead) → full VM (strongest in-stack isolation, heaviest). For untrusted multi-tenant code I'd choose Kata (or gVisor if its compatibility/perf trade-offs fit), because the workload can't be trusted not to attempt a kernel/container escape, and Kata's hardware-virtualization boundary means an escape is contained to a throwaway guest kernel rather than the shared host kernel — far safer than runc — while still presenting a normal container/Kubernetes interface and far less overhead than a full VM per workload.
</details>
