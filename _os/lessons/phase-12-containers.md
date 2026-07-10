---
title: "Phase 12: Containers & Resource Control"
nav_order: 14
has_children: true
---

# Phase 12: Containers & Resource Control

Everything converges here. A container is not a thing the kernel provides —
it's a *combination* of primitives you've met across the whole course:
namespaces (isolation), cgroups (resource limits), overlayfs (the image
filesystem), pivot_root, capabilities, and seccomp. This phase generalizes
networking Lesson 01's namespace idea to all namespace types, meters resources
with cgroups v2, unpacks what a container image really is, and — the capstone
— builds a working container by hand with no Docker.

| # | Lesson | Status |
|---|--------|--------|
| 59 | [Namespaces — all of them](lesson-59-namespaces) | Ready |
| 60 | [cgroups v2](lesson-60-cgroups) | Ready |
| 61 | [Container images and overlayfs](lesson-61-container-images) | Ready |
| 62 | [Build a container by hand — capstone](lesson-62-build-a-container) | Ready |
