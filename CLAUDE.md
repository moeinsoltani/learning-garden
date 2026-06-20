# CLAUDE.md — Linux Systems Learning Project

This repo hosts **two independent learning tracks** on one Just the Docs site:
- **Networking** — in the `_networking/` collection
- **Virtualization (QEMU/KVM)** — in the `_virtualization/` collection

## Student Profile
- Experience level: very limited (beginner)
- Platform: Windows 11, with access to a Linux VM and WSL2
- Use the Linux VM for labs (WSL2 as fallback for early lessons)

## Repository Structure
The site is split into two Jekyll collections, each its own nav section:
```
index.md                         # landing page / track chooser (nav_order 1, site root)
_config.yml                      # defines both collections (see Site Theme)
_networking/
  learning-plan.md               # networking curriculum
  lessons/                       # phase-NN-*.md parents + lesson-NN-*.md
_virtualization/
  learning-plan.md               # virtualization curriculum
  lessons/                       # (created as virtualization lessons are written)
```
- `<track>` below means `networking` or `virtualization`.
- A page's URL is `/<track>/lessons/<file>.html` (collection permalink keeps the
  `lessons/` path so relative `(lesson-NN-...)` links between phase and lesson
  pages resolve).
- Lessons for a track live under that track's collection only.

## Teaching Methodology
For each lesson:
1. Create a lesson file at `_<track>/lessons/lesson-NN-topic.md` before teaching
2. The file contains: mental model, mechanics, lab commands with expected output, and checkpoint questions with hidden answers
3. Student reads the file and asks questions in chat if needed
4. Student moves to the next lesson at their own pace — no checkpoint gate
5. At the end of each Phase, give a written test covering all lessons in that phase

## Site Theme
- Theme: **Just the Docs** (dark color scheme) via `remote_theme: just-the-docs/just-the-docs@v0.10.0`
- Config: `_config.yml` — defines the `networking` and `virtualization` collections
  and their nav names under `just_the_docs.collections`. Do not change the theme or
  collection setup without updating this file.
- Phase parent pages live in `_<track>/lessons/phase-NN-name.md` with `has_children: true`
- Lesson files live in `_<track>/lessons/lesson-NN-topic.md` with `parent: "Phase N: Name"`
- `parent`/`nav_order`/`has_children` nest pages **within their own collection** — a
  networking lesson's `parent` must match a networking phase page title, and likewise
  for virtualization. The two collections never share parents.

## Lesson File Format
Each lesson file must have this exact front matter:
```yaml
---
title: "Lesson NN — Title"
nav_order: N
parent: "Phase N: Name"
---
```

Then immediately after front matter:
- **Home button**: `[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }` (Just the Docs button style, uses Jekyll relative_url so it works under /learn-networking/ baseurl)
- **Concept** — mental model, analogy, ASCII diagram
- **How it works** — mechanics (brief, focused on "why")
- **Lab** — exact commands with expected output (`$` = run this, `#` = comment)
- **Checkpoint** — each question has:
  1. `**Your answer:**` blank field for the student
  2. A hidden model answer: `<details><summary>Show Answer</summary><br>Answer here.</details>`
- **Homework** — one harder task with its own hidden model answer

Model answers must always be written — never leave a checkpoint or homework without one.

## Handling Student Questions
When the student asks a question about a term or concept from a lesson:
1. Answer briefly in chat
2. Then update the lesson file to incorporate the answer — either as a `{: .note }` callout block inline with the relevant section, or as expanded text
3. Add reference links (Wikipedia, man7.org man pages) for any technical term introduced — inline on first mention AND in the lesson's **Further Reading** table
4. Every lesson must have a **Further Reading** table before the Checkpoint section

### Callout syntax (Just the Docs)
```markdown
{: .note }
> **Title**
> Body text here.
```

### Reference link sources to prefer
- Linux man pages: `https://man7.org/linux/man-pages/manN/name.N.html`
- Wikipedia: `https://en.wikipedia.org/wiki/Topic`
- Only include links you are certain exist — do not guess URLs

## Curriculum
Each track has its own plan; **always consult the relevant one before creating a new lesson** — it specifies the topic, goals, and key commands for every lesson and phase. Never invent a lesson topic; follow the plan in order.
- Networking: `_networking/learning-plan.md`
- Virtualization: `_virtualization/learning-plan.md`

