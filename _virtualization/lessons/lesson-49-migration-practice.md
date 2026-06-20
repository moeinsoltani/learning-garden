---
title: "Lesson 49 — Live Migration in Practice"
nav_order: 49
parent: "Phase 12: Snapshots, Backup & Migration"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 49: Live Migration in Practice

## Concept

Lesson 48 was the theory; now you actually move a VM. In practice live migration has three
hard requirements that, once met, make `virsh migrate` almost anticlimactic:

```
   1. STORAGE     the disk must be reachable by BOTH hosts
                  → shared storage (NFS/Ceph)  OR  copy it (--copy-storage-all)
   2. CPU         the destination CPU must be COMPATIBLE with the guest's
                  → use a common baseline model (Lesson 17), not host-passthrough
   3. NETWORK     the hosts must reach each other on the migration port(s)
                  → libvirt/QEMU migration ports + firewall open
```

Miss any one and migration fails before it starts. Get them right and the VM hops hosts
with milliseconds of downtime.

---

## How It Works

### virsh migrate

The core command:

```
   virsh migrate --live web01 qemu+ssh://dest-host/system
```

Useful flags:

- **`--live`** — migrate while running (the default goal; without it the VM is paused).
- **`--p2p`** — peer-to-peer: libvirt on the source talks directly to libvirt on the dest
  (vs the client orchestrating).
- **`--tunnelled`** — tunnel the migration data through the libvirtd connection (e.g. over
  the SSH channel) instead of a direct QEMU-to-QEMU socket — simpler firewalling, some
  overhead.
- **`--copy-storage-all`** / **`--copy-storage-inc`** — also migrate the disk when storage
  is *not* shared (full or incremental).
- **`--auto-converge`**, **`--postcopy`** — the convergence aids from Lesson 48.
- **`--persistent`** / **`--undefinesource`** — define the domain on the dest / remove it
  from the source.

### Requirement 1: storage

The guest's disk must exist on the destination. Two ways:

- **Shared storage** — both hosts mount the *same* disk (NFS, Ceph/RBD, iSCSI, a clustered
  FS). Then migration only moves RAM/CPU state — fast, the normal production setup.
- **Storage migration** — if storage isn't shared, add `--copy-storage-all` so QEMU copies
  the disk image to the destination *during* the migration. Works without shared storage
  but moves far more data (the whole disk) and takes longer.

This is *why* live migration "usually requires shared storage or an explicit storage-copy
flag": the destination must have the disk, either by sharing it or by copying it.

### Requirement 2: CPU compatibility (the baseline-CPU problem)

The destination CPU must be able to provide every feature the guest is using (Lesson 17).
If the source guest booted with `host-passthrough` exposing AVX-512 and the destination
lacks AVX-512, the guest would lose an instruction mid-flight → migration is refused. The
fix is to run VMs with a **common baseline CPU model** that every host in the pool
supports. Compute one with:

```
   virsh hypervisor-cpu-baseline ...    # or the older: virsh cpu-baseline host-cpus.xml
```

Pin VMs to that model so any VM migrates to any host. This is the production reason you
avoid host-passthrough in migration clusters.

### Requirement 3: network/firewall

libvirt uses a control connection (e.g. SSH for `qemu+ssh://`) plus QEMU migration data
ports (a configurable range, often `49152–49215`). These must be open between hosts. With
`--tunnelled`/`--p2p` the data can ride the libvirt connection, simplifying firewalling.

### Verifying

`virsh domjobinfo web01` during migration shows progress (data remaining, dirty rate);
after, the domain appears `running` on the destination and gone (or shut off) on the
source. Confirm zero data loss by checking the guest never rebooted (uptime) and its
application state persisted.

{: .note }
> **Why shared storage OR a copy flag, and why CPU must be compatible**
> Live migration moves the VM's <em>execution</em> (RAM + CPU/device state) but the guest's
> disk must be present on the destination — so either both hosts share the same storage
> (only RAM moves) or you pass <code>--copy-storage-all</code> to ship the disk too. And
> because the guest's code is literally executing on the destination CPU after the handoff,
> that CPU must provide every feature the guest was using; otherwise an instruction
> vanishes mid-execution. A common baseline CPU model (not host-passthrough) guarantees
> compatibility across the cluster.

---

## Lab

