---
title: "Lesson 35 — The IOMMU"
nav_order: 35
parent: "Phase 9: Device Passthrough"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 35: The IOMMU

## Concept

Passing a real device to a guest is dangerous without help. Devices do **DMA** —
they read and write memory addresses *directly*, bypassing the CPU. A guest tells "its"
device to DMA to *guest-physical* addresses — but those mean nothing to real RAM, and a
malicious or buggy guest could point the device at *any* host memory. The **IOMMU**
solves this: it's "EPT for devices," translating and confining device DMA.

```
   WITHOUT IOMMU                         WITH IOMMU (VT-d / AMD-Vi)
   ─────────────                         ─────────────────────────
   device DMA addr ─────► real RAM        device DMA addr ─► [IOMMU] ─► real RAM
   (a passed-through device could          translates guest-physical → host-physical
    write ANYWHERE in host memory)         AND blocks addresses outside the guest's
                                           allowed regions  (DMA remapping)
```

The IOMMU sits between devices and memory, just as the MMU/EPT sits between the CPU and
memory. It makes passthrough *safe* by ensuring a device can only touch the memory its
assigned guest owns.

---

## How It Works

### DMA remapping — "EPT for devices"

Recall EPT/NPT (Lesson 6) translate *CPU* accesses from guest-physical to host-physical.
The **IOMMU** (Intel **VT-d**, AMD **AMD-Vi**) does the analogous job for *device DMA*:
when a passed-through device issues a DMA to a guest-physical address, the IOMMU
translates it to the correct host-physical page **and** rejects any access outside the
guest's permitted memory. Without this, passthrough would let a device DMA anywhere —
a total breach of host integrity. So the IOMMU is *mandatory* for safe passthrough.

### IOMMU groups — the unit of assignment

The IOMMU doesn't isolate every device individually; it isolates **groups**. An
**IOMMU group** is the smallest set of devices that the hardware can isolate from each
other — determined by the PCIe topology (how devices sit behind bridges/switches and
whether they support **ACS**, Access Control Services, which enforces isolation between
peers).

The critical rule: **you must assign an entire IOMMU group to a guest, not a single
device within it.** If two devices share a group, the IOMMU can't prevent them from
DMAing to each other, so handing one to a guest while the host keeps the other would
break isolation. Everything in the group goes to the guest (or to `vfio-pci`), or none
of it does.

```
   /sys/kernel/iommu_groups/
     ├── 12/devices/0000:01:00.0   (GPU)
     │            0000:01:00.1   (GPU's HDMI audio)   ← same group: both must go
     └── 14/devices/0000:03:00.0   (NIC)              ← alone: easy to assign
```

### Enabling the IOMMU

It's off unless enabled in *both* firmware and kernel:

1. **BIOS/UEFI:** enable **VT-d** (Intel) / **AMD-Vi / IOMMU** (Lesson 7).
2. **Kernel cmdline:** add `intel_iommu=on` (Intel) or `amd_iommu=on` (AMD), often with
   `iommu=pt` (passthrough mode for devices the host keeps, reducing overhead).

Then groups appear under `/sys/kernel/iommu_groups/`.

