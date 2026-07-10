---
title: "Lesson 52 — systemd: PID 1"
nav_order: 3
parent: "Phase 10: Boot & Init"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 52: systemd — PID 1

## Concept

The initramfs handed off to `/sbin/init` — on nearly every modern Linux, that
is **systemd**, running as PID 1 for the life of the system. Two jobs make
PID 1 special, and both trace to lessons you've already done:

```
  PID 1 has two unshirkable duties (Lessons 06/08):

  1. REAP orphans:  every process whose parent dies re-parents to PID 1
                    (Lesson 08). PID 1 MUST wait() on them or zombies
                    accumulate forever. It's the system's garbage collector
                    for dead processes.

  2. Supervise:     start, stop, restart, and order everything in userspace.
                    If PID 1 dies, the KERNEL PANICS — there's no one to
                    reap or supervise. It is the one truly irreplaceable
                    process.
```

systemd's big idea over old SysV init: describe the desired *state*
declaratively as **units**, and let PID 1 figure out how to reach it — in
parallel, on demand, with dependencies. A unit is a small text file
describing one thing to manage:

- **service** (`.service`) — a daemon or process to run (the common one)
- **socket** (`.socket`) — a socket to listen on, starting the service on
  first connection (Lesson 32's socket activation!)
- **timer** (`.timer`) — cron-replacement scheduling
- **mount**, **target** (a named group/milestone, like the old runlevels),
  **device**, **path** (inotify-triggered), **slice** (a cgroup — Lesson 60!)

You've *been* a systemd user throughout: `systemctl restart` was a D-Bus call
to PID 1 (Lesson 34), service sandboxing was mount-namespace surgery
(Lesson 40's homework), and `Restart=on-failure` was the supervisor loop you
built in Lesson 08. This lesson assembles the picture.

---

## How It Works

### Units, dependencies, targets

Units declare relationships: `Requires=`/`Wants=` (needs/wants another unit),
`After=`/`Before=` (ordering — separate from requirement!), `Conflicts=`.
systemd builds a dependency graph and starts everything it can *in parallel*,
respecting order only where declared — the reason modern boots are seconds,
not the sequential minute of SysV. **Targets** are sync points/groups:
`multi-user.target` (normal server state), `graphical.target`,
`rescue.target` — the successors to runlevels, reached by satisfying their
dependencies. `systemctl get-default` shows your boot goal.

### The daily workflow (worth muscle memory)

`systemctl status/start/stop/restart/enable/disable NAME` (enable = start at
boot, via symlink into a `.wants/` dir — declarative again); `systemctl list-
units`/`--failed`; `journalctl -u NAME` (logs — systemd captures each
service's stdout/stderr into the structured journal, Lesson 35's fd capture),
`journalctl -b` (this boot), `-f` (follow). `systemctl cat NAME` shows the
unit; `systemctl edit NAME` makes override drop-ins (never edit vendor units
directly).

### Writing a unit, and the sandboxing payoff

A minimal service is ~6 lines (the lab writes one). The power is the
hardening directives — each one machinery from earlier phases, declared:
`User=`/`Group=` (Lesson 11 credentials), `Restart=`/`RestartSec=` (Lesson
08 supervision), `MemoryMax=`/`CPUWeight=` (Lesson 22/60 cgroups),
`ProtectSystem=`/`PrivateTmp=`/`ReadOnlyPaths=` (Lesson 40 mount namespaces),
`NoNewPrivileges=`/`SystemCallFilter=` (Lesson 11/56 — seccomp).
`systemd-analyze security NAME` scores a unit's exposure — a checklist of
everything this course taught, applied per service. systemd is, from one
angle, a friendly front-end to namespaces + cgroups + capabilities +
seccomp — the same primitives a container runtime uses (Lesson 62).

{: .note }
> **Not just servers: user services and slices**
> <code>systemctl --user</code> manages a per-login systemd (your desktop
> session's services). And every service runs in a <strong>cgroup</strong>
> (Lesson 60) under a <em>slice</em> hierarchy — <code>systemd-cgls</code>
> shows the tree, <code>systemctl status</code> shows a service's cgroup and
> its resource use. systemd is the cgroup manager on modern Linux; you don't
> poke <code>/sys/fs/cgroup</code> directly, you set unit properties.

---

## Lab

```bash
# ---- 1. Meet PID 1 and confirm its identity ----
$ ps -p 1 -o pid,comm,args
# 1 systemd /sbin/init ...     ← the initramfs's handoff, still running
$ sudo cat /proc/1/comm; ls -l /sbin/init      # → systemd
$ systemctl --version | head -1

# ---- 2. PID 1's reaping duty, observed (Lesson 08 from the top) ----
$ grep -E 'Threads|SigCgt' /proc/1/status | head -2
# systemd catches SIGCHLD to reap the whole system's orphans
$ ps -ef | awk '$3==1' | wc -l        # how many processes PID 1 directly parents
# many — every daemon + every orphan re-parented to it

# ---- 3. The unit graph and boot analysis ----
$ systemctl get-default                # multi-user.target or graphical.target
$ systemd-analyze                      # kernel time + userspace time
$ systemd-analyze blame | head -8      # slowest units — where boot time went
$ systemd-analyze critical-chain | head -12   # the dependency CRITICAL PATH
# note: parallel starts, ordering only where After=/Requires= demand it

# ---- 4. Explore a real service ----
$ systemctl status ssh 2>/dev/null || systemctl status systemd-resolved
$ systemctl cat systemd-resolved | head -20    # the actual unit file
$ systemctl show systemd-resolved -p Restart -p User -p ProtectSystem \
    -p MemoryMax -p NoNewPrivileges           # the hardening in effect
$ systemctl list-dependencies systemd-resolved | head -8

# ---- 5. Write, run, and inspect YOUR OWN service ----
$ sudo tee /etc/systemd/system/lab-clock.service << 'EOF'
[Unit]
Description=Lab clock — prints time every 5s
After=network.target

[Service]
ExecStart=/bin/bash -c 'while true; do date; sleep 5; done'
Restart=on-failure
DynamicUser=yes                 # runs as a transient unprivileged user (L11)
ProtectSystem=strict            # read-only filesystem view (L40)
PrivateTmp=yes                  # private /tmp (L40)
MemoryMax=50M                   # cgroup limit (L22/L60)

[Install]
WantedBy=multi-user.target
EOF
$ sudo systemctl daemon-reload
$ sudo systemctl start lab-clock
$ systemctl status lab-clock --no-pager | head -12
# Active: active (running) ... CGroup: /system.slice/lab-clock.service
#                                       └─ its cgroup, its resource accounting
$ journalctl -u lab-clock -n 4 --no-pager      # its output, captured (L35)
$ systemd-analyze security lab-clock | tail -6  # exposure score of YOUR unit

# ---- 6. See the supervision + sandbox in action ----
$ MAINPID=$(systemctl show lab-clock -p MainPID --value)
$ grep -E 'Uid|Cpus' /proc/$MAINPID/status | head -1    # NOT your uid (DynamicUser!)
$ sudo findmnt -N $(pgrep -f 'while true; do date' | head -1) 2>/dev/null | grep -E 'tmp|/ ' | head -3
# the private tmpfs + ro remounts — Lesson 40's mount surgery, per-service
$ sudo kill $MAINPID; sleep 2; systemctl status lab-clock --no-pager | grep Active
# systemd RESTARTED it (Restart=on-failure) — Lesson 08's supervisor, declared
$ sudo systemctl stop lab-clock && sudo rm /etc/systemd/system/lab-clock.service
$ sudo systemctl daemon-reload
```

---

## Further Reading

| Topic | Link |
|---|---|
| systemd (Wikipedia) | <https://en.wikipedia.org/wiki/Systemd> |
| `systemd(1)` man page | <https://man7.org/linux/man-pages/man1/systemd.1.html> |
| `systemd.service(5)` | <https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html> |
| `systemd.exec(5)` — the sandboxing directives | <https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html> |
| `journalctl(1)` man page | <https://man7.org/linux/man-pages/man1/journalctl.1.html> |
| `systemd-analyze(1)` | <https://www.freedesktop.org/software/systemd/man/latest/systemd-analyze.html> |

---

## Checkpoint

**Q1.** What two jobs must PID 1 do even in the most minimal system, and what
happens if it dies?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) <strong>Reap orphaned processes</strong>: when any process's parent dies,
the kernel re-parents its children to PID 1 (Lesson 08); PID 1 must
<code>wait()</code> on them when they exit, or their zombies (Lesson 08)
accumulate forever, leaking the PID space (Lesson 06's pid_max) until the
system can't fork. PID 1 is the system's final backstop for dead-process
cleanup. (2) <strong>Supervise / be the root of userspace</strong>: it starts
the initial services and, at minimum, must not exit — everything descends
from it. If PID 1 dies, the kernel <strong>panics</strong>
("Attempted to kill init!") and halts: there's no one left to reap or to own
the process tree, and the kernel considers the system unrecoverable. This is
why PID 1 is the one process you cannot SIGKILL to death (the kernel blocks
signals PID 1 hasn't handled — Lesson 09's special-case), why container
"PID 1 problems" (Lesson 08) matter, and why systemd is engineered to never
crash (and to re-exec itself in place for upgrades rather than restart).
Everything else systemd does — units, targets, cgroups — is elaboration; these
two are the irreducible duties of being init.
</details>

**Q2.** systemd boots in parallel while SysV init ran scripts sequentially,
yet services still start in the right order. Explain how declarative
dependencies achieve both — and the difference between `Requires=` and
`After=`.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
SysV encoded ordering <em>imperatively</em>: numbered scripts (S10, S20…)
ran one after another, so a slow service blocked everything behind it even
if unrelated — sequential by construction. systemd instead has each unit
<em>declare</em> only its actual relationships, then builds a dependency
graph and launches everything whose prerequisites are met <em>concurrently</em>
— dozens of independent services start at once; ordering constrains only the
genuinely dependent pairs. Boot time becomes the critical path
(<code>systemd-analyze critical-chain</code>), not the sum. The two directive
kinds are orthogonal and often confused: <strong>Requires=</strong> is a
<em>requirement</em> (if A Requires B, starting A pulls in B, and B failing/
stopping affects A) — it says nothing about <em>when</em>. <strong>After=</strong>
is pure <em>ordering</em> (if A is After B, and both are being started, A
waits until B has started) — it says nothing about <em>whether</em> B is
needed. You usually want both together (Requires=network.target +
After=network.target = "I need network AND must start after it"), but they're
separable: After= without Requires= means "if it's starting too, go after it,
but don't drag it in"; Requires= without After= means "pull it in, but I don't
care about ordering" (rarely what you want — a classic bug is Requires without
After, so your service races its own dependency). Socket activation
(Lesson 32) dissolves many ordering needs entirely: systemd holds the socket
from the start, so dependents can "connect" before the provider even runs.
</details>

**Q3.** A colleague says "systemd does too much — it's init, but also
manages services, logging, cgroups, mounts, sandboxing, and more; that's
scope creep." Steelman systemd's design *and* the criticism, using this
course's concepts.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Steelman for systemd: those responsibilities are not arbitrary — they're the
<em>coupled</em> concerns of managing a service's whole lifecycle, and
integrating them is what enables features impossible when they're separate
tools. Supervising a service (Lesson 08) is entangled with confining it
(cgroups, Lesson 60 — you must know a service's process tree to limit and
account it; PID 1 owning the cgroup hierarchy makes resource control
reliable), isolating it (namespaces/mounts, Lesson 40), securing it
(capabilities/seccomp, Lessons 11/56), capturing its output (the journal —
Lesson 35's fd capture, tied to the process it supervises), and ordering it
against sockets/mounts/devices. Doing these in one coordinated init means:
socket activation (Lesson 32) for parallel boot, declarative sandboxing
(<code>systemd-analyze security</code> — a checklist of this whole course),
correct reaping+cgroup cleanup on service death, and a uniform interface
(D-Bus, Lesson 34). The criticism, steelmanned: this is the Unix-philosophy
violation (do one thing well) traded for integration — it concentrates a
huge, complex, privileged surface into PID 1 (the one process that must never
fail — Q1), creates a single vendor's ABI that the ecosystem now depends on
(the portability/lock-in concern), and makes the pieces hard to replace
individually (you can't easily swap just the logger or just the mount
manager). Both are correct: systemd bet that service management's concerns
are genuinely coupled and integration beats composition <em>here</em> — a bet
most distros accepted for its capabilities, while the criticism correctly
identifies the architectural cost (concentration, complexity in the critical
process, reduced modularity). It's the same monolithic-vs-microkernel tension
from Lesson 01, replayed in userspace init — and, tellingly, resolved the
same way the kernel was: integration won on capability and performance, with
the acknowledged trade of a large trusted core.
</details>

---

## Homework

Take a script you'd otherwise run in a terminal or a crude `nohup ... &`
(Lesson 10) — say a small web server or a periodic backup — and design its
proper systemd unit(s). Specify: service vs socket vs timer (justify), the
supervision directives (Restart policy, and why), at least three sandboxing
directives from earlier phases (naming the lesson each implements), and the
resource limits. Then explain what this unit gives you that
`nohup script &` or a cron job never could.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
For a periodic backup: a <strong>oneshot service</strong> (the backup run) +
a <strong>timer</strong> (the schedule) — not a long-running service (the
work is intermittent), and a timer over cron because it integrates with the
rest (journald logging, dependency ordering, catch-up with Persistent=true
for missed runs while off). backup.service:
<code>Type=oneshot</code>, <code>ExecStart=/usr/local/bin/backup.sh</code>,
<code>User=backup</code>/<code>DynamicUser=yes</code> (Lesson 11 — least
privilege, not root), <code>ReadOnlyPaths=/</code> + explicit
<code>ReadWritePaths=/backup</code> and <code>ProtectSystem=strict</code>
(Lesson 40 — it can read everything to back up but can only write the backup
target; a compromised backup script can't tamper with the system),
<code>PrivateTmp=yes</code> (Lesson 40 — isolated scratch),
<code>NoNewPrivileges=yes</code> + <code>SystemCallFilter=@system-service</code>
(Lessons 11/56 — no privilege escalation, restricted syscalls),
<code>MemoryMax=500M</code> + <code>IOWeight=20</code> (Lessons 22/60 — the
backup can't OOM the box or starve foreground I/O — Lesson 42's noisy-neighbor
problem, solved declaratively). backup.timer: <code>OnCalendar=daily</code>,
<code>Persistent=true</code>. For a web server instead: a
<strong>service</strong> + optionally a <strong>socket</strong> (activation:
systemd binds :443, hands the fd over — Lesson 32 — enabling zero-downtime
restarts and privilege-free port binding), with <code>Restart=on-failure</code>
+ <code>RestartSec=2</code> + <code>StartLimitBurst</code> (Lesson 08's
supervisor with backoff and a crash-loop circuit breaker). What this beats
<code>nohup &</code>/cron at: supervision (auto-restart with backoff vs "it
died silently at 3am and nobody noticed"); logging (structured, queryable
<code>journalctl -u</code> vs a stray nohup.out or lost cron mail);
confinement (the whole sandbox — a nohup'd or cron'd script runs with your
full privileges and full filesystem access, an obvious blast radius);
resource control (cgroup limits vs "the backup ate all RAM and the OOM killer
took the database" — Lesson 22); dependency ordering (After=network-online,
mounts ready — vs cron firing before the network exists); and clean lifecycle
(stop reliably kills the whole process tree via the cgroup, vs nohup leaving
orphans and cron's no-stop-concept). Every one of those is a primitive from
this course that systemd exposes as one line of declarative config — which is
the real answer to "why not just nohup": you'd be hand-reimplementing half
the OS course, badly.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 53 — Kernel Modules and udev →](lesson-53-modules-udev){: .btn .btn-primary }
