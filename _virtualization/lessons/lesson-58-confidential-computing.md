---
title: "Lesson 58 — Confidential Computing"
nav_order: 58
parent: "Phase 14: Nested Virt & Security"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 58: Confidential Computing

## Concept

Every security mechanism so far (sVirt, sandboxing) protects the **host from the guest**, or
guests from each other — but they all **trust the hypervisor completely**. Confidential
computing flips the threat model: it protects the **guest from the host/hypervisor** itself,
using hardware to encrypt guest memory so even a malicious or compromised host can't read
it.

```
   TRADITIONAL                          CONFIDENTIAL (SEV-SNP / TDX)
   ───────────                          ────────────────────────────
   host/hypervisor can read ALL          guest RAM is ENCRYPTED by the CPU
   guest memory (it manages it)          with a key the HOST cannot access
   → fine if you trust the host          → host sees only ciphertext;
                                           tampering is detected (integrity)
                                         + attestation proves the guest is
                                           genuine before secrets are released
```

This matters for cloud: you run a VM on someone else's hardware and want them *unable* to
read your data — even with full physical and hypervisor access.

---

## How It Works

### The threat model shift

Ordinary virtualization assumes the hypervisor is trusted — it allocates and can inspect
guest memory, which is how snapshots, migration, and debugging work. Confidential computing
**removes the hypervisor from the trust boundary**: the guest's data should be protected
*even if the host OS, hypervisor, or a malicious cloud admin is hostile*. The hardware (CPU)
becomes the root of trust instead of the host software.

### The technologies (high level)

- **AMD SEV** (Secure Encrypted Virtualization) — encrypts guest **memory** with a per-VM
  key managed by the CPU's secure processor; the host can't read plaintext guest RAM.
- **AMD SEV-ES** (Encrypted State) — additionally encrypts the guest's **CPU register
  state** on VM exits, so the host can't read/modify registers during exits.
- **AMD SEV-SNP** (Secure Nested Paging) — adds **integrity** protection: prevents the host
  from corrupting, replaying, or remapping guest memory (not just reading it), plus
  attestation. The strongest of the SEV line.
- **Intel TDX** (Trust Domain Extensions) — Intel's equivalent: encrypted, integrity-
  protected "trust domains" isolated from the hypervisor, with attestation.

### Encrypted memory, encrypted state, integrity, attestation

- **Encrypted memory:** the CPU transparently encrypts/decrypts guest RAM with a key the
  host never sees, so reads of guest memory by the host yield ciphertext.
- **Encrypted register state** (ES/SNP/TDX): CPU state isn't exposed to the host on exits.
- **Integrity** (SNP/TDX): the host can't silently alter, replay, or remap guest pages
  without detection.
- **Attestation:** before you trust the VM with secrets, the hardware can produce a signed
  **attestation report** proving the VM booted the expected firmware/configuration on
  genuine confidential-computing hardware. You verify it, then release secrets (e.g. a disk
  decryption key) only to a proven-genuine guest.

### What changes operationally

Confidential VMs constrain the things that rely on the host reading guest memory/state:

- **Migration** is restricted/specialized — you can't just copy plaintext RAM; it must stay
  encrypted and re-keyed/attested, so live migration is limited or needs special support.
