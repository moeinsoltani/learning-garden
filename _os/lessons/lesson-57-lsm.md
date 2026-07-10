---
title: "Lesson 57 — LSM: AppArmor and SELinux"
nav_order: 4
parent: "Phase 11: Kernel Interfaces & Security"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 57: LSM — AppArmor and SELinux

## Concept

Lesson 11 showed the classic permission model: a file's owner/group/mode, and
root bypassing all of it. That's **DAC** — Discretionary Access Control:
*discretionary* because the owner decides who can access their files, and root
is omnipotent. It has a fatal gap for security: a process running as root (or
as a file's owner) can do *anything* to *everything* — so a compromised root
daemon owns the machine.

**MAC** — Mandatory Access Control — closes it: a system-wide policy, set by
the administrator, that *even root cannot override*, restricting each process
to exactly the objects it legitimately needs.

```
   DAC (Lesson 11):  "can this UID access this file's mode bits?"
                     root: YES to everything. owner: decides.

   MAC (LSM):        "does POLICY permit THIS program to touch THIS object?"
                     a compromised web server running as root can STILL only
                     read /var/www and its own logs — because policy says so,
                     and policy outranks root.
```

The Linux kernel implements MAC through the **LSM** framework (Linux Security
Modules): hooks placed at every security-relevant operation (open a file,
send a packet, ptrace a process, load a module) where a security module gets
to say yes/no *in addition to* the DAC check. Two major modules:

- **AppArmor** (Ubuntu/SUSE) — **path-based** profiles: "this program may read
  `/etc/nginx/**` and write `/var/log/nginx/**`, nothing else." Readable,
  per-program, easy to reason about.
- **SELinux** (RHEL/Fedora/Android) — **label-based** type enforcement: every
  file and process gets a security *context* (label); policy defines which
  process types may access which object types. More powerful and fine-grained,
  famously harder to author.

This completes virtualization Lesson 56 (sVirt): sVirt *is* SELinux/AppArmor
confining each QEMU process to only its own VM's disk images — so a
guest-escape exploit finds itself in a process that policy forbids from
touching any other VM's files. MAC is the containment that assumes the process
is already compromised.

---

## How It Works

### LSM hooks: policy in addition to DAC

The kernel calls LSM hooks at security decision points *after* the standard
DAC check passes — so LSM can only further *restrict*, never grant (a denial
by DAC still denies; LSM adds a second gate). The active module(s) consult
their loaded policy and return allow/deny. This is why "root can read every
file" (DAC) becomes "root can read every file *the policy permits this
process to read*" — the DAC answer is necessary but no longer sufficient.

### AppArmor: profiles you can read

A profile (`/etc/apparmor.d/`) names an executable and lists permitted
accesses: file paths with permission flags (`r`, `w`, `x`, `m` for mmap-exec),
capabilities (Lesson 11 — `capability net_bind_service,`), network rules.
Modes: **enforce** (violations blocked + logged) and **complain** (allowed but
logged — for building a profile by observing real behavior, the empirical
approach seccomp's homework praised). `aa-status` lists profiles;
`aa-complain`/`aa-enforce` switch modes; denials appear in the audit log and
`dmesg`. Its path-based nature is its strength (legible) and weakness (rename
tricks, hardlinks, and bind mounts can confuse path matching — where SELinux's
inode labels are more robust).

### SELinux: labels and type enforcement

Every object carries a context `user:role:type:level`
(`system_u:object_r:httpd_sys_content_t:s0`); the **type** is what matters
most. Policy rules say "a process of type `httpd_t` may read files of type
`httpd_sys_content_t`" — so nginx (running as `httpd_t`) can serve web content
but cannot read `shadow_t` (/etc/shadow) *even as root*, because no rule
permits `httpd_t → shadow_t`. `ls -Z`/`ps -Z` show contexts;
`getenforce`/`setenforce` toggle enforcing/permissive; `ausearch`/`audit2allow`
analyze denials (and can generate policy from them). Labels travel with inodes
(robust against path tricks) but authoring policy is a genuine skill — which
is why AppArmor traded power for approachability, and both survive.

{: .note }
> **Why MAC feels invisible until it bites**
> On a well-configured system MAC is silent — every legitimate access is
> permitted by policy, so you never notice it. You meet it when something
> <em>legitimate</em> is denied: a service can't read a file it should
> (wrong SELinux label after you moved a file — <code>restorecon</code> fixes
> it), or an AppArmor profile is too tight after an app update. The reflex
> "just <code>setenforce 0</code>" disables your last containment layer —
> the right move is reading the audit log and fixing the label/profile. This
> is the operational skill MAC actually demands.

---

## Lab

```bash
# ---- 1. Which MAC is active on your system? ----
$ cat /sys/kernel/security/lsm         # the stack: capability,apparmor / ...selinux
$ aa-status 2>/dev/null | head -5 || sudo apparmor_status 2>/dev/null | head -5
$ getenforce 2>/dev/null               # (SELinux systems: Enforcing/Permissive)
# Ubuntu → AppArmor; RHEL/Fedora → SELinux. The labs below adapt.

# ---- 2. See the labels/profiles on running processes ----
$ ps -eZ 2>/dev/null | head -5 || ps -eo pid,label,comm 2>/dev/null | head -5
# AppArmor: /usr/sbin/mysqld (enforce)   SELinux: system_u:system_r:httpd_t:s0
$ ls -Z /etc/shadow 2>/dev/null || echo "(no SELinux labels — AppArmor system)"
# on SELinux: ...:shadow_t:s0  ← the type that policy protects fiercely

# ---- 3. AppArmor: read a real profile ----
$ ls /etc/apparmor.d/ 2>/dev/null | head -8
$ sudo cat /etc/apparmor.d/usr.sbin.tcpdump 2>/dev/null | grep -E '^\s*(/|capability|network)' | head -12 \
    || sudo cat /etc/apparmor.d/*man* 2>/dev/null | head -20
# the profile literally lists: which PATHS, which CAPABILITIES (L11),
# which NETWORK — a human-readable "this program may only do X" contract

# ---- 4. Build a profile by OBSERVING (complain mode) ----
$ cat > /tmp/reader.sh << 'EOF'
#!/bin/bash
cat /etc/hostname          # legitimate
cat /etc/shadow 2>/dev/null # should be denied by a tight profile
EOF
$ chmod +x /tmp/reader.sh
$ if command -v aa-genprof >/dev/null; then cat << 'NOTE'
  # on an AppArmor system with apparmor-utils:
  sudo aa-genprof /tmp/reader.sh     # run it, aa-logprof learns the accesses
  # → generates a profile allowing ONLY /etc/hostname; then enforce it and
  #   the /etc/shadow read is BLOCKED even though DAC (root) would allow it.
NOTE
  else echo "(install apparmor-utils to build profiles interactively)"; fi

# ---- 5. The MAC-beats-root demonstration (conceptual + observable) ----
# Even as root, a confined process is limited. Find a confined one:
$ for p in $(pgrep -x mysqld snapd cups-browsed 2>/dev/null | head -1); do
    echo "PID $p profile:"; cat /proc/$p/attr/current 2>/dev/null; done
# a nonzero profile/context = this process, even if root, is MAC-confined:
# it can touch only what its profile/type policy permits, not all of root's power

# ---- 6. See a denial in the log (the operational reality) ----
$ sudo dmesg 2>/dev/null | grep -iE 'apparmor|audit.*denied|avc:' | tail -5
# apparmor="DENIED" operation="open" profile="..." name="/etc/shadow" ...
#   OR (SELinux)  avc:  denied  { read } ... scontext=...httpd_t tcontext=...shadow_t
# THIS log line is what you read instead of `setenforce 0` — it names exactly
# what was denied, to whom, on what object: fix the policy, keep the shield.
$ rm -f /tmp/reader.sh
```

---

## Further Reading

| Topic | Link |
|---|---|
| Mandatory access control | <https://en.wikipedia.org/wiki/Mandatory_access_control> |
| Linux Security Modules | <https://en.wikipedia.org/wiki/Linux_Security_Modules> |
| AppArmor (Wikipedia) | <https://en.wikipedia.org/wiki/AppArmor> |
| SELinux (Wikipedia) | <https://en.wikipedia.org/wiki/Security-Enhanced_Linux> |
| SELinux — kernel/docs | <https://www.kernel.org/doc/html/latest/admin-guide/LSM/SELinux.html> |
| `apparmor.d(5)` — profile syntax | <https://manpages.ubuntu.com/manpages/latest/en/man5/apparmor.d.5.html> |

---

## Checkpoint

**Q1.** Root can read every file under DAC. Under MAC, can it? What does that
buy for a compromised daemon running as root?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
No — that's the entire point of MAC. Under DAC (Lesson 11), the effective UID
0 bypasses all file-permission checks, so a root process reads anything. Under
MAC, after the DAC check passes, the kernel also consults the LSM policy,
which restricts the <em>process</em> to only the objects its profile/type
permits — and this policy is <em>mandatory</em>: it is not the process's or
even root's to override (only the security administrator changes policy, and
on a locked-down system even that is gated). So a root process confined to
type <code>httpd_t</code> (or an nginx AppArmor profile) can read /var/www but
<em>not</em> /etc/shadow — the read is denied though DAC would have allowed it,
because no policy rule connects that process type to that object type. What it
buys for a compromised root daemon: containment of a total compromise. Attacker
gains root-level code execution in the web server — normally game over, they
read shadow, install a rootkit (load a module — Lesson 54), pivot everywhere.
With MAC, that root shell is stuck inside <code>httpd_t</code>: it can't read
password hashes (no httpd_t→shadow_t rule), can't load kernel modules (policy
denies the capability regardless of DAC — Lesson 11's capabilities enforced by
MAC too), can't write outside its permitted paths, can't ptrace other services.
The exploit succeeded but the blast radius is one confined service. MAC is
security's assumption of failure made concrete: design so that owning a
process — even as root — doesn't own the machine. (This is exactly sVirt's job
for QEMU, virt Lesson 56: a guest escape lands in a process policy-forbidden
from touching any other VM.)
</details>

