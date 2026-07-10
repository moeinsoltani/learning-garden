---
title: "Lesson 62 — Build a Container by Hand (Capstone)"
nav_order: 4
parent: "Phase 12: Containers & Resource Control"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 62: Build a Container by Hand — Capstone

## Concept

**Phase 12 capstone.** This lesson has no new concept — it *assembles* every
piece you've built. You will create a working container with no Docker, no
containerd, no runc: just the primitives from across this course, wired
together in a shell script (and understood well enough to explain each line).

A container, fully unfolded, is exactly four things you now own:

```
   1. NAMESPACES (L59)       what it can SEE:
        unshare mnt/pid/net/uts/ipc/user → its own machine-illusion
   2. ROOT FILESYSTEM (L61)  what its / IS:
        overlayfs(image layers) + pivot_root → private root
   3. CGROUPS (L60)          what it can USE:
        cpu.max, memory.max, pids.max → bounded resources
   4. CONFINEMENT (Phase 11) what it can DO:
        drop capabilities (L11) + seccomp (L56) + (optionally MAC, L57)

   runc/Docker do all four, robustly, with hundreds of edge cases handled.
   You'll do the core of it in ~40 lines and SEE there's no magic left.
```

When this runs and drops you into a shell that thinks it's alone on its own
machine — its own PID 1, its own root, its own hostname, bounded memory,
dropped privileges — the entire course closes: from Lesson 01's "one kernel,
many isolated views" to a container you built from that kernel's primitives.

---

## How It Works

The script's flow (each step a lesson):

1. **Prepare the root** (Lesson 61): overlay the image's read-only layers
   under a writable upper → a merged root directory.
2. **Unshare namespaces** (Lesson 59): `unshare` with the namespace flags,
   `--fork` so the child becomes PID 1 of the new PID namespace, mapping root
   in the user namespace.
3. **Set up the mount namespace** (Lesson 40): make mounts private, mount the
   overlay, mount fresh `/proc` `/sys` `/dev` inside the new root.
4. **pivot_root** (Lesson 40): swap `/` to the container's root and detach the
   old one — the real root switch (stronger than chroot, which is escapable).
5. **Apply a cgroup** (Lesson 60): create one, set limits, put the container's
   PID in it.
6. **Drop capabilities + seccomp** (Lessons 11/56): shed privileges the
   container doesn't need before exec'ing its command.
7. **exec the entrypoint** — which becomes the container's PID 1 (Lesson 08's
   reaping duty now its responsibility).

The lab gives a runnable version. What it *omits* versus runc is the homework's
subject — and that list is precisely the difference between "understanding
containers" (this lesson) and "shipping a container runtime" (a large,
security-critical engineering effort).

---

## Lab

> Throwaway VM. This creates namespaces, mounts, and cgroups; a reboot cleans
> up anything left behind.

