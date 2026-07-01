---
title: "Lesson 45 — tcpdump Mastery"
nav_order: 45
parent: "Phase 15: Troubleshooting"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 45: tcpdump Mastery

## Concept

You met tcpdump back in Lesson 6 and have used it in every lab since. This lesson levels it up from "show me packets" to **surgical evidence-gathering** — precise filters that capture *only* the packets that matter, payload inspection, capture rotation for long-running problems, and scripted analysis. The guiding principle of all troubleshooting (Lesson 47) is **never guess — prove it with a packet**, and tcpdump is how you produce that proof.

```
   "It's probably the firewall."          ← guessing
   "I captured at the sender: SYN goes out.
    I captured at the receiver: no SYN arrives.
    The packet is lost between them."      ← evidence
```

The skill that separates beginners from experts isn't knowing tcpdump exists — it's writing a filter tight enough that the one packet you need isn't buried under ten thousand you don't, and reading the output fluently enough to draw a conclusion.

---

## How it works

**The two filter languages.** tcpdump uses **BPF capture filters** (the expression *after* the options, e.g. `tcp port 443`) — these decide what gets captured, in the kernel, cheaply. Don't confuse them with display filters (that's Wireshark). Capture filters compose with `and`, `or`, `not` and can reach into header bytes.

**Bit-level flag matching.** TCP flags live in a byte you can test directly:

- `tcp[tcpflags] & tcp-syn != 0` — any packet with the SYN bit set
- `tcp[tcpflags] == tcp-syn` — *only* SYN (SYN set, everything else clear) — i.e. connection openers, not SYN-ACKs
- combine: `tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn` — SYN set **and** ACK clear → pure SYNs

**Reading payloads.** `-X` shows hex **and** ASCII side by side; `-A` shows ASCII only (handy for HTTP). Add `-vv` for verbose protocol decoding, `-e` for link-layer (MAC) headers, `-n` to skip DNS, `-nn` to skip DNS *and* port-name lookup, `-s 0` to capture full packets (modern tcpdump defaults to full length already).

**Long-running captures.** `-w file.pcap` writes raw packets (loss-free, openable in Wireshark). For problems that take hours to reproduce, **rotate**: `-G 60 -W 5 -w cap-%H%M%S.pcap` makes 60-second files, keeping 5 — a bounded ring buffer on disk. Read back with `-r file.pcap` (and you can re-apply a filter while reading).

**Scripted analysis with tshark.** When you need fields, not eyeballs: `tshark -r cap.pcap -T fields -e ip.src -e tcp.dstport -e frame.time` emits columns you can pipe into `sort`/`uniq`/`awk` — turning a capture into data.

{: .note }
> **Capture at both ends to localize a drop**
> The single most powerful tcpdump technique: capture **simultaneously at the sender and the receiver** (e.g. on both veth ends, or on both NICs of a router). If a packet appears at the sender but not the receiver, it died *in between* (a drop, a firewall, a routing black hole). If it arrives but gets no reply, the problem is at/after the receiver (firewall input rule, no listening socket, wrong return route). Comparing the two captures converts "somewhere it breaks" into "it breaks *here*."

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `tcpdump 'tcp[tcpflags] & tcp-syn != 0'` | Match packets with the SYN flag set |
| `tcpdump -X` / `-A` | Hex+ASCII / ASCII payload dump |
| `tcpdump -w cap.pcap` / `-r cap.pcap` | Write / read a capture file |
| `tcpdump -G 60 -W 5 -w 'c-%H%M%S.pcap'` | Rotating capture (60s files, keep 5) |
| `tcpdump -nn -e -vv` | No name resolution, show MACs, verbose decode |
| `tshark -r cap.pcap -T fields -e ip.src ...` | Extract specific fields for scripting |

---

## Lab

### Step 1 — Capture only connection-opening SYNs to port 443

```bash
$ sudo tcpdump -ni any 'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn and dst port 443'
# In another terminal:
$ curl -s https://example.com >/dev/null
# You see exactly ONE line per connection attempt — the pure SYN to :443
... IP 10.0.0.5.54321 > 93.184.216.34.443: Flags [S], seq ...
```

`Flags [S]` = SYN only. A SYN-ACK would be `[S.]`; the filter excluded it because the ACK bit is set.

### Step 2 — Watch a full handshake by flags

```bash
$ sudo tcpdump -ni any 'tcp port 443 and host example.com'
... Flags [S]     # SYN     (client → server)
... Flags [S.]    # SYN-ACK (server → client)
... Flags [.]     # ACK     (client → server) — handshake complete
... Flags [P.]    # PSH+ACK # data
... Flags [F.]    # FIN+ACK # teardown
```

### Step 3 — Inspect payload (plaintext HTTP)

```bash
$ sudo tcpdump -nni any -A 'tcp port 80 and host neverssl.com'
$ curl -s http://neverssl.com >/dev/null
# -A shows the ASCII request/response: "GET / HTTP/1.1", "Host: ...", "HTTP/1.1 200 OK"
```

### Step 4 — Save, then analyze offline

```bash
$ sudo tcpdump -ni any -w /tmp/web.pcap 'tcp port 80 or tcp port 443' &
$ curl -s http://neverssl.com >/dev/null; curl -s https://example.com >/dev/null
$ sudo kill %1
$ tcpdump -nr /tmp/web.pcap 'tcp[tcpflags] == tcp-syn'      # re-filter on read: just SYNs
$ tshark -r /tmp/web.pcap -T fields -e ip.dst -e tcp.dstport | sort | uniq -c
```

### Step 5 — Rotating capture for an intermittent problem

