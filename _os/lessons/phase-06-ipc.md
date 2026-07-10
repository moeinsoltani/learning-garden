---
title: "Phase 6: Inter-Process Communication"
nav_order: 8
has_children: true
---

# Phase 6: Inter-Process Communication

Threads share everything; processes share nothing — so processes that must
cooperate need explicit channels. This phase covers the working set of Unix
IPC: pipes (the mechanism behind every `|`), Unix domain sockets (Docker's,
systemd's, and X11's transport — including passing open file descriptors
between processes), shared memory (the fastest channel, which reunites you
with Phase 5's locks), message queues and D-Bus, and the modern
"everything-is-an-fd" event primitives (eventfd, signalfd, timerfd) that feed
Phase 8's event loops.

| # | Lesson | Status |
|---|--------|--------|
| 31 | [Pipes and FIFOs](lesson-31-pipes-fifos) | Ready |
| 32 | [Unix domain sockets](lesson-32-unix-sockets) | Ready |
| 33 | [Shared memory](lesson-33-shared-memory) | Ready |
| 34 | [Message queues and D-Bus](lesson-34-message-queues) | Ready |
| 35 | [eventfd, signalfd, timerfd](lesson-35-eventfd-signalfd) | Ready |