```bash
# Setup: two hosts (or two libvirt URIs) + SHARED storage (e.g. NFS pool, Lesson 41)
# holding web01's disk, reachable at the SAME path on both.

# 1. Confirm the destination can provide a compatible CPU (baseline, Lesson 17):
$ virsh hypervisor-cpu-baseline /tmp/src-cpu.xml /tmp/dst-cpu.xml 2>/dev/null
# (or ensure the VM uses a named model both hosts support, not host-passthrough)

# 2. Confirm shared storage: the disk path resolves on BOTH hosts:
$ virsh -c qemu+ssh://dest/system pool-list      # the shared pool is active on dest

# 3. LIVE migrate with shared storage (only RAM/CPU move):
$ virsh migrate --live --persistent --verbose \
    web01 qemu+ssh://dest/system
Migration: [ 87 %]

# 4. Watch progress from the source while it runs:
$ virsh domjobinfo web01
Job type:         Unbounded
Data remaining:   0.300 GiB
Memory remaining: 0.300 GiB
Dirty rate:       20 MiB/s

# 5. After completion — running on dest, gone/shut off on source:
$ virsh list                              # source: web01 no longer running
$ virsh -c qemu+ssh://dest/system list    # dest: web01 running
$ ssh web01 uptime                        # guest uptime CONTINUOUS = no reboot, no loss

# 6. WITHOUT shared storage — copy the disk during migration:
$ virsh migrate --live --copy-storage-all --verbose \
    web01 qemu+ssh://dest/system
# (moves the whole disk too; slower, but no shared storage needed)

# 7. If a write-heavy guest won't converge, add the Lesson 48 aids:
$ virsh migrate --live --auto-converge --postcopy \
    web01 qemu+ssh://dest/system
```

**Expected result:** With shared storage and a compatible CPU, `virsh migrate --live`
moves the running VM to the destination; `domjobinfo` shows the dirty set shrinking, and
afterward the guest is running on the destination with continuous uptime (no reboot, no
data loss). Without shared storage, `--copy-storage-all` migrates the disk too.

---

## Further Reading

| Topic | Link |
|---|---|
| libvirt migration | [libvirt.org — Migration](https://libvirt.org/migration.html) |
| `virsh migrate` | [libvirt.org — virsh](https://libvirt.org/manpages/virsh.html) |
| CPU model baseline | [libvirt.org — CPU model comparison](https://libvirt.org/formatdomaincaps.html) |
| Networking Lesson 17 (CPU models) | [vCPU models]({{ '/virtualization/lessons/lesson-17-vcpu-models.html' | relative_url }}) |
| Shared storage / NFS pool (Lesson 41) | [Storage pools]({{ '/virtualization/lessons/lesson-41-storage-pools.html' | relative_url }}) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Why does live migration usually require shared storage *or* an explicit storage-copy flag, and why must the destination CPU be "compatible"?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Live migration moves the VM's execution state (RAM + CPU/device state), but the guest's disk must also be present on the destination. So either both hosts share the same storage (NFS/Ceph/iSCSI), in which case only RAM/CPU move, or you pass --copy-storage-all so QEMU copies the disk to the destination during migration. Without one of these, the destination has no disk to run from. The destination CPU must be compatible because after the handoff the guest's code literally executes on that CPU; if it lacks a feature the guest was using (e.g. AVX-512 the guest got via host-passthrough), that instruction would be missing mid-execution. A common baseline CPU model across the pool guarantees the destination can provide everything the guest needs.
</details>

---

**Q2. What's the difference between using shared storage and `--copy-storage-all`, and when do you need the latter?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With shared storage, both hosts already access the same disk (same NFS/Ceph/iSCSI path), so migration only transfers RAM and CPU/device state — fast and the normal production approach. --copy-storage-all is used when storage is NOT shared: QEMU copies the entire disk image to the destination during the migration in addition to the RAM. You need it whenever the destination can't reach the guest's disk on its own; the downside is moving far more data (the whole disk), making migration slower and heavier.
</details>

---

**Q3. How do you confirm a completed migration had zero data loss / no guest disruption?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
After migration, verify the domain is "running" on the destination and no longer running on the source (virsh list on both). Then check from inside the guest that it never rebooted — e.g. <code>uptime</code> shows continuous uptime spanning the migration — and that application/session state persisted (open connections, in-memory data still present). During the migration, virsh domjobinfo should have shown the dirty set converging to a small final stop-and-copy. Continuous guest uptime plus intact application state confirms the live migration moved the running VM without a restart or data loss.
</details>

---

## Homework

Set up two libvirt endpoints (two hosts, or `qemu:///system` and a remote `qemu+ssh://`) with shared storage for one VM's disk. Run `virsh migrate --live --verbose` and watch `virsh domjobinfo`. Record the final downtime and confirm the guest's `uptime` is continuous afterward. Then explain what specifically would have failed if (a) the disk were on local-only storage and (b) the source VM used `host-passthrough` and the destination CPU was an older generation.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With shared storage and a compatible CPU, the migration completes with only milliseconds of downtime and the guest's uptime is continuous (it never rebooted) — confirming a true live move. (a) If the disk were on local-only storage, the destination couldn't access the guest's disk, so plain --live migration would fail/refuse; you'd have to add --copy-storage-all to ship the disk during migration. (b) If the VM used host-passthrough and the destination CPU were an older generation lacking some feature the guest acquired (e.g. a newer instruction set), the migration would be refused because the destination CPU can't provide every feature the running guest depends on — the fix is to run the VM on a common baseline CPU model both hosts support (Lesson 17) rather than host-passthrough.
</details>