## Networking Lesson Index
- Lesson 01: `_networking/lessons/lesson-01-namespaces-intro.md` — What a network namespace is ✓
- Lesson 02: `_networking/lessons/lesson-02-host-as-namespace.md` — The host is just another namespace ✓
- Lesson 03: `_networking/lessons/lesson-03-build-namespaces.md` — Build two isolated namespaces ✓
- Lesson 04: `_networking/lessons/lesson-04-ip-addressing.md` — IP addressing fundamentals (addrs) ✓
- Lesson 05: `_networking/lessons/lesson-05-arp-neigh.md` — Neighbor tables & ARP (neigh) ✓
- Lesson 06: `_networking/lessons/lesson-06-tcpdump.md` — tcpdump, your constant companion ✓
- Lesson 07: `_networking/lessons/lesson-07-veth-pairs.md` — veth pairs ✓
- Lesson 08: `_networking/lessons/lesson-08-loopback.md` — Loopback interface ✓
- Lesson 09: `_networking/lessons/lesson-09-bonding.md` — Bond interfaces ✓
- Lesson 10: `_networking/lessons/lesson-10-dummy.md` — Dummy interfaces ✓
- Lesson 11: `_networking/lessons/lesson-11-bridges.md` — Linux bridges ✓
- Lesson 12: `_networking/lessons/lesson-12-vlans.md` — VLAN interfaces ✓
- Lesson 13: `_networking/lessons/lesson-13-macvlan.md` — MACVLAN ✓
- Lesson 14: `_networking/lessons/lesson-14-tap.md` — TAP interfaces ✓
- Lesson 15: `_networking/lessons/lesson-15-vxlan.md` — VXLAN ✓
- Lesson 16: `_networking/lessons/lesson-16-routing-fundamentals.md` — Routing fundamentals (routes) ✓
- Lesson 17: `_networking/lessons/lesson-17-routing-namespaces.md` — Routing between namespaces ✓
- Lesson 18: `_networking/lessons/lesson-18-policy-routing.md` — Policy routing & multiple tables ✓
- Lesson 19: `_networking/lessons/lesson-19-frr-intro.md` — FRR introduction ✓
- Lesson 20: `_networking/lessons/lesson-20-ospf.md` — OSPF with FRR ✓
- Lesson 21: `_networking/lessons/lesson-21-bgp.md` — BGP with FRR ✓
- Lesson 22: `_networking/lessons/lesson-22-nat.md` — NAT with nftables ✓
- Lesson 23: `_networking/lessons/lesson-23-conntrack.md` — Conntrack ✓
- Lesson 24: `_networking/lessons/lesson-24-dhcp.md` — DHCP ✓
- Lesson 25: `_networking/lessons/lesson-25-nftables-arch.md` — nftables architecture ✓
- Lesson 26: `_networking/lessons/lesson-26-nft-filtering.md` — Packet filtering with nft ✓
- Lesson 27: `_networking/lessons/lesson-27-nft-sets.md` — nftables sets & verdict maps ✓
- Lesson 28: `_networking/lessons/lesson-28-flowtable.md` — nftables flowtable (offload) ✓
- Lesson 29: `_networking/lessons/lesson-29-tc-model.md` — Traffic control model ✓
- Lesson 30: `_networking/lessons/lesson-30-classless-qdisc.md` — Classless qdiscs (shaping/netem) ✓
- Lesson 31: `_networking/lessons/lesson-31-htb.md` — Classful qdiscs — HTB ✓
- Lesson 32: `_networking/lessons/lesson-32-tc-filters.md` — tc filters ✓
- Lesson 33: `_networking/lessons/lesson-33-bandwidth.md` — Bandwidth measurement ✓
- Lesson 34: `_networking/lessons/lesson-34-sysctl-tunables.md` — Network sysctl tunables ✓
- Lesson 35: `_networking/lessons/lesson-35-sysctl-persistence.md` — sysctl persistence ✓
- Lesson 36: `_networking/lessons/lesson-36-ebpf-fundamentals.md` — eBPF fundamentals ✓
- Lesson 37: `_networking/lessons/lesson-37-bpf-maps.md` — BPF maps ✓
- Lesson 38: `_networking/lessons/lesson-38-xdp.md` — XDP, Express Data Path ✓
- Lesson 39: `_networking/lessons/lesson-39-nftbpf.md` — nftbpf: calling BPF from nftables ✓
- Lesson 40: `_networking/lessons/lesson-40-systemd-networkd.md` — systemd-networkd ✓
- Lesson 41: `_networking/lessons/lesson-41-persistence-alternatives.md` — Other persistence approaches ✓
- Lesson 42: `_networking/lessons/lesson-42-docker-networking.md` — Docker networking from first principles ✓
- Lesson 43: `_networking/lessons/lesson-43-kubernetes-networking.md` — Kubernetes networking ✓
- Lesson 44: `_networking/lessons/lesson-44-flannel-vxlan.md` — VXLAN-based CNI (Flannel) ✓
- Lesson 45: `_networking/lessons/lesson-45-tcpdump-mastery.md` — tcpdump mastery ✓
- Lesson 46: `_networking/lessons/lesson-46-diagnostic-toolchain.md` — Full diagnostic toolchain ✓
- Lesson 47: `_networking/lessons/lesson-47-debugging-methodology.md` — Network debugging methodology ✓

*(Networking track complete — update this index if lessons change)*

## Virtualization Lesson Index
*(No lessons implemented yet. Follow `_virtualization/learning-plan.md` in order;
create files at `_virtualization/lessons/lesson-NN-topic.md` with phase parent pages
`_virtualization/lessons/phase-NN-name.md`. Update this index as lessons are created.)*
