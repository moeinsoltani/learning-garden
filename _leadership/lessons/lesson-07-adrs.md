---
title: "Lesson 07 — Architecture Decisions and ADRs"
nav_order: 2
parent: "Phase 2: Technical Leadership"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 07: Architecture Decisions and ADRs

## Concept

Big technical decisions get made in meetings, Slack threads, and someone's head —
then six months later nobody remembers *why*, someone "fixes" the weird choice
that was actually deliberate, and the reasoning is lost. An **Architecture
Decision Record (ADR)** fixes this: a short, durable document capturing one
significant decision, its context, and its consequences.

```
   ADR structure (one per significant decision):
   ┌────────────────────────────────────────────────┐
   │ TITLE:  short, e.g. "Use Postgres, not Mongo"   │
   │ STATUS: proposed / accepted / superseded        │
   │ CONTEXT: what forces are at play? the problem,  │
   │          constraints, requirements — the "why    │
   │          are we deciding this"                    │
   │ DECISION: what we chose, stated plainly          │
   │ CONSEQUENCES: what follows — good AND bad; what  │
   │          becomes easier, what we're accepting    │
   └────────────────────────────────────────────────┘
```

The value isn't bureaucracy — it's **legibility over time**. An ADR lets a future
engineer (often future-you) understand a decision without archaeology, prevents
re-litigating settled questions, and — because writing the consequences forces
you to think them through — improves the decision itself.

