---
title: "Lesson 61 — Management Platforms and Where to Go Next"
nav_order: 61
parent: "Phase 15: Lightweight VMs & Ecosystem"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 61: Management Platforms and Where to Go Next

## Concept

You've built the whole stack by hand: KVM, QEMU, storage, networking, libvirt, migration,
tuning, security. **Management platforms** automate all of it at scale across many hosts —
clustering, scheduling, high availability, multi-tenant storage/network orchestration. This
capstone places those platforms on the map and shows that each one is "just" automation of
things you now understand.

```
   YOU (by hand)              PLATFORM (automates)
   ─────────────             ─────────────────────
   virsh on one host    ──►  Proxmox / oVirt: cluster of hosts, GUI, HA
   qemu-img + pools     ──►  orchestrated storage (Ceph/NFS) across nodes
   migrate between 2    ──►  live-migrate + auto-rebalance across a cluster
   bridges + libvirt    ──►  software-defined networking across nodes
   one KVM VM           ──►  OpenStack Nova / KubeVirt: VMs as cloud/K8s workloads
```

---

## How It Works

### libvirt-based platforms: Proxmox VE and oVirt/RHV

These build a **cluster manager + web UI** on top of the exact libvirt/QEMU/KVM stack you
learned:

- **Proxmox VE** — a Debian-based, libvirt-adjacent (it uses QEMU/KVM directly) hypervisor
  distro with a web GUI, clustering, built-in Ceph, backups, and HA. Popular for homelabs and
  SMBs. Everything it does maps to virsh/qemu-img/migrate under the hood.
- **oVirt** (upstream of Red Hat Virtualization) — a libvirt-based datacenter virtualization
  manager: a central engine managing many KVM hosts, with storage domains, logical networks,
  live migration, and HA — enterprise VMware-vSphere-style management of KVM.

### OpenStack Nova and KubeVirt

- **OpenStack Nova** — the compute service of OpenStack, a full IaaS cloud. Nova schedules VMs
  across a fleet of KVM hosts (via libvirt), integrating with Neutron (networking), Cinder
  (block storage), Glance (images). It's how you'd build an AWS-like private cloud on KVM.
- **KubeVirt** — runs **VMs as Kubernetes workloads**. A VM becomes a Kubernetes custom
  resource scheduled onto nodes (each VM is a QEMU/KVM process in a pod), so you manage VMs
  with `kubectl` alongside containers — and they plug into Kubernetes networking (ties back to
  **Networking Lessons 43–44**, K8s/CNI) and storage. It's the convergence of the VM and
  container worlds.

### What platforms add

Above raw libvirt, platforms provide:

- **Clustering** — many hosts managed as one pool.
- **Scheduling/placement** — pick which host a new VM lands on (capacity, affinity, NUMA).
- **High availability (HA)** — restart VMs elsewhere automatically when a host fails.
- **Storage/network orchestration** — provision shared storage (Ceph) and software-defined
  networks across the cluster, automatically.
- **Multi-tenancy** — projects/tenants, quotas, RBAC, isolation.
- **Self-service APIs/UI** — users request VMs without touching hosts.

Each is automation of a manual step from Phases 4–13: scheduling = choosing a host + CPU
baseline (Lessons 17/49); HA = restart-on-failure built on migration; storage orchestration =
pools (Lesson 41) at fleet scale; networking = libvirt networks/bridges (Lesson 42) as
software-defined networking.

### The capstone

Build a tiny two-node "cloud" by hand: two KVM hosts, shared storage (NFS/Ceph pool), a
common CPU baseline, bridged networking, and live-migrate VMs between them with `virsh
migrate`. Then map each manual piece to what a platform automates — and you'll understand
exactly what Proxmox/oVirt/OpenStack/KubeVirt are doing for you.

### Where to go next

- Pick **one platform** and deploy it (Proxmox is the easiest start; KubeVirt if you're
  container-focused).
