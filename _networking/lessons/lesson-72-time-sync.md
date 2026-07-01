---
title: "Lesson 72 — Time Synchronization (NTP & PTP)"
nav_order: 72
parent: "Phase 20: Modern Transport & Observability"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 72: Time Synchronization — NTP & PTP

## Concept

Every computer's clock drifts — crystal oscillators run slightly fast or slow, so clocks diverge by
seconds per day if left alone. Synchronized time is quietly load-bearing for networks: TLS certificate
validity, log correlation across machines, Kerberos tickets, distributed databases, and financial
timestamps all break or misbehave when clocks disagree. **NTP** keeps clocks within milliseconds;
**PTP** pushes to sub-microsecond.

```
   drift: every clock wrong by a bit → events out of order across machines

   NTP:  "good enough" ms-level sync over the internet (hierarchy of time sources)
   PTP:  ns/µs-level sync on a LAN with hardware timestamping
```

---

## How it works

**NTP — a hierarchy of sources.** [NTP](https://en.wikipedia.org/wiki/Network_Time_Protocol) organizes
time sources into **strata**: stratum 0 is a reference clock (GPS, atomic), stratum 1 servers attach
directly to one, stratum 2 sync from stratum 1, and so on. A client exchanges timestamped packets with
servers and computes the **offset** (how wrong its clock is) and **round-trip delay**, then *slews*
(gently speeds/slows) or steps the clock to converge. Because it accounts for network delay
symmetrically, it reaches ~milliseconds even over the internet. `chrony` is the modern Linux
implementation (better on intermittent/virtual machines than the classic `ntpd`).

**Why milliseconds isn't always enough.** Some applications need *much* tighter sync: telecom, power
grids, high-frequency trading, distributed databases that order events by timestamp. NTP's accuracy is
limited by software timestamping — the time spent in the OS/network stack between "now" and when the
packet actually hits the wire is variable and adds jitter.

**PTP — hardware timestamps.** [PTP](https://en.wikipedia.org/wiki/Precision_Time_Protocol) (IEEE 1588)
reaches sub-microsecond by **timestamping packets in hardware** — the NIC records the exact moment a
sync packet crosses the wire, removing the OS/stack jitter. With PTP-aware switches (boundary/transparent
clocks) correcting for their own queuing delay along the path, a LAN can sync to nanoseconds. It needs
hardware support (PTP-capable NICs/switches), so it's a LAN/data-center technology, not an internet one.

{: .note }
> **The TLS/security tie-in**
> A wrong clock breaks security directly: certificates have <code>notBefore</code>/<code>notAfter</code>
> dates, so a badly-skewed clock will reject valid certs or accept expired ones; Kerberos rejects tickets
> outside a few minutes' skew; TOTP codes (Security track) are time-based. "Random TLS failures" on a
> fleet are often just clock drift — which is why time sync is a prerequisite, not an afterthought.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `chronyc tracking` | Current offset, drift, and reference source |
| `chronyc sources -v` | The time sources and their stratum/reachability |
| `timedatectl` | System clock, timezone, NTP sync status |
| `ethtool -T <if>` | Whether a NIC supports hardware (PTP) timestamping |
| `ptp4l -i <if>` / `phc2sys` | Run PTP / sync the system clock to the NIC's PTP clock |

---

## Lab

Inspect NTP sync with chrony and check whether your NIC could do PTP.

### Step 1 — See the system's sync status

```bash
$ timedatectl
               ...
   System clock synchronized: yes
                 NTP service: active
```

### Step 2 — Look at chrony's view of offset and sources

```bash
$ chronyc tracking
Reference ID    : ...
Stratum         : 3
System time     : 0.000023 seconds slow of NTP time   # current offset (sub-ms)
Frequency       : 12.5 ppm slow                         # measured local clock drift
$ chronyc sources -v          # the upstream servers, their stratum and reachability
```

### Step 3 — Check PTP hardware capability

```bash
$ ethtool -T eth0
Time stamping parameters for eth0:
Capabilities:
        hardware-transmit
        hardware-receive
        hardware-raw-clock        # ← present = NIC can do PTP hardware timestamping
```

### Step 4 — (If supported) glimpse PTP

```bash
$ sudo ptp4l -i eth0 -m        # joins/forms a PTP domain; prints offset converging to sub-µs
```

---

## Further Reading

| Topic | Link |
|---|---|
| NTP | [Wikipedia — Network Time Protocol](https://en.wikipedia.org/wiki/Network_Time_Protocol) |
| PTP / IEEE 1588 | [Wikipedia — Precision Time Protocol](https://en.wikipedia.org/wiki/Precision_Time_Protocol) |
| chrony | [chrony.tuxfamily.org](https://chrony.tuxfamily.org/) |
| Clock drift | [Wikipedia — Clock drift](https://en.wikipedia.org/wiki/Clock_drift) |

---

## Checkpoint

**Q1. Why does synchronized time matter for a network, beyond "nice to have"? Give concrete failures.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Many systems treat time as ground truth, so clock disagreement causes real breakage: (1) <strong>TLS
certificates</strong> have validity windows (<code>notBefore</code>/<code>notAfter</code>), so a skewed
clock rejects valid certs or accepts expired ones — "random" TLS failures. (2) <strong>Log
correlation</strong> across machines becomes impossible if timestamps don't line up, crippling incident
investigation. (3) <strong>Kerberos</strong> rejects tickets whose timestamps differ by more than a few
minutes, so skew breaks authentication. (4) <strong>Distributed databases / event ordering</strong> rely
on timestamps to order or reconcile events; drift causes inconsistency. (5) <strong>TOTP</strong>
one-time codes are time-derived and fail if the clock is off. So time sync is a prerequisite for
security, observability, and correctness — not a cosmetic detail.
</details>

---

**Q2. How does NTP achieve millisecond accuracy even over a variable-delay internet?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
NTP exchanges <strong>timestamped packets</strong> with its servers, recording four times (when the
client sent, when the server received, when the server replied, when the client received). From these it
computes both the <strong>clock offset</strong> (how far off the local clock is) and the
<strong>round-trip delay</strong>, and — assuming the path delay is roughly symmetric — it can subtract
out the network transit time to estimate the true offset despite variable latency. It also samples many
servers and many exchanges, filtering outliers and jitter, and organizes sources into a
<strong>stratum hierarchy</strong> traceable back to a reference clock (GPS/atomic) so it trusts the most
accurate sources. Finally it <strong>slews</strong> the clock (gradually adjusts its rate) rather than
jumping, converging smoothly to within milliseconds.
</details>

---

**Q3. When is NTP insufficient and PTP required, and what makes PTP so much more precise?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
NTP's accuracy (~milliseconds) is limited by <strong>software timestamping</strong>: the variable,
unpredictable time a packet spends in the OS and network stack before it actually hits the wire adds
jitter you can't fully remove. That's insufficient for applications needing micro/nanosecond precision —
telecom, power grids, high-frequency trading, and tightly-ordered distributed systems. <strong>PTP</strong>
(IEEE 1588) is required there, and it's far more precise because it <strong>timestamps packets in
hardware</strong>: the NIC records the exact instant a sync packet crosses the wire, eliminating the
OS/stack jitter. Combined with PTP-aware switches (boundary/transparent clocks) that account for their
own queuing delay along the path, this brings sync down to sub-microsecond/nanosecond — at the cost of
needing PTP-capable NICs and switches, which is why PTP is a LAN/data-center technology rather than an
internet one.
</details>

---

## Homework

Use `chronyc tracking` to record your machine's measured drift (ppm) and current offset over a few
hours (or compare two machines' offsets). Then deliberately reason about a scenario: a server's clock is
5 minutes fast. Walk through what happens when (a) it validates a freshly-issued TLS certificate, and
(b) a user tries to authenticate with Kerberos. Tie both back to why the clock is the dependency.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The drift figure (ppm) shows how fast the local oscillator diverges — e.g. 12 ppm ≈ ~1 second every
~23 hours — which is exactly why continuous correction is needed; the offset shows how far off "now" is
at each check. For the 5-minutes-fast server: (a) Validating a <strong>freshly-issued TLS
certificate</strong> can <em>fail</em>: certificates have a <code>notBefore</code> time, and to a clock
running 5 minutes ahead a cert issued "now" may appear to start <em>in the future</em> (notBefore >
perceived now), so the server rejects it as not-yet-valid — a confusing failure for a perfectly good
cert. (b) <strong>Kerberos</strong> authentication will be <em>rejected</em>: Kerberos enforces a tight
clock-skew tolerance (commonly 5 minutes) between client and KDC to prevent replay; a 5-minute-fast
clock pushes ticket timestamps outside the allowed window, so the KDC/service refuses the tickets and
login fails. Both trace back to the same root cause — these protocols encode <em>absolute time</em> into
their security decisions (validity windows, replay windows), so the clock is a hard dependency, and
keeping it synced (NTP/chrony) is what prevents these failures.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 73 — Network Observability & Telemetry →](lesson-73-observability){: .btn .btn-primary }
