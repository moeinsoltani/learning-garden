---
title: "Lesson 42 — Docker Networking from First Principles"
nav_order: 42
parent: "Phase 14: Container & Cloud"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 42: Docker Networking from First Principles

## Concept

Docker's default "bridge" network feels like magic — `docker run` and suddenly a container can reach the internet and you have no idea how. But there is **no magic**: Docker assembles primitives you've already built by hand. A container is a **network namespace** (Lesson 1). Its `eth0` is one end of a **veth pair** (Lesson 7). The other end plugs into a **bridge** called `docker0` (Lesson 11). Outbound traffic is **SNAT/masqueraded** (Lesson 22) so it can use the host's IP on the real network.

```
   ┌─ container netns ─┐
   │  eth0 172.17.0.2  │   (veth, renamed eth0 inside)
   └─────────┬─────────┘
             │ veth pair
   ┌─────────┴───────────────────────────────────┐  host netns
   │  vethXXXX ── docker0 (bridge, 172.17.0.1) ── │
   │                       │                       │
   │              nft MASQUERADE (SNAT)            │
   │                       │                       │
   │                  eth0 (host's real NIC)  ─────┼──► internet
   └───────────────────────────────────────────────┘
```

By the end of this lesson you'll build that **entire diagram by hand**, no Docker involved — and then `docker network inspect bridge` will hold zero surprises.

---

## How it works

The pieces and the role each plays:

1. **Namespace = the container's network stack.** Its own interfaces, routes, ARP table, isolated from the host (Lessons 1–3).
2. **veth pair = the virtual cable** connecting the container namespace to the host. One end lives inside the namespace (renamed `eth0`), the other stays in the host (Lesson 7).
3. **`docker0` bridge = the software switch.** All containers' host-side veth ends attach to it, so containers on the same bridge talk to each other at L2, and the bridge's IP (`172.17.0.1`) is the containers' **default gateway** (Lesson 11).
4. **Default route + gateway.** Inside the container, `default via 172.17.0.1` sends off-subnet traffic to the bridge, where the host (with `ip_forward=1`) routes it onward (Lesson 17).
5. **MASQUERADE (SNAT).** The container's `172.17.0.0/16` address is private and unroutable on the real network, so on the way out the host **rewrites the source** to its own IP; conntrack remembers the flow so replies come back and get rewritten back to the container (Lessons 22–23).

That's the whole "bridge network." Port publishing (`-p 8080:80`) adds one more piece: a **DNAT** rule that rewrites inbound traffic to the host's port toward the container's IP:port.

{: .note }
> **"Container networking" is just orchestration of primitives**
> Docker's contribution here is not a new networking technology — it's *automation*: it creates the namespace, makes the veth pair, attaches one end to `docker0`, assigns an IP from the bridge subnet, sets the default route, and installs the masquerade/DNAT rules — all on `docker run`, and tears them down on stop. Everything it programs, you can program with `ip` and `nft`.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `ip netns add <name>` | Create the container's namespace |
| `ip link add ... type veth peer name ...` | Create the virtual cable |
| `ip link set <veth> netns <name>` | Move one end into the namespace |
| `ip link set <veth> master <bridge>` | Attach host-side end to the bridge |
| `nft add rule ip nat postrouting ... masquerade` | SNAT for outbound traffic |
| `docker network inspect bridge` | Compare your build to real Docker |

---

## Lab

