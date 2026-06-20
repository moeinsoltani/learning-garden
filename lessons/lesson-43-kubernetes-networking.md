---
title: "Lesson 43 — Kubernetes Networking"
nav_order: 43
parent: "Phase 14: Container & Cloud"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 43: Kubernetes Networking

## Concept

Docker gives each container a private IP behind NAT (Lesson 42). Kubernetes makes a stronger, simpler promise — the **flat pod network**:

> **Every pod gets its own IP, and every pod can reach every other pod directly, with no NAT between them.**

```
   Node 1 (10.0.1.0/24 pods)            Node 2 (10.0.2.0/24 pods)
   ┌──────────────────────┐             ┌──────────────────────┐
   │  Pod A  10.0.1.5      │             │  Pod B  10.0.2.7      │
   │   eth0 ─veth─ cni0 ───┼── routing ──┼─── cni0 ─veth─ eth0   │
   └──────────────────────┘   (or       └──────────────────────┘
                              overlay)
        Pod A talks to Pod B at 10.0.2.7 — its REAL IP, no NAT
```

To a pod, the whole cluster looks like one big flat network where it can dial any other pod by IP. *How* that flat network is realized (plain routing, or a VXLAN overlay, or eBPF) is the job of a **CNI plugin** — and you've already built every mechanism a CNI uses.

---

## How it works

**The pod is a network namespace.** A pod is one network namespace (Lesson 1) shared by **all containers in that pod** — that's why containers in a pod reach each other over `localhost` and share one IP. There's a tiny "pause" container that holds the namespace open.

**CNI (Container Network Interface)** is the contract kubelet uses. When a pod is created, the kubelet calls the configured **CNI plugin** with `ADD`; on delete, `DEL`. The plugin does exactly what you did by hand in Lesson 42:

1. Create/enter the pod's network namespace.
2. Create a **veth pair**; put one end in the pod (as `eth0`), the other in the host.
3. Attach the host end to a **bridge** (or route it directly, depending on plugin).
4. **Assign an IP** from the node's pod CIDR (IP Address Management — IPAM).
5. Set up **routes** so the pod reaches other pods and the outside world.

**Pod-to-pod across nodes** is whatever the CNI arranges: either the underlying network **routes** each node's pod CIDR (e.g., Calico in BGP mode, or cloud route tables), or the CNI builds an **overlay** that encapsulates pod traffic between nodes (Flannel/VXLAN — Lesson 44, and you saw VXLAN itself in Lesson 15).

