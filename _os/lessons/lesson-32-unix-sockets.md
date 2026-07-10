---
title: "Lesson 32 — Unix Domain Sockets"
nav_order: 2
parent: "Phase 6: Inter-Process Communication"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 32: Unix Domain Sockets

## Concept

Pipes are perfect for one-way, parent-child streams. But real local services
need more: bidirectional conversations, many clients connecting to one
server, message boundaries, credentials. The tool for all of it — used by
Docker (`/var/run/docker.sock`), systemd, X11/Wayland, PostgreSQL, MySQL,
your ssh-agent — is the **Unix domain socket** (UDS).

It's the sockets API you know from networking — `socket`, `bind`, `listen`,
`accept`, `connect` — with the network removed:

```
   TCP socket to 127.0.0.1:5432          Unix socket to /run/pg.sock
   ──────────────────────────           ───────────────────────────
   loopback "wire": packets through      no packets at all: the kernel
   the whole network stack — IP,         copies bytes buffer-to-buffer
   TCP state machine, checksums,         between the two sockets. done.
   conntrack, nftables (your whole
   networking track applies!)            addressing: a FILE PATH
   addressing: IP + port                 access control: file PERMISSIONS
   anyone local may try to connect       + peer credentials (who connected?)
```

Three superpowers TCP-on-localhost can't match:

1. **Filesystem-native access control**: the socket is a file (`srwxrw----`);
   directory and socket permissions decide who may connect. Docker's entire
   privilege model is "who can write to docker.sock" (the `docker` group).
2. **Peer credentials**: the server can ask the kernel *who* connected —
   PID, UID, GID (`SO_PEERCRED`) — authenticated by the kernel, unforgeable.
   This is how systemd and D-Bus authorize without passwords.
3. **File descriptor passing**: a process can send *an open fd* to another
   process over a UDS (`SCM_RIGHTS`) — handing over a file, socket, or pipe
   as a living capability. This one deserves its own section.

---

## How It Works

### Stream and datagram flavors

`SOCK_STREAM` — connection-oriented, ordered, reliable byte stream (like TCP
without TCP); `SOCK_DGRAM` — datagrams with preserved message boundaries
(like UDP but reliable and in-order — no loss on a single kernel!);
`SOCK_SEQPACKET` — connections *and* boundaries. Boundaries matter: Lesson
31's PIPE_BUF framing dance disappears — one send, one recv, whole message.

There's also the **abstract namespace** (`@name`, a leading NUL): sockets
with no filesystem presence at all — no file to clean up, but also *no file
permissions* (anyone in the network namespace may connect — a real security
footgun; several CVEs are exactly this).

### FD passing: capabilities over sockets

`sendmsg` with an `SCM_RIGHTS` control message carries fds; the kernel
*installs* them into the receiver's fd table (renumbered — receiver's fd 7
may be sender's fd 3; both refer to the same open file description —
Lesson 36's three layers make this precise). Why it's profound: fds are
unforgeable kernel handles, so passing one *transfers a capability* —
"here is the client connection, you handle it" (load-balancing across
worker processes: nginx, systemd socket activation), "here is the file you
may access" (privilege separation: a root broker opens, an unprivileged
worker uses — sshd's architecture, Lesson 11's endgame). None of this is
possible over TCP: an fd is meaningless outside its kernel.

### Where the bytes go

No IP, no checksums, no conntrack, no MTU: sendmsg copies into the peer
socket's receive buffer; the peer's read copies out (two copies, same as
pipes). Latency ~2× better than loopback TCP, throughput often 2–5× — but
the *architectural* wins (permissions, creds, fd passing) usually matter
more than the speed.

{: .note }
> **`ss -x` and friends**
> <code>ss -xlp</code> lists listening Unix sockets with owning processes —
> the local-services map of your machine. <code>lsof -U</code> shows all UDS
> endpoints. Try both on any box: the number of Unix sockets quietly running
> your desktop/server is always a surprise.

---

## Lab

