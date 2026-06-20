---
title: "Lesson 34 — macvtap and SR-IOV Networking"
nav_order: 34
parent: "Phase 8: Networking for VMs"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 34: macvtap and SR-IOV Networking

## Concept

TAP + bridge (Lesson 32) is the flexible default, but it adds a software bridge in the
path. Two lower-overhead alternatives put guests closer to the wire:

```
   macvtap                              SR-IOV
   ───────                              ──────
   guest ─► macvtap ─► physical NIC      guest ─► NIC Virtual Function (VF)
   (no bridge; NIC carries multiple      (the physical NIC hardware presents
    guest MACs directly)                  multiple "mini-NICs"; near bare-metal)
```

**macvtap** is "bridgeless bridging" — it builds on **MACVLAN** (Networking
Lesson 13), giving each guest its own MAC straight on the physical NIC, skipping the
software bridge. **SR-IOV** goes further: the *NIC hardware* splits itself into virtual
functions that get assigned to guests almost like passthrough.

---

## How It Works

### macvtap (recall MACVLAN, Networking L13)

macvtap attaches a guest directly to a physical NIC, with the NIC carrying multiple MAC
addresses (one per guest) — no separate Linux bridge device. It's lower overhead and
simpler to set up than a bridge for the common "put guests on the LAN" case.

Modes (inherited from MACVLAN):

- **bridge** — guests on the same macvtap parent can talk to each other (the NIC/macvtap
  switches between them). Most common.
- **vepa** — frames go *out* to the external switch and back (for switch-based policy);
  guests can't talk directly unless the switch hairpins.
- **private** — guests are isolated from each other entirely.

**The classic gotcha:** in macvtap (bridge mode), the **host itself usually cannot
reach the guests** through the same physical NIC. This is inherited from MACVLAN: the
parent interface and its macvlan/macvtap children are deliberately isolated from each
other for security/loop reasons — traffic from the host's own stack to a macvtap child
on the same NIC is not switched locally. Guests can reach the LAN and each other (bridge
mode) and the LAN can reach guests, but **host↔guest** on that NIC is blocked.

### SR-IOV (preview of full passthrough, Phase 9)

**SR-IOV** (Single Root I/O Virtualization) is a hardware feature of capable NICs. One
physical NIC exposes:

- One **Physical Function (PF)** — the "real" full-featured NIC the host manages.
- Many **Virtual Functions (VFs)** — lightweight hardware NICs the card creates, each
  with its own queues and MAC.

You assign a **VF directly to a guest** (via VFIO, Phase 9), and the guest drives that
VF as if it owned hardware — the datapath bypasses the host software entirely, giving
**near bare-metal performance** and low CPU. The NIC's internal switch (embedded
switch) moves frames between VFs and the wire.

### Trade-offs across the three approaches

| Approach | Overhead | Setup | Host↔guest | Live migration |
|---|---|---|---|---|
| **TAP + bridge** | software bridge | most flexible | works | yes |
| **macvtap** | lower (no bridge) | simple | **often blocked** | yes |
| **SR-IOV VF** | near zero (HW) | needs SR-IOV NIC + VFIO | via PF | hard (VF is HW-bound) |

SR-IOV's catch: because the guest is bound to specific NIC hardware, **live migration is
hard** (the destination must have an equivalent free VF, and state in the VF can't be
trivially moved) — the same trade-off all passthrough makes (Phase 9). macvtap keeps
migration but loses host↔guest on the NIC. TAP+bridge keeps everything but pays the
software-switch cost.

{: .note }
> **Why the host can't reach a macvtap guest — and what to do**
> macvtap/MACVLAN deliberately isolates the parent interface from its children: the
> host's own IP on the physical NIC and the guests' macvtap endpoints don't switch to
> each other locally. So management traffic from the host to the guest fails on that
> NIC. Workarounds: add a separate macvlan interface for the host on the same NIC (the
> standard MACVLAN host-access trick from Networking L13), manage the guest over a
> different network/NIC, use vsock (Lesson 30) for host↔guest control, or just use a
> bridge instead when host↔guest reachability matters.

---

## Lab