**Services and kube-proxy.** Pods are ephemeral and their IPs change, so a **Service** gives a stable virtual IP (**ClusterIP**) fronting a set of pods. **kube-proxy** programs the dataplane so traffic to the ClusterIP is **load-balanced (DNAT'd) to a real pod IP** — historically via **iptables** rules, or **IPVS** for scale. A modern **eBPF** dataplane (**Cilium**) replaces kube-proxy entirely: it uses **BPF maps** (Lesson 37) for the service→backend lookup and does the translation in BPF, skipping the iptables chains.

{: .note }
> **The "no NAT between pods" rule is the whole model**
> Kubernetes deliberately forbids NAT *between pods* so that a pod always sees another pod's real, stable-for-its-lifetime IP — which keeps the network model simple for applications and for policy. NAT still appears at the **edges**: ClusterIP Services DNAT to a backend pod, and pod traffic leaving the cluster to the internet is SNAT'd to the node IP (just like Docker). Inside the pod network, though, it's flat and NAT-free.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `kubectl get pods -o wide` | Show pod IPs and which node they're on |
| `kubectl get svc` | Show Services and their ClusterIPs |
| `ip route` (on a node) | See routes to other nodes' pod CIDRs |
| `iptables-save \| grep KUBE` *(or `nft list ruleset`)* | kube-proxy's service DNAT rules |
| `ip netns` / `nsenter -t <pid> -n ip addr` | Inspect a pod's namespace from the node |

---

## Lab

You don't need a full cluster to *see* the primitives. We'll model the cross-node pod path with two namespaces acting as pods and a router namespace acting as "the node-to-node network" — exactly the Lesson 17 topology, reframed as Kubernetes.

### Step 1 — Two "pods" on two "nodes" joined by a router

```bash
# pod-a (10.0.1.5) — node1 subnet 10.0.1.0/24
# pod-b (10.0.2.7) — node2 subnet 10.0.2.0/24
# router-ns simulates the routed underlay between nodes
$ sudo ip netns add pod-a; sudo ip netns add pod-b; sudo ip netns add router

$ sudo ip link add a-r netns pod-a type veth peer name r-a netns router
$ sudo ip link add b-r netns pod-b type veth peer name r-b netns router

$ sudo ip netns exec pod-a ip addr add 10.0.1.5/24 dev a-r
$ sudo ip netns exec router ip addr add 10.0.1.1/24 dev r-a
$ sudo ip netns exec pod-b ip addr add 10.0.2.7/24 dev b-r
$ sudo ip netns exec router ip addr add 10.0.2.1/24 dev r-b

$ for ns in pod-a pod-b router; do sudo ip netns exec $ns ip link set lo up; done
$ sudo ip netns exec pod-a ip link set a-r up
$ sudo ip netns exec pod-b ip link set b-r up
$ sudo ip netns exec router ip link set r-a up
$ sudo ip netns exec router ip link set r-b up
```

### Step 2 — Routing, the "flat network, no NAT" way

```bash
$ sudo ip netns exec router sysctl -w net.ipv4.ip_forward=1
$ sudo ip netns exec pod-a ip route add default via 10.0.1.1
$ sudo ip netns exec pod-b ip route add default via 10.0.2.1
```

### Step 3 — Pod A reaches Pod B by its real IP, no NAT

```bash
$ sudo ip netns exec pod-a ping -c2 10.0.2.7
$ sudo ip netns exec router tcpdump -ni r-b icmp
# the capture shows src 10.0.1.5 dst 10.0.2.7 — the pods' REAL IPs, untranslated
```

That untranslated, end-to-end IP visibility *is* the Kubernetes pod-network promise.

### Step 4 — (Concept) what a Service would add

A ClusterIP like `10.96.0.10` doesn't exist on any interface; kube-proxy installs a DNAT rule: "to `10.96.0.10:80`, rewrite destination to a backend pod, e.g. `10.0.2.7:8080`, and load-balance across backends." Inspect this on a real node with:

```bash
$ sudo iptables-save | grep -E 'KUBE-SERVICES|KUBE-SEP'   # DNAT chains per service/endpoint
# or, on an IPVS cluster:
$ sudo ipvsadm -Ln
```

### Step 5 — Clean up

```bash
$ sudo ip netns delete pod-a; sudo ip netns delete pod-b; sudo ip netns delete router
```

---

## Further Reading

| Topic | Link |
|---|---|
| Kubernetes networking model | [kubernetes.io — cluster networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/) |
| CNI spec | [github.com/containernetworking/cni](https://github.com/containernetworking/cni) |
| Services & kube-proxy | [kubernetes.io — Service](https://kubernetes.io/docs/concepts/services-networking/service/) |
| IPVS mode | [kubernetes.io — IPVS proxy](https://kubernetes.io/docs/reference/networking/virtual-ips/#proxy-mode-ipvs) |
| Cilium (eBPF dataplane) | [cilium.io](https://cilium.io/) |

---

## Checkpoint

**Q1. Trace a packet from Pod A on Node 1 to Pod B on Node 2. Name every hop and transformation.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>

1. **Pod A** (namespace) sends to Pod B's IP. Its routing table has a default route via its `eth0` (one end of a veth).
2. The packet crosses the **veth pair** into **Node 1's** host namespace, arriving on the host-side veth (attached to a CNI bridge or a routed interface).
3. **Node 1 routes** the packet: it has a route for Pod B's CIDR pointing to Node 2 — either *directly* (the underlay/cloud route table or BGP knows Node 2 owns that pod CIDR) or via an **overlay**, where the CNI **encapsulates** the packet (e.g., VXLAN — wraps the pod-to-pod frame in a UDP packet between node IPs).
4. The packet travels the **node-to-node network** to **Node 2** (as a routed pod packet, or as an encapsulated outer packet).
5. On **Node 2**, if encapsulated it's **decapsulated** back to the original pod-to-pod packet; Node 2 routes it to the local veth for Pod B.
6. It crosses Pod B's **veth pair** into Pod B's namespace and is delivered to **`eth0`**.

**Transformations:** crucially, **no NAT** of the pod source/destination IPs — Pod B sees the packet *from* Pod A's real IP. The only optional transformation is **encapsulation/decapsulation** if the CNI uses an overlay (the inner pod IPs are preserved; only an outer node-IP header is added and removed). This is the flat pod network in action.
</details>

---

**Q2. Why is a pod a network namespace shared by all its containers, rather than one namespace per container (like Docker)?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because Kubernetes' unit of deployment is the **pod**, and the model says a pod has **one IP**. All containers in a pod share a **single network namespace** (held open by the "pause" container), so they share that one IP, one set of interfaces, and one localhost. Consequently, containers in the same pod communicate over **`localhost`** and just have to avoid using the same port — exactly like processes on one machine. This makes tightly-coupled co-located containers (e.g., an app plus a sidecar proxy) behave like they're on the same host, which is the intended design. Docker's default gives each *container* its own namespace and IP because its unit is the individual container; Kubernetes raises the unit to the pod and shares the namespace within it.
</details>

---

**Q3. What does kube-proxy do, and how does an eBPF dataplane like Cilium change it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**kube-proxy** implements **Services**. A Service has a stable virtual **ClusterIP** that fronts a changing set of backend pods. kube-proxy watches the API server for Services/Endpoints and programs the node's dataplane so that traffic to a ClusterIP is **DNAT'd and load-balanced to a real backend pod IP** — classically via **iptables** rules (chains like `KUBE-SERVICES`/`KUBE-SEP`), or via **IPVS** for better performance at scale. It also handles return-path translation via conntrack.

An **eBPF dataplane (Cilium)** replaces kube-proxy: instead of long iptables chains, it stores the service→backend mapping in **BPF maps** and performs the ClusterIP→pod translation and load-balancing directly in **eBPF programs** at the socket/TC/XDP hooks (Lessons 36–38). The win is performance and scalability — a BPF map lookup is O(1) and avoids walking large iptables rule sets that grow linearly with the number of services/endpoints — plus richer policy and observability. Same *function* (stable VIP → load-balanced real pods), faster *mechanism* (BPF maps instead of iptables/IPVS).
</details>

---

## Homework

Add a third namespace `pod-c` (10.0.2.8/24) on "node 2" and model a **ClusterIP Service** by hand: pick a fake service IP `10.96.0.10`, and on `router` install an nftables DNAT rule that rewrites traffic destined to `10.96.0.10` to one of the backend pods, then test it from `pod-a`. Then explain what kube-proxy automates that you did manually, and why ClusterIP translation is the *one* place NAT legitimately appears inside the cluster.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Manual Service:** on `router` (acting as the node doing service handling), add a NAT table/chain and a PREROUTING DNAT rule, e.g. `nft add rule ip nat prerouting ip daddr 10.96.0.10 tcp dport 80 dnat to 10.0.2.7:8080` (and a route so `pod-a` sends `10.96.0.10` toward the router). A request from `pod-a` to `10.96.0.10:80` gets its destination rewritten to a real backend pod, conntrack tracks it, and the reply is translated back. To "load balance," you'd add multiple backends (nftables can pick among them, e.g. with a `numgen`/round-robin or random selector) — which is exactly what kube-proxy's per-endpoint rules do probabilistically.

**What kube-proxy automates:** it *watches the API server* for Service and Endpoint changes and continuously **regenerates** these DNAT/load-balancing rules (iptables or IPVS) so the mapping always reflects the current healthy backend pods — adding/removing endpoints as pods come and go, with no manual edits. You hard-coded one backend; kube-proxy maintains the whole dynamic set.

**Why ClusterIP is a legitimate NAT exception:** pod-to-pod must stay NAT-free so pods see real IPs, but a **Service IP is a virtual abstraction that points to no single real interface** — it deliberately fronts a *changing set* of pods. The only way to turn "connect to this stable virtual IP" into "reach one of these real, ephemeral pod IPs" is to **rewrite the destination (DNAT)** and pick a backend. So NAT appears precisely where the model needs to translate a stable abstraction into a concrete, changing reality — at the Service edge — while the underlying pod-to-pod fabric stays flat and untranslated. (The other edge NAT is SNAT for cluster→internet egress.)
</details>
