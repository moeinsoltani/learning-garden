---
title: "Lesson 40 — The Domain XML"
nav_order: 40
parent: "Phase 10: libvirt"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 40: The Domain XML

## Concept

The **domain XML** is libvirt's canonical description of a VM — the single source of
truth from which libvirt generates the QEMU command line (Lesson 39). Learning to read
and write it means learning to express everything from Phases 4–9 declaratively.

```
   <domain type='kvm'>                  ← accelerator (Lesson 13)
     <name>db01</name>
     <memory unit='GiB'>8</memory>      ← -m  (Phase 5)
     <vcpu>4</vcpu>                      ← -smp (Lesson 18)
     <cpu mode='host-passthrough'/>     ← -cpu (Lesson 17)
     <os><type machine='pc-q35-8.2'>... ← -machine (Lesson 13)
     <devices>
       <disk>...</disk>                  ← -blockdev + -device (Phase 6)
       <interface>...</interface>        ← -netdev + -device (Phase 8)
       <controller/> <graphics/> ...
     </devices>
   </domain>
```

Every QEMU flag you've learned has a home in this tree. Editing the XML is editing the
VM.

---

## How It Works

### The structure

A `<domain>` element contains:

- **Identity & resources:** `<name>`, `<uuid>`, `<memory>`/`<currentMemory>`, `<vcpu>`.
- **`<cpu>`:** model/mode (host-passthrough/host-model/named), topology, feature flags.
- **`<os>`:** machine type, firmware (BIOS vs OVMF/UEFI with `<loader>`/`<nvram>`), boot
  order.
- **`<devices>`:** the bulk — `<disk>` (with `<driver>` for cache/aio, `<source>`,
  `<target>` bus), `<interface>` (network type + model), `<controller>` (e.g.
  virtio-scsi), `<graphics>` (vnc/spice), `<video>`, `<hostdev>` (passthrough),
  `<memballoon>`, `<rng>`, `<channel>` (guest agent), etc.

A `<disk>` example mapping straight to Phase 6 concepts:

```xml
   <disk type='file' device='disk'>
     <driver name='qemu' type='qcow2' cache='none' io='native' discard='unmap'/>
     <source file='/var/lib/libvirt/images/db01.qcow2'/>
     <target dev='vda' bus='virtio'/>
   </disk>
```

### virsh edit and config validation

`virsh edit <dom>` opens the XML in your editor, validates it on save, and applies it to
the **persistent** definition. Invalid XML is rejected, protecting you from typos.

### Live vs persistent config — the crucial gotcha

A defined, running VM has **two** configurations:

- **Persistent (inactive) config** — what's on disk; takes effect on next boot.
- **Live (active) config** — what the running QEMU actually has right now.

`virsh edit` changes the **persistent** config. So if you bump `<vcpu>` from 4 to 8 with
`virsh edit` while the VM runs, **nothing changes** in the running guest — the edit
applies only at the next start. To change a running VM you must either:

- Use a command with `--live` (and/or `--config`), e.g. `virsh setvcpus`,
  `virsh setmem`, `virsh attach-device`, with the appropriate flag, **and** the change
  must be hot-pluggable; or
- Restart the VM so it re-reads the persistent config.

The `--config` flag affects the persistent definition; `--live` affects the running
instance; both together change now *and* persist.

### Hotplug via attach-device / detach-device

You can add/remove devices on a running VM with `virsh attach-device <dom> dev.xml
--live` (and `--config` to persist), e.g. hot-adding a disk or NIC. This issues the QMP
`device_add` (Lesson 15) under the hood. Not everything is hot-pluggable (CPUs and memory
have constraints), but disks, NICs, and USB generally are.

{: .note }
> **"I changed &lt;vcpu&gt; in virsh edit but nothing happened"**
> Because virsh edit only updates the *persistent* config, which the *running* QEMU isn't
> using. The running guest keeps its live config until you either restart it (so it
> re-reads persistent) or apply a live change with the proper command/flags (e.g.
> <code>virsh setvcpus &lt;dom&gt; 8 --live</code>, within the domain's maxvcpus and only
> if vCPU hotplug is supported). Always think: am I editing on-disk config or the live
> instance?

---

## Lab

