---
title: "Phase 7: Files & Filesystems"
nav_order: 9
has_children: true
---

# Phase 7: Files & Filesystems

"Everything is a file" finally gets its full treatment. This phase runs the
whole storage stack top to bottom: the fd table and how redirection really
works, the VFS abstraction and inodes, ext4's on-disk reality and journaling,
the other filesystems you'll actually meet (xfs, btrfs, tmpfs, overlayfs),
mounts and bind mounts (the container filesystem primitive), when your data is
*actually* on disk (fsync and friends), and the block layer underneath it all.

| # | Lesson | Status |
|---|--------|--------|
| 36 | [File descriptors, dup, and redirection](lesson-36-file-descriptors) | Ready |
| 37 | [VFS and inodes](lesson-37-vfs-inodes) | Ready |
| 38 | [ext4 and journaling](lesson-38-ext4-journaling) | Ready |
| 39 | [The filesystem zoo](lesson-39-filesystem-zoo) | Ready |
| 40 | [Mounts and bind mounts](lesson-40-mounts-bind) | Ready |
| 41 | [Buffered vs direct I/O, and fsync](lesson-41-io-fsync) | Ready |
| 42 | [The block layer and I/O schedulers](lesson-42-block-layer) | Ready |