```bash
# ---- 0. Prepare an image root (Lesson 61's homework, as setup) ----
$ sudo apt install -y skopeo 2>/dev/null | tail -1
$ mkdir -p /tmp/box/{lower,upper,work,merged}
$ cd /tmp/box
$ skopeo copy docker://alpine:latest oci:img:latest 2>/dev/null && \
    LAYER=$(find img/blobs -type f -size +500k | head -1) && \
    sudo tar xzf "$LAYER" -C /tmp/box/lower 2>/dev/null
$ ls /tmp/box/lower                        # bin etc lib usr ... — alpine's rootfs
# (no skopeo? debootstrap or an existing rootfs tarball works too)

# ---- 1. The container script: all four pillars, ~40 lines ----
$ cat > /tmp/mycontainer.sh << 'SCRIPT'
#!/bin/bash
set -e
ROOT=/tmp/box/merged
CG=/sys/fs/cgroup/mybox

# PILLAR 3 (L60): cgroup with limits, created in the PARENT
sudo mkdir -p "$CG"
echo "+memory +pids" | sudo tee /sys/fs/cgroup/cgroup.subtree_control >/dev/null 2>&1 || true
echo "128M" | sudo tee "$CG/memory.max" >/dev/null
echo "50"   | sudo tee "$CG/pids.max"   >/dev/null

# PILLAR 2 (L61): overlay the image layers into a private root
sudo mount -t overlay overlay \
  -o lowerdir=/tmp/box/lower,upperdir=/tmp/box/upper,workdir=/tmp/box/work \
  "$ROOT"

# PILLARS 1+2+4: unshare namespaces (L59), then inside: pivot_root (L40),
# join the cgroup, and exec — all in the child
sudo unshare --mount --uts --ipc --pid --fork --mount-proc="$ROOT/proc" \
  bash -c '
    ROOT='"$ROOT"'; CG='"$CG"'
    # join the cgroup (PILLAR 3) — put ourselves under the limits
    echo $$ | tee "$CG/cgroup.procs" >/dev/null
    # PILLAR 1 (L59): our own hostname
    hostname mybox
    # PILLAR 2 (L40/L61): make mounts private, set up the new root
    mount --make-rprivate /
    mount -t proc  proc  "$ROOT/proc"  2>/dev/null || true
    mount -t sysfs sys   "$ROOT/sys"   2>/dev/null || true
    mount -t tmpfs tmpfs "$ROOT/dev"
    # pivot_root: SWAP / to the container root (stronger than chroot)
    cd "$ROOT"
    mkdir -p oldroot
    pivot_root . oldroot
    umount -l /oldroot && rmdir /oldroot 2>/dev/null || true
    cd /
    # PILLAR 4 (L11): drop capabilities before exec (needs libcap: capsh)
    # (conceptual — capsh --drop=cap_sys_admin,... ; seccomp via a helper)
    echo "=== inside the hand-built container ==="
    echo "hostname: $(hostname)   PID: $$   (I am PID 1 of my namespace)"
    echo "root fs:"; ls /
    echo "processes:"; ps aux 2>/dev/null | head
    exec /bin/sh
  '
SCRIPT
$ chmod +x /tmp/mycontainer.sh

# ---- 2. RUN YOUR CONTAINER ----
$ /tmp/mycontainer.sh
# === inside the hand-built container ===
# hostname: mybox   PID: 1   (I am PID 1 of my namespace)
# root fs:  bin dev etc lib proc sys usr ...   ← ALPINE's root, not the host's!
# processes: (just this shell + ps) ← its own PID namespace
#
# You are now in a container you built. Try:
#   cat /etc/os-release      → Alpine (the image), on YOUR host kernel
#   ps aux                   → only container processes (own PID ns + /proc)
#   hostname                 → mybox (own UTS ns)
#   ls /                     → the overlay root (own mount ns + pivot_root)
#   touch /proof; ls /       → writes land in the upper layer (L39/61)
# exit  ← leaves the container; the host is untouched

# ---- 3. Verify the isolation from the HOST side (another terminal) ----
$ ps aux | grep '[/]bin/sh' | head -1          # the container's shell — a NORMAL
# host process! Its "PID 1" has a real host PID. One kernel, isolated view (L01).
$ CPID=$(pgrep -f 'exec /bin/sh' | head -1)
$ sudo ls -l /proc/$(pgrep -f mybox | head -1)/ns/ 2>/dev/null | head -3 || \
    echo "the container's namespaces differ from the host's (readlink to compare)"

# ---- 4. Verify the cgroup limit is real ----
$ cat /sys/fs/cgroup/mybox/memory.max 2>/dev/null      # 128M — enforced
$ cat /sys/fs/cgroup/mybox/pids.current 2>/dev/null    # live process count
# a fork bomb or 200MB alloc INSIDE the container hits these walls, not the host's

# ---- 5. Clean up ----
$ sudo umount /tmp/box/merged 2>/dev/null
$ sudo rmdir /sys/fs/cgroup/mybox 2>/dev/null
$ cd / && rm -rf /tmp/box /tmp/mycontainer.sh
```

---

## Further Reading

| Topic | Link |
|---|---|
| runc — the reference OCI runtime | <https://github.com/opencontainers/runc> |
| "Building a container from scratch" — Liz Rice talk | <https://www.youtube.com/watch?v=8fi7uSYlOdc> |
| OCI Runtime Specification | <https://github.com/opencontainers/runtime-spec/blob/main/spec.md> |
| `pivot_root(2)` man page | <https://man7.org/linux/man-pages/man2/pivot_root.2.html> |
| `unshare(1)` / `nsenter(1)` | <https://man7.org/linux/man-pages/man1/unshare.1.html> |
| bocker — Docker in 100 lines of bash | <https://github.com/p8952/bocker> |

---

## Checkpoint