- Go deeper on **performance** (NUMA, SR-IOV, DPDK/vhost-user for NFV) or **security**
  (confidential VMs, attestation).
- Explore **alternative VMMs** (Cloud Hypervisor, Firecracker) and the **microVM/Kata**
  ecosystem (Lessons 59–60).
- Revisit the **networking track** — VM networking, K8s/CNI, and overlays are where the two
  curricula fully converge.

{: .note }
> **The unifying idea**
> A management platform is not magic — it's automation of the manual operations you now
> understand. Proxmox/oVirt orchestrate libvirt/QEMU/KVM across a cluster; OpenStack Nova
> schedules KVM VMs as cloud instances; KubeVirt runs them as Kubernetes pods. Scheduling,
> HA, storage/network orchestration, and multi-tenancy are each built from primitives you've
> used by hand: KVM execution, qemu-img/pools, libvirt domains/networks, and live migration.
> Knowing the primitives means you can reason about — and debug — any platform built on them.

---

## Lab

```bash
# A conceptual capstone: a two-node KVM "cloud" by hand, then map to platforms.

# 1. Two KVM hosts (hostA, hostB), each ready (Lesson 7), reachable via libvirt:
$ virsh -c qemu+ssh://hostA/system version
$ virsh -c qemu+ssh://hostB/system version

# 2. Shared storage both can see (NFS pool, Lesson 41) — same path on both:
$ virsh -c qemu+ssh://hostA/system pool-list | grep shared
$ virsh -c qemu+ssh://hostB/system pool-list | grep shared

# 3. A common CPU baseline so VMs migrate either direction (Lessons 17, 49):
$ virsh hypervisor-cpu-baseline /tmp/A.xml /tmp/B.xml

# 4. Bridged networking on both nodes so a VM keeps connectivity after moving (L42).

# 5. Live-migrate a VM hostA -> hostB, then back (Lesson 49):
$ virsh -c qemu+ssh://hostA/system migrate --live --persistent webvm \
    qemu+ssh://hostB/system
$ virsh -c qemu+ssh://hostB/system list      # webvm now runs on hostB

# 6. Now map each manual piece to platform automation:
#    shared pool   -> Proxmox/Ceph or OpenStack Cinder
#    CPU baseline  -> the scheduler's compute-capability matching
#    migrate cmd   -> HA / live auto-rebalancing
#    define+net    -> self-service VM creation API/UI (Nova, KubeVirt CRDs)

# 7. (Optional) Try a platform to see the automation:
#    - Proxmox VE: install the ISO, create a cluster, click "Migrate".
#    - KubeVirt: kubectl apply a VirtualMachine CR on a K8s cluster.
$ kubectl get vms -A 2>/dev/null    # if KubeVirt is installed
```

**Expected result:** With two KVM hosts, shared storage, a common CPU baseline, and bridged
networking, you can live-migrate a VM between them by hand — and you can name, for each
manual step, the platform feature (scheduler, HA, storage/network orchestration) that
automates it at scale.

---

## Further Reading

