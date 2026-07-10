---
title: "Lesson 53 — Kernel Modules and udev"
nav_order: 4
parent: "Phase 10: Boot & Init"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 53: Kernel Modules and udev

## Concept

Two loose ends from the boot story: the kernel loaded *modules* to reach your
disk (Lesson 51), and device files like `/dev/vda` *appeared* somehow. This
lesson closes both — how the kernel grows at runtime, and how hardware
becomes usable files.

**Kernel modules** (`.ko`) are kernel code loaded *after* boot into the
running kernel — same privilege as the built-in kernel (ring 0, Lesson 01),
just not compiled in. They're why one generic `vmlinuz` supports thousands of
devices without being gigantic: drivers load on demand.

```
   built-in (=y):  in vmlinuz, always present   — the essentials
   module   (=m):  a .ko file, loaded when needed — everything else
                        │  modprobe / auto-loaded on device detection
                        ▼
                   runs IN the kernel, ring 0. A bug = kernel oops (L54!)
```

**udev** answers "how does `/dev/vda` appear?" When the kernel detects
hardware (at boot or hotplug), it emits a **uevent** — a message
"device added: block/vda, here are its attributes." The userspace daemon
`systemd-udevd` receives these, applies **rules** (create the device node,
name it predictably, set permissions, load firmware, run helpers), and the
device becomes a usable file. The chain — interrupt → driver → uevent →
udev → /dev node — is the payoff of the whole course, from Lesson 05's
interrupts to Lesson 37's device files.

