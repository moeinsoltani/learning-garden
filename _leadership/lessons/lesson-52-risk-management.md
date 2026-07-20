---
title: "Lesson 52 — Risk Management"
nav_order: 3
parent: "Phase 10: Project Leadership"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 52: Risk Management

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **pre-mortem** — imagining the project has *already failed* and asking why; surfaces the risks optimism hides (opposite of a post-mortem).
> - **risk register** — the maintained list of a project's risks, each with likelihood, impact, and an owner.
> - **likelihood × impact** — how risks are prioritized: probability times damage.
> - **owner** (of a risk) — the named person responsible for watching and mitigating it.
> - **mitigate** (MIT-ih-gate) — to reduce a risk's likelihood or impact in advance.
> - **theater** — activity that looks like the real thing but changes nothing ("risk theater").
> - **risk vs issue** — something that *might* happen vs something that *has* happened; prevented vs resolved.
> - **naysayer** — the person seen as negative for pointing out problems (the pre-mortem removes this stigma).
> - **front-load** — to do the hard, scary parts first, not last.
> - **the quiet risks** — the non-technical killers: people leaving, dependencies slipping, unclear ownership.

## Concept

Most projects that fail are killed by a risk that was **identifiable in week one** but wasn't found (or
was ignored) until it was too late. Good risk management is about **finding the thing that will actually
kill the project early** — surfacing the real risks, prioritizing them (likelihood × impact), assigning
owners, and **retiring the scariest ones first.** The most powerful tool is the **pre-mortem**: imagine
the project has already failed, and ask why — which surfaces risks that optimism hides.

```
   THE PRE-MORTEM (imagine you already failed)
   ┌──────────────────────────────────────────────────────┐
   │ "It's six months later. The project FAILED. Why?"      │
   │ → surfaces risks optimism hides (people say the quiet   │
   │   fears out loud when failure is assumed)              │
   │ RISK = likelihood × impact × OWNER (not theater)        │
   │ DE-RISK ORDER: hardest/scariest FIRST                   │
   │ watch the QUIET risks: people, dependencies (not tech)  │
   └──────────────────────────────────────────────────────┘
```

The reframe: **find the project-killing risk in week one, not week ten — via pre-mortems, and retire the
scariest risks first.** Optimism and momentum hide risks (nobody wants to voice the fear that it might
fail); the pre-mortem ("assume we failed — why?") gives permission to surface them. And once you know the
risks, you retire the scariest/hardest ones <em>first</em> (so if the project is doomed, you find out
early) rather than doing the easy parts first and hitting the fatal risk late.

---

## How It Works

### The pre-mortem — imagine you already failed

The most powerful risk-surfacing tool: the **pre-mortem.** Before/early in a project, gather the team and
pose: **"It's six months from now and this project failed. What went wrong?"** Then everyone writes down
the reasons. This works because it <em>gives permission to voice fears</em>: normally, optimism and social
pressure suppress risk-talk (nobody wants to be the naysayer, and momentum assumes success), so real risks
go unspoken; but by <em>assuming</em> failure, the pre-mortem makes it safe and natural to name what could
go wrong (you're not being negative — you're doing the exercise). It surfaces the quiet fears people were
already thinking but not saying — often the real risks. Run a pre-mortem early to surface risks while
there's time to address them (versus discovering them when they materialize).

### Risk registers that aren't theater — likelihood × impact × owner

A **risk register** (a list of risks) is only useful if it's real, not theater. Make it real by: (1)
prioritizing risks by **likelihood × impact** (a high-likelihood, high-impact risk matters far more than a
remote, minor one — focus on the big ones); and crucially (2) assigning each significant risk an
**owner** — a specific person responsible for monitoring and mitigating it. A risk register that's just a
list (no prioritization, no owners) is theater (it looks like risk management but nothing happens);
likelihood × impact × owner makes it actionable (you focus on the biggest risks, and someone's
accountable for each). The owner is what turns "we noted the risk" into "someone's actively managing it."

### De-risking order — hardest/scariest first

The key sequencing principle: **retire the hardest, scariest, most-uncertain risks FIRST** — not last.
The instinct (and comfort) is to do the easy, known parts first (visible progress, feels good) and leave
the scary uncertain parts for later. But that's backwards: if a project has a fatal risk (the hard
integration won't work, the approach is infeasible), you want to find out <em>early</em> (when you can
pivot, cut scope, or cancel cheaply), not after months of investment in the easy parts. Retiring the
scariest risks first means: if the project is doomed, you learn it early (cheap failure); if it's viable,
you've removed the biggest uncertainty (and the rest is more predictable). Front-load the risk, don't
defer it. (This is why milestones retire risk hardest-first — Lesson 50.)

