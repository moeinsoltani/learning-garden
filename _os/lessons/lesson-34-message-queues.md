---
title: "Lesson 34 — Message Queues and D-Bus"
nav_order: 4
parent: "Phase 6: Inter-Process Communication"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 34: Message Queues and D-Bus

## Concept

Pipes move bytes; sockets move bytes-or-datagrams between two endpoints; shm
moves nothing (you visit it). One niche remains: **discrete messages, with
priorities, decoupled in time** — the producer and consumer don't need to run
simultaneously, and urgent messages jump the line. That's the **POSIX message
queue** (`mq_*`):

```
   mq_send(q, msg, prio 31)  ──▶ ┌──────────────────────┐
   mq_send(q, msg, prio 0)   ──▶ │  prio 31: [urgent]   │ ──▶ mq_receive:
                                 │  prio 5:  [normal]   │     ALWAYS the
   sender may exit now —         │  prio 0:  [bulk][bulk]│     highest prio,
   messages persist in the       └──────────────────────┘     oldest first
   kernel until read (or reboot)
```

Named like shm objects (`/name`, visible under `/dev/mqueue`), kernel-backed,
with hard bounds (max messages, max size — misconfiguration fails loudly, a
feature), blocking or non-blocking ends, and even async notification
(`mq_notify` — "signal me or start my thread when a message lands").

Honesty first: plain mqueues are the *least used* of this phase's tools in
modern services (sockets + explicit framing won). But their *idea* —
message-oriented, prioritized, decoupled — grew two enormous families you'll
touch constantly:

- **D-Bus** — the Linux desktop/system *message bus*: not point-to-point but
  a hub where services own names (`org.freedesktop.NetworkManager`), export
  objects with typed methods/signals/properties, and any authorized client
  can call or subscribe. systemd, NetworkManager, desktop notifications,
  power management — the control plane of a modern Linux box *is* D-Bus
  (transported, fittingly, over Unix sockets from Lesson 32).
- The **broker pattern** at datacenter scale — RabbitMQ/Kafka/SQS are this
  lesson's concepts with a network address (the homework maps them).

---

## How It Works

### mqueue mechanics worth knowing

`mq_open("/q", O_CREAT, 0600, &attr)` with attr = {mq_maxmsg, mq_msgsize} —
both capped by `/proc/sys/fs/mqueue/*`. Each `mq_send` is one message
(boundaries preserved, like SOCK_DGRAM); receive *must* offer a buffer ≥
msgsize or fail (no partial reads — a message is atomic). Priorities 0–31,
strictly ordered. The queue descriptor is select/poll-able on Linux (it's
secretly an fd — the "everything is an fd" convergence again). Persistence:
kernel lifetime — survives both processes exiting, dies at reboot or
`mq_unlink` (contrast: pipes die with their fds; files survive reboot;
mqueues sit exactly between).

### D-Bus in one screen

