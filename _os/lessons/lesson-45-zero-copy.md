---
title: "Lesson 45 — Zero-Copy: sendfile and splice"
nav_order: 3
parent: "Phase 8: Event-Driven & Async I/O"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 45: Zero-Copy — sendfile and splice

## Concept

Serve a file over a socket the obvious way and count what actually happens
to the bytes:

```
  while (n = read(file_fd, buf, 64K))        the byte's journey:
      write(sock_fd, buf, n);
                                             disk ──DMA──▶ page cache      (0)
                                             page cache ──copy──▶ buf      (1)
                                             buf ──copy──▶ socket buffer   (2)
                                             socket buf ──DMA──▶ NIC       (0)

  TWO CPU copies + TWO syscalls per chunk — and your buffer added nothing:
  you never LOOKED at the bytes. You were a very expensive conveyor belt.
```

For a 10 GB file that's 20 GB of memcpy through your CPU and its caches
(evicting actual working data — Lesson 12's cache damage, self-inflicted),
plus 300K syscalls. **Zero-copy** APIs delete the conveyor belt:

- **`sendfile(out_fd, in_fd, offset, count)`** — "kernel: move file→socket
  yourself." Data goes page cache → socket **by reference** (the NIC DMAs
  straight from page-cache pages): zero CPU copies, one syscall for
  gigabytes. nginx's `sendfile on`, kafka's consumer fan-out, every static
  file server.