{: .note }
> **Why an entire group, not one device**
> The IOMMU group is the hardware's isolation boundary — the finest granularity at which
> it can guarantee one set of devices can't DMA into another's (or the host's) memory.
> If a group contains, say, a GPU and its audio function, the hardware can't isolate
> those two from each other; giving the GPU to a guest while the host keeps the audio
> would leave a DMA path between guest-controlled and host-controlled devices, defeating
> the protection. So VFIO requires the *whole* group be assigned (or bound to vfio-pci).
> Poor ACS support can lump unrelated devices together, which is the usual passthrough
> headache.

---

## Lab

```bash
# 1. Is the IOMMU enabled? (after BIOS VT-d/AMD-Vi + kernel cmdline)
$ dmesg | grep -iE 'DMAR|IOMMU|AMD-Vi' | head
[    0.000000] DMAR: IOMMU enabled
[    0.300000] DMAR: Intel(R) Virtualization Technology for Directed I/O

# If empty, add to kernel cmdline and reboot:
#   /etc/default/grub: GRUB_CMDLINE_LINUX_DEFAULT="... intel_iommu=on iommu=pt"
#   sudo update-grub && reboot     (use amd_iommu=on on AMD)

# 2. Enumerate IOMMU groups and what's in each:
$ for g in /sys/kernel/iommu_groups/*/devices/*; do
    grp=$(echo "$g" | awk -F/ '{print $5}')
    dev=$(basename "$g")
    printf 'group %s -> %s  %s\n' "$grp" "$dev" "$(lspci -nns "$dev" | cut -d' ' -f2-)"
  done | sort -V | head -40
group 0 -> 0000:00:00.0  Host bridge ...
group 12 -> 0000:01:00.0  VGA compatible controller [NVIDIA ...]
group 12 -> 0000:01:00.1  Audio device [NVIDIA HDMI Audio]      ← grouped WITH the GPU
group 14 -> 0000:03:00.0  Ethernet controller [Intel I210]      ← alone

# 3. Helper: print a specific device's group and ALL its group-mates:
$ dev=0000:01:00.0
$ grp=$(basename $(dirname $(dirname $(readlink -f /sys/bus/pci/devices/$dev/iommu_group/))) 2>/dev/null \
       || ls -l /sys/bus/pci/devices/$dev/iommu_group | grep -o '[0-9]*$')
$ ls /sys/bus/pci/devices/$dev/iommu_group/devices/
0000:01:00.0  0000:01:00.1     ← to pass the GPU you must also pass its audio fn

# 4. Confirm ACS / isolation quality by inspecting group sizes (smaller = better):
$ for d in /sys/kernel/iommu_groups/*/ ; do
    echo "group $(basename $d): $(ls $d/devices | wc -l) device(s)"
  done | sort -V | head
```

**Expected result:** `dmesg` confirms the IOMMU is enabled; `/sys/kernel/iommu_groups/`
lists groups, and you can see which devices share a group (e.g. a GPU with its HDMI
audio function) — exactly the set you must assign together.

---

## Further Reading

| Topic | Link |
|---|---|
| IOMMU | [Wikipedia — Input–output memory management unit](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit) |
| VFIO & IOMMU groups | [kernel.org — VFIO](https://docs.kernel.org/driver-api/vfio.html) |
| Direct memory access (DMA) | [Wikipedia — Direct memory access](https://en.wikipedia.org/wiki/Direct_memory_access) |
| PCIe Access Control Services (ACS) | [Wikipedia — PCI Express](https://en.wikipedia.org/wiki/PCI_Express) |
| Intel VT-d | [Wikipedia — x86 virtualization (VT-d)](https://en.wikipedia.org/wiki/X86_virtualization#I/O_MMU_virtualization_(AMD-Vi_and_VT-d)) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why must you pass through an *entire* IOMMU group, not just one device in it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The IOMMU group is the hardware's finest isolation boundary — the smallest set of devices it can guarantee won't DMA into each other's (or the host's) memory, determined by PCIe topology and ACS support. If two devices share a group, the IOMMU can't isolate them from one another, so assigning one to a guest while the host keeps the other would leave a DMA path between guest-controlled and host-controlled hardware, breaking the protection. Therefore the whole group must be assigned to the guest (or bound to vfio-pci); you can't safely split a group.
</details>

---

**Q2. Why is passthrough unsafe without an IOMMU?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Devices perform DMA, reading/writing memory addresses directly without CPU mediation. A guest programs "its" passed-through device with guest-physical addresses; without an IOMMU to translate and confine those accesses, the device would issue raw physical addresses that could point anywhere in host RAM. A malicious or buggy guest could thus make the device DMA over host kernel memory or other VMs' memory — a complete compromise of host integrity and isolation. The IOMMU prevents this by translating device DMA (guest-physical → host-physical) and blocking any access outside the guest's allowed regions.
</details>

---

**Q3. What is the IOMMU's relationship to EPT/NPT from Lesson 6?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
They're analogous mechanisms for different initiators. EPT/NPT (SLAT) translate and confine *CPU* memory accesses from guest-physical to host-physical. The IOMMU does the same job for *device DMA*: it translates a device's guest-physical DMA addresses to host-physical and restricts them to the guest's allowed memory. So the IOMMU is "EPT for devices" — both ensure that whatever issues memory accesses (CPU via EPT, devices via IOMMU) can only reach the host memory belonging to the right guest.
</details>

---

## Homework

Enable the IOMMU on your host (BIOS VT-d/AMD-Vi + `intel_iommu=on`/`amd_iommu=on` kernel cmdline) and confirm with `dmesg | grep -i IOMMU`. Then pick the device you'd most like to pass through (e.g. a spare NIC) and list every device in its IOMMU group. Explain whether it's "cleanly isolatable" (alone in its group) or whether you'd have to assign companions too.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
After enabling, dmesg shows "DMAR: IOMMU enabled" (or AMD-Vi). Listing <code>/sys/bus/pci/devices/&lt;addr&gt;/iommu_group/devices/</code> for your chosen device reveals its group-mates. If it's alone in its group, it's cleanly isolatable — you can bind just that device to vfio-pci and assign it. If the group contains other devices (e.g. a GPU sharing a group with its HDMI audio function, or several devices behind a non-ACS bridge), you must assign all of them together, because the IOMMU can't isolate within a group; you can't keep some on the host while giving others to the guest. Groups with multiple unrelated devices usually indicate poor ACS support and are the typical passthrough obstacle.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 36 — VFIO PCI Passthrough →](lesson-36-vfio-pci){: .btn .btn-primary }
