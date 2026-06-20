---
title: "Phase 11: Guest Images & Provisioning"
nav_order: 13
has_children: true
---

# Phase 11: Guest Images & Provisioning

Installing an OS by hand from an ISO is fine once; doing it fifty times is madness. This
phase is about producing ready-to-run guests at scale: **cloud images** + **cloud-init**
(boot a pre-built OS that configures itself), **libguestfs** (edit disk images offline,
without booting them), and **templates + cloning** (mass-produce VMs from a golden
image).

| # | Lesson | Status |
|---|--------|--------|
| 44 | [Cloud images and cloud-init](lesson-44-cloud-init) | In progress |
| 45 | [libguestfs — editing images without booting them](lesson-45-libguestfs) | In progress |
| 46 | [Templates and cloning](lesson-46-templates-cloning) | In progress |
