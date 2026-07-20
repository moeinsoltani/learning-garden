---
title: "Lesson 40 — Working with Product Managers and Designers"
nav_order: 1
parent: "Phase 8: Stakeholder Management"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 40: Working with Product Managers and Designers

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **the triad** — product + design + engineering working as three equal partners.
> - **PM (product manager)** — owns *what* to build and *why* (user needs, business value, priorities).
> - **feature factory** — an engineering team that only executes finished specs with no say in what gets built.
> - **spec** — a written specification of what to build; **ticket** — one packaged unit of work.
> - **healthy tension** — productive disagreement between perspectives that produces better decisions.
> - **problem definition** — the stage where the *problem* is chosen, before any solution; where engineering should engage.
> - **feasibility** (fee-zih-BIL-ih-tee) — whether something can practically be built at acceptable cost.
> - **veto** — blocking something outright; push back **with alternatives** instead.
> - **conduit** (KON-doo-it) — a channel things flow through; the lead is the conduit for the "why."
> - **UX** — user experience: how the product works for the humans using it.

## Concept

Engineering doesn't ship in isolation — it works with **product** (what to build and why) and
**design** (how it should work for users), forming a **triad**. The best products come from this
triad working as genuine partners with *healthy tension* — each bringing their perspective — not from
engineering being a passive "feature factory" that receives fully-specced tickets and builds them. The
key shift: engage at **problem definition**, not ticket receipt — be a partner in deciding *what* to
build, not just an executor of decisions already made.

```
   THE FEATURE FACTORY              THE HEALTHY TRIAD
   ──────────────────              ─────────────────
   PM decides WHAT →               PM (what/why) + Design (UX) +
   hands eng a spec →              Eng (feasibility/how) as PARTNERS
   eng builds it                   engage at PROBLEM DEFINITION,
   (eng = executor)                not ticket-receipt
        ↑ eng has no say,               ↑ healthy tension, each perspective,
          builds the wrong thing          ships the RIGHT thing
```

The reframe: **be a partner in the triad, engaging at problem definition — not a feature factory
receiving specs.** When engineering is only handed finished specs to build, it (a) can't contribute
its perspective (feasibility, technical opportunities, simpler alternatives), (b) often builds the
wrong thing (the spec solved the wrong problem), and (c) is disengaged. Engaging early — at the
problem, not the solution — lets engineering shape what gets built, catch problems early, and ship
the right thing.

---

## How It Works

### The triad and healthy tension

Product, design, and engineering form a **triad**, each with a distinct perspective: **product** owns
the *what* and *why* (the problem, the user need, the business value, priorities); **design** owns the
user experience (how it works for users); **engineering** owns feasibility and the *how* (what's
buildable, the technical approach, the costs and constraints). The best outcomes come from these three
in genuine partnership with **healthy tension** — each advocating their perspective, pushing on the
others, and reaching better decisions together than any would alone. The tension is *healthy* (it
surfaces trade-offs and better options); the goal isn't for one to dominate but for the three to
collaborate as equals.

### Engage at problem definition, not ticket receipt

