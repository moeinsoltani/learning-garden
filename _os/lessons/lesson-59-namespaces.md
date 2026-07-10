---
title: "Lesson 59 — Namespaces: All of Them"
nav_order: 1
parent: "Phase 12: Containers & Resource Control"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 59: Namespaces — All of Them

## Concept

The networking track's very first lesson introduced *network* namespaces — an
independent networking "universe" per process. That was one of **eight**
namespace types, and together they are half of what makes a container. A
namespace virtualizes one kind of global kernel resource so that processes in
different namespaces see different, isolated instances of it:

```
   global kernel resource        namespace that isolates it
   ──────────────────────        ──────────────────────────
   the filesystem mount tree  →  MNT   (Lesson 40's mount namespaces)
   process IDs                →  PID   (its own PID 1, own pstree)
   network stack              →  NET   (networking Lesson 01 — you know this!)
   hostname/domain            →  UTS   (own `hostname`)
   IPC objects (shm, L33)     →  IPC   (own shared memory, mqueues)
   user/group IDs             →  USER  (uid 0 inside = uid 100000 outside!)
   cgroup root view           →  CGROUP
   system clocks (boot/mono)  →  TIME
```

A container is a process placed into a fresh set of these namespaces at once,
so it sees its own root filesystem, its own PID 1, its own network, its own
hostname — the *illusion of a separate machine* — while actually being an
ordinary process on the host kernel (Lesson 01: one kernel, many isolated
views). No hypervisor, no guest kernel (that's virtualization's VM — virt
Lesson 59's microVMs bridge the two).

The tools are `unshare` (create new namespaces for a new process) and
`nsenter` (join an existing process's namespaces — how `docker exec` and
`kubectl exec` work). You'll drive each namespace type by hand and *see* the
isolation appear.

---

## How It Works

### clone flags, one more time

Namespaces are created by the `clone()`/`unshare()`/`setns()` syscalls with
flags (`CLONE_NEWNET`, `CLONE_NEWPID`, `CLONE_NEWUSER`…) — the same `clone`
that made threads and processes (Lesson 24). A namespace exists as long as a
process is in it (or an fd/bind-mount pins it — `/proc/PID/ns/*` are the
handles, and `nsenter` opens them). Namespaces nest and combine freely: a
container typically unshares all of them; a sandbox might unshare only mount +
network.

### The interesting ones' quirks

- **PID**: the first process in a new PID namespace becomes **PID 1** inside it
  — with all of PID 1's duties (reaping, Lesson 08 — the container zombie
  problem) and protections. It still has a *different, real* PID on the host
  (namespaces are a view, not a relocation). A subtlety: you must also give it
  a private `/proc` (a mount-namespace action) or `ps` shows host processes —
  the classic first-container gotcha (the lab reproduces it).
- **USER**: the game-changer for security. It maps UIDs — uid 0 *inside* the
  namespace maps to an unprivileged uid *outside*. So a process can be "root"
  in its container (install packages, bind low ports within its net ns) while
  being a powerless normal user on the host: **rootless containers**. It's
  also the only namespace an *unprivileged* user can create, which bootstraps
  all the others (unprivileged podman, `unshare -r`).
- **MNT** (Lesson 40): private mount tree — the container's `/` via overlayfs
  (Lesson 61) + pivot_root (Lesson 62).
- **TIME** (newest): per-namespace boottime/monotonic offsets — mainly for
  checkpoint/restore (migrate a container and keep its clocks consistent).

### It's a view, not a sandbox by itself

