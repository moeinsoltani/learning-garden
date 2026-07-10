---
title: Learning Plan
nav_order: 2
---

<!--
  Curriculum for the Linux Virtualization (QEMU/KVM) track. Each phase and lesson
  heading links to its page under _virtualization/lessons/. See CLAUDE.md for the
  file format and methodology.
-->

# Linux Virtualization Learning Plan — QEMU & KVM

A practical, bottom-up curriculum: from "what does the CPU actually do when a guest
runs" up through QEMU/KVM internals, storage, virtio, passthrough, libvirt,
migration, tuning, security, and the modern microVM/Kata ecosystem.

Where a topic overlaps the **networking** curriculum (`learning-plan.md`) — TAP
devices, bridges, libvirt networks — this plan points back to it so the two tracks
reinforce each other.

{: .note }
> Before Lesson 1, prepare your VM with the [Lab Setup]({{ '/lab-setup.html' | relative_url }})
> page — this track needs `/dev/kvm` (nested virtualization if you learn inside a VM).
> Work the lessons in order — each builds on the ones before it.

---

## [Phase 1: Virtualization Foundations](lessons/phase-01-foundations.html)

### [Lesson 1: What virtualization actually is](lessons/lesson-01-what-virtualization-is.html)

**Goal:** Build the core mental model before touching any tool.

**Topics:**
- Emulation vs virtualization vs paravirtualization — who executes the guest's instructions
- Full virtualization (unmodified guest) vs paravirtualization (guest is virtualization-aware)
- The Popek & Goldberg requirements for a virtualizable architecture
- Guest, host, hypervisor (VMM), and "the world switch"
- Why x86 was historically *hard* to virtualize

**Checkpoint:** Explain the difference between QEMU emulating an ARM CPU on an x86 host and KVM running an x86 guest on an x86 host. Which one actually runs guest instructions natively?

---

### [Lesson 2: Hypervisor types and where KVM fits](lessons/lesson-02-hypervisor-types.html)

**Goal:** Place KVM/QEMU on the map of hypervisor designs.

**Topics:**
- Type 1 (bare-metal: ESXi, Xen, Hyper-V) vs Type 2 (hosted: VirtualBox)
- Why KVM blurs the line: it turns the Linux kernel *into* a Type 1-like hypervisor
- The Linux kernel as hypervisor; userspace VMM (QEMU) as the device model
- Comparison: KVM vs Xen vs VMware vs Hyper-V at a high level

**Checkpoint:** Is KVM a Type 1 or Type 2 hypervisor? Defend your answer — why is it arguable both ways?

---

### [Lesson 3: The QEMU + KVM division of labor](lessons/lesson-03-qemu-kvm-division.html)

**Goal:** Understand the single most important relationship in this curriculum.

**Topics:**
- KVM = the in-kernel module that uses CPU virtualization extensions to run guest code
- QEMU = the userspace process that emulates devices, memory, and the machine, and *drives* KVM
- The loop: QEMU `ioctl(KVM_RUN)` → guest runs on the CPU → VM exit → QEMU handles I/O → repeat
- What runs in the kernel vs what runs in userspace, and why that split exists
- Other VMMs that drive KVM: crosvm, Cloud Hypervisor, Firecracker (preview)

**Lab:**

```
   QEMU (userspace VMM)  ──ioctl──►  /dev/kvm (KVM kernel module)
        device model                       │
        memory mgmt                  VT-x / AMD-V runs guest vCPU
        I/O handling   ◄──VM exit────────  │
```

**Checkpoint:** When a guest writes to an emulated NIC register, where does that write get handled — kernel or userspace? What about a guest page fault on RAM?

---

## [Phase 2: CPU & Hardware Virtualization](lessons/phase-02-cpu-hardware-virt.html)

### [Lesson 4: Rings, privileged instructions, and trap-and-emulate](lessons/lesson-04-rings-trap-emulate.html)

**Goal:** Understand the classical technique and why plain x86 broke it.

**Topics:**
- x86 protection rings (0–3); kernel in ring 0, apps in ring 3
- Privileged vs sensitive instructions; trap-and-emulate
- Why a guest kernel "wants" ring 0 but can't have it
- The 17 problematic x86 instructions; historical workarounds (binary translation, ring deprivileging)

**Checkpoint:** Why can't you simply run a guest OS kernel directly in ring 0 alongside the host kernel?

---

### [Lesson 5: Hardware virtualization extensions (VT-x / AMD-V)](lessons/lesson-05-vtx-amdv.html)

**Goal:** Understand the hardware feature that makes KVM possible.

**Topics:**
- Intel VMX and AMD SVM; `vmx`/`svm` CPU flags
- VMX root mode (hypervisor) vs non-root mode (guest); `VMXON`/`VMLAUNCH`/`VMRESUME`
- The VMCS (Intel) / VMCB (AMD): the per-vCPU control structure
- VM entry and VM exit; exit reasons
- How this gives the guest its *own* ring 0 safely

**Checkpoint:** What is a VMCS, and what kind of guest activity causes a "VM exit" back to the hypervisor?

---

### [Lesson 6: Memory virtualization — EPT / NPT (SLAT)](lessons/lesson-06-ept-npt-slat.html)

**Goal:** Understand how guest physical memory maps to host physical memory.

**Topics:**
- Three address spaces: guest virtual → guest physical → host physical
- Shadow page tables (the old, slow software way) vs SLAT
- Intel EPT / AMD NPT (nested/second-level address translation)
- TLB, ASIDs/VPIDs; why SLAT dramatically cut VM-exit overhead
- Preview: the IOMMU does the analogous job for device DMA (Phase 9)

