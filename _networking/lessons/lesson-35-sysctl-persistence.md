---
title: "Lesson 35 — sysctl Persistence"
nav_order: 35
parent: "Phase 10: sysctl"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 35: sysctl Persistence

## Concept

A `sysctl -w` change vanishes on reboot (Lesson 34). To make it stick, you write it to a configuration file that the system applies at boot. This lesson covers *where* those files live, *how* they're ordered, and a subtle but crucial point: some sysctls are **per-namespace** and some are **global** — which determines whether a container can change a setting for itself or whether it affects the whole host.

```
   runtime:     sysctl -w net.ipv4.ip_forward=1     (now, but lost on reboot)
   persistent:  echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/10-forwarding.conf
                                                    (applied every boot)
```

---

## How it works

At boot, systemd (via `systemd-sysctl`) reads sysctl settings from several locations and applies them. The files:

- `/etc/sysctl.conf` — the traditional single file (still honored).
- `/etc/sysctl.d/*.conf` — drop-in directory; the modern, modular approach.
- `/run/sysctl.d/*.conf` and `/usr/lib/sysctl.d/*.conf` — for runtime and package-provided defaults.

Files are processed in **lexicographic order by filename** across all directories, and **later settings override earlier ones**. So `99-custom.conf` overrides `10-defaults.conf`. This is why convention prefixes filenames with numbers — to control precedence.

To apply a file without rebooting:

```bash
sysctl -p /etc/sysctl.d/10-forwarding.conf      # load one file
sysctl --system                                  # reload all standard locations
```

**Namespaced vs global sysctls** — the key distinction:

| Sysctl | Namespaced? | Meaning |
|---|---|---|
| `net.ipv4.ip_forward` | **Yes** (per netns) | Each network namespace has its own value. A container can be a router without affecting the host. |
| `net.ipv4.conf.*` (per-interface) | **Yes** | Interface settings live with the interface's namespace. |
| `net.netfilter.nf_conntrack_max` | Mostly global | Conntrack table sizing tends to be host-wide. |
| `net.core.rmem_max` / `wmem_max` | **No** (global) | Socket buffer maxes are host-global; a namespace can't change them independently. |

A sysctl is "namespaced" if the kernel keeps a separate copy per network namespace. Most `net.ipv4.*` and `net.ipv6.*` *routing/interface* knobs are namespaced; several `net.core.*` resource limits are not.