Crucial framing: namespaces provide *isolation of naming/visibility*, not
resource limits (that's cgroups, Lesson 60) and not privilege reduction by
themselves (that's caps/seccomp/LSM, Phase 11). A process in its own PID+net+
mnt namespace can still exhaust the host's CPU and memory unless a cgroup
bounds it. Container = namespaces (what it can *see*) + cgroups (what it can
*use*) + capabilities/seccomp/LSM (what it can *do*) — this phase and Phase 11
assembled.

{: .note }
> **/proc/PID/ns — namespaces as files**
> <code>ls -l /proc/self/ns/</code> shows one symlink per namespace, each with
> an inode number identifying <em>which</em> namespace. Two processes with the
> same inode share that namespace; different inodes = isolated. It's how you
> tell what's shared (<code>readlink</code> and compare) and how nsenter finds
> a target — the "everything is a file" theme (Lesson 35) reaching namespaces
> themselves.

---

## Lab

```bash
# ---- 1. See your namespaces (and that all processes share the defaults) ----
$ ls -l /proc/self/ns/
# net -> net:[4026531840]  pid -> pid:[...]  mnt -> ... uts -> ... user -> ...
$ readlink /proc/self/ns/net; readlink /proc/1/ns/net   # same = shared with init

# ---- 2. UTS namespace: isolated hostname (the simplest) ----
$ sudo unshare --uts bash -c 'hostname container-lab; echo "inside: $(hostname)"'
# inside: container-lab
$ hostname                                       # host UNCHANGED — isolated

# ---- 3. PID namespace: your own PID 1 (+ the /proc gotcha) ----
$ sudo unshare --pid --fork bash -c 'echo "my PID is $$"; sleep 1'
# my PID is 1        ← inside the ns, the shell is PID 1!
# but ps still shows host processes without a private /proc:
$ sudo unshare --pid --fork bash -c 'ps aux | head -3'
# shows HOST processes — the gotcha. Fix with a mount ns + fresh /proc:
$ sudo unshare --pid --fork --mount-proc bash -c 'ps aux'
# NOW ps shows only what's in this PID namespace — a tiny process world

# ---- 4. NET namespace: no interfaces but loopback (networking L01, revisited) ----
$ sudo unshare --net bash -c 'ip link show'
# only 'lo' (down) — a fresh, empty network stack. Everything the whole
# networking track built starts from exactly this.

# ---- 5. USER namespace: root inside, nobody outside (the security key) ----
$ unshare --user --map-root-user bash -c 'echo "inside uid: $(id -u)"; id'
# inside uid: 0       ← I am ROOT in here...
$ unshare --user --map-root-user bash -c 'cat /proc/self/uid_map'
#          0       1000          1   ← ...but uid 0 inside MAPS to uid 1000 outside
# no privilege gained on the host. Note: NO sudo needed — user ns is the
# one unprivileged users can create. Try touching a root-owned host file:
$ unshare --user --map-root-user bash -c 'echo x > /etc/lab_test 2>&1 | tail -1'
# Permission denied   ← "root" here has no power over host-owned files

# ---- 6. Combine them: a hand-made "container" view (no Docker) ----
$ sudo unshare --uts --pid --net --mount --fork --mount-proc bash -c '
    hostname mini-container
    echo "hostname: $(hostname)"
    echo "PID: $$"
    echo "network:"; ip -o link show | wc -l
    echo "processes:"; ps aux | wc -l'
# hostname: mini-container / PID: 1 / network: 1 (just lo) / processes: ~3
# five namespaces, one command: the ISOLATION half of a container. Lesson 62
# adds the root filesystem (pivot_root) + cgroups + caps to finish it.

# ---- 7. nsenter: join a running namespace (how docker/kubectl exec works) ----
$ sudo unshare --uts --fork bash -c 'hostname joined-ns; sleep 30' &
$ sleep 1; TARGET=$(pgrep -f 'sleep 30' | head -1)
$ sudo nsenter --uts --target $TARGET hostname
# joined-ns          ← entered the other process's UTS namespace and saw ITS hostname
$ sudo kill $TARGET 2>/dev/null
$ sudo rm -f /etc/lab_test
```

---

## Further Reading

| Topic | Link |
|---|---|
| `namespaces(7)` overview | <https://man7.org/linux/man-pages/man7/namespaces.7.html> |
| `unshare(1)` man page | <https://man7.org/linux/man-pages/man1/unshare.1.html> |
| `nsenter(1)` man page | <https://man7.org/linux/man-pages/man1/nsenter.1.html> |
| `user_namespaces(7)` | <https://man7.org/linux/man-pages/man7/user_namespaces.7.html> |
| `pid_namespaces(7)` | <https://man7.org/linux/man-pages/man7/pid_namespaces.7.html> |
| Linux namespaces (Wikipedia) | <https://en.wikipedia.org/wiki/Linux_namespaces> |

---

## Checkpoint

**Q1.** Inside a new PID namespace, `ps` still shows all host processes. What
did you forget, and why does that fix it?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You forgot a private <code>/proc</code>. The PID namespace correctly isolated
the process IDs — your shell is PID 1 inside, and only namespace members have
visible PIDs there — but <code>ps</code> doesn't ask the kernel directly; it
reads <code>/proc</code> (Lesson 04: ps is a /proc pretty-printer), and
<code>/proc</code> is still the <em>host's</em> procfs mount, showing the host's
process tree. The fix is to also create a <strong>mount namespace</strong>
(Lesson 40) and mount a fresh <code>proc</code> filesystem
(<code>unshare --mount-proc</code>, or <code>mount -t proc proc /proc</code>
inside a mount ns) — procfs is namespace-aware, so a proc mounted from inside
the PID namespace shows only that namespace's processes. This reveals a deeper
truth: namespaces isolate the <em>kernel resource</em>, but userspace tools
observe through <em>files</em> (/proc, /sys), so full isolation requires
isolating both — the PID namespace for the IDs and the mount namespace for the
/proc view of them. It's why containers always combine mount + PID namespaces
(never PID alone), and a preview of Lesson 62's assembly: a container isn't
one namespace, it's a coordinated set where each covers what the others expose.
</details>

**Q2.** The user namespace lets a process be uid 0 inside while being
unprivileged outside. Explain the mechanism (uid_map) and why this single
feature enabled rootless containers — and note the security caveat it also
introduced.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Mechanism: a user namespace establishes a <em>mapping</em> between UIDs inside
and UIDs outside, written to <code>/proc/PID/uid_map</code> (and gid_map) as
"inside-id outside-id count" — e.g. <code>0 1000 1</code> means "uid 0 inside
= uid 1000 outside." Inside the namespace, the process genuinely <em>is</em>
uid 0: it passes root-only checks <em>for operations scoped to namespaces it
owns</em> — it can create other namespaces, bind ports &lt;1024 in its own
net namespace, chown files it owns within its user namespace, install packages
into its own root filesystem. But every action that reaches a host-owned
resource is checked against the <em>outside</em> uid (1000): touching a
root-owned host file fails (the lab showed it), because the kernel translates
the container-root's identity back to the unprivileged real user for any
cross-namespace decision. This enabled <strong>rootless containers</strong>
(podman, unprivileged Docker): a normal user can run a container whose
processes think they're root — install software, run services as "root",
match images built expecting root — without any actual host privilege, and
without a setuid helper for most operations, because the user namespace is the
<em>one namespace unprivileged users may create</em>, and it bootstraps the
rest (you get root-inside, then unshare the others). The security caveat: user
namespaces vastly <em>expanded the unprivileged attack surface</em> — an
ordinary user can now reach kernel code paths (all the namespace and, through
container-root, many privileged-looking operations) that previously required
real root, and a string of privilege-escalation CVEs came from bugs in those
newly-reachable paths. So some hardened distros restrict unprivileged user
namespaces (<code>kernel.unprivileged_userns_clone</code> sysctl) — the classic
trade: a powerful capability (rootless isolation) inseparable from a larger
surface, gated by policy where the threat model demands.
</details>

**Q3.** Someone says "a container is a lightweight VM." Correct this precisely
using namespaces and Lesson 01 — what does a container share with the host
that a VM does not, and what does that imply for isolation strength?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A container is <em>not</em> a lightweight VM — it's an ordinary host process
(or process tree) with an <em>isolated view</em>, sharing the host's single
kernel (Lesson 01: one kernel, many namespaced views). A VM (virtualization
track) runs its <em>own guest kernel</em> on virtual hardware, with the
hypervisor and CPU virtualization (VT-x/EPT — virt Lessons 05–06) enforcing
a hardware boundary; a container has no guest kernel, no hypervisor — its
"isolation" is entirely the namespaces (visibility) + cgroups (resources) +
caps/seccomp/LSM (permissions) that the shared host kernel applies to a normal
process. What's shared that a VM doesn't share: <strong>the kernel itself</strong>
— all containers' syscalls (Lesson 02) are served by the one host kernel, they
share its scheduler, memory manager, and every kernel bug. Implications for
isolation strength: (1) a container escape is a <em>kernel</em> exploit — one
kernel vulnerability reachable from the container (a bad syscall, a namespace
bug) can break out to the host and every other container, whereas escaping a
VM requires defeating the far smaller, hardware-enforced hypervisor boundary;
(2) containers can't run a different kernel (no Windows containers on a Linux
kernel; no custom kernel per container — Lesson 58 is host-wide); (3) but
containers are dramatically lighter — start in milliseconds, share the kernel's
page cache and memory, no guest OS overhead — because there's no second kernel
to boot or RAM to duplicate. The trade is isolation strength vs efficiency:
VMs give a strong hardware boundary at the cost of a full guest kernel;
containers give near-native efficiency at the cost of a shared-kernel attack
surface. Which is why the two are converging (virt Lesson 59–60's microVMs and
Kata Containers): put each container in a minimal VM to get the container UX
with the VM boundary — because "share the kernel" is the container's defining
property <em>and</em> its defining risk.
</details>

---

## Homework

Build the isolation half of a container step by step, observing what each
namespace adds, then identify what's still missing. Start with
`unshare --uts` and add one namespace at a time (--pid --fork --mount-proc,
--net, --mount, --user --map-root-user), after each checking what changed
(hostname, ps, ip link, mount, id). Then answer: after all namespaces, what
can this "container" still do that a real container can't (name at least
three), and which lessons provide each missing piece?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Building up: <code>--uts</code> → private hostname (host's unchanged);
<code>+ --pid --fork --mount-proc</code> → own PID 1 and a <code>ps</code>
showing only namespace processes (the Q1 fix); <code>+ --net</code> → only
loopback, a blank network stack; <code>+ --mount</code> → a private mount tree
(though still viewing the host's root until you pivot); <code>+ --user
--map-root-user</code> → root-inside/unprivileged-outside. Now you have the
full <em>visibility</em> isolation — but this "container" is dangerously
incomplete: (1) <strong>No resource limits</strong> — it can fork-bomb, exhaust
all host RAM (triggering the host OOM killer — Lesson 22), or spin every CPU;
nothing bounds its <em>usage</em> because namespaces isolate <em>naming</em>,
not <em>consumption</em>. Missing piece: <strong>cgroups</strong> (Lesson 60) —
cpu.max, memory.max, pids.max. (2) <strong>Still uses the host root
filesystem</strong> — it sees the host's <code>/</code> (a mount namespace
alone doesn't change what <code>/</code> <em>is</em>, only that changes are
private); a real container has its own root. Missing pieces:
<strong>overlayfs</strong> (Lesson 61, the image layers) +
<strong>pivot_root</strong> (Lesson 40/62) to install and switch into a
private root. (3) <strong>Full syscall + capability surface</strong> — inside
its user namespace it can make any syscall and holds broad capabilities over
its namespaced resources; a compromise has the whole kernel API to attack.
Missing pieces: <strong>seccomp</strong> (Lesson 56, syscall allowlist) +
<strong>capability dropping</strong> (Lesson 11) + <strong>LSM/MAC</strong>
(Lesson 57, per-container confinement — e.g. Docker's AppArmor profile). Also
missing: no cgroup namespace (it can see the host's cgroup tree), no seccomp,
and networking is unplumbed (just lo — the networking track's veth/bridge work
would connect it). The lesson: namespaces are exactly one of the four pillars
— what a container can <em>see</em> — and Lesson 62's capstone adds the other
three (use, root filesystem, do) to turn this isolated <em>view</em> into a
real, bounded, confined container. "A container is namespaces" is the
half-truth this homework corrects into "a container is namespaces + cgroups +
rootfs + confinement."
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 60 — cgroups v2 →](lesson-60-cgroups){: .btn .btn-primary }