**Checkpoint:** Why is hardware SLAT (EPT/NPT) faster than shadow page tables? What extra translation step does it add, and why is that still a win?

---

### [Lesson 7: Checking and enabling virtualization on the host](lessons/lesson-07-host-readiness.html)

**Goal:** Get a host ready to run KVM.

**Topics:**
- `grep -E 'vmx|svm' /proc/cpuinfo`; `kvm-ok` (cpu-checker)
- BIOS/UEFI: enabling VT-x/AMD-V and VT-d/AMD-Vi
- `lsmod | grep kvm`; loading `kvm`, `kvm_intel`/`kvm_amd`
- Permissions: the `kvm` group and `/dev/kvm`
- `virt-host-validate` as a one-shot readiness check

**Lab:** From a fresh host, verify CPU support, confirm modules are loaded, run `virt-host-validate`, and fix any FAIL it reports.

**Checkpoint:** `/dev/kvm` exists but your VM falls back to slow TCG emulation. List three things you'd check.

---

## [Phase 3: KVM Internals](lessons/phase-03-kvm-internals.html)

### [Lesson 8: The /dev/kvm ioctl API](lessons/lesson-08-dev-kvm-ioctl.html)

**Goal:** See that KVM is "just" a character device with an ioctl interface.

**Topics:**
- `open("/dev/kvm")`; `KVM_CREATE_VM`, `KVM_CREATE_VCPU`
- Memory slots: `KVM_SET_USER_MEMORY_REGION` (guest RAM is mmap'd host memory)
- `KVM_RUN` and the shared `kvm_run` struct
- The minimal "run 16 bytes of guest code" example program (conceptual walkthrough)

**Lab:** Read and annotate a ~150-line C program that boots a tiny real-mode guest via raw KVM ioctls (no QEMU). Identify create-VM, create-vCPU, map-memory, and the KVM_RUN loop.

**Checkpoint:** How does guest RAM physically exist on the host? What KVM call establishes it?

---

### [Lesson 9: KVM kernel modules and parameters](lessons/lesson-09-kvm-modules-params.html)

**Goal:** Know the modules and the knobs that matter.

**Topics:**
- `kvm` (arch-independent core) + `kvm_intel` / `kvm_amd`
- Useful module params: `nested`, `ept`/`npt`, `halt_poll_ns`
- `/sys/module/kvm*/parameters/`
- Reloading modules with new params

**Checkpoint:** Which module provides the generic KVM logic, and which provides the Intel-specific VMX glue?

---

### [Lesson 10: The VM-exit loop in depth](lessons/lesson-10-vm-exit-loop.html)

**Goal:** Trace the most important control flow in the whole stack.

**Topics:**
- Exit reasons: I/O port access, MMIO, EPT violation, HLT, external interrupt, hypercall
- Which exits KVM handles in-kernel vs which it bounces to userspace (QEMU)
- In-kernel device acceleration: irqchip, PIT, why some I/O never leaves the kernel
- Cost of an exit; why reducing exits is the heart of performance tuning

**Checkpoint:** A guest executing a tight CPU loop with no I/O causes almost no VM exits. Why is that good, and what kinds of guest behavior *force* exits?

---

### [Lesson 11: Observing KVM at runtime](lessons/lesson-11-observing-kvm.html)

**Goal:** Make the invisible visible.

**Topics:**
- `kvm_stat` / `kvm_stat -1`: exit counters by reason
- `perf kvm stat`; `/sys/kernel/debug/kvm/`
- `trace-cmd`/ftrace `kvm` tracepoints
- Reading exit-reason histograms to spot pathological guests

**Lab:** Run a guest, watch `kvm_stat` live, then trigger heavy disk and network I/O and watch which exit reasons spike.

**Checkpoint:** You see a huge count of MMIO exits. What does that suggest about the guest's device configuration, and what would you change?

---

## [Phase 4: QEMU Fundamentals](lessons/phase-04-qemu-fundamentals.html)

### [Lesson 12: Anatomy of a QEMU command line](lessons/lesson-12-qemu-command-line.html)

**Goal:** Boot your first VM and understand every flag.

**Topics:**
- `qemu-system-x86_64` vs other targets (`qemu-system-aarch64`, etc.)
- Minimal boot: `-m`, `-smp`, `-drive`/`-cdrom`, `-nographic`
- The object/device model: `-device`, `-drive`, `-netdev`, `-object`
- `-machine`, `-cpu`, `-accel`

**Lab:**

```
qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
  -drive file=disk.qcow2,if=virtio -nographic
```

Boot a small Linux image; reach a login prompt over serial.

**Checkpoint:** What is the difference between `-drive` and `-device`, and how do they relate?

---

### [Lesson 13: Machine types and accelerators](lessons/lesson-13-machine-types-accel.html)

**Goal:** Understand the virtual motherboard and how guest code is executed.

**Topics:**
- Machine types: `pc` (i440FX) vs `q35`; versioned machine types and migration compatibility
- `-accel kvm` (hardware) vs `-accel tcg` (pure emulation) vs `hvf`/`whpx` on other hosts
- Why `q35` for modern guests (PCIe, better passthrough)
- `-cpu host` vs named models (preview of Phase 5)

**Checkpoint:** Why do versioned machine types (`pc-q35-8.2`) exist? What breaks if you change a running VM's machine type on migration?

---

### [Lesson 14: Firmware and the boot path](lessons/lesson-14-firmware-boot.html)

**Goal:** Understand how the virtual machine starts executing.

**Topics:**
- SeaBIOS (legacy BIOS) vs UEFI via OVMF
- `-bios` / `pflash` for OVMF; the NVRAM varstore
- Secure Boot in a VM
- Boot order, `-boot`, PXE boot
- The boot sequence: firmware → bootloader → guest kernel

**Lab:** Boot the same disk image once with SeaBIOS and once with OVMF (UEFI); observe the difference.

**Checkpoint:** What is OVMF, and why does a UEFI guest need a writable NVRAM/varstore file?

---

### [Lesson 15: The QEMU monitor — HMP and QMP](lessons/lesson-15-monitor-hmp-qmp.html)

**Goal:** Learn to introspect and control a running VM.

**Topics:**
- Human Monitor Protocol (HMP): `info`, `device_add`, `system_powerdown`, hotplug
- QEMU Machine Protocol (QMP): the JSON API automation uses
- Connecting a monitor: `-monitor`, `-qmp`; multiplexing on serial
- `info cpus`, `info block`, `info network`, `info migrate`

**Lab:** Attach to the QMP socket, issue `query-status` and `query-block` by hand, then hot-add a disk via HMP `device_add`.

**Checkpoint:** What does libvirt use under the hood to talk to a running QEMU process — HMP or QMP? Why does the distinction matter for automation?

---

### [Lesson 16: Display, console, and remote access](lessons/lesson-16-display-console.html)

**Goal:** Get pixels and serial out of a VM.

**Topics:**
- Serial console (`-nographic`, `-serial`) vs graphical
- VGA models: `std`, `qxl`, `virtio-gpu`
- VNC and SPICE; clipboard/USB redirection with SPICE
- `virt-viewer`, `remote-viewer`

**Checkpoint:** When would you prefer SPICE over VNC for a desktop guest? When is a plain serial console the right answer?

---

## [Phase 5: CPU & Memory Configuration](lessons/phase-05-cpu-memory.html)

### [Lesson 17: vCPU models and feature flags](lessons/lesson-17-vcpu-models.html)

**Goal:** Control what CPU the guest believes it has.

**Topics:**
- `host-passthrough` vs `host-model` vs named models (e.g. `Haswell-noTSX`)
- CPU feature flags: enabling/disabling (`+aes`, `-svm`)
- Migration safety: why `host-passthrough` hurts migration across heterogeneous hosts
- CPUID, microarchitecture levels, and mitigations (spectre/meltdown flags)

**Checkpoint:** Why might `host-passthrough` give the best performance but be the wrong choice for a migration cluster?

---

### [Lesson 18: CPU topology](lessons/lesson-18-cpu-topology.html)

**Goal:** Present sockets/cores/threads sensibly to the guest.

**Topics:**
- `-smp cpus=,sockets=,cores=,threads=`
- Why topology affects guest scheduler and licensing
- Overcommitting vCPUs; vCPU:pCPU ratios
- Intro to pinning (full treatment in Phase 13)

**Checkpoint:** A guest with 8 vCPUs presented as 8 sockets behaves differently from 1 socket × 4 cores × 2 threads. Why might the guest care?

---

### [Lesson 19: Guest memory backends and hugepages](lessons/lesson-19-memory-hugepages.html)

**Goal:** Control how guest RAM is backed on the host.

**Topics:**
- `-object memory-backend-ram` / `memory-backend-file`
- Transparent hugepages vs explicit hugepages (2 MiB / 1 GiB)
- `prealloc`, `share`, `mem-path` (hugetlbfs)
- Memory overcommit and its risks

**Lab:** Reserve 1 GiB hugepages on the host, back a guest with them, and confirm via `/proc/meminfo` and the guest's perception.

**Checkpoint:** Why do explicit hugepages reduce TLB pressure and improve performance for large guests?

---

### [Lesson 20: Ballooning and KSM](lessons/lesson-20-balloon-ksm.html)

**Goal:** Reclaim and deduplicate memory across guests.

**Topics:**
- `virtio-balloon`: how the host reclaims guest memory cooperatively
- Inflate/deflate; limits and pitfalls
- KSM (Kernel Samepage Merging): merging identical pages across VMs
- `ksmd`, `/sys/kernel/mm/ksm/`; CPU cost vs memory savings; security caveats

**Checkpoint:** What does the balloon driver actually do to "give memory back" to the host, and why does it require guest cooperation?

---

### [Lesson 21: NUMA awareness](lessons/lesson-21-numa.html)

**Goal:** Avoid the classic cross-node performance cliff.

**Topics:**
- Host NUMA topology: `numactl -H`, `lscpu`
- Pinning guest memory and vCPUs to a single node
- Exposing a virtual NUMA topology to large guests
- `numad`/automatic NUMA balancing interactions

**Checkpoint:** Why can a VM whose vCPUs and memory straddle two NUMA nodes be much slower than one confined to a single node?

---

## [Phase 6: Storage](lessons/phase-06-storage.html)

### [Lesson 22: Disk image formats and qemu-img](lessons/lesson-22-image-formats-qemu-img.html)

**Goal:** Master the most-used QEMU tool after the emulator itself.

**Topics:**
- `raw` vs `qcow2` (and a nod to vmdk/vdi/vhdx)
- `qemu-img create / info / convert / resize / check`
- Thin vs thick provisioning; sparseness
- Choosing a format: performance (raw) vs features (qcow2)

**Lab:** Create a qcow2, inspect it, convert it to raw and back, resize it, and read the metadata with `qemu-img info`.

**Checkpoint:** When would you choose raw over qcow2 despite losing snapshots and compression?

---

### [Lesson 23: qcow2 internals — backing files and copy-on-write](lessons/lesson-23-qcow2-internals.html)

**Goal:** Understand the format that powers templates, clones, and snapshots.

**Topics:**
- L1/L2 reference tables; clusters; lazy allocation
- Backing files and overlays; the copy-on-write chain
- `qemu-img create -b base.qcow2 overlay.qcow2`
- `qemu-img rebase` and `commit`; flattening a chain

**Lab:**

```
base.qcow2  ◄──backing──  overlay-A.qcow2   (VM A's writes)
            ◄──backing──  overlay-B.qcow2   (VM B's writes)
```

Build a base + two overlays; show that writes land in overlays and the base stays pristine.

**Checkpoint:** Two VMs share one read-only base image via separate overlays. Where does each VM's data go, and what happens to VM A if you delete the base?

---

### [Lesson 24: Block device models](lessons/lesson-24-block-device-models.html)

**Goal:** Connect images to the guest the right way.

**Topics:**
- Emulated controllers: IDE, AHCI/SATA, virtio-blk, virtio-scsi
- `-blockdev` (modern) vs `-drive` (legacy); the `-device` pairing
- virtio-blk vs virtio-scsi: when each wins (queues, passthrough, many disks, TRIM/UNMAP)
- Discard/TRIM and thin reclamation

**Checkpoint:** You need 30 disks and SCSI passthrough on one guest. virtio-blk or virtio-scsi? Why?

---

### [Lesson 25: Caching modes and AIO](lessons/lesson-25-caching-aio.html)

**Goal:** Trade safety against speed deliberately, not accidentally.

**Topics:**
- `cache=` modes: `none`, `writeback`, `writethrough`, `directsync`, `unsafe`
- Host page cache vs O_DIRECT; where the data actually is on a crash
- AIO backends: `threads`, `native`, `io_uring`
- How cache mode interacts with guest flush/FUA semantics

**Checkpoint:** Which cache mode risks data loss on host power failure, and which is the usual production default for correctness + performance?

---

### [Lesson 26: Snapshots](lessons/lesson-26-snapshots.html)

**Goal:** Capture and roll back VM state.

**Topics:**
- Internal snapshots (inside one qcow2) vs external snapshots (overlay chain)
- Disk-only vs full (disk + RAM) snapshots
- Live snapshots via QMP (`blockdev-snapshot`)
- Pitfalls: snapshot sprawl, performance, deleting/merging (`blockdev-commit`)

**Lab:** Take an internal snapshot with `qemu-img snapshot`, then an external live snapshot via QMP, and merge it back.

**Checkpoint:** What is the difference between an internal and an external snapshot, and why do production tools usually prefer external?

---

## [Phase 7: VirtIO & Paravirtualization](lessons/phase-07-virtio.html)

### [Lesson 27: The virtio standard](lessons/lesson-27-virtio-standard.html)

**Goal:** Understand the paravirtual device framework everything else builds on.

**Topics:**
- Why paravirtual devices beat emulated hardware (fewer exits, shared rings)
- Virtqueues and vrings: descriptor/available/used rings
- Split vs packed virtqueues
- Feature negotiation; the virtio device/driver handshake
- virtio-pci vs virtio-mmio transports

**Checkpoint:** Why does a virtio-net device cause far fewer VM exits than an emulated e1000 NIC?

---

### [Lesson 28: The core virtio devices](lessons/lesson-28-virtio-devices.html)

**Goal:** Know the catalog and what each is for.

**Topics:**
- virtio-net, virtio-blk, virtio-scsi
- virtio-rng (entropy), virtio-balloon (Phase 5), virtio-console/serial
- virtio-gpu, virtio-fs (shared host directory), virtio-vsock
- Guest driver requirements (Linux built-in; Windows needs virtio-win)

**Checkpoint:** A Windows guest sees "unknown device" for its virtio NIC and disk. What's missing, and how do you fix installation order (boot disk!) ?

---

### [Lesson 29: vhost — moving the datapath into the kernel](lessons/lesson-29-vhost.html)

**Goal:** Understand the first big performance offload.

**Topics:**
- The problem: virtio datapath in QEMU userspace = extra copies and context switches
- vhost-net: the kernel handles the virtqueue datapath directly
- vhost-scsi; the general vhost model (kernel worker threads)
- What still goes through QEMU (control plane) vs kernel (data plane)

**Checkpoint:** With vhost-net, which part of packet processing no longer requires switching into the QEMU process, and why does that speed things up?

---

### [Lesson 30: vhost-user, vsock, and shared-memory datapaths](lessons/lesson-30-vhost-user-vsock.html)

**Goal:** See where high-performance and host/guest comms go next.

**Topics:**
- vhost-user: datapath handled by *another userspace process* (DPDK, SPDK, OVS-DPDK)
- Shared memory + eventfd/irqfd signalling
- virtio-vsock: socket communication between host and guest without networking
- virtio-fs vs 9p for filesystem sharing

**Checkpoint:** vhost-net moves the datapath to the kernel; vhost-user moves it to a userspace process. What problem is vhost-user solving that the in-kernel version can't?

---

## [Phase 8: Networking for VMs](lessons/phase-08-vm-networking.html)

> This phase deliberately reuses the **networking curriculum** (`learning-plan.md`):
> TAP (Lesson 14), bridges (Lesson 11), and libvirt networks (Phase 9 below).

### [Lesson 31: User-mode networking (SLIRP)](lessons/lesson-31-slirp.html)

**Goal:** Understand the zero-config default and its limits.

**Topics:**
- `-netdev user` (SLIRP): NAT in userspace, no privileges needed
- Built-in DHCP/DNS; the `10.0.2.0/24` default
- Port forwarding: `hostfwd`
- Why SLIRP is slow and can't accept inbound by default

**Lab:** Boot a guest with user networking, reach the internet, and forward host `:2222 → guest :22` for SSH.

**Checkpoint:** Why can't another machine on your LAN initiate a connection to a SLIRP guest without `hostfwd`?

---

### [Lesson 32: TAP + bridge networking](lessons/lesson-32-tap-bridge.html)

**Goal:** Put guests on a real L2 network — the production pattern.

**Topics:**
- TAP devices (recall **Networking Lesson 14**) as the guest's "cable"
- Attaching a TAP to a Linux bridge (recall **Networking Lesson 11**)
- `-netdev tap,...` + `-device virtio-net-pci`
- qemu bridge helper; static vs scripted TAP setup

**Lab:**

```
guest eth0 ─ virtio-net ─ tap0 ─ br0 ─ host eth0 ─ LAN
```

Bridge a guest onto the host LAN; confirm it gets a LAN IP and is reachable from other machines.

**Checkpoint:** Trace a packet from the guest's `eth0` to the physical LAN. Which piece is the TAP and which is the bridge?

---

### [Lesson 33: Accelerated networking — vhost-net and multiqueue](lessons/lesson-33-accelerated-net.html)

**Goal:** Make virtio-net fast.

**Topics:**
- Enabling `vhost=on`; verifying with `kvm_stat`/`ethtool`
- Multiqueue virtio-net: scaling to multiple vCPUs
- Offloads (checksum, TSO/GSO) and when to toggle them
- Measuring throughput (`iperf3`, recall **Networking Lesson 33**)

**Checkpoint:** Single-queue virtio-net caps out at one CPU's worth of packet processing. How does multiqueue fix this, and what must the guest also do?

---

### [Lesson 34: macvtap and SR-IOV networking](lessons/lesson-34-macvtap-sriov.html)

**Goal:** Know the lower-overhead alternatives to bridging.

**Topics:**
- macvtap (recall **Networking Lesson 13** MACVLAN): bridgeless guest-on-LAN
- macvtap modes (bridge/vepa/private) and the host-can't-talk-to-guest gotcha
- SR-IOV: physical NIC virtual functions assigned to guests (full passthrough in Phase 9)
- Trade-offs: simplicity vs performance vs live-migration support

**Checkpoint:** Why can the host often *not* reach a guest that's attached via macvtap, and what does that imply for management traffic?

---

## [Phase 9: Device Passthrough (VFIO & IOMMU)](lessons/phase-09-passthrough.html)

### [Lesson 35: The IOMMU](lessons/lesson-35-iommu.html)

**Goal:** Understand the hardware that makes safe passthrough possible.

**Topics:**
- Intel VT-d / AMD-Vi; DMA remapping (the "EPT for devices")
- Why passthrough without an IOMMU is unsafe (a device could DMA anywhere)
- IOMMU groups: the unit of assignment, and why they matter
- Enabling: `intel_iommu=on` / `amd_iommu=on`, kernel cmdline

**Lab:** Enable the IOMMU, then enumerate IOMMU groups under `/sys/kernel/iommu_groups/` and identify which devices are grouped together.

**Checkpoint:** Why must you pass through an *entire* IOMMU group, not just one device in it?

---

### [Lesson 36: VFIO PCI passthrough](lessons/lesson-36-vfio-pci.html)

**Goal:** Assign a real PCI device to a guest.

**Topics:**
- The `vfio-pci` driver; unbinding from the host driver and binding to vfio-pci
- `-device vfio-pci,host=0000:01:00.0`
- Persisting binding (driverctl, modprobe options, initramfs)
- Resetting devices (FLR) and the reset bug class

**Lab:** Bind a spare NIC to vfio-pci and pass it through to a guest; confirm the guest drives it directly and the host no longer sees it.

**Checkpoint:** What happens to the host's use of a device once it's bound to vfio-pci and handed to a guest?

---

### [Lesson 37: GPU passthrough](lessons/lesson-37-gpu-passthrough.html)

**Goal:** Tackle the hardest, most popular passthrough case.

**Topics:**
- GPU + its audio function in the same IOMMU group
- The infamous NVIDIA "Code 43"; vendor-reset; ROM/vBIOS issues
- Single-GPU vs dual-GPU host setups
- Looking-glass / VFIO for desktop workstations (overview)

**Checkpoint:** Why is GPU passthrough notably harder than NIC passthrough? Name two GPU-specific obstacles.

---

### [Lesson 38: SR-IOV and mediated devices](lessons/lesson-38-sriov-mdev.html)

**Goal:** Share one physical device among many guests.

**Topics:**
- SR-IOV: Physical Function (PF) vs Virtual Functions (VFs); `sriov_numvfs`
- Assigning VFs to guests via VFIO
- Mediated devices (mdev): time-sliced sharing (e.g. NVIDIA vGPU, Intel GVT-g)
- Trade-offs vs full passthrough; live-migration limitations

**Checkpoint:** What is the difference between passing through a whole device, an SR-IOV VF, and an mdev? Order them by isolation vs sharing.

---

## [Phase 10: libvirt — The Management Layer](lessons/phase-10-libvirt.html)

### [Lesson 39: libvirt architecture and virsh](lessons/lesson-39-libvirt-virsh.html)

**Goal:** Step up from raw QEMU to a managed, persistent API.

**Topics:**
- libvirtd / modular daemons (virtqemud, virtnetworkd, …); the stateless API
- Drivers (qemu/kvm, lxc, …) and connection URIs (`qemu:///system` vs `qemu:///session`)
- `virsh` essentials: `list`, `start`, `shutdown`, `destroy`, `define`, `undefine`
- How libvirt generates the QEMU command line for you (`virsh domxml-to-native`)

**Lab:** Take the QEMU command line from Phase 4 and find the equivalent libvirt domain; run `virsh domxml-to-native` to see libvirt regenerate a QEMU invocation.

**Checkpoint:** What's the difference between `qemu:///system` and `qemu:///session`, and when would you use each?

---

### [Lesson 40: The domain XML](lessons/lesson-40-domain-xml.html)

**Goal:** Read and write the canonical VM definition.

**Topics:**
- Structure: `<domain>`, `<cpu>`, `<memory>`, `<devices>` (disks, interfaces, controllers)
- `virsh edit`; live vs persistent config (`--config`/`--live`)
- Mapping every QEMU flag from Phases 4–8 onto its XML element
- Hotplug via `virsh attach-device`/`detach-device`

**Checkpoint:** You change `<vcpu>` in `virsh edit` but the running guest doesn't change. Why, and how do you apply it?

---

### [Lesson 41: Storage pools and volumes](lessons/lesson-41-storage-pools.html)

**Goal:** Let libvirt manage disk storage.

**Topics:**
- Pool types: dir, LVM, NFS, iSCSI, RBD/Ceph, ZFS
- `virsh pool-define/start/autostart`; `vol-create`, `vol-clone`
- How pools relate to qcow2/raw from Phase 6
- Permissions and sVirt labelling of volumes (preview Phase 14)

**Checkpoint:** What does a libvirt storage *pool* abstract that you were doing by hand with `qemu-img` earlier?

---

### [Lesson 42: libvirt virtual networks](lessons/lesson-42-libvirt-networks.html)

**Goal:** Manage VM networking declaratively — bridging the two curricula.

**Topics:**
- Network modes: NAT (default `virbr0` + dnsmasq), routed, bridged, isolated, open
- `virsh net-define/start/autostart`; the network XML
- How libvirt's NAT network is just bridge + dnsmasq + nftables (recall **Networking Phases 3 & 7**)
- Connecting a domain to an existing host bridge

**Lab:** Inspect the default NAT network; find `virbr0`, the dnsmasq instance, and the masquerade rules — and map each to a concept from the networking curriculum.

**Checkpoint:** libvirt's default network "just works" with NAT. Name the three networking primitives (from the other curriculum) it wires together to do that.

---

### [Lesson 43: virt-install and the desktop tools](lessons/lesson-43-virt-install.html)

**Goal:** Create and operate VMs efficiently.

**Topics:**
- `virt-install`: scripted VM creation
- `virt-manager` (GUI), `virt-viewer`/`remote-viewer`
- Autostart, managed save, lifecycle hooks
- `osinfo`/`virt-install --osinfo` for sane device defaults per guest OS

**Lab:** Create a VM end-to-end with a single `virt-install` command from a cloud/ISO image.

**Checkpoint:** Why does telling `virt-install` the correct guest OS (`--osinfo`) matter for the devices it picks?

---

## [Phase 11: Guest Images & Provisioning](lessons/phase-11-images-provisioning.html)

### [Lesson 44: Cloud images and cloud-init](lessons/lesson-44-cloud-init.html)

**Goal:** Stop installing OSes by hand.

**Topics:**
- What a cloud image is (pre-installed, cloud-init enabled, grows on first boot)
- cloud-init data sources; the NoCloud datasource for local labs
- `user-data` (users, SSH keys, packages, run-cmd) and `meta-data`
- Building a seed ISO; first-boot vs per-boot modules

**Lab:** Boot an Ubuntu cloud image with a NoCloud seed that injects your SSH key and a hostname; log in without ever touching an installer.

**Checkpoint:** Where does cloud-init get its configuration from on first boot, and why does a fresh cloud image need *something* like a seed ISO?

---

### [Lesson 45: libguestfs — editing images without booting them](lessons/lesson-45-libguestfs.html)

**Goal:** Manipulate guest disks offline.

**Topics:**
- `guestfish` interactive shell; the libguestfs appliance model
- `virt-customize` (install packages, set passwords, run commands)
- `virt-sysprep` (de-personalize before templating)
- `virt-cat`, `virt-edit`, `virt-df`, `virt-resize`

**Lab:** Use `virt-customize` to inject a package and a root password into a qcow2 *without booting it*, then `virt-sysprep` it into a clean template.

**Checkpoint:** Why run `virt-sysprep` before turning a VM disk into a template? What kinds of state does it remove?

---

### [Lesson 46: Templates and cloning](lessons/lesson-46-templates-cloning.html)

**Goal:** Mass-produce VMs.

**Topics:**
- Full clone vs linked clone (backing-file overlay — recall Phase 6)
- `virt-clone`; libvirt volume cloning
- Golden image workflow: customize → sysprep → template → clone
- Identity collisions (MACs, machine-id, SSH host keys) and how to avoid them

**Checkpoint:** A linked clone starts instantly and uses almost no disk. What is it built on (from Phase 6), and what's the downside?

---

## [Phase 12: Snapshots, Backup & Live Migration](lessons/phase-12-migration.html)

### [Lesson 47: Domain snapshots, checkpoints, and incremental backup](lessons/lesson-47-backup-checkpoints.html)

**Goal:** Protect and capture VM state at the management layer.

**Topics:**
- `virsh snapshot-create-as`; disk-only vs full system snapshots
- Checkpoints and dirty bitmaps; incremental/differential backup
- `virsh backup-begin` / `backup` API; integration with backup tools
- Snapshot vs backup vs replication — different jobs

**Checkpoint:** What does a dirty bitmap let a backup tool do that a naive "copy the whole disk every night" approach cannot?

---

### [Lesson 48: Live migration mechanics](lessons/lesson-48-migration-mechanics.html)

**Goal:** Move a running VM between hosts with minimal downtime.

**Topics:**
- Pre-copy: iteratively copy dirty pages, then a brief stop-and-copy
- Post-copy: switch first, fault pages in on demand
- The convergence problem (a busy guest dirties pages faster than you can send)
- Auto-converge, throttling, and downtime targets

**Checkpoint:** Explain pre-copy vs post-copy. Which risks the VM "pausing forever" on a write-heavy guest, and which risks the VM dying if the network drops mid-migration?

---

### [Lesson 49: Live migration in practice](lessons/lesson-49-migration-practice.html)

**Goal:** Actually migrate something.

**Topics:**
- `virsh migrate` (peer-to-peer, tunnelled, `--live`)
- Shared storage (NFS/Ceph) vs storage migration (`--copy-storage-all`)
- CPU compatibility & the baseline-CPU problem (recall Lesson 17)
- Network/firewall requirements; verifying with `virsh domjobinfo`

**Lab:** Live-migrate a running VM between two hosts (or two libvirt URIs) backed by shared storage; watch `domjobinfo` and confirm zero data loss.

**Checkpoint:** Why does live migration usually require shared storage *or* an explicit storage-copy flag, and why must the destination CPU be "compatible"?

---

## [Phase 13: Performance Tuning & Observability](lessons/phase-13-performance.html)

### [Lesson 50: CPU pinning and thread placement](lessons/lesson-50-cpu-pinning.html)

**Goal:** Eliminate scheduling jitter for latency-sensitive guests.

**Topics:**
- vCPU → pCPU pinning (`virsh vcpupin`, `<cputune>`)
- `isolcpus`, `nohz_full`, `rcu_nocbs` for dedicated cores
- Placing the QEMU emulator thread and I/O threads off the vCPU cores
- Hyperthread siblings and noisy-neighbour effects

**Checkpoint:** Why pin the *emulator* thread to different cores than the vCPU threads for a real-time guest?

---

### [Lesson 51: Memory and NUMA tuning in production](lessons/lesson-51-memory-numa-tuning.html)

**Goal:** Apply Phase 5 knowledge for real workloads.

**Topics:**
- Hugepage pools sized for the guest set; 1 GiB pages
- `<numatune>` and `<memnode>`; strict vs preferred binding
- Disabling automatic NUMA balancing for pinned guests
- Avoiding swap for VM hosts

**Checkpoint:** You pinned vCPUs to node 0 but left memory unpinned. What can go wrong, and what completes the fix?

---

### [Lesson 52: Storage and I/O tuning](lessons/lesson-52-storage-io-tuning.html)

**Goal:** Get the disk subsystem out of the way.

**Topics:**
- iothreads and assigning disks to them
- virtio-blk multiqueue; queue depth
- `cache=none` + `aio=native`/`io_uring` revisited for throughput
- Discard/TRIM, write-back vs guarantees, and benchmarking with `fio`

**Checkpoint:** What does dedicating an iothread to a busy disk accomplish that the default (main loop) handling does not?

---

### [Lesson 53: Observability — measuring a running VM](lessons/lesson-53-observability.html)

**Goal:** Diagnose performance like the networking track diagnoses packets.

**Topics:**
- `kvm_stat`, `perf kvm stat`, `perf kvm top` (revisit Lesson 11)
- Host view (`virt-top`, `top`/`htop` per thread) vs guest view
- Tracing exits and I/O; identifying the bottleneck layer
- A bottom-up methodology: CPU exits → memory/NUMA → disk → net (mirror of **Networking Lesson 47**)

**Checkpoint:** A guest "feels slow." Outline the order of checks you'd run, from CPU exits down to I/O, and which tool answers each.

---

### [Lesson 54: tuned and host profiles](lessons/lesson-54-tuned-profiles.html)

**Goal:** Apply sane defaults quickly.

**Topics:**
- `tuned` profiles: `virtual-host`, `virtual-guest`
- What they change (governor, THP, dirty ratios, sched knobs)
- When to override the profile
- Host-vs-guest profile pairing

**Checkpoint:** What category of settings does the `virtual-host` tuned profile adjust, and why are host and guest profiles different?

---

## [Phase 14: Nested Virtualization & Security](lessons/phase-14-nested-security.html)

### [Lesson 55: Nested virtualization](lessons/lesson-55-nested-virt.html)

**Goal:** Run KVM inside KVM (for labs, CI, and this very curriculum).

**Topics:**
- Enabling: `kvm_intel nested=1` / `kvm_amd nested=1`; exposing `vmx`/`svm` to the guest
- How nested EPT/VMCS shadowing works (briefly) and why it's slower
- Use cases: testing hypervisors, CI runners, training environments
- Limitations and migration caveats

**Lab:** Enable nesting, expose virtualization to an L1 guest, and successfully boot an L2 guest inside it.

**Checkpoint:** What host setting must change to let a guest itself run KVM, and why is nested virtualization inherently slower than a single level?

---

### [Lesson 56: sVirt — confining QEMU with MAC](lessons/lesson-56-svirt.html)

**Goal:** Contain the blast radius if a QEMU process is compromised.

**Topics:**
- The threat: QEMU is a large userspace process touching guest data and host resources
- sVirt with SELinux (`svirt_t`) or AppArmor; per-domain dynamic labels
- How libvirt labels disks/sockets per VM so one VM can't touch another's resources
- Debugging denials (`audit2why`, AppArmor logs)

**Checkpoint:** How does sVirt stop a compromised QEMU process for VM A from reading VM B's disk image, even though both run as the same user?

---

### [Lesson 57: QEMU hardening and sandboxing](lessons/lesson-57-qemu-hardening.html)

**Goal:** Reduce QEMU's privileges and attack surface.

**Topics:**
- Running QEMU as an unprivileged user (`-runas`, libvirt's `qemu` user)
- seccomp sandbox (`-sandbox on`): syscall filtering
- Dropping capabilities; namespaces around QEMU
- Minimizing the device model (fewer emulated devices = smaller surface)

**Checkpoint:** Why does removing unused emulated devices from a VM improve security, not just performance?

---

### [Lesson 58: Confidential computing](lessons/lesson-58-confidential-computing.html)

**Goal:** Understand memory encryption and attestation for VMs.

**Topics:**
- The threat model: protecting the guest *from the host/hypervisor*
- AMD SEV / SEV-ES / SEV-SNP; Intel TDX (high level)
- Encrypted guest memory, encrypted register state, integrity, attestation
- What changes operationally (migration, debugging, device model constraints)

**Checkpoint:** Traditional virtualization trusts the hypervisor completely. What threat do SEV-SNP / TDX address that ordinary KVM does not?

---

## [Phase 15: Lightweight VMs & the Ecosystem](lessons/phase-15-lightweight-ecosystem.html)

### [Lesson 59: microVMs](lessons/lesson-59-microvms.html)

**Goal:** Meet the stripped-down VMs behind serverless and sandboxing.

**Topics:**
- QEMU `microvm` machine type: no PCI, no BIOS, minimal devices, fast boot
- Firecracker and Cloud Hypervisor: Rust VMMs that also drive `/dev/kvm`
- Why a tiny device model = millisecond boots and tiny memory overhead
- Trade-offs vs full QEMU (no passthrough, limited devices)

**Lab:** Boot a guest with QEMU's `microvm` machine type (or Firecracker) and measure boot time vs a full `q35` VM.

**Checkpoint:** Firecracker boots in tens of milliseconds. Name two things it removes from the classic QEMU machine to get there.

---

### [Lesson 60: VM-isolated containers and the VM-vs-container question](lessons/lesson-60-kata-containers.html)

**Goal:** Place virtualization next to containers honestly.

**Topics:**
- Kata Containers: OCI containers each wrapped in a microVM
- The isolation spectrum: namespaces (containers) → microVM → full VM → separate host
- gVisor as a contrasting (userspace-kernel) approach
- Choosing: density vs isolation vs compatibility

**Checkpoint:** Kata runs each container in its own tiny VM. What security property does that buy over a plain runc container, and what does it cost?

---

### [Lesson 61: Management platforms and where to go next](lessons/lesson-61-platforms-next.html)

**Goal:** See how everything composes at scale, and chart further study.

**Topics:**
- libvirt-based platforms: Proxmox VE, oVirt/RHV
- OpenStack Nova; KubeVirt (VMs as Kubernetes workloads — ties to the **networking** K8s lessons)
- What these add: clustering, scheduling, HA, storage/network orchestration, multi-tenancy
- A capstone: build a small two-node KVM "cloud" by hand, then map each piece to what a platform automates

**Checkpoint:** Pick one management platform and list which manual steps from Phases 4–13 it automates for you.

---

## Recommended Learning Order

Do not skip ahead. Each step depends on the previous ones.

1. Virtualization concepts (emulation vs virtualization; hypervisor types)
2. QEMU + KVM division of labor
3. CPU extensions (VT-x/AMD-V), VMCS, VM exits
4. Memory virtualization (EPT/NPT); host readiness
5. KVM ioctl model and the VM-exit loop
6. QEMU command line, machine types, accelerators
7. Firmware/boot; HMP/QMP monitors
8. vCPU models, topology, memory backends, hugepages, NUMA
9. Storage: qemu-img, qcow2, block devices, caching, snapshots
10. virtio and vhost (paravirtual performance)
11. VM networking: SLIRP → TAP+bridge → vhost-net → macvtap *(reuse the networking curriculum)*
12. IOMMU + VFIO passthrough; SR-IOV; mdev
13. libvirt: virsh, domain XML, pools, networks, virt-install
14. Cloud images, cloud-init, libguestfs, templating/cloning
15. Snapshots, backup, live migration
16. Performance tuning (pinning, NUMA, I/O) and observability
17. Nested virtualization
18. Security: sVirt, sandboxing, confidential computing
19. microVMs, Kata, and management platforms

---

## Cross-references to the Networking Curriculum

This track and `learning-plan.md` are designed to interlock:

| Virtualization topic | Reuses networking lesson |
|---|---|
| TAP-based VM networking (L32) | Networking L14 (TAP interfaces) |
| Bridging VMs onto the LAN (L32) | Networking L11 (Linux bridges) |
| macvtap (L34) | Networking L13 (MACVLAN) |
| Throughput measurement (L33) | Networking L33 (iperf3/bandwidth) |
| libvirt NAT network internals (L42) | Networking Phases 3 & 7 (bridges, NAT/nftables) |
| KubeVirt / platform networking (L61) | Networking L43–44 (Kubernetes, CNI) |
| Performance debugging methodology (L53) | Networking L47 (layer-by-layer methodology) |
