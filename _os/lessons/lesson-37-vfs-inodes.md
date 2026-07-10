---
title: "Lesson 37 — VFS and Inodes"
nav_order: 2
parent: "Phase 7: Files & Filesystems"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 37: VFS and Inodes

## Concept

One `read()` syscall works identically on an ext4 file, a /proc entry, a
tmpfs scratch file, an NFS mount, and /dev/urandom. Five wildly different
implementations, one interface — because between the syscall layer and every
filesystem sits the **VFS** (Virtual File System): the kernel's
filesystem-shaped interface that concrete filesystems implement.

```
        open/read/write/stat…  (one API)
                 │
        ┌────────▼────────┐
        │       VFS       │   objects: superblock, inode, dentry, file
        └──┬──┬──┬──┬──┬──┘   (each with a table of function pointers)
           │  │  │  │  │
        ext4 xfs tmpfs proc nfs …   ← each fills in the pointers:
                                      "here's MY read, MY lookup…"
```

It's mechanism/policy again (Lesson 01), and it's why /proc could pretend to
be files (Lesson 04), why tmpfs is a filesystem with no disk, why FUSE can
put a filesystem in userspace.

The VFS's central object — and this lesson's real subject — is the
**inode**: the file's true identity. The revelation that reorganizes your
mental model of every filesystem:

**A filename is not a property of a file.** The inode (numbered, per-fs)
holds everything — size, permissions, owner, timestamps, where the data
blocks live — *except* a name. Names live in **directories**, which are just
tables mapping `name → inode number`. A file can have many names (hard
links), or, transiently, none at all (open but deleted — the lab's finale and
Lesson 36's mystery, now fully explained):

```
  directory "/var/log":            inode 8812:
  ┌────────────────────┐          ┌──────────────────────┐
  │ syslog     → 8812  │ ───────▶ │ size, mode, owner,   │
  │ syslog.bak → 8812  │ ───────▶ │ mtime, link count: 2,│──▶ data blocks
  └────────────────────┘          │ NO NAME ANYWHERE     │
   two names, one file             └──────────────────────┘
```

---

## How It Works

### stat: reading an inode

`stat file` shows the inode's contents: inode number, link count, type,
mode, owner, size, the three timestamps (atime access / mtime data modified /
ctime *inode* changed — chmod bumps ctime, not mtime!). `ls -i` shows
numbers; two names with one number = hard links, provably the same file.

### Hard links vs symlinks — mechanically

**Hard link** (`ln a b`): a second directory entry for the same inode; link
count +1. Fully symmetric — no "original": removing either name just
decrements the count. Constraints follow from the mechanism: same filesystem
only (inode numbers are per-fs), and no directories (cycles would break the
tree). **Symlink** (`ln -s a b`): a *new* inode of type symlink whose data is
a *pathname string*, resolved at every access. Cross-fs, dangling-capable
(the target name may not exist), and visible as `l` in ls. Rule of thumb:
hard links are aliasing (identity); symlinks are redirection (by name).

### Deletion, properly named

`rm` is `unlink()`: remove a directory entry, decrement the link count. Data
is freed only when **link count = 0 AND no process holds it open** (open fds
count as references — Lesson 36's OFD chain ends at the inode). Hence: log
rotation must signal the daemon (the old inode lives while held open —
`lsof +L1` finds these), self-deleting temp files (`open` then immediately
`unlink` — the file exists only as your fd: no name, crash-proof cleanup,
`O_TMPFILE` formalizes it), and atomic replace: `rename()` atomically points
a name at a new inode — readers see old file or new file, never a
half-written one. Every "safe config update" is `write tmp; fsync; rename`.

### dentries: the name cache

Pathname→inode resolution walks every component (`/var/log/syslog`: look up
`var` in `/`, `log` in that, …) — so the kernel caches the walk in the
**dentry cache** (including *negative* entries: "no such file" cached too —
why repeated failed lookups are cheap). It's the `slabtop` heavyweight and
part of Lesson 20's droppable memory.

{: .note }
> **inode exhaustion: full disk with free space**
> Filesystems allocate a finite inode table at mkfs (ext4 default: one per
> 16 KB). Millions of tiny files (mail spools, cache dirs, node_modules…)
> can consume every inode while blocks remain free: writes fail "No space
> left on device" but <code>df -h</code> looks fine. The reveal:
> <code>df -i</code> (IUse%). The fix: delete files or rebuild with more
> inodes — you can't add them to a live ext4.

---

## Lab

```bash
# ---- 1. Meet an inode ----
$ echo "hello inode" > /tmp/f1
$ stat /tmp/f1
#   Inode: 1837  Links: 1  Size: 12 ...
#   Access/Modify/Change: three timestamps, three meanings
$ chmod 600 /tmp/f1 && stat -c 'mtime=%y ctime=%z' /tmp/f1
# mtime unchanged, ctime bumped — chmod changed the INODE, not the data

# ---- 2. Hard links: two names, one identity ----
$ ln /tmp/f1 /tmp/f2
$ ls -li /tmp/f1 /tmp/f2
# 1837 -rw------- 2 ... f1      ← same inode number, Links: 2
# 1837 -rw------- 2 ... f2
$ echo "via f2" >> /tmp/f2 && cat /tmp/f1
# hello inode / via f2          ← "both files" changed: there is only one file
$ rm /tmp/f1 && cat /tmp/f2    # no "original" existed — f2 works fine
$ stat -c 'Links: %h' /tmp/f2  # Links: 1

# ---- 3. Symlinks: a name containing a name ----
$ ln -s /tmp/f2 /tmp/sym
$ ls -li /tmp/sym
# 1901 lrwxrwxrwx 1 ... sym -> /tmp/f2    ← DIFFERENT inode, type l, size 7
$ readlink /tmp/sym            # the inode's "data" is literally this string
$ rm /tmp/f2 && cat /tmp/sym
# No such file or directory     ← dangling: resolution happens at access time
$ rm /tmp/sym

# ---- 4. A directory is a table — watch link counts prove it ----
$ mkdir -p /tmp/d/sub1 /tmp/d/sub2 /tmp/d/sub3
$ stat -c 'Links: %h' /tmp/d
# Links: 5     ← itself + its '.' + one '..' per subdir (2 + 3). The
#                link count of a directory = 2 + number of subdirectories!
$ ls -a /tmp/d | head -3       # . and .. are real entries in the table

# ---- 5. Deletion vs identity: the full story in one experiment ----
$ python3 - << 'EOF'
import os, time
fd = os.open("/tmp/ghost", os.O_RDWR | os.O_CREAT, 0o600)
os.write(fd, b"I have no name but I exist\n")
os.unlink("/tmp/ghost")                    # link count → 0. Name GONE.
print("file deleted. fd still works:")
os.lseek(fd, 0, 0)
print(" ", os.read(fd, 100).decode().strip())
print("  /proc shows:", os.readlink(f"/proc/self/fd/{fd}"))
os.close(fd)                               # last reference → NOW data is freed
EOF
#   /tmp/ghost (deleted)   ← the (deleted) suffix from Lesson 36, explained

# ---- 6. Atomic replace: why rename is sacred ----
$ echo "v1 config" > /tmp/conf
$ echo "v2 config half-wri" > /tmp/conf.tmp   # imagine a crash here...
$ mv /tmp/conf.tmp /tmp/conf                   # rename(): ATOMIC pointer swap
$ cat /tmp/conf
# readers at every instant saw v1 OR v2 — never a torn file. (mv within one
# fs = rename(2); across filesystems it degrades to copy+delete — NOT atomic!)

# ---- 7. df -i: the other kind of full ----
$ df -i /tmp | tail -1
# Inodes  IUsed  IFree IUse%   ← the budget nobody monitors until the outage
$ rm -rf /tmp/d /tmp/conf
```

---

## Further Reading

| Topic | Link |
|---|---|
| inode (Wikipedia) | <https://en.wikipedia.org/wiki/Inode> |
| `stat(2)` man page | <https://man7.org/linux/man-pages/man2/stat.2.html> |
| Virtual file system | <https://en.wikipedia.org/wiki/Virtual_file_system> |
| Hard link / Symbolic link | <https://en.wikipedia.org/wiki/Hard_link> |
| `rename(2)` — atomicity | <https://man7.org/linux/man-pages/man2/rename.2.html> |
| `link(2)` / `unlink(2)` | <https://man7.org/linux/man-pages/man2/link.2.html> |

---

## Checkpoint

**Q1.** Deleting a 10 GB file that a process still has open frees no space.
Explain via reference counting, name the tool that finds the holder, and give
the two operational resolutions.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An inode's storage is freed only when both reference types reach zero:
directory entries (link count — rm removed the last one) <em>and</em> open
file descriptions (the process's fd chain still points at the inode — Lesson
36's three layers). The file is in the "no name, still referenced" state:
invisible to ls, absent from du (which walks names), but alive — which is
exactly the df-vs-du discrepancy that triggers these investigations. Finder:
<code>lsof +L1</code> (link count &lt; 1) or <code>ls -l /proc/*/fd | grep
deleted</code>. Resolutions: restart/signal the holding process (log daemons:
the rotation-reopen signal, e.g. SIGHUP/copytruncate — the whole reason
logrotate has those options); or, emergency space recovery without a restart:
truncate the data through the fd — <code>: &gt; /proc/PID/fd/N</code> —
the inode stays open but its blocks free (the process better tolerate its
file shrinking!).
</details>

**Q2.** Why can't hard links cross filesystems or point at directories, while
symlinks can do both? Derive all four answers from the mechanisms.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A hard link IS a directory entry: <code>name → inode number</code>. Inode
numbers are meaningful only within one filesystem's inode table — a directory
on fs A storing "inode 8812" of fs B would be ambiguous or dangling (8812
exists on A too, as something else): no cross-fs hard links, structurally. No
directory hard links because multiple parent entries for one directory create
cycles and ambiguity ('..' points where?) — the fsck-breaking, tree-walk-
looping case the kernel simply forbids (the '.'/'..' entries in the lab ARE
directory hard links, the only sanctioned ones). Symlinks dodge both limits
because they store a <em>pathname string</em>, resolved fresh at each access:
a string can name anything — other filesystems, directories, files that don't
exist yet (dangling), even relative paths whose meaning changes with the
symlink's location. Cost of the flexibility: resolution overhead per access,
TOCTOU races on the target (Lesson 25's file-flavored sibling), and no
guarantee the target outlives the link — identity vs indirection, each with
its native trade.
</details>

**Q3.** Every well-behaved program updates config files as
`write tmp → fsync → rename`. Justify each of the three steps by naming the
failure it prevents — and explain what property of rename() the pattern
leans on.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Write to a temp file</strong> (same directory!): writing the real
file in place means any crash mid-write leaves a torn config — half old, half
new, possibly unparseable: services fail to boot at 4 a.m. The temp file
absorbs the danger where nobody reads it. <strong>fsync before rename</strong>
(Lesson 41's full story): rename can reach disk <em>before</em> the temp
file's data does (metadata journals faster than data writeback — Lesson 20's
dirty pages); after a crash the name would point at a zero-length or garbage
inode — the infamous "ext4 ate my file" bug class. fsync forces the data
durable first, so the swap only ever exposes complete contents.
<strong>rename</strong>: the kernel guarantees the name's transition
old-inode → new-inode is atomic with respect to all observers — every
concurrent open() gets one version or the other, never a mixture and never
ENOENT. (Same-directory matters twice: rename is only atomic within a
filesystem, and it lets one directory-fsync persist the swap.) The pattern is
the userspace edition of a database commit: prepare off to the side, force
durability, then flip one atomic pointer.
</details>

---

## Homework

Explain the design of `git`'s object store using this lesson: objects live at
`.git/objects/ab/cdef123...` named by content hash, are written once and
never modified, and `git gc` uses hard-link-like refcounting ideas. Why is
content-addressing a perfect match for the inode model — and where does git
deliberately *avoid* relying on filesystem semantics (hint: what does it do
instead of rename for refs, and why do packfiles exist)?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Content-addressing means an object's name IS its identity — immutable by
construction (change a byte, it's a different hash → different file), so git
maps beautifully onto inodes: write-once files never face torn-update
problems, dedup is free (same content = same path — like hard links, sharing
by identity), and safe creation is trivial (write tmp, rename into place —
the atomic-replace pattern; if the object already exists, the collision is
harmless because contents are identical by definition). Where git distrusts
the filesystem: <strong>refs</strong> (branch pointers — the only mutable
state) use the full write-lock-fsync-rename ritual, and packed-refs +
reflogs, because a ref is precisely the pointer-flip that needs commit
semantics; and <strong>packfiles</strong> exist because "one inode per
object" collides with real filesystem limits this lesson taught: millions of
small objects exhaust inodes (df -i!), waste a block each (internal
fragmentation), and make directory lookups and backups crawl — so git
consolidates into few large packfiles with its own internal index,
effectively implementing a filesystem-within-a-file where the VFS's
per-object costs are too high. The meta-lesson runs both ways: model
immutable data as content-addressed files (the fs gives you integrity
almost free), and when object counts hit filesystem scaling walls, wrap
them in your own container format — every serious datastore (git, databases,
object stores) lands on that same pair of moves.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 38 — ext4 and Journaling →](lesson-38-ext4-journaling){: .btn .btn-primary }