```bash
# 1. Dump a domain's full XML and read its structure:
$ virsh dumpxml db01 | head -40
<domain type='kvm'>
  <name>db01</name>
  <memory unit='KiB'>8388608</memory>
  <vcpu placement='static'>4</vcpu>
  <cpu mode='host-passthrough'/>
  <os><type arch='x86_64' machine='pc-q35-8.2'>hvm</type></os>
  <devices>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='none' io='native'/>
      <source file='/var/lib/libvirt/images/db01.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <interface type='network'>
      <source network='default'/>
      <model type='virtio'/>
    </interface>
  ...

# 2. Edit the persistent config (validated on save):
$ virsh edit db01     # change <vcpu>4</vcpu> -> 8, save

# 3. Show that the RUNNING guest didn't change (live still 4):
$ virsh vcpucount db01
maximum      config         8
maximum      live           4         ← live unchanged by virsh edit
current      config         8
current      live           4

# 4. Apply a LIVE change correctly (within maxvcpus, if hotplug supported):
$ virsh setvcpus db01 6 --live
$ virsh vcpucount db01 | grep 'current      live'
current      live           6         ← now the running guest has 6

# 5. Hot-attach a disk to the running VM (and persist it):
$ cat > newdisk.xml <<'EOF'
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2'/>
  <source file='/var/lib/libvirt/images/extra.qcow2'/>
  <target dev='vdb' bus='virtio'/>
</disk>
EOF
$ qemu-img create -f qcow2 /var/lib/libvirt/images/extra.qcow2 5G
$ virsh attach-device db01 newdisk.xml --live --config
#   (guest)$ lsblk    # /dev/vdb appears without reboot
```

**Expected result:** `virsh dumpxml` shows each Phase 4–8 concept as an XML element. A
`virsh edit` bump of `<vcpu>` changes only the *config* value, not *live* — until you use
`setvcpus --live`. `attach-device --live` hot-adds a disk the guest sees immediately.

---

## Further Reading

| Topic | Link |
|---|---|
| libvirt domain XML format | [libvirt.org — Domain XML format](https://libvirt.org/formatdomain.html) |
| `virsh` (edit, attach-device, setvcpus) | [libvirt.org — virsh](https://libvirt.org/manpages/virsh.html) |
| libvirt disk/driver elements | [libvirt.org — Hard drives](https://libvirt.org/formatdomain.html#hard-drives-floppy-disks-cdroms) |
| Live vs persistent config | [libvirt.org — Domain lifecycle](https://libvirt.org/formatdomain.html) |
| QMP device_add (hotplug) | [qemu.org — QMP](https://www.qemu.org/docs/master/interop/qemu-qmp-ref.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. You change `<vcpu>` in `virsh edit` but the running guest doesn't change. Why, and how do you apply it?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
virsh edit modifies the *persistent* (on-disk) config, but the *running* QEMU uses its *live* config, which it loaded at start — so the edit only takes effect the next time the VM boots. To change the running guest now, either restart the VM (so it re-reads the persistent config) or apply a live change with the proper command, e.g. <code>virsh setvcpus &lt;dom&gt; N --live</code> (optionally with --config to also persist), valid only within the domain's maximum vCPUs and if vCPU hotplug is supported.
</details>

---

**Q2. What is the difference between `--config` and `--live` on virsh commands?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
--config affects the persistent (on-disk) definition, so the change applies on the next boot. --live affects the currently running instance, taking effect immediately (for hot-pluggable changes). Using both together changes the running VM now AND persists it for future boots. With neither, the default depends on the command/domain state. Distinguishing them is essential: --live alone won't survive a reboot; --config alone won't affect the running guest.
</details>

---

**Q3. Where in the domain XML do the Phase 6 caching settings (e.g. cache=none, aio=native) live, and what element holds the disk's bus type (virtio vs SATA)?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The caching/AIO settings live on the disk's <code>&lt;driver&gt;</code> element — e.g. <code>&lt;driver name='qemu' type='qcow2' cache='none' io='native' discard='unmap'/&gt;</code>. The bus type (the device model / how the guest sees the disk) is set by the <code>&lt;target&gt;</code> element's <code>bus</code> attribute — e.g. <code>&lt;target dev='vda' bus='virtio'/&gt;</code> for virtio-blk (vdX) versus <code>bus='sata'</code> or a virtio-scsi controller for sdX. So <code>&lt;driver&gt;</code> = backend behavior (format/cache/aio), <code>&lt;target&gt;</code> = frontend bus/naming.
</details>

---

## Homework

On a running VM, use `virsh edit` to change `<vcpu>` and confirm with `virsh vcpucount` that only the *config* value changed, not *live*. Then apply the change to the live guest correctly with `virsh setvcpus ... --live`. Finally, hot-attach a second disk with `virsh attach-device ... --live --config` and confirm the guest sees it without rebooting. Summarize the live-vs-persistent rule in your own words.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
After virsh edit, vcpucount shows the config maximum/current updated but the live value unchanged — proving the edit only touched persistent config. <code>virsh setvcpus &lt;dom&gt; N --live</code> then raises the live count (within maxvcpus), and the guest sees the new CPUs without reboot. <code>virsh attach-device ... --live --config</code> hot-adds the disk (visible via lsblk in the guest immediately) and also persists it for next boot. The rule: libvirt keeps two configs — persistent (on disk, applied at next start) and live (the running instance). Editing XML changes persistent only; to affect the running VM use --live, and to make a change both immediate and durable use --live --config (and the change must be hot-pluggable / within limits).
</details>
