---
title: "Lesson 70 — TLS at the Packet Level"
nav_order: 70
parent: "Phase 20: Modern Transport & Observability"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 70: TLS at the Packet Level

{: .note }
> **Scope of this lesson**
> This lesson stays at the **wire / observability** level — what a TLS connection looks like in a
> packet capture and what an on-path observer can and can't see. The TLS *protocol internals* — the
> full handshake mechanics, cipher negotiation, key exchange, and certificate validation — are covered
> in depth in the **Security & Identity** track, Phase 3 (TLS & SSL). Here we care about *reading the
> wire*, the networking engineer's angle.

## Concept

[TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security) encrypts traffic, but a connection still
*starts in the clear*: the handshake exchanges some unencrypted fields before keys are established.
Knowing exactly what's visible — and what isn't — is essential for troubleshooting and for
understanding what a passive observer (or a firewall, or a censor) can learn.

```
   TCP handshake          TLS handshake                 encrypted application data
   SYN/SYN-ACK/ACK   →    ClientHello (incl. SNI)   →   ░░░░ everything from here is opaque ░░░░
                          ServerHello, Certificate*
   (* visible in TLS 1.2; encrypted in TLS 1.3)
```

---

## How it works

**The visible bootstrap.** Before encryption is active, the **ClientHello** goes out in plaintext. It
contains the **SNI** (Server Name Indication) — the hostname the client wants — plus offered cipher
suites, TLS versions, and ALPN (which app protocol, e.g. `h2`). The server's **ServerHello** picks
parameters. In **TLS 1.2** the server's **certificate is sent in the clear**, so a sniffer sees the
cert (and thus the identity). In **TLS 1.3**, the certificate and the rest of the handshake are
**encrypted** after the first exchange, so much less leaks.

**What a passive observer still learns** even with encryption: the **SNI** (so which site you're
visiting — unless ECH is used), the server **IP** and port, traffic **volume and timing**, and the
TLS version/ALPN. What they *don't* learn: the actual request/response contents, URLs/paths, cookies,
or headers.

**SNI, ECH.** SNI exists so one IP can host many TLS sites (the server needs to know which cert to
present). The privacy cost is that SNI reveals the hostname in plaintext — which firewalls and censors
exploit to block by name. **ECH** (Encrypted Client Hello) encrypts the SNI to close that leak.

{: .note }
> **Why this matters for networking**
> Filtering, policy, and observability often hinge on the SNI, because it's the one human-meaningful
> field visible without breaking encryption. "Block social media at work," "log which sites are
> visited," and SNI-based censorship all read this field. As TLS 1.3 + ECH spread, that visibility
> shrinks — pushing operators toward DNS-based or endpoint-based controls instead. (Cross-reference:
> QUIC, Lesson 69, hides even more.)

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `openssl s_client -connect <host>:443 -servername <host>` | Do a TLS handshake, show certs/params |
| `tshark -i <if> -Y 'tls.handshake.type==1' -T fields -e tls.handshake.extensions_server_name` | Extract SNI from ClientHellos |
| `tcpdump -ni <if> 'tcp port 443'` | Capture TLS traffic |
| `curl -v https://<host>` | See the negotiated TLS version and cipher |

---

## Lab

Capture a TLS handshake and read the cleartext fields (SNI), then see how TLS 1.3 hides the certificate.

### Step 1 — Extract SNI from a live handshake

```bash
$ sudo tshark -i any -f 'tcp port 443' -Y 'tls.handshake.type==1' \
    -T fields -e tls.handshake.extensions_server_name &
$ curl -sI https://www.wikipedia.org >/dev/null
www.wikipedia.org          # the SNI is plainly visible in the ClientHello
```

### Step 2 — See negotiated version and whether the cert is in the clear

```bash
$ openssl s_client -connect www.wikipedia.org:443 -servername www.wikipedia.org </dev/null 2>/dev/null \
    | grep -E 'Protocol|subject='
# Protocol: TLSv1.3  → the certificate exchange was encrypted on the wire
# (with a TLSv1.2 server you'd be able to capture the cert bytes directly in tcpdump)
```

### Step 3 — Confirm what's opaque

```bash
$ sudo tcpdump -ni any 'tcp port 443' -c 20
# After the handshake, only "Application Data" records appear — lengths/timing visible, content not.
```

---

## Further Reading