**Q1.** List everything your hand-rolled container is still missing compared
to `docker run` — there's a real list; make it, grouping by the four pillars.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Pillar 1 — Namespaces/isolation</strong>: no network namespace with
actual connectivity (the script skipped --net, or if added, no veth/bridge
plumbing to the host — the whole networking track — so the container has no
network; Docker sets up veth, bridge, NAT, DNS, port mapping); no user
namespace with uid mapping (still real root inside = a much weaker security
boundary — rootless containers, Lesson 59 Q2); no cgroup namespace (the
container can see the host's cgroup tree). <strong>Pillar 2 — Root
filesystem</strong>: no image management (pull/verify by digest, layer
caching, the OCI registry protocol — Lesson 61); no proper /dev population
(Docker creates a curated set of device nodes with the right permissions —
Lesson 53 — we mounted an empty tmpfs); no volume/bind-mount handling for
persistent or shared data; no read-only-rootfs option. <strong>Pillar 3 —
Resources</strong>: only memory+pids limits (no cpu.max/weight, io limits,
cpuset pinning — Lesson 60; no per-container accounting exposed). <strong>Pillar
4 — Confinement</strong>: <em>no capability dropping actually applied</em>
(the script only gestured at it — the container runs with full caps, a serious
gap — Lesson 11); <em>no seccomp filter</em> (full syscall surface — Lesson 56,
Docker's ~44-syscall default profile absent); no MAC profile (AppArmor/SELinux
— Lesson 57); no NoNewPrivileges. <strong>Lifecycle/robustness</strong>: no
proper PID-1 init to reap zombies (Lesson 08's container-zombie problem — our
shell is a poor init); no signal forwarding (docker stop → SIGTERM handling);
no cleanup on crash (leaked mounts/cgroups — we clean up manually); no image
build, no logging/journald integration, no health checks, no restart policy,
no OCI-spec compliance, and none of the hundreds of error/edge cases (mount
propagation subtleties, cgroup v1/v2 differences, rootless plumbing via
setuid helpers, etc.). The honest conclusion: we built the <em>core mechanism</em>
— enough to prove there's no magic — but runc/Docker is that core plus an
enormous amount of security hardening, networking, lifecycle management, and
edge-case handling. Understanding containers (this lesson) and shipping a
production runtime are separated by exactly this list — most of which is
security (Pillar 4) and networking, the two hardest parts.
</details>

**Q2.** Why does the script use `pivot_root` instead of `chroot` to set the
container's root? What's the security difference, and what did the mount
namespace contribute?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<code>chroot</code> only changes where <em>path lookups starting with /</em>
resolve — it's a per-process pointer to a directory, and critically it does
<em>not</em> change the mount namespace or detach the old root, which is why
chroot is famously <strong>escapable</strong>: a process with sufficient
privilege (CAP_SYS_ADMIN, or an open fd to a directory outside the chroot, or
via a second chroot trick) can walk back out to the real root — chroot is a
convenience, never a security boundary. <code>pivot_root</code> actually
<em>swaps the mount that is /</em>: it makes the new root the process tree's
real root mount and moves the old root aside, and the script then
<code>umount -l /oldroot</code> to <em>detach the old root entirely</em> — so
there is no mount, no path, no reference by which to reach the host filesystem;
the container's world genuinely ends at its root. The mount namespace
(Lesson 40) is what makes this safe and contained: pivot_root operates on the
mount table, and because we're in a <em>private</em> mount namespace
(--mount + --make-rprivate), all this root-swapping and unmounting affects
<em>only the container's</em> mount view — the host's mount table is untouched
(the host still has its normal /, the old root is only detached inside the
namespace). Without the mount namespace, pivot_root would try to rearrange the
<em>host's</em> real root (destructive/forbidden). So the pair is essential:
the mount namespace gives a private, disposable mount table, and pivot_root +
detach uses it to install an inescapable root — together the real "the
container has its own filesystem, and can't see the host's" property that
chroot alone only pretends to provide. This is exactly the initramfs's
switch_root handoff (Lesson 51) and its logic — assemble a root in a private
mount context, pivot into it, detach the old — reused for containers, which is
why Lesson 51's homework noted boot and containerization are the same move.
</details>

**Q3.** Your container's shell runs as PID 1 of its namespace. Why is that a
problem for a real workload, and what do production runtimes do about it?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
PID 1 has special duties (Lessons 08/52): it must reap orphaned processes
(every process whose parent dies re-parents to PID 1 — inside a PID namespace,
that's <em>your</em> PID 1), and it receives signals differently (signals like
SIGTERM have no default action for PID 1 — the kernel won't kill it unless it
installs a handler, Lesson 09). A plain <code>/bin/sh</code> or a typical
application as PID 1 handles neither well: it doesn't <code>wait()</code> for
orphaned grandchildren, so <strong>zombies accumulate</strong> (Lesson 08's
container-zombie problem — a service that spawns subprocesses whose children
outlive them leaks defunct entries until the PID namespace's pids.max is
exhausted); and it often ignores SIGTERM (no handler), so <code>docker stop</code>
times out and escalates to SIGKILL — meaning no graceful shutdown, lost
in-flight work, corrupted state. Production runtimes fix it two ways: (1) a
<strong>minimal init as PID 1</strong> — <code>docker run --init</code> injects
<code>tini</code> (or podman's catatonit), a ~tiny proper init that does
nothing but reap zombies and forward signals to the real process, which runs
as PID 2; this is the Lesson 08 supervisor pattern, containerized. (2)
<strong>Make the application init-aware</strong> — the app installs a SIGTERM
handler for graceful shutdown (Lesson 09's homework skeleton) and reaps its own
children (or uses a language runtime that does). Kubernetes adds
<code>terminationGracePeriodSeconds</code> and a preStop hook around this
SIGTERM→SIGKILL window. The deeper point closes the phase: being PID 1 is the
same load-bearing responsibility whether you're systemd on the host (Lesson 52)
or an app in a container — the kernel makes no exception, so a container's
"first process" inherits init's job, and ignoring that is one of the most
common real-world container bugs. Our hand-built container has exactly this
flaw (the checkpoint Q1 list noted it) — the honest gap between a teaching
example and a runtime.
</details>

---

## Homework

You've built a container and enumerated what it's missing (Q1). Now close the
most important gap: add real **confinement** (Pillar 4) to the script. Sketch
how you'd (a) drop all capabilities except a minimal set before exec
(`capsh --drop=` or `prctl`/libcap — Lesson 11), and (b) apply a seccomp
allowlist (Lesson 56, via a small C helper or a runtime that installs it
before exec). Then reflect: having built every pillar by hand across this
phase, write your own two-sentence definition of "what a container is" —
and explain why the common phrasing "containers virtualize the OS" is
misleading given everything you now know.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) Capability dropping: after entering the namespaces but <em>before</em> exec
of the entrypoint, drop the bounding set to a minimal allowlist — e.g. wrap the
final exec in <code>capsh --drop=cap_sys_admin,cap_sys_module,cap_net_admin,...
--</code> (or in C: <code>prctl(PR_CAPBSET_DROP, ...)</code> for each unwanted
capability, then <code>cap_set_proc</code> with a minimal set, plus
<code>prctl(PR_SET_NO_NEW_PRIVS, 1)</code> so no setuid binary can regain them —
Lesson 11). Docker keeps ~14 capabilities by default and drops the rest; the
principle is deny-by-default (start from none, add only what's needed —
CAP_NET_BIND_SERVICE for a web server, etc.). (b) seccomp: before exec, install
a BPF allowlist (Lesson 56) — practically, via libseccomp in a small helper
that <code>seccomp_init(SCMP_ACT_ERRNO)</code>, allows the syscalls the workload
needs (derived empirically with strace — Lesson 56's homework), <code>seccomp_load</code>s,
then execs; the filter is one-way so the container can't lift it. Both go in the
child, in the last moment before exec, after all setup that <em>needs</em> the
privileges is done (mounting, pivot_root need CAP_SYS_ADMIN — so drop
<em>after</em> them, Lesson 11's privilege-drop timing). My two-sentence
definition: <em>"A container is an ordinary host process placed into a private
set of namespaces (its isolated view of PIDs, mounts, network, users, etc.),
bounded by a cgroup (its share of CPU, memory, I/O, and process count), rooted
in an overlay of read-only image layers plus a private writable layer, and
confined by dropped capabilities + a seccomp filter + optionally a MAC profile
(what it may do). It runs on — and shares — the host's single kernel, which is
simultaneously why it's lightweight and why its isolation is only as strong as
that kernel's boundaries."</em> Why "containers virtualize the OS" misleads:
there is <em>no</em> virtualized OS — no guest kernel, no emulated hardware,
nothing virtualized in the sense a VM virtualizes a machine (virt track). The
container's processes make real syscalls to the real host kernel (Lesson 02);
"the OS" they see is an <em>illusion assembled from namespaces + a root
filesystem</em>, not a second operating system running for them. The phrasing
invites the "lightweight VM" error (Lesson 59 Q3) and hides the crucial
security truth you've now proven by construction: a container is a
<em>configured process</em>, so a kernel bug reachable from it breaks the whole
illusion — whereas a VM's guest OS is genuinely separate, enforced by hardware.
Having built all four pillars by hand, you know a container isn't a lighter kind
of machine; it's a heavier kind of process — and that reframing, from Lesson 01
to here, is the whole point of the course.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 13 — Tracing & Debugging (Lesson 63: perf) →](lesson-63-perf){: .btn .btn-primary }