A crucial framing that shapes *how much* deliberation a decision deserves: Jeff
Bezos's **one-way vs two-way doors**. A *two-way door* is reversible (you can walk
back through if it's wrong) — decide fast, don't over-deliberate. A *one-way door*
is hard or impossible to reverse (a public API, a data model everything depends
on, a framework choice that's expensive to unwind) — slow down, get it right,
write the ADR. Matching deliberation to reversibility is a core leadership skill.

---

## How It Works

### Reversible vs one-way doors

Most decisions are reversible, and teams waste enormous energy deliberating them
as if they weren't (endless debate over a choice you could change next week). The
lead's job is to identify which kind a decision is: for two-way doors, push for a
fast decision and move on (the cost of being wrong is small — you can reverse);
for one-way doors, invest the deliberation, get more eyes, write the careful ADR
(the cost of being wrong is large and lasting). Miscategorizing wastes time
(over-deliberating reversible things) or causes disasters (rushing irreversible
ones). Ask: "if this is wrong, how expensive is it to change?" — the answer sets
the deliberation budget.

### Deciding when to decide

A subtle skill: not every decision should be made *now*. Sometimes the
responsible move is to *defer* — keep options open until you have more
information, or make a reversible interim choice while you learn. But deferring
has a cost too (the team is blocked or building on uncertainty), so it's a
judgment: decide now if the team needs to move and the decision is reversible or
clear; defer if deciding now means guessing on a one-way door you could
de-risk by waiting. The anti-patterns are both directions: chronic indecision
(the team stalls waiting for a call that never comes) and premature commitment
(locking in a one-way door before you had to).

### Disagree and commit

Not everyone will agree with every decision, and requiring consensus on
everything is paralysis. **Disagree and commit** (Amazon's principle, and Intel's
before it): once a decision is made — with genuine input heard — everyone
executes it fully, even those who argued against it, rather than half-heartedly
undermining it. The lead's responsibility is to make the disagreement genuinely
welcome *before* the decision (people must feel heard, or commit becomes
resentment) and the commit genuinely expected *after* (no relitigating in the
hallway, no passive sabotage). The ADR helps: it records that the decision was
made, the reasoning, and — good practice — the alternatives considered and why
they lost, so dissenters see their view was understood, not ignored.

{: .note }
> **The alternatives-considered section is underrated**
> The most valuable part of an ADR is often "options we considered and rejected,
> and why." It shows the decision wasn't arbitrary, prevents someone later
> "discovering" an option you already ruled out (and re-opening the debate), and
> makes dissenters feel heard (their preferred option is there, with an honest
> reason it lost). A decision that lists only the choice looks like a decree; one
> that shows the roads not taken looks like judgment — and reads far better to
> the team and to future engineers.

---

## Lab — Scenario

**The situation:** Think of a significant architecture decision from your own past
— one your team made (or you made) that had real consequences: a database choice,
a framework, a service boundary, a build-vs-buy call, a major refactor approach.
Chances are it was decided in conversations and never written down.

**Write the ADR for it, retrospectively:** title, status, context (the forces and
constraints at the time), the decision, the alternatives considered and why they
lost, and the consequences (both the good and the costs you accepted). Then
reflect: was it a one-way or two-way door, and did the team's deliberation match
that?

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
There's no single right ADR — the value is in producing a real one and noticing
what the exercise surfaces. A strong response has all the parts done well, and a
useful reflection. What good looks like:
<br><br>
<strong>Context</strong> that captures the <em>forces at the time</em>, not
just hindsight — the actual constraints, requirements, team situation, and
pressures that made this a real decision (e.g., "we needed to ship in 6 weeks, had
no ops team, expected moderate scale, and the team knew Postgres well"). This is
the part that makes the decision legible later: without it, a future engineer
sees the choice but not the reasoning, and might "fix" it not realizing the
constraints that drove it. <strong>Decision</strong> stated plainly (what you
chose). <strong>Alternatives considered</strong> — the load-bearing section: the
real options you weighed and an honest reason each lost ("we considered Mongo for
the flexible schema, but rejected it because our data was relational and the team
had no operational experience with it; we considered a managed service but it
exceeded budget"). <strong>Consequences</strong> — and critically, the
<em>costs</em> you accepted, not just the benefits ("this makes X easy and Y hard;
we're accepting that migrations will be more involved; we're betting scale stays
moderate — if it 10x's, we'll need to revisit"). An ADR that lists only upsides is
dishonest and less useful.
<br><br>
<strong>The reflection</strong> is where the learning is. Two common findings: (1)
Many teams discover, writing it retrospectively, that they <em>never actually
articulated the alternatives or the costs</em> at the time — the decision was made
on intuition and momentum, and forcing the ADR structure retroactively reveals
gaps in the original reasoning (which is exactly why writing ADRs <em>prospectively</em>
improves decisions — the act of filling in "alternatives" and "consequences"
forces thinking that intuition skips). (2) The one-way/two-way reflection often
reveals a mismatch: either the team agonized for weeks over a reversible choice
(over-deliberated a two-way door — wasted energy on something they could have
changed), or rushed a genuinely irreversible one (a data model or public API that
turned out very expensive to change, decided in an afternoon without the care a
one-way door deserves). A good reflection names which it was and whether the
deliberation matched — building the instinct to ask, going forward, "how
expensive is this to reverse?" before deciding how much to deliberate. The
meta-payoff: doing this once retrospectively usually converts people into ADR
believers, because they viscerally feel both the value (this decision's reasoning
was <em>lost</em> and I had to reconstruct it) and the decision-improving effect
(the structure surfaced considerations the original decision skipped).
<br><br>
Common mistakes: (1) writing only the decision and its benefits, omitting the
alternatives and the accepted costs (turns an ADR into a decree, loses most of the
value); (2) writing context as hindsight ("we should have known...") rather than
the forces that actually existed at the time (the point is to capture the
<em>then</em>-reasoning); (3) skipping the reversibility reflection, missing the
lesson about matching deliberation to door-type.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| ADRs — the original format | Michael Nygard — <https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions> |
| One-way vs two-way doors | Jeff Bezos — Amazon shareholder letters |
| Disagree and commit | *The Amazon Way*; Intel's origin (Andy Grove) |
| Lightweight ADR tooling & templates | adr.github.io — <https://adr.github.io/> |
| Decision-making for engineering leaders | *The Manager's Path*, Camille Fournier |

---

## Checkpoint

**Q1.** Explain one-way vs two-way doors and how the distinction should change how
much a team deliberates a decision. Give an example of each miscategorization
going wrong.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A <strong>two-way door</strong> is a reversible decision — if it turns out wrong,
you can walk back through and change it at modest cost (a library choice you could
swap, a reversible config, an internal API you control and can refactor). A
<strong>one-way door</strong> is hard or impossible to reverse — a public API
other teams/customers now depend on, a data model everything is built around, a
choice that's expensive or destructive to unwind. The distinction should set the
<em>deliberation budget</em>: two-way doors deserve fast decisions (the cost of
being wrong is small — you can reverse, so spending weeks deliberating wastes far
more than a wrong-but-reversible choice would); one-way doors deserve investment
(more eyes, careful analysis, an ADR, maybe prototyping — because the cost of
being wrong is large and lasting). The question that sets it: "if this is wrong,
how expensive is it to change?" Miscategorization going wrong in both directions:
<em>treating a two-way door as one-way</em> — a team spends three weeks debating
which of two logging libraries to use, holding meetings, writing comparison docs,
delaying the actual work — when they could have picked one in an hour, and if it
were wrong, swapped it in a day (weeks of energy wasted deliberating something
reversible; the delay cost more than any wrong choice would have). <em>Treating a
one-way door as two-way</em> — a team rushes a decision on their core data model
or a public API shape in an afternoon to "just ship it," and eighteen months later
that model is baked into a hundred places and thousands of customer integrations,
so fixing the flaw that a bit more deliberation would have caught now costs a
multi-quarter migration (or is simply lived with forever). The second is far more
dangerous — under-deliberating an irreversible decision creates lasting damage,
while over-deliberating a reversible one only wastes time. So the lead's instinct
should be: default to <em>fast</em> for the many reversible decisions (push the
team not to over-agonize — "just pick one, we can change it"), and <em>slow and
careful</em> for the few irreversible ones (resist the pressure to rush — "this is
a one-way door, it's worth getting right"). Correctly sorting decisions into these
buckets, and setting the deliberation accordingly, is a high-leverage skill: it
both speeds up the team (stops wasting energy on reversible choices) and prevents
disasters (invests appropriately in the irreversible ones).
</details>

**Q2.** "Disagree and commit" is easy to state and hard to do well. What must a
lead do *before* a decision and *after* it for the principle to work — and what
goes wrong if either half is missing?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Before the decision</strong>, the lead must make the disagreement
<em>genuinely welcome and genuinely heard</em>: create real space for dissent,
solicit the objections actively (especially from those less likely to speak up),
and demonstrably <em>consider</em> them — ideally recording the alternatives and
why they lost (the ADR section) so dissenters can see their view was understood,
not dismissed. The "disagree" half is real: people need to feel they had a fair
shot at influencing the outcome. <strong>After the decision</strong>, the lead
must make the commit <em>genuinely expected and modeled</em>: once the call is
made (with input heard), everyone — including those who argued against it —
executes it fully and in good faith; no relitigating in hallways, no
passive-aggressive foot-dragging, no "well I said this wouldn't work" when it
hits a bump. The lead enforces this norm (and models it when <em>they</em> were
overruled by their own boss). If the <strong>"disagree" half is missing</strong>
(the decision was made without genuinely hearing objections, or people didn't
feel heard), then "commit" becomes <em>resentful compliance</em> or quiet
sabotage — people execute half-heartedly, withhold their best effort, or wait for
it to fail so they can say "told you so," because they experience the decision as
imposed rather than made after fair consideration. Forced commitment without real
voice breeds exactly the disengagement it's trying to prevent. If the
<strong>"commit" half is missing</strong> (dissent continues after the decision —
people relitigate, undermine, or execute reluctantly), then the team is
<em>paralyzed by endless re-debate</em> and decisions never stick — every settled
question re-opens, nothing gets real momentum, and the cost of the ongoing
conflict exceeds any benefit of the "right" decision (a good decision executed
half-heartedly by a divided team beats a perfect decision nobody commits to). The
principle only works with <em>both</em> halves: robust disagreement <em>before</em>
(so people are heard and the decision is well-informed) and full commitment
<em>after</em> (so the team can actually move). The lead's craft is running that
sequence deliberately — really opening the floor to dissent and showing it
mattered, then clearly closing the debate and expecting unified execution — and
the ADR supports both by recording that input was considered and that a decision
was made, giving the team a durable "this is settled, here's why" to point to.
Getting this right is what lets a team make decisions <em>and</em> stay
cohesive — the alternative failures are decision-by-decree (fast but breeds
resentment) or decision-by-consensus (inclusive but paralyzed), and disagree-and-
commit is the path between them that requires the lead to do both halves well.
</details>

---

## Homework

Adopt ADRs for your team (or, if you can't yet, draft the practice): write ADRs
for the next two significant decisions your team faces — *prospectively*, before
deciding, so the act of filling in "alternatives considered" and "consequences"
shapes the decision itself. For each, first classify it as a one-way or two-way
door and set your deliberation accordingly. Then reflect: did writing the ADR
change the decision or surface a consideration you'd have missed? Did classifying
the door-type change how much time you spent? And how would you introduce the
practice to the team without it feeling like bureaucracy?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise's payoff is experiencing ADRs as a <em>thinking tool</em>, not
documentation overhead. A strong response reports the decision-improving effect:
writing "alternatives considered" prospectively forces you to actually enumerate
and evaluate options you might have skipped on intuition (revealing an option you
hadn't seriously weighed, or a reason your favored choice is weaker than you
thought), and writing "consequences" forces you to think through the costs and
downstream effects <em>before</em> committing rather than discovering them later —
so the ADR structure often changes or sharpens the decision, which is its most
underrated value (the artifact is a byproduct; the improved thinking is the
point). The door-type classification should visibly change your process: for the
two-way door, you deliberately move fast ("this is reversible, I'm not going to
agonize — pick, note why, move on") and notice you saved time you'd otherwise have
spent over-deliberating; for the one-way door, you invest more (more input,
prototyping, careful consequences) and feel justified doing so rather than
rushing. On <strong>introducing it without bureaucracy</strong>: the key is
keeping ADRs <em>lightweight and valuable</em>, not heavy and mandatory — a good
introduction (1) makes them short (one page, a simple template — a heavy multi-
page process guarantees resentment and abandonment); (2) reserves them for
genuinely <em>significant</em> decisions (one-way doors, choices with lasting
consequences — not every library pick, which would make them noise); (3) sells
them by the pain they prevent, ideally by having the team feel it — e.g., the
retrospective ADR from the lab, or the next time someone asks "why did we do it
this way?" and nobody remembers, point out an ADR would have answered it; (4)
models it yourself first (write a few, show they're quick and useful) rather than
mandating it top-down; (5) stores them where they're found (in the repo, next to
the code) not in a wiki nobody reads. The framing that lands: "this isn't
paperwork, it's so that in six months we (or a new teammate) can understand this
decision without archaeology, and so we don't keep re-debating settled things —
and honestly, writing it makes us think the decision through better." Teams
resist bureaucracy but appreciate tools that save them future pain, so introduce
ADRs as the latter (a small investment that prevents lost-reasoning and
re-litigation) and keep them light enough that the value clearly exceeds the cost.
If the practice starts feeling heavy or perfunctory (people writing ADRs to check
a box, not to think), that's a signal to trim it back — the goal is better
decisions and preserved reasoning, and the moment the ritual outweighs that
benefit it's failing regardless of how thorough it looks.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 8 — Evaluating Trade-offs →](lesson-08-tradeoffs){: .btn .btn-primary }
