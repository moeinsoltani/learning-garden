---
title: "Lesson 53 — Dependency Management"
nav_order: 4
parent: "Phase 10: Project Leadership"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 53: Dependency Management

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **critical path** — the chain of dependent work that sets the project's minimum duration; a slip on it slips everything (Lesson 03).
> - **gate** (verb) — to block progress until done ("this dependency gates the launch").
> - **dependency kickoff** — the early alignment conversation with a team you'll depend on, held *before* you're blocked.
> - **interface-first** — agreeing the API/contract up front so both teams build in **parallel** instead of waiting in sequence.
> - **mock / stub** — a fake stand-in for a not-yet-built component, letting you build against it.
> - **serialize** — to force work into one-after-another order (what dependencies do if unmanaged).
> - **buffer or decouple** — the two defenses: plan slack around a risky dependency, or remove its power to block you.
> - **blindsided** — surprised by something you should have seen coming (Lesson 36/41).
> - **early-warning signs** — the small signals that a promise is slipping, visible before the miss.
> - **track like your own** — monitoring others' commitments as actively as your own tasks.

## Concept

Dependencies — on other teams, external services, or things outside your control — are one of the most
common reasons projects slip, precisely because they're *not in your hands.* Managing them well is about
**not being blindsided by them**: mapping the critical path, aligning early (before you're blocked),
designing interfaces so teams can work in parallel, tracking others' promises like your own tasks, and
building in buffers or decoupling. This complements Lesson 43 (getting things from other teams) — here the
focus is the *project-management* discipline of coordinating dependencies.

```
   DEPENDENCY MANAGEMENT (don't let them sink you)
   ┌──────────────────────────────────────────────────────┐
   │ • MAP the CRITICAL PATH (which dependencies gate the    │
   │   whole project?)                                       │
   │ • DEPENDENCY KICKOFFS (align BEFORE you're blocked)     │
   │ • INTERFACE-FIRST contracts → teams work in PARALLEL    │
   │ • track others' PROMISES like your own tasks            │
   │ • BUFFER or DECOUPLE (reduce the dependency's risk)     │
   └──────────────────────────────────────────────────────┘
```

The reframe: **manage dependencies proactively — map them, align early, interface-first, track them, and
buffer/decouple — so they don't blindside you.** The failure mode is passive: assume the dependency will be
there when you need it, then discover (when you're blocked) that it's late or wrong — and slip. Proactive
management (surfacing the critical dependencies, aligning before you're blocked, agreeing interfaces so you
can work in parallel, tracking others' commitments actively, and reducing the risk via buffer/decoupling) is
what keeps dependencies from being the reason you slip.

---

## How It Works

### Map the critical path

First, understand which dependencies **gate the project** — the **critical path.** Not all dependencies are
equal: some are on the critical path (the project can't finish until they're done — they gate everything),
others have slack (they're needed but not blocking). Map which dependencies are critical-path (a delay in
them delays the whole project) so you focus your management on those. The critical path is the sequence of
dependent work that determines the minimum project duration — a slip anywhere on it slips the project, so
critical-path dependencies get the most attention and risk-management.

### Dependency kickoffs — align before you're blocked

The key proactive move: **align on dependencies early — before you're blocked by them**, not when you hit
the block. A **dependency kickoff** (early conversation with the team/provider you depend on) establishes:
what you need, when, the interface, and their commitment — <em>at the start</em>, so there's time to
resolve issues, and so they can plan for it. The failure is passive: assume the dependency will be ready
when you need it, then discover at the moment you're blocked that they didn't know, can't deliver in time,
or built the wrong thing. Aligning early (dependency kickoff) surfaces problems when there's still time to
address them (renegotiate, find alternatives, adjust the plan), versus discovering them when you're already
blocked (too late).

### Interface-first contracts — work in parallel

A powerful technique: **agree the interface first, so teams can work in parallel.** If team A depends on
team B's API, don't wait for B to build it before A starts — agree the <strong>interface</strong> (the API
contract, the data format) up front, then both teams can work simultaneously (A builds against the agreed
interface, e.g., with a mock/stub; B builds to fulfill it). This decouples the work in time — parallel
instead of sequential — dramatically reducing the schedule impact of the dependency (you're not waiting).
Interface-first (defining the contract early, then building on both sides in parallel) is a core technique
for keeping dependencies from serializing your project.

### Track others' promises like your own tasks

A discipline: **track external commitments (what other teams promised) as actively as your own tasks.**
People naturally track their own work but assume others' promises will just happen — and then are surprised
when they don't. Treat a dependency's commitment ("team B will deliver the API by week 4") as a tracked item
you actively monitor: check in on its progress, watch for early signs of slip, and don't assume it's on
track just because it was promised. Others' commitments slip (they have their own pressures), so tracking
them like your own tasks (with check-ins and early-warning) catches slips early rather than at the moment
you need the thing.

### The buffer-or-decouple decision

For a risky dependency (one that might slip and hurt you), you have two main risk-reducing options: (1)
**buffer** — build slack around the dependency (assume it might be late, and plan buffer so a slip doesn't
immediately sink you); or (2) **decouple** — reduce or remove the dependency's ability to block you (build
against a mock so you're not waiting, find an alternative, do a simpler version yourself, or restructure so
you don't need it on the critical path). Decoupling (removing the dependency's blocking power) is often the
stronger move where possible — a dependency that can't block you can't sink your project. Where you can't
decouple, buffer (so a slip is absorbed). The buffer-or-decouple decision is how you actively reduce a
risky dependency's threat rather than just hoping it delivers.

{: .note }
> **Manage dependencies proactively — map, align early, interface-first, track, buffer/decouple</br>**
> Dependencies (on other teams, external things) are a top reason projects slip, because they're not in your
> hands — so manage them proactively rather than being blindsided. <em>Map the critical path</em> (which
> dependencies gate the whole project — focus there). Do <em>dependency kickoffs</em> (align on what/when/
> interface early, before you're blocked, so problems surface with time to fix them). Use <em>interface-first
> contracts</em> (agree the API up front so teams work in parallel, not serially — decoupling the work in
> time). <em>Track others' promises like your own tasks</em> (actively monitor, don't assume they'll just
> happen). And <em>buffer or decouple</em> risky dependencies (build slack, or remove their power to block
> you — decoupling is often stronger). Proactive dependency management is what keeps dependencies from being
> the reason your project slips.

---

## Lab — Scenario

**The situation:** You're leading an **8-week project** that depends on **three other teams**: the platform
team (needs to provide an API change by ~week 4), the data team (needs to set up a data pipeline by ~week
5), and the design team (needs final designs by ~week 2). Any of these slipping could sink your timeline.
You want to set up the coordination structure so these dependencies don't blindside you.

**Design the coordination structure** — what's written down, who syncs when, and the early-warning
tripwires. Then note the principles and mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong coordination structure maps the dependencies, aligns early (kickoffs), writes down contracts,
enables parallel work (interface-first), tracks the promises, and sets early-warning tripwires. Example:
<br><br>
<strong>Map the critical path:</strong> "First, which dependencies gate the project? Design (week 2) gates
a lot of the build; the platform API (week 4) and data pipeline (week 5) gate later integration. I'll map
which of my work depends on each and identify the critical path — likely design early (blocks build), then
platform/data mid-project (block integration). Focus most attention on the critical-path ones."
<br><br>
<strong>Dependency kickoffs (align early — week 0/1):</strong> "Kick off with each team <em>now</em>, before
I'm blocked: confirm what I need, exactly when, the interface, and get their explicit commitment — and make
sure they've planned for it (not hearing about it for the first time when I'm blocked in week 4). Surface any
problems now (if the platform team says 'week 4 is tight,' I learn it in week 1, not week 4)."
<br><br>
<strong>What's written down (contracts):</strong> "For each dependency, a written contract (Lesson 43): what
they'll deliver, by when, and the exact interface. Especially: <strong>agree the interfaces first</strong> —
the platform API's contract and the data pipeline's format — up front, so my team can build against them (with
mocks/stubs) in parallel, rather than waiting for the real thing. Interface-first decouples the timing."
<br><br>
<strong>Who syncs when:</strong> "A lightweight sync cadence: a brief weekly check-in with each dependency
team (or a shared channel/standup) — are they on track for their date? any risks emerging? I track each
team's commitment like my own task (an item I actively monitor), not assume it'll just happen."
<br><br>
<strong>Early-warning tripwires:</strong> "Define tripwires that signal a slip <em>early</em>: e.g., 'if the
platform API isn't started by week 2, it's at risk for week 4'; 'if design isn't in review by week 1.5, week
2 is at risk.' These intermediate checkpoints catch a slip before the deadline (when I can still respond —
escalate, adjust, decouple). Also: for the riskiest dependency, have a fallback (a mock to keep working, a
simpler alternative) so a slip doesn't immediately block me — buffer or decouple."
<br><br>
<strong>Buffer/decouple the risky ones:</strong> "Interface-first mocks decouple most of the timing risk
(I'm not idle waiting). Where I can't decouple, build in buffer (don't schedule my dependent work for the
exact day the dependency is due — assume some slip). And identify which dependency is riskiest (e.g., the
platform team with a full roadmap — Lesson 43) and give it extra attention/escalation-readiness."
<br><br>
<strong>Principles:</strong> (1) <strong>Map the critical path</strong> — focus on the dependencies that gate
the project. (2) <strong>Dependency kickoffs early</strong> — align on what/when/interface before you're
blocked, surfacing problems with time to fix. (3) <strong>Written contracts + interface-first</strong> — what/
when/interface in writing, and agree interfaces up front so teams work in parallel (with mocks). (4)
<strong>Track promises like your own tasks</strong> — active weekly check-ins, not assuming they'll happen.
(5) <strong>Early-warning tripwires</strong> — intermediate checkpoints that signal a slip early (when you
can respond). (6) <strong>Buffer/decouple</strong> — mocks decouple timing, buffer absorbs slips, extra
attention on the riskiest. <strong>Mistakes to avoid:</strong> (1) <strong>passive dependency management</strong>
— assuming the dependencies will be ready when needed, discovering slips when blocked (too late); (2)
<strong>no early kickoff</strong> — not aligning until you hit the block (missing the chance to surface
problems early); (3) <strong>sequential work</strong> — waiting for each dependency before starting (not
interface-first/parallel), serializing the project; (4) <strong>not tracking others' promises</strong> —
assuming they'll just happen, then surprised by a slip; (5) <strong>no tripwires</strong> — only finding out
about a slip at the due date (no early warning); (6) <strong>no buffer/decouple</strong> — scheduling
dependent work for the exact due date with no fallback (a slip immediately sinks you); (7) <strong>nothing
written down</strong> — vague verbal dependencies (Lesson 43) that get forgotten/misinterpreted.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Critical path & dependencies | project management (CPM); *The Mythical Man-Month* |
| Getting things from other teams | Lesson 43 (cross-team dependencies) |
| Interface-first / parallel work | contract-first API design; Team Topologies |
| Buffers & decoupling | *Critical Chain*, Goldratt; slack buffers |

---

## Checkpoint

**Q1.** Why do "dependency kickoffs" (aligning early) and "interface-first contracts" help keep dependencies
from slipping your project?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Dependency kickoffs</strong> (aligning early — before you're blocked) help because <strong>they
surface dependency problems when there's still time to address them, rather than discovering them at the
moment you're blocked, when it's too late</strong>. The failure mode with dependencies is passive: you assume
the dependency will be ready when you need it, do your own work, and then — when you reach the point where you
need the dependency — discover a problem (they didn't know you needed it, they can't deliver in time, they
built the wrong thing, they deprioritized it). At that point you're <em>already blocked</em>, with no time to
resolve it (the dependency is needed now and isn't there), so your project slips. A <strong>dependency
kickoff</strong> — an early conversation with the team/provider you depend on, at the <em>start</em> of the
project — prevents this by aligning early: you establish what you need, when, the interface, and get their
explicit commitment, <em>before</em> you're blocked. This surfaces problems early: if the platform team says
"week 4 is tight" or "we didn't know you needed this" or "that interface won't work," you learn it in week 1
(when you can respond — renegotiate, find alternatives, adjust your plan, escalate) rather than in week 4
(when you're blocked and out of time). It also lets them <em>plan for it</em> (they know your need is coming
and can schedule it), rather than being surprised by a demand when you hit the block. So aligning early via a
dependency kickoff converts the passive "assume it'll be there, discover the problem when blocked" (which
causes slips) into proactive "surface any issues early, when there's time to address them" — which is how you
keep a dependency from blindsiding you. The principle: surface dependency problems early (kickoff), when you
can still respond, rather than at the moment of blocking (too late). <strong>Interface-first contracts</strong>
help because <strong>agreeing the interface up front lets the dependent teams work in parallel rather than
sequentially — dramatically reducing the schedule impact of the dependency by decoupling the work in
time</strong>. Without interface-first, dependencies serialize work: if team A depends on team B's API, A
waits for B to build the API before A can build against it — so A's work happens <em>after</em> B's,
sequentially, and any delay in B directly delays A (the dependency is on A's critical path, and A is idle
waiting). With <strong>interface-first</strong>, you agree the <em>interface</em> (the API contract, the data
format) up front, then <em>both teams work simultaneously</em>: A builds against the agreed interface (using
a mock/stub that implements it) while B builds the real implementation to fulfill the same interface. This
<strong>decouples the work in time</strong> — A and B work in parallel instead of A waiting for B — which:
(1) <strong>removes the waiting</strong> — A isn't idle waiting for B (A builds against the mock), so the
dependency doesn't serialize the schedule; (2) <strong>reduces the slip impact</strong> — if B slips, A has
still been progressing against the interface (only the final integration waits for B, not all of A's work),
so B's slip has far less schedule impact; (3) <strong>parallelizes the project</strong> — the total time is
closer to max(A, B) than A + B. So interface-first is a powerful technique for keeping dependencies from
serializing (and thus dominating) your schedule: by defining the contract early and building on both sides in
parallel (against the agreed interface), you decouple the timing so the dependency doesn't force you to wait
and doesn't directly delay you if it slips. Both points reduce the schedule risk of dependencies: dependency
kickoffs surface problems early (so you can respond before being blocked, rather than discovering issues too
late), and interface-first contracts decouple the timing (so teams work in parallel and a dependency slip has
less impact, rather than serializing and directly delaying you) — together making dependencies proactively
managed (aligned early, worked in parallel) rather than passive blindsides that slip your project. They
embody the proactive-dependency-management principle: don't wait to be blocked (align early, work in
parallel), so dependencies don't become the reason you slip.
</details>

**Q2.** Why track others' promises "like your own tasks," and what is the buffer-or-decouple decision?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You should track others' promises "like your own tasks" because <strong>people naturally track their own
work but passively assume others' promises will just happen — and then are surprised when they slip; actively
tracking external commitments catches their slips early rather than at the moment you need the thing</strong>.
There's a natural asymmetry in attention: you actively monitor your own tasks (their progress, their risks,
whether they're on track), but you tend to treat others' commitments as done deals ("team B promised the API
by week 4, so it'll be there") and not monitor them — you assume the promise will be fulfilled and move on.
But others' commitments slip just like yours do (they have their own pressures, priorities, surprises, and
optimistic estimates), so <em>assuming</em> a promise will happen is exactly how you get blindsided: you don't
watch it, it slips (unbeknownst to you), and you discover the slip when you reach the point of needing it (too
late). Tracking a dependency's commitment <strong>like your own task</strong> — as an item you actively
monitor, with check-ins on its progress, watching for early signs of slip, not assuming it's on track just
because it was promised — catches slips <em>early</em>: if you're checking in weekly on "team B's API,
on track for week 4?", you'll notice in week 2 if it's at risk (when you can respond — escalate, adjust,
decouple), rather than discovering in week 4 that it's not ready (when you're blocked). So actively tracking
others' promises (rather than passively assuming them) applies to dependencies the same monitoring you give
your own work, which is what surfaces their slips early enough to respond. The principle: don't assume others'
commitments will happen — monitor them like your own tasks, because they slip too, and early detection is
what lets you respond. The <strong>buffer-or-decouple decision</strong> is the choice, for a risky dependency
(one that might slip and hurt you), between two risk-reducing strategies: (1) <strong>buffer</strong> — build
slack around the dependency: assume it might be late and plan buffer (extra time) so that a slip doesn't
immediately sink you (the buffer absorbs the slip). You don't remove the dependency's risk, but you cushion
its impact (a slip within the buffer doesn't blow the project). (2) <strong>decouple</strong> — reduce or
remove the dependency's ability to block you: build against a mock/stub so you're not waiting for it (interface-
first), find an alternative, do a simpler version yourself, or restructure the project so the dependency isn't
on your critical path. Decoupling removes or reduces the dependency's <em>power to block you</em>, so even if
it slips, it doesn't stop your progress. The decision between them: <strong>decoupling is often the stronger
move where possible</strong>, because a dependency that <em>can't block you</em> (decoupled) can't sink your
project at all — you've eliminated the risk, not just cushioned it. Where you can't fully decouple, <strong>buffer</strong>
(so a slip is absorbed by slack rather than immediately blowing the timeline). So for each risky dependency,
you decide: can I decouple it (remove its blocking power — build against a mock, find an alternative,
restructure)? If yes, do that (strongest). If not, buffer it (build in slack to absorb a slip). The buffer-or-
decouple decision is how you <em>actively reduce a risky dependency's threat</em> — rather than passively
hoping it delivers on time (which leaves you fully exposed to its slip). It turns a dependency from a passive
risk (it might slip and sink you) into a managed one (you've either removed its blocking power or cushioned
its impact). Both points reflect proactively reducing dependency risk: track others' promises actively (like
your own tasks, so you catch slips early rather than assuming they'll happen and being blindsided), and
buffer-or-decouple risky dependencies (actively reduce their threat — decouple to remove blocking power, or
buffer to absorb slips — rather than passively hoping) — together keeping dependencies from being the reason
your project slips, by managing them actively (monitored, de-risked) rather than passively (assumed, exposed).
</details>

---

## Homework

Improve how you manage dependencies. (1) For a project with dependencies, map the critical path — which
dependencies gate the whole project? (2) Have you done dependency kickoffs (aligned early on what/when/
interface), or are you assuming they'll be there? Align early on a current dependency. (3) Can you use
interface-first contracts to let teams work in parallel (rather than waiting)? (4) Are you tracking others'
promises like your own tasks, with early-warning tripwires? (5) For a risky dependency, decide: buffer or
decouple? Reflect: are dependencies a frequent reason your projects slip, and what's one dependency you should
align on earlier, decouple, or track more actively?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds dependency management — keeping dependencies from slipping projects. A strong response
maps the critical path, does dependency kickoffs (aligning early), uses interface-first contracts for parallel
work, tracks others' promises actively with tripwires, and makes buffer-or-decouple decisions for risky
dependencies. The realizations: (1) <strong>map the critical path</strong> — focus dependency management on
the ones that gate the whole project; (2) <strong>align early (dependency kickoffs)</strong> — surface
problems before you're blocked (when there's time to respond), rather than discovering them at the block (too
late); (3) <strong>interface-first contracts enable parallel work</strong> — agreeing the interface up front
lets teams work simultaneously (against mocks) instead of serially, decoupling the timing and reducing slip
impact; (4) <strong>track others' promises like your own tasks</strong> — people passively assume others'
commitments will happen and get blindsided; active monitoring with tripwires catches slips early; (5)
<strong>buffer or decouple risky dependencies</strong> — decouple (remove blocking power) is stronger where
possible; buffer (absorb slips) where you can't. On reflection, people commonly find they manage dependencies
passively (assuming they'll be there, discovering slips when blocked), work sequentially rather than interface-
first/parallel, and don't track others' promises actively. The highest-value change is usually: align early
(dependency kickoffs), use interface-first to work in parallel, and track dependencies' commitments like your
own tasks with early-warning tripwires. The meta-point: dependencies are a top reason projects slip precisely
because they're not in your hands — so managing them well is about not being blindsided: mapping the critical
path, aligning early (before you're blocked), interface-first (parallel work), tracking promises actively, and
buffering/decoupling to reduce risk. Proactive dependency management (surfacing, aligning, parallelizing,
tracking, de-risking) keeps dependencies from being the reason you slip — versus passive management (assume,
then be surprised). This is a major quiet risk (Lesson 52) and complements getting things from other teams
(Lesson 43 — the influence side; this is the project-coordination side). The next lesson turns to
prioritization — saying no (or "not now") all day while staying trusted, and handling the constant stream of
competing demands.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 54 — Prioritization and Trade-offs →](lesson-54-prioritization){: .btn .btn-primary }
