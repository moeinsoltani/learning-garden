---
title: "Lesson 11 — Credentials and Capabilities"
nav_order: 6
parent: "Phase 2: Processes"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 11: Credentials and Capabilities

## Concept

Every permission check in the kernel asks one question: **who is this process?**
The answer is the process's **credentials** — a set of numeric IDs carried in
the task_struct and inherited across fork/exec (Lesson 06's homework showed
them riding unchanged down your shell's ancestry).

But "who" is more subtle than one number. Each process carries *three* user IDs:

```
            real UID  (ruid)   who you ARE          — the login identity;
                                                       used for "who may signal me"
       effective UID  (euid)   who you ACT AS       — used for permission checks
           saved UID  (suid)   who you MAY BECOME   — lets you switch back
```

Why three? Because of programs like `passwd`: any user may run it, but it must
write `/etc/shadow`, which only root can. The fix is the **setuid bit**: an
executable marked setuid-root runs with *euid 0* regardless of who launched it
(ruid stays you — that's how it still knows who you are). The saved UID lets
such a program drop privilege for most of its work and raise it only for the
one critical write.

And the second half of the story: being root isn't one power, it's ~40 separate
ones. **Capabilities** split "root" into named privileges — `CAP_NET_ADMIN`
(change network config — you used this all through the networking track),
`CAP_SYS_ADMIN` (the infamous grab-bag), `CAP_NET_BIND_SERVICE` (bind ports
<1024), `CAP_SYS_PTRACE` (trace other processes)… A process or file can hold
exactly the ones it needs — the least-privilege tool that containers, systemd,
and modern packaging lean on (and which you met from the policy side in
security Lesson 22).

---

## How It Works

### Where checks happen

Every syscall that touches something protected runs a check against the
caller's credentials: file access compares euid/egid (and supplementary groups)
against the inode's owner/mode; `kill()` compares the *real or effective* UID of
sender vs target; binding port 80 requires `CAP_NET_BIND_SERVICE`; and so on.
`sudo` is nothing magical: a setuid-root binary that checks policy
(`/etc/sudoers`) and then execs your command with root credentials.

### How credentials change

Only a few ways: **exec of a setuid/setgid file** (euid/egid become the file's
owner); the **set*uid syscalls** (`setuid`, `seteuid`, `setresuid`) whose rules
boil down to "unprivileged processes may only shuffle between their real,
effective, and saved IDs; root/CAP_SETUID may set anything"; and **login
machinery** (sshd, login — running as root — call `setuid(you)` before exec'ing
your shell: the moment your identity was born).

The classic privilege-drop bug: a root daemon calling `seteuid(user)` (leaving
saved UID 0 — trivially reversible) instead of `setresuid(user,user,user)`
(irreversible). Checking all three IDs is why `/proc/PID/status` shows four
values per line (the 4th, fsuid, is a historical NFS artifact that today tracks
euid).

### File capabilities and ambient rules

Capabilities live in *sets* per process (permitted, effective, inheritable,
ambient, bounding) and can be attached to *files* (`setcap` — how `ping`
works without setuid on modern distros). The full inheritance algebra is
notoriously baroque; what you need: **file caps or systemd's
`AmbientCapabilities=` grant specific powers without setuid-root**, and the
**bounding set** caps what a process tree can ever regain — how containers
permanently amputate dangerous capabilities (`docker run --cap-drop`).

{: .note }
> **setuid is a security cliff**
> A setuid-root binary is attack surface: it runs YOUR input with root power —
> environment, arguments, fds, locale all become hostile input. That's why the
> loader ignores <code>LD_PRELOAD</code> for setuid binaries (Lesson 49), why
> distros keep shrinking the setuid list (<code>find / -perm -4000</code> gets
> shorter every release), and why file capabilities + tightly-scoped services
> are the modern replacement.

---

## Lab

```bash
# ---- 1. Read your own credentials ----
$ id
# uid=1000(you) gid=1000(you) groups=1000(you),27(sudo),...
$ grep -E '^(Uid|Gid|Groups)' /proc/$$/status
# Uid: 1000 1000 1000 1000      ← real, effective, saved, fs

# ---- 2. Watch euid flip during a setuid exec ----
$ ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ...    ← the 's': setuid bit, owner root
$ passwd --help >/dev/null &    # harmless; inspect a real run's creds via ps:
$ ps -o pid,ruid,euid,comm -p $(pgrep -n passwd) 2>/dev/null
#   ruid 1000, euid 0           ← you, acting as root (if you catch it in time)

# sudo is the same trick:
$ sudo grep ^Uid /proc/self/status
# Uid: 0 0 0 0                  ← full root after sudo's policy check

# ---- 3. kill permission uses UIDs (Lesson 09 meets Lesson 11) ----
$ kill -0 1 && echo "may signal init" || echo "not permitted"
# not permitted                 ← PID 1 is root's; your ruid/euid don't match
$ sudo kill -0 1 && echo "root may"

# ---- 4. Capabilities: how ping works without setuid ----
$ ls -l $(which ping)
# -rwxr-xr-x ...                ← NO setuid bit!
$ getcap $(which ping)
# ping cap_net_raw=ep           ← a FILE capability instead
# (on some distros ping uses unprivileged ICMP sockets — then getcap is empty
#  and the enabler is the sysctl net.ipv4.ping_group_range. Check both!)

# ---- 5. Grant and revoke a capability yourself ----
$ cp /usr/bin/python3 /tmp/p3
$ /tmp/p3 -c "import socket; socket.socket().bind(('0.0.0.0', 80))"
# PermissionError: port 80 needs privilege
$ sudo setcap cap_net_bind_service=ep /tmp/p3
$ /tmp/p3 -c "import socket; s=socket.socket(); s.bind(('0.0.0.0',80)); print('bound port 80 as uid', __import__('os').getuid())"
# bound port 80 as uid 1000     ← one power, no root!
$ sudo setcap -r /tmp/p3 && rm /tmp/p3

# ---- 6. What's running with capabilities right now? ----
$ grep -E 'Cap(Prm|Eff)' /proc/self/status
# CapPrm: 0000000000000000      ← you: none
$ sudo grep CapEff /proc/1/status
# CapEff: 000001ffffffffff      ← PID 1: everything
$ systemctl show systemd-resolved -p CapabilityBoundingSet | cut -c1-80
# a tight, explicit list — modern service hardening in action

# ---- 7. The shrinking setuid list ----
$ find /usr/bin /usr/sbin -perm -4000 2>/dev/null
# a short list: sudo, passwd, mount... each one a carefully audited cliff
```

---

## Further Reading

| Topic | Link |
|---|---|
| `credentials(7)` man page | <https://man7.org/linux/man-pages/man7/credentials.7.html> |
| `capabilities(7)` man page | <https://man7.org/linux/man-pages/man7/capabilities.7.html> |
| `setresuid(2)` man page | <https://man7.org/linux/man-pages/man2/setresuid.2.html> |
| Setuid (Wikipedia) | <https://en.wikipedia.org/wiki/Setuid> |
| `setcap(8)` man page | <https://man7.org/linux/man-pages/man8/setcap.8.html> |
| `sudo(8)` man page | <https://man7.org/linux/man-pages/man8/sudo.8.html> |

---

## Checkpoint

**Q1.** `ping` works without sudo on your VM. Explain the two possible
mechanisms and how you'd check which one is in effect.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Mechanism 1: a <strong>file capability</strong> — the binary carries
<code>cap_net_raw</code> (create raw ICMP sockets); check with
<code>getcap $(which ping)</code>. Mechanism 2: <strong>unprivileged ICMP
sockets</strong> — the kernel allows plain users to open ICMP echo datagram
sockets when their group falls inside the sysctl
<code>net.ipv4.ping_group_range</code>; check that sysctl and whether getcap
shows nothing. (Historic mechanism 3, now rare: the setuid bit —
<code>ls -l</code> would show <code>rws</code>.) Same visible behavior, three
different privilege designs — an object lesson in how "just works" hides policy.
</details>

**Q2.** A root daemon wants to serve requests as user `www` but occasionally
re-read a root-only config. It calls `seteuid(www)` at startup. A security
review calls this dangerous, but also notes the *design* has a real
justification. Explain both sides, and what full privilege-drop would look like.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Danger: <code>seteuid</code> changes only the effective UID; the saved UID is
still 0, so <em>any code-execution bug</em> in the www-serving path can call
<code>seteuid(0)</code> and be root again — the "drop" is a curtain, not a
wall. Justification: that reversibility is exactly what lets it re-raise to
re-read the root-only config (the passwd pattern: work unprivileged, raise
briefly for the one privileged act). Full drop is
<code>setgroups()+setgid()+setresuid(www,www,www)</code> — all three UIDs, gid
first, irreversible. Better modern designs avoid the tension: keep a tiny
privileged parent that reads configs and passes them (or an fd) to an
unprivileged worker — privilege separation instead of privilege toggling
(sshd's architecture, and the spirit of security Lesson 35's workload identity).
</details>

**Q3.** Your container runs as root (uid 0) yet cannot change the system time
or load kernel modules. Reconcile this with everything above.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
"Root" only means uid 0; the actual powers are capabilities, and the container
runtime removed them: Docker's default set omits <code>CAP_SYS_TIME</code> and
<code>CAP_SYS_MODULE</code> (and dozens more), and the <em>bounding set</em>
makes the amputation permanent — no exec or setuid inside the container can
ever regain them. Check from inside: <code>grep CapEff /proc/1/status</code>
and decode with <code>capsh --decode</code>. Add user namespaces (Lesson 59)
and even that uid 0 maps to an unprivileged uid on the host. Modern "root in a
container" is a bundle of specific capabilities over namespaced objects — which
is why <code>--privileged</code> (all caps, real ones) is such a big red switch.
</details>

---

## Homework

Reproduce the passwd pattern in miniature. Write a small C program that: prints
its three UIDs (`getresuid`), opens `/etc/shadow` (expect failure), and exits.
Compile it, then: (a) run it normally, (b) `sudo chown root` + `chmod u+s` it
and run again, observing all three UIDs and the open result, (c) inside the
program (setuid version), drop to your real UID with
`setresuid(getuid(), getuid(), getuid())` before the open and observe. Clean up
the binary afterwards! What did (c) prove about where the privilege lives?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<pre>
#include &lt;stdio.h&gt;
#include &lt;unistd.h&gt;
int main(void){
    uid_t r,e,s; getresuid(&r,&e,&s);
    printf("ruid=%d euid=%d suid=%d\n", r, e, s);
    /* for (c): setresuid(getuid(), getuid(), getuid()); */
    FILE *f = fopen("/etc/shadow","r");
    printf("shadow: %s\n", f ? "OPENED" : "denied");
    return 0;
}
</pre>
(a) <code>1000 1000 1000</code>, denied. (b) <code>1000 0 0</code> — ruid is
still you, euid/suid are root because exec of a setuid file sets them — and
shadow OPENS: the euid is what the file-permission check uses. (c) after
setresuid all three are 1000 and the open is denied again — proving privilege
lives in the <em>process credentials</em>, not in the binary or the moment of
launch: the same setuid program, having irreversibly shed its effective and
saved root, is powerless. That's the entire theory of privilege dropping — and
why auditing <em>when</em> a program drops matters as much as whether it does.
Remove the binary (<code>sudo rm</code>) when done — a forgotten setuid-root
toy is a real backdoor.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 3 — Scheduling (Lesson 12: Context Switches) →](lesson-12-context-switches){: .btn .btn-primary }