### Risk vs issue — different things

Distinguish a **risk** (something that <em>might</em> happen — a future possibility, managed by
mitigation/monitoring) from an **issue** (something that <em>has</em> happened — a current problem, managed
by resolution). They need different handling: risks are about probability and prevention (reduce
likelihood/impact before they materialize); issues are about response (deal with the problem now). Confusing
them — treating a live issue as a "risk" to monitor (when it needs fixing now), or ignoring a risk until
it becomes an issue (missing the chance to prevent it) — leads to poor management. Manage risks proactively
(before they happen) and issues reactively (once they have).

### The quiet risks — people and dependencies

The risks that actually kill projects are often <em>not</em> the technical ones (which get attention) but
the **quiet risks: people and dependencies.** People risks: a key person leaving/being unavailable, team
capacity, skills gaps, burnout, unclear ownership. Dependency risks: another team not delivering, an
external dependency slipping, an unclear interface (Lesson 53). These are easy to under-attend (they're
less concrete than technical risks, and feel awkward to name — "what if Priya leaves?") but they're
frequently the real project-killers. Deliberately watch for the quiet, non-technical risks (people,
dependencies, organizational) — the pre-mortem helps surface them, and they often deserve as much
attention as the technical risks.

{: .note }
> **Find the project-killing risk in week one — via pre-mortem, and retire the scariest risks first</br>**
> Most failed projects are killed by a risk identifiable early but found too late. Surface risks with a
> <em>pre-mortem</em> ("it's six months later and we failed — why?"), which gives permission to voice the
> fears optimism suppresses. Make the risk register real (not theater) with <em>likelihood × impact ×
> owner</em> (prioritize the big ones, someone accountable for each). Retire the <em>hardest/scariest risks
> first</em> (so if the project's doomed, you find out early and cheaply — not after investing in the easy
> parts). Distinguish <em>risks</em> (might happen — prevent) from <em>issues</em> (have happened — resolve).
> And watch the <em>quiet risks</em> — people and dependencies, not just technical — which are often the
> real project-killers. Finding and retiring the real risks early is what prevents the late, expensive
> surprises that kill projects.

---

## Lab — Scenario

**The situation:** You're about to start a significant **database migration project** — moving your main
production database to a new system (say, a major version upgrade or a move to a different database), over
the next few months. It's high-stakes (production data, everything depends on the DB) and the kind of
project that can go badly wrong. Before you dive in, you want to run a pre-mortem to surface the real
risks.