| Topic | Link |
|---|---|
| Proxmox VE | [Wikipedia — Proxmox Virtual Environment](https://en.wikipedia.org/wiki/Proxmox_Virtual_Environment) |
| oVirt | [Wikipedia — oVirt](https://en.wikipedia.org/wiki/OVirt) |
| OpenStack (Nova) | [Wikipedia — OpenStack](https://en.wikipedia.org/wiki/OpenStack) |
| KubeVirt | [kubevirt.io](https://kubevirt.io/) |
| Networking Lessons 43–44 (K8s/CNI) | [Kubernetes networking]({{ '/networking/lessons/lesson-43-kubernetes-networking.html' | relative_url }}) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Pick one management platform and list which manual steps from Phases 4–13 it automates for you.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Example — Proxmox VE (or oVirt): it automates VM creation (the QEMU command line / libvirt domain XML from Phases 4 and 10) behind a GUI/API; storage provisioning (qemu-img images and libvirt pools, Phase 6/Lesson 41) including clustered Ceph/NFS; networking (bridges and libvirt networks, Phase 8/Lesson 42) as cluster-wide software-defined networks; live migration (Lesson 49) as click-to-migrate and automatic HA restart on host failure; scheduling/placement (choosing a host and compatible CPU baseline, Lessons 17/49); snapshots/backups (Phase 12); and tuning defaults (Phase 13). In short, it orchestrates the libvirt/QEMU/KVM operations you did by hand across a whole cluster, adding clustering, HA, multi-tenancy, and self-service. (OpenStack Nova and KubeVirt automate the same primitives as a cloud IaaS or as Kubernetes workloads, respectively.)
</details>

---

**Q2. How does KubeVirt differ from Proxmox/oVirt in how it presents and manages VMs?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Proxmox and oVirt are dedicated virtualization managers: they present VMs through their own cluster manager and web UI/API, managing KVM hosts directly. KubeVirt instead runs VMs *as Kubernetes workloads* — a VM is a Kubernetes custom resource (VirtualMachine) scheduled onto nodes as a QEMU/KVM process inside a pod, managed with kubectl alongside containers, and integrated with Kubernetes networking (CNI) and storage (PVCs). So Proxmox/oVirt are VM-centric platforms, while KubeVirt folds VMs into the container/Kubernetes control plane — the convergence of the VM and container worlds.
</details>

---

**Q3. What capabilities do management platforms add on top of raw libvirt, and how does each map to something you did manually?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
They add: clustering (managing many hosts as one pool — vs your single-host virsh); scheduling/placement (auto-choosing a host with capacity and a compatible CPU baseline — vs you picking a host and computing cpu-baseline in Lessons 17/49); high availability (auto-restart/migrate VMs when a host fails — built on the live migration you ran by hand in Lesson 49); storage orchestration (provisioning shared Ceph/NFS across nodes — vs creating pools/volumes by hand in Lesson 41); network orchestration (software-defined networks across the cluster — vs bridges/libvirt networks in Lesson 42); and multi-tenancy/self-service (quotas, RBAC, APIs — new at the org level). Each is automation, at fleet scale, of a primitive you operated manually.
</details>

---

## Homework

Sketch (on paper or in a doc) a two-node KVM cloud: two hosts, shared storage, a common CPU baseline, bridged networking, and a VM you can live-migrate between them. For each component, write the manual command/concept you'd use (from earlier lessons) AND name the platform feature that would automate it. Then choose one platform (Proxmox, oVirt, OpenStack, or KubeVirt) to learn next and justify your choice based on your goals.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Sketch mapping (manual → platform): two hosts ready via virt-host-validate (Lesson 7) → platform's node onboarding/cluster join; shared storage via an NFS/Ceph libvirt pool reachable at the same path (Lesson 41) → orchestrated storage (Proxmox Ceph / OpenStack Cinder); common CPU model via virsh hypervisor-cpu-baseline so VMs run either way (Lessons 17/49) → the scheduler's CPU-compatibility matching; bridged networking on both nodes (Lesson 42) → cluster software-defined networking; a VM defined from domain XML (Lesson 40) → self-service VM creation API/UI; live migration via virsh migrate --live (Lesson 49) → one-click migrate + automatic HA on host failure. Platform choice (example justification): I'd learn Proxmox VE next because it's the fastest way to get a real clustered KVM environment with HA, Ceph, and backups on modest hardware (great for a homelab), and everything it does maps directly to the libvirt/QEMU primitives I now understand — so I can both use the GUI and drop to the CLI to debug. (If my goal were container-native infrastructure, I'd pick KubeVirt to manage VMs alongside pods via kubectl; if building a private IaaS cloud, OpenStack Nova.)
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Back to the learning plan →](../learning-plan.html){: .btn .btn-primary }

*🎉 You've reached the end of this track.*