```bash
# ---- 1. Census: who's listening on Unix sockets right now? ----
$ ss -xlp 2>/dev/null | head -8
# systemd's private bus, journald streams, dbus, ssh-agent…
$ ls -l /run/dbus/system_bus_socket /run/systemd/private 2>/dev/null
# srwxrwxrwx / srw------- ← 's' type; PERMISSIONS doing security work

# ---- 2. A server and client, with peer credentials ----
$ cat > /tmp/uds_server.py << 'EOF'
import socket, struct, os
path = "/tmp/lab.sock"
try: os.unlink(path)
except FileNotFoundError: pass
srv = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
srv.bind(path)                      # creates the file
os.chmod(path, 0o660)               # filesystem ACL = connection ACL
srv.listen()
print("listening on", path)
conn, _ = srv.accept()
pid, uid, gid = struct.unpack("3i",
    conn.getsockopt(socket.SOL_SOCKET, socket.SO_PEERCRED, 12))
print(f"peer: pid={pid} uid={uid} gid={gid}  ← kernel-attested, unforgeable")
print("client says:", conn.recv(100).decode())
conn.send(b"hello " + str(pid).encode())
EOF
$ python3 /tmp/uds_server.py &
$ sleep 0.5
$ python3 -c "
import socket, os
c = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
c.connect('/tmp/lab.sock')
c.send(f'I am pid {os.getpid()}'.encode())
print('server replied:', c.recv(100).decode())"
$ wait
# the server LEARNED the client's identity from the kernel — no tokens needed.
# This exact mechanism authorizes you to systemd, D-Bus, and docker.sock.

# ---- 3. The crown jewel: pass an OPEN FILE DESCRIPTOR ----
$ cat > /tmp/fdpass.py << 'EOF'
import socket, os, array

parent, child = socket.socketpair(socket.AF_UNIX, socket.SOCK_STREAM)
if os.fork() == 0:                                  # CHILD: receives an fd
    parent.close()
    msg, anc, flags, addr = child.recvmsg(100, socket.CMSG_SPACE(4))
    fd = array.array("i", anc[0][2]).tolist()[0]    # kernel installed it!
    print(f"child: received fd {fd}, reading from it:")
    print(" ", os.read(fd, 100).decode().strip())
    os._exit(0)
child.close()                                       # PARENT: opens + sends
fd = os.open("/etc/hostname", os.O_RDONLY)          # child may lack this right!
parent.sendmsg([b"gift"],
    [(socket.SOL_SOCKET, socket.SCM_RIGHTS, array.array("i", [fd]))])
os.wait()
EOF
$ python3 /tmp/fdpass.py
# child: received fd 4, reading from it:
#   yourhostname
# An OPEN FILE crossed a process boundary. The receiver never called open() —
# it received the CAPABILITY. (sshd, systemd socket activation, nginx workers.)

# ---- 4. Datagram flavor: message boundaries survive ----
$ python3 - << 'EOF'
import socket, os
a, b = socket.socketpair(socket.AF_UNIX, socket.SOCK_DGRAM)
a.send(b"first"); a.send(b"second"); a.send(b"third")
for _ in range(3):
    print(b.recv(4096))     # three sends = three recvs, boundaries intact
EOF
# b'first' b'second' b'third' — no framing protocol needed (contrast pipes!)

# ---- 5. Speed: UDS vs loopback TCP (both directions of the trade) ----
$ python3 - << 'EOF'
import socket, time, os
def bench(make_pair, label):
    a, b = make_pair()
    buf = b"x" * 65536
    t = time.time(); total = 0
    if os.fork() == 0:
        for _ in range(2000): a.sendall(buf)
        os._exit(0)
    while total < 2000 * 65536:
        total += len(b.recv(1 << 20))
    os.wait()
    mb = total / 1e6 / (time.time() - t)
    print(f"{label}: {mb:.0f} MB/s")
def uds(): return socket.socketpair()
def tcp():
    s = socket.socket(); s.bind(("127.0.0.1", 0)); s.listen()
    c = socket.socket(); c.connect(s.getsockname())
    return c, s.accept()[0]
bench(uds, "unix socket "); bench(tcp, "loopback tcp")
EOF
# UDS typically 1.5-4x faster — no IP/TCP machinery in the path.

$ rm /tmp/uds_server.py /tmp/fdpass.py /tmp/lab.sock 2>/dev/null
```

---

## Further Reading

| Topic | Link |
|---|---|
| `unix(7)` — Unix sockets overview | <https://man7.org/linux/man-pages/man7/unix.7.html> |
| Unix domain socket (Wikipedia) | <https://en.wikipedia.org/wiki/Unix_domain_socket> |
| `sendmsg(2)` / SCM_RIGHTS | <https://man7.org/linux/man-pages/man3/cmsg.3.html> |
| `socketpair(2)` man page | <https://man7.org/linux/man-pages/man2/socketpair.2.html> |
| systemd socket activation | <https://www.freedesktop.org/software/systemd/man/latest/systemd.socket.html> |

---

## Checkpoint