{: .note }
> **Why namespacing matters for containers**
> When you run a container, it gets its own network namespace. Because `net.ipv4.ip_forward` is namespaced, the container runtime (or you) can set it to `1` inside the container to make it forward packets, and the **host's** forwarding setting is completely unaffected — they're independent values. But a non-namespaced sysctl like `net.core.rmem_max` is shared: a container *cannot* raise its own socket buffer ceiling beyond the host's, because there's only one value for the whole machine. This is why some performance tunables must be set on the host, not in the container, and why container security relies partly on which sysctls are namespaced (a container shouldn't be able to change host-global kernel behavior).

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `sysctl -p <file>` | Apply settings from a specific file |
| `sysctl --system` | Reload all standard sysctl.d locations |
| `echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/10-forwarding.conf` | Persist a setting |
| `ls /etc/sysctl.d/ /usr/lib/sysctl.d/` | See the drop-in files and their order |
| `ip netns exec <ns> sysctl <name>` | Read a namespaced sysctl's per-ns value |

---

## Lab

We'll persist a setting, prove ordering/precedence, and demonstrate that `ip_forward` is namespaced.

### Step 1 — Persist IP forwarding

```bash
$ echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/10-forwarding.conf
$ sudo sysctl -p /etc/sysctl.d/10-forwarding.conf
net.ipv4.ip_forward = 1
```

After a reboot this would be re-applied automatically. Verify it's loaded now:

```bash
$ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
```

### Step 2 — Demonstrate file precedence

```bash
# A lower-numbered file sets 0...
$ echo 'net.ipv4.conf.all.log_martians=0' | sudo tee /etc/sysctl.d/10-test.conf
# ...a higher-numbered file overrides to 1
$ echo 'net.ipv4.conf.all.log_martians=1' | sudo tee /etc/sysctl.d/99-test.conf
$ sudo sysctl --system 2>/dev/null
$ sysctl net.ipv4.conf.all.log_martians
net.ipv4.conf.all.log_martians = 1     # 99- won over 10- (later overrides earlier)
```

`--system` processes all locations in lexicographic order; `99-test.conf` is applied last, so its value wins.

### Step 3 — Prove `ip_forward` is namespaced

```bash
# Host has forwarding ON (from step 1)
$ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1

# A NEW namespace starts with its OWN value (default 0), independent of the host
$ sudo ip netns add c1
$ sudo ip netns exec c1 sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 0          # NOT 1 — the namespace has its own copy

# Set it inside the namespace; the host is unaffected
$ sudo ip netns exec c1 sysctl -w net.ipv4.ip_forward=1
$ sudo ip netns exec c1 sysctl net.ipv4.ip_forward   # 1
$ sysctl net.ipv4.ip_forward                          # still 1 (host) — but the two are independent
```

To make the point cleaner, set the host to 0 and the namespace to 1 (or vice versa) and confirm they hold different values simultaneously — proof the sysctl is per-namespace.

### Step 4 — Contrast with a global sysctl

```bash
# A global (non-namespaced) sysctl shows the SAME value everywhere
$ sysctl net.core.rmem_max
net.core.rmem_max = 212992
$ sudo ip netns exec c1 sysctl net.core.rmem_max
net.core.rmem_max = 212992          # identical — there's only one host-wide value
# Changing it inside the namespace changes it for the whole host (or is disallowed),
# because it is not namespaced.
```

### Step 5 — Clean up

```bash
$ sudo rm /etc/sysctl.d/10-test.conf /etc/sysctl.d/99-test.conf
$ sudo ip netns delete c1
# (leave /etc/sysctl.d/10-forwarding.conf if you want forwarding to persist)
```

---

## Further Reading

| Topic | Link |
|---|---|
| sysctl.conf / sysctl.d | [man7.org — sysctl.d(5)](https://man7.org/linux/man-pages/man5/sysctl.d.5.html) |
| systemd-sysctl | [man7.org — systemd-sysctl(8)](https://man7.org/linux/man-pages/man8/systemd-sysctl.service.8.html) |
| Linux namespaces | [man7.org — namespaces(7)](https://man7.org/linux/man-pages/man7/namespaces.7.html) |
| Network namespace | [Wikipedia — Linux namespaces](https://en.wikipedia.org/wiki/Linux_namespaces) |

---

## Checkpoint

**Q1. Why doesn't setting `net.ipv4.ip_forward=1` inside a container affect the host?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because `net.ipv4.ip_forward` is a **namespaced** sysctl: the kernel maintains a **separate copy of the value for each network namespace**. A container runs in its own network namespace, so when you set `ip_forward=1` inside it, you're modifying *that namespace's* copy — the host's network namespace has its own independent copy that is untouched. They're different variables that happen to share a name. This isolation is intentional and important: it lets a container act as a router for its own internal networking without turning the host into a router (which would have security and routing implications for everything else on the machine). The same applies to per-interface `conf` settings — they belong to whichever namespace the interface lives in. Contrast this with a *non*-namespaced sysctl (like `net.core.rmem_max`), where there's a single host-wide value and a container can't have its own — changing it would affect the whole machine. So whether a container's sysctl change stays local depends entirely on whether that particular sysctl is namespaced.
</details>

---

**Q2. You have `net.ipv4.ip_forward=0` in `/etc/sysctl.d/10-base.conf` and `net.ipv4.ip_forward=1` in `/etc/sysctl.d/90-router.conf`. After boot, what is the value, and why?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The value is **`1`**. The sysctl files in `/etc/sysctl.d/` (and the other standard locations) are processed in **lexicographic order by filename**, and **later-applied settings override earlier ones**. Since `90-router.conf` sorts after `10-base.conf`, it's applied last, so its `ip_forward=1` overrides the earlier `ip_forward=0`. This numeric-prefix convention exists precisely to control precedence: lower numbers for base/defaults, higher numbers for overrides that should win. If you wanted `0` to win instead, you'd either change the value in the higher-numbered file or rename the files so the `=0` one sorts later. The takeaway: with drop-in config directories, *filename ordering determines the final value when the same key appears in multiple files* — the last write wins, and filename sort order defines "last."
</details>

---

**Q3. Which of these can a container meaningfully set for *itself* without affecting the host: `net.ipv4.ip_forward`, `net.ipv4.conf.eth0.rp_filter`, `net.core.wmem_max`? Explain.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
- **`net.ipv4.ip_forward` — yes, for itself.** It's namespaced; the container has its own copy and can enable forwarding inside its own network namespace without touching the host's value.
- **`net.ipv4.conf.eth0.rp_filter` — yes, for itself.** Per-interface `conf` sysctls are namespaced and bound to the interface's namespace. If `eth0` lives in the container's namespace, the container controls its rp_filter independently of the host.
- **`net.core.wmem_max` — no.** Socket buffer maximums under `net.core.*` are **global (not namespaced)**: there is a single host-wide value. A container can't carve out its own higher (or different) limit; any change applies to the whole machine (and is often disallowed from within an unprivileged container precisely because it isn't namespaced).

The general rule: **routing and per-interface networking knobs tend to be namespaced** (each network namespace gets its own value), so containers can tune them locally, while several **`net.core.*` resource limits are global**, shared by the whole host. This distinction matters operationally (some performance tunables must be set on the host, not in the container) and for security (containers shouldn't be able to change host-global kernel behavior). When in doubt, check by reading the sysctl inside a fresh namespace versus the host — if they can differ, it's namespaced.
</details>

---

## Homework

Create two namespaces, `nsA` and `nsB`. Set `net.ipv4.ip_forward` to `1` in `nsA` and `0` in `nsB`, and confirm they hold different values simultaneously (and differ from the host). Then pick one *global* sysctl and try to set it differently in each namespace; observe what happens. Finally, write a persistent `/etc/sysctl.d/` file that sets two networking tunables, use `sysctl --system` to apply it, and verify. Summarize the rule for predicting whether a given sysctl can be set per-namespace.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Namespaced sysctl (ip_forward):** after `ip netns exec nsA sysctl -w net.ipv4.ip_forward=1` and `...nsB... =0`, reading the value in each namespace returns the value you set there — `nsA`=1, `nsB`=0 — simultaneously, and both can differ from the host. This proves the kernel keeps a separate copy per network namespace; the three values (host, nsA, nsB) are fully independent.

**Global sysctl (e.g. net.core.rmem_max):** trying to set it to different values in nsA vs nsB doesn't give you per-namespace values — there's only one host-wide copy. Either the change isn't permitted from within the namespace (common for unprivileged/global knobs), or if it does apply, it changes the value for the *entire host* (and thus both namespaces and the host read the same number). You cannot make nsA and nsB hold different values for a non-namespaced sysctl, because no per-namespace storage exists for it.

**Persistent file:** e.g.
```
# /etc/sysctl.d/30-mynet.conf
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 2
```
Then `sudo sysctl --system` applies it (processing all sysctl.d locations in lexicographic order), and `sysctl net.ipv4.ip_forward net.ipv4.conf.all.rp_filter` confirms the values. On reboot they're re-applied automatically.

**Rule for predicting per-namespace settability:** a sysctl can be set per-namespace **if and only if it is *namespaced*** — i.e., the kernel maintains a separate value per network namespace for it. In practice, **routing and per-interface networking knobs** (`net.ipv4.ip_forward`, `net.ipv4.conf.<if>.*`, `net.ipv6.conf.<if>.*`, neighbor/route settings) are namespaced and thus per-namespace settable, while **host-wide resource limits** (`net.core.rmem_max`/`wmem_max` and similar `net.core.*`, and some `net.netfilter.*` table sizes) are global and shared. The empirical test: read the sysctl in a fresh namespace and on the host — if they can independently differ, it's namespaced and you can set it per-namespace; if they always track each other, it's global and must be set on the host.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 36 — eBPF Fundamentals →](lesson-36-ebpf-fundamentals){: .btn .btn-primary }
