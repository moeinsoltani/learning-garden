---
title: "Lesson 62 — Org Design and Team Topologies"
nav_order: 6
parent: "Phase 11: People Management (the EM path)"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 62: Org Design and Team Topologies

## Concept

How you structure teams — their boundaries, ownership, and interactions — profoundly shapes what they can
build and how well: **"structure eats process for breakfast."** The key insights: **Conway's Law** (your
system's architecture mirrors your org's communication structure — so you can design team boundaries to get
the architecture you want), team **cognitive load** limits (a team can only own so much before it drowns),
and the **Team Topologies** patterns (stream-aligned, platform, enabling teams). Designing team boundaries
deliberately is a powerful, underused leadership lever.

```
   ORG DESIGN (structure shapes everything)
   ┌──────────────────────────────────────────────────────┐
   │ CONWAY'S LAW: system architecture mirrors team/comms    │
   │   structure → design teams to get the architecture     │
   │   you want (the "inverse Conway maneuver")             │
   │ COGNITIVE LOAD: a team can only own so much (too much   │
   │   = drowning); size teams & scope to fit                │
   │ TEAM TOPOLOGIES: stream-aligned · platform · enabling   │
   │ REORGS: real cost — do them when needed, not casually   │
   └──────────────────────────────────────────────────────┘
```

The reframe: **team structure is a deliberate design choice that shapes the architecture and the team's
effectiveness — design boundaries intentionally (Conway's Law, cognitive load), don't let them just
happen.** Most people treat team structure as fixed or accidental, but it's a powerful lever: because
architecture mirrors org structure (Conway), you can shape the system by shaping the teams; and because
teams have cognitive-load limits, right-sizing their ownership determines whether they thrive or drown.
Structure is a design problem, and it eats process for breakfast (the best process can't overcome bad
structure).

---

## How It Works

### Conway's Law — and using it on purpose

**Conway's Law**: "organizations design systems that mirror their own communication structure" — i.e., your
software architecture ends up reflecting your org's team and communication boundaries (teams build systems
shaped like the org). This is a powerful insight because it works both ways: since architecture mirrors org
structure, **you can design your team boundaries to get the architecture you want** — the "**inverse Conway
maneuver**" (structure teams around the desired architecture, and the architecture will follow). If you want
loosely-coupled services with clear boundaries, structure teams that way (each owning a service, clear
interfaces); if teams are tangled and overlapping, the architecture will be too. So team boundaries are an
architectural lever — design them intentionally to shape the system, rather than letting an accidental org
structure impose an accidental (often bad) architecture.

### Cognitive load — a team can only own so much

A crucial constraint (from *Team Topologies*): a team has a limited **cognitive load** — it can only
understand and effectively own so much (so many services, domains, technologies) before it's overwhelmed and
its effectiveness collapses. A team owning too much (too many disparate systems, too broad a domain) drowns:
they can't hold it all in their heads, quality and velocity suffer, and they're perpetually firefighting.
So **size a team's ownership to fit its cognitive load** — scope teams so their responsibilities are
something they can genuinely master, and when a team's ownership exceeds its capacity, that's a signal to
split (give some ownership to another team). Cognitive load is a real limit that should drive team scoping —
a team drowning in too much is an org-design problem, not a work-harder problem.

### Team Topologies — the patterns

*Team Topologies* (Skelton & Pais) offers a useful vocabulary of team types: (1) **Stream-aligned teams** —
teams aligned to a flow of work / a product or domain area (the primary team type — they own and deliver a
stream of value end-to-end); (2) **Platform teams** — teams that provide internal platforms/services that
make stream-aligned teams more effective (reducing their cognitive load by providing self-service
capabilities — Lesson 43's platform-as-a-product); (3) **Enabling teams** — teams that help other teams
adopt new skills/practices (temporary uplift, coaching); (4) **Complicated-subsystem teams** — teams owning
a specialized, complex subsystem needing deep expertise. The value is the vocabulary and the principle:
different teams have different purposes and interaction modes, and designing the right mix (mostly stream-
aligned teams, supported by platform/enabling teams) with clear interactions is how you structure an
effective engineering org.

### Reorgs — real cost, do them when needed

**Reorganizations** (changing team structures) are sometimes necessary (the current structure doesn't fit
the work, a team is drowning, boundaries are wrong) — but they have **real costs**: disruption (people
re-form relationships, re-learn context, lose momentum), uncertainty and anxiety (reorgs are unsettling),
and lost productivity during the transition. So reorgs shouldn't be done casually or frequently (constant
reorgs are hugely disruptive and signal thrashing) — do them when there's a genuine structural problem that
warrants the cost, do them thoughtfully (clear rationale, good communication, minimizing disruption), and
not more than needed. Weigh the real cost of a reorg against the benefit; a reorg is a significant
intervention, not a casual reshuffling.