**Run a written pre-mortem on the migration project** — imagine it failed, list what went wrong, then
extract the top three risks and the week-one de-risking actions. Then note the principles and mistakes to
avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong pre-mortem surfaces a wide range of failure modes (including quiet people/dependency risks), then
prioritizes and plans early de-risking. Example:
<br><br>
<strong>Pre-mortem — "It's [N] months later and the migration failed. Why?":</strong> <br>
• Data corruption / data loss during migration (catastrophic). <br>
• Downtime exceeded the acceptable window (the cutover took too long / went wrong). <br>
• The new database didn't handle our production load / query patterns (performance regression). <br>
• Subtle incompatibilities (queries, features, behaviors that differ) broke things in unexpected ways. <br>
• We couldn't roll back when something went wrong (no safe rollback path). <br>
• A dependent service broke because it relied on old-DB-specific behavior. <br>
• The one person who deeply understood the old DB left / was unavailable (people risk). <br>
• We underestimated the effort and ran out of time / it dragged on (estimation). <br>
• Testing didn't catch a critical case that only appeared in production. <br>
<br>
<strong>Top three risks (likelihood × impact):</strong> <br>
1. <strong>Data loss/corruption during migration</strong> (moderate likelihood × catastrophic impact = top
risk). <br>
2. <strong>The new DB can't handle production load/query patterns</strong> (moderate likelihood × severe
impact — a performance disaster in prod). <br>
3. <strong>No safe rollback / excessive downtime at cutover</strong> (moderate likelihood × severe impact —
the risky cutover is often the real danger, not the new code). <br>
<em>(Prioritized by likelihood × impact; note these are the scariest, most-uncertain — to retire first.)</em>
<br><br>
<strong>Week-one de-risking actions (retire the scariest first):</strong> <br>
• For risk 1 (data loss): week one, prove the migration process on a copy of production data — validate data
integrity end-to-end, build verification/checksums, and prove we can detect any corruption. <em>Retire "we
can't migrate data safely" early.</em> <br>
• For risk 2 (load/performance): week one, test the new DB against production-like load and our real query
patterns (on a representative dataset) — find performance problems now, not in prod. <em>Retire "it can't
handle our load."</em> <br>
• For risk 3 (rollback/downtime): week one, design and test the cutover and rollback plan — prove we can
roll back safely and estimate the downtime, before committing to the approach. <em>Retire "we can't cut over
safely."</em> <br>
• Assign each risk an owner (someone accountable for monitoring/mitigating it). <br>
• Watch the quiet risks: ensure the old-DB knowledge isn't bus-factor-1 (Lesson 34); confirm dependent teams
are aware and involved.
<br><br>
<strong>Principles:</strong> (1) <strong>Pre-mortem surfaces the real risks</strong> — assuming failure gives
permission to name the fears (data loss, load, rollback, the person leaving) that optimism would suppress.
(2) <strong>Prioritize by likelihood × impact</strong> — focus on the catastrophic/severe ones (data loss,
load, cutover). (3) <strong>Retire the scariest first (week one)</strong> — prove data-migration integrity,
load handling, and rollback <em>early</em>, so if the approach is doomed, we find out cheaply. (4)
<strong>Owners</strong> — each risk has someone accountable. (5) <strong>Watch quiet risks</strong> — the
old-DB knowledge holder (people), dependent services (dependencies) — not just technical. (6) <strong>The
cutover/rollback is often the real risk</strong> — not the new code, but the risky migration/cutover.
<strong>Mistakes to avoid:</strong> (1) <strong>skipping the pre-mortem</strong> — diving in optimistically,
so risks surface when they materialize (too late); (2) <strong>doing the easy parts first</strong> (building
the new schema) and leaving the scary parts (data migration, cutover, load) for last — discovering a fatal
problem in month three; (3) <strong>a risk register that's theater</strong> — a list with no
prioritization or owners; (4) <strong>only technical risks</strong> — missing the people (knowledge holder)
and dependency (dependent services) risks; (5) <strong>no rollback plan</strong> — the classic migration
killer (can't undo when something breaks); (6) <strong>testing only late</strong> — not proving data
integrity, load, and rollback in week one when there's time to change approach.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Pre-mortems | Gary Klein, "Performing a Project Premortem" (HBR) |
| Risk registers & management | PMI risk management; likelihood × impact matrices |
| De-risking order | Lesson 50 (risk-retiring milestones); *The Lean Startup* (riskiest assumption) |
| People & dependency risks | Lesson 34 (bus factor); Lesson 53 (dependencies) |

---

## Checkpoint

**Q1.** Why is a pre-mortem effective at surfacing risks, and why retire the "hardest/scariest risks first"?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A <strong>pre-mortem</strong> (imagining the project has already failed and asking why) is effective at
surfacing risks because <strong>it gives people permission to voice the fears and risks that optimism and
social pressure normally suppress</strong>. In a normal project setting, real risks often go unspoken for
psychological and social reasons: (1) <strong>optimism and momentum</strong> — a starting project has energy
and an assumption of success, so voicing "this might fail because X" feels off-key, negative, or disloyal;
(2) <strong>nobody wants to be the naysayer</strong> — the person who raises risks can seem like a
pessimist, a blocker, or not a team player, so people self-censor their concerns; (3) <strong>the fears are
uncomfortable to name</strong> — some real risks (a key person might leave, the approach might not work, a
dependency might slip) feel awkward or even taboo to say out loud. So the real risks — often already sensed
by team members — stay unvoiced, and the project proceeds without addressing them (until they materialize).
The pre-mortem cleverly removes these barriers by <strong>assuming failure and asking why</strong>: "It's
six months later and the project failed — what went wrong?" This reframing makes voicing risks <em>the
task</em>, not negativity — you're not being a pessimist or disloyal by naming failure modes; you're doing
the exercise (everyone is imagining failure and its causes). This gives permission and even encouragement to
surface the fears people were already thinking but not saying — the quiet concerns, the uncomfortable risks,
the "what if X" worries — which are often the real risks. By making it safe and natural to name what could go
wrong (because failure is the premise), the pre-mortem surfaces risks that would otherwise stay hidden until
they killed the project. It's a psychological trick that unlocks the team's real risk-awareness (which
optimism and social pressure normally suppress), surfacing risks early (when there's time to address them)
rather than having them emerge when they materialize. Why <strong>retire the hardest/scariest risks
first</strong>: <strong>because if a project has a fatal risk, you want to discover it early (when you can
pivot, cut scope, or cancel cheaply), not after investing months in the easy parts — so front-loading the
scary risks means you fail cheap if doomed, and remove the biggest uncertainty if viable</strong>. The
instinct (and comfort) is to do the easy, known parts first: they show visible progress, feel productive,
and defer the scary uncertain parts. But this is backwards from a risk perspective: if the project has a
fatal risk (the hard integration won't work, the new database can't handle the load, the approach is
infeasible), doing the easy parts first means you <em>don't find out about the fatal risk until late</em> —
after you've invested months in the easy parts, when discovering the project is doomed is expensive (much
sunk investment, and less time/room to respond). Retiring the <strong>hardest/scariest risks first</strong>
inverts this: you tackle the most uncertain, highest-risk parts <em>early</em>, so: (1) <strong>if the
project is doomed</strong> (the scary risk can't be retired — the approach doesn't work), you find out
<em>early</em>, when you've invested little and can still pivot, cut scope, or cancel cheaply (a cheap
failure — Lesson 49's cheapest-test-first and killing projects well); (2) <strong>if the project is
viable</strong> (the scary risk is retired — the hard part works), you've removed the biggest uncertainty
early, so the rest of the project is more predictable and lower-risk (you're building on validated
foundations). Either way, front-loading the risk is better: you either fail cheap (if doomed) or de-risk
early (if viable), versus deferring the risk and discovering a fatal problem late (expensive failure) or
carrying big uncertainty throughout. This is why risk-retiring milestones go hardest-first (Lesson 50): the
sequencing that retires the scariest risks first minimizes the cost of failure (find out early) and maximizes
predictability (remove big unknowns early). Both points reflect proactive risk management: surface the real
risks early (pre-mortem, giving permission to voice suppressed fears) and retire the scariest ones first
(so you fail cheap if doomed, de-risk early if viable) — together finding the project-killing risk in week
one rather than week ten, which is what prevents the late, expensive surprises that kill projects.
</details>

**Q2.** Why must a risk register have "likelihood × impact × owner" to be real, and why watch the "quiet
risks" (people, dependencies)?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A risk register must have <strong>likelihood × impact × owner</strong> to be real (not theater) because
<strong>those three elements are what turn a list of risks into actionable risk management — prioritization
(likelihood × impact) focuses effort on the risks that matter, and owners make someone accountable for
actually managing each, without which the register is just a list that changes nothing</strong>. A risk
register that's merely a list of risks (no prioritization, no owners) is <strong>theater</strong>: it looks
like risk management (you've documented the risks), but nothing actually happens as a result — the risks sit
on a list, unprioritized (so you don't know which to focus on) and unowned (so nobody's responsible for
doing anything about them), and the project proceeds as if the register didn't exist. To make it real: (1)
<strong>likelihood × impact</strong> — prioritize risks by how likely they are and how bad they'd be, because
risks aren't equal: a high-likelihood, high-impact risk (data loss in a migration — moderately likely,
catastrophic) demands serious attention, while a remote, minor risk doesn't; prioritizing by likelihood ×
impact focuses your limited risk-management effort on the risks that actually matter (the big ones), rather
than spreading attention evenly or on trivial risks. Without prioritization, you can't tell which risks to
focus on, so you either try to manage all equally (impossible) or manage none. (2) <strong>an owner</strong>
— each significant risk needs a specific person responsible for monitoring and mitigating it, because
without an owner, "we noted the risk" doesn't become "someone's actively managing it" — a risk with no owner
is nobody's job, so it's monitored by no one and mitigated by no one (it just sits on the list until it
materializes). Assigning an owner makes someone accountable for actually watching the risk and doing
something about it (mitigation, monitoring, escalation), which is what turns risk-documentation into
risk-management. So likelihood × impact × owner is what makes a risk register actionable: you focus on the
biggest risks (likelihood × impact) and someone's accountable for each (owner) — versus a theater list that
documents risks but manages none. Why watch the <strong>quiet risks (people, dependencies)</strong>:
<strong>because the risks that actually kill projects are often not the technical ones (which get attention)
but the people and dependency risks — which are easy to under-attend (less concrete, awkward to name) yet
frequently the real project-killers</strong>. Technical risks get attention naturally — they're concrete,
engineers are comfortable discussing them, and they feel like "real" project risks (will the architecture
work, can we handle the load). But the risks that most often actually kill projects are frequently the
<strong>quiet, non-technical ones</strong>: (1) <strong>people risks</strong> — a key person leaving or
being unavailable (bus factor — Lesson 34), team capacity being insufficient, a skills gap, burnout, unclear
ownership (nobody's really driving it); and (2) <strong>dependency risks</strong> — another team not
delivering what you depend on, an external dependency slipping, an unclear interface (Lesson 53). These are
easy to <em>under-attend</em> for several reasons: they're <em>less concrete</em> than technical risks
(harder to pin down and analyze), they feel <em>awkward to name</em> ("what if Priya leaves?" or "what if
the platform team doesn't deliver?" can feel uncomfortable, political, or like doubting people), and they're
outside the technical comfort zone engineers naturally focus on. So they get less attention than technical
risks — yet they're frequently what actually sinks projects (the project fails because a dependency slipped
or the key person left, not because the architecture was wrong). Deliberately watching for the quiet risks —
people (key-person dependencies, capacity, ownership) and dependencies (other teams, external, interfaces) —
ensures these frequent real project-killers get the attention they deserve, rather than being overshadowed
by the more comfortable technical risks. The pre-mortem helps surface them (giving permission to name the
awkward people/dependency fears), and they often deserve as much attention as (or more than) the technical
risks. Both points reflect making risk management real and complete: a real register (likelihood × impact ×
owner — prioritized and accountable, not theater) and attention to the quiet risks (people, dependencies —
the frequent real killers, not just the comfortable technical ones) — together ensuring you actually manage
the risks that matter, including the non-technical ones that optimism and technical focus would otherwise
leave unaddressed until they kill the project.
</details>

---

## Homework

Practice risk management on a project. (1) Run a pre-mortem — imagine the project failed, and list (with the
team) all the reasons why; notice what surfaces that optimism was hiding. (2) Prioritize the risks by
likelihood × impact, and assign each significant one an owner (make the register real, not theater). (3)
Sequence your work to retire the hardest/scariest risks first (not the easy parts first) — what would you do
in week one to retire the top risk? (4) Deliberately look for the quiet risks (people, dependencies), not
just technical ones. Reflect: are you finding project-killing risks early or late, and what's the scariest
risk on a current project that you should be retiring first?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds risk management — finding and retiring the project-killing risks early. A strong response
runs a pre-mortem (noticing what optimism was hiding), prioritizes risks by likelihood × impact with owners
(a real register, not theater), sequences to retire the scariest risks first (week-one de-risking), and
watches for quiet people/dependency risks. The realizations: (1) <strong>the pre-mortem surfaces hidden
risks</strong> — assuming failure gives permission to voice the fears optimism and social pressure suppress,
surfacing the real risks early (when there's time to address them); (2) <strong>likelihood × impact × owner
makes a register real</strong> — prioritization focuses effort on the big risks, and owners make someone
accountable, without which the register is theater; (3) <strong>retire the scariest risks first</strong> — so
if the project's doomed you find out early and cheaply (not after investing in easy parts), and if viable you
remove the biggest uncertainty early; (4) <strong>watch the quiet risks</strong> — people (key-person
dependencies, capacity, ownership) and dependencies (other teams, interfaces) are often the real
project-killers but get under-attended vs. technical risks. On reflection, people commonly find they don't
run pre-mortems (so risks surface late), do the easy parts first (leaving scary risks unretired), keep
theater-registers (lists with no owners/prioritization), and focus on technical risks while under-attending
people and dependency risks. The highest-value change is usually: run a pre-mortem to surface the real risks
early, and sequence to retire the scariest ones first (week-one de-risking). The meta-point: most failed
projects are killed by a risk identifiable in week one but found too late, so good risk management is finding
the project-killing risk early — via pre-mortems (surfacing suppressed fears), a real register (likelihood ×
impact × owner), retiring the scariest risks first (fail cheap if doomed, de-risk early if viable),
distinguishing risks from issues, and watching the quiet people/dependency risks. Finding and retiring the
real risks early prevents the late, expensive surprises that kill projects. It pairs with risk-retiring
milestones (Lesson 50) and dependency management (Lesson 53). The next lesson digs into one of the biggest
quiet risks — dependencies — and how to keep them from being the reason projects slip.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 53 — Dependency Management →](lesson-53-dependency-management){: .btn .btn-primary }