You've met the pieces: `/sys` (Lesson 04) is the device model udev reads;
device nodes are special inodes (Lesson 37); kvm's module parameters were
Lesson 09 (virt track); and Lesson 55 will make a device node's *other* end
(the driver's read/write handlers) yours to write.

---

## How It Works

### Managing modules

`lsmod` (loaded modules + their use counts + dependencies), `modinfo NAME`
(description, parameters, dependencies, signature), `modprobe NAME`
(load + dependencies, from `/lib/modules/$(uname -r)/`), `modprobe -r` /
`rmmod` (unload — fails if in use, the refcount in lsmod). `insmod` is the
raw version (one file, no dep resolution — rarely used directly).
**Module parameters** tune drivers at load: `modprobe kvm ignore_msrs=1`
(virt Lesson 09) or via `/etc/modprobe.d/*.conf` for persistence, and
`/sys/module/NAME/parameters/` reads current values (Lesson 04's /sys again).
Blacklisting (`/etc/modprobe.d/blacklist.conf`) stops autoloading a
problematic driver.

### Autoloading and the modalias magic

You rarely `modprobe` manually — modules autoload. Hardware advertises IDs
(PCI vendor/device, USB IDs); the kernel forms a **modalias** string; udev
(and depmod's `modules.alias` table) maps it to the right module and loads
it. Plug a USB device → uevent with its modalias → udev requests the matching
driver → module loads → new uevents for the resulting device → /dev node.
`cat /sys/devices/.../modalias` shows a device's ID string; this is why Linux
"just works" with hardware it's never seen configured.

### udev rules and predictable naming

Rules (`/lib/udev/rules.d/`, override in `/etc/udev/rules.d/`) match device
attributes and take actions: `KERNEL=="sd*", MODE="0660", GROUP="disk"`;
persistent symlinks (`/dev/disk/by-uuid/...`, `by-id/...` — how fstab and
GRUB name disks stably despite enumeration order, Lesson 38); predictable NIC
names (`enp3s0` instead of racy `eth0` — the networking track's interface
names come from here); triggering scripts on plug/unplug. `udevadm monitor`
watches events live; `udevadm info /dev/X` dumps everything udev knows about
a device.

{: .note }
> **Module signing and lockdown**
> Loading a module = injecting ring-0 code, so it's a prime attack/rootkit
> vector. Secure Boot (Lesson 50) enforces <strong>module signature
> verification</strong> — unsigned modules are rejected (you'll hit this
> building your own in Lesson 54: sign it, or the machine must be in
> "insecure"/SB-off mode). Kernel <em>lockdown</em> mode further restricts
> even root from loading unsigned modules or reading kernel memory — the
> boot chain of trust (Lesson 50) extended into runtime.

---

## Lab

```bash
# ---- 1. What's loaded, and the dependency graph ----
$ lsmod | head -10
# Module          Size  Used by
# virtio_net     ...      0
# virtio_blk     ...      0        ← your disk driver (loaded by initramfs!)
$ lsmod | awk '$3 > 0 {print $1" used by "$3" others: "$4}' | head -5
# the "Used by" column is a REFCOUNT — can't unload while > 0

# ---- 2. Inspect a module: parameters and metadata ----
$ modinfo virtio_blk 2>/dev/null | grep -E 'description|depends|parm' | head
$ modinfo ext4 | grep -E 'filename|signer|sig_key' | head -3   # signing info
$ ls /sys/module/ | head -8                     # every loaded module in /sys
$ ls /sys/module/*/parameters/ 2>/dev/null | head    # live tunable params

# ---- 3. Load and unload a harmless module, watch the refcount ----
$ sudo modprobe dummy 2>/dev/null && echo "loaded dummy (a fake netdev driver)"
$ lsmod | grep dummy
$ sudo ip link add dummy-lab type dummy         # USE it → refcount rises
$ lsmod | grep dummy                            # Used by: 1 now
$ sudo rmmod dummy 2>&1 | tail -1               # FAILS — in use!
$ sudo ip link del dummy-lab                     # release it
$ sudo modprobe -r dummy && echo "unloaded cleanly (refcount hit 0)"

# ---- 4. The autoload magic: modalias ----
$ cat /sys/class/net/*/device/modalias 2>/dev/null | head -2 || \
    find /sys/devices -name modalias 2>/dev/null | head -1 | xargs cat
# pci:v00001AF4d00001000sv...   ← the ID string that SELECTED the driver
$ D=$(find /sys/devices -name modalias 2>/dev/null | head -1)
$ modprobe -R $(cat $D) 2>/dev/null || echo "modalias → module resolution"
# (modprobe -R resolves an alias to the module name it would load)

# ---- 5. Watch uevents FLOW as devices change (the live chain) ----
$ sudo udevadm monitor --udev &                 # start watching
$ sleep 0.5; sudo ip link add utest type dummy  # create a "device"
$ sleep 0.5; sudo ip link del utest             # remove it
$ sleep 0.5; sudo kill %1 2>/dev/null
# UDEV [..] add   /devices/virtual/net/utest (net)   ← the uevent, live
# UDEV [..] remove /devices/virtual/net/utest (net)
# THIS is how /dev nodes and interface names appear — a message + a daemon

# ---- 6. udev's knowledge of a real device ----
$ udevadm info /dev/$(lsblk -dno NAME | head -1) 2>/dev/null | head -15
# every property udev used: DEVNAME, ID_SERIAL, symlinks (by-uuid/by-id)...
$ ls -l /dev/disk/by-id/ 2>/dev/null | head -3
# the STABLE names udev created — what fstab/GRUB use instead of racy /dev/sdX

# ---- 7. Device nodes are special inodes (Lesson 37, closing the loop) ----
$ ls -l /dev/vda /dev/null /dev/zero 2>/dev/null | head -3 || ls -l /dev/sda /dev/null
# brw-rw---- ... 254,  0 vda    ← 'b' = block device; 254,0 = major,minor
# crw-rw-rw- ...   1,  3 null    ← 'c' = char device; the numbers route to
#                                   the driver (Lesson 55 makes one!)
$ stat -c '%F: major=%t minor=%T' /dev/null /dev/zero
```

---

## Further Reading

| Topic | Link |
|---|---|
| Loadable kernel module | <https://en.wikipedia.org/wiki/Loadable_kernel_module> |
| `modprobe(8)` man page | <https://man7.org/linux/man-pages/man8/modprobe.8.html> |
| udev (Wikipedia) | <https://en.wikipedia.org/wiki/Udev> |
| `udevadm(8)` man page | <https://man7.org/linux/man-pages/man8/udevadm.8.html> |
| `udev(7)` — rules syntax | <https://man7.org/linux/man-pages/man7/udev.7.html> |
| Kernel module signing — docs | <https://www.kernel.org/doc/html/latest/admin-guide/module-signing.html> |

---

## Checkpoint

**Q1.** You plug in a USB disk and `/dev/sdb` appears. List the chain of
events from interrupt to device node.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) Plugging in raises a hardware <strong>interrupt</strong> (Lesson 05) on
the USB controller; its driver handles it and detects a new device on the
bus. (2) The device advertises its IDs; the kernel reads them, forms a
<strong>modalias</strong> string, and — if the needed driver (usb-storage,
then the SCSI disk driver) isn't loaded — the autoload mechanism
(kmod/udev consulting modules.alias) <strong>loads the modules</strong>
(Lesson 51's mechanism, at runtime). (3) The driver probes the disk, and the
kernel creates the in-kernel device object, assigning a <strong>major/minor
number</strong> and emitting a <strong>uevent</strong> ("add block/sdb")
into the kernel→userspace netlink channel. (4) <strong>systemd-udevd</strong>
receives the uevent, matches it against its <strong>rules</strong>, and
<strong>creates the device node</strong> <code>/dev/sdb</code> (a special
inode — Lesson 37 — carrying that major/minor, which route future
reads/writes to the driver), sets ownership/permissions, and creates the
stable symlinks (<code>/dev/disk/by-id/...</code>, by-uuid after the
filesystem is probed). (5) Further uevents fire for each partition
(<code>sdb1</code>), and if configured, udev rules or systemd mount units
auto-mount it. From electrical signal to a mountable file: interrupt →
driver → module load → uevent → udev → node — every arrow a subsystem this
course covered.
</details>

**Q2.** A kernel module can't be unloaded — `rmmod` reports it's in use. How
does the kernel know, and what are the implications of the fact that a module
runs at the same privilege as the built-in kernel?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Each module carries a <strong>reference count</strong> (the "Used by" column
in lsmod): incremented whenever something depends on it — another module
that needs its symbols, or an active use of its resources (a mounted
filesystem holds its fs module; a configured interface holds its net driver;
an open device node holds the driver — the lab's dummy example). rmmod
refuses while the count is nonzero, because unloading code that's still in
use would leave dangling function pointers and instant kernel corruption:
you must release all users first (unmount, delete the interface, close the
device). Privilege implication: a module executes in <strong>ring 0 with the
full kernel's power</strong> (Lesson 01) — no memory protection from the rest
of the kernel, no syscall boundary, direct hardware access. So (a) a bug in a
module is a <em>kernel</em> bug: a null dereference is an oops/panic that can
take down the whole system (Lesson 54 makes this visceral), not a segfault
confined to one process; (b) loading a module is injecting arbitrary
privileged code — the ultimate rootkit vector — which is why module
<em>signing</em> and Secure Boot / lockdown exist (only signed modules load),
and why "can load kernel modules" (CAP_SYS_MODULE — Lesson 11) is
effectively root-over-the-kernel and is dropped in containers. The
convenience of extending the kernel at runtime is inseparable from the danger
of running unvetted code at the highest privilege.
</details>

**Q3.** Predictable naming: your fstab uses `/dev/disk/by-uuid/...` and your
NIC is `enp3s0`, not `/dev/sda2` and `eth0`. Explain what problem udev is
solving here and how, connecting to a specific failure the old names caused.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The problem is <strong>enumeration nondeterminism</strong>: kernel device
names like <code>sda</code>/<code>sdb</code> and <code>eth0</code>/
<code>eth1</code> are assigned in the order devices are <em>detected</em> at
boot — which depends on driver load timing, probe races, and hardware
init order, and can therefore <em>change between boots</em> or when hardware
is added. Old failures: a system with two disks boots, and after a kernel
update (changing module load order) the disk that was <code>sda</code> is now
<code>sdb</code> — fstab's <code>/dev/sda2 / ext4</code> now mounts the wrong
disk as root, or fails: unbootable, from a name that silently moved. Networking
was worse: a server's <code>eth0</code>/<code>eth1</code> swapping meant the
firewall rules and IP config applied to the wrong physical port — a security
and connectivity break with no code change (the classic "added a NIC, lost
the network"). udev solves it by naming devices from <em>stable
attributes</em> instead of detection order: disks get symlinks under
<code>/dev/disk/by-uuid/</code> (the filesystem's UUID — Lesson 38 — travels
with the data), <code>by-id/</code> (the hardware serial), <code>by-path/</code>
(the physical connection); NICs get names derived from PCI topology
(<code>enp3s0</code> = PCI bus 3 slot 0 — fixed by where the card physically
is). fstab, GRUB, and network config reference these invariants, so the same
storage/port is found regardless of enumeration. udev is applying database
thinking to devices: never key on a volatile ordinal; key on an intrinsic,
stable identity — the same principle as Lesson 37's inode-vs-name and
Lesson 06's pidfd-vs-recycled-PID. Stable identity over positional
addressing, everywhere.
</details>

---

## Homework

Write a udev rule (real, in `/etc/udev/rules.d/`) that fires when you add or
remove a `dummy` network interface (from lab 3/5) and logs a message with the
device name — then trigger it and verify via the system log. Explain each
field of your rule (match keys vs action), why udev rules are a safer
extension point than modifying kernel drivers, and one real-world use of
rule-triggered scripts (hint: think backups, or the networking track's
interface setup).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Rule (<code>/etc/udev/rules.d/99-lab.rules</code>):
<pre>
SUBSYSTEM=="net", KERNEL=="dummy*", ACTION=="add", \
    RUN+="/bin/sh -c 'logger udev-lab: added %k'"
SUBSYSTEM=="net", KERNEL=="dummy*", ACTION=="remove", \
    RUN+="/bin/sh -c 'logger udev-lab: removed %k'"
</pre>
Reload + trigger: <code>sudo udevadm control --reload</code>, then
<code>sudo ip link add dtest type dummy</code> and
<code>sudo ip link del dtest</code>; verify with
<code>journalctl -t udev-lab</code> or <code>grep udev-lab
/var/log/syslog</code>. Fields: <strong>match keys</strong> (<code>==</code>)
filter which events fire the rule — <code>SUBSYSTEM=="net"</code> (only
network devices), <code>KERNEL=="dummy*"</code> (name glob),
<code>ACTION=="add"/"remove"</code> (the uevent type); <strong>action
keys</strong> (<code>+=</code>, <code>=</code>) do things —
<code>RUN+=</code> queues a program, others set <code>NAME=</code>,
<code>SYMLINK+=</code>, <code>MODE=</code>, <code>OWNER=</code>;
<code>%k</code> is the kernel device name (one of many substitutions).
Why safer than modifying drivers: udev rules run in <strong>userspace</strong>
(the udevd daemon) — a broken rule logs an error or fails a script, it does
<em>not</em> crash the kernel the way a driver bug does (Q2 / Lesson 54);
they're declarative, hot-reloadable, and per-machine config rather than code
you compile and sign; and they hook the well-defined uevent interface instead
of the unstable in-kernel driver API (Lesson 46's stable-ABI theme —
uevents are the stable extension point, driver internals are not). Real uses:
running a backup or sync script when a specific external drive is plugged
(match its <code>by-id</code> serial); setting up a network interface —
applying addresses, bringing up bonds/bridges, or naming — when a NIC appears
(the networking track's interface config is largely udev-triggered);
adjusting permissions so a plugged USB serial adapter is usable by a
non-root group; loading firmware or spinning down idle disks. The pattern:
udev turns "hardware changed" into "run my policy" without anyone polling and
without touching kernel code — device management as event-driven userspace,
the same architecture as the rest of modern Linux.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 11 — Kernel Interfaces & Security (Lesson 54: Write a Kernel Module) →](lesson-54-kernel-module){: .btn .btn-primary }