Two buses per machine: the **system bus** (root services, policy-controlled
in XML — who may own which name, call which method) and a **session bus**
per login (desktop apps). Concepts: *names* (`org.freedesktop.login1` —
ownership arbitrated by the bus daemon), *object paths*
(`/org/freedesktop/login1`), *interfaces* with typed **methods** (call,
get reply), **signals** (broadcast events — subscribe, don't poll!), and
**properties** (get/set attributes). The type system is real (signatures
like `a{sv}`) — and `busctl` lets you explore all of it interactively: the
lab does. Under the hood: Unix socket, peer credentials for identity
(Lesson 32's SO_PEERCRED doing authorization work), optional kernel fast
paths. When you `systemctl restart foo`, systemctl makes a D-Bus method call
to PID 1. You've been a D-Bus user all along.

{: .note }
> **Choosing across the whole phase (the IPC decision table)**
> Parent→child byte stream: <strong>pipe</strong>. Local client/server,
> creds, fd-passing: <strong>Unix socket</strong>. Big shared state, chatty:
> <strong>shm + Phase 5</strong>. Discrete prioritized hand-offs, decoupled
> lifetimes: <strong>mqueue</strong> (or a socket + your own framing).
> Service control plane / broadcasts on one machine: <strong>D-Bus</strong>.
> Cross-machine: none of these — network sockets and real brokers.

---

## Lab

```bash
# ---- 1. POSIX mqueue: priorities and time-decoupling ----
$ python3 - << 'EOF'
import ctypes, os
rt = ctypes.CDLL("librt.so.1", use_errno=True)

class Attr(ctypes.Structure):
    _fields_ = [("flags", ctypes.c_long), ("maxmsg", ctypes.c_long),
                ("msgsize", ctypes.c_long), ("curmsgs", ctypes.c_long)]

attr = Attr(0, 10, 256, 0)
q = rt.mq_open(b"/lab", os.O_CREAT | os.O_RDWR, 0o600, ctypes.byref(attr))

# send OUT of priority order...
for prio, msg in [(0, b"bulk-report"), (31, b"EMERGENCY"), (5, b"normal-task")]:
    rt.mq_send(q, msg, len(msg), prio)
print("sent: bulk(0), EMERGENCY(31), normal(5) — in that order")

# ...receive in STRICT priority order
buf = ctypes.create_string_buffer(256)
prio = ctypes.c_uint()
for _ in range(3):
    n = rt.mq_receive(q, buf, 256, ctypes.byref(prio))
    print(f"  received prio {prio.value:2d}: {buf.value[:n].decode()}")
rt.mq_close(q)
EOF
#   received prio 31: EMERGENCY
#   received prio  5: normal-task
#   received prio  0: bulk-report      ← the queue re-ordered by priority

# ---- 2. Kernel persistence: sender long dead, message alive ----
$ sudo mkdir -p /dev/mqueue 2>/dev/null; mount | grep mqueue || sudo mount -t mqueue none /dev/mqueue
$ ls -l /dev/mqueue/
# -rw------- ... lab            ← the queue, as a file-ish object
$ cat /dev/mqueue/lab
# QSIZE:0 NOTIFY:0 SIGNO:0 NOTIFY_PID:0    ← live introspection (0 = drained)
$ python3 -c "
import ctypes, os
rt = ctypes.CDLL('librt.so.1')
q = rt.mq_open(b'/lab', os.O_RDWR)
rt.mq_send(q, b'from beyond the grave', 21, 7)"    # sender exits immediately
$ cat /dev/mqueue/lab
# QSIZE:21 ...                  ← 21 bytes wait in the KERNEL, no process holds it
$ python3 -c "
import ctypes, os
rt = ctypes.CDLL('librt.so.1'); q = rt.mq_open(b'/lab', os.O_RDWR)
buf = bytes(256); import ctypes as c; b = c.create_string_buffer(256); p = c.c_uint()
n = rt.mq_receive(q, b, 256, c.byref(p)); print('minutes later:', b.value[:n])"
$ python3 -c "import ctypes; ctypes.CDLL('librt.so.1').mq_unlink(b'/lab')"

# ---- 3. D-Bus: explore the control plane you already use ----
$ busctl list | head -8
# NAME                        PID PROCESS         USER  ...
# org.freedesktop.login1      800 systemd-logind  root
# org.freedesktop.systemd1      1 systemd         root  ← PID 1, on the bus
$ busctl tree org.freedesktop.login1 | head -6
# └─ /org/freedesktop/login1            ← object paths, like a filesystem
$ busctl introspect org.freedesktop.login1 /org/freedesktop/login1 | head -12
# .ListSessions  method  -  a(susso)  -      ← typed methods!
# .SessionNew    signal  so -         -      ← subscribable events

# ---- 4. CALL a method — this is what loginctl does inside ----
$ busctl call org.freedesktop.login1 /org/freedesktop/login1 \
    org.freedesktop.login1.Manager ListSessions
# a(susso) 1 "2" 1000 "you" "seat0" "/org/freedesktop/login1/session/_32"
$ loginctl list-sessions --no-legend     # same data, porcelain view

# ---- 5. Watch signals flow (broadcasts, not polling!) ----
$ timeout 10 busctl monitor --match "type='signal'" 2>/dev/null | head -20 &
$ sleep 1; systemctl daemon-reload       # cause some bus chatter
$ wait
# signal traffic: UnitFilesChanged, PropertiesChanged... — the machine
# narrating its own state changes. Subscribers react; nobody polls.
```

---

## Further Reading

| Topic | Link |
|---|---|
| `mq_overview(7)` — POSIX mqueues | <https://man7.org/linux/man-pages/man7/mq_overview.7.html> |
| `mq_send(3)` / `mq_receive(3)` | <https://man7.org/linux/man-pages/man3/mq_send.3.html> |
| D-Bus (Wikipedia) | <https://en.wikipedia.org/wiki/D-Bus> |
| `busctl(1)` man page | <https://www.freedesktop.org/software/systemd/man/latest/busctl.html> |
| Message queue (Wikipedia) | <https://en.wikipedia.org/wiki/Message_queue> |
| Message broker | <https://en.wikipedia.org/wiki/Message_broker> |

---

## Checkpoint

**Q1.** What does a message queue give you that a pipe doesn't? Name the
three distinct properties and a concrete use each enables.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <strong>Message boundaries</strong> — one send = one receive, atomic,
no framing protocol (a pipe is an undifferentiated byte stream needing
length-prefixes or delimiters — Lesson 31's PIPE_BUF dance): enables
record-oriented workers that can't misparse. (2) <strong>Priorities</strong>
— strict 31-level ordering at the queue itself: an emergency-shutdown command
overtakes 10,000 queued bulk jobs without a separate channel. (3)
<strong>Time decoupling</strong> — messages persist in the kernel independent
of any process's lifetime (lab 2): a producer can enqueue and exit before the
consumer ever starts — cron-style handoffs, boot-order independence; a pipe's
data dies with its fds and requires both ends alive simultaneously. (Bonus
differences: fan-in from many senders without interleaving worries at any
size, and mq_notify's push notification.)
</details>

**Q2.** D-Bus adds a *bus daemon* between processes that could just talk
directly over Unix sockets. What does the intermediary buy, and what's the
architectural cost?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Buys: (1) <strong>discovery & naming</strong> — services own well-known names
arbitrated centrally; clients need no socket paths, and name ownership
doubles as a singleton lock + liveness signal (name vanishes when the owner
dies); (2) <strong>broadcast</strong> — signals fan out to any number of
subscribers with match rules, which point-to-point sockets simply can't do
(N×M connections otherwise); (3) <strong>uniform policy & introspection</strong>
— one place enforces who may call what (XML policy + peer credentials), and
every service is explorable with one tool (busctl) because types/interfaces
are part of the protocol; (4) activation — the bus can start services on
first call (socket activation's cousin). Costs: the daemon is a hop (two
socket traversals per message — latency and a throughput ceiling; large data
goes out-of-band via fd-passing!), a single point of failure/contention, and
a privileged component whose policy language is its attack surface. The same
trade-off — broker vs point-to-point — repeats at datacenter scale
verbatim.
</details>

**Q3.** Map the phase's local concepts onto their datacenter siblings: what
do Kafka/RabbitMQ share with a POSIX mqueue, and which *new* problems do they
exist to solve that no single-kernel IPC has?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Shared concepts, almost embarrassingly directly: message boundaries,
producer/consumer decoupling in time, bounded queues with backpressure
policies (Lesson 30!), priorities/ordering guarantees, and the broker
pattern's discovery + fan-out (D-Bus's exact pitch — RabbitMQ's exchanges ≈
signals + match rules; consumer groups ≈ multiple mq_receivers competing).
New problems from crossing machines: <strong>durability</strong> beyond a
kernel's lifetime (disk-persisted logs, replication — a mqueue dies at
reboot); <strong>delivery semantics</strong> under partial failure —
at-least-once/exactly-once, acknowledgments, redelivery (a single kernel
never "loses" a delivered message, so local IPC never needed acks);
<strong>ordering across partitions/replicas</strong> (one kernel has one
clock and one queue; a cluster must choose what "order" even means); and
<strong>elastic scale</strong> — partitioning across brokers/consumers. The
mental model transfers; the failure model is the upgrade — which is a fine
one-line summary of distributed systems generally.
</details>

---

## Homework

Using `busctl` only (no systemctl/loginctl porcelain), find and call the
D-Bus method that systemd uses to report a unit's active state: discover the
service name, find the object path for the `ssh.service` unit (hint:
`busctl call org.freedesktop.systemd1 /org/freedesktop/systemd1
org.freedesktop.systemd1.Manager GetUnit s ssh.service`), then read its
`ActiveState` property. Write down each command and what layer of the D-Bus
model (name/path/interface/method/property) each argument exercised.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<pre>
# 1. the NAME (who): org.freedesktop.systemd1 — verify it's on the bus:
busctl list | grep systemd1
# 2. METHOD call on the Manager INTERFACE at the root PATH to resolve the unit:
busctl call org.freedesktop.systemd1 /org/freedesktop/systemd1 \
    org.freedesktop.systemd1.Manager GetUnit s ssh.service
#    → o "/org/freedesktop/systemd1/unit/ssh_2eservice"   ← an OBJECT PATH
#      (note the escaping: '.' → _2e — paths have a restricted charset)
# 3. read a PROPERTY on that object via the standard Properties interface:
busctl get-property org.freedesktop.systemd1 \
    /org/freedesktop/systemd1/unit/ssh_2eservice \
    org.freedesktop.systemd1.Unit ActiveState
#    → s "active"
</pre>
Layers exercised: <em>name</em> = the service (systemd) owning a well-known
bus identity; <em>object path</em> = one unit among thousands, addressed like
a filesystem node (returned typed as <code>o</code>, an object-path value —
the type system at work); <em>interface</em> = two different contracts on the
same objects (Manager for lookup, Unit for unit-state; get-property secretly
uses org.freedesktop.DBus.Properties.Get — the standard property interface);
<em>method vs property</em> = GetUnit is an action with arguments (signature
<code>s</code>), ActiveState a readable attribute. This is exactly the call
chain inside <code>systemctl is-active ssh</code> — porcelain is D-Bus with
better manners.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 35 — eventfd, signalfd, timerfd →](lesson-35-eventfd-signalfd){: .btn .btn-primary }
