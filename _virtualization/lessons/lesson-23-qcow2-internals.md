---
title: "Lesson 23 — qcow2 Internals: Backing Files and Copy-on-Write"
nav_order: 23
parent: "Phase 6: Storage"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 23: qcow2 Internals — Backing Files and Copy-on-Write

## Concept

qcow2 is the format that powers **templates, linked clones, and snapshots** — and
all three rest on one idea: a **backing file**. An overlay image stores only what
*changed*; everything unchanged is read from a read-only base underneath it.

```
   base.qcow2  (read-only golden image: OS installed)
        ▲ backing
        │
   overlay-A.qcow2   ← VM A: stores only A's writes
        ▲ backing
        │
   overlay-B.qcow2   ← (chains can stack: B reads through A then base)

   Read a block:  look in the top overlay; if not present, fall through
                  to the backing file; repeat down to the base.
   Write a block: ALWAYS goes into the top overlay (copy-on-write).
```

This is why you can deploy 50 VMs from one 5 GiB base image and consume almost no
extra space — each VM is a thin overlay over the shared, untouched base.

---

## How It Works

### The on-disk structure (briefly)

A qcow2 file maps the guest's linear address space to **clusters** (default 64 KiB)
via a two-level table:

- **L1 table** → points to **L2 tables** → point to **data clusters**.
- A guest block is found by walking L1 → L2 → cluster. If the L2 entry says
  "unallocated," the data isn't in this file.
- **Lazy allocation:** clusters (and L2 tables) are created only when first written,
  which is why qcow2 is thin.
- A **refcount table** tracks how many references each cluster has (used by internal
  snapshots).

### Backing files and the copy-on-write chain

An overlay records a **backing file** pointer in its header. The read/write rules:

- **Read:** check this image's L2 tables. If the cluster is allocated here, use it.
  If not, **fall through** to the backing file (and its backing file, recursively)
  until found. Unwritten regions ultimately read from the base.
- **Write:** always land in the **top** (active) overlay. The first write to a region
  copies nothing from the base — it just allocates a fresh cluster in the overlay
  (copy-on-write at cluster granularity). The base is **never modified**.

Create an overlay with:

```
   qemu-img create -f qcow2 -b base.qcow2 -F qcow2 overlay-A.qcow2
   #                         ^backing file  ^backing format (required, for safety)
```

### Managing the chain: rebase and commit

- **`qemu-img commit overlay-A.qcow2`** — merge the overlay's changes *down* into its
  backing file, then the overlay can be discarded. (Careful: this mutates the base —
  don't do it to a base shared by other overlays.)
- **`qemu-img rebase -b newbase.qcow2 overlay.qcow2`** — change which backing file an
  overlay points to (e.g. move a chain to a relocated base), reconciling differences.
  `rebase -b ""` flattens by pulling all data into the overlay so it stands alone.

### The danger: deleting the base

Because overlays depend on the base for all unchanged data, **deleting or modifying
a shared base corrupts every overlay built on it.** An overlay is *not*
self-contained — it's only the diff. If VM A's overlay falls through to a base that's
gone, those reads fail. To make an overlay independent you must flatten it
(`qemu-img convert` or `rebase -b ""`) first.

{: .note }
> **Linked clone = overlay; full clone = flattened copy**
> A "linked clone" is exactly an overlay over a shared base — instant to create, tiny
> on disk, but tied to the base's existence and slightly slower (reads may traverse
> the chain). A "full clone" is a standalone image (flattened, via convert) — bigger
> and slower to create, but independent and with no chain-traversal cost. Lesson 46
> builds the golden-image → clone workflow on top of this.

---

## Lab

