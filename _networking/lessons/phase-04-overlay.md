---
title: "Phase 4: Overlay Networks"
nav_order: 6
has_children: true
---

# Phase 4: Overlay Networks

Everything so far stayed on one machine or one L2 segment. Real data centers and Kubernetes
clusters span many hosts across a routed (L3) network. Overlay networks stretch a virtual L2
network *over* that L3 fabric by tunnelling. VXLAN is the workhorse, and this phase builds one
by hand.

| # | Lesson | Status |
|---|--------|--------|
| 15 | [VXLAN](lesson-15-vxlan) | Ready |
