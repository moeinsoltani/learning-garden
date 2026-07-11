# CLAUDE.md — Linux Systems Learning Project

This repo hosts **six independent learning tracks** on one Just the Docs site:
- **Networking** — in the `_networking/` collection
- **Virtualization (QEMU/KVM)** — in the `_virtualization/` collection
- **Security & Identity** — in the `_security/` collection
- **Operating Systems** — in the `_os/` collection
- **Engineering Leadership** — in the `_leadership/` collection (scenario labs, no terminal)
- **English for Work** — in the `_english/` collection (rewrite-drill labs, no terminal)

## Student Profile
- Linux experience: was a beginner; has now completed the networking, virtualization,
  and security tracks — the OS track can build on all three (cross-link freely)
- Day job: Senior Software Developer aiming for Lead/EM (the leadership track's audience)
- Non-native English speaker; common error patterns: subject–verb agreement,
  articles, dropped words. Wants a warm, soft tone in messages (the English track's focus)
- Platform: Windows 11, with access to a Linux VM and WSL2
- Use the Linux VM for labs (WSL2 as fallback for early lessons)

## Repository Structure
The site is split into six Jekyll collections, each its own nav section:
```
index.md                         # landing page / track chooser (nav_order 1, site root)
lab-setup.md                     # environment prep + how-to-study guide (nav_order 2, site root)
_config.yml                      # defines all collections (see Site Theme)
_networking/                     # each track: learning-plan.md + lessons/
_virtualization/                 #   lessons/ = phase-NN-*.md parents + lesson-NN-*.md
_security/
_os/
_leadership/
_english/
```
- `<track>` below means `networking`, `virtualization`, `security`, `os`,
  `leadership`, or `english`.
- A page's URL is `/<track>/lessons/<file>.html` (collection permalink keeps the
  `lessons/` path so relative `(lesson-NN-...)` links between phase and lesson
  pages resolve).
- Lessons for a track live under that track's collection only.

## Teaching Methodology
For each lesson:
1. Create a lesson file at `_<track>/lessons/lesson-NN-topic.md` before teaching
2. The file contains: mental model, mechanics, lab commands with expected output, and checkpoint questions with hidden answers
3. Student reads the file and asks questions in chat if needed
4. Student moves to the next lesson at their own pace — no checkpoint gate
5. At the end of each Phase, give a written test covering all lessons in that phase

## Site Theme
- Theme: **Just the Docs** (dark color scheme) via `remote_theme: just-the-docs/just-the-docs@v0.10.0`
- Config: `_config.yml` — defines all six collections and their nav names under
  `just_the_docs.collections`. Do not change the theme or collection setup without
  updating this file.
- Phase parent pages live in `_<track>/lessons/phase-NN-name.md` with `has_children: true`
- Lesson files live in `_<track>/lessons/lesson-NN-topic.md` with `parent: "Phase N: Name"`
- `parent`/`nav_order`/`has_children` nest pages **within their own collection** — a
  networking lesson's `parent` must match a networking phase page title, and likewise
  for virtualization. The two collections never share parents.

## Lesson File Format
Each lesson file must have this exact front matter:
```yaml
---
title: "Lesson NN — Title"
nav_order: N
parent: "Phase N: Name"
---
```

Then immediately after front matter:
- **Home button**: `[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }` (Just the Docs button style, uses Jekyll relative_url so it works under /learning-garden/ baseurl)
- **Concept** — mental model, analogy, ASCII diagram
- **How it works** — mechanics (brief, focused on "why")
- **Lab** — exact commands with expected output (`$` = run this, `#` = comment)
- **Checkpoint** — each question has:
  1. `**Your answer:**` blank field for the student
  2. A hidden model answer: `<details><summary>Show Answer</summary><br>Answer here.</details>`
- **Homework** — one harder task with its own hidden model answer

Model answers must always be written — never leave a checkpoint or homework without one.

### Lab variants for the non-terminal tracks
The `leadership` and `english` tracks keep the same section names and hidden-answer
format, but their **Lab** is not terminal commands:
- **Leadership** — a realistic scenario ("your PM promised a date you can't hit");
  the student writes their response (`**Your response:**` field), then reveals a
  model answer explaining the reasoning, common mistakes, and useful phrasing.
- **English** — a rewrite drill: broken/blunt source messages, `**Your rewrite:**`
  fields, hidden model rewrites with the reasoning. Every English lesson also ends
  with a **Phrase Bank** table (before Further Reading) of same-day-usable lines.
- Further Reading for these tracks cites books/articles (The Manager's Path,
  StaffEng, LeadDev, style guides) instead of man pages; only link URLs certain
  to exist.

## Handling Student Questions
When the student asks a question about a term or concept from a lesson:
1. Answer briefly in chat
2. Then update the lesson file to incorporate the answer — either as a `{: .note }` callout block inline with the relevant section, or as expanded text
3. Add reference links (Wikipedia, man7.org man pages) for any technical term introduced — inline on first mention AND in the lesson's **Further Reading** table
4. Every lesson must have a **Further Reading** table before the Checkpoint section

### Callout syntax (Just the Docs)
```markdown
{: .note }
> **Title**
> Body text here.
```
Available callout classes are declared in `_config.yml` under `callouts:` —
currently `note`, `warning`, `important`. A class not declared there renders
unstyled; add it to `_config.yml` before using a new one.

### Reference link sources to prefer
- Linux man pages: `https://man7.org/linux/man-pages/manN/name.N.html`
- Wikipedia: `https://en.wikipedia.org/wiki/Topic`
- Only include links you are certain exist — do not guess URLs

## Curriculum
Each track has its own plan; **always consult the relevant one before creating a new lesson** — it specifies the topic, goals, and key commands for every lesson and phase. Never invent a lesson topic; follow the plan in order.
- Networking: `_networking/learning-plan.md`
- Virtualization: `_virtualization/learning-plan.md`
- Security & Identity: `_security/learning-plan.md`
- Operating Systems: `_os/learning-plan.md`
- Engineering Leadership: `_leadership/learning-plan.md`
- English for Work: `_english/learning-plan.md`

## Networking Lesson Index
- Lesson 01: `_networking/lessons/lesson-01-namespaces-intro.md` — What a network namespace is ✓
- Lesson 02: `_networking/lessons/lesson-02-host-as-namespace.md` — The host is just another namespace ✓
- Lesson 03: `_networking/lessons/lesson-03-build-namespaces.md` — Build two isolated namespaces ✓
- Lesson 04: `_networking/lessons/lesson-04-ip-addressing.md` — IP addressing fundamentals (addrs) ✓
- Lesson 05: `_networking/lessons/lesson-05-arp-neigh.md` — Neighbor tables & ARP (neigh) ✓
- Lesson 06: `_networking/lessons/lesson-06-tcpdump.md` — tcpdump, your constant companion ✓
- Lesson 07: `_networking/lessons/lesson-07-veth-pairs.md` — veth pairs ✓
- Lesson 08: `_networking/lessons/lesson-08-loopback.md` — Loopback interface ✓
- Lesson 09: `_networking/lessons/lesson-09-bonding.md` — Bond interfaces ✓
- Lesson 10: `_networking/lessons/lesson-10-dummy.md` — Dummy interfaces ✓
- Lesson 11: `_networking/lessons/lesson-11-bridges.md` — Linux bridges ✓
- Lesson 12: `_networking/lessons/lesson-12-vlans.md` — VLAN interfaces ✓
- Lesson 13: `_networking/lessons/lesson-13-macvlan.md` — MACVLAN ✓
- Lesson 14: `_networking/lessons/lesson-14-tap.md` — TAP interfaces ✓
- Lesson 15: `_networking/lessons/lesson-15-vxlan.md` — VXLAN ✓
- Lesson 16: `_networking/lessons/lesson-16-routing-fundamentals.md` — Routing fundamentals (routes) ✓
- Lesson 17: `_networking/lessons/lesson-17-routing-namespaces.md` — Routing between namespaces ✓
- Lesson 18: `_networking/lessons/lesson-18-policy-routing.md` — Policy routing & multiple tables ✓
- Lesson 19: `_networking/lessons/lesson-19-frr-intro.md` — FRR introduction ✓
- Lesson 20: `_networking/lessons/lesson-20-ospf.md` — OSPF with FRR ✓
- Lesson 21: `_networking/lessons/lesson-21-bgp.md` — BGP with FRR ✓
- Lesson 22: `_networking/lessons/lesson-22-nat.md` — NAT with nftables ✓
- Lesson 23: `_networking/lessons/lesson-23-conntrack.md` — Conntrack ✓
- Lesson 24: `_networking/lessons/lesson-24-dhcp.md` — DHCP ✓
- Lesson 25: `_networking/lessons/lesson-25-nftables-arch.md` — nftables architecture ✓
- Lesson 26: `_networking/lessons/lesson-26-nft-filtering.md` — Packet filtering with nft ✓
- Lesson 27: `_networking/lessons/lesson-27-nft-sets.md` — nftables sets & verdict maps ✓
- Lesson 28: `_networking/lessons/lesson-28-flowtable.md` — nftables flowtable (offload) ✓
- Lesson 29: `_networking/lessons/lesson-29-tc-model.md` — Traffic control model ✓
- Lesson 30: `_networking/lessons/lesson-30-classless-qdisc.md` — Classless qdiscs (shaping/netem) ✓
- Lesson 31: `_networking/lessons/lesson-31-htb.md` — Classful qdiscs — HTB ✓
- Lesson 32: `_networking/lessons/lesson-32-tc-filters.md` — tc filters ✓
- Lesson 33: `_networking/lessons/lesson-33-bandwidth.md` — Bandwidth measurement ✓
- Lesson 34: `_networking/lessons/lesson-34-sysctl-tunables.md` — Network sysctl tunables ✓
- Lesson 35: `_networking/lessons/lesson-35-sysctl-persistence.md` — sysctl persistence ✓
- Lesson 36: `_networking/lessons/lesson-36-ebpf-fundamentals.md` — eBPF fundamentals ✓
- Lesson 37: `_networking/lessons/lesson-37-bpf-maps.md` — BPF maps ✓
- Lesson 38: `_networking/lessons/lesson-38-xdp.md` — XDP, Express Data Path ✓
- Lesson 39: `_networking/lessons/lesson-39-nftbpf.md` — nftbpf: calling BPF from nftables ✓
- Lesson 40: `_networking/lessons/lesson-40-systemd-networkd.md` — systemd-networkd ✓
- Lesson 41: `_networking/lessons/lesson-41-persistence-alternatives.md` — Other persistence approaches ✓
- Lesson 42: `_networking/lessons/lesson-42-docker-networking.md` — Docker networking from first principles ✓
- Lesson 43: `_networking/lessons/lesson-43-kubernetes-networking.md` — Kubernetes networking ✓
- Lesson 44: `_networking/lessons/lesson-44-flannel-vxlan.md` — VXLAN-based CNI (Flannel) ✓
- Lesson 45: `_networking/lessons/lesson-45-tcpdump-mastery.md` — tcpdump mastery ✓
- Lesson 46: `_networking/lessons/lesson-46-diagnostic-toolchain.md` — Full diagnostic toolchain ✓
- Lesson 47: `_networking/lessons/lesson-47-debugging-methodology.md` — Network debugging methodology ✓
- Lesson 48: `_networking/lessons/lesson-48-tunneling-fundamentals.md` — Tunneling fundamentals (GRE/IPIP/SIT/FOU) ✓
- Lesson 49: `_networking/lessons/lesson-49-vpn-crypto.md` — VPN cryptography building blocks ✓
- Lesson 50: `_networking/lessons/lesson-50-ipsec-xfrm.md` — IPsec & the xfrm framework ✓
- Lesson 51: `_networking/lessons/lesson-51-wireguard-fundamentals.md` — WireGuard fundamentals ✓
- Lesson 52: `_networking/lessons/lesson-52-wireguard-internals.md` — WireGuard internals & userspace ✓
- Lesson 53: `_networking/lessons/lesson-53-nat-traversal.md` — NAT traversal (STUN/ICE/hole punching) ✓
- Lesson 54: `_networking/lessons/lesson-54-relays.md` — Relays and fallback paths ✓
- Lesson 55: `_networking/lessons/lesson-55-mesh-vpns.md` — Mesh VPNs & coordination planes ✓
- Lesson 56: `_networking/lessons/lesson-56-tls-vpns.md` — TLS-based VPNs (OpenVPN) ✓
- Lesson 57: `_networking/lessons/lesson-57-vpn-capstone.md` — Capstone: encrypted mesh across NAT ✓
- Lesson 58: `_networking/lessons/lesson-58-bgp-scale.md` — Large-scale BGP (route reflectors) ✓
- Lesson 59: `_networking/lessons/lesson-59-evpn.md` — BGP EVPN & VXLAN fabrics ✓
- Lesson 60: `_networking/lessons/lesson-60-srv6.md` — Segment Routing & SRv6 ✓
- Lesson 61: `_networking/lessons/lesson-61-anycast.md` — Anycast ✓
- Lesson 62: `_networking/lessons/lesson-62-tcp-internals.md` — TCP internals & congestion control ✓
- Lesson 63: `_networking/lessons/lesson-63-nic-offloads.md` — NIC offloads & multiqueue scaling ✓
- Lesson 64: `_networking/lessons/lesson-64-kernel-bypass.md` — Kernel bypass (AF_XDP/DPDK) ✓
- Lesson 65: `_networking/lessons/lesson-65-mptcp.md` — Multipath TCP (MPTCP) ✓
- Lesson 66: `_networking/lessons/lesson-66-dns.md` — DNS deep dive ✓
- Lesson 67: `_networking/lessons/lesson-67-ipvs.md` — Layer-4 load balancing (IPVS/LVS) ✓
- Lesson 68: `_networking/lessons/lesson-68-vrrp-keepalived.md` — High availability (VRRP/keepalived) ✓
- Lesson 69: `_networking/lessons/lesson-69-quic.md` — QUIC & HTTP/3 ✓
- Lesson 70: `_networking/lessons/lesson-70-tls.md` — TLS at the packet level ✓
- Lesson 71: `_networking/lessons/lesson-71-multicast.md` — Multicast & IGMP ✓
- Lesson 72: `_networking/lessons/lesson-72-time-sync.md` — Time synchronization (NTP/PTP) ✓
- Lesson 73: `_networking/lessons/lesson-73-observability.md` — Network observability & telemetry ✓

*(Networking track complete — lessons 01–73 all written. Update this index if lessons change.)*

## Virtualization Lesson Index
Phase parent pages live at `_virtualization/lessons/phase-NN-name.md`.
- Lesson 01: `_virtualization/lessons/lesson-01-what-virtualization-is.md` — What virtualization actually is ✓
- Lesson 02: `_virtualization/lessons/lesson-02-hypervisor-types.md` — Hypervisor types and where KVM fits ✓
- Lesson 03: `_virtualization/lessons/lesson-03-qemu-kvm-division.md` — The QEMU + KVM division of labor ✓
- Lesson 04: `_virtualization/lessons/lesson-04-rings-trap-emulate.md` — Rings, privileged instructions, trap-and-emulate ✓
- Lesson 05: `_virtualization/lessons/lesson-05-vtx-amdv.md` — Hardware virtualization extensions (VT-x/AMD-V) ✓
- Lesson 06: `_virtualization/lessons/lesson-06-ept-npt-slat.md` — Memory virtualization: EPT/NPT (SLAT) ✓
- Lesson 07: `_virtualization/lessons/lesson-07-host-readiness.md` — Checking/enabling virtualization on the host ✓
- Lesson 08: `_virtualization/lessons/lesson-08-dev-kvm-ioctl.md` — The /dev/kvm ioctl API ✓
- Lesson 09: `_virtualization/lessons/lesson-09-kvm-modules-params.md` — KVM kernel modules and parameters ✓
- Lesson 10: `_virtualization/lessons/lesson-10-vm-exit-loop.md` — The VM-exit loop in depth ✓
- Lesson 11: `_virtualization/lessons/lesson-11-observing-kvm.md` — Observing KVM at runtime ✓
- Lesson 12: `_virtualization/lessons/lesson-12-qemu-command-line.md` — Anatomy of a QEMU command line ✓
- Lesson 13: `_virtualization/lessons/lesson-13-machine-types-accel.md` — Machine types and accelerators ✓
- Lesson 14: `_virtualization/lessons/lesson-14-firmware-boot.md` — Firmware and the boot path ✓
- Lesson 15: `_virtualization/lessons/lesson-15-monitor-hmp-qmp.md` — The QEMU monitor: HMP and QMP ✓
- Lesson 16: `_virtualization/lessons/lesson-16-display-console.md` — Display, console, and remote access ✓
- Lesson 17: `_virtualization/lessons/lesson-17-vcpu-models.md` — vCPU models and feature flags ✓
- Lesson 18: `_virtualization/lessons/lesson-18-cpu-topology.md` — CPU topology ✓
- Lesson 19: `_virtualization/lessons/lesson-19-memory-hugepages.md` — Guest memory backends and hugepages ✓
- Lesson 20: `_virtualization/lessons/lesson-20-balloon-ksm.md` — Ballooning and KSM ✓
- Lesson 21: `_virtualization/lessons/lesson-21-numa.md` — NUMA awareness ✓
- Lesson 22: `_virtualization/lessons/lesson-22-image-formats-qemu-img.md` — Disk image formats and qemu-img ✓
- Lesson 23: `_virtualization/lessons/lesson-23-qcow2-internals.md` — qcow2 internals: backing files & CoW ✓
- Lesson 24: `_virtualization/lessons/lesson-24-block-device-models.md` — Block device models ✓
- Lesson 25: `_virtualization/lessons/lesson-25-caching-aio.md` — Caching modes and AIO ✓
- Lesson 26: `_virtualization/lessons/lesson-26-snapshots.md` — Snapshots ✓
- Lesson 27: `_virtualization/lessons/lesson-27-virtio-standard.md` — The virtio standard ✓
- Lesson 28: `_virtualization/lessons/lesson-28-virtio-devices.md` — The core virtio devices ✓
- Lesson 29: `_virtualization/lessons/lesson-29-vhost.md` — vhost: moving the datapath into the kernel ✓
- Lesson 30: `_virtualization/lessons/lesson-30-vhost-user-vsock.md` — vhost-user, vsock, shared-memory datapaths ✓
- Lesson 31: `_virtualization/lessons/lesson-31-slirp.md` — User-mode networking (SLIRP) ✓
- Lesson 32: `_virtualization/lessons/lesson-32-tap-bridge.md` — TAP + bridge networking ✓
- Lesson 33: `_virtualization/lessons/lesson-33-accelerated-net.md` — Accelerated networking: vhost-net & multiqueue ✓
- Lesson 34: `_virtualization/lessons/lesson-34-macvtap-sriov.md` — macvtap and SR-IOV networking ✓
- Lesson 35: `_virtualization/lessons/lesson-35-iommu.md` — The IOMMU ✓
- Lesson 36: `_virtualization/lessons/lesson-36-vfio-pci.md` — VFIO PCI passthrough ✓
- Lesson 37: `_virtualization/lessons/lesson-37-gpu-passthrough.md` — GPU passthrough ✓
- Lesson 38: `_virtualization/lessons/lesson-38-sriov-mdev.md` — SR-IOV and mediated devices ✓
- Lesson 39: `_virtualization/lessons/lesson-39-libvirt-virsh.md` — libvirt architecture and virsh ✓
- Lesson 40: `_virtualization/lessons/lesson-40-domain-xml.md` — The domain XML ✓
- Lesson 41: `_virtualization/lessons/lesson-41-storage-pools.md` — Storage pools and volumes ✓
- Lesson 42: `_virtualization/lessons/lesson-42-libvirt-networks.md` — libvirt virtual networks ✓
- Lesson 43: `_virtualization/lessons/lesson-43-virt-install.md` — virt-install and the desktop tools ✓
- Lesson 44: `_virtualization/lessons/lesson-44-cloud-init.md` — Cloud images and cloud-init ✓
- Lesson 45: `_virtualization/lessons/lesson-45-libguestfs.md` — libguestfs: editing images without booting ✓
- Lesson 46: `_virtualization/lessons/lesson-46-templates-cloning.md` — Templates and cloning ✓
- Lesson 47: `_virtualization/lessons/lesson-47-backup-checkpoints.md` — Domain snapshots, checkpoints, incremental backup ✓
- Lesson 48: `_virtualization/lessons/lesson-48-migration-mechanics.md` — Live migration mechanics ✓
- Lesson 49: `_virtualization/lessons/lesson-49-migration-practice.md` — Live migration in practice ✓
- Lesson 50: `_virtualization/lessons/lesson-50-cpu-pinning.md` — CPU pinning and thread placement ✓
- Lesson 51: `_virtualization/lessons/lesson-51-memory-numa-tuning.md` — Memory and NUMA tuning in production ✓
- Lesson 52: `_virtualization/lessons/lesson-52-storage-io-tuning.md` — Storage and I/O tuning ✓
- Lesson 53: `_virtualization/lessons/lesson-53-observability.md` — Observability: measuring a running VM ✓
- Lesson 54: `_virtualization/lessons/lesson-54-tuned-profiles.md` — tuned and host profiles ✓
- Lesson 55: `_virtualization/lessons/lesson-55-nested-virt.md` — Nested virtualization ✓
- Lesson 56: `_virtualization/lessons/lesson-56-svirt.md` — sVirt: confining QEMU with MAC ✓
- Lesson 57: `_virtualization/lessons/lesson-57-qemu-hardening.md` — QEMU hardening and sandboxing ✓
- Lesson 58: `_virtualization/lessons/lesson-58-confidential-computing.md` — Confidential computing ✓
- Lesson 59: `_virtualization/lessons/lesson-59-microvms.md` — microVMs ✓
- Lesson 60: `_virtualization/lessons/lesson-60-kata-containers.md` — VM-isolated containers (Kata) ✓
- Lesson 61: `_virtualization/lessons/lesson-61-platforms-next.md` — Management platforms & where to go next ✓

*(Virtualization track complete — update this index if lessons change)*

## Security & Identity Lesson Index
Phase parent pages live at `_security/lessons/phase-NN-name.md`. Lesson numbering is
per-track (01–40). Mark each ✓ as its file lands.
- Phase 1 — Cryptography Foundations: lessons 01 symmetric-aead, 02 hashing-macs, 03 asymmetric-dh, 04 signatures, 05 randomness
- Phase 2 — PKI & Certificates: 06 x509, 07 ca-trust, 08 csr-openssl, 09 validation, 10 revocation, 11 ct, 12 private-ca, 13 acme
- Phase 3 — TLS & SSL: 14 tls-history, 15 tls12-handshake, 16 tls13-handshake, 17 cipher-suites, 18 mtls, 19 sni-alpn-resumption, 20 tls-hardening, 21 tls-attacks
- Phase 4 — Authentication Fundamentals: 22 authn-authz, 23 sessions-tokens, 24 passwords, 25 mfa, 26 webauthn, 27 kerberos
- Phase 5 — Federated Identity & Authorization: 28 oauth2, 29 oauth-flows, 30 oidc, 31 jwt, 32 saml, 33 sso, 34 idp-keycloak
- Phase 6 — Applied Security & Identity: 35 workload-identity, 36 secrets, 37 api-auth, 38 zero-trust, 39 token-lifecycle, 40 threat-modeling

File paths follow `_security/lessons/lesson-NN-<slug>.md`. Cross-links to networking:
the security TLS phase ↔ networking Lesson 70 (TLS on the wire); security crypto phase ↔
networking Lesson 49 (VPN crypto primer); security zero-trust ↔ networking Phase 16 (mesh VPNs).

*(Security track complete — lessons 01–40 all written. Update this index if lessons change.)*

## Operating Systems Lesson Index
Phase parent pages live at `_os/lessons/phase-NN-name.md`. File paths follow
`_os/lessons/lesson-NN-<slug>.md`. Mark each ✓ as its file lands.
- Phase 1 — The Kernel Boundary: 01 what-an-os-does ✓, 02 syscalls-strace ✓, 03 libc-abi-vdso ✓, 04 proc-sys ✓, 05 interrupts-timers ✓
- Phase 2 — Processes: 06 process-anatomy ✓, 07 fork-exec ✓, 08 process-lifecycle ✓, 09 signals ✓, 10 sessions-job-control ✓, 11 credentials-capabilities ✓
- Phase 3 — Scheduling: 12 context-switches ✓, 13 cfs-eevdf ✓, 14 priorities-realtime ✓, 15 affinity-psi ✓
- Phase 4 — Memory: 16 virtual-memory ✓, 17 page-tables-tlb ✓, 18 process-memory-layout ✓, 19 page-faults ✓, 20 page-cache ✓, 21 swap-reclaim ✓, 22 oom-overcommit ✓, 23 hugepages-numa ✓
- Phase 5 — Concurrency: 24 threads ✓, 25 race-conditions ✓, 26 mutex-futex ✓, 27 condvars-semaphores ✓, 28 deadlock ✓, 29 atomics-ordering ✓, 30 async-patterns ✓
- Phase 6 — IPC: 31 pipes-fifos ✓, 32 unix-sockets ✓, 33 shared-memory ✓, 34 message-queues ✓, 35 eventfd-signalfd ✓
- Phase 7 — Files & Filesystems: 36 file-descriptors ✓, 37 vfs-inodes ✓, 38 ext4-journaling ✓, 39 filesystem-zoo ✓, 40 mounts-bind ✓, 41 io-fsync ✓, 42 block-layer ✓
- Phase 8 — Event-Driven & Async I/O: 43 epoll ✓, 44 io-uring ✓, 45 zero-copy ✓
- Phase 9 — Linking & Loading: 46 elf-static-linking ✓, 47 dynamic-loader ✓, 48 shared-libraries ✓, 49 ld-preload ✓
- Phase 10 — Boot & Init: 50 boot-process ✓, 51 initramfs ✓, 52 systemd ✓, 53 modules-udev ✓
- Phase 11 — Kernel Interfaces & Security: 54 kernel-module ✓, 55 char-device ✓, 56 seccomp ✓, 57 lsm ✓, 58 build-kernel ✓
- Phase 12 — Containers & Resource Control: 59 namespaces ✓, 60 cgroups ✓, 61 container-images ✓, 62 build-a-container ✓
- Phase 13 — Tracing & Debugging: 63 perf ✓, 64 ftrace-kprobes ✓, 65 core-dumps ✓, 66 debugging-capstone ✓

*(Operating Systems track complete — lessons 01–66 all written. Update this index if lessons change.)*

Cross-links: rings/privilege ↔ virt Lesson 04; page tables ↔ virt Lesson 06 (EPT);
hugepages/NUMA ↔ virt Lessons 19/21; namespaces ↔ networking Lesson 01; seccomp/LSM ↔
virt Lessons 56/57; boot ↔ virt Lesson 14; eBPF tools ↔ networking Phase 11.

## Engineering Leadership Lesson Index
Phase parent pages live at `_leadership/lessons/phase-NN-name.md`. File paths follow
`_leadership/lessons/lesson-NN-<slug>.md`. Labs are scenario exercises (see Lab
variants). Mark each ✓ as its file lands.
- Phase 1 — The Transition: 01 what-changes ✓, 02 lead-em-staff ✓, 03 letting-go-of-code ✓, 04 time-leverage ✓, 05 identity-psychology ✓
- Phase 2 — Technical Leadership: 06 tech-vision ✓, 07 adrs ✓, 08 tradeoffs ✓, 09 design-reviews ✓, 10 technical-standards ✓, 11 tech-debt ✓, 12 how-much-to-code ✓
- Phase 3 — Communication Foundations: 13 audience-first ✓, 14 explaining-tech ✓, 15 effective-meetings ✓, 16 design-docs ✓, 17 presenting ✓, 18 async-communication ✓
- Phase 4 — Feedback & Difficult Conversations: 19 sbi-feedback ✓, 20 praise ✓, 21 receiving-feedback ✓, 22 crucial-conversations ✓, 23 defusing-emotions ✓
- Phase 5 — 1:1s, Coaching & Mentoring: 24 one-on-ones ✓, 25 listening-questions ✓, 26 coaching-vs-mentoring ✓, 27 mentoring-engineers ✓, 28 code-review-teaching ✓, 29 sponsorship ✓
- Phase 6 — Delegation & Growing the Team: 30 delegation-ladder ✓, 31 assigning-for-growth ✓, 32 career-conversations ✓, 33 growing-leads ✓, 34 succession-bus-factor ✓
- Phase 7 — Influence Without Authority: 35 sources-of-influence, 36 building-buy-in, 37 resolving-conflict, 38 aligning-teams, 39 driving-change
- Phase 8 — Stakeholder Management: 40 pms-designers, 41 managing-up, 42 customers, 43 cross-team-dependencies, 44 negotiation-expectations
- Phase 9 — Business & Product Thinking: 45 business-model, 46 product-strategy, 47 metrics-kpis, 48 tech-to-business, 49 should-we-build-this
- Phase 10 — Project Leadership: 50 planning-roadmaps, 51 estimation, 52 risk-management, 53 dependency-management, 54 prioritization, 55 when-projects-slip, 56 incidents-postmortems
- Phase 11 — People Management (EM path): 57 performance-management, 58 underperformance, 59 hiring-interviewing, 60 evaluating-closing, 61 motivation, 62 org-design, 63 psychological-safety
- Phase 12 — Your Path: 64 first-90-days-lead, 65 first-90-days-em, 66 support-system, 67 choosing-your-path

## English for Work Lesson Index
Phase parent pages live at `_english/lessons/phase-NN-name.md`. File paths follow
`_english/lessons/lesson-NN-<slug>.md`. Labs are rewrite drills + phrase banks (see
Lab variants). Mark each ✓ as its file lands.
- Phase 1 — Sentence Mechanics: 01 sentence-core ✓, 02 articles-a-the ✓, 03 articles-zero ✓, 04 verb-tenses ✓, 05 subject-verb-agreement ✓, 06 prepositions ✓, 07 connecting-ideas ✓, 08 punctuation-chat ✓, 09 error-clinic ✓
- Phase 2 — Words That Sound Natural: 10 do-make-confusions ✓, 11 phrasal-verbs ✓, 12 translated-sentences ✓, 13 concise-words ✓
- Phase 3 — Tone & Warmth: 14 register ✓, 15 softeners-hedging ✓, 16 openers-closers ✓, 17 requests ✓, 18 disagreeing-no ✓, 19 apologies-thanks ✓, 20 sounding-engaged ✓
- Phase 4 — Slack Communication: 21 message-shapes ✓, 22 status-updates ✓, 23 asking-for-help ✓, 24 answering-unblocking ✓, 25 async-etiquette ✓, 26 announcements ✓
- Phase 5 — Meetings (Speaking): 27 agendas-opening ✓, 28 facilitation ✓, 29 interrupting-clarifying ✓, 30 disagreeing-live ✓, 31 summarizing-actions ✓, 32 presenting-work ✓, 33 small-talk ✓
- Phase 6 — Spoken Fluency: 34 thinking-time ✓, 35 paraphrasing ✓, 36 asking-repeat ✓, 37 contractions-rhythm ✓
- Phase 7 — Design Docs & Proposals: 38 doc-structure ✓, 39 plain-language ✓, 40 paragraphs-flow ✓, 41 exec-summaries ✓, 42 persuasive-proposals ✓, 43 doc-comments ✓
- Phase 8 — Feedback & Hard Conversations: 44 feedback-language ✓, 45 code-review-comments ✓, 46 performance-conversations ✓, 47 bad-news ✓, 48 de-escalating ✓
- Phase 9 — Capstone & Habits: 49 error-checklist-v2 ✓, 50 rewrite-week ✓, 51 daily-habits ✓

*(English for Work track complete — lessons 01–51 all written. Update this index if lessons change.)*

English-track note: examples must be software-workplace ones (Slack, PRs, standups),
never textbook sentences. Target the student's known error patterns (subject–verb
agreement, articles, dropped words) in drills across all phases, not just Phase 1.
