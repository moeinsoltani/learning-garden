---
title: "Lesson 61 — Container Images and overlayfs"
nav_order: 3
parent: "Phase 12: Containers & Resource Control"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 61: Container Images and overlayfs

## Concept

Namespaces gave the container its isolated view (Lesson 59); cgroups bounded
its resources (Lesson 60). The third pillar is its **filesystem** — where does
a container's root `/` come from? The answer combines two things you already
know: a **container image** (Lesson 46's "tarballs + metadata" hint) and
**overlayfs** (Lesson 39, built by hand).

A container image is far less magical than it looks:

```
   an image = an ordered stack of LAYERS + a manifest (JSON)

   layer 3 (your app)      ┐
   layer 2 (dependencies)  │  each layer = a tarball of filesystem changes,
   layer 1 (distro base)   ┘  named by the SHA256 of its content (L37 homework!)

   manifest.json: which layers, in what order, + config (entrypoint, env, ...)

   at RUN time: the layers become overlayfs lowerdirs (read-only, SHARED
                across all containers using them), plus a fresh writable
                upperdir per container:

        lower: layer1 + layer2 + layer3   (read-only, one copy on disk)
        upper: this container's writes    (private, tiny)
                        │
                   overlayfs merges them → the container's /
```

Two enormous consequences fall straight out of Lesson 39's overlayfs
mechanics: **sharing** (ten containers from one image share the read-only
lower layers — one copy on disk, one set of pages in the page cache, Lesson
20) and **copy-on-write** (a container writing to a lower-layer file copies it
up to its private upper — Lesson 39's copy-up, with the latency-spike and
deleted-doesn't-shrink gotchas that lesson's homework predicted).

Content-addressing (each layer named by its hash) gives dedup, integrity, and
cacheability for free — the exact git-object-store design of Lesson 37's
homework, applied to filesystem layers. This lesson makes it concrete: you'll
pull a real image, unpack its layers, and see they're just tarballs assembled
by overlayfs.

---

## How It Works

### The image format (OCI)

The **OCI image spec** standardizes it: an image is a *manifest* (JSON listing
layers by digest + a config blob) and the *layer blobs* (gzipped tarballs of
filesystem diffs). A *registry* (Docker Hub, ghcr.io) stores these
content-addressed blobs; `docker pull` fetches the manifest, then each layer
by its digest (skipping any already cached — content-addressing means
identical layers download once across all images). `skopeo` and `crane`
inspect/copy images without a running daemon — the lab uses them to dissect an
image into its raw parts.

### Assembly at run time

The runtime (containerd/runc, or podman) unpacks each layer tarball into a
directory, then mounts an overlay with those directories as `lowerdir`s
(stacked, read-only) and a fresh `upperdir`+`workdir` for the container's
writes (Lesson 39's exact `mount -t overlay` — you did it by hand). The merged
mount becomes the container's root via pivot_root (Lesson 40/62). `docker
inspect` shows the `GraphDriver.Data` with the actual lowerdir/upperdir paths
— the lab reveals them.

### Where the gotchas live (Lesson 39's homework, now real)

- A file deleted in a later image layer isn't gone — it's a whiteout; the
  bytes still occupy the lower layer (why `RUN rm -rf /cache` in a *later*
  Dockerfile line doesn't shrink the image — clean up in the *same* RUN).
- First write to a big lower-layer file triggers a full copy-up (the
  mysterious first-write latency; VOLUME/bind-mounts bypass overlayfs to
  avoid it).
- `docker commit` snapshots the whole upper layer — every temp file and
  copied-up artifact (why images are built from Dockerfiles, not committed
  from pets).

{: .note }
> **The build side: layers as a cache**
> Each Dockerfile instruction creates a layer; unchanged instructions reuse
> cached layers (content-addressing again). This is why ordering matters —
> put rarely-changing steps (install dependencies) <em>before</em>
> frequently-changing ones (COPY your code) so a code change only rebuilds the
> last layer. Layer caching is Lesson 20's "the second time is faster" applied
> to image builds — and mastering it is most of "why is my build slow / image
> huge."

---

## Lab

```bash
# ---- 0. tools: skopeo/crane for daemonless dissection (or use docker) ----
$ sudo apt install -y skopeo umoci 2>/dev/null | tail -1 || echo "(docker works too)"

# ---- 1. An image is a manifest + layer tarballs. Pull one to disk (no daemon) ----
$ cd /tmp && skopeo copy docker://alpine:latest oci:alpinelab:latest 2>/dev/null \
  || docker pull alpine:latest
$ ls -R /tmp/alpinelab 2>/dev/null | head -12
# blobs/sha256/<hashes>  index.json  oci-layout
#   ← content-addressed blobs (L37): the manifest + config + layer tarball(s)
$ cat /tmp/alpinelab/index.json 2>/dev/null | python3 -m json.tool | head -12
# points to a manifest by digest → which points to layers + config by digest

# ---- 2. A layer IS a tarball of files ----
$ LAYER=$(find /tmp/alpinelab/blobs -type f -size +1M 2>/dev/null | head -1)
$ file "$LAYER" 2>/dev/null                     # gzip compressed data / tar
$ tar tzf "$LAYER" 2>/dev/null | head -8        # bin/ etc/ lib/ ... a ROOT FS!
# alpine's whole userland is one ~3MB tarball. That's the "base layer."

# ---- 3. Multi-layer image: see the stack ----
$ skopeo inspect docker://python:3-alpine 2>/dev/null | python3 -c \
    "import json,sys; d=json.load(sys.stdin); print(len(d['Layers']),'layers'); [print(' ',l[:23],'...') for l in d['Layers']]" \
  2>/dev/null || echo "(inspect shows N layers, each a content-addressed tarball)"
# several layers: alpine base + python install + ... each shareable & cached

# ---- 4. Sharing proven: many "containers" from one image = one copy ----
$ command -v docker >/dev/null && {
    for i in 1 2 3; do docker run -d --name shr$i alpine sleep 300 >/dev/null; done
    echo "3 containers running from alpine:"
    docker ps --format '{{.Names}}' | grep shr
    echo "image on disk (ONE copy, shared lower layers):"
    docker system df | grep -i images
    docker inspect shr1 --format '{{.GraphDriver.Data.LowerDir}}' 2>/dev/null | tr ':' '\n' | head -2
    # ↑ the lowerdir = the shared read-only image layers (L39!)
    docker rm -f shr1 shr2 shr3 >/dev/null
  } || echo "(docker not present — concept: N containers share the lower layers)"

# ---- 5. overlayfs IS the mechanism (Lesson 39, now under a real container) ----
$ command -v docker >/dev/null && {
    docker run -d --name ov alpine sleep 300 >/dev/null
    docker inspect ov --format 'upper: {{.GraphDriver.Data.UpperDir}}' 2>/dev/null
    docker exec ov sh -c 'echo "container wrote this" > /newfile'
    UP=$(docker inspect ov --format '{{.GraphDriver.Data.UpperDir}}' 2>/dev/null)
    sudo ls "$UP" 2>/dev/null            # newfile is HERE — in the upper layer
    sudo cat "$UP/newfile" 2>/dev/null   # the container's write, on the host
    docker rm -f ov >/dev/null
  }
# the container's writes land in its private upperdir — exactly Lesson 39's
# hand-built overlay, now assembled automatically by the runtime.

# ---- 6. The whiteout gotcha, at image scale (L39 homework, real) ----
$ command -v docker >/dev/null && {
    printf 'FROM alpine\nRUN dd if=/dev/zero of=/big bs=1M count=50\nRUN rm /big\n' > /tmp/Dockerfile.bad
    docker build -q -t lab-bad -f /tmp/Dockerfile.bad /tmp >/dev/null 2>&1
    printf 'FROM alpine\nRUN dd if=/dev/zero of=/big bs=1M count=50 && rm /big\n' > /tmp/Dockerfile.good
    docker build -q -t lab-good -f /tmp/Dockerfile.good /tmp >/dev/null 2>&1
    docker images lab-bad lab-good --format '{{.Repository}}: {{.Size}}'
    # lab-bad: ~55MB  (the 50MB lives in an earlier layer + a whiteout!)
    # lab-good: ~5MB  (created & deleted in ONE layer — never shipped)
    docker rmi lab-bad lab-good >/dev/null 2>&1
  }
$ rm -rf /tmp/alpinelab /tmp/Dockerfile.*
```

---

## Further Reading

| Topic | Link |
|---|---|
| OCI Image Specification | <https://github.com/opencontainers/image-spec/blob/main/spec.md> |
| Overlayfs — kernel docs | <https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html> |
| `skopeo(1)` | <https://github.com/containers/skopeo> |
| Docker storage drivers (overlay2) | <https://docs.docker.com/storage/storagedriver/overlayfs-driver/> |
| Content-addressable storage | <https://en.wikipedia.org/wiki/Content-addressable_storage> |
| Dockerfile best practices | <https://docs.docker.com/develop/develop-images/dockerfile_best-practices/> |

---

## Checkpoint

**Q1.** Ten containers run from one 1 GiB image. Roughly how much disk do they
use before writing anything, and why?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Roughly <strong>1 GiB total</strong> — not 10 GiB. The image's layers are
mounted as overlayfs <em>lowerdirs</em>, which are <strong>read-only and
shared</strong>: all ten containers reference the same on-disk layer files,
one physical copy (Lesson 39/20's sharing). Each container adds only a fresh,
empty <em>upperdir</em> (its private writable layer) — a few KB of overhead
until it writes anything. So ten containers ≈ 1 GiB (the shared image) + ~10 ×
(tiny upper) ≈ 1 GiB. The saving compounds in RAM too: because the lower
layers are one set of files, reading them populates the page cache once
(Lesson 20) and all ten containers share those cached pages — ten containers
running the same binary share its code pages exactly like ten processes
running one program (Lesson 16's shared-library economics, now at image
scale). This is <em>the</em> reason containers are dense where VMs are not: a
VM duplicates the entire guest OS per instance (its own kernel, own memory —
virt track), while containers share the image's read-only bytes on disk and in
cache, paying real storage/memory only for each container's actual writes.
Content-addressing amplifies it further — if two <em>different</em> images
share a base layer (same alpine base), that layer is stored once across both,
and pulled once (Lesson 37's homework: identical content = identical hash =
one copy).
</details>

**Q2.** A developer runs `RUN rm -rf /var/cache/apt/*` as the last line of
their Dockerfile to shrink the image, but the image doesn't get smaller.
Explain using overlayfs layers, and give the correct fix.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Each Dockerfile instruction creates a new <em>layer</em> — a read-only diff
stacked on the previous ones (Lesson 39's lowerdirs). The cache files were
created in an <em>earlier</em> layer (the apt-get install RUN); the final
<code>rm</code> runs in a <em>new, later</em> layer, where deleting a
lower-layer file doesn't remove its bytes — it creates a <strong>whiteout</strong>
(Lesson 39: a marker saying "hide this name") in the new layer while the actual
cache files remain, in full, in the earlier read-only layer that's still part
of the image. So the image now contains the cache <em>plus</em> whiteout
metadata — same size (or slightly larger), the exact result the lab's
Dockerfile.bad reproduced. Layers are immutable and additive; you cannot make
a shipped layer smaller from a later one. Correct fix: create and delete the
cache <em>within the same layer</em> (same RUN) so the files never enter any
persisted layer — <code>RUN apt-get update && apt-get install -y foo && rm -rf
/var/cache/apt/*</code> as one instruction: the intermediate cache exists only
during that layer's build and the committed layer is the net result (installed
package, no cache). More broadly: for cleanup to shrink an image it must not
outlive its own layer — hence multi-stage builds (copy only the final
artifacts into a fresh minimal image, discarding all build detritus), and the
general rule that image size is determined by what each layer <em>persists</em>,
not by the final filesystem state. It's Lesson 39's homework as a daily
engineering reality.
</details>

**Q3.** Content-addressing (naming each layer by its content hash) gives
containers three properties "for free." Name them, and connect this design to
Lesson 37's git-object-store homework.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Three free properties: (1) <strong>Deduplication</strong> — identical content
produces an identical hash, so a layer shared by many images (the same distro
base, the same dependency set) is stored once on disk and in the registry, and
identical layers across different images collapse to one copy — the storage
efficiency of Q1 extended across the whole image library. (2)
<strong>Integrity / tamper-evidence</strong> — the hash <em>is</em> the name,
so a downloaded layer can be verified by re-hashing it: any corruption or
tampering changes the content and thus mismatches the expected digest, caught
automatically (the supply-chain security foundation — you can pin an image by
digest and be certain you got exactly those bytes). (3)
<strong>Cacheability</strong> — because a layer's identity is its content, a
<code>docker pull</code> can skip any layer already present locally (same hash
= same bytes, no need to re-download), and a <code>docker build</code> can
reuse cached layers for unchanged instructions — Lesson 20's "second time is
faster," made content-exact. This is precisely Lesson 37's homework about
git's object store: git names objects (blobs, trees, commits) by the SHA of
their content, giving it the same three properties — dedup (identical files
stored once), integrity (the hash verifies the content, so history can't be
silently altered), and cacheability (objects you have are never re-fetched).
Both systems independently arrived at content-addressed, immutable,
write-once storage because it's the right model for versioned, distributable
data: mutation is confined to small pointers (git refs / image tags) while the
bulk data is immutable and self-verifying. Container images are, structurally,
a git-like content-addressed store of filesystem layers — which is why the
same tooling instincts (pin by digest, not tag; understand what creates a new
object; cache aggressively) transfer directly, and why registries, like git
remotes, can dedup and verify without trusting the transport.
</details>

---

## Homework

Assemble a minimal container root filesystem by hand, bridging to Lesson 62's
capstone: pull alpine (lab 1), extract its single layer tarball into a
directory, then use Lesson 39's `mount -t overlay` to stack a read-only lower
(the extracted alpine) under a writable upper — producing a merged root you
could `chroot`/`pivot_root` into. Then explain what you've built relative to
what `docker run alpine` does, and precisely what's still missing to make it a
container (tie each gap to a lesson).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Steps: extract the base layer —
<code>mkdir -p /tmp/c/{lower,upper,work,merged}; tar xzf &lt;alpine-layer&gt;
-C /tmp/c/lower</code> (now /tmp/c/lower holds alpine's bin/etc/lib — a whole
userland); then <code>sudo mount -t overlay overlay
-o lowerdir=/tmp/c/lower,upperdir=/tmp/c/upper,workdir=/tmp/c/work
/tmp/c/merged</code> — /tmp/c/merged is now alpine's root filesystem, with
writes landing privately in upper (Lesson 39, verbatim). You could
<code>sudo chroot /tmp/c/merged /bin/sh</code> and get an alpine shell using
the host kernel. What you've built vs <code>docker run alpine</code>: you have
the <strong>image filesystem pillar</strong> — the read-only base layer shared
via overlayfs + a private writable upper, exactly how the runtime assembles a
container's root (Q1/lab 5). But it's a chroot, not a container — the gaps, each
a lesson: (1) <strong>No namespaces</strong> (Lesson 59) — the "container" shares
the host's PID space (ps sees everything), network (host interfaces), hostname,
IPC, and its "root" is still the host's uid 0; it needs unshare of
mnt/pid/net/uts/ipc/user to get an isolated view, and <strong>pivot_root</strong>
(Lesson 40/62) instead of chroot (which is escapable — chroot only changes
path lookup, not the mount reality). (2) <strong>No resource limits</strong>
(Lesson 60) — it can exhaust host CPU/memory/pids; needs a cgroup with
cpu.max/memory.max/pids.max. (3) <strong>No confinement</strong> (Phase 11) —
full syscall surface (needs seccomp, Lesson 56), full capabilities (needs
dropping, Lesson 11), no MAC profile (Lesson 57); a compromise has the whole
host kernel to attack. (4) <strong>No proper /proc, /sys, /dev</strong>
(Lessons 04/53) mounted inside the new mount namespace, no network plumbing
(veth/bridge — networking track). So this homework built one of the four
pillars (rootfs), and Lesson 62's capstone adds the other three — namespaces
for the view, pivot_root for the real root switch, cgroups for the limits,
caps+seccomp for the confinement — turning this overlay-mounted chroot into an
actual container, by hand, proving the thesis of the whole phase: a container
is not a kernel object, it's this assembly of primitives you've now built
every piece of.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 62 — Build a Container by Hand (capstone) →](lesson-62-build-a-container){: .btn .btn-primary }