- **`splice(fd_in, fd_out, len)`** — the general engine: move data between
  an fd and a **pipe** without copying (pipes internally are rings of page
  *references* — Lesson 31's buffer, revealed!). Chain two splices
  (file→pipe, pipe→socket) for arbitrary fd pairs; `sendfile` is a splice
  convenience.
- **`vmsplice`** — gift user pages into a pipe; **`tee`** — duplicate pipe
  contents (references again!) without consuming; **`copy_file_range`** —
  file→file in-kernel (enables fs magic: server-side NFS copy, btrfs
  reflinks — CoW files, Lesson 39!).
- **`MSG_ZEROCOPY`** (networking track's territory) — zero-copy *send from
  user memory*, with completion notifications when the NIC is done reading
  your pages.

The through-line of the whole phase: Lesson 43 removed *waiting waste*,
Lesson 44 removed *crossing waste*, this lesson removes *copying waste*.

---

## How It Works

### Why the copies were ever there

`read()`'s contract requires your buffer to hold private, stable data —
generality costs a copy in each direction. Zero-copy works by *narrowing
the contract*: sendfile never shows you the bytes (fine — you didn't look),
splice moves ownership of page references through a pipe (the pipe's ring
holds `struct page` pointers + offsets, not bytes). The kernel keeps the
pages pinned until the consumer (NIC DMA) finishes — reference counting
(Lessons 36/37's pattern) instead of duplication.

### The limits that keep it honest

- sendfile's `in_fd` must be mmap-able (a *file*, not a socket); socket→
  socket needs the splice-through-a-pipe trick; socket→file works.
- **TLS breaks naive zero-copy**: encryption must transform every byte —
  the CPU must touch them after all. The fix is **kTLS** (kernel TLS,
  networking Lesson 70's neighbor): the socket encrypts, so
  sendfile→kTLS-socket stays zero-copy into the record layer — how modern
  CDNs serve HTTPS at line rate.
- Small transfers gain little (per-call overhead dominates — the wins are
  proportional to bytes moved), and if you must *inspect* data (parsing,
  filtering), you've re-bought the copy by definition — though
  `splice`+`tee` lets you duplicate a stream cheaply and inspect one arm.
- `mmap`+`write` is the halfway house (one copy removed, page-table costs
  added — Lesson 17); it lost to sendfile for serving, but wins when you
  need the bytes visible too.

### Seeing it

`strace` shows the syscall collapse (thousands of read/write → a few
sendfile). `perf stat` shows the story in hardware: cycles and
cache-references drop while bytes/s holds. The lab does both.

{: .note }
> **Where you've already used this**
> <code>cp --reflink</code> (btrfs, Lesson 39) is copy_file_range hitting
> CoW; qemu-img convert uses it when filesystems allow; Docker layer
> "copies" on overlayfs try reflinks first; and the kernel's NFS/SMB
> server-side-copy means "cp on a mount" may move zero bytes over the
> network. Zero-copy is less a feature than a design instinct: <em>who
> actually needs to touch these bytes?</em>

---

## Lab

```bash
# ---- 0. a big test file, cache-warm for fairness ----
$ dd if=/dev/urandom of=/tmp/payload bs=1M count=512 2>/dev/null
$ cat /tmp/payload > /dev/null                      # warm the page cache (L20)

# ---- 1. read/write vs sendfile: syscalls and time ----
$ cat > /tmp/serve.py << 'EOF'
import os, socket, sys, time

mode = sys.argv[1]
srv = socket.socket(); srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv.bind(("127.0.0.1", 9010)); srv.listen(1)

if os.fork() == 0:                                   # child: the client/sink
    c = socket.socket(); c.connect(("127.0.0.1", 9010))
    total = 0
    while (d := c.recv(1 << 20)): total += len(d)
    print(f"  received {total>>20} MB"); os._exit(0)

conn, _ = srv.accept()
fd = os.open("/tmp/payload", os.O_RDONLY)
size = os.fstat(fd).st_size
t = time.time()
if mode == "copy":
    while (buf := os.read(fd, 65536)):
        conn.sendall(buf)
else:                                                # sendfile
    off = 0
    while off < size:
        off += os.sendfile(conn.fileno(), fd, off, size - off)
elapsed = time.time() - t
conn.close(); os.wait()
print(f"{mode}: {elapsed*1000:.0f} ms  ({size/elapsed/1e9:.2f} GB/s)")
EOF
$ python3 /tmp/serve.py copy
$ python3 /tmp/serve.py sendfile
# sendfile typically 1.5-4x faster on loopback — and the gap is all CPU:

# ---- 2. count the work, not just the time ----
$ strace -cf python3 /tmp/serve.py copy 2>&1 | grep -E 'read|sendto|write' | head -3
# ~8192 reads + ~8192 sends for 512MB
$ strace -cf python3 /tmp/serve.py sendfile 2>&1 | grep -E 'sendfile' | head -2
# ~a handful of sendfile calls. The conveyor belt, retired.
$ perf stat -e cycles,cache-references python3 /tmp/serve.py copy 2>&1 | grep -E 'cycles|cache'
$ perf stat -e cycles,cache-references python3 /tmp/serve.py sendfile 2>&1 | grep -E 'cycles|cache'
# fewer cycles AND fewer cache refs for the same bytes: the copies were
# real CPU work polluting real caches (L12's damage, now measured).

# ---- 3. splice: the general plumbing, by hand ----
$ cat > /tmp/splice.py << 'EOF'
import os
# file → pipe → file, no bytes through userspace:
src = os.open("/tmp/payload", os.O_RDONLY)
dst = os.open("/tmp/payload.out", os.O_WRONLY | os.O_CREAT | os.O_TRUNC)
r, w = os.pipe()
total = 0
while True:
    n = os.splice(src, w, 1 << 20)         # file → pipe (page refs!)
    if n == 0: break
    while n:
        m = os.splice(r, dst, n)           # pipe → file
        n -= m
    total += 1 << 20
print(f"spliced ~{os.fstat(dst).st_size >> 20} MB through a pipe, 0 copies")
EOF
$ python3 /tmp/splice.py && cmp /tmp/payload /tmp/payload.out && echo "bytes identical"

# ---- 4. copy_file_range: file→file, possibly no data movement AT ALL ----
$ python3 - << 'EOF'
import os, time
src = os.open("/tmp/payload", os.O_RDONLY)
dst = os.open("/tmp/payload.cfr", os.O_WRONLY | os.O_CREAT | os.O_TRUNC)
size = os.fstat(src).st_size
t = time.time(); off = 0
while off < size:
    off += os.copy_file_range(src, dst, size - off)
print(f"copy_file_range: {(time.time()-t)*1000:.0f} ms")
# on btrfs/xfs with reflink support this can be near-instant (CoW metadata
# only — L39!); on ext4 it's still an in-kernel copy: no userspace transit.
EOF

# ---- 5. tee: duplicate a stream without consuming it ----
$ echo "one stream, two destinations" | tee >(md5sum > /tmp/sum1) | md5sum > /tmp/sum2
$ cat /tmp/sum1 /tmp/sum2               # same hash — tee(1) uses tee(2)'s idea
$ rm -f /tmp/payload* /tmp/serve.py /tmp/splice.py /tmp/sum[12]
```

---

## Further Reading

| Topic | Link |
|---|---|
| `sendfile(2)` man page | <https://man7.org/linux/man-pages/man2/sendfile.2.html> |
| `splice(2)` man page | <https://man7.org/linux/man-pages/man2/splice.2.html> |
| `copy_file_range(2)` man page | <https://man7.org/linux/man-pages/man2/copy_file_range.2.html> |
| Zero-copy (Wikipedia) | <https://en.wikipedia.org/wiki/Zero-copy> |
| `tee(2)` man page | <https://man7.org/linux/man-pages/man2/tee.2.html> |
| Kernel TLS — kernel docs | <https://www.kernel.org/doc/html/latest/networking/tls-offload.html> |

---

## Checkpoint

**Q1.** Count the user↔kernel copies for a 1 GB file served over a socket:
(a) read/write loop, (b) mmap+write, (c) sendfile. Where does each
eliminated copy go?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) Two copies per byte: page cache → user buffer (read), user buffer →
socket buffer (write) — 2 GB of memcpy for 1 GB served, plus the disk→cache
and cache→NIC DMA transfers that all variants share (DMA isn't CPU work).
(b) One copy: mmap maps the page-cache pages into your address space
(Lesson 20 — the cache pages ARE your "buffer", no read copy), but write()
still copies from those mapped pages into the socket buffer. The read-side
copy became page-table setup + faults (Lesson 17/19's costs — why mmap
loses for streaming despite the count). (c) Zero copies: sendfile hands the
socket layer <em>references</em> to the page-cache pages; the NIC DMAs
directly from them (with scatter-gather hardware); the "copy" became
refcount bookkeeping and pinned pages until transmission completes. Each
elimination converts CPU/memory-bandwidth work into either nothing or
metadata — and frees cache capacity for data someone actually inspects.
</details>

**Q2.** Why does plain TLS traditionally destroy sendfile's benefit, and how
does kTLS restore it? What does the answer teach about where zero-copy is
even possible?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
TLS must transform every payload byte (encrypt + MAC): userspace TLS
libraries need the file bytes <em>in user memory</em> to run the cipher —
so serving becomes read (copy) → encrypt (touch every byte) → write (copy):
zero-copy is structurally impossible because the application genuinely
consumes the data. kTLS moves the record layer into the kernel: after the
handshake (still userspace — the complex, non-performance-critical part),
the socket itself encrypts whatever passes through it. Now sendfile's
page-cache references flow into the kTLS socket, which encrypts once on the
way to the NIC (often with hardware offload doing even that — networking
Lesson 63's themes): the pipeline is again zero-copy <em>up to the
unavoidable transformation</em>, performed at the last layer that must
touch bytes anyway. The general principle: zero-copy is possible exactly
along path segments where no layer needs to <em>read or modify</em> the
data — every mandatory transformation (encryption, compression, parsing)
is a copy-equivalent; the design move is relocating transformations into
layers that were already touching the bytes (kernel socket, NIC hardware)
so the count of byte-touching stages hits its theoretical minimum of one.
</details>

**Q3.** splice moves data "through a pipe with no copies." Reconcile that
with Lesson 31's picture of a pipe as a 64 KB kernel byte buffer — what is a
pipe, really, and why did the splice designers choose the pipe as their
rendezvous object?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The revision: a pipe's buffer is not a byte array but a ring of
<em>(struct page reference, offset, length)</em> slots. Ordinary write()
fills fresh pages and adds references; ordinary read() copies out of them —
byte semantics preserved, so Lesson 31's model was observationally correct.
splice exploits the representation: file→pipe inserts references to
<em>existing page-cache pages</em> (no allocation, no copy); pipe→socket
hands those references onward; the "capacity" is slot count as much as
bytes. Why the pipe as hub: it already was the kernel's neutral,
process-visible buffer object with established fd semantics — blocking,
poll-ability, backpressure (Lesson 31's built-in flow control now
regulating reference flow!), permission-free rendezvous between unrelated
fds — and using it made splice composable: any fd pair connects through it,
tee() duplicates by copying <em>references</em>, vmsplice donates user
pages into the same structure. One pre-existing object, upgraded from
byte-mover to reference-mover, gave Linux a generic zero-copy plumbing
layer without inventing a new primitive — very much the Unix move: the
pipe turns out to have been the right abstraction twice, forty years apart.
</details>

---

## Homework

Kafka's original performance story: consumers reading recent messages are
served via sendfile from files that were just written by producers — and the
page cache makes it all RAM-speed. Assemble the full pipeline from Lessons
20/31/41/42/45: trace a message from producer `send()` to consumer bytes,
naming every copy, every cache interaction, and where fsync sits — then
explain why this design makes Kafka's performance *degrade gracefully* (not
cliff) when consumers fall behind.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Producer path: message arrives on a socket (NIC DMA → socket buffer; one
copy into Kafka's process for protocol parsing — unavoidable, it inspects
the data), Kafka appends to the active log segment with write() — one copy
user→page cache (Lesson 20), where the data is now DIRTY and, crucially,
<em>resident</em>. Durability: Kafka fsyncs lazily by default, relying on
replication instead — the flush (Lesson 41) happens by policy/interval, off
the hot path; the block layer (Lesson 42) sees large sequential merged
writes (append-only logs are the device's happy path). Consumer path
(caught-up consumer): the requested bytes are the pages <em>just
written</em> — still in the page cache — so sendfile(socket, segment_file)
attaches references and the NIC DMAs straight from the same physical pages
the producer's write filled: zero copies, zero disk reads; RAM-to-NIC at
line rate. The OS page cache is Kafka's actual serving tier; the JVM heap
stays small deliberately. Graceful degradation: a lagging consumer's reads
target older segments evicted from cache (Lesson 20/21's LRU) — those
sendfiles trigger real disk reads (readahead-friendly sequential ones), so
throughput for <em>that consumer</em> drops to disk speed while caught-up
consumers continue at RAM speed — per-consumer degradation, no shared
cliff. Contrast a design with an app-level cache and private buffers:
laggards would evict the hot working set and copy everything twice,
degrading everyone. The whole story is this phase compressed: let the
kernel's cache be the cache, let references replace copies, and align your
access pattern (append + sequential) with every layer's fast path.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 9 — Linking & Loading (Lesson 46: ELF and Static Linking) →](lesson-46-elf-static-linking){: .btn .btn-primary }
