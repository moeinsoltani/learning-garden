---
title: "Lesson 15 — The QEMU Monitor: HMP and QMP"
nav_order: 15
parent: "Phase 4: QEMU Fundamentals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 15: The QEMU Monitor — HMP and QMP

## Concept

A running QEMU is not a black box. It exposes a **control channel** through which
you can introspect and command the live VM: query state, hotplug devices, take
snapshots, start a migration, power it down. There are two dialects of this channel:

```
   ┌────────────────────────────────────────────────────────────┐
   │  HMP  — Human Monitor Protocol                              │
   │   • plain text, for HUMANS at a terminal                    │
   │   • (qemu) info cpus ;  device_add ... ;  system_powerdown  │
   ├────────────────────────────────────────────────────────────┤
   │  QMP  — QEMU Machine Protocol                               │
   │   • JSON, for PROGRAMS (libvirt, scripts)                   │
   │   • {"execute":"query-status"}                              │
   └────────────────────────────────────────────────────────────┘
        Same capabilities underneath; different audience.
```

HMP is what *you* type interactively. QMP is what *tools* speak. libvirt drives
QEMU exclusively over **QMP** — which is why understanding QMP demystifies what
libvirt is actually doing.

---

## How It Works

### HMP — the human monitor

HMP is the classic interactive monitor. With `-nographic` you toggle to it with
**Ctrl-A C**; or attach it explicitly with `-monitor`. Useful commands:

| Command | Does |
|---|---|
| `info status` | Running / paused? |
| `info cpus` | List vCPUs and their host thread IDs. |
| `info block` | Block devices and their backing files. |
| `info network` | NICs and netdevs. |
| `info migrate` | Migration progress. |
| `device_add` / `device_del` | Hotplug / unplug a device. |
| `system_powerdown` | Send an ACPI power button event (graceful). |
| `stop` / `cont` | Pause / resume the vCPUs. |
| `savevm` / `loadvm` | Internal snapshot (Phase 6). |

### QMP — the machine protocol

QMP exposes the *same* capabilities as a **JSON API** over a socket. Every libvirt
action ultimately becomes QMP commands. A session looks like:

```json
// QEMU greets you, then you must enable commands:
<- {"QMP": {"version": {...}, "capabilities": [...]}}
-> {"execute": "qmp_capabilities"}
<- {"return": {}}
// now you can issue commands:
-> {"execute": "query-status"}
<- {"return": {"status": "running", "running": true}}
-> {"execute": "query-block"}
<- {"return": [ ... block devices ... ]}
```

You expose it with `-qmp` (e.g. a Unix socket) and talk to it with a tool like
`qmp-shell` or plain `socat`/`nc`.

### Why the distinction matters for automation

- **HMP is unstable for scripting:** its output is free-form human text that can
  change between QEMU versions. Parsing it is fragile.
- **QMP is a contract:** structured JSON, versioned, with introspection
  (`query-commands`, `query-qmp-schema`). Tools can discover capabilities and rely
  on stable shapes.

So *humans* use HMP for quick interactive pokes; *software* (libvirt, OpenStack,
backup tools) uses QMP. (You can even run HMP commands *through* QMP via
`human-monitor-command`.)

{: .note }
> **What libvirt uses under the hood**
> libvirt talks to each running QEMU exclusively over **QMP**. When you run
> `virsh attach-device`, `virsh snapshot-create`, or `virsh migrate`, libvirt
> translates that into QMP JSON commands on the VM's monitor socket. This is why
> the distinction matters: anything libvirt can do is ultimately a QMP command, and
> when you debug libvirt you often end up reading its QMP exchange in the logs.

---

## Lab

