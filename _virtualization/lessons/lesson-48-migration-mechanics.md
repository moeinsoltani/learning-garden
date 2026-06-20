---
title: "Lesson 48 — Live Migration Mechanics"
nav_order: 48
parent: "Phase 12: Snapshots, Backup & Migration"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 48: Live Migration Mechanics

## Concept

**Live migration** moves a *running* VM from one host to another with minimal — ideally
imperceptible — downtime. The CPU and devices are easy to move (small state); the hard
part is the **RAM**, which is large and *changing while you copy it*. Two strategies solve
this from opposite directions:

```
   PRE-COPY                              POST-COPY
   ────────                             ─────────
   copy RAM while VM runs on SOURCE,     switch to DEST first (tiny pause),
   re-copy pages that change, then        then fault missing pages over from
   brief stop-and-copy at the end         SOURCE on demand as guest touches them
   risk: a busy guest dirties pages       risk: if the network dies mid-migration,
   faster than you can send (never        the guest is split across hosts and
   converges)                             can DIE (pages stuck on the dead source)
```

The whole challenge is RAM that mutates faster than the link can transfer it — the
**convergence problem**.

---

## How It Works

### What moves

A VM's state = CPU registers (tiny) + device state (small) + **RAM** (large) + disk. CPU
and device state transfer in the final instant. Disk is handled by shared storage or
storage migration (Lesson 49). RAM is the problem child, and the migration strategy is
all about RAM.

### Pre-copy (the default)

1. **Iterative copy:** while the VM keeps running on the **source**, copy all RAM pages to
   the destination. The guest keeps dirtying pages as you go.
2. **Re-copy dirty pages:** send the pages that changed during the previous pass. Repeat —
   each pass copies a smaller "dirty set" (ideally).
3. **Stop-and-copy:** once the remaining dirty set is small enough to transfer within the
   **downtime target**, briefly pause the VM, copy the last pages + CPU/device state,
   then resume on the **destination**.

The VM runs on the source the whole time except a tiny final pause — so if anything fails,
the source VM is still intact (safe). The danger is **non-convergence**: a write-heavy
guest dirties pages faster than the network sends them, so the dirty set never shrinks and
migration never finishes.

### Post-copy

1. **Switch first:** after a minimal initial transfer, briefly pause and **resume the VM
   on the destination** immediately — before most RAM has been copied.
2. **Demand paging:** as the destination guest touches pages it doesn't have yet, those
   pages are **faulted in over the network from the source** on demand (plus background
   push of the rest).