```bash
# ---- macvtap: attach a guest directly to the physical NIC, no bridge ----
# 1. Create a macvtap device on top of eth0 in bridge mode:
$ sudo ip link add link eth0 name macvtap0 type macvtap mode bridge
$ sudo ip link set macvtap0 up
# Find its tap char device (needed by QEMU as an fd):
$ ip link show macvtap0
12: macvtap0@eth0: ... link/ether 5a:...    # note the ifindex (12)
$ ls -l /dev/tap12        # the char device for macvtap with ifindex 12

# 2. Boot QEMU using the macvtap fd (open the /dev/tapN and pass it):
$ sudo qemu-system-x86_64 -accel kvm -m 2G -smp 2 \
    -netdev tap,id=n0,fd=3,vhost=on 3<>/dev/tap12 \
    -device virtio-net-pci,netdev=n0,mac=5a:... \
    -drive file=disk.qcow2,if=virtio -nographic &
#   (guest)$ ip addr show eth0   # gets a real LAN IP, like bridge mode
#   from another LAN host: ping the guest -> works
#   from THIS host: ping the guest -> typically FAILS (the macvtap gotcha)

# ---- SR-IOV: create VFs on a capable NIC, then assign one to a guest ----
# 3. How many VFs can this NIC make?
$ cat /sys/class/net/eth0/device/sriov_totalvfs
64
# 4. Create 4 VFs:
$ echo 4 | sudo tee /sys/class/net/eth0/device/sriov_numvfs
$ ip link show eth0
eth0: ... vf 0 MAC ..., vf 1 MAC ..., vf 2 ..., vf 3 ...   # VFs appear
$ lspci | grep -i 'Virtual Function'
01:10.0 Ethernet ... Virtual Function
...
# 5. Assigning a VF to a guest = VFIO passthrough (full treatment in Phase 9):
#    -device vfio-pci,host=0000:01:10.0
$ sudo kill %1 2>/dev/null
```

**Expected result:** A macvtap guest gets a real LAN IP and is reachable from other LAN
machines, but **not** from the host itself over that NIC (the MACVLAN isolation gotcha).
On an SR-IOV NIC, writing `sriov_numvfs` makes Virtual Functions appear as separate PCI
devices ready to be assigned to guests.

---

## Further Reading

| Topic | Link |
|---|---|
| Networking Lesson 13 (MACVLAN) | [MACVLAN]({{ '/networking/lessons/lesson-13-macvlan.html' | relative_url }}) |
| macvtap | [Wikipedia — macvtap](https://en.wikipedia.org/wiki/Macvtap) |
| Single-root IOV (SR-IOV) | [Wikipedia — Single-root input/output virtualization](https://en.wikipedia.org/wiki/Single-root_input/output_virtualization) |
| Linux SR-IOV docs | [kernel.org — SR-IOV](https://docs.kernel.org/PCI/pci-iov-howto.html) |
| VFIO (Phase 9 preview) | [kernel.org — VFIO](https://docs.kernel.org/driver-api/vfio.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why can the host often *not* reach a guest that's attached via macvtap, and what does that imply for management traffic?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
macvtap is built on MACVLAN, which deliberately isolates the parent physical interface from its macvlan/macvtap children: the host's own IP on the NIC and the guests' macvtap endpoints are not switched to each other locally. So traffic from the host's stack to a macvtap guest on the same NIC is dropped, even though the LAN and the guests can reach each other. The implication: you can't manage the guest from the host over that NIC — you need a workaround (add a separate macvlan interface for the host, manage over a different network/NIC, or use vsock), or use a bridge instead when host↔guest reachability matters.
</details>

---

**Q2. In SR-IOV, what is the difference between a PF and a VF, and how does a guest using a VF get near bare-metal performance?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The Physical Function (PF) is the full-featured "real" NIC the host manages and which controls the card; Virtual Functions (VFs) are lightweight hardware NIC instances the card creates, each with its own queues and MAC. A guest assigned a VF (via VFIO passthrough) drives that hardware directly, so the packet datapath bypasses the host's software networking entirely — the NIC's embedded switch moves frames between VFs and the wire. With no host bridge/vhost in the path and direct DMA to/from the guest, it achieves near bare-metal throughput and very low CPU overhead.
</details>

---

**Q3. SR-IOV gives great performance but complicates one major operation. Which, and why?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Live migration. Because the guest is bound directly to specific NIC hardware (a VF), its device state lives in that physical card and can't be trivially serialized and moved. The destination host must have an equivalent NIC with a free VF, and the in-flight hardware state isn't portable the way an emulated/virtio device's state is. This is the same trade-off all device passthrough makes (Phase 9): near bare-metal performance at the cost of easy migration (and host independence). macvtap and TAP+bridge, being software, migrate cleanly.
</details>

---

## Homework

If your NIC supports SR-IOV, set `sriov_numvfs` to 2 and confirm the VFs appear in `lspci` and as `vf 0`/`vf 1` under `ip link show eth0`. (If not, do the macvtap part instead.) Then write down, for your environment, which of TAP+bridge, macvtap, or SR-IOV you'd choose for: (a) a VM that must be live-migratable and managed from the host, (b) a latency-critical packet-processing VM that never migrates. Justify each.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) Live-migratable + host-managed → <strong>TAP + bridge</strong>. It's all software, so it migrates cleanly, and host↔guest works (unlike macvtap), at the cost of the software-bridge overhead — acceptable for a general-purpose, manageable, mobile VM. (b) Latency-critical, never migrates → <strong>SR-IOV VF</strong>. Assigning a VF gives near bare-metal throughput and minimal CPU/latency by bypassing host software, and since the VM won't migrate, SR-IOV's poor migration story doesn't matter. (macvtap sits between: lower overhead than a bridge and migratable, but the host-can't-reach-guest gotcha makes it a poorer fit when host management is required.)
</details>
