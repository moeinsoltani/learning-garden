---
title: "Lesson 40 — Mounts and Bind Mounts"
nav_order: 5
parent: "Phase 7: Files & Filesystems"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 40: Mounts and Bind Mounts

## Concept

Filesystems (Lessons 38–39) are trees; the **mount table** is how the kernel
grafts them into the single directory hierarchy every process sees. Each
mount says: "at this directory, stop looking in the parent filesystem —
descend into this other one instead."

```
        /            ← ext4 on /dev/vda1
        ├── home/    ← (same ext4)
        ├── tmp/     ← graft: tmpfs
        ├── proc/    ← graft: procfs (L04!)
        ├── mnt/usb  ← graft: vfat on /dev/sdb1
        └── data/    ← graft: xfs on /dev/vdb
                        └── path resolution walks ACROSS these
                            seams without programs ever noticing
```

The mount table is a first-class kernel object with real algebra:

- **Bind mounts** — graft *a directory* to a second place:
  `mount --bind /var/data /srv/www`. Not a symlink (no path string to
  resolve or escape — it *is* the same subtree, at VFS level), and it can
  carry different options: the same files read-write here, **read-only**
  there (`--bind` + remount ro) — one data set, per-consumer policy.
- **Per-process mount namespaces** (Lesson 59's foundation): the mount table
  itself can be cloned — different processes can see *different* grafts.
  Container filesystems = a mount namespace + overlayfs (L39) + bind mounts
  for volumes. Every `docker run -v /host:/ctr` is a bind mount across
  namespaces.
- **Propagation** — when namespaces share mounts, do new mounts flow between
  them? (shared/private/slave — the systemd-era subtlety that explains
  "mounted a USB stick, container can't see it".)

Mount options are *policy at the seam*: `ro`, `noexec`, `nosuid`, `nodev` —
enforced by the VFS regardless of what the filesystem contains. A world of
hardening lives in these four words.

---

## How It Works

### Reading the table

`findmnt` shows the tree (source, target, fstype, options — the modern tool);
`/proc/self/mounts` is the raw per-namespace truth. A path's governing mount:
`findmnt -T /some/path`. Stacking is legal: mounting over a non-empty
directory *shadows* its contents (still on disk, unreachable — a classic
"where did my disk space go": files hidden under a later mount; bind-mount
the parent elsewhere to peek).

### The security options, precisely

- `ro` — VFS rejects writes (even root's) through this mount point.
- `noexec` — execve() fails for binaries here (scripts via `sh file` still
  run — it's an execve gate, not a content gate).
- `nosuid` — setuid/setgid bits ignored (Lesson 11's cliff, fenced): a
  setuid-root binary on a nosuid mount runs as *you*.
- `nodev` — device nodes inert (a crafted /dev/sda char node on a USB stick
  does nothing).

Untrusted media and writable service directories get all four; `/tmp` as
`nosuid,nodev` (often `noexec`) is standard hardening; every container
runtime mounts /proc and friends with these.

### Propagation and peer groups

Mounts under a **shared** mount propagate to namespace peers (systemd makes
`/` shared by default — desktop USB sticks appear everywhere); **private**
means isolation both ways; **slave** receives but doesn't send (the container
default for host mounts: host changes flow in, container mounts don't leak
out); **unbindable** refuses even binding. `findmnt -o TARGET,PROPAGATION`
reveals the topology. The one rule that saves debugging hours: a mount made
*after* a namespace was created reaches it only if the parent mount's
propagation says so.

{: .note }
> **pivot_root, conceptually**
> Changing "what / means" for a process tree is <code>pivot_root</code>:
> move the current root elsewhere, make a new mount the root — the final
> step of every container runtime and of boot's initramfs→real-root handoff
> (Lesson 51). <code>chroot</code> is its weak ancestor (a path-lookup trick,
> escapable by design); pivot_root rewrites the mount reality. Lesson 62
> performs it.

---

## Lab

```bash
# ---- 1. Read your mount tree fluently ----
$ findmnt | head -12                      # the tree, grafts visible
$ findmnt -T /var/log                     # which mount governs this path?
$ findmnt -no OPTIONS /proc /dev/shm      # options doing quiet security work

# ---- 2. Bind mounts: one subtree, two names, two policies ----
$ mkdir -p /tmp/data /tmp/view && echo "secret v1" > /tmp/data/file
$ sudo mount --bind /tmp/data /tmp/view
$ echo "changed via view" >> /tmp/view/file && cat /tmp/data/file
# both paths ARE the subtree — no symlink resolution anywhere:
$ ls -l /tmp/view                          # no 'l', no ->: real directory
# now: same data, READ-ONLY at one name:
$ sudo mount -o remount,ro,bind /tmp/view
$ echo nope >> /tmp/view/file 2>&1 | tail -1
# Read-only file system            ← policy at the seam
$ echo "still writable here" >> /tmp/data/file && tail -1 /tmp/view/file
# writes via the rw name are VISIBLE through the ro name. One tree, two doors.

# ---- 3. nosuid/noexec: fencing the cliff ----
$ mkdir -p /tmp/fenced && sudo mount -t tmpfs -o nosuid,noexec tmpfs /tmp/fenced
$ cp /bin/ls /tmp/fenced/ && /tmp/fenced/ls 2>&1 | tail -1
# Permission denied                ← noexec: execve refused
$ sudo cp /usr/bin/sudo /tmp/fenced/sudo_copy 2>/dev/null; ls -l /tmp/fenced/sudo_copy
# the setuid bit may survive the copy — but nosuid makes it INERT here:
$ /tmp/fenced/sudo_copy -V 2>&1 | head -1 || echo "(and noexec blocked it anyway)"
$ sudo umount /tmp/fenced

# ---- 4. Shadowing: mounting over a full directory ----
$ mkdir -p /tmp/shadow && echo "hidden treasure" > /tmp/shadow/gold
$ sudo mount -t tmpfs tmpfs /tmp/shadow
$ ls /tmp/shadow                           # empty! gold is SHADOWED, not gone
$ sudo mount --bind / /mnt 2>/dev/null && ls /mnt/tmp/shadow/ && sudo umount /mnt
# gold                             ← peeking under the mount via a bind of /
$ sudo umount /tmp/shadow && ls /tmp/shadow
# gold                             ← unmount reveals it again

# ---- 5. Mount namespaces: your private mount table ----
$ sudo unshare --mount bash -c '
    mount -t tmpfs tmpfs /mnt
    echo "only my namespace sees this" > /mnt/private
    findmnt /mnt | tail -1
    echo "--- inside: /mnt has:"; ls /mnt'
$ ls /mnt
# (empty on the host!)             ← the namespace's mount never existed here
# THIS is why every container has its own /: a cloned mount table + grafts

# ---- 6. Propagation: why the container missed your USB stick ----
$ findmnt -o TARGET,PROPAGATION / | head -2
# /  shared                        ← systemd's default: mounts flow to peers
$ sudo unshare --mount --propagation slave bash -c \
    'findmnt -o TARGET,PROPAGATION / | head -2'
# /  private,slave                 ← receives host mounts, leaks nothing back —
#                                    the container runtimes' choice, explained
$ sudo umount /tmp/view; rm -rf /tmp/data /tmp/view /tmp/shadow /tmp/fenced
```

---

## Further Reading

| Topic | Link |
|---|---|
| `mount(8)` man page | <https://man7.org/linux/man-pages/man8/mount.8.html> |
| `mount_namespaces(7)` | <https://man7.org/linux/man-pages/man7/mount_namespaces.7.html> |
| Bind mount / mount --bind | <https://man7.org/linux/man-pages/man8/mount.8.html#BIND_MOUNT_OPERATION> |
| Shared subtrees — kernel docs | <https://www.kernel.org/doc/html/latest/filesystems/sharedsubtree.html> |
| `pivot_root(2)` man page | <https://man7.org/linux/man-pages/man2/pivot_root.2.html> |
| `findmnt(8)` man page | <https://man7.org/linux/man-pages/man8/findmnt.8.html> |

---

## Checkpoint

**Q1.** Bind mount vs symlink: `mount --bind /var/data /srv/www` vs
`ln -s /var/data /srv/www`. Name three behavioral differences that matter in
production.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <strong>Resolution vs identity</strong>: a symlink is a pathname string
resolved at every access — programs that call realpath, tools that refuse to
follow links, and chroot/namespace boundaries (a symlink to /var/data inside
a container resolves against the <em>container's</em> /var/data — probably
nothing) all behave differently; a bind mount IS the subtree at VFS level —
invisible to applications, works across chroots because there's nothing to
resolve. (2) <strong>Independent options</strong>: the bind can be ro,
nosuid, noexec while the original stays rw (lab 2) — a symlink can carry no
policy whatsoever. (3) <strong>Lifecycle & escape semantics</strong>: the
bind pins the filesystem (unmounting /var's fs fails while bound —
EBUSY protection), survives the target directory being renamed, and can't
dangle; a symlink can dangle silently and — the security classic — can be
swapped mid-use to redirect a privileged process (TOCTOU, Lesson 25/37).
Rule: user-visible convenience shortcuts → symlink; infrastructure grafting
with policy → bind mount.
</details>

**Q2.** Why does `mount -o remount,ro,bind /tmp/view` make the *bind* read-
only while `/tmp/data` stays writable — where in the kernel's object model
does the ro live? And why doesn't plain `mount -o remount,ro` on the original
mount give you this?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Mount options live on the <em>mount object</em> (the graft), not on the
filesystem or the inodes: each mount of the same subtree is a separate VFS
object carrying its own flag set, checked during permission evaluation on
every access <em>through that mount</em>. The bind created a second mount
object over the same dentry tree; remounting it ro sets MNT_READONLY on that
object alone — writes through /tmp/view die at the VFS gate while
/tmp/data's mount object still says rw. (Historically bind mounts couldn't
diverge on options; per-mount read-only arrived precisely for this pattern.)
Remounting the <em>original</em> ro would flip the whole filesystem's
superblock (or that mount object) — everything under it becomes ro for
everyone, which is a different, blunter policy. The layered model to
remember: superblock flags (fs-wide) → mount flags (per graft) → inode modes
(per file, Lesson 37) — three stacked gates, and "why can't I write?" can be
any of them (findmnt for the middle one).
</details>

**Q3.** You plug in a USB disk; it appears on the host but not inside a
running container that bind-mounts /media. Explain via propagation, and the
two ways to fix it (one mount-side, one design-side).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The container's mount namespace cloned the host's table at <em>start</em>
time; whether <em>later</em> host mounts appear inside depends on the
propagation type of the mounts involved. Container runtimes typically make
the container's tree <code>private</code> or <code>slave</code> — and if the
volume was bind-mounted as private (or the runtime marked everything
private, the old Docker default), the USB automount under /media on the host
propagates nowhere: the container sees the /media directory as it was, new
graft absent. Fixes: mount-side — make the bind propagation-aware: bind
/media into the container as <code>slave</code> (or shared) — e.g. Docker's
<code>-v /media:/media:rslave</code> — so host-side mounts flow in (slave =
one-way: container mounts still don't leak out, keeping the isolation
direction that matters); design-side — don't depend on propagation at all:
mount the specific device/directory before container start, or restart/
signal the consumer when media changes (hotplug handled by the host, data
handed over as a plain directory). Debug kit: <code>findmnt -o
TARGET,PROPAGATION</code> on both sides — the answer is always visible there.
</details>

---

## Homework

systemd units support `ProtectSystem=strict`, `ReadOnlyPaths=`,
`PrivateTmp=yes`, `InaccessiblePaths=`. Pick a service on your VM, read
`systemd-analyze security <unit>`, and explain — in mount-table terms from
this lesson — what machinery each of those four options builds when the
service starts. Verify one of them by inspecting the running service's
namespace (`findmnt -N <PID>` or `nsenter`).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
All four are the same recipe: systemd gives the service a <strong>private
mount namespace</strong> (lab 5) and then rearranges its table before exec.
<code>ProtectSystem=strict</code>: remount-bind <em>the entire tree</em>
read-only (every mount object gets MNT_READONLY in the namespace), with
carve-outs (ReadWritePaths) bind-mounted rw back over it — Q2's per-mount ro,
applied wholesale. <code>ReadOnlyPaths=</code>: targeted version — bind each
listed path over itself, remount ro (exactly lab 2's two commands).
<code>PrivateTmp=</code>: mount a fresh private tmpfs at /tmp and /var/tmp in
the namespace — the service's temp files are invisible to other services
(and vice versa: no /tmp symlink attacks across services — Lesson 37's
TOCTOU class, amputated), destroyed at service stop. 
<code>InaccessiblePaths=</code>: mount an empty, 000-mode "jail" node over
the path (shadowing, lab 4, as a weapon) — the subtree exists but cannot be
entered. Verification: <code>findmnt -N $(systemctl show -p MainPID --value
unit)</code> (or <code>sudo nsenter -t PID -m findmnt</code>) shows the
namespace's table: the ro remounts, the tmpfs at /tmp, the shadow mounts —
none visible in your own findmnt. The takeaway: "sandboxing" in systemd is
overwhelmingly <em>mount-table surgery</em> — the primitives of this lesson,
scripted per service, which is why <code>systemd-analyze security</code>
reads like a mount-option checklist.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 41 — Buffered vs Direct I/O, and fsync →](lesson-41-io-fsync){: .btn .btn-primary }
