---
title: "Phase 12: nftables + BPF"
nav_order: 14
has_children: true
---

# Phase 12: nftables + BPF Integration

nftables and BPF are both packet-processing engines, and they can work together: nftables can
invoke a BPF program as a match expression, letting you keep declarative firewall policy while
delegating logic that rules can't express (deep payload matching, custom classification) to BPF.

| # | Lesson | Status |
|---|--------|--------|
| 39 | [nftbpf — calling BPF from nftables](lesson-39-nftbpf) | Ready |
