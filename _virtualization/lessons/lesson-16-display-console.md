---
title: "Lesson 16 — Display, Console, and Remote Access"
nav_order: 16
parent: "Phase 4: QEMU Fundamentals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 16: Display, Console, and Remote Access

## Concept

A VM needs a way to get input/output to you. There are two fundamentally different
channels:

```
   ┌──────────────────────────────────────────────────────────────┐
   │  SERIAL CONSOLE — text only, like a teletype                  │
   │   -nographic / -serial ;  perfect for servers, scripting, SSH │
   ├──────────────────────────────────────────────────────────────┤
   │  GRAPHICAL — a virtual GPU + a remote display protocol        │
   │   VGA model (std/qxl/virtio-gpu)  +  VNC or SPICE             │
   └──────────────────────────────────────────────────────────────┘
```

For a headless Linux server, a **serial console** is often all you want — it's
text, it logs cleanly, it works over SSH. For a desktop guest you want a
**graphical** display: a virtual GPU paired with a remote protocol (**VNC** or
**SPICE**) you connect to with a viewer.

---

## How It Works

### Serial console

`-nographic` disables graphics and multiplexes the guest's first serial port onto
your terminal (Lesson 12). `-serial <backend>` is the general form — you can send
serial to a `pty`, a TCP socket, a file, etc. The guest must be configured to use
the serial port as a console (Linux: `console=ttyS0` on the kernel cmdline). Serial
is ideal for: headless servers, capturing boot logs, automation, and low-bandwidth
remote access.

### Virtual GPU models

For graphics, you pick a **VGA/GPU model** — the virtual display adapter the guest
driver binds to:

| Model | Notes |
|---|---|
| `std` (`-vga std`) | Standard VGA, broad compatibility, basic 2D. |
| `qxl` (`-vga qxl`) | Paravirtual GPU designed for **SPICE**; good 2D, multi-monitor. |
| `virtio-gpu` (`-device virtio-gpu-pci`) | Modern virtio GPU; can do 3D (virgl) with the right stack. |
| `none` | No display adapter (serial-only / headless). |

### Remote display: VNC vs SPICE

The GPU's framebuffer is exported over a protocol you connect to remotely:

- **VNC** (`-vnc :0`) — simple, ubiquitous, works with any VNC client. Just pixels +
  keyboard/mouse. No audio, no USB redirection, no clipboard by default. Great for
  "I just need to see the screen," especially across firewalls/tooling that already
  speaks VNC.
- **SPICE** (`-spice ...`) — purpose-built for *virtual desktops*. Adds: clipboard
  sharing, USB redirection, audio, multi-monitor, and dynamic resolution — and it's
  more efficient for desktop workloads (it understands GUI primitives). Pair it with
  `qxl` or `virtio-gpu` and the guest's SPICE agent.

You view them with **`virt-viewer`** / **`remote-viewer`** (which speak both VNC and
SPICE), or any VNC client for VNC.

### Choosing

- **Serial console:** headless servers, boot debugging, automation, anything you'd
  normally SSH into. Lowest overhead, text-only.
- **VNC:** simple graphical access, maximum client compatibility, occasional GUI use.
- **SPICE:** interactive desktop guests where you want clipboard, USB redirection,
  audio, and smooth resizing — i.e. using the VM like a workstation.

{: .note }
> **A guest can have both at once**
> It's common to give a VM a serial console *and* a graphical display: serial for
> scripted/headless management and boot logs, graphics for interactive desktop use.
> They're independent channels. For pure servers, many people configure only serial
> (`console=ttyS0`) and skip the GPU entirely to shrink the device model (a small
> security and performance win — see Phase 14).

---

## Lab