We'll **manually reconstruct** Docker's default bridge network and get a process in the namespace onto the internet. (Run on the Linux VM with a working uplink. The host's real NIC is called `eth0` here — substitute yours from `ip route get 8.8.8.8`.)

### Step 1 — Create the bridge (our docker0)

```bash
$ sudo ip link add mybr0 type bridge
$ sudo ip addr add 172.18.0.1/16 dev mybr0
$ sudo ip link set mybr0 up
```

### Step 2 — Create the container namespace and veth pair

```bash
$ sudo ip netns add c1
$ sudo ip link add veth-c1 type veth peer name veth-h1
$ sudo ip link set veth-c1 netns c1            # container end
$ sudo ip link set veth-h1 master mybr0        # host end → bridge
$ sudo ip link set veth-h1 up
```

### Step 3 — Configure the container's interface

```bash
$ sudo ip netns exec c1 ip link set lo up
$ sudo ip netns exec c1 ip link set veth-c1 name eth0      # rename, like Docker
$ sudo ip netns exec c1 ip addr add 172.18.0.2/16 dev eth0
$ sudo ip netns exec c1 ip link set eth0 up
$ sudo ip netns exec c1 ip route add default via 172.18.0.1
```

### Step 4 — Enable forwarding and masquerade on the host

```bash
$ sudo sysctl -w net.ipv4.ip_forward=1
$ sudo nft add table ip nat
$ sudo nft 'add chain ip nat postrouting { type nat hook postrouting priority 100 ; }'
$ sudo nft add rule ip nat postrouting ip saddr 172.18.0.0/16 oifname "eth0" masquerade
```

### Step 5 — Test connectivity, layer by layer

```bash
$ sudo ip netns exec c1 ping -c2 172.18.0.1     # gateway (bridge) — L2/L3 local
$ sudo ip netns exec c1 ping -c2 8.8.8.8         # internet — proves routing + NAT
$ sudo ip netns exec c1 ping -c2 google.com      # needs DNS (see note)
```

If `8.8.8.8` works but names don't, set a resolver inside the namespace (Docker injects `/etc/resolv.conf`):

```bash
$ sudo mkdir -p /etc/netns/c1
$ echo "nameserver 8.8.8.8" | sudo tee /etc/netns/c1/resolv.conf
```

### Step 6 — Watch the SNAT happen

```bash
# In one terminal, capture on the host's real NIC:
$ sudo tcpdump -ni eth0 icmp
# In another, ping from the namespace — outbound packets show the HOST's source IP, not 172.18.0.2
$ sudo ip netns exec c1 ping -c2 8.8.8.8
$ sudo conntrack -L | grep 8.8.8.8              # the translated flow
```

### Step 7 — Compare to real Docker

```bash
$ docker network inspect bridge                 # Subnet 172.17.0.0/16, Gateway 172.17.0.1
$ ip addr show docker0                          # the real bridge — same shape as mybr0
$ sudo nft list ruleset | grep -i masquerade    # Docker's own SNAT rule
```

### Step 8 — Clean up

```bash
$ sudo ip netns delete c1
$ sudo ip link del mybr0
$ sudo nft delete table ip nat
$ sudo rm -rf /etc/netns/c1
```

---

## Further Reading

| Topic | Link |
|---|---|
| Docker bridge networking | [docs.docker.com — bridge networks](https://docs.docker.com/network/drivers/bridge/) |
| libnetwork (Docker's net) | [moby/libnetwork](https://github.com/moby/libnetwork) |
| Linux bridge | [Wikipedia — Bridging (networking)](https://en.wikipedia.org/wiki/Bridging_(networking)) |
| veth | [man7.org — veth(4)](https://man7.org/linux/man-pages/man4/veth.4.html) |
| Network namespaces | [man7.org — network_namespaces(7)](https://man7.org/linux/man-pages/man7/network_namespaces.7.html) |

---

## Checkpoint

**Q1. Reconstruct Docker's default bridge network from scratch — name every component and the role it plays.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>

1. **Network namespace** — the container's isolated network stack (own interfaces, routes, ARP), created with `ip netns add`.
2. **veth pair** — the virtual cable; one end goes inside the namespace and is renamed **`eth0`**, the other (`vethXXXX`) stays in the host.
3. **Bridge (`docker0`)** — a software L2 switch in the host; the container's host-side veth end attaches to it (`master docker0`). The bridge has an IP (`172.17.0.1`) that serves as the **default gateway** for all containers on it.
4. **IP + default route in the container** — an address from the bridge subnet (`172.17.0.2/16`) and `default via 172.17.0.1`, so off-subnet traffic goes to the gateway.
5. **`ip_forward=1` on the host** — lets the host route packets between the bridge and the real NIC.
6. **MASQUERADE (SNAT) rule** — rewrites the container's private source IP to the host's real IP on egress; **conntrack** tracks the flow so replies are translated back.

Optionally, **DNAT** rules implement published ports (`-p`). Put together, a process in the namespace can reach the internet, with replies finding their way back via conntrack. Docker simply automates all of this per container.
</details>

---

**Q2. The container has source IP 172.18.0.2, which is private and not routable on the internet. How does a reply from 8.8.8.8 ever get back to it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because of **SNAT/masquerade plus connection tracking**. On the way out, the host's POSTROUTING masquerade rule rewrites the packet's **source** from `172.18.0.2` to the host's own (routable) IP, and **conntrack** records the original flow (orig src `172.18.0.2:port` ↔ reply expecting the host IP:port). The remote server replies to the **host's** IP — which *is* routable — so the reply reaches the host. There, conntrack matches the reply to the stored flow and performs the **reverse translation**, rewriting the destination back to `172.18.0.2`, and the kernel forwards it across the bridge into the container. The container never knows translation happened. This is exactly the NAT/conntrack mechanism from Lessons 22–23, applied to containers.
</details>

---

**Q3. When you run `docker run -p 8080:80 nginx`, what networking primitive makes the published port work, and how does it differ from outbound masquerade?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Published ports are implemented with **DNAT** (destination NAT) in the PREROUTING chain, the mirror image of the outbound SNAT/masquerade. Masquerade rewrites the **source** address of *outbound* traffic so private containers can use the host's IP on the internet. A published port instead rewrites the **destination** of *inbound* traffic: a packet arriving at the host on `:8080` has its destination rewritten to the container's `IP:80`, then is forwarded across the bridge to the container (conntrack again handles the return path). So: masquerade = SNAT on egress (let containers out); port publishing = DNAT on ingress (let the world in to a specific container port). Both rely on conntrack to translate the reverse direction automatically.
</details>

---

## Homework

Add a **second** container namespace (`c2`, `172.18.0.3/16`) to your manual bridge from the lab. Verify that `c1` and `c2` can ping each other **directly** (no NAT, no routing — they're on the same bridge/subnet) by capturing on the bridge and confirming the packets carry their *real* private IPs. Then explain why container-to-container traffic on the same bridge does **not** get masqueraded, even though container-to-internet traffic does.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
After adding `c2` with another veth pair into `mybr0`, `c1` can `ping 172.18.0.3` directly. A `tcpdump -ni mybr0` shows ICMP between `172.18.0.2` and `172.18.0.3` with their **real private addresses** — no translation. They're on the **same subnet** (`172.18.0.0/16`) and the **same bridge**, so this is pure **L2 switching**: ARP resolves the peer's MAC and the bridge forwards the frame; the traffic is *local* and never hits a routing/POSTROUTING decision that would NAT it.

Masquerade only applies to **internet-bound** traffic for two reasons. First, the masquerade rule is scoped to packets **leaving via the host's uplink** (`oifname "eth0"`) — same-bridge traffic never egresses that interface, so the rule simply doesn't match. Second, NAT is only *needed* when the source IP is unroutable at the destination: the private `172.18.0.x` addresses are perfectly routable **within** the bridge subnet, so peers can reach each other unmodified; they're only unroutable out on the public internet, which is exactly where (and only where) SNAT kicks in. In short: same-subnet = L2 forwarding, no NAT; off-subnet to the internet = routing + SNAT.
</details>
