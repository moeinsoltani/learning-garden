---
title: "Lesson 36 — eBPF Fundamentals"
nav_order: 36
parent: "Phase 11: eBPF"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 36: eBPF Fundamentals

## Concept

**eBPF** (extended Berkeley Packet Filter) lets you load small programs into the running kernel that execute at defined hook points — when a packet arrives, when a syscall fires, when a function is called. It's a safe, programmable kernel: you extend kernel behavior *without* writing a kernel module or rebooting. This is the engine behind Cilium (Kubernetes networking), modern observability tools (bpftrace), and line-rate packet processing.

```
   your C code → clang compiles to → eBPF bytecode → [ verifier ] → JIT → runs in kernel
                                                          │
                                              rejects anything unsafe
                                              (unbounded loops, bad memory access)
```

The headline guarantee: an eBPF program **cannot crash the kernel**. Before any program runs, the **verifier** mathematically proves it's safe — bounded, no invalid memory access, terminates. That's what makes "run my code in the kernel" acceptable.

---

## How it works

The lifecycle:

1. **Write** a program in restricted C (a subset — no unbounded loops, limited stack, only approved helper calls).
2. **Compile** with `clang` to eBPF bytecode (an architecture-independent instruction set).
3. **Load** into the kernel (via `bpftool`, libbpf, or a framework), attaching it to a **hook**.
4. **Verify** — the kernel's verifier walks all possible execution paths and rejects the program if it can't prove safety.
5. **JIT-compile** the bytecode to native machine code for speed.
6. **Run** at the hook, on every event, until detached.

**Program types** (where they attach) include:

| Type | Hook | Use |
|---|---|---|
| `XDP` | Driver RX, before `skb` allocation | Earliest/fastest packet processing (Lesson 38) |
| `TC` (`sched_cls`) | After `skb`, in the tc datapath | Classification, shaping, richer packet context |
| `socket filter` | On a socket | Filter/inspect a socket's traffic |
| `kprobe` / `tracepoint` | Kernel functions / static trace points | Observability, tracing |

The **verifier** enforces: **bounded loops** (the kernel must prove the program terminates — historically loops had to be unrolled; bounded loops are now allowed but must have a provable limit), **no null/out-of-bounds memory access** (every pointer is checked, packet data access must be bounds-checked against `data_end`), a limited stack, and only **approved helper functions** for interacting with the kernel. If any path could be unsafe, the program is rejected at load time — before it ever runs.

The toolchain: **`clang`** (compiler), **`libbpf`** (loading library), **BTF** (BPF Type Format — type information that enables "compile once, run everywhere" portability across kernels), and **`bpftool`** (the swiss-army inspection/management CLI).

{: .note }
> **Why eBPF can't run an infinite loop**
> The verifier performs a static analysis of every possible execution path through the program *before* loading it. For loops, it must be able to prove an upper bound on the number of iterations (originally by requiring the compiler to fully unroll them; modern verifiers accept loops with a provable bounded trip count). A loop whose termination can't be proven is rejected. This guarantees every eBPF program **terminates in bounded time**, so it can't hang the kernel — critical, because these programs run in kernel context on hot paths (every packet, every syscall) where an infinite loop would freeze the machine. The same analysis proves memory safety, so a buggy program is refused at load rather than crashing the kernel at runtime.

---

## New Commands This Lesson

| Command | What it does |
|---|---|
| `clang -O2 -g -target bpf -c prog.c -o prog.o` | Compile C to eBPF object |
| `bpftool prog list` | List loaded eBPF programs |
| `bpftool prog show id <N>` | Details of one program |
| `bpftool prog dump xlated id <N>` | Disassemble the (verified) bytecode |
| `bpftool feature probe` | What eBPF features the kernel supports |
| `bpftool net show` | Show programs attached to network hooks |
| `ip link set dev <if> xdp obj prog.o sec xdp` | Attach an XDP program to an interface |

---

## Lab

We'll write, compile, load, and inspect a minimal XDP program (the simplest "hello world" of eBPF networking) — one that just passes all packets.

### Step 1 — Install the toolchain

```bash
$ sudo apt install -y clang llvm libbpf-dev bpftool linux-headers-$(uname -r)
$ clang --version
$ bpftool version
```