```bash
# 1. Build a base image and put a marker "OS" file's worth of data in it.
$ qemu-img create -f qcow2 base.qcow2 10G
$ qemu-img info base.qcow2 | grep -E 'file format|backing'
file format: qcow2
# (no backing file — it's a standalone base)

# 2. Create TWO overlays over the same read-only base:
$ qemu-img create -f qcow2 -b base.qcow2 -F qcow2 overlay-A.qcow2
$ qemu-img create -f qcow2 -b base.qcow2 -F qcow2 overlay-B.qcow2

# 3. Inspect an overlay — note the backing-file pointer and tiny size:
$ qemu-img info overlay-A.qcow2
image: overlay-A.qcow2
file format: qcow2
virtual size: 10 GiB
disk size: 196 KiB                      ← only A's (zero) writes so far
backing file: base.qcow2
backing file format: qcow2

# 4. See the whole chain at once:
$ qemu-img info --backing-chain overlay-A.qcow2 | grep -E 'image|backing file:'
image: overlay-A.qcow2
backing file: base.qcow2
image: base.qcow2

# 5. Boot VM A on overlay-A and VM B on overlay-B; write inside each guest.
#    A's writes land ONLY in overlay-A, B's ONLY in overlay-B; base stays pristine:
$ md5sum base.qcow2          # record the base hash
$ # ...run both VMs, write data inside each, shut down...
$ md5sum base.qcow2          # UNCHANGED — base was never written
$ du -h overlay-A.qcow2 overlay-B.qcow2   # each grew by only its own writes

# 6. Flatten overlay-A into a standalone image (makes it base-independent):
$ qemu-img convert -O qcow2 overlay-A.qcow2 standalone-A.qcow2
$ qemu-img info standalone-A.qcow2 | grep backing || echo "no backing file — independent"
```

**Expected result:** Two overlays share one base; each VM's writes go only into its
own overlay while the base's checksum stays identical (proving it's untouched).
Flattening produces a standalone image with no backing file.

---

## Further Reading

| Topic | Link |
|---|---|
| qcow2 format | [Wikipedia — qcow](https://en.wikipedia.org/wiki/Qcow) |
| `qemu-img` (create -b, commit, rebase) | [qemu.org — qemu-img](https://www.qemu.org/docs/master/tools/qemu-img.html) |
| Copy-on-write | [Wikipedia — Copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write) |
| qcow2 internal spec | [qemu.org — qcow2 spec](https://www.qemu.org/docs/master/interop/qcow2.html) |
| Linked clones / overlays | [libvirt.org — Storage](https://libvirt.org/storage.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Two VMs share one read-only base image via separate overlays. Where does each VM's data go, and what happens to VM A if you delete the base?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Each VM writes only into its own overlay (copy-on-write): VM A's changed clusters land in overlay-A, VM B's in overlay-B, and the shared base is never modified. Reads of unchanged data fall through the overlay to the base. If you delete the base, VM A breaks: its overlay only contains the diff, so any read that needs to fall through to the (now-missing) base fails — the overlay is not self-contained. To survive base deletion you'd first have to flatten the overlay (qemu-img convert or rebase -b "") into a standalone image.
</details>

---

**Q2. When the guest writes to a block for the first time, what does qcow2 do, and is the base touched?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
On first write to a region, qcow2 allocates a fresh cluster in the *top* (active) overlay and writes the new data there, updating that overlay's L2 table to point to it. The base/backing file is never modified — it stays read-only. Subsequent reads of that region now find the cluster in the overlay and no longer fall through to the base. This cluster-granular copy-on-write is what keeps the base pristine and shareable across many overlays.
</details>

---

**Q3. What's the difference between `qemu-img commit` and `qemu-img rebase`, and which one mutates the backing file?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
qemu-img commit merges an overlay's changes *down into* its backing file and then the overlay can be discarded — so it mutates the backing file (dangerous if that base is shared by other overlays). qemu-img rebase changes which backing file an overlay points to (reconciling differences), or with -b "" flattens the overlay to stand alone; it modifies the overlay's backing pointer/data rather than committing changes into a shared base. So commit is the one that writes into the backing file.
</details>

---

## Homework

Build `base.qcow2`, create `overlay.qcow2` over it, boot a VM on the overlay and create a file inside the guest. Record `md5sum base.qcow2` before and after. Then run `qemu-img commit overlay.qcow2` and check the base's md5sum again. Explain what `commit` did and why you must NOT commit an overlay whose base is shared by other live overlays.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Before commit, the base's md5sum is unchanged by the guest's writes (they live in the overlay). After <code>qemu-img commit overlay.qcow2</code>, the base's md5sum changes because commit merged the overlay's clusters down into the base — the base now contains the guest's new file. You must not commit when the base is shared by other overlays because those overlays assume the base holds the *original* unchanged data; mutating it injects one VM's writes underneath all the others, corrupting their view (they'd suddenly "see" data they never wrote, or inconsistent state). Shared bases must stay immutable; only flatten/commit overlays whose base is private.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 24 — Block Device Models →](lesson-24-block-device-models){: .btn .btn-primary }
