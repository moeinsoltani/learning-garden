---
title: "Lesson 41 — Other Persistence Approaches"
nav_order: 41
parent: "Phase 13: Persistence"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 41: Other Persistence Approaches

## Concept

systemd-networkd (Lesson 40) is one way to persist network config — but you'll walk onto machines that use something else, and you need to recognize them and know which to choose. They mostly fall into two camps: **renderers** that ultimately program the same kernel objects you already understand, and the **per-subsystem persistence** for firewall/sysctl/tc that lives alongside whatever manages interfaces.

```
   You (declare intent)
        │
        ├── netplan (YAML)  ──renders to──►  systemd-networkd  ─┐
        │                                  or NetworkManager   ─┤
        ├── /etc/network/interfaces (ifupdown, legacy)         ─┤──► same kernel:
        ├── NetworkManager (nmcli/nmtui)                       ─┤    links, addrs,
        ├── systemd-networkd (Lesson 40)                       ─┘    routes
        │
        ├── nft -f /etc/nftables.conf       ──► firewall ruleset
        ├── /etc/sysctl.d/*.conf            ──► kernel tunables
        └── systemd unit running tc commands ──► qdiscs/filters
```

The key realization: **these tools are interchangeable front-ends to the same kernel state.** Once you understand links, addresses, routes, nftables, and sysctl (which you do), learning a new config system is just learning its *syntax* — the underlying objects are identical.

---

## How it works

**Interface/address/route persistence — pick one manager:**

| Tool | Format | Where it shines | Notes |
|---|---|---|---|
| **netplan** (Ubuntu) | YAML in `/etc/netplan/` | Ubuntu servers & cloud images | A *renderer*: `netplan generate` emits systemd-networkd or NetworkManager config; `netplan apply` activates it. Not a daemon itself. |
| **NetworkManager** | `nmcli` / `nmtui` / keyfiles | Laptops/desktops, Wi-Fi, VPNs, roaming | Event-driven, great for *dynamic* environments; the default on many desktop distros. |
| **systemd-networkd** | `.network`/`.netdev` | Headless servers, containers | Lightweight, declarative, no GUI assumptions (Lesson 40). |
| **ifupdown** | `/etc/network/interfaces` | Older Debian systems | The legacy `ifup`/`ifdown` scheme; still seen in the wild. |

Don't run two interface managers over the same device at once — they'll fight. Choose one per machine.

**Per-subsystem persistence (independent of the interface manager):**

- **nftables:** save a ruleset to `/etc/nftables.conf` (`nft list ruleset > ...`), load it with `nft -f /etc/nftables.conf`, and `systemctl enable nftables` to apply at boot.
- **sysctl:** drop `.conf` files in `/etc/sysctl.d/` (Lesson 35); applied at boot by `systemd-sysctl`, or on demand with `sysctl -p <file>`.
- **tc (qdiscs/filters):** there's no standard config file — persist them with a **systemd service unit** whose `ExecStart` runs your `tc` commands (or a script), ordered after the interface comes up.

{: .note }
> **netplan is a renderer, not a daemon**
> netplan does not itself talk to the kernel continuously. `netplan generate` translates your YAML into config for a *backend* — systemd-networkd (default on servers) or NetworkManager — and that backend does the actual work. So "Ubuntu uses netplan" really means "Ubuntu uses systemd-networkd (or NM), configured via netplan YAML." Understanding Lesson 40 means you already understand what netplan renders to.

---

## Why systemd-networkd over NetworkManager on a server

This is the lesson's checkpoint, so frame it clearly. NetworkManager is **event-driven and interactive**: it's built for machines whose network *changes* — laptops switching Wi-Fi, plugging/unplugging cables, dialing VPNs — and it ships D-Bus APIs and TUIs/GUIs for that. A **headless server** has a *static* network that should come up identically every boot and never surprise you. For that, systemd-networkd is **lighter** (fewer dependencies, smaller footprint), **fully declarative** (plain files, easy to put in config management and review), and makes **no desktop/dynamic assumptions**. You don't want a daemon that might renegotiate or "helpfully" change a server's networking based on link events. Use the simplest tool that matches the workload: static server → networkd; dynamic endpoint → NetworkManager.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `netplan generate` / `netplan apply` | Render YAML to backend config / activate it |
| `nmcli device status`, `nmcli connection show` | Inspect NetworkManager state |
| `nmtui` | Curses UI for NetworkManager |
| `nft -f /etc/nftables.conf` | Load a saved nftables ruleset |
| `systemctl enable nftables` | Restore firewall at boot |
| `sysctl -p /etc/sysctl.d/<f>.conf` | Apply a sysctl drop-in now |

