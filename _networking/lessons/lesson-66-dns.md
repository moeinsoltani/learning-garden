---
title: "Lesson 66 — DNS Deep Dive"
nav_order: 66
parent: "Phase 19: Network Services"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 66: DNS Deep Dive

## Concept

DNS turns names into addresses, and almost every connection starts with it. It's a **distributed,
hierarchical, cached** database. "Hierarchical" because the name `www.example.com.` is resolved
right-to-left: root → `.com` → `example.com` → `www`. "Cached" because every layer remembers answers
for a TTL, so the full walk is rare.

```
   www.example.com.   →   .          (root: "ask the .com servers")
                          com.        (TLD: "ask example.com's servers")
                          example.com.(authoritative: "www is 93.184.x.x")
```

---

## How it works

**Recursive vs authoritative.** Your client asks a **recursive resolver** ("just get me the answer").
The resolver does the legwork, querying **authoritative** servers — the ones that actually *hold* a
zone's records — starting at the root, following referrals down the hierarchy, caching each step. A
recursive resolver answers from cache or by walking; an authoritative server answers only for its own
zones.

**Record types you'll meet:** `A`/`AAAA` (name→IPv4/IPv6), `CNAME` (alias), `MX` (mail), `NS`
(delegation), `TXT` (arbitrary, e.g. SPF/verification), `PTR` (reverse: IP→name), `SOA` (zone
metadata/TTLs).

**The Linux client path.** Name resolution goes through the C library's NSS (`/etc/nsswitch.conf`),
typically consulting `/etc/hosts` then DNS via `/etc/resolv.conf`. On modern distros
**`systemd-resolved`** sits in the middle: a local stub on `127.0.0.53` that caches, can do
**split DNS** (different upstream servers per domain — essential for VPNs/overlays, cf. Lesson 55's
mesh naming), and supports encrypted transports.

**Encrypted DNS.** Plain DNS is UDP/53, unencrypted — anyone on-path sees your lookups.
**DoT** (DNS over TLS, port 853) and **DoH** (DNS over HTTPS, 443) encrypt the query so it's private
and tamper-resistant (DoH additionally hides among normal web traffic).

{: .note }
> **TTL and caching: the double-edged sword**
> Every record has a TTL telling resolvers how long to cache it. High TTLs reduce load and latency but
> make changes slow to propagate; low TTLs make changes fast but increase query volume. This is why
> you *lower* a record's TTL well *before* a planned IP migration, then raise it again after.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `dig <name>` / `dig +trace <name>` | Query DNS / show the full recursive walk from root |
| `dig <name> @<server>` | Query a specific server (e.g. an authoritative one) |
| `dig -x <ip>` | Reverse lookup (PTR) |
| `resolvectl status` / `resolvectl query <name>` | systemd-resolved config & queries |
| `resolvectl domain <if> ~example.com` | Configure split DNS for a domain |

---

## Lab

Watch a full recursive resolution, then run a tiny authoritative zone with dnsmasq (Lesson 24).

### Step 1 — See the hierarchy with +trace

```bash
$ dig +trace www.iana.org
# Shows queries to the root servers, then .org TLD servers, then iana.org's
# authoritative servers — each referral (NS) leading one level deeper.
```

### Step 2 — Inspect record types

```bash
$ dig example.com A +short
$ dig example.com MX +short
$ dig -x 1.1.1.1 +short        # reverse: who is 1.1.1.1
```

### Step 3 — Run a small authoritative resolver in a namespace

```bash
$ sudo ip netns add dns
# give it an address, then:
$ sudo ip netns exec dns dnsmasq --no-daemon --address=/test.lab/10.9.9.9 --listen-address=<ip> &
$ dig @<ip> anything.test.lab +short
10.9.9.9         # dnsmasq answered authoritatively for the test.lab domain
```

### Step 4 — Check the local resolver and split DNS

```bash
$ resolvectl status        # shows the stub (127.0.0.53), per-link DNS servers, and any split domains
```

### Step 5 — Clean up

```bash
$ sudo ip netns delete dns
```

