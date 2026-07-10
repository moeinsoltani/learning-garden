---
title: "Phase 9: Linking & Loading"
nav_order: 11
has_children: true
---

# Phase 9: Linking & Loading

What is a binary, actually — and what happens between `execve` and `main()`?
This phase opens the box: the ELF format and what the static linker does,
the dynamic loader that assembles your process from shared libraries at
runtime, how libraries are built and versioned without breaking the world,
and LD_PRELOAD — the interposition superpower that lets you hook any library
call in any program.

| # | Lesson | Status |
|---|--------|--------|
| 46 | [ELF and static linking](lesson-46-elf-static-linking) | Ready |
| 47 | [The dynamic loader](lesson-47-dynamic-loader) | Ready |
| 48 | [Shared libraries and versioning](lesson-48-shared-libraries) | Ready |
| 49 | [LD_PRELOAD and symbol interposition](lesson-49-ld-preload) | Ready |