```bash
# 30-second files, keep the last 4 (2 minutes of history), until you Ctrl-C
$ sudo tcpdump -ni any -G 30 -W 4 -w '/tmp/ring-%H%M%S.pcap' 'icmp or arp'
$ ls -lt /tmp/ring-*.pcap        # never more than 4 files
```

### Step 6 — Clean up

```bash
$ rm -f /tmp/web.pcap /tmp/ring-*.pcap
```

---

## Further Reading

| Topic | Link |
|---|---|
| `tcpdump` | [man7.org — tcpdump(1)](https://man7.org/linux/man-pages/man1/tcpdump.1.html) |
| pcap filter syntax | [man7.org — pcap-filter(7)](https://man7.org/linux/man-pages/man7/pcap-filter.7.html) |
| `tshark` | [tshark — Wireshark docs](https://www.wireshark.org/docs/man-pages/tshark.html) |
| TCP flags | [Wikipedia — TCP § Flags](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#TCP_segment_structure) |
| Wireshark | [wireshark.org](https://www.wireshark.org/) |

---

## Checkpoint

**Q1. Write a tcpdump filter that captures only TCP SYN packets destined for port 443.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>

```
tcpdump -ni any 'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn and dst port 443'
```

The key is `tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn`: mask off all but the SYN and ACK bits, and require the result to equal SYN-only — i.e. **SYN set, ACK clear**. That matches *connection-opening* SYNs while excluding SYN-ACKs (which have both bits set). Adding `dst port 443` restricts it to that destination port. (A looser `tcp[tcpflags] & tcp-syn != 0 and dst port 443` would also catch SYN-ACKs going *to* :443, which usually isn't what you want — the masked-equality form is the precise answer.)
</details>

---

**Q2. You suspect packets are being dropped between two hosts. What is the single most effective tcpdump technique to confirm and localize the drop?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Capture simultaneously at both ends** (sender and receiver — e.g. on both NICs of the path, or both veth ends), with the *same* tight filter, and compare. The comparison localizes the failure precisely:

- Packet appears at the **sender but not the receiver** → it was **dropped in transit** (a firewall/route black hole/loss *between* them).
- Packet **arrives at the receiver but gets no reply** → the problem is **at or after the receiver** — an input firewall rule, no socket listening, or a broken **return route**.
- Packet **never leaves the sender** → the problem is local to the sender (routing, ARP, egress firewall).

This turns a vague "it breaks somewhere" into "it breaks *between these two specific points*," which is the whole game in network debugging. If the path has a router in the middle, capturing on both of its interfaces narrows it further (did it arrive on ingress? did it leave on egress?).
</details>

---

**Q3. What is the difference between `tcpdump -w file.pcap` and just redirecting output to a file with `>`? When do you need each?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
`-w file.pcap` writes the **raw captured packets** in pcap binary format — full headers and payloads, losslessly — which you can later re-read with `-r`, re-filter, or open in **Wireshark** for deep analysis (stream following, protocol dissection, graphs). Redirecting with `>` only saves tcpdump's **already-decoded text output** — it's just the human-readable summary lines, with no way to recover the underlying packets, re-filter, or load into Wireshark.

Use **`-w`** whenever you might want to analyze later, share the capture, or do anything beyond glancing at it — which is most real investigations, and essential for intermittent problems (combine with `-G`/`-W` rotation) and for handing evidence to someone else. Use plain text output (or `>`) only for quick, throwaway live watching where you just need to eyeball what's happening right now.
</details>

---

## Homework

Reproduce a **broken NAT** scenario (reuse the Lesson 22 router topology, then deliberately remove or misconfigure the masquerade rule) and debug it **using only tcpdump** — no peeking at `nft list ruleset`. Capture on the LAN side and the WAN side of the router simultaneously, and from the packets alone, prove (a) the outbound packet leaves with the wrong (un-NAT'd, private) source IP, and (b) therefore no reply comes back. Write down the exact filters you used and the one packet that nailed the diagnosis.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Setup:** router with LAN iface (e.g. 192.168.100.1) and WAN iface (e.g. 10.0.0.1), a LAN client `192.168.100.10`, masquerade rule deleted.

**Captures (two terminals):**

```
# LAN side of the router:
tcpdump -ni <lan-if> 'icmp and host 192.168.100.10'
# WAN side of the router:
tcpdump -ni <wan-if> icmp
```

Then `ping 10.0.0.2` (the "internet" host) from the client.

**What the packets prove:**

- On the **LAN side** you see the echo request `192.168.100.10 → 10.0.0.2` — correct, the client sent it.
- On the **WAN side** you see the request *still* sourced from `192.168.100.10 → 10.0.0.2` — **this is the smoking gun.** With masquerade working, the WAN-side source would have been rewritten to the router's `10.0.0.1`. Seeing the **private `192.168.100.10` source egressing the WAN interface** proves SNAT didn't happen.
- **(b)** The reply never appears on either capture: the remote host (or any upstream router) has no route back to the private `192.168.100.x` source, so the echo reply is unroutable/dropped — hence the ping times out. The absence of a return packet, combined with the un-rewritten egress source, *is* the proof.

**The one packet that nailed it:** the WAN-side **echo request carrying source `192.168.100.10`**. A correctly NAT'd setup can never show a private source leaving the public interface; seeing it there localizes the fault to "SNAT/masquerade is not being applied on egress" — diagnosed entirely from packets, before ever looking at the ruleset.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 46 — Full Diagnostic Toolchain →](lesson-46-diagnostic-toolchain){: .btn .btn-primary }