### Splitting a team well

A common org-design task: **splitting a team that's grown too big or owns too much.** Do it well: (1)
**split along good boundaries** — cohesive ownership (each new team owns a coherent domain/set of services
with clear interfaces — Conway/cognitive-load), not arbitrary lines that cut through tightly-coupled work;
(2) **staff each team viably** — each needs the skills and size to own its area (not one team left
non-viable); (3) **migrate ownership cleanly** — clear handoff of what each team now owns (no orphaned or
contested areas); (4) **communicate the rationale and the change well** (people need to understand why and
what it means for them). A good split gives each team a coherent, right-sized, well-staffed ownership with
clear boundaries; a bad split creates confusion, orphaned work, and non-viable teams.

{: .note }
> **Design team boundaries deliberately — structure eats process for breakfast</br>**
> How you structure teams profoundly shapes what they build and how well — structure eats process for
> breakfast (bad structure defeats good process). Use <em>Conway's Law</em> deliberately: architecture
> mirrors org/communication structure, so design team boundaries to get the architecture you want (the
> inverse Conway maneuver). Respect <em>cognitive load</em>: a team can only own so much before it drowns, so
> size ownership to fit (too much = split). Use the <em>Team Topologies</em> patterns (stream-aligned,
> platform, enabling, complicated-subsystem) to design the right mix with clear interactions. Do <em>reorgs</em>
> only when a genuine structural problem warrants the real cost (not casually). And <em>split teams well</em>
> (good boundaries, viable staffing, clean ownership migration, clear communication). Team structure is a
> deliberate design lever — one of the most powerful and underused — not something to let happen accidentally.

---

## Lab — Scenario

**The situation:** Your **12-person team** has grown to own **too much** — a sprawl of services and domains
that no one fully understands, constant firefighting, slow delivery, and the team is drowning under the
cognitive load. It's clearly too big and owns too broad a scope. You need to **split it** into smaller,
focused teams.