- **Debugging/introspection** by the host is curtailed (that's the point) — host-side memory
  inspection sees ciphertext.
- **Device model constraints:** DMA to encrypted memory needs bounce buffers / shared
  (unencrypted) regions for I/O; virtio is adapted to designate shared buffers. Some
  passthrough/snapshot/ballooning features are limited.
- **Boot/firmware** must be measured and attested (OVMF builds with SEV/TDX support).

{: .note }
> **What SEV-SNP / TDX address that ordinary KVM does not**
> Ordinary KVM trusts the hypervisor fully — the host manages and can read guest memory and
> CPU state, so a malicious or compromised host (or cloud operator with physical access) can
> read or tamper with your VM's data. SEV-SNP and TDX address exactly that: they encrypt
> guest memory (and register state) with CPU-managed keys the host can't access, add
> integrity protection so the host can't silently alter/replay/remap guest memory, and
> provide attestation so you can verify the VM is genuine before entrusting it with secrets.
> The protected party flips from "host (from guest)" to "guest (from host)."

---

## Lab

```bash
# 1. Does the CPU support confidential computing? (AMD SEV family / Intel TDX)
$ grep -oE 'sev|sev_es|sev_snp|tdx' /proc/cpuinfo | sort -u
sev
sev_es
sev_snp
#   (AMD shown; Intel TDX hosts expose it differently — check dmesg/kernel)

# 2. Is SEV enabled in the kernel/KVM?
$ dmesg | grep -iE 'SEV|SNP|TDX' | head
[ ... ] ccp: SEV API: 1.55
[ ... ] kvm_amd: SEV enabled (ASIDs ...)
[ ... ] kvm_amd: SEV-ES enabled
[ ... ] kvm_amd: SEV-SNP enabled
$ ls /sys/module/kvm_amd/parameters/ | grep -iE 'sev'
sev  sev_es  sev_snp

# 3. Launch a confidential (SEV-SNP) guest (requires OVMF with SNP support):
$ qemu-system-x86_64 -accel kvm -m 4G -smp 2 -cpu host \
    -machine q35,confidential-guest-support=sev0 \
    -object sev-snp-guest,id=sev0,cbitpos=51,reduced-phys-bits=1 \
    -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF.amdsev.fd \
    -drive file=cvm.qcow2,if=virtio -nographic &

# 4. From INSIDE the guest, confirm it's running encrypted:
#   (guest)$ dmesg | grep -i 'Memory Encryption\|SEV'
#   AMD Memory Encryption Features active: SEV SEV-ES SEV-SNP
#   (guest)$ lscpu | grep -i 'Virtualization\|encryption'

# 5. From the HOST, demonstrate you canNOT read guest plaintext:
#    Host-side memory inspection of the guest's RAM yields ciphertext, not data.
#    (Conceptually: tools that dump guest memory see encrypted bytes.)

# 6. Attestation (concept): request a signed report proving the guest's identity/config,
#    then a key-broker releases secrets only if the report verifies. (Tooling varies:
#    e.g. snpguest / virtee tools fetch and verify the attestation report.)
$ kill %1 2>/dev/null
```

**Expected result:** The CPU advertises `sev`/`sev_es`/`sev_snp` (or TDX), the kernel
reports SEV(-SNP) enabled, and a confidential guest boots reporting active memory
encryption inside it — while the host can only see ciphertext for that guest's memory.

---

## Further Reading

| Topic | Link |
|---|---|
| Confidential computing | [Wikipedia — Confidential computing](https://en.wikipedia.org/wiki/Confidential_computing) |
| AMD SEV / SEV-SNP | [Wikipedia — AMD SEV](https://en.wikipedia.org/wiki/Secure_Encrypted_Virtualization) |
| Intel TDX | [Wikipedia — Trust Domain Extensions](https://en.wikipedia.org/wiki/Trust_Domain_Extensions) |
| Remote attestation | [Wikipedia — Trusted Computing (attestation)](https://en.wikipedia.org/wiki/Trusted_Computing#Remote_attestation) |
| AMD SEV in KVM | [kernel.org — AMD memory encryption](https://docs.kernel.org/virt/kvm/x86/amd-memory-encryption.html) |

---

## Checkpoint

Answer each question in your own words first. Then reveal the answer to check yourself.

---

**Q1. Traditional virtualization trusts the hypervisor completely. What threat do SEV-SNP / TDX address that ordinary KVM does not?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
They address a malicious or compromised host/hypervisor (or cloud operator with physical access) reading or tampering with the guest's data. Ordinary KVM trusts the hypervisor fully — the host manages and can inspect guest memory and CPU state — so anyone controlling the host can see your VM's data. SEV-SNP and TDX protect the guest *from* the host: they encrypt guest memory (and register state) with CPU-managed keys the host can't access, add integrity protection so the host can't silently alter/replay/remap guest memory, and provide attestation to prove the VM is genuine before secrets are released. The protected party flips from "host from guest" to "guest from host."
</details>

---

**Q2. What is the difference between encrypting guest memory (SEV) and adding integrity protection (SEV-SNP)?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
SEV encrypts guest memory so the host can't *read* plaintext — it provides confidentiality. But encryption alone doesn't stop the host from *tampering*: a malicious host could still corrupt, replay old contents into, or remap guest pages (attacks on integrity), even without reading them. SEV-SNP (Secure Nested Paging) adds integrity protection that detects/prevents such manipulation of guest memory by the host, plus attestation. So SEV = confidentiality (can't read); SEV-SNP = confidentiality + integrity (can't read AND can't silently alter/replay/remap). SEV-ES sits in between by also protecting CPU register state on exits.
</details>

---

**Q3. Why does confidential computing complicate live migration and host-side debugging?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Both rely on the host being able to read the guest's memory/state in plaintext — exactly what confidential computing forbids. Live migration normally copies plaintext RAM between hosts; with an encrypted, integrity-protected guest the memory must remain encrypted and be securely re-keyed and re-attested on the destination, so migration is restricted or needs special hardware/firmware support. Host-side debugging/introspection (dumping guest memory) now yields ciphertext, so the usual tools can't read guest state — which is the intended protection. The same constraint forces I/O changes (shared/bounce buffers for DMA) and limits some snapshot/ballooning/passthrough features.
</details>

---

## Homework

Check whether your CPU supports SEV/SEV-SNP or TDX (`grep -oE 'sev|sev_es|sev_snp|tdx' /proc/cpuinfo` and `dmesg | grep -iE 'SEV|TDX'`). Whether or not it does, write a short explanation of: (a) who the "attacker" is in the confidential-computing threat model, and (b) why attestation is necessary before sending secrets to a confidential VM. Tie (b) back to why encryption alone isn't sufficient to trust the guest.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
(a) In the confidential-computing threat model the "attacker" is the host/hypervisor itself — a malicious or compromised host OS, hypervisor, or cloud operator/admin with physical and software access to the machine running your VM. (Traditional models trust the host and treat the guest as the threat; this flips it.) (b) Attestation is necessary because encryption protects the data but doesn't, by itself, prove *what* is running. Before you hand a confidential VM real secrets (e.g. a disk-decryption key or API credentials), you need cryptographic proof that it's a genuine confidential VM on real SEV-SNP/TDX hardware, booted with the expected firmware/configuration and not, say, a fake VM the host stood up to harvest your secrets. The hardware produces a signed attestation report; you verify it against the vendor's keys and your expected measurements, and only then release secrets. Encryption alone isn't enough to *trust* the guest because a hostile host could spin up a look-alike VM (without the protections, or with tampered firmware) and request your secrets — attestation is what distinguishes a real, correctly-configured confidential guest from an impostor before any secret leaves your control.
</details>
