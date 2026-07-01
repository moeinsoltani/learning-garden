---
title: "Lesson 40 — systemd-networkd"
nav_order: 40
parent: "Phase 13: Persistence"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 40: systemd-networkd

## Concept

So far every interface, address, and route you've created was **imperative** — typed with `ip` commands that vanish on reboot. **systemd-networkd** is the **declarative** alternative: you write small `.ini`-style files describing the network you *want*, and the daemon makes reality match them at boot and keeps them that way.

```
   /etc/systemd/network/        systemd-networkd reads these and applies them
   ┌──────────────────────┐
   │ 10-br0.netdev   ──────┼──►  CREATE virtual device br0 (a bridge)
   │ 10-br0.network  ──────┼──►  CONFIGURE br0: address, routes
   │ 20-eth0.network ──────┼──►  CONFIGURE eth0: enslave to br0
   └──────────────────────┘
        files = desired state          daemon = makes it so, every boot
```

The mental shift: stop thinking "run these commands" and start thinking "describe the end state." The same idea you'll meet again in Kubernetes manifests and Terraform — **declarative configuration** that's reproducible, reviewable, and reboot-proof.

---

## How it works

systemd-networkd uses **three file types**, processed from `/etc/systemd/network/` in **lexicographic order** (hence the `NN-` prefixes):

| Extension | Answers | Examples of what it does |
|---|---|---|
| `.netdev` | "What virtual device should *exist*?" | create a bridge, vlan, bond, vxlan, dummy, veth |
| `.network` | "How is this device *configured*?" | assign addresses, routes, policy rules, enslave to a bridge |
| `.link` | "How is a physical device *named/tuned* at the udev layer?" | rename `enp3s0`→`eth0`, set MAC/MTU |

The split is the key idea: a **`.netdev` brings a virtual device into existence**; a **`.network` configures an existing device** (whether physical or one a `.netdev` just created). A bridge therefore needs *both* — a `.netdev` to create `br0`, and a `.network` to give it an address — while a physical NIC needs only a `.network`.

A `.network` file matches a device with a `[Match]` section (by `Name=`, `MACAddress=`, etc.) and configures it with `[Network]`, `[Address]`, `[Route]`, and `[RoutingPolicyRule]` sections. To enslave an interface to a bridge/bond, the *member's* `.network` sets `Bridge=br0` (or `Bond=`) under `[Network]`.

```ini
# 10-br0.netdev — create the bridge device
[NetDev]
Name=br0
Kind=bridge

# 10-br0.network — configure the bridge
[Match]
Name=br0
[Network]
Address=192.168.50.1/24

# 20-eth1.network — put eth1 into the bridge
[Match]
Name=eth1
[Network]
Bridge=br0
```

`networkctl` is the inspection tool: `networkctl list` (all links + state), `networkctl status br0` (detail). `systemd-networkd-wait-online` is a boot-ordering helper that blocks until the network is configured, so services that need the network don't start too early.

