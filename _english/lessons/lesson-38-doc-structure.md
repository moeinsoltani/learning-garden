---
title: "Lesson 38 — Document Structure"
nav_order: 1
parent: "Phase 7: Design Docs & Proposals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 38: Document Structure

## Concept

A design doc's job is to get an idea understood and a decision made — and *structure* is what
makes that happen. A well-structured doc lets a reader grasp the problem, the proposal, and
the trade-offs quickly, and find the part they care about without reading everything. A
structureless wall of text buries a good idea and exhausts reviewers. The good news:
engineering docs follow a **reusable skeleton** you can lean on every time.

```
   THE DESIGN DOC SKELETON:
   ┌────────────────────────────────────────────────────┐
   │ 1. SUMMARY / TL;DR   what & why, in a few lines     │
   │ 2. CONTEXT / PROBLEM what's the situation & problem  │
   │ 3. GOALS / NON-GOALS what we are (and aren't) solving│
   │ 4. PROPOSAL          the design / approach           │
   │ 5. ALTERNATIVES      what else we considered & why not│
   │ 6. RISKS / TRADE-OFFS what could go wrong / costs     │
   │ 7. (rollout, open questions, appendix)               │
   └────────────────────────────────────────────────────┘
```

The reframe: **you don't invent structure from scratch — you fill in a known skeleton.** The
sections above answer the questions any reviewer has (what, why, what are we solving, how,
what else did you consider, what's the risk). Following the skeleton makes your doc complete
and scannable, and frees you to focus on the content instead of the shape.

---

## How It Works

### Start with a summary / TL;DR

Open with a few lines stating **what you're proposing and why** — the whole doc in miniature.
Busy readers (especially senior ones) read the summary and decide whether to read on or trust
it; it also frames the rest so detailed sections make sense. This is lead-with-the-point
(Lesson 21) at document scale. Never make a reader get to page 3 to learn what the doc is
about.

### Context / problem before solution

Before the proposal, establish the **problem and context**: what's the current situation, why
is it a problem, why now. Readers need the problem framed before the solution lands (a solution
to an unclear problem is confusing). Give just enough context for someone reasonably informed
to follow — link out for deep background rather than including it all.

### Goals and non-goals

State explicitly what you're solving (**goals**) and, crucially, what you're *not* solving
(**non-goals**). Non-goals prevent scope confusion and head off "but what about X?" — you've
declared X out of scope on purpose. This small section saves enormous review churn.

### The proposal — your design

The core: the approach/design you're recommending, in enough detail to evaluate. Use structure
here too — subheadings, diagrams, bullet points for components. Explain not just *what* but
*why* (the reasoning behind key choices), since reviewers evaluate the reasoning, not just the
conclusion.

### Alternatives considered

Show the options you weighed and why you didn't pick them. This is one of the most valuable
sections: it proves you did the thinking (building trust in your recommendation), pre-answers
"did you consider X?", and helps reviewers evaluate the decision. A proposal with no
alternatives looks under-considered.

### Risks / trade-offs

State honestly what could go wrong and what the costs are. Acknowledging trade-offs and risks
builds credibility (it shows you see clearly, not just selling) and invites the reviewers to
help mitigate them. Hiding the downsides backfires — reviewers find them anyway, and now they
distrust the rest.

### Make it scannable

Use headings, short paragraphs, bullets, and diagrams so readers can navigate to what they
care about. A reviewer should be able to skim the headings and find the section they need.
Dense unbroken text forces linear reading and hides structure; scannable formatting respects
the reader's time and different reading needs (some skim, some read deeply).

{: .note }
> **You fill in a known skeleton, not invent structure from scratch**
> Engineering design docs follow a reusable skeleton — summary, context/problem, goals/
> non-goals, proposal, alternatives, risks/trade-offs — that answers the questions any
> reviewer has. Leaning on it makes your doc complete (you won't forget the alternatives or
> the non-goals) and scannable (readers navigate by the familiar sections), and it frees you
> to focus on content rather than shape. Lead with a summary (the point up front), frame the
> problem before the solution, declare non-goals to prevent scope creep, show your
> alternatives and risks honestly (which builds trust), and format for scanning. A
> well-structured doc gets a good idea understood and approved; a structureless one buries it.
> For a lead, writing clear docs is a defining, high-leverage skill.

---

## Lab — Structure / Rewrite Drill

Work each structure problem.

**Your answer:**

1. **No summary:** A design doc opens with three paragraphs of background about the history of
   the caching system before ever saying what it proposes. What's wrong, and what should the
   opening be?
2. **Missing non-goals:** A proposal to add a Redis cache keeps getting comments like "what
   about caching the user service too?" and "should this also handle session storage?" How
   would a non-goals section help, and what might it say?
3. **Outline a doc:** You need to propose migrating a service from a monolith to its own
   deployable. Sketch the section headings you'd use (the skeleton, adapted).

<details>
<summary>Show Model Answers</summary>
<br>
<strong>1. No summary — what's wrong + fix:</strong> The problem: the reader gets three
paragraphs in without knowing what the doc is even proposing, so they can't tell what the
background is <em>for</em> or whether the doc is relevant to them — the point is buried. Fix:
open with a summary/TL;DR — "<strong>Summary:</strong> This doc proposes adding a Redis cache
in front of our hot database queries to fix checkout slowdowns under load. It's a one-sprint
change expected to cut p99 latency significantly. Below: the problem, the design, alternatives
considered, and risks." Then the background follows, now framed by the point. (Lead with what
and why; put the history later or link it.)
<br><br>
<strong>2. Non-goals section:</strong> A non-goals section explicitly declares what's out of
scope, heading off the scope-expansion comments. It might say: "<strong>Non-goals:</strong>
This proposal covers <em>only</em> caching the product-catalog queries that slow checkout.
Caching the user service, session storage, and other services are explicitly out of scope for
this change — they can be considered separately later. This keeps the change small and
measurable." Now "what about the user service?" is pre-answered (out of scope, on purpose),
saving the review churn. (Non-goals prevent the endless scope-creep comments.)
<br><br>
<strong>3. Outline (skeleton adapted):</strong><br>
• <strong>Summary</strong> — extract Payments into its own deployable service; why, and the
expected benefit.<br>
• <strong>Context / Problem</strong> — Payments in the monolith causes [coupling, deploy risk,
scaling limits].<br>
• <strong>Goals</strong> — independent deploys, isolated scaling, clear ownership.<br>
• <strong>Non-goals</strong> — not re-architecting the payment logic; not touching other
modules; not changing the DB yet.<br>
• <strong>Proposal</strong> — the target architecture, the API boundary, data ownership, the
migration steps.<br>
• <strong>Alternatives considered</strong> — keep in monolith with modularization; strangler
pattern vs big-bang; why the chosen path.<br>
• <strong>Risks / Trade-offs</strong> — split-brain data, added network hop/latency, operational
overhead, migration risk — and mitigations.<br>
• <strong>Rollout plan</strong> — phased steps, how to validate, rollback.<br>
• <strong>Open questions</strong> — the undecided bits for reviewers to weigh in on.<br>
(The familiar skeleton, adapted to the topic — complete and scannable.)
<br><br>
The patterns: open with a summary (point up front); frame the problem before the solution;
declare non-goals to prevent scope creep; and follow the reusable skeleton (summary → context →
goals/non-goals → proposal → alternatives → risks → rollout/open questions).
</details>