```bash
# 1. SERIAL CONSOLE — headless, text only (guest needs console=ttyS0):
$ qemu-system-x86_64 -accel kvm -m 1G -smp 2 \
    -drive file=disk.qcow2,if=virtio -nographic
# Boot logs and a login prompt appear right in your terminal. Ctrl-A X to quit.

# 2. VNC — export the display on VNC port 5900 ( :0 = 5900, :1 = 5901, ... ):
$ qemu-system-x86_64 -accel kvm -m 2G -smp 2 -vga std \
    -drive file=disk.qcow2,if=virtio -vnc :0 &
# Connect from another machine:
$ remote-viewer vnc://HOST:5900
# (or any VNC client)

# 3. SPICE — desktop-grade, with qxl GPU; connect with remote-viewer:
$ qemu-system-x86_64 -accel kvm -m 4G -smp 4 -vga qxl \
    -drive file=disk.qcow2,if=virtio \
    -spice port=5930,disable-ticketing=on \
    -device virtio-serial-pci \
    -chardev spicevmc,id=spicechannel0,name=vdagent \
    -device virtserialport,chardev=spicechannel0,name=com.redhat.spice.0 &
$ remote-viewer spice://HOST:5930
# With the guest's spice-vdagent installed you get clipboard + auto-resize.

# 4. Inspect available VGA models:
$ qemu-system-x86_64 -device help 2>&1 | grep -iE 'vga|qxl|virtio-gpu'
$ kill %1 %2 2>/dev/null
```

**Expected result:** The serial console gives you a text login in your terminal;
VNC exposes the screen to any VNC client on port 5900; SPICE gives a richer desktop
experience (clipboard, resize) with `remote-viewer` when the guest agent is
present.

---

## Further Reading

| Topic | Link |
|---|---|
| Virtual Network Computing (VNC) | [Wikipedia — Virtual Network Computing](https://en.wikipedia.org/wiki/Virtual_Network_Computing) |
| SPICE protocol | [Wikipedia — SPICE (protocol)](https://en.wikipedia.org/wiki/SPICE_(protocol)) |
| QEMU display options | [qemu.org — Display devices](https://www.qemu.org/docs/master/system/invocation.html) |
| `virt-viewer` / `remote-viewer` | [virt-manager.org](https://virt-manager.org/) |
| Serial console (Linux) | [kernel.org — Serial console](https://docs.kernel.org/admin-guide/serial-console.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. When would you prefer SPICE over VNC for a desktop guest? When is a plain serial console the right answer?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Prefer SPICE for interactive desktop guests where you want workstation-like features: shared clipboard, USB device redirection, audio, multi-monitor, and dynamic resolution/auto-resize — SPICE is purpose-built and more efficient for GUI workloads. Use a plain serial console for headless servers, boot/kernel debugging, and automation — anything text-only you'd normally SSH into. Serial has the lowest overhead, logs cleanly, and needs no GPU. VNC sits in between: simple graphical access with maximum client compatibility but without SPICE's desktop extras.
</details>

---

**Q2. What two components make up "graphical" output for a VM, and what does each provide?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(1) A virtual GPU/VGA model (std, qxl, virtio-gpu) — the display adapter the guest's video driver binds to, which renders the framebuffer. (2) A remote display protocol (VNC or SPICE) — which exports that framebuffer and carries keyboard/mouse (and for SPICE, clipboard/USB/audio) to a viewer like remote-viewer. The GPU model produces the pixels; the protocol delivers them to you.
</details>

---

**Q3. For a headless Linux server VM, what must be configured in the *guest* for the serial console to show login and boot messages?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The guest's kernel must be told to use the serial port as a console — add <code>console=ttyS0</code> (often <code>console=ttyS0,115200</code>) to the kernel command line — and a getty/login service must run on ttyS0 so you get a login prompt. On the QEMU side you use -nographic or -serial to wire that serial port to your terminal. Without console=ttyS0 the kernel logs to the (absent) graphical console and the serial line stays blank.
</details>

---

## Homework

Boot a guest twice: once with `-nographic` (serial) and once with `-vnc :1` plus `-vga std`. For the VNC case, connect with `remote-viewer vnc://localhost:5901`. Explain why the VNC port was 5901 and not 5900, and describe one capability SPICE would add that neither plain VNC nor the serial console offers.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
VNC display numbers map to TCP ports as 5900 + N, so <code>-vnc :1</code> means display 1 → port 5901 (and <code>:0</code> would be 5900). SPICE adds desktop-integration capabilities that neither plain VNC nor serial provides — for example USB device redirection (plug a host USB device straight into the guest), shared clipboard between host and guest, audio, and automatic resolution resizing via the guest spice-vdagent. Serial is text-only; VNC is pixels+input only; SPICE is built for full desktop interaction.
</details>