### Step 2 — Write a minimal XDP program

```c
// xdp_pass.c  — the simplest XDP program: let every packet through
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_pass_prog(struct xdp_md *ctx) {
    return XDP_PASS;     // verdict: pass the packet up the stack unchanged
}

char _license[] SEC("license") = "GPL";
```

`SEC("xdp")` marks the function as an XDP program; `return XDP_PASS` is the verdict (do nothing, let the packet continue).

### Step 3 — Compile to eBPF bytecode

```bash
$ clang -O2 -g -target bpf -c xdp_pass.c -o xdp_pass.o
$ file xdp_pass.o
xdp_pass.o: ELF 64-bit ... eBPF ...
```

### Step 4 — Attach to an interface (in a namespace, safely)

```bash
$ sudo ip netns add lab
$ sudo ip netns exec lab ip link add eth0 type dummy
$ sudo ip netns exec lab ip link set eth0 up
$ sudo ip netns exec lab ip link set dev eth0 xdp obj xdp_pass.o sec xdp
$ sudo ip netns exec lab ip link show eth0
... prog/xdp id N ...        # the XDP program is attached
```

### Step 5 — Inspect the loaded program

```bash
$ sudo bpftool prog list
N: xdp  name xdp_pass_prog  tag ...  gpl
    loaded_at ...  uid 0
    xlated NNB  jited NNB  ...        # verified ('xlated') and JIT-compiled ('jited')

$ sudo bpftool prog dump xlated id N
   0: (b7) r0 = 2                     # XDP_PASS == 2
   1: (95) exit
```

The disassembly shows the verified bytecode the kernel actually runs — here, "set return value to 2 (XDP_PASS) and exit." `jited` size being nonzero confirms it was JIT-compiled to native code.

### Step 6 — Detach and observe

```bash
$ sudo ip netns exec lab ip link set dev eth0 xdp off
$ sudo bpftool prog list | grep xdp_pass    # gone
```

### Step 7 — (Optional) Watch the verifier reject unsafe code

Try adding an unbounded loop or an out-of-bounds packet read to the program, recompile, and attempt to load — the kernel refuses with a verifier error explaining exactly which instruction/path it couldn't prove safe. This is the safety net in action.

### Step 8 — Clean up

```bash
$ sudo ip netns delete lab
```

---

## Further Reading