The critical shift: engineering should engage at **problem definition** (what problem are we solving,
why, for whom) — <em>not</em> just receive a finished spec to build. When engineering is brought in
early (at the problem), it can (1) contribute technical perspective to the solution (feasibility,
opportunities, simpler alternatives, "we could solve that problem much more cheaply this way"), (2)
catch problems early (a spec that's technically expensive or solves the wrong thing), and (3) shape
what gets built. When engineering only receives finished tickets, all of that is lost — you build what
was decided without your input, often discovering too late that there was a better or simpler way. Push
to be in the room at problem definition, not just handed the conclusion.

### Push back with alternatives, not vetoes

When you disagree with product/design (a feature that's too expensive, solves the wrong problem, or
has a better approach), push back **with alternatives, not vetoes.** A veto ("we can't/won't build
that") is adversarial and unhelpful; an alternative ("that would take three months, but we could solve
the same user problem this simpler way in three weeks — would that work?") is collaborative and
constructive. Framing pushback as "here's a better way to achieve <em>your</em> goal" (their goal is
the user/business outcome, not the specific spec) keeps you a partner solving the shared problem, not
a blocker. Bring options, not refusals.

### Share the "why" downward

As the eng lead in the triad, you learn the *why* behind what you're building (the user problem, the
business goal, the priorities). **Share that "why" with your team** — engineers who understand why
they're building something make better decisions, are more motivated, and can contribute better ideas,
where engineers just handed tasks build blindly. Being the conduit for the product context (not just
the tasks) is part of the lead's role in the triad — you translate the "why" from product to your
engineers.

### When engineering should drive product ideas

The triad isn't one-directional (product decides, eng builds). **Engineering should also drive product
ideas** — engineers often see opportunities others can't (what's newly possible technically, where the
real user pain is from support/data, a simpler solution, a technical capability that enables a product
direction). A healthy triad welcomes engineering-originated product ideas, and a good eng lead
surfaces them (not just responds to product's). Engineering's technical perspective is a source of
product insight, not just execution capacity.

{: .note }
> **Be a partner in the triad, engaging at problem definition — not a feature factory</br>**
> The best products come from product (what/why), design (UX), and engineering (feasibility/how) as
> genuine partners with healthy tension — not from engineering as a passive feature factory receiving
> specs. The key shift: engage at <em>problem definition</em>, not ticket receipt — so engineering can
> contribute its perspective (feasibility, simpler alternatives), catch problems early, and shape the
> right thing (rather than building a finished spec that may solve the wrong problem). Push back with
> <em>alternatives, not vetoes</em> ("here's a simpler way to achieve your goal"), share the <em>why</em>
> with your team (so they build with context), and surface engineering-driven product ideas (eng sees
> opportunities others can't). Being a genuine triad partner — engaging early, collaborating on what to
> build — is how engineering ships the right thing, not just the thing it was told to build.

---

## Lab — Scenario

**The situation:** Your PM, Jordan, hands you a fully-specced feature — a detailed design for a complex
"advanced filtering" system for your product's dashboard. But as you read it, you realize it solves the
*wrong problem*: the spec assumes users need powerful filtering, but from what you've seen (support
tickets, usage data), users' actual pain is that they can't *find* the one thing they're looking for —
which a good search would solve far better and more simply than a complex filtering system. Jordan has
clearly put a lot of work into this spec and is expecting you to build it. You need to push back —
without damaging the relationship.

**Write how you'd push back** — the actual conversation. Then note the principles and mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong pushback engages at the problem (not the spec), brings an alternative (not a veto), respects
Jordan's work, and frames it as achieving Jordan's goal better. Example:
<br><br>
"Jordan, thanks for this — I can see you've put a lot of thought into it. Before we dive into building
it, I want to make sure we're solving the right problem, and I'd love your help thinking it through.
<em>(Respects the work, engages at the problem level, collaborative framing — not 'this is wrong.')</em>
<br><br>
Can I ask — what's the core user problem we're trying to solve with the advanced filtering? <em>(Goes to
the problem/why behind the spec — the problem-definition level.)</em> [Jordan: users are struggling to
work with their data / find things.] <br><br>
That's really helpful. Here's what's on my mind: from what I've been seeing in the support tickets and
the usage data, the main pain point users hit is that they can't <em>find</em> the specific thing they're
looking for — they're scrolling and hunting. Advanced filtering would help power users, but I'm not sure
it addresses that core 'I can't find my thing' pain for most users — and it's a pretty big build (a few
months). <em>(Brings the evidence — support/data — for the real problem; raises the concern that the spec
may not solve it; notes the cost.)</em>
<br><br>
What I'm wondering is — could we solve that core problem more directly and simply with a really good
search? I think we could ship that much faster and it might address the actual pain for way more users.
Could we look at the data together and see whether search or filtering better solves what users are
struggling with? <em>(Offers the alternative — search — framed as better achieving Jordan's goal (solving
the user problem), and proposes deciding it together with data, not vetoing.)</em>
<br><br>
I might be missing context on why filtering — totally open to that. I just want us to build the thing
that actually solves the user's problem." <em>(Humble, open to being wrong, reaffirms the shared goal —
the right thing for users.)</em>
<br><br>
<strong>Principles:</strong> (1) <strong>Engage at the problem, not the spec</strong> — ask what user
problem it solves, then discuss whether the spec solves it, rather than just critiquing the spec's
details. (2) <strong>Bring an alternative, not a veto</strong> — propose search as a better/simpler way
to achieve the goal, not "we shouldn't build this." (3) <strong>Frame it as achieving Jordan's goal</strong>
— Jordan wants to solve the user problem (not specifically to build filtering), so "here's a better way
to solve it" is helping, not blocking. (4) <strong>Bring evidence</strong> — support tickets/usage data
for the real problem (grounds it, not just opinion). (5) <strong>Respect Jordan's work and stay humble</strong>
— acknowledge the effort, be open to missing context (protects the relationship). (6) <strong>Propose
deciding together with data</strong> — collaborative, not a unilateral eng override. <strong>Mistakes to
avoid:</strong> (1) <strong>just building the wrong thing</strong> — avoiding the awkward conversation and
building a spec you think solves the wrong problem (wasteful, and not being a real partner); (2) <strong>vetoing</strong>
("this is wrong, we're not building it") — adversarial, damages the relationship, disrespects Jordan's
work; (3) <strong>critiquing spec details instead of the problem</strong> — engaging at the wrong level
(the fix is at problem-definition, not the spec's fields); (4) <strong>making Jordan wrong / not respecting
the work</strong> — "you solved the wrong problem" bluntly, ego-bruising; (5) <strong>being certain you're
right</strong> — maybe Jordan has context you lack (there was research showing filtering matters); stay
open; (6) <strong>going around Jordan</strong> (to Jordan's boss, or just deciding) — damages trust; work
it out as partners; (7) <strong>no alternative/evidence</strong> — just "I don't think this is right"
without the search proposal and the data (unconstructive).
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| The PM/design/eng triad | *Inspired*, Marty Cagan (product triads); *EMPOWERED*, Cagan & Jones |
| Feature factory vs product team | John Cutler on "feature factory" |
| Engaging at problem definition | *Inspired* (problem vs solution); Lesson 49 (should we build this?) |
| Pushing back well (the language) | English for Work track, Lesson 30 (disagreeing live); Lesson 42 |

---

## Checkpoint

**Q1.** What is the product/design/engineering triad with "healthy tension," and why should engineering
engage at "problem definition, not ticket receipt"?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The <strong>product/design/engineering triad</strong> is the collaboration of the three disciplines that
together build a product, each owning a distinct perspective: <strong>product</strong> owns the <em>what</em>
and <em>why</em> (the problem to solve, the user need, business value, priorities); <strong>design</strong>
owns the user experience (how it works for users); and <strong>engineering</strong> owns feasibility and
the <em>how</em> (what's buildable, the technical approach, costs and constraints). <strong>Healthy
tension</strong> means the three work as genuine partners who each advocate their perspective and push on
the others — the product person pushing for user/business value, the designer for user experience, the
engineer for feasibility and technical wisdom — reaching better decisions together than any one would
alone. The tension is <em>healthy</em> because it surfaces trade-offs and better options (each perspective
checks and improves the others): design's ideal UX is pressure-tested against engineering's feasibility,
product's scope against engineering's cost, etc. The goal isn't for one discipline to dominate (product
dictating, or engineering vetoing) but for the three to collaborate as equals, with their different
perspectives in productive tension producing the best product. Why engineering should engage at
<strong>problem definition, not ticket receipt</strong>: <strong>because engaging early (at what problem
to solve and why) lets engineering contribute its perspective, catch problems early, and shape the right
thing — whereas receiving finished specs makes engineering a passive executor that often builds the wrong
thing</strong>. When engineering is brought in at problem definition (the problem, the why, before the
solution is locked), it can: (1) <strong>contribute technical perspective to the solution</strong> —
feasibility insights, technical opportunities, and simpler alternatives ("we could solve that user problem
much more cheaply this way," "there's a technical approach that opens up a better solution"), which often
lead to a better or far simpler product than the solution designed without engineering input; (2)
<strong>catch problems early</strong> — a proposed solution that's technically very expensive, or that
solves the wrong problem, can be caught at the cheap stage (before it's fully specced and built), rather
than discovered late; and (3) <strong>shape what gets built</strong> — engineering's perspective
influences the actual product decisions, not just their execution. When engineering only <em>receives
finished tickets</em> (the feature factory model — product decides everything, hands eng a spec, eng
builds it), all of this is lost: engineering builds what was already decided without its input, so (a) the
technical perspective never informs the solution (missing simpler/better options, or building something
needlessly expensive), (b) problems are discovered too late (you build the spec, then realize it solved
the wrong problem or there was a much simpler way), and (c) engineering is a disengaged executor rather
than an invested partner. The scenario (a fully-specced feature solving the wrong problem) illustrates
exactly this: engineering, engaged only at ticket-receipt, is handed a solution to build that its
perspective (support/usage data showing the real problem is findability, solvable more simply by search)
would have improved — but by ticket-receipt time, that input comes too late and awkwardly. So engaging at
problem definition (being in the room when what-to-build is decided, contributing engineering's
perspective on the problem and solution) is how engineering helps build the <em>right</em> thing (not just
the thing it was told to build) — which is the difference between being a genuine triad partner and being
a feature factory. The underlying principle: engineering's technical perspective is valuable input to
<em>what</em> gets built (not just capacity to build it), and that value is only captured by engaging
early (problem definition) rather than late (ticket receipt).
</details>

**Q2.** Why is pushing back "with alternatives, not vetoes" more effective, and why should a lead share
the "why" with their team?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Pushing back "with alternatives, not vetoes" is more effective because <strong>an alternative is
collaborative (it helps achieve the shared goal a better way, keeping you a partner), while a veto is
adversarial (it just blocks, making you an obstacle) — and the alternative respects that the other person's
real goal is the outcome, not the specific spec</strong>. When you disagree with product/design (a feature
that's too expensive, solves the wrong problem, or has a better approach), you can respond two ways: (1)
<strong>a veto</strong> — "we can't/won't build that," a flat refusal — which is adversarial and unhelpful:
it blocks without offering a way forward, positions you as an obstacle to the PM's goals, damages the
relationship (especially if it disrespects their work), and leaves the actual problem unsolved; or (2)
<strong>an alternative</strong> — "that approach would take three months, but we could solve the same user
problem this simpler way in three weeks — would that work?" — which is collaborative and constructive: it
offers a way forward, keeps you a partner working toward the shared goal, and is far more likely to be
received well. The key insight is that <strong>the PM's/designer's real goal is the user/business outcome,
not the specific spec</strong> — they want to solve the user problem (or hit the business goal), and the
particular solution is just their current means to it. So framing your pushback as "here's a better way to
achieve <em>your</em> goal" (a better/cheaper/faster path to the outcome they care about) is genuinely
helping them, not opposing them — you're a partner finding a better route to the shared destination, not a
blocker refusing to go. This keeps the relationship healthy (you're collaborating, not fighting), gets to
a better outcome (the alternative may genuinely be better), and preserves your credibility as a
constructive partner. A veto, by contrast, forfeits all of this — it's a dead end that makes you an
adversary. So bring options, not refusals: pushback framed as alternatives toward the shared goal is
collaborative and effective, where vetoes are adversarial and damaging. Why a lead should <strong>share
the "why" with their team</strong>: <strong>because engineers who understand why they're building
something make better decisions, are more motivated, and contribute better ideas — where engineers just
handed tasks build blindly</strong>. As the eng lead in the triad, you learn the <em>why</em> behind the
work (the user problem, the business goal, the priorities, the context). Sharing that with your team
(rather than just passing down the tasks) has several benefits: (1) <strong>better decisions</strong> —
engineers who understand the goal and context make better implementation choices (they can weigh
trade-offs against the actual objective, choose approaches that serve the real need, and avoid decisions
that technically satisfy the ticket but miss the point) — understanding the why lets them exercise good
judgment, where just having the task means they might optimize for the wrong thing; (2) <strong>more
motivation</strong> — people are more engaged and motivated when they understand the purpose and value of
their work (it's meaningful, connected to real user/business impact) than when they're just executing
tasks blindly (which is demotivating — feature-factory drudgery); (3) <strong>better ideas</strong> —
engineers who understand the problem (not just the assigned solution) can contribute better ideas and
alternatives (they might see a simpler approach, spot an issue, or suggest something better) — the same
value engineering brings to the triad, but only if they understand the why. So the lead being the conduit
for the "why" (translating the product context down to the team, not just the tasks) makes the team more
effective, motivated, and creative — versus a lead who just distributes tickets, leaving engineers
building blindly without context. Both points reflect the lead's role as a genuine triad partner and
translator: push back constructively (alternatives toward the shared goal, keeping you a collaborative
partner) and share the context downward (the why, so the team builds with understanding) — which together
make engineering an engaged, effective partner in building the right thing, rather than an adversarial
blocker or a blind executor.
</details>

---

## Homework

Reflect on your work with product and design. (1) Are you a genuine triad partner or a feature factory —
do you engage at problem definition, or mostly receive finished specs? How could you get engaged earlier
(at the problem)? (2) When you disagree with product/design, do you push back with alternatives (better
ways to achieve their goal) or with vetoes/complaints? Practice reframing a pushback as an alternative. (3)
Do you share the "why" (the product context) with your team, or just pass down tasks? (4) Have you surfaced
any engineering-driven product ideas (opportunities eng sees that others can't)? Reflect: is your
engineering a real partner in deciding what to build, or an executor — and what's one way to engage earlier
and more as a partner?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the triad-partnership skill — working with product and design to ship the right thing.
A strong response assesses whether they're a genuine partner or a feature factory (engaging at problem
definition vs. receiving specs), whether they push back with alternatives or vetoes, whether they share the
why with their team, and whether they surface engineering-driven product ideas. The realizations: (1)
<strong>be a partner engaging at problem definition, not a feature factory</strong> — engaging early (at
the problem/why) lets engineering contribute its perspective, catch problems early, and shape the right
thing, where receiving finished specs makes eng a passive executor that often builds the wrong thing; (2)
<strong>push back with alternatives, not vetoes</strong> — an alternative ("here's a better way to achieve
your goal") is collaborative and keeps you a partner, where a veto is adversarial and blocks; the other
person's real goal is the outcome, not the spec, so a better path to the outcome helps them; (3)
<strong>share the why with your team</strong> — engineers who understand the purpose make better decisions,
are more motivated, and contribute better ideas, where blind task-executors build without judgment; (4)
<strong>surface engineering-driven product ideas</strong> — eng sees opportunities others can't (what's
newly possible, real user pain from data/support), so a healthy triad welcomes them. On reflection, people
commonly find they operate more as a feature factory than they'd like (receiving specs rather than engaging
at the problem), that they sometimes push back as complaints/vetoes rather than alternatives, and that they
don't always share the why with their team. The highest-value change is usually: engage earlier (get into
problem-definition, not just ticket-receipt), and push back with alternatives framed as better ways to
achieve the shared goal. The meta-point: the best products come from product, design, and engineering as
genuine partners with healthy tension, not from engineering as a feature factory — and the key is engaging
at problem definition (contributing eng's perspective to what gets built), pushing back constructively
(alternatives, not vetoes), sharing the why downward, and surfacing eng-driven product ideas. This makes
engineering a real partner in shipping the right thing, which requires influence and collaboration skills
(the phase's theme) applied to the closest cross-functional relationships. The next lesson turns to a
different critical stakeholder — your own manager — and managing up so they're effective on your behalf and
never surprised.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 41 — Managing Up →](lesson-41-managing-up){: .btn .btn-primary }