---

## Lab

We'll persist a **firewall** and a **sysctl**, the two subsystem-level persistences you'll use most, then peek at netplan.

### Step 1 — Persist an nftables ruleset

```bash
# Build a tiny ruleset, then save it
$ sudo nft add table inet filter
$ sudo nft 'add chain inet filter input { type filter hook input priority 0 ; policy drop ; }'
$ sudo nft add rule inet filter input ct state established,related accept
$ sudo nft add rule inet filter input tcp dport 22 accept
$ sudo nft list ruleset | sudo tee /etc/nftables.conf
$ sudo systemctl enable nftables          # reload /etc/nftables.conf at boot
```

Verify it reloads from the file:

```bash
$ sudo nft flush ruleset                  # wipe live rules
$ sudo nft -f /etc/nftables.conf          # restore from file
$ sudo nft list ruleset                   # rules are back
```

### Step 2 — Persist a sysctl (forwarding)

```bash
# /etc/sysctl.d/30-forwarding.conf
net.ipv4.ip_forward = 1
```

```bash
$ sudo sysctl -p /etc/sysctl.d/30-forwarding.conf
net.ipv4.ip_forward = 1
$ sysctl net.ipv4.ip_forward             # confirms; also applied automatically at boot
```

### Step 3 — Inspect netplan (Ubuntu) without changing your live network

```bash
$ ls /etc/netplan/
$ sudo cat /etc/netplan/*.yaml           # read the YAML intent
$ sudo netplan generate                  # render to backend (no activation)
$ ls /run/systemd/network/               # see the generated systemd-networkd files!
```

That last step is the punchline: netplan YAML becomes the very `.network` files you learned in Lesson 40.

### Step 4 — (Optional) tc via a systemd unit

```ini
# /etc/systemd/system/shape-eth0.service
[Unit]
After=network-online.target
Wants=network-online.target
[Service]
Type=oneshot
ExecStart=/sbin/tc qdisc add dev eth0 root netem delay 50ms
RemainAfterExit=yes
ExecStop=/sbin/tc qdisc del dev eth0 root
[Install]
WantedBy=multi-user.target
```

```bash
$ sudo systemctl enable --now shape-eth0.service   # tc rules now persist across boots
```

---

## Further Reading