| Topic | Link |
|---|---|
| eBPF | [Wikipedia — eBPF](https://en.wikipedia.org/wiki/EBPF) |
| eBPF official site | [ebpf.io](https://ebpf.io/) |
| The BPF verifier | [kernel.org — BPF verifier](https://docs.kernel.org/bpf/verifier.html) |
| `bpftool` | [man7.org — bpftool(8)](https://man7.org/linux/man-pages/man8/bpftool.8.html) |
| BTF (BPF Type Format) | [kernel.org — BTF](https://docs.kernel.org/bpf/btf.html) |

---

## Checkpoint

**Q1. What prevents an eBPF program from running an infinite loop (or otherwise hanging/crashing the kernel)?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The **verifier**. Before the kernel will load any eBPF program, the verifier performs a static analysis of *every possible execution path* through the bytecode and must *prove* the program is safe and terminating. For loops, it requires a provable upper bound on the number of iterations — historically this meant loops had to be fully unrolled by the compiler; modern verifiers accept loops only if they have a statically provable bounded trip count. A loop whose termination can't be proven is **rejected at load time**, so it never runs. The same path analysis also proves memory safety — every pointer dereference and every packet-data access must be bounds-checked (e.g., against `data_end`), or the program is refused. Because all of this is verified *before* execution, an eBPF program is guaranteed to terminate in bounded time and never access invalid memory, which is exactly why it's safe to run untrusted-ish code in kernel context on hot paths. The protection is preventive (reject bad programs at load) rather than reactive (catch crashes at runtime).
</details>

---

**Q2. Describe the journey from C source to a running eBPF program. What does each major stage do?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
1. **Write** the program in a restricted subset of C (no unbounded loops, limited stack, only approved helper functions for kernel interaction), annotated with section markers like `SEC("xdp")`.
2. **Compile** with `clang -target bpf`, which produces architecture-independent **eBPF bytecode** in an ELF object file (often with BTF type info embedded for portability).
3. **Load** the bytecode into the kernel and **attach** it to a hook (XDP on an interface, a tc filter, a kprobe, etc.) — done via `bpftool`, libbpf, or a framework like Cilium.
4. **Verify:** the kernel's verifier statically checks every execution path, proving the program terminates and never makes unsafe memory accesses. If it can't prove safety, the load is rejected with an error.
5. **JIT-compile:** the kernel translates the verified bytecode into native machine instructions for the host CPU, so it runs at near-native speed rather than being interpreted.
6. **Run:** the program executes at its hook on every relevant event (every packet, every syscall, etc.) until it's detached.

The key stages that make eBPF special are **verification** (safety before execution) and **JIT** (speed). Compilation to a portable bytecode plus BTF is what enables "compile once, run on many kernel versions."
</details>

---

**Q3. What is the difference between an XDP program and a TC (sched_cls) program in terms of *where* in the packet path they run, and why might you choose one over the other?**

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**XDP** runs at the **earliest possible point** — in the NIC driver's receive path, *before* the kernel allocates a socket buffer (`skb`) for the packet. Because it acts before most of the networking stack and before any per-packet allocation, it's the **fastest** hook: ideal for dropping (DDoS mitigation), redirecting, or load-balancing packets at very high rates with minimal overhead. The trade-off is a more limited context — you have raw packet bytes but not the rich metadata the stack later attaches, and XDP is ingress-only.

**TC (sched_cls)** runs **later**, after the `skb` has been built, within the traffic-control datapath. It therefore has access to the **fuller packet context/metadata** (the skb, conntrack info in some setups, etc.) and works on **both ingress and egress**, but it's a bit slower than XDP because the skb already exists and more of the stack has run.

**Choosing:** use **XDP** when you need maximum performance and can act on raw packets early — high-volume drop/redirect/balancing. Use **TC** when you need the richer skb context, want to act on egress as well as ingress, or need to integrate with stack features that only exist after the skb is allocated. Many real systems (like Cilium) use *both*: XDP for the fast drop/redirect cases and TC for richer policy.
</details>

---

## Homework

Write and load the `XDP_PASS` program from the lab. Then modify it to count something trivial (you'll add a real map in the next lesson — for now, just experiment with returning different verdicts conditionally, e.g. `XDP_DROP` for packets and observe the effect with ping). Deliberately introduce a verifier violation (an unbounded loop, or reading past the packet without a bounds check) and capture the verifier's rejection message. Explain in your own words what the verifier objected to.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
**Verdict experiment:** changing the program to `return XDP_DROP` and attaching it to an interface causes *all* received packets on that interface to be silently dropped at the earliest point — a ping across it gets 100% loss, and nothing reaches the stack (you won't even see the packets in a higher-level capture, since XDP_DROP happens before that). Returning `XDP_PASS` restores normal behavior. This demonstrates XDP verdicts directly controlling packet fate at the driver.

**Verifier violations and what they mean:**

- **Out-of-bounds packet read:** if you access packet bytes (e.g., read the IP header) *without* first checking `if (data + offset > data_end) return XDP_PASS;`, the verifier rejects the program with a message like *"invalid access to packet, off=… size=… R_ pointer arithmetic … beyond packet end"*. The objection: the verifier can't prove your read stays within the actual packet bounds, so it could read kernel memory past the packet — a safety violation. You must add an explicit bounds check against `data_end` so the verifier can prove every access is in-range.

- **Unbounded loop:** a loop the verifier can't bound triggers something like *"back-edge from insn … to … , … unbounded loop"* or "too many instructions"/"infinite loop detected". The objection: it cannot prove the loop terminates, so it cannot guarantee the program halts in bounded time. The fix is to give the loop a compile-time-known bound (or `#pragma unroll`) so termination is provable.

In both cases the lesson is the same: the verifier refuses to load anything whose safety (memory-in-bounds, guaranteed termination) it cannot *prove* by static analysis. It's not catching a runtime crash — it's preventing the program from ever running until you've written it so the proof succeeds. That up-front proof obligation is the price (and the point) of being allowed to run code in the kernel datapath.
</details>
