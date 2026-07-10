---
title: "Phase 8: Event-Driven & Async I/O"
nav_order: 10
has_children: true
---

# Phase 8: Event-Driven & Async I/O

Phase 5 ended with event loops as a pattern; Phase 6 gave every waitable thing
an fd. This phase builds the engine: epoll and non-blocking I/O (the mechanism
behind every modern server), io_uring (the completion-based successor that
finally covers disk files and batches syscalls away), and zero-copy
(sendfile/splice — deleting the copies you didn't know you were paying for).

| # | Lesson | Status |
|---|--------|--------|
| 43 | [epoll and non-blocking I/O](lesson-43-epoll) | Ready |
| 44 | [io_uring](lesson-44-io-uring) | Ready |
| 45 | [Zero-copy — sendfile and splice](lesson-45-zero-copy) | Ready |