**Q2.** Contrast AppArmor's path-based approach with SELinux's label-based
type enforcement: give one concrete scenario where the path-based model is
fooled but labels hold, and one reason AppArmor persists despite that
weakness.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
AppArmor matches on <em>pathnames</em> ("may access /var/www/**"); SELinux
matches on <em>inode labels</em> that travel with the file's identity
regardless of name. The path model's fooling scenario: a process is confined
to /var/www but has write access somewhere, and there's a <strong>hard link
or rename/bind-mount</strong> trick — e.g., an attacker (or a bug) creates a
hard link making a sensitive file <em>also</em> reachable under a permitted
path, or a bind mount (Lesson 40) grafts a protected directory into an
allowed path, and the path rule now matches an object it was never meant to —
because AppArmor sees only the path used, not the underlying inode. It can
also mis-handle files accessed through symlinks or files renamed after the
profile was written. SELinux is immune to these: the label lives on the
<em>inode</em> (Lesson 37), so the same file has the same protected type
whether reached via its real name, a hard link, or a bind mount — identity,
not naming (the same inode-vs-path lesson as 37, applied to security). Why
AppArmor persists anyway: <strong>approachability</strong>. Its profiles are
human-readable "this program may touch these paths and capabilities" contracts
that a normal admin can write, audit, and reason about in complain-then-
enforce workflow; SELinux's type-enforcement policy is powerful but notoriously
hard to author and debug (whole careers, and <code>audit2allow</code> as a
crutch), driving the "just setenforce 0" reflex the note warns against. The
security community kept both because they occupy different points on the
power-vs-usability curve — SELinux where the threat model and expertise justify
it (RHEL servers, Android's per-app isolation), AppArmor where legible
per-program confinement that admins will actually maintain beats theoretically
stronger policy that gets disabled. A robust-but-unused control is weaker than
a modest-but-enforced one — the recurring security truth.
</details>

**Q3.** Put together the phase's three confinement layers — capabilities
(Lesson 11), seccomp (Lesson 56), and MAC (this lesson) — for a single
hardened web server. What does each layer uniquely contribute, and why is
running all three "defense in depth" rather than redundant?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Each restricts a <em>different axis</em>, so they compose rather than
duplicate. <strong>Capabilities</strong> (Lesson 11) slice up root: drop
everything except <code>CAP_NET_BIND_SERVICE</code> so the server can bind
:443 but can't load modules, change system time, mount filesystems, or
override file permissions — it limits <em>which privileged operations</em>
are available even to a root process. <strong>seccomp</strong> (Lesson 56)
limits <em>which syscalls</em> the process can make at all: no <code>execve</code>
(a compromise can't spawn a shell — the single most valuable denial), no
<code>ptrace</code>, no io_uring, no exotic calls — shrinking the kernel
attack surface reachable from the process, independent of privilege
(a process drops execve even while root). <strong>MAC</strong> (this lesson)
limits <em>which objects</em> the process may touch: confined to type
<code>httpd_t</code>, it reads /var/www and its logs and nothing else —
enforcing object-level policy that neither of the others expresses (seccomp
can't see paths — Lesson 56 Q2; capabilities don't restrict which specific
files). Defense in depth, not redundancy, because each covers the others'
blind spots and an attacker must defeat <em>all three</em>: suppose a syscall
slips past seccomp (a variant you missed, or via a bug) — MAC still denies the
object access; suppose a MAC-policy gap lets an object through — the process
still lacks the capability to escalate, and seccomp still blocks the execve to
spawn a shell; suppose the process is fully root (capability bypass via a
kernel bug) — MAC's mandatory policy still confines it, and seccomp still
blocks the syscalls needed to pivot. Any single layer has holes (seccomp's
path-blindness, AppArmor's path-tricks, capability bugs); the intersection of
three orthogonal restrictions leaves a far smaller reachable set than any one.
Plus the outer layers — namespaces + cgroups (Phase 12) for isolation and
resource bounds, and for the strongest threat models, VM isolation (Kata,
virt Lesson 60) so even a kernel-level escape lands in a throwaway guest kernel.
Modern container/service hardening is exactly this stack, which is why
<code>systemd-analyze security</code> (Lesson 52) scores all of them together:
security is the product of independent containments, not any single wall.
</details>

---

## Homework

Your team moved a web app's files to a new directory and now the (SELinux-
enforcing) server returns 403 — the files are readable by the right UID
(DAC is fine). Walk the diagnosis and fix using this lesson: what does
`ls -Z` reveal, why did moving files break the labels (vs copying), what does
the audit log show, and the two-command fix. Then explain why "the app works
after `setenforce 0`" proves the problem is MAC but is the wrong fix — tying
to the defense-in-depth argument.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Diagnosis: <code>ls -Z /new/path</code> shows the files carry the <em>wrong
type</em> — e.g. <code>default_t</code> or <code>user_home_t</code> instead of
<code>httpd_sys_content_t</code> — so the web server (type <code>httpd_t</code>)
has no policy rule permitting it to read them: denied, 403, even though the
UID/mode (DAC) is correct. Why moving broke it: <code>mv</code> preserves the
file's existing label (it's the same inode relabeled only by directory
context rules that mv doesn't apply), so files dragged from a home dir or
/tmp keep their origin's type; <code>cp</code> into the directory, by
contrast, creates <em>new</em> inodes that inherit the destination's default
type (often correct) — the classic "cp works, mv 403s" SELinux surprise
(inode identity again — Lesson 37). The audit log confirms it:
<code>sudo ausearch -m avc -ts recent</code> (or grep <code>avc: denied</code>
in the log) shows <code>denied { read } ... scontext=...httpd_t
tcontext=...default_t ... name="index.html"</code> — naming exactly the
subject type, object type, and operation denied. Two-command fix: tell the
policy this path should have web content's type and apply it —
<code>sudo semanage fcontext -a -t httpd_sys_content_t "/new/path(/.*)?"</code>
(record the rule) then <code>sudo restorecon -Rv /new/path</code> (relabel the
existing files to match): now their type is <code>httpd_sys_content_t</code>,
policy permits httpd_t→it, 403 gone — with MAC still enforcing. Why
setenforce 0 "proving it" is the wrong fix: disabling enforcement makes the
app work, which confirms the block was MAC (useful diagnostic!) — but it does
so by <em>turning off the entire mandatory-access layer for the whole
system</em>, not just fixing this one label. You've traded a two-command
relabel for permanently removing the containment that would confine <em>every</em>
service if compromised (Q1's compromised-root-daemon scenario) — the outer
wall of the defense-in-depth stack (Q3). The audit log already told you
precisely what to permit; the disciplined move is to read it and grant the
one specific access (fixing the label, or in rarer cases
<code>audit2allow</code> to author a narrow policy rule), keeping the shield up
for everything else. "It works with security off" is a diagnosis, never a
remedy.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 58 — Build and Boot Your Own Kernel →](lesson-58-build-kernel){: .btn .btn-primary }