---

## Phrase Bank

| Section | Opening phrase |
|---|---|
| Summary | "This doc proposes [X] to [solve Y]. In short: …" |
| Context | "Today, [current situation]. The problem is [Z], because …" |
| Goals | "We aim to: [1], [2], [3]." |
| Non-goals | "Out of scope: [A], [B] — these can be addressed separately." |
| Proposal | "We propose [approach]. The key components are …" |
| Alternatives | "We also considered [X], but chose against it because …" |
| Risks | "The main risks are [R]; we'd mitigate by …" |
| Open questions | "Still to decide: [Q1], [Q2] — input welcome." |

---

## Further Reading

| Topic | Source |
|---|---|
| Design docs | Leadership track Lesson 16 (design docs) |
| Design doc culture | [Google's approach to design docs](https://www.industrialempathy.com/posts/design-docs-at-google/) |
| Lead with the point | English track Lesson 21 (message shapes) |

---

## Checkpoint

**Q1.** What is the reusable design-doc skeleton, and why does following it (rather than
inventing structure each time) produce better docs?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The reusable design-doc skeleton is a standard set of sections that answer the questions any
reviewer has: (1) <strong>Summary / TL;DR</strong> — what you're proposing and why, in a few
lines (the whole doc in miniature); (2) <strong>Context / Problem</strong> — the current
situation, what's wrong, and why now; (3) <strong>Goals / Non-goals</strong> — what you're
solving and, explicitly, what you're <em>not</em> solving; (4) <strong>Proposal</strong> — the
design/approach you recommend, in enough detail to evaluate, with the reasoning behind key
choices; (5) <strong>Alternatives considered</strong> — the options you weighed and why you
didn't pick them; (6) <strong>Risks / Trade-offs</strong> — what could go wrong and what the
costs are; plus (7) optional sections like rollout plan, open questions, and appendix.
Following this skeleton produces better docs than inventing structure each time for several
reasons: (1) <strong>completeness</strong> — the skeleton is a checklist that ensures you cover
everything reviewers need; inventing structure freehand, you're likely to forget important
sections (the alternatives, the non-goals, the risks), leaving gaps that generate review churn
("did you consider X?", "what's out of scope?", "what are the risks?"). The skeleton prompts
you to include them. (2) <strong>scannability / familiarity</strong> — because the skeleton is
standard, reviewers know where to find what they care about (they can jump to "alternatives" or
"risks" directly); a doc with idiosyncratic or no structure forces linear reading and makes
navigation hard. The familiar sections let readers orient instantly. (3) <strong>it answers the
right questions in the right order</strong> — the skeleton reflects how a reviewer evaluates a
proposal: they need the point (summary), then the problem framed (context), then the scope
(goals/non-goals), then the solution (proposal), then evidence you considered options
(alternatives) and see clearly (risks). This order makes the doc logical and persuasive —
problem before solution, reasoning before conclusion. (4) <strong>it frees you to focus on
content</strong> — not having to figure out the shape each time, you spend your energy on the
substance (the actual design, the trade-offs) rather than on organizing, and you avoid the
blank-page problem (the skeleton gives you a starting frame to fill in). (5) <strong>it builds
credibility</strong> — a doc that follows the expected structure (with real alternatives and
honest risks) signals you did thorough thinking, which makes reviewers trust your
recommendation. So the skeleton isn't a constraint; it's a tool that makes your docs complete,
scannable, logical, and credible, while making them easier to write. The reframe — "you fill in
a known skeleton, not invent structure from scratch" — is freeing: you always have a reliable
frame to start from, so writing a design doc becomes filling in familiar sections rather than
facing a blank page and hoping you covered everything. For a lead, whose ideas get evaluated
through docs, this reliably-good structure is high-leverage: it's the difference between a doc
that gets your idea understood and approved and one that buries it in disorganization.
</details>

**Q2.** Why do the "alternatives considered" and "risks/trade-offs" sections — which might
seem to weaken your proposal by airing downsides — actually make it more credible and more
likely to be approved?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The "alternatives considered" and "risks/trade-offs" sections make a proposal <em>more</em>
credible and more likely to be approved, despite airing downsides, because <strong>they
demonstrate thorough, honest thinking — which is exactly what makes reviewers trust your
recommendation and feel safe approving it</strong>. It seems counterintuitive: why show the
other options and admit what could go wrong, when you're trying to sell your proposal? But the
psychology of a reviewer is the key. On <strong>alternatives considered</strong>: a reviewer
evaluating a proposal naturally wonders "is this the best option, or just the first one they
thought of? Did they consider X?" A proposal that presents only the chosen solution, with no
alternatives, looks <em>under-considered</em> — the reviewer can't tell if you weighed options
or just went with a hunch, so they either distrust it or have to raise all the alternatives
themselves (generating churn: "did you think about Y?"). By showing the alternatives and why
you rejected them, you (a) prove you did the thinking — you explored the space and made a
reasoned choice, which builds confidence in the recommendation; (b) pre-answer the "did you
consider X?" questions, saving review cycles; and (c) help reviewers evaluate the decision (they
can see the trade-offs you weighed and agree or push back on specific reasoning, rather than
starting from scratch). So alternatives make your chosen option look like the <em>considered
best</em>, not a random pick — which is more persuasive, not less. On <strong>risks/trade-offs</strong>:
admitting what could go wrong seems to hand reviewers ammunition, but hiding the downsides
backfires badly. Reviewers are smart; they'll spot the risks and costs anyway — and if <em>you</em>
didn't mention them, one of two bad things happens: either they think you didn't see the risks
(you look naive, careless), or they suspect you saw them and hid them (you look like you're
selling, not leveling — and now they distrust the <em>whole</em> doc, wondering what else you
glossed over). Either way, unaddressed risks that reviewers discover themselves destroy
credibility. By stating the risks and trade-offs honestly (with mitigations), you (a) show you
see the situation clearly and completely — the mark of sound judgment, which builds trust; (b)
demonstrate integrity — you're giving them the real picture, not a sales pitch, so they can
trust your assessment; and (c) invite collaboration — reviewers can help mitigate risks you've
surfaced, turning them from objections into shared problem-solving. Crucially, <strong>a proposal
with honestly-stated risks is easier to approve than one with hidden ones</strong>, because
approvers need to feel the decision is safe and well-understood; a doc that transparently lays
out the risks and mitigations lets them approve with open eyes, whereas a too-good-to-be-true
doc with no downsides makes them nervous (what's being hidden?) and inclined to dig or delay. The
underlying principle: <strong>credibility comes from demonstrated thoroughness and honesty, not
from making your proposal look flawless</strong> — and reviewers trust (and approve) a clear-eyed
proposal that shows its work and owns its trade-offs far more than a one-sided sales pitch. So
airing the alternatives and risks, counterintuitively, strengthens the proposal: it makes you
look thorough and trustworthy, pre-empts objections, invites help, and lets approvers say yes
with confidence. For a lead, this honest, thorough style is also reputation-building — you become
known as someone whose proposals can be trusted, which makes future ones easier to approve.
</details>

---

## Homework

For your next design doc or proposal (or rewrite an old one), use the skeleton: open with a
summary (what + why), frame the problem before the solution, add explicit goals AND non-goals,
present the proposal with reasoning, include an alternatives-considered section, and state risks/
trade-offs honestly with mitigations. Format it to be scannable (headings, short paragraphs,
bullets, a diagram). Notice how the skeleton makes it easier to write and more complete. Add to
your habits: "summary up front? problem before solution? non-goals? alternatives? honest risks?
scannable?"

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the design-doc structure skill — a defining, high-leverage skill for a lead,
whose ideas get evaluated and approved through docs. A strong response uses the reusable skeleton
(summary → context/problem → goals/non-goals → proposal with reasoning → alternatives → risks/
trade-offs → rollout/open questions), formats for scanning, and notices that the skeleton makes
the doc both easier to write and more complete. The realizations: (1) <strong>you fill in a known
skeleton, not invent structure</strong> — the standard sections answer every reviewer's questions
and act as a completeness checklist, so you won't forget the alternatives or non-goals, and you
avoid the blank-page problem; (2) <strong>lead with a summary and frame the problem before the
solution</strong> — readers need the point up front and the problem framed before the design
lands; (3) <strong>non-goals prevent scope creep</strong> — declaring what's out of scope
pre-empts the "what about X?" churn; (4) <strong>airing alternatives and risks builds
credibility</strong> — showing your work and owning trade-offs makes reviewers trust the
recommendation and approve with confidence, where a flawless-looking one-sided doc makes them
nervous. The meta-point: structure is what makes a design doc do its job (get an idea understood
and approved) — a well-structured doc lets reviewers grasp the problem, proposal, and trade-offs
quickly and navigate to what they care about, while a structureless wall of text buries a good
idea and exhausts reviewers. The skeleton makes reliably-good structure achievable every time,
and the honest, thorough style (real alternatives, honest risks) builds the credibility that gets
proposals approved and builds your reputation. Add the structure checks to your habits. This
begins Phase 7 (Design Docs & Proposals), longer-form writing built on your foundation. The next
lesson covers plain language — writing to be understood, not to sound impressive, which is what
actually makes a doc (and you) credible.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 39 — Plain Language →](lesson-39-plain-language){: .btn .btn-primary }