```bash
# Start a VM exposing BOTH a QMP socket and using -nographic (HMP via Ctrl-A C):
$ qemu-system-x86_64 -accel kvm -m 1G -smp 2 -nographic \
    -drive file=disk.qcow2,if=virtio \
    -qmp unix:/tmp/qmp.sock,server,nowait &

# ---- HMP (interactive) ----
# Press Ctrl-A then C to reach the (qemu) prompt, then:
#   (qemu) info status
#   VM status: running
#   (qemu) info block
#   ... your virtio disk ...
#   (qemu) info cpus
#   * CPU #0: thread_id=12347
#     CPU #1: thread_id=12348
# Ctrl-A C again returns to the guest console.

# ---- QMP (programmatic) ----
# Use the bundled qmp-shell, or raw with socat. First, qmp-shell:
$ qmp-shell /tmp/qmp.sock
(QEMU) query-status
{"return": {"status": "running", "running": true}}
(QEMU) query-block
{"return": [ ... ]}
(QEMU) quit

# Raw QMP with socat (note: must send qmp_capabilities first):
$ printf '%s\n%s\n' \
    '{"execute":"qmp_capabilities"}' \
    '{"execute":"query-status"}' | socat - UNIX-CONNECT:/tmp/qmp.sock
{"QMP": {"version": ...}}
{"return": {}}
{"return": {"status": "running", "running": true}}

# ---- Hotplug a disk via HMP device_add ----
$ qemu-img create -f qcow2 extra.qcow2 1G
# In the HMP monitor:
#   (qemu) drive_add 0 file=extra.qcow2,if=none,id=d1,format=qcow2
#   (qemu) device_add virtio-blk-pci,drive=d1,id=vblk1
#   (qemu) info block      # the new disk now appears; guest sees a hotplugged disk
```

**Expected result:** You can query the same VM two ways — typing `info` commands in
HMP and sending JSON `query-*` commands over the QMP socket — and you can hot-add a
disk that appears live in the guest, all without restarting the VM.

---

## Further Reading

| Topic | Link |
|---|---|
| QMP (QEMU Machine Protocol) | [qemu.org — QMP](https://www.qemu.org/docs/master/interop/qmp-intro.html) |
| QMP command reference | [qemu.org — QMP reference](https://www.qemu.org/docs/master/interop/qemu-qmp-ref.html) |
| HMP monitor | [qemu.org — Monitor](https://www.qemu.org/docs/master/system/monitor.html) |
| libvirt & QEMU | [libvirt.org — QEMU driver](https://libvirt.org/drvqemu.html) |
| `socat` | [man7.org — socat(1)](https://man7.org/linux/man-pages/man1/socat.1.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. What does libvirt use under the hood to talk to a running QEMU process — HMP or QMP? Why does the distinction matter for automation?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
libvirt uses QMP. It matters because QMP is a structured, versioned JSON API with introspection — a stable contract that programs can reliably parse and that supports capability discovery. HMP, by contrast, is free-form human-readable text whose output can change between QEMU versions, making it fragile to script against. Automation (libvirt, OpenStack, backup tools) needs the predictable JSON of QMP; humans use HMP for quick interactive commands.
</details>

---

**Q2. Before you can issue normal QMP commands on a fresh connection, what one command must you send first, and why?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You must send <code>{"execute":"qmp_capabilities"}</code> first. When you connect, QEMU sends a greeting and enters "capabilities negotiation" mode where most commands are rejected; issuing qmp_capabilities completes the handshake (optionally enabling negotiated features) and switches the session into command mode so subsequent commands like query-status work.
</details>

---

**Q3. Name two things you can do through the monitor on a *running* VM without restarting it.**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Examples: hotplug or unplug a device (device_add/device_del — e.g. add a disk or NIC live); query live state (info cpus/block/network, query-status/query-block); pause and resume the vCPUs (stop/cont); take a snapshot (savevm or blockdev-snapshot); send an ACPI graceful shutdown (system_powerdown); or initiate a live migration. All operate on the live VM without a restart.
</details>

---

## Homework

Expose a QMP socket on a running VM and use `qmp-shell` (or raw JSON over `socat`) to run `query-block`, then `query-cpus-fast`. Identify, from the JSON output, (a) the host thread ID of vCPU 0 and (b) the backing file of the VM's disk. Then explain why a backup tool would prefer reading this from QMP rather than scraping the HMP `info block` text.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) query-cpus-fast returns an array where each entry has a "thread-id" field — vCPU 0's host thread ID is that value for cpu-index 0. (b) query-block returns each device with an "inserted" object whose "file" (and "image"/"backing-image") fields give the backing file path. A backup tool prefers QMP because the JSON is a stable, versioned, machine-parseable contract with explicit fields — it can reliably extract the file path and node names, discover capabilities, and not break when QEMU changes the human-readable HMP formatting. Scraping HMP text is fragile and version-dependent.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 16 — Display, Console, and Remote Access →](lesson-16-display-console){: .btn .btn-primary }