Post-copy *guarantees* convergence (each page transfers exactly once, no re-dirtying to
chase) and bounds downtime — but it has a **catastrophic failure mode**: the running state
is split across two hosts mid-migration. If the network drops or the source dies before
all pages arrive, the destination guest is missing pages it can never get → the **VM is
lost** (can't roll back to source, which already handed off control).

### Convergence aids

For pre-copy's convergence problem:

- **Auto-converge:** throttle the guest's vCPUs (slow it down) so it dirties memory slower
  than the link can send — trading guest performance for progress.
- **Bandwidth/downtime tuning:** raise allowed migration bandwidth or the acceptable
  downtime target so the final stop-and-copy can complete.
- **Post-copy switch:** start pre-copy, and if it won't converge, **switch to post-copy**
  to force completion (common hybrid).
- **Compression / multifd / RDMA:** move pages faster.

{: .note }
> **Pre-copy vs post-copy in one line each**
> <strong>Pre-copy</strong>: keep running on the source, copy-and-recopy RAM until the
> dirty set is tiny, then a brief final pause — <em>safe</em> (source stays valid) but can
> <em>fail to converge</em> on write-heavy guests (risk: "pausing/looping forever").
> <strong>Post-copy</strong>: switch to the destination almost immediately and fault pages
> in on demand — <em>always converges</em> and bounds downtime, but if the network drops
> mid-migration the guest is split and can <em>die</em> (unrecoverable). Production often
> starts pre-copy and falls back to post-copy if needed.

---

## Lab

```bash
# (Full hands-on migration is Lesson 49; here we inspect the mechanics/knobs.)

# 1. See migration capabilities/parameters QEMU exposes (via QMP on a running VM):
$ virsh qemu-monitor-command web01 --pretty '{"execute":"query-migrate-capabilities"}'
{ "return": [
    {"capability": "auto-converge", "state": false},
    {"capability": "postcopy-ram",  "state": false},
    {"capability": "multifd",       "state": false},
    {"capability": "compress",      "state": false} ] }

# 2. Tunables that address convergence (set before/while migrating):
$ virsh migrate-setmaxdowntime web01 300        # target <=300 ms final pause
$ virsh migrate-setspeed web01 10000            # cap at 10000 MiB/s

# 3. Enable auto-converge (throttle the guest to help pre-copy converge):
$ virsh migrate web01 qemu+ssh://dest/system --live --auto-converge   # (Lesson 49)

# 4. Watch convergence in real time during a migration (the dirty set shrinking):
$ virsh domjobinfo web01
Job type:         Unbounded
Data processed:   12.300 GiB
Data remaining:   0.512 GiB       ← shrinking dirty set = converging
Memory processed: 12.300 GiB
Memory remaining: 0.512 GiB
Dirty rate:       45 MiB/s        ← how fast the guest dirties pages
Iteration:        7

# 5. Force post-copy if pre-copy stalls (start live, then switch):
$ virsh migrate web01 qemu+ssh://dest/system --live --postcopy &
$ virsh migrate-postcopy web01     # trigger the switch to post-copy phase
```

**Expected result:** `query-migrate-capabilities` shows the available strategies
(auto-converge, postcopy-ram, multifd). `domjobinfo` lets you watch "Data remaining" and
the dirty rate, making convergence (or non-convergence) visible, and you can enable
auto-converge or switch to post-copy to force completion.

---

## Further Reading

| Topic | Link |
|---|---|
| QEMU migration | [qemu.org — Migration](https://www.qemu.org/docs/master/devel/migration/main.html) |
| Pre-copy / post-copy | [qemu.org — Postcopy](https://www.qemu.org/docs/master/devel/migration/postcopy.html) |
| Live migration (concept) | [Wikipedia — Live migration](https://en.wikipedia.org/wiki/Live_migration) |
| libvirt migration | [libvirt.org — Migration](https://libvirt.org/migration.html) |
| Auto-converge / convergence | [qemu.org — Migration features](https://www.qemu.org/docs/master/devel/migration/main.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Explain pre-copy vs post-copy. Which risks the VM "pausing forever" on a write-heavy guest, and which risks the VM dying if the network drops mid-migration?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Pre-copy keeps the VM running on the source while iteratively copying RAM and re-copying pages that get dirtied, then does a brief final stop-and-copy once the dirty set is small. Post-copy switches the VM to the destination almost immediately after a minimal transfer, then faults missing pages in from the source on demand. Pre-copy is the one that risks "pausing/looping forever" (non-convergence) on a write-heavy guest that dirties pages faster than the link can send them. Post-copy is the one that risks the VM dying if the network drops mid-migration, because the running state is split across hosts and the destination can be left missing pages it can never retrieve (with no valid source to fall back to).
</details>

---

**Q2. Why is RAM the hard part of live migration, while CPU and device state are easy?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
CPU registers and device state are small and quiescent at the moment of transfer, so they copy almost instantly in the final handoff. RAM is large (gigabytes) and, critically, it keeps *changing while you copy it* — the guest is still running and dirtying pages. So you can't just snapshot it once; you must either repeatedly re-copy the changing pages (pre-copy, risking non-convergence) or move execution first and fetch pages on demand (post-copy, risking a split-state failure). The mutation-under-copy of large memory is what makes RAM the central challenge.
</details>

---

**Q3. What is auto-converge, and what trade-off does it make?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Auto-converge throttles the guest's vCPUs during pre-copy migration, deliberately slowing the guest so it dirties memory more slowly than the network can transfer pages. This helps the dirty set shrink so the migration can converge and complete. The trade-off is guest performance: while throttled, the application inside the VM runs slower. It's a way to force convergence on write-heavy guests without resorting to post-copy, at the cost of temporarily degrading the guest until migration finishes.
</details>

---

## Homework

On a running VM, run `virsh domjobinfo` during a (test) migration or read `query-migrate-capabilities`. Identify the "Dirty rate" or the available capabilities. Then reason about a database VM under heavy write load: would plain pre-copy likely converge? Describe two specific things you'd enable/tune to get it to complete, and the downside of each.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A heavily-writing database VM likely will NOT converge under plain pre-copy: its high dirty rate means each iteration re-dirties pages about as fast as they're sent, so "Data remaining" never drops below the threshold for the final stop-and-copy. Two things to enable/tune: (1) <strong>auto-converge</strong> — throttle the guest's vCPUs so it dirties slower than the link sends; downside: the database runs slower (degraded performance) until migration finishes. (2) Switch to / enable <strong>post-copy</strong> — move execution to the destination and demand-page the rest, which guarantees convergence and bounds downtime; downside: if the network drops mid-migration the VM can be lost, since its state is split across hosts with no source fallback. (Also valid: raise migration bandwidth/downtime target, or use multifd/compression to move pages faster — downsides being network/CPU usage and a longer acceptable pause.)
</details>