**Design the split** — the boundaries, staffing, ownership migration, and the announcement. Then note the
principles and mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong split defines cohesive boundaries (Conway/cognitive-load), staffs each team viably, migrates
ownership cleanly, and communicates well. Example:
<br><br>
<strong>Define the boundaries (cohesive ownership, right cognitive load):</strong> "First, map what the team
owns and group it into <strong>cohesive domains</strong> — clusters of services/work that belong together
(tightly coupled, shared context) with clear interfaces between clusters. Split along those natural
boundaries (not arbitrary lines through coupled work). E.g., if the 12-person team owns [domain A:
checkout/payments], [domain B: catalog/search], [domain C: platform/infra], split into three teams each
owning one coherent domain — each with a cognitive load a ~4-person team can actually master. The boundaries
follow Conway (design teams around the architecture you want — clean domain boundaries → cleaner
architecture) and cognitive load (each team owns something masterable)."
<br><br>
<strong>Staff each team viably:</strong> "Each new team needs the skills and size to own its area viably: ~4
people each, with the right mix of skills (not one team missing critical expertise), and ideally a clear tech
lead. Distribute people thoughtfully — keep domain knowledge with the relevant domain (the people who know
checkout go to the checkout team), balance seniority, and ensure no team is left non-viable (too small or
missing key skills). Watch bus-factor (Lesson 34) — don't leave a domain with only one person who knows it."
<br><br>
<strong>Migrate ownership cleanly:</strong> "Clear handoff of what each team now owns — an explicit ownership
map (team A owns X, team B owns Y), no orphaned areas (everything has an owner) and no contested/overlapping
ownership (clear boundaries). Handle the interfaces between teams (the APIs/contracts where domains meet —
Lesson 43). Transition the on-call/support ownership too. Give it a transition period with support."
<br><br>
<strong>The announcement:</strong> "Communicate clearly and reassuringly: <strong>why</strong> (the team grew
too big and owns too much — drowning under the load, slow delivery; splitting into focused teams so each can
master its area and move faster — frame it positively, as enabling the team, not as a problem/punishment);
<strong>what</strong> (the new team structure, who's on each, what each owns); <strong>what it means for
people</strong> (address the uncertainty — where they'll be, that it's about focus not performance, their
growth); and <strong>the transition plan</strong>. Reorgs are unsettling, so over-communicate the rationale
and reassure people." <em>(Clear why/what/impact/plan — addressing the anxiety a reorg causes.)</em>
<br><br>
<strong>Principles:</strong> (1) <strong>Cohesive boundaries (Conway + cognitive load)</strong> — split along
natural domain boundaries (coupled work stays together, clear interfaces between), each team owning a
masterable scope. (2) <strong>Viable staffing</strong> — each team has the skills, size, and domain knowledge
to own its area (no non-viable teams). (3) <strong>Clean ownership migration</strong> — explicit ownership
map, no orphaned or contested areas, interfaces handled. (4) <strong>Communicate the reorg well</strong> —
clear why/what/impact/plan, reassuring (reorgs cause anxiety). (5) <strong>Frame positively</strong> — the
split enables focus and speed (not a punishment). <strong>Mistakes to avoid:</strong> (1) <strong>arbitrary
boundaries</strong> — splitting through tightly-coupled work (creating cross-team coupling and coordination
pain — bad Conway); (2) <strong>non-viable teams</strong> — a team too small or missing critical skills/
domain knowledge; (3) <strong>orphaned or contested ownership</strong> — areas with no clear owner or
overlapping ownership (confusion); (4) <strong>losing domain knowledge</strong> — separating people from the
domains they know, or leaving a bus-factor-1; (5) <strong>poor communication</strong> — a confusing or
anxiety-inducing announcement (reorgs are unsettling — over-communicate); (6) <strong>ignoring the
interfaces</strong> — not defining how the split teams interact where their domains meet; (7) <strong>reorg-
ing casually / too often</strong> — but here a split is genuinely warranted (the team is drowning), so it's
the right call, done well.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Team Topologies | *Team Topologies*, Skelton & Pais |
| Conway's Law | Melvin Conway's original; "inverse Conway maneuver" |
| Cognitive load | *Team Topologies* (cognitive load); *The Manager's Path* (team size) |
| Org design | *An Elegant Puzzle*, Will Larson (org design) |

---

## Checkpoint

**Q1.** What is Conway's Law and how do you use it "on purpose," and why does a team's cognitive load matter
for how you scope it?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Conway's Law</strong> states that "organizations design systems that mirror their own communication
structure" — meaning your software architecture ends up reflecting your organization's team and communication
boundaries. Teams build systems shaped like the org: the interfaces between software components tend to
mirror the interfaces (communication boundaries) between the teams that build them, because teams that
communicate closely build tightly-integrated components, while teams that are separate build separated
components with defined interfaces. So the shape of your architecture follows the shape of your org. How you
use it <strong>on purpose</strong> (the "inverse Conway maneuver"): since architecture mirrors org structure,
you can <strong>design your team boundaries to get the architecture you want</strong> — rather than letting
an accidental org structure impose an accidental (often bad) architecture, you deliberately structure teams
around the <em>desired</em> architecture, and the architecture will follow. If you want loosely-coupled
services with clean boundaries, you structure teams that way (each team owning a service, with clear
interfaces between teams) — and the resulting architecture will tend to have those clean service boundaries
(because the teams, communicating within and interfacing between, build systems that mirror that structure).
Conversely, if teams are tangled and overlapping (unclear ownership, everyone touching everything), the
architecture will be tangled too (a big ball of mud mirroring the org's lack of clear boundaries). So team
boundaries are an <strong>architectural lever</strong>: by designing team structure deliberately (around the
architecture you want), you shape the system — "on purpose" means recognizing that team structure determines
architecture, so you design the teams to produce the architecture, rather than treating org structure and
architecture as independent. This is powerful because it means org design <em>is</em> architecture design (at
one remove), so a leader shaping team boundaries is shaping the system's structure — a lever most people don't
realize they have. Why a team's <strong>cognitive load</strong> matters for how you scope it: <strong>because
a team can only understand and effectively own so much before it's overwhelmed and its effectiveness collapses
— so a team's ownership must be sized to fit its cognitive capacity, or the team drowns</strong>. Cognitive
load (from <em>Team Topologies</em>) is the amount a team can hold in their heads and effectively manage — the
services, domains, technologies, and context they can genuinely understand and own well. This is a real,
finite limit: a team owning <em>too much</em> (too many disparate systems, too broad a domain, too much
complexity) exceeds its cognitive load — they can't hold it all, so they can't understand their own systems
well, quality and velocity suffer, they're perpetually firefighting (reacting to things they don't fully
grasp), and their effectiveness collapses (the drowning described in the scenario). So cognitive load must
<strong>drive team scoping</strong>: you should size a team's ownership (the scope of services/domains it
owns) to fit what it can genuinely master — scope teams so their responsibilities are something they can hold
and own well, not more than they can handle. When a team's ownership exceeds its cognitive load (it's
drowning), that's a clear signal to <strong>split</strong> — reduce the team's scope by giving some ownership
to another team, so each team owns a masterable amount. Ignoring cognitive load (piling more and more onto a
team, or creating teams that own too much) produces drowning teams that can't be effective no matter how hard
they work — it's an org-design problem (too much scope), not a work-harder problem. So cognitive load matters
because it's the constraint that determines how much a team can effectively own — respecting it (scoping teams
to fit) keeps teams effective; violating it (over-scoping) makes them drown. Both concepts make org design a
deliberate lever: Conway's Law means team boundaries shape the architecture (so design them to get the
architecture you want), and cognitive load means team scope must fit capacity (so size ownership to what a
team can master, splitting when it's too much) — together showing that how you structure teams (boundaries
and scope) profoundly determines both the system's architecture and the teams' effectiveness, which is why
"structure eats process for breakfast" and why deliberate org design is a powerful, underused leadership
lever.
</details>

