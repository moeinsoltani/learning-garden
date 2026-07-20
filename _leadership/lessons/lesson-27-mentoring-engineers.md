---
title: "Lesson 27 — Mentoring Engineers"
nav_order: 4
parent: "Phase 5: 1:1s, Coaching & Mentoring"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 27: Mentoring Engineers

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **scaffolding** — temporary support around growing skill (from construction scaffolds): a stretch task + a safety net, removed gradually.
> - **stretch (task/assignment)** — work slightly beyond what someone can currently do alone.
> - **safety net** — the support (check-ins, pairing, fallback) that keeps a stretch from failing badly.
> - **by osmosis** (oz-MOH-sis) — absorbing skills passively just by being nearby (doesn't work).
> - **the edge of ability** — the zone just past current skill where growth is fastest.
> - **productive struggle** — difficulty that teaches, as opposed to drowning.
> - **fade (the support)** — to withdraw scaffolding step by step as skill grows.
> - **"I do, we do, you do"** — demonstrate → do it together → they do it while you watch.
> - **reverse mentoring** — learning from someone junior about things they know better.
> - **bootcamp** — an intensive months-long programming course; a "bootcamp grad" is new to professional work.
> - **green** — inexperienced.

## Concept

Growing juniors and mid-level engineers is one of a lead's highest-return investments — but it
rarely happens well "by osmosis" (hoping people absorb skills by proximity). Deliberate mentoring —
giving people the right stretch with the right support, teaching *judgment* (not just facts), and
tracking growth intentionally — is what actually develops engineers fast. The core technique is
**scaffolding**: a stretch that's beyond what they can do alone, plus a safety net that keeps them
from failing badly, so they grow at the edge of their ability without drowning.

```
   SCAFFOLDING = STRETCH + SAFETY NET
   ┌──────────────────────────────────────────────────────┐
   │ too easy (no stretch)   → boredom, no growth           │
   │ too hard (no net)       → overwhelm, failure, damage   │
   │ STRETCH + SAFETY NET    → growth at the edge, safely   │
   │   (a task beyond them,    (check-ins, pairing, a        │
   │    but reachable)          fallback so it can't fail    │
   │                            catastrophically)           │
   └──────────────────────────────────────────────────────┘

   Grow judgment: "what would you do first?" not "here's what to do."
```

The reframe: **deliberate scaffolding beats osmosis — grow people by giving them a stretch they
couldn't do alone, with a safety net so they can attempt it safely.** People grow fastest at the
edge of their ability — doing things slightly beyond their current level, with enough support that
they can struggle productively without failing catastrophically. A lead's job is to engineer those
stretches deliberately, not hope growth happens on its own.

---

## How It Works

### Scaffolding: stretch + safety net

The core mentoring technique, from how skills are actually learned: give a **stretch** (a task
beyond what the person can currently do alone — so they grow reaching for it) plus a **safety net**
(support that keeps the stretch from failing badly — check-ins, pairing, a fallback, your
availability). Too easy = no growth (boredom); too hard with no net = overwhelm and damaging
failure. The sweet spot is a task at the edge of their ability with enough scaffolding that they
struggle *productively* (learning) rather than drowning. As they grow, remove scaffolding
gradually (fade the support), increasing the stretch — the classic apprenticeship progression.

### Teach judgment, not just facts

Facts and skills can be looked up; **judgment** — how to approach a problem, what to prioritize,
when to do what — is what makes engineers valuable and is harder to teach. Teach it by making your
thinking visible and prompting theirs: instead of "here's what to do," ask **"what would you do
first?"**, "how would you approach this?", "what are the risks you see?" — then discuss their
reasoning and add yours. This develops their decision-making (not just their knowledge), which is
what actually levels them up. Narrate your own judgment too ("here's how I'm thinking about this and
why") so they see expert reasoning, not just conclusions.

### Apprenticeship patterns

Engineering skill transfers well through apprenticeship-style patterns: **pairing** (working
together, so they see how you think and get real-time guidance), **"I do, we do, you do"** (you
demonstrate, then you do it together, then they do it with you watching — a gradual handoff),
**reviewing their work with teaching** (Lesson 28), and **giving progressively harder real work**
with decreasing support. The through-line is learning by *doing* with guidance, not by being
lectured — engineers grow through guided practice on real problems.

### Mentoring outside your team, and being mentored

Two broadening points: (1) **Mentor beyond your direct reports** — mentoring juniors on other
teams, newcomers, or through programs extends your impact and develops the broader org (and often
your reports can't get everything from you alone). (2) **Be mentored yourself, always** — no matter
your level, seek mentors (people ahead of you, peers, even reverse-mentoring from juniors on things
they know better). Modeling that you're still learning and being mentored sets a culture of growth
and keeps you developing. Growth is never "done."

### Match the stretch to the person

Scaffolding must be calibrated to the individual — the same task is an appropriate stretch for one
person and overwhelming or trivial for another. Know where each person is (their current level,
their growth edges, their confidence), and pick stretches at *their* edge. And read the person: some
need more push (they under-reach), some need more safety (they'll overwhelm themselves); some need
encouragement, some need challenge. Deliberate mentoring is individualized, not one-size-fits-all.

{: .note }
> **Grow engineers deliberately with scaffolding (stretch + safety net) — not by osmosis</br>**
> Deliberate mentoring beats hoping people grow by proximity. The core technique is scaffolding: give
> a <em>stretch</em> (a task beyond what they can do alone, so they grow reaching for it) plus a
> <em>safety net</em> (check-ins, pairing, a fallback, so it can't fail catastrophically) — the sweet
> spot where they struggle productively rather than drowning, with support faded as they grow. Teach
> <em>judgment</em>, not just facts ("what would you do first?" and narrating your own reasoning),
> use apprenticeship patterns (pairing, "I do/we do/you do", guided real work), mentor beyond your
> team, and keep being mentored yourself. Calibrate the stretch to each individual. People grow
> fastest at the edge of their ability with a safety net — and engineering those stretches
> deliberately, teaching judgment, is how a lead develops engineers fast, which is one of the
> highest-return investments a lead makes.

---

## Lab — Scenario

**The situation:** A bootcamp graduate, Amir, is joining your team — his first professional
engineering job. He's smart and eager but green: he can write code, but has never worked in a large
codebase, done code review, shipped to production, or handled the judgment side of engineering. You
want to deliberately develop him over his first 90 days, not just throw him tickets and hope.

**Design a 90-day growth arc for Amir** — week-by-week (or phase-by-phase) responsibilities, the
scaffolding at each stage, checkpoints, and failure signals to watch for. Then note the principles
and mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong 90-day arc scaffolds increasing stretch with decreasing support, teaches judgment, and has
checkpoints and failure signals. Example:
<br><br>
<strong>Weeks 1–2 (Orientation — heavy scaffolding):</strong> Goal: get oriented and ship something
tiny. Responsibilities: environment setup, read the codebase with a guide, pair with a buddy, and
ship one <em>tiny</em>, well-scoped change (a small bug fix or copy change) end-to-end — to learn the
whole pipeline (branch, PR, review, deploy) on something low-risk. Scaffolding: assigned buddy,
pairing, hand-picked trivial first task, lots of availability. Checkpoint: did he ship the tiny change
and understand the pipeline? Failure signals: totally lost in setup (needs more help), or disengaged.
<br><br>
<strong>Weeks 3–4 (First real tickets — strong scaffolding):</strong> Goal: complete small, real
tickets with support. Responsibilities: a few small, well-defined tickets; start doing code review
(receiving thorough teaching reviews — Lesson 28); begin learning to break down a task. Scaffolding:
tickets pre-scoped for him, frequent check-ins, pairing on the hard parts, thorough review. Teach
judgment: before he starts each, ask "how would you approach this? what would you do first?" —
develop his planning. Checkpoint: completing small tickets with decreasing help; engaging well with
review. Failure signals: consistently stuck without asking (isolation), or work needing heavy rework
(gap in fundamentals to address).
<br><br>
<strong>Weeks 5–8 (Growing independence — fading scaffolding):</strong> Goal: own slightly larger,
less-defined work. Responsibilities: a medium feature (with some ambiguity he has to resolve), doing
his own task breakdown (which you review), more independent code review. Scaffolding: less pre-
scoping (he scopes, you check), check-ins moved from daily to a few times a week, pairing only on the
genuinely hard parts. Teach judgment: have him propose the approach and reasoning before building;
discuss trade-offs. Checkpoint: can he take a medium, somewhat-ambiguous task and deliver it with
light guidance? Failure signals: struggling with ambiguity (needs more scaffolding on breaking down
work), or over-engineering/under-thinking (judgment gaps to coach).
<br><br>
<strong>Weeks 9–12 (Approaching autonomy — light scaffolding):</strong> Goal: operate as a
contributing junior with normal support. Responsibilities: own a feature reasonably independently,
participate fully in review (giving and receiving), handle a small production task (with a safety
net). Scaffolding: normal team support (1:1s, review, ask-when-stuck), pairing only when he requests
or for genuinely new territory. Checkpoint (end of 90 days): is he a contributing junior — shipping
real work with normal support, showing developing judgment? Failure signals: still needs heavy
hand-holding (re-assess — pace, fit, or support), or conversely cruising with no stretch (increase
challenge).
<br><br>
<strong>Throughout:</strong> weekly 1:1s (how's he feeling, is it too much/little, what's confusing);
teach judgment constantly ("what would you do first?", narrate your reasoning); psychological safety
(it's okay to ask, to not know, to make safe mistakes); adjust the pace to <em>him</em> (the arc is a
template, calibrated to his actual growth).
<br><br>
<strong>Principles:</strong> (1) <strong>Scaffolding: increasing stretch, decreasing support</strong>
— each phase stretches a bit beyond the last, with support faded as he grows (the apprenticeship
progression), so he's always at his productive edge. (2) <strong>Start tiny and safe</strong> — the
week-1 trivial end-to-end change teaches the pipeline on something low-risk (a safe first win). (3)
<strong>Teach judgment throughout</strong> — "what would you do first?" and narrating your reasoning
develop his decision-making, not just his coding. (4) <strong>Checkpoints and failure signals</strong>
— defined checkpoints track progress; watching for failure signals (drowning, isolating,
disengaging, or under-stretched) lets you adjust. (5) <strong>Calibrate to him</strong> — the arc is
a template; adjust pace and support to his actual growth. <strong>Mistakes to avoid:</strong> (1)
<strong>throwing him in with no scaffolding</strong> — a hard task with no safety net overwhelms and
can damage confidence (or cause real failures); (2) <strong>too little stretch</strong> — trivial
work for 90 days = boredom and no growth (bootcamp grads are often more capable than assumed if
scaffolded); (3) <strong>osmosis</strong> — "here are tickets, figure it out" without deliberate
development, judgment-teaching, or checkpoints; (4) <strong>not teaching judgment</strong> — only
giving him tasks, never developing his decision-making (he stays a code-writer, not an engineer); (5)
<strong>not watching failure signals</strong> — missing that he's drowning (and losing confidence) or
isolating (stuck but not asking) or under-challenged; (6) <strong>rigidly following the arc</strong>
— ignoring his actual pace (some grads move faster, some need more time); (7) <strong>no safety for
mistakes</strong> — if he's afraid to ask or err, he won't stretch (psychological safety is the
foundation of the safety net).
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Scaffolding & apprenticeship | *Apprenticeship Patterns*, Hoover & Oshineye |
| Growing engineers | *The Manager's Path*, Fournier (managing juniors); Lara Hogan |
| Teaching judgment / coaching | Lesson 25 (questions); Lesson 26 (coaching vs mentoring) |
| Code review as teaching (next) | Lesson 28 (code review as teaching) |

---

## Checkpoint

**Q1.** What is "scaffolding" (stretch + safety net), and why does it grow people faster than either
too-easy work or too-hard work with no support?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Scaffolding</strong> is the core mentoring technique of giving someone a <strong>stretch</strong>
(a task beyond what they can currently do alone, so they grow reaching for it) combined with a
<strong>safety net</strong> (support that keeps the stretch from failing badly — check-ins, pairing, a
fallback, your availability). The term comes from how skills are actually learned: like scaffolding
around a building, it provides temporary support that lets someone attempt something they couldn't do
unsupported, and is gradually removed (faded) as their own capability grows. The key is the combination:
a stretch <em>with</em> a net, so the person operates at the edge of their ability — struggling
productively (which is how growth happens) — without drowning. Why it grows people faster than the
alternatives: <strong>people grow fastest at the edge of their ability, and scaffolding puts them
exactly there safely, whereas too-easy work provides no growth and too-hard work with no net causes
damaging failure rather than learning</strong>. Consider the three cases: (1) <strong>too-easy work (no
stretch)</strong> — if the task is within what the person can already do comfortably, there's nothing to
reach for, so no growth happens; they're just exercising existing skills (and often getting bored and
disengaged). Comfort produces no development. (2) <strong>too-hard work with no safety net</strong> — if
the task is far beyond their ability with no support, they're overwhelmed: they can't reach it, they
flounder unproductively, and they're likely to fail badly — which (a) doesn't teach (drowning isn't
learning; productive struggle requires being <em>able</em> to make progress with effort), (b) damages
their confidence (a demoralizing failure), and (c) may cause real harm (a botched important task). Too
much stretch without a net is destructive, not developmental. (3) <strong>stretch + safety net
(scaffolding)</strong> — a task beyond their current solo ability, but with enough support that they
can struggle <em>productively</em> and reach it: this is the growth sweet spot, because they're
genuinely stretched (so they grow) but supported enough that the struggle is learning rather than
drowning (they make progress, with help available when truly stuck, so they don't fail catastrophically
or lose confidence). The person operates at the edge of their ability — the exact place where learning
is fastest — safely. And as they grow, you fade the scaffolding (less support, more stretch), so
they're continuously at their (rising) edge — the apprenticeship progression. So scaffolding grows
people faster because it deliberately keeps them in the productive-struggle zone (stretched but
supported), which is where growth actually happens — whereas too-easy work leaves them below that zone
(no stretch, no growth) and too-hard-with-no-net throws them above it (overwhelmed, failing, not
learning). The safety net is what makes the stretch <em>developmental</em> rather than
<em>destructive</em> — it converts a task that would otherwise overwhelm into one they can learn from.
This is why deliberate scaffolding beats "osmosis" (hoping people grow by proximity) and beats both
under-challenging (safe but stagnant) and over-challenging (sink-or-swim, which often sinks people): a
lead who engineers stretches at each person's edge, with calibrated support faded over time, develops
engineers fast and safely — which is the deliberate craft of mentoring rather than leaving growth to
chance.
</details>

**Q2.** Why should mentoring teach *judgment* (not just facts), and how do you do that?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Mentoring should teach judgment (not just facts) because <strong>judgment — how to approach problems,
what to prioritize, when to do what, how to weigh trade-offs — is what makes an engineer genuinely
valuable and capable, and it's the harder, higher-value thing to develop; facts and skills can be
looked up or picked up, but judgment must be deliberately cultivated</strong>. The distinction: facts
and specific skills (how a particular API works, the syntax, a specific pattern) are relatively easy to
acquire — they can be looked up, googled, or learned from documentation, and they're concrete. Judgment
— knowing <em>which</em> approach fits a situation, what to prioritize, what the risks are, when the
simple solution suffices versus when you need the complex one, how to break down an ambiguous problem —
is what separates someone who can write code from someone who can engineer well, and it's much harder
to acquire (it's not in the docs; it comes from experience and guided reflection). So if mentoring only
transfers facts and skills (here's how to do X), it produces someone who knows things but can't
navigate the judgment calls that real engineering requires — they'll keep needing others to make the
decisions. Teaching judgment develops the decision-making that makes someone an autonomous, capable
engineer — which is the actual goal of growing someone (not a walking fact-database, but a good
decision-maker), and the higher-leverage investment (judgment transfers across all future situations,
where a specific fact applies narrowly). How you teach judgment: (1) <strong>prompt their thinking
instead of giving answers</strong> — ask "what would you do first?", "how would you approach this?",
"what are the risks you see?", "what are the trade-offs?" — which makes them exercise and develop their
own decision-making (rather than receiving your conclusion), then discuss their reasoning and add
yours. This is coaching (Lesson 26) applied to developing judgment: they build the judgment by
practicing it with your guidance, not by being handed conclusions. (2) <strong>narrate your own
reasoning</strong> — make your expert thinking visible: "here's how I'm thinking about this and why —
I'm weighing X against Y, and I'm leaning this way because..." — so they see not just <em>what</em> you
decide but <em>how</em> you reason (the judgment process itself), which is what they need to learn.
Showing your reasoning (not just your conclusions) lets them internalize how an experienced engineer
thinks. (3) <strong>discuss trade-offs and decisions, not just implementations</strong> — engage them
on the judgment-level questions (why this approach, what are the alternatives, what could go wrong)
rather than only the mechanics, so they practice the decision-making. (4) <strong>use real decisions as
teaching moments</strong> — when they face a judgment call, coach them through it (their reasoning +
yours) rather than deciding for them, so each real decision develops their judgment. The through-line:
judgment is developed by <em>practicing it with guidance</em> — prompting their reasoning, showing
yours, and discussing the decision-level questions — not by being told facts or given answers. So
mentoring that teaches judgment (through questions that develop their thinking and narration that shows
expert reasoning) produces engineers who can make good decisions autonomously, which is what actually
levels them up — versus mentoring that only transfers facts, which produces someone who knows things
but can't navigate the judgment that real engineering requires. Teaching judgment is the higher-value,
harder mentoring work, and it's done by developing the person's own reasoning (coaching) and making
expert reasoning visible (narration), on real problems — which is exactly the deliberate, judgment-
focused mentoring that grows engineers into capable, autonomous professionals.
</details>

---

## Homework

Pick a junior or mid-level engineer you can develop (or think about how you would). (1) Assess where
they are and their growth edges — what would be an appropriate <em>stretch</em> (beyond their current
solo ability but reachable)? (2) Design the scaffolding — what safety net (check-ins, pairing, a
fallback) would let them attempt the stretch safely? (3) Plan to teach judgment — where can you ask
"what would you do first?" and narrate your own reasoning, rather than just giving answers? (4)
Consider: are you developing people deliberately, or hoping for osmosis? Are you mentoring beyond your
team, and being mentored yourself? Reflect: what's one person you could develop more deliberately with
scaffolding, and what's the stretch + net?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds deliberate mentoring — growing engineers with intent rather than hoping for osmosis.
A strong response picks a real person, assesses their growth edge, designs a stretch + safety net
(scaffolding), plans to teach judgment (questions + narrating reasoning), and reflects on whether their
mentoring is deliberate. The realizations: (1) <strong>scaffolding (stretch + safety net) grows people
fastest</strong> — a task beyond their solo ability with enough support to struggle productively rather
than drown puts them at their growth edge safely, where too-easy work (no stretch) stagnates and
too-hard-with-no-net overwhelms and damages; support is faded as they grow; (2) <strong>teach judgment,
not just facts</strong> — judgment (how to approach, prioritize, weigh trade-offs) is what makes
engineers valuable and is the harder, higher-value thing to develop, taught by prompting their reasoning
("what would you do first?") and narrating yours, not by giving answers; (3) <strong>deliberate beats
osmosis</strong> — hoping people grow by proximity underdelivers; engineering stretches, teaching
judgment, and tracking growth develops engineers fast; (4) <strong>calibrate to the individual and keep
growing yourself</strong> — the stretch must match each person's edge, and modeling that you're still
being mentored sets a growth culture. On reflection, people commonly find they've been relying on
osmosis (giving tickets and hoping) rather than deliberately scaffolding and teaching judgment, and that
picking a person and designing a real stretch + net + judgment-teaching would develop them much faster.
The highest-value shift is usually: mentor deliberately (design stretches at each person's edge, with
calibrated safety nets and faded support) and teach judgment (prompt their reasoning, narrate yours)
rather than just assigning work. The meta-point: growing juniors and mid-levels is one of a lead's
highest-return investments, and it rarely happens well by osmosis — deliberate scaffolding (stretch +
safety net), teaching judgment (not just facts), apprenticeship patterns (pairing, "I do/we do/you
do"), and individualized calibration are what actually develop engineers fast. This is the craft of
mentoring, built on the coaching/questioning skills (Lessons 25–26). Developing people is how a lead's
impact scales beyond their own hands — a team that keeps getting better outproduces a static one. The
next lesson focuses on the highest-volume, most frequent mentoring channel most leads have: code review
as teaching — turning review comments into a deliberate teaching tool.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 28 — Code Review as Teaching →](lesson-28-code-review-teaching){: .btn .btn-primary }