**Q1.** Why can a Unix socket pass an open file descriptor when a TCP socket
can't — what actually crosses, and what does the kernel do on arrival?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An fd is not data — it's an index into a process's fd table pointing at a
kernel open-file object (Lesson 36). What SCM_RIGHTS transfers is a
<em>reference to that kernel object</em>: the kernel, mediating both ends of
the UDS, takes the sender's open-file reference and installs a new entry in
the receiver's fd table pointing to the same object (bumping its refcount) —
the number may differ; the capability is identical, including file offset and
open mode. Over TCP the two ends may be different kernels — there is no
shared object table to reference; and even locally, TCP's byte-stream
machinery has no channel for kernel-mediated object transfer, only for bytes.
Consequence: UDS fd passing is <em>capability delegation</em> — unforgeable,
revocable-by-close, permission-check-free at use time (the check happened at
open) — the primitive under privilege separation and socket activation.
</details>

**Q2.** Docker's entire remote API is a Unix socket at
`/var/run/docker.sock`, mode `srw-rw---- root:docker`. Analyze this design:
what's elegant, and why is adding a user to the `docker` group equivalent to
giving them root?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Elegant: authentication and authorization are delegated entirely to
mechanisms the kernel already enforces — connecting requires write permission
on the socket file, so "who may control Docker" is managed with
chown/chgrp/usermod like any file ACL, auditable with <code>ls -l</code>;
optional SO_PEERCRED gives the daemon the caller's attested identity; no
passwords, tokens, or TLS on the local path. The root-equivalence: the socket
talks to a daemon running as root that will, on request, start a container
with <code>-v /:/host --privileged</code> — i.e., mount the host filesystem
into a root process you control. Write access to the socket = ability to
issue that request = root, one command away. The lesson generalizes: an IPC
endpoint's security level is the <em>privilege of the actions behind it</em>,
not of the transport — file permissions gate the door, but what's inside the
room is the real classification (rootless Docker/podman restructure the room
itself).
</details>

**Q3.** When would you choose loopback TCP over a Unix socket for two
processes on one machine — name three legitimate reasons?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <strong>Location transparency</strong>: the peer may move to another host
someday (or already runs in both configurations) — TCP makes local vs remote
a config change, not a code path; UDS-only code needs a second transport
implementation later. (2) <strong>Tooling parity</strong>: TCP gets the whole
networking-track toolbox — tcpdump/wireshark capture, nftables policy,
conntrack, proxies, load balancers; UDS traffic is invisible to packet tools
(debugging means strace, not pcap). (3) <strong>Ecosystem constraints</strong>:
libraries/clients that only speak host:port (many database drivers, browsers
for local dev servers, anything in a language/framework without UDS support),
or containers where sharing a socket file across mount namespaces is more
awkward than a port. The speed argument runs the other way, and the security
argument usually does too (fd passing, peer creds, file ACLs) — so UDS is the
default for same-host forever-local services, TCP for anything with a future
across machines.
</details>

---

## Homework

systemd "socket activation": systemd creates and listens on a service's
socket itself, starts the service only when the first client connects, and
hands the *listening socket* to the service as an inherited fd. Using this
lesson plus Lessons 07/31: explain the full mechanism (how does the fd get
into the service? what must the service do differently from binding its own
socket?), and list two operational benefits this design buys.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Mechanism: systemd (PID 1) creates the socket, binds the path/port, and
listens. On first connection it forks/execs the service — and the listening
fd rides ordinary <strong>fd inheritance across fork+exec</strong> (Lesson 07:
fds survive exec unless O_CLOEXEC). Convention (sd_listen_fds): the fd is
placed at number 3 (SD_LISTEN_FDS_START), with <code>LISTEN_FDS=1
LISTEN_PID=$$</code> in the environment so the service can verify the fds are
really for it. The service therefore skips socket/bind/listen entirely and
goes straight to <code>accept()</code> on fd 3. (For multi-fd or on-demand
per-connection modes, SCM_RIGHTS passing appears too — Accept=yes services
receive the <em>connected</em> socket.) Benefits: (1) <strong>lazy start +
no lost connections</strong> — services start on demand (memory saved), and
because the kernel queues connections on the listening socket from boot,
clients connecting during startup/restart just wait in the backlog instead of
getting refused: zero-downtime restarts for free; (2) <strong>privilege and
ordering decoupling</strong> — PID 1 binds the privileged port (&lt;1024,
Lesson 11) so the service runs fully unprivileged, and dependencies relax:
anything can connect "to the service" the moment systemd holds the socket,
long before the service exists — collapsing boot-order graphs (this is how
systemd parallelized boot).
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 33 — Shared Memory →](lesson-33-shared-memory){: .btn .btn-primary }