---

## Further Reading

| Topic | Link |
|---|---|
| DNS | [Wikipedia — Domain Name System](https://en.wikipedia.org/wiki/Domain_Name_System) |
| DNS over HTTPS / TLS | [Wikipedia — DNS over HTTPS](https://en.wikipedia.org/wiki/DNS_over_HTTPS) |
| systemd-resolved | [man7.org — systemd-resolved(8)](https://man7.org/linux/man-pages/man8/systemd-resolved.service.8.html) |
| `dig` | [man7.org — dig(1)](https://man7.org/linux/man-pages/man1/dig.1.html) |

---

## Checkpoint

**Q1. What is the difference between a recursive resolver and an authoritative server?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A <strong>recursive resolver</strong> is the server your client asks to "get me the answer by whatever
means necessary." It does the full lookup on your behalf — querying the root, then the TLD, then the
domain's authoritative servers, following referrals and caching each step — and returns the final
answer. An <strong>authoritative server</strong> holds the actual records for specific zones and answers
<em>only</em> for those zones (e.g. example.com's authoritative servers know example.com's records). The
resolver is the legwork/caching layer; the authoritative servers are the source of truth. Your client
talks to a resolver; the resolver talks to authoritative servers.
</details>

---

**Q2. Trace, at a high level, how `www.example.com` is resolved from a cold cache.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
With nothing cached: (1) the stub resolver on the client sends the query to its recursive resolver.
(2) The resolver asks a <strong>root</strong> server, which doesn't know the answer but refers it to the
<strong>.com</strong> TLD servers (an NS referral). (3) The resolver asks a .com server, which refers it
to <strong>example.com's authoritative</strong> servers. (4) The resolver asks an example.com
authoritative server, which returns the <code>A</code>/<code>AAAA</code> record for
<code>www.example.com</code>. (5) The resolver caches each response for its TTL and returns the final
answer to the client. Subsequent lookups are served from cache until the TTLs expire, so the full
root→TLD→authoritative walk only happens on cache misses.
</details>

---

**Q3. What problem does split DNS solve, and where is it especially important?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Split DNS routes queries for <em>different domains to different upstream resolvers</em>, rather than
sending everything to one server. It solves the problem of resolving <strong>private/internal names
that public resolvers don't know</strong> while still using public DNS for everything else. It's
especially important for <strong>VPNs and overlay networks</strong> (e.g. the mesh naming from Lesson
55) and corporate networks: queries for the internal domain (say <code>~corp.example</code> or the mesh
domain) go to the internal/overlay resolver, while all other queries go to the normal upstream. Without
split DNS you'd either leak internal names to public servers (which can't answer them) or be forced to
send <em>all</em> DNS through the internal resolver. systemd-resolved implements this with per-link
routing domains.
</details>

---

## Homework

Capture DNS traffic with `tcpdump -ni any port 53` while doing a `dig` for a name that isn't cached,
then repeat the same `dig` immediately and capture again. Explain the difference between the two
captures, and then explain what would change if the resolver were configured for DoT instead of plain
UDP/53.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The <strong>first</strong> capture (cold cache) shows the resolver doing work: the query plus, if you're
watching a recursive resolver, the chain of outbound queries to root/TLD/authoritative servers before
the answer comes back. The <strong>second</strong> capture (warm cache) shows essentially just the
client's query and an immediate answer from the resolver, with <em>no</em> upstream queries — because
the record is now cached and still within its TTL, so the resolver answers locally. That contrast is
caching in action: latency drops and upstream traffic disappears on the repeat. If the resolver used
<strong>DoT</strong> instead of plain UDP/53, the client↔resolver queries would no longer be visible as
readable DNS on port 53 — they'd be a <strong>TLS session on port 853</strong>, so tcpdump would show
only encrypted bytes, not the names being looked up. The lookups become private and tamper-resistant on
that hop (an on-path observer can't see or modify them), though the resolver's own upstream behavior and
caching are unchanged. (DoH would similarly hide them, as TLS on 443 indistinguishable from web traffic.)
</details>