| Topic | Link |
|---|---|
| TLS | [Wikipedia — Transport Layer Security](https://en.wikipedia.org/wiki/Transport_Layer_Security) |
| Server Name Indication | [Wikipedia — Server Name Indication](https://en.wikipedia.org/wiki/Server_Name_Indication) |
| Encrypted Client Hello | [Wikipedia — Encrypted Client Hello](https://en.wikipedia.org/wiki/Server_Name_Indication#Encrypted_Client_Hello) |
| TLS protocol internals | Security & Identity track, Phase 3 |

---

## Checkpoint

**Q1. A connection is TLS-encrypted. What can a passive on-path observer still learn about it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Quite a lot of metadata, even though the content is encrypted: the <strong>server IP and port</strong>;
the <strong>SNI</strong> (the hostname being requested), which is sent in cleartext in the ClientHello
unless ECH is in use; the <strong>TLS version and offered/negotiated cipher suites</strong> and
<strong>ALPN</strong> (which application protocol, e.g. HTTP/2); and the <strong>traffic volume and
timing</strong> (sizes and pacing of records, which can fingerprint activity). In TLS 1.2 the observer
can also capture the server's <strong>certificate</strong> in the clear. What they cannot learn: the
actual request/response bodies, URLs/paths, headers, or cookies — those are encrypted.
</details>

---

**Q2. What does SNI do, why is it sent in the clear, and what does ECH change?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>SNI</strong> (Server Name Indication) tells the server, in the ClientHello, <em>which
hostname</em> the client wants to reach. It's needed because one IP can host many TLS sites, and the
server must know which name to choose the correct certificate for — and that decision happens
<strong>before</strong> the encrypted channel exists, so SNI is sent in <strong>plaintext</strong>. The
side effect is a privacy/censorship leak: anyone on-path can see the hostname and block or log by name.
<strong>ECH</strong> (Encrypted Client Hello) fixes this by encrypting the ClientHello (including SNI)
to a key the server publishes (via DNS), so on-path observers can no longer read the hostname — closing
the last major plaintext identity leak in a TLS connection.
</details>

---

**Q3. How does TLS 1.3 reduce what a sniffer can see compared to TLS 1.2?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
In <strong>TLS 1.2</strong>, a large part of the handshake is in the clear, most notably the server's
<strong>certificate</strong> — so a passive capture reveals the server's identity directly. In
<strong>TLS 1.3</strong>, after the initial ClientHello/ServerHello key-establishment exchange, the rest
of the handshake — including the <strong>certificate</strong> and other handshake messages — is
<strong>encrypted</strong>. So a sniffer can no longer read the cert on the wire; it's left with mainly
the ClientHello fields (still including SNI unless ECH is used), the server IP/port, and traffic
patterns. TLS 1.3 thus shrinks the cleartext surface considerably, and ECH + encrypted DNS shrink it
further.
</details>

---

## Homework

Capture both a TLS 1.2 and a TLS 1.3 handshake to the same kind of server (force versions with
`openssl s_client -tls1_2` vs `-tls1_3`) and compare in Wireshark which handshake messages are
readable in each. Then argue how the trend toward TLS 1.3 + ECH + QUIC shifts where network policy and
monitoring must happen.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
In the <strong>TLS 1.2</strong> capture you can read more of the handshake — ServerHello, the
<strong>Certificate</strong> message (so the server's identity), and the key-exchange parameters are all
visible — in addition to the ClientHello's SNI. In the <strong>TLS 1.3</strong> capture, after the
opening hello messages the certificate and remaining handshake are <strong>encrypted</strong>, so
Wireshark shows them as encrypted handshake data; you're left mainly with the ClientHello (SNI, ciphers,
ALPN) and metadata. The trend (TLS 1.3 → ECH hides SNI → QUIC encrypts nearly all transport metadata
and rides UDP) steadily removes the cleartext fields that in-network devices relied on. The consequence:
<strong>policy and monitoring can no longer live "in the middle" reading the wire</strong>. Operators
must shift to (1) <strong>endpoint-based controls</strong> — agents/proxies on the client or server that
see plaintext before/after encryption — and (2) <strong>DNS-layer controls</strong> (filtering or
logging at the resolver, since names are still resolved), plus coarse signals like destination IP and
traffic analysis. In short, encryption pushes visibility and enforcement out of the network core and
onto the endpoints and DNS.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 71 — Multicast & IGMP →](lesson-71-multicast){: .btn .btn-primary }