**Q2.** Why do reorgs have "real costs" that mean you shouldn't do them casually, and what makes a team split
good?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Reorgs have <strong>real costs</strong> that mean you shouldn't do them casually because <strong>changing
team structures is genuinely disruptive — it breaks relationships and context, creates uncertainty and
anxiety, and loses productivity during the transition — so a reorg is a significant intervention whose cost
must be justified by a real structural need</strong>. The costs: (1) <strong>disruption of relationships and
context</strong> — reorgs break up teams, so people have to re-form working relationships (trust and rapport
built over time are disrupted), re-learn context (new domains, new teammates, new systems), and lose the
momentum and tacit coordination a settled team had; (2) <strong>uncertainty and anxiety</strong> — reorgs are
inherently unsettling: people worry about where they'll land, who they'll work with, whether it reflects on
their performance, what it means for their growth — this anxiety is real and costly (distraction, stress,
even attrition if handled badly); (3) <strong>lost productivity during the transition</strong> — while people
re-form teams, re-learn context, and settle in, productivity drops (the transition period is disruptive to
delivery). So a reorg is a significant, costly intervention — not a free reshuffling. This means: (1) reorgs
shouldn't be done <strong>casually or frequently</strong> — constant reorgs are hugely disruptive (people
never settle, relationships and context keep breaking) and signal thrashing (leadership doesn't know what it's
doing), which is demoralizing and unproductive; (2) reorgs should be done <strong>when there's a genuine
structural problem</strong> that warrants the cost — a real mismatch between structure and work (a team
drowning, wrong boundaries, structure impeding the architecture) where the benefit of fixing it exceeds the
reorg's cost; and (3) reorgs should be done <strong>thoughtfully</strong> — with a clear rationale, good
communication (addressing the anxiety), and minimized disruption — because how you do a reorg affects its
cost. So you weigh the real cost of a reorg against its benefit and do it only when genuinely warranted, well
— not as a casual or frequent reshuffling. What makes a <strong>team split good</strong>: (1) <strong>good
boundaries (cohesive ownership)</strong> — split along natural boundaries where each new team owns a
<em>coherent</em> domain/set of services with clear interfaces between teams (following Conway and cognitive
load), NOT arbitrary lines that cut through tightly-coupled work (which would create cross-team coupling and
constant coordination pain — a bad split that makes things worse). Each team should own something cohesive and
masterable. (2) <strong>viable staffing</strong> — each new team must have the skills, size, and domain
knowledge to viably own its area — the right mix of expertise, enough people, and the domain knowledge for
its area (keep the people who know a domain with that domain) — so no team is left non-viable (too small,
missing critical skills, or bus-factor-1 on its domain). (3) <strong>clean ownership migration</strong> — a
clear handoff of what each team now owns (an explicit ownership map), with <em>no orphaned areas</em>
(everything has a clear owner) and <em>no contested/overlapping ownership</em> (clear boundaries), and the
interfaces between teams handled (where domains meet). (4) <strong>clear communication</strong> — communicate
the rationale (why) and the change (what, who's where, what each owns, what it means for people) well, and
reassuringly (reorgs cause anxiety, so over-communicate and address the uncertainty), framing it positively
(the split enables focus and speed, not a punishment). A good split gives each team a coherent, right-sized,
well-staffed ownership with clear boundaries and clean migration, communicated well — so each team can
effectively own its area and the org is better structured. A bad split (arbitrary boundaries cutting coupled
work, non-viable teams, orphaned/contested ownership, poor communication) creates confusion, coordination
pain, and anxiety — making things worse. So a split is good when it creates cohesive, viable, clearly-owned,
well-communicated teams — applying the org-design principles (Conway, cognitive load, clear boundaries) with
attention to the human transition. Both points reflect org design as a deliberate, careful practice: reorgs
are powerful but costly (do them when genuinely needed, well — not casually), and splits (a common reorg) must
be done thoughtfully (good boundaries, viable staffing, clean migration, clear communication) to actually
improve things rather than create new problems — because structure profoundly shapes everything, so getting
it right (and changing it carefully) matters greatly.
</details>

---

## Homework

Reflect on your team structure. (1) Does your architecture mirror your org structure (Conway's Law)? Are there
places where bad team boundaries are causing bad architecture (or vice versa)? Could you use team boundaries
deliberately? (2) Is any team drowning under too much cognitive load (owning more than it can master)? Is a
split warranted? (3) Do you have the right mix of team types (stream-aligned, platform, enabling)? (4) If
you're considering a reorg, is it warranted by a real structural problem (worth the real cost), and would you
do it well? Reflect: how deliberately is your team structure designed, and what's one structural issue
(boundaries, cognitive load, a needed split) to address?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds org-design awareness — shaping team structure deliberately. A strong response examines
whether the architecture mirrors the org (Conway), whether any team is drowning under cognitive load (needing
a split), whether the mix of team types is right, and whether any reorg is warranted and would be done well.
The realizations: (1) <strong>Conway's Law</strong> — architecture mirrors org/communication structure, so
you can design team boundaries to get the architecture you want (the inverse Conway maneuver), making team
structure an architectural lever; (2) <strong>cognitive load</strong> — a team can only own so much before it
drowns, so scope teams to fit their capacity (too much = split); (3) <strong>Team Topologies patterns</strong>
— stream-aligned (primary), platform (reduce others' load), enabling (uplift), complicated-subsystem — design
the right mix with clear interactions; (4) <strong>reorgs have real costs</strong> — disruption, anxiety, lost
productivity — so do them only when a genuine structural problem warrants it, and do them well; (5) <strong>split
teams well</strong> — cohesive boundaries, viable staffing, clean ownership migration, clear communication. On
reflection, people commonly find their team structure evolved accidentally (not designed), that a team may be
drowning under too much scope (a split candidate), and that boundaries sometimes cut through coupled work
(causing coordination pain). The highest-value realization is usually a specific structural issue — a drowning
team to split, bad boundaries causing architectural pain, or a missing platform team — plus the recognition
that structure is a deliberate lever. The meta-point: how you structure teams (boundaries, ownership,
interactions) profoundly shapes what they can build and how well — structure eats process for breakfast — so
design it deliberately: use Conway's Law (architecture mirrors org, so design teams to get the architecture
you want), respect cognitive load (size ownership to fit, split when too much), use the Team Topologies
patterns, and change structure (reorgs) carefully (real cost — only when warranted, done well). Team structure
is one of the most powerful and underused leadership levers, and treating it as a deliberate design problem
(not an accident) is a mark of a mature EM. This is a higher-altitude leadership skill (shaping the
environment), connecting to cross-team work (Lesson 43) and the systemic thinking behind good engineering. The
last Phase 11 lesson covers the foundation that makes everything work — psychological safety and culture — the
environment where problems surface early and people do their best work.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 63 — Psychological Safety and Culture →](lesson-63-psychological-safety){: .btn .btn-primary }
