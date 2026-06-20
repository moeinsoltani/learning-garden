---
title: "Lesson 02 — The Host Is Just Another Namespace"
nav_order: 2
parent: "Phase 1: Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 02: The Host Is Just Another Namespace

## Concept

Most people think of the host machine as the "real" network, and namespaces as
isolated bubbles inside it. This is the wrong mental model. Drop it now.

The correct picture is this: **every network stack on the machine — including the
host's — lives inside a namespace.** The host's networking is just the *initial*
namespace, the one the kernel created at boot. It has no special status. It is
one apartment in a building, not the building itself.

```
┌─────────────────────────────────────────────────────────┐
│                       Linux Kernel                      │
│                                                         │
│  ┌─────────────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Initial (host)  │  │   ns1    │  │   ns2    │       │
│  │                 │  │          │  │          │       │
│  │ eth0, lo        │  │ lo       │  │ lo       │       │
│  │ routes, sockets │  │ (empty)  │  │ (empty)  │       │
│  └─────────────────┘  └──────────┘  └──────────┘       │
│       ↑                                                 │
│  This is what people call "the host network" —          │
│  it is just namespace #1, created at boot.              │
└─────────────────────────────────────────────────────────┘
```

The reason the initial namespace has `eth0` and working routes is simply that the
kernel wired the physical NIC into it at boot. There is nothing stopping you from
moving an interface into a different namespace — and in later lessons, you will do
exactly that.

---

## What Makes the Initial Namespace Special

Only one thing: it exists first. Everything else follows from that.

| Property | Initial namespace | Any other namespace |
|---|---|---|
| Created by | Kernel at boot | You, with `ip netns add` |
| Has physical NIC (`eth0`) | Yes — kernel puts it here | No — must be explicitly moved |
| Visible in `ip netns list` | **No** | Yes |
| Can be deleted | No | Yes |
| Special kernel privileges | None | None |

The initial namespace is invisible in `ip netns list` because it has no name —
it was never created by the `ip netns` tool. It is the namespace every process
starts in unless moved elsewhere.

---

## Packet Visibility Across Namespaces

Packets do not cross namespace boundaries on their own. A frame arriving on `eth0`
in the initial namespace stays there. Processes in `ns1` will never see it.

This is not a firewall rule. It is not a routing decision. It is a deeper isolation:
the network stacks are completely separate kernel objects. There is no path for a
packet to travel between them without a virtual interface explicitly connecting the
two namespaces.

```
 Initial namespace          ns1
 ┌──────────────┐           ┌──────────────┐
 │ eth0         │           │ lo           │
 │              │    ???    │              │
 │ ping 10.0.0.1│  ──────▶  │              │
 └──────────────┘    no     └──────────────┘
                   path
                   exists
```

This is why a container cannot reach the host's network by default — and why Docker
must explicitly create a virtual cable (a veth pair) between the container's namespace
and the host's bridge. You will build this yourself in Lesson 7.

---

## How It Works

Every process in Linux has a reference to a network namespace stored in its task
struct. When you call any networking syscall — `socket()`, `bind()`, `connect()`,
`sendmsg()` — the kernel looks up that reference and operates entirely within that
namespace's context.