{: .note }
> **.netdev vs .network — the one distinction to remember**
> A **`.netdev`** *creates a virtual device* (something that didn't exist — a bridge, vlan, bond, vxlan, dummy, veth). A **`.network`** *configures an already-existing device* (gives it addresses, routes, bridge membership) — that device may be physical, or one a `.netdev` made. Physical NIC: `.network` only. Virtual device: `.netdev` to create it **and** a `.network` to configure it.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `networkctl list` | List all links and their networkd state |
| `networkctl status <dev>` | Detailed state of one device |
| `networkctl reload` | Re-read config files without restarting |
| `systemctl restart systemd-networkd` | Apply changes / re-run from scratch |
| `systemd-networkd-wait-online` | Block until the network is up (boot ordering) |

---

## Lab

We'll recreate a **bridge + two members** topology declaratively, then prove it survives a service restart. (Do this on the Linux VM where systemd-networkd manages the network — *not* over your only SSH link, or you may cut yourself off.)

### Step 1 — Enable the daemon

```bash
$ sudo systemctl enable --now systemd-networkd
$ networkctl list
```

### Step 2 — Create a bridge with two dummy members (safe to experiment with)

```bash
# /etc/systemd/network/10-br0.netdev
[NetDev]
Name=br0
Kind=bridge
```

```bash
# /etc/systemd/network/10-d0.netdev   (dummy member 1)
[NetDev]
Name=d0
Kind=dummy
```

```bash
# /etc/systemd/network/10-d1.netdev   (dummy member 2)
[NetDev]
Name=d1
Kind=dummy
```

### Step 3 — Configure the bridge and enslave the members

```bash
# /etc/systemd/network/20-br0.network
[Match]
Name=br0
[Network]
Address=192.168.50.1/24
```

```bash
# /etc/systemd/network/30-d0.network
[Match]
Name=d0
[Network]
Bridge=br0
```

```bash
# /etc/systemd/network/30-d1.network
[Match]
Name=d1
[Network]
Bridge=br0
```

### Step 4 — Apply and verify

```bash
$ sudo systemctl restart systemd-networkd
$ networkctl list
IDX LINK   TYPE     OPERATIONAL SETUP
  ... br0  bridge   ...         configured
  ... d0   ether    ...         configured
  ... d1   ether    ...         configured

$ ip -br addr show br0
br0   UP   192.168.50.1/24

$ bridge link show          # d0 and d1 show master br0
```

### Step 5 — Prove persistence

```bash
$ sudo systemctl restart systemd-networkd
$ ip -br addr show br0       # 192.168.50.1/24 still there — config reapplied
```

A reboot would do the same: the files *are* the configuration.

### Step 6 — Clean up

```bash
$ sudo rm /etc/systemd/network/{10-br0.netdev,10-d0.netdev,10-d1.netdev,20-br0.network,30-d0.network,30-d1.network}
$ sudo systemctl restart systemd-networkd
$ sudo ip link del br0 2>/dev/null; sudo ip link del d0 2>/dev/null; sudo ip link del d1 2>/dev/null
```

---

## Further Reading

| Topic | Link |
|---|---|
| systemd-networkd | [systemd.network(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.network.html) |
| `.netdev` files | [systemd.netdev(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.netdev.html) |
| `.link` files | [systemd.link(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.link.html) |
| `networkctl` | [networkctl(1)](https://www.freedesktop.org/software/systemd/man/latest/networkctl.html) |
| systemd-networkd (overview) | [Wikipedia — systemd](https://en.wikipedia.org/wiki/Systemd#Ancillary_components) |

---

## Checkpoint

**Q1. What is the difference between a `.netdev` and a `.network` file?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A **`.netdev`** file *creates a virtual network device* — it tells systemd-networkd to bring into existence something that doesn't exist on its own: a bridge, VLAN, bond, VXLAN, dummy, veth, etc. (defined by `Kind=`). A **`.network`** file *configures an already-existing device* — it matches a device (by name/MAC in `[Match]`) and applies addresses, routes, policy rules, DHCP settings, and bridge/bond membership. The device a `.network` configures can be a **physical** NIC or a **virtual** one that a `.netdev` just created. Consequence: a physical interface needs only a `.network`; a virtual device needs **both** a `.netdev` (to create it) and a `.network` (to configure it). One creates, the other configures.
</details>

---

**Q2. Why are the files named with numeric prefixes like `10-`, `20-`, `30-`?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
systemd-networkd processes the files in `/etc/systemd/network/` in **lexicographic (alphabetical) order**, and for `.network` files the **first** one whose `[Match]` matches a given device wins. The numeric prefixes give you explicit, predictable ordering: lower numbers are read first, so you can ensure, e.g., that devices are created/configured in the right sequence and that a specific match file takes precedence over a more general catch-all one. It's the same `NN-name` convention used across systemd drop-in directories — it makes ordering obvious and lets you insert new files between existing ones without renaming everything.
</details>

---

**Q3. Why is a declarative config (files) better than the `ip` commands you used in earlier lessons for a production server?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`ip` commands are **imperative and ephemeral** — they mutate live state and vanish on reboot, so the running configuration only exists in the kernel and in whatever shell history or script you happened to keep. Declarative files describe the **desired end state**, which gives you: **persistence** (re-applied automatically every boot), **reproducibility** (copy the files to build an identical host), **reviewability** (they can live in version control and go through code review), and **idempotency** (re-applying converges to the same state rather than stacking changes). You stop worrying about "which commands did I run in what order" and instead manage "what should this machine's network look like." It's the same reason infrastructure-as-code (Kubernetes manifests, Terraform) won — declarative state is auditable and recoverable in a way ad-hoc commands never are.
</details>

---

## Homework

Take the bridge+VLAN topology from Phase 3 (Lesson 12) and express it entirely in systemd-networkd files: a VLAN-aware bridge, a VLAN interface (`Kind=vlan`, `[VLAN] Id=`), and the address configuration. Restart `systemd-networkd` and confirm with `networkctl status` and `bridge vlan show` that it comes back identically. Then describe one thing that is *harder* to express declaratively this way than with raw `ip` commands, and how you'd handle it.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You'd create: a `.netdev` for the bridge with `[Bridge] VLANFiltering=yes`, a `.netdev` for each VLAN device (`Kind=vlan`, `[VLAN] Id=10`), and `.network` files wiring members into the bridge and assigning addresses; bridge-VLAN membership (PVID/tagged) is set via `[BridgeVLAN]` sections in the member `.network` files. After `systemctl restart systemd-networkd`, `networkctl status` shows everything `configured` and `bridge vlan show` shows the same VID assignments as the manual lab — and crucially it survives reboot.

**What's harder declaratively:** anything **dynamic or namespace-based**. systemd-networkd manages devices in the host (or a given) network namespace and is built around devices that persist; orchestrating per-namespace topologies (like the `ip netns exec` labs), or steps that must happen in a specific runtime order with conditional logic, don't map cleanly onto static `[Match]`-based files. For those you'd still use a script or a tool that drives namespaces (or run a networkd instance per namespace), or use systemd service units / `ExecStart` hooks for the imperative glue, reserving the declarative files for the persistent device/address/route layer. The practical pattern: declarative files for steady-state device/addressing/routing, scripts/units for the genuinely procedural or namespace-spanning parts.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 41 — Other Persistence Approaches →](lesson-41-persistence-alternatives){: .btn .btn-primary }
