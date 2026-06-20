---
title: "Lesson 42 — libvirt Virtual Networks"
nav_order: 42
parent: "Phase 10: libvirt"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 42: libvirt Virtual Networks

## Concept

libvirt's networking ties the two curricula together: a libvirt **virtual network** is
just the networking primitives you already know — a **bridge**, **dnsmasq**, and
**nftables/iptables NAT** — wired together and managed declaratively. The famous
"default network that just works" is `virbr0` + dnsmasq + masquerade rules.

```
   libvirt "default" NAT network = three primitives you already know:
   ┌────────────────────────────────────────────────────────────┐
   │  virbr0  (a Linux BRIDGE, Networking L11)                   │
   │     │  guests' vNICs attach here                            │
   │  dnsmasq  (DHCP + DNS for the guests)                       │
   │     │  hands out 192.168.122.x                              │
   │  nftables/iptables MASQUERADE  (NAT, Networking L22)        │
   │     └─► out the host's real uplink to the internet          │
   └────────────────────────────────────────────────────────────┘
```

---

## How It Works

### Network modes

| Mode | Behavior |
|---|---|
| **NAT** (default) | Guests on a private subnet behind `virbr0`; outbound via masquerade; inbound only via forwards. Like SLIRP but kernel-based and shared. The default `default` network. |
| **routed** | Guests on a separate subnet that is *routed* (not NAT'd) to the LAN — needs the LAN to know the route back. No address translation. |
| **bridged** | The guest joins an existing host bridge directly on the LAN (Lesson 32) — real LAN presence. |
| **isolated** | A private bridge with no uplink; guests talk only to each other and the host. |
| **open** | Like routed but libvirt adds no firewall rules — you manage filtering yourself. |

### The default NAT network, dissected

When libvirt starts the `default` network, it:

1. Creates the bridge **`virbr0`** (Networking Lesson 11) and gives it `192.168.122.1`.
2. Launches a **dnsmasq** instance bound to `virbr0` providing **DHCP** (leases in
   `192.168.122.2–254`) and **DNS** forwarding for the guests.
3. Installs **nftables/iptables** rules: a **MASQUERADE** (SNAT) so guest traffic exits
   via the host's real uplink (Networking Lesson 22 — NAT), plus forward rules.

So the "magic" default network is literally bridge + dnsmasq + NAT — every piece is
something from the networking track. Each guest's `<interface type='network'>` gets a
TAP enslaved to `virbr0`.

### Managing networks

```
   virsh net-list --all
   virsh net-dumpxml default          # see the bridge name, subnet, DHCP range
   virsh net-define mynet.xml         # define from XML
   virsh net-start mynet ; virsh net-autostart mynet
   virsh net-destroy mynet            # stop (doesn't delete)
   virsh net-undefine mynet           # remove definition
```

A network XML for an isolated or NAT network specifies `<forward mode='nat'/>`, the
`<bridge name='virbr0'/>`, and an `<ip>` block with a `<dhcp>` range.

### Connecting to an existing host bridge

For real LAN presence (Lesson 32), you don't even need a libvirt-managed network — point
the domain interface at an existing bridge:

```xml
   <interface type='bridge'>
     <source bridge='br0'/>      <!-- a host bridge YOU created (Lesson 32) -->
     <model type='virtio'/>
   </interface>
```

{: .note }
> **The three primitives libvirt wires together for "it just works"**
> libvirt's default NAT network = (1) a Linux <strong>bridge</strong> (virbr0) acting as
> the virtual switch for guests, (2) a <strong>dnsmasq</strong> process giving the guests
> DHCP and DNS, and (3) <strong>nftables/iptables MASQUERADE</strong> (NAT) so guest
> traffic reaches the internet via the host's uplink. All three come straight from the
> networking curriculum (bridges = L11, NAT = L22). libvirt just assembles and manages
> them so a fresh VM gets an IP and internet with zero configuration.

---

## Lab

```bash
# 1. Look at the default network and find its bridge + subnet + DHCP range:
$ virsh net-list --all
 Name      State    Autostart   Persistent
---------------------------------------------
 default   active   yes         yes
$ virsh net-dumpxml default
<network>
  <name>default</name>
  <bridge name='virbr0' stp='on'/>
  <forward mode='nat'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp><range start='192.168.122.2' end='192.168.122.254'/></dhcp>
  </ip>
</network>

# 2. Find the bridge libvirt created (Networking L11):
$ ip addr show virbr0
virbr0: ... inet 192.168.122.1/24 ...
$ bridge link show | grep virbr0     # guest TAPs (vnetN) enslaved here

# 3. Find the dnsmasq instance serving DHCP/DNS for this network:
$ ps aux | grep -E 'dnsmasq.*virbr0' | grep -v grep
libvirt+ ... /usr/sbin/dnsmasq --conf-file=/var/lib/libvirt/dnsmasq/default.conf ...
$ cat /var/lib/libvirt/dnsmasq/default.conf | grep -E 'interface|dhcp-range'

# 4. Find the NAT/masquerade rules (Networking L22) libvirt installed:
$ sudo nft list ruleset | grep -iE 'masquerade|192.168.122' 
# (or, if iptables backend:)
$ sudo iptables -t nat -L LIBVIRT_PRT -n 2>/dev/null | grep MASQUERADE

# 5. See active DHCP leases handed to guests:
$ virsh net-dhcp-leases default
 Expiry         MAC                IP Address          Hostname
 ...            52:54:00:ab:cd:ef  192.168.122.45/24   web01

# 6. Define an ISOLATED network (no uplink, no NAT) for a private lab segment:
$ cat > isolated.xml <<'EOF'
<network>
  <name>labnet</name>
  <bridge name='virbr-lab'/>
  <ip address='10.10.0.1' netmask='255.255.255.0'>
    <dhcp><range start='10.10.0.10' end='10.10.0.100'/></dhcp>
  </ip>
</network>
EOF
$ virsh net-define isolated.xml && virsh net-start labnet
```

**Expected result:** `net-dumpxml default` reveals the bridge (`virbr0`), the
`192.168.122.0/24` subnet, and the DHCP range. You can find the matching `virbr0`
interface, the dnsmasq process, and the masquerade rules — each one a concept straight
from the networking curriculum.

---

## Further Reading

| Topic | Link |
|---|---|
| Networking Lesson 11 (bridges) | [Linux bridges]({{ '/networking/lessons/lesson-11-bridges.html' | relative_url }}) |
| Networking Lesson 22 (NAT) | [NAT with nftables]({{ '/networking/lessons/lesson-22-nat.html' | relative_url }}) |
| libvirt networking | [libvirt.org — Networking](https://libvirt.org/formatnetwork.html) |
| libvirt network XML | [libvirt.org — Network XML format](https://libvirt.org/formatnetwork.html) |
| dnsmasq | [Wikipedia — dnsmasq](https://en.wikipedia.org/wiki/Dnsmasq) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. libvirt's default network "just works" with NAT. Name the three networking primitives (from the other curriculum) it wires together to do that.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) A Linux bridge (virbr0) acting as the virtual switch guests attach to (Networking Lesson 11). (2) A dnsmasq instance bound to that bridge providing DHCP (handing out 192.168.122.x leases) and DNS forwarding. (3) nftables/iptables MASQUERADE (source NAT) so guest traffic is translated and sent out the host's real uplink to the internet (Networking Lesson 22). libvirt assembles and manages these three so a fresh VM gets an address and internet access with no configuration.
</details>

---

**Q2. What's the difference between libvirt's NAT mode and bridged mode for guest networking?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
NAT mode puts guests on a private subnet behind virbr0; their outbound traffic is masqueraded out the host's uplink, and they aren't directly reachable from the LAN (inbound needs explicit forwarding) — like a home router. Bridged mode attaches the guest directly to a host bridge that includes the physical NIC (Lesson 32), so the guest is a first-class L2 citizen on the real LAN with its own LAN IP, reachable by any machine. NAT is convenient and isolated by default; bridged gives true LAN presence.
</details>

---

**Q3. How would you give a domain real LAN presence using an existing host bridge `br0` instead of a libvirt-managed network?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Point the domain's interface at the host bridge directly with an <code>&lt;interface type='bridge'&gt;</code> element whose <code>&lt;source bridge='br0'/&gt;</code> names the bridge you created in Lesson 32, with <code>&lt;model type='virtio'/&gt;</code>. libvirt then creates a TAP and enslaves it to br0, so the guest joins the same L2 segment as the host's physical NIC and gets a real LAN IP from the network's DHCP — no libvirt NAT network involved.
</details>

---

## Homework

Inspect the default network end-to-end: run `virsh net-dumpxml default`, then find on the host (a) the bridge interface, (b) the dnsmasq process serving it, and (c) the masquerade rule. Map each to the specific networking-track lesson that introduced it. Then run `virsh net-dhcp-leases default` while a VM is running and confirm the lease matches what the guest sees with `ip addr`.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) <code>ip addr show virbr0</code> shows the bridge at 192.168.122.1 — the Linux bridge from Networking Lesson 11. (b) <code>ps aux | grep dnsmasq</code> shows the dnsmasq bound to virbr0 reading default.conf — providing DHCP/DNS. (c) <code>nft list ruleset</code> (or iptables -t nat) shows a MASQUERADE rule for 192.168.122.0/24 — NAT from Networking Lesson 22. Running <code>virsh net-dhcp-leases default</code> lists the guest's MAC→IP (e.g. 192.168.122.45), which matches the address the guest reports with <code>ip addr</code> inside it — confirming dnsmasq leased that address over the bridge. This demonstrates the default network is literally bridge + dnsmasq + NAT assembled by libvirt.
</details>
