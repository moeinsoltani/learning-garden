---
title: "Phase 9: Device Passthrough"
nav_order: 11
has_children: true
---

# Phase 9: Device Passthrough (VFIO & IOMMU)

Sometimes virtio isn't enough — you want a guest to drive *real* hardware: a GPU, a
NIC, an NVMe drive. **Passthrough** assigns a physical PCI device directly to a guest.
This phase covers the hardware that makes it *safe* (the IOMMU), the driver that makes
it *work* (VFIO), the hardest popular case (GPU passthrough), and the sharing variants
(SR-IOV VFs, mediated devices).

| # | Lesson | Status |
|---|--------|--------|
| 35 | [The IOMMU](lesson-35-iommu) | In progress |
| 36 | [VFIO PCI passthrough](lesson-36-vfio-pci) | In progress |
| 37 | [GPU passthrough](lesson-37-gpu-passthrough) | In progress |
| 38 | [SR-IOV and mediated devices](lesson-38-sriov-mdev) | In progress |