You can see this directly via the `/proc` filesystem. Every process has a
`/proc/<PID>/ns/net` entry that is a symlink pointing to its network namespace.
Two processes pointing to the same [inode](https://en.wikipedia.org/wiki/Inode) are in the same namespace.

{: .note }
> **What is an inode?**
> An inode is a unique ID number the kernel assigns to every file, directory, and
> kernel object (including namespaces). It is the object's identity — independent
> of any name. When you see `net:[4026531992]` in the symlink output, that number
> is the inode of your network namespace. If two processes show the same inode
> number, they point at the exact same kernel object — same namespace. Different
> number means different namespace.

```bash
# Your shell's namespace (a symlink to an inode like net:[4026531992])
ls -la /proc/$$/ns/net

# PID 1 (systemd/init) is in the initial namespace
ls -la /proc/1/ns/net

# Both show the same inode number → same namespace → you are in the initial namespace
```

{: .note }
> **What is `$$`?**
> `$$` is a special shell variable that expands to the PID of the current shell.
> So `/proc/$$/ns/net` is shorthand for `/proc/<your-shell-PID>/ns/net` — the kernel
> exposes each process's namespace membership there. You can verify it with `echo $$`.

When you run `sudo ip netns exec ns1 bash`, the kernel changes the new process's
namespace reference to point at ns1's inode before executing bash. The process now
lives in a different room.

---

## Lab

```bash
# Step 1: confirm you are in the initial namespace
# Your shell and PID 1 should show the same namespace inode
$ ls -la /proc/$$/ns/net
lrwxrwxrwx ... /proc/1234/ns/net -> 'net:[4026531992]'

$ ls -la /proc/1/ns/net
lrwxrwxrwx ... /proc/1/ns/net -> 'net:[4026531992]'

# The inode numbers match — same namespace

# Step 2: create two namespaces
$ sudo ip netns add ns1
$ sudo ip netns add ns2

# Step 3: confirm each has a different namespace inode
$ sudo ip netns exec ns1 ls -la /proc/$$/ns/net
lrwxrwxrwx ... -> 'net:[4026532241]'   # different number

$ sudo ip netns exec ns2 ls -la /proc/$$/ns/net
lrwxrwxrwx ... -> 'net:[4026532352]'   # yet another number

# Step 4: show that ns1 cannot see host interfaces
$ sudo ip netns exec ns1 ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN ...
# Only lo — eth0 is invisible because it belongs to the initial namespace

# Step 5: show that ns2 cannot see ns1 either
$ sudo ip netns exec ns2 ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN ...
# Also only lo — ns2 has no visibility into ns1 at all

# Step 6: list all network namespaces on the system
$ lsns -t net
# Shows all namespaces with their inodes and which PIDs are in them
# The initial namespace appears here with PID 1

# Compare with ip netns list — it only shows named namespaces (created via ip netns add)
$ ip netns list
# ns1 and ns2 appear here because you created them with ip netns add
# The initial namespace does NOT appear — it was never registered as a named namespace

# Clean up
$ sudo ip netns delete ns1
$ sudo ip netns delete ns2
```

**What to observe:** three different inode numbers — one for the initial namespace,
one for ns1, one for ns2. Completely separate kernel objects. No shared state.

{: .note }
> **`lsns -t net` vs `ip netns list`**
> `ip netns list` only shows namespaces created with `ip netns add` — it reads
> `/var/run/netns/` and only finds named, registered namespaces. The initial
> namespace never appears there.
>
> `lsns -t net` reads `/proc` directly and finds every network namespace on the
> system — named or not, including those created by Docker, Kubernetes, or any
> other tool. If you ever run a Docker container and check `ip netns list`, the
> container's namespace won't show up — but `lsns -t net` will find it.

---

## Further Reading

| Topic | Link |
|---|---|
| Linux namespaces overview | [man7.org — namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html) |
| Network namespaces | [man7.org — network_namespaces(7)](https://man7.org/linux/man-pages/man7/network_namespaces.7.html) |
| `lsns` — list namespaces | [man7.org — lsns(8)](https://man7.org/linux/man-pages/man8/lsns.8.html) |
| The `/proc` filesystem | [man7.org — proc(5)](https://man7.org/linux/man-pages/man5/proc.5.html) |
| Linux namespaces (deep dive) | [Wikipedia — Linux namespaces](https://en.wikipedia.org/wiki/Linux_namespaces) |
| Inodes | [Wikipedia — inode](https://en.wikipedia.org/wiki/Inode) |

---

## Checkpoint

**Q1. The initial namespace has `eth0`. A new namespace has only `lo`. What is the
actual reason for this difference?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The kernel wired the physical NIC into the initial namespace at boot — that is simply where eth0 was placed when the system started. There is nothing structurally special about the initial namespace that gives it eth0. Interfaces can be moved between namespaces; eth0 just happens to start in the initial one.
</details>

---

**Q2. Why can't a process in `ns1` see packets arriving on the host's `eth0`?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because namespaces are completely separate network stacks — separate kernel objects. A packet arriving on eth0 in the initial namespace is processed entirely within that namespace's context. There is no mechanism for it to leak into ns1. This is not a firewall rule or a routing decision; it is structural isolation. To get a packet from the initial namespace into ns1, you need a virtual interface explicitly connecting the two.
</details>

---

**Q3. You run `sudo ip netns exec ns1 bash`. What does the kernel actually do
to put your shell inside ns1?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The kernel changes the new process's namespace reference — the pointer in its task struct — to point at ns1's namespace inode instead of the initial namespace inode. From that point on, every networking syscall the process makes operates within ns1's network stack. The process is still visible in the host's process table; only its network context has changed.
</details>

---

## Homework

Run this on your VM and explain what you see:

```bash
$ lsns -t net
```

What does each column mean? How many network namespaces are listed?
Does the initial namespace appear? How can you tell?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`lsns -t net` lists all network namespaces currently active on the system. Columns: NS (inode number), TYPE (net), NPROCS (number of processes using it), PID (one example PID in that namespace), USER (owner), COMMAND (that process's command).

The initial namespace appears as the entry containing PID 1 (systemd/init). You can confirm it is the initial namespace because its inode matches what you saw in `/proc/1/ns/net`. There will typically be one entry per namespace — one for the initial namespace, and one each for ns1 and ns2 if they are still running.
</details>
