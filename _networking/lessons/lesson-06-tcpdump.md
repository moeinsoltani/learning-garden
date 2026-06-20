---
title: "Lesson 06 — tcpdump, Your Constant Companion"
nav_order: 6
parent: "Phase 1: Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 06: tcpdump — Your Constant Companion

## Concept

Everything you will do from this point on involves packets you cannot see with your eyes. `tcpdump` is how you see them. It is a **packet sniffer**: it asks the kernel to copy every frame that passes a chosen interface and prints a human-readable summary of each one.

Treat this lesson as a tool you will reach for in *every* lab that follows. When something doesn't work, the answer is almost never "guess." The answer is "capture the traffic and look at where the packet stops appearing."

```
        application
            │
   ┌────────┴────────┐
   │   kernel network │  ← tcpdump taps in here, via a raw AF_PACKET socket
   │      stack       │     it sees a COPY of every frame on the interface
   └────────┬────────┘
            │
        NIC / veth
```

{: .note }
> **tcpdump sees copies, it does not interfere**
> Capturing traffic does not drop, delay, or modify packets. tcpdump attaches a raw [`AF_PACKET`](https://man7.org/linux/man-pages/man7/packet.7.html) socket and receives a *copy* of each frame. The original continues through the stack untouched. This is why it is safe to run in production.

---

## How it works

tcpdump puts the interface into a mode where the kernel hands it every frame (optionally **promiscuous mode**, where it even receives frames not addressed to this host). It then applies a **BPF filter** — a tiny program compiled from your filter expression and run in the kernel — so only matching packets are copied to userspace. Filtering in the kernel is what makes tcpdump fast even on busy links.

The output format for one packet is roughly:

```
22:14:03.123456 IP 10.0.0.1.54321 > 10.0.0.2.80: Flags [S], seq 12345, win 64240, length 0
└─ timestamp     └proto └─ source     └─ dest       └─ TCP flags  └ details
                        addr.port      addr.port
```

---

## New Flags This Lesson

| Flag | What it does |
|---|---|
| `-i <iface>` | Capture on this interface. `-i any` captures on all interfaces. |
| `-n` | Do **not** resolve IPs to hostnames (and `-nn` also skips port-name lookup). Faster, and avoids surprise DNS traffic. |
| `-e` | Print the **Ethernet header** — shows source/dest MAC addresses. |
| `-v`, `-vv`, `-vvv` | Increasing verbosity — TTL, IP id, options, checksums. |
| `-c <N>` | Stop after capturing N packets. |
| `-w file.pcap` | Write raw packets to a file (for Wireshark or later analysis). |
| `-r file.pcap` | Read and display packets from a saved file. |
| `-X` | Print packet payload as hex **and** ASCII. |
| `-t`, `-tttt` | Control timestamp format (`-tttt` = readable date+time). |
| `-s <N>` | Snap length — bytes captured per packet (`-s 0` = whole packet). |

## Filter Primitives

Filters are combined with `and`, `or`, `not`:

| Filter | Matches |
|---|---|
| `host 10.0.0.2` | Traffic to or from that IP |
| `src host 10.0.0.1` / `dst host 10.0.0.2` | Only source / only destination |
| `net 10.0.0.0/24` | Any address in that subnet |
| `port 80` | TCP or UDP port 80, either direction |
| `src port 443` / `dst port 53` | Directional port match |
| `tcp` / `udp` / `icmp` / `arp` | By protocol |
| `tcp port 22 and host 10.0.0.5` | Combined |
| `not port 22` | Everything except SSH (handy when you're SSH'd in) |

---

## Lab

### Step 1 — Build a two-namespace topology

Same veth setup as Lesson 5. Packets entering one veth end exit the other.

```bash
$ sudo ip netns add ns1
$ sudo ip netns add ns2
$ sudo ip link add veth-ns1 type veth peer name veth-ns2
$ sudo ip link set veth-ns1 netns ns1
$ sudo ip link set veth-ns2 netns ns2
$ sudo ip netns exec ns1 ip link set veth-ns1 up
$ sudo ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth-ns1
$ sudo ip netns exec ns2 ip link set veth-ns2 up
$ sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth-ns2
```

### Step 2 — Capture a full ICMP echo exchange

Open **two terminals**. In terminal A, start the capture inside ns1:

```bash
# Terminal A
$ sudo ip netns exec ns1 tcpdump -n -i veth-ns1 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on veth-ns1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

In terminal B, send two pings from ns1:

```bash
# Terminal B
$ sudo ip netns exec ns1 ping -c 2 10.0.0.2
```

Terminal A now shows the request/reply pairs:

```
22:14:03.100 IP 10.0.0.1 > 10.0.0.2: ICMP echo request, id 4242, seq 1, length 64
22:14:03.100 IP 10.0.0.2 > 10.0.0.1: ICMP echo reply,   id 4242, seq 1, length 64
22:14:04.101 IP 10.0.0.1 > 10.0.0.2: ICMP echo request, id 4242, seq 2, length 64
22:14:04.101 IP 10.0.0.2 > 10.0.0.1: ICMP echo reply,   id 4242, seq 2, length 64
```

Each `echo request` is answered by an `echo reply` with the same `id` and `seq`. That pairing is how `ping` matches replies to requests.

### Step 3 — Add `-e` to see MAC addresses

```bash
$ sudo ip netns exec ns1 tcpdump -e -n -i veth-ns1 icmp
22:15:01.200 <ns1-mac> > <ns2-mac>, ethertype IPv4 (0x0800), length 98: 10.0.0.1 > 10.0.0.2: ICMP echo request ...
22:15:01.200 <ns2-mac> > <ns1-mac>, ethertype IPv4 (0x0800), length 98: 10.0.0.2 > 10.0.0.1: ICMP echo reply ...
```

`-e` reveals the L2 frame: source MAC, dest MAC, and `ethertype 0x0800` (the EtherType value that means "IPv4 payload inside").

### Step 4 — Capture a TCP handshake

Start a listener in ns2 and a capture in ns1, then connect.

```bash
# Terminal A — listen on port 8080 in ns2
$ sudo ip netns exec ns2 nc -l -p 8080

# Terminal B — capture TCP in ns1
$ sudo ip netns exec ns1 tcpdump -n -i veth-ns1 tcp port 8080

# Terminal C — connect from ns1
$ sudo ip netns exec ns1 nc 10.0.0.2 8080
```

The capture shows the three-way handshake:

```
IP 10.0.0.1.50000 > 10.0.0.2.8080: Flags [S],   seq 1000,           win 64240, length 0
IP 10.0.0.2.8080 > 10.0.0.1.50000: Flags [S.],  seq 5000, ack 1001, win 65160, length 0
IP 10.0.0.1.50000 > 10.0.0.2.8080: Flags [.],   ack 5001,           win 64240, length 0
```

Reading the `Flags` field:

| Flags | Name | Meaning |
|---|---|---|
| `[S]` | SYN | "Let's open a connection. My starting sequence number is X." |
| `[S.]` | SYN-ACK | The `.` is the ACK bit. "I accept. My start is Y, and I acknowledge your X+1." |
| `[.]` | ACK | "Acknowledged. Connection established." |
| `[P.]` | PSH-ACK | Carries data; push to the application now. |
| `[F.]` | FIN-ACK | One side is closing the connection. |
| `[R]` | RST | Reset — connection refused or aborted. |

### Step 5 — Save to a file and replay it

```bash
$ sudo ip netns exec ns1 tcpdump -n -i veth-ns1 -w /tmp/cap.pcap -c 10
# ... generate some traffic (ping, nc) in another terminal ...

# Read it back later, no live capture needed:
$ tcpdump -n -r /tmp/cap.pcap
# Or open /tmp/cap.pcap in Wireshark for a graphical view
```

`-w` writes the raw bytes (not the text summary), so you can re-analyze the *same* packets with different verbosity or in Wireshark.

### Step 6 — Capture inside a namespace with `-i any`

When you don't know which interface the traffic uses, capture all of them:

```bash
$ sudo ip netns exec ns1 tcpdump -n -i any icmp
```

`-i any` captures across every interface in that namespace. Note it captures only interfaces *in this namespace* — a capture in ns1 never sees ns2's internal traffic unless it crosses the veth.

### Step 7 — Clean up

```bash
$ sudo ip netns delete ns1
$ sudo ip netns delete ns2
```

---

{: .note }
> **Watch out for `-n`**
> Without `-n`, tcpdump does a reverse-DNS lookup on every IP it prints. On a busy capture that generates its own DNS traffic — which then shows up in your capture — a confusing feedback loop. Almost always run with `-n` (or `-nn` to also skip port-name resolution).

{: .note }
> **Snap length and truncation**
> By default modern tcpdump captures the whole packet (`snapshot length 262144`). On older versions or with `-s 96` you may only capture headers — the payload is truncated. If you need to inspect payload with `-X`, make sure the snap length covers it (`-s 0` = unlimited).

---

## Further Reading

| Topic | Link |
|---|---|
| `tcpdump` man page | [man7.org — tcpdump(8)](https://man7.org/linux/man-pages/man8/tcpdump.8.html) |
| `pcap-filter` — filter syntax | [man7.org — pcap-filter(7)](https://man7.org/linux/man-pages/man7/pcap-filter.7.html) |
| Berkeley Packet Filter | [Wikipedia — BPF](https://en.wikipedia.org/wiki/Berkeley_Packet_Filter) |
| TCP three-way handshake | [Wikipedia — TCP connection establishment](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_establishment) |
| `AF_PACKET` raw sockets | [man7.org — packet(7)](https://man7.org/linux/man-pages/man7/packet.7.html) |
| Wireshark | [Wikipedia — Wireshark](https://en.wikipedia.org/wiki/Wireshark) |

---

## Checkpoint

**Q1. You run `tcpdump -i eth0 host 10.0.0.5` and see ICMP requests leave but no replies. A colleague says "the firewall must be dropping the replies." What can you NOT yet conclude, and what additional capture would settle it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You cannot conclude the firewall is dropping replies. All you know is that *no reply arrived back on eth0*. The reply might never have been sent (the remote host is down, or its route back is wrong), or it was dropped anywhere along the return path. To localize the drop, capture on the **other host** too: if 10.0.0.5 shows the request arriving and a reply leaving, the loss is on the return path between them. If 10.0.0.5 never sees the request, the loss is on the forward path. Comparing sender-side and receiver-side captures is the core technique for localizing where a packet disappears.
</details>

---

**Q2. In a TCP handshake capture, identify which packet is the SYN, which is the SYN-ACK, and which is the ACK — by their `Flags` field.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
- **SYN** = `Flags [S]` — sent by the client to initiate. Carries an initial sequence number, no ACK yet.
- **SYN-ACK** = `Flags [S.]` — sent by the server. The `S` means it also has its own SYN (its initial sequence number); the `.` means the ACK bit is set, acknowledging the client's SYN.
- **ACK** = `Flags [.]` — sent by the client. Only the ACK bit is set, confirming the server's sequence number. After this packet the connection is ESTABLISHED.

In tcpdump's flag shorthand, `.` always represents the ACK bit, and the letters `S F R P` represent SYN, FIN, RST, PSH respectively.
</details>

---

**Q3. Why is it usually a mistake to run a capture without `-n` when you're debugging a DNS problem?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Without `-n`, tcpdump performs a reverse-DNS lookup for every IP address it displays. Those lookups generate their own DNS queries — which then appear in your capture, polluting it with traffic tcpdump itself created. When the thing you're debugging *is* DNS, this feedback loop makes the output nearly unreadable and can even mask the real problem. Always use `-n` (or `-nn`) so tcpdump prints raw addresses and ports and generates no extra traffic.
</details>

---

## Homework

Capture a full TCP connection lifecycle — handshake, data transfer, and teardown — between two namespaces, and save it to a pcap file. Then read it back and annotate, packet by packet, what each one is (SYN, SYN-ACK, ACK, data, FIN, etc.).

Hint: use `nc -l -p 8080` as a server, type a line of text into the client, then close the client with Ctrl-D to trigger a clean FIN teardown.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A clean lifecycle capture (`tcpdump -n -r cap.pcap`) looks like:

```
1  10.0.0.1.50000 > 10.0.0.2.8080: Flags [S],  seq 1000              ← SYN (client opens)
2  10.0.0.2.8080 > 10.0.0.1.50000: Flags [S.], seq 5000, ack 1001    ← SYN-ACK (server accepts)
3  10.0.0.1.50000 > 10.0.0.2.8080: Flags [.],  ack 5001              ← ACK (handshake done)
4  10.0.0.1.50000 > 10.0.0.2.8080: Flags [P.], seq 1001, length 6    ← data ("hello\n")
5  10.0.0.2.8080 > 10.0.0.1.50000: Flags [.],  ack 1007             ← server ACKs the data
6  10.0.0.1.50000 > 10.0.0.2.8080: Flags [F.], seq 1007             ← client sends FIN (Ctrl-D)
7  10.0.0.2.8080 > 10.0.0.1.50000: Flags [.],  ack 1008             ← server ACKs the FIN
8  10.0.0.2.8080 > 10.0.0.1.50000: Flags [F.], seq 5001             ← server sends its own FIN
9  10.0.0.1.50000 > 10.0.0.2.8080: Flags [.],  ack 5002             ← client ACKs — connection closed
```

Packets 1–3 are the three-way handshake. Packet 4 carries the data (`[P.]` = PSH+ACK). Packets 6–9 are the four-way teardown: each side sends a FIN and the other ACKs it. The connection closes independently in each direction, which is why teardown takes four packets but setup only three (the server's SYN and ACK are combined into one SYN-ACK).
</details>