| Topic | Link |
|---|---|
| netplan | [netplan.io documentation](https://netplan.io/) |
| NetworkManager | [NetworkManager docs](https://networkmanager.dev/docs/) |
| `nmcli` | [man7.org — nmcli(1)](https://man7.org/linux/man-pages/man1/nmcli.1.html) |
| `/etc/network/interfaces` | [man7.org — interfaces(5)](https://man7.org/linux/man-pages/man5/interfaces.5.html) |
| nftables persistence | [netfilter.org — nftables wiki](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page) |

---

## Checkpoint

**Q1. Why would you prefer systemd-networkd over NetworkManager on a headless server?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A headless server has a **static** network that should come up identically every boot, with no GUI and no human plugging cables in and out. systemd-networkd fits that: it's **lightweight** (small footprint, few dependencies), **purely declarative** (plain config files you can version-control and review), and makes **no desktop/dynamic assumptions** — it just realizes the files you wrote. NetworkManager is **event-driven and interactive**, designed for machines whose connectivity *changes* — laptops roaming between Wi-Fi networks, VPNs, hotplugging — with D-Bus APIs and TUIs/GUIs to support that. On a static server those capabilities are unnecessary weight, and you specifically *don't* want a daemon that might react to link events and change the server's networking. Match the tool to the workload: static server → systemd-networkd; dynamic/desktop endpoint → NetworkManager.
</details>

---

**Q2. netplan is described as a "renderer." What does that mean, and what actually programs the kernel?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
netplan does not talk to the kernel itself. You write your intent as **YAML** in `/etc/netplan/`, and `netplan generate` **translates** that YAML into native configuration for a **backend** — either **systemd-networkd** (the default on servers) or **NetworkManager**. That backend is what actually creates devices and programs addresses/routes into the kernel. So netplan is a thin, distro-friendly front-end; the real work is done by the same managers you already know. (You can literally see this: after `netplan generate`, the rendered systemd-networkd `.network` files appear under `/run/systemd/network/`.) "Ubuntu uses netplan" therefore means "Ubuntu uses systemd-networkd/NM, configured through netplan YAML."
</details>

---

**Q3. How do you persist (a) an nftables ruleset, (b) a sysctl, and (c) a tc qdisc across reboots?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>

- **(a) nftables:** write the ruleset to `/etc/nftables.conf` (e.g. `nft list ruleset > /etc/nftables.conf`) and `systemctl enable nftables`, so the `nftables.service` runs `nft -f /etc/nftables.conf` at boot.
- **(b) sysctl:** put the setting in a drop-in file under `/etc/sysctl.d/` (e.g. `/etc/sysctl.d/30-forwarding.conf` containing `net.ipv4.ip_forward = 1`); `systemd-sysctl` applies all of `/etc/sysctl.d/` at boot (apply now with `sysctl -p <file>`).
- **(c) tc:** there's **no standard config file**, so persist it with a **systemd service unit** (`Type=oneshot`, `RemainAfterExit=yes`) whose `ExecStart` runs your `tc` commands and `ExecStop` tears them down, ordered `After=network-online.target`; then `systemctl enable` it.

The pattern: interface/address/route persistence is one subsystem (networkd/NM/netplan), but firewall, sysctl, and tc each have their **own** persistence mechanism that you set up independently.
</details>

---

## Homework

You inherit an Ubuntu server. Determine, from the command line only, **which interface manager is actually in charge** (netplan rendering to networkd? to NetworkManager? raw networkd? ifupdown?), and where the firewall and sysctl persistence live. List the exact commands you'd run and what each answer tells you. Then state the risk of "fixing" the network by editing the wrong layer.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Discovery commands:**

- `ls /etc/netplan/` and `cat /etc/netplan/*.yaml` — is netplan present, and what `renderer:` does it declare (`networkd` or `NetworkManager`)?
- `systemctl is-enabled systemd-networkd NetworkManager` and `systemctl status ...` — which backend is actually running/enabled? (`networkctl list` vs `nmcli device status` confirms which one "owns" the links.)
- `ls /run/systemd/network/` — if netplan rendered to networkd, the generated `.network` files appear here, confirming the chain.
- `ls /etc/network/interfaces*` — legacy ifupdown config, if any (older boxes).
- Firewall: `systemctl is-enabled nftables` and `cat /etc/nftables.conf` (or `iptables-save`/`netfilter-persistent` on iptables-based setups).
- sysctl: `ls /etc/sysctl.d/ /etc/sysctl.conf` and `sysctl -a` to see effective values.
- tc: `tc qdisc show` for live rules, then grep systemd units (`systemctl list-units | grep -i tc`, or look for a custom `*.service`) since tc has no standard file.

Each answer pins down one layer: which front-end you should edit (netplan YAML vs networkd files vs nmcli vs interfaces), and where firewall/sysctl/tc state is restored from.

**The risk of editing the wrong layer:** on a netplan system, hand-editing the generated `/run/systemd/network/` files (or running raw `ip` commands) gets **silently overwritten** the next time `netplan apply` or a reboot regenerates them — your fix "doesn't stick" and you waste hours. Worse, editing a layer that *isn't* the active manager (e.g., `/etc/network/interfaces` on a netplan box) does nothing, while two managers touching the same device can fight and flap the link. Always change the **authoritative source** (the YAML/keyfile/.network that the running backend reads), then apply through that tool — never patch the rendered output or mix imperative `ip` changes with a declarative manager on the same interface.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 42 — Docker Networking from First Principles →](lesson-42-docker-networking){: .btn .btn-primary }
