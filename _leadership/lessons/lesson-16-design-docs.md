---
title: "Lesson 16 — Writing Design Documents and Proposals"
nav_order: 4
parent: "Phase 3: Communication Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 16: Writing Design Documents and Proposals

## Concept

Design docs and proposals are how significant technical and organizational
decisions get made in most engineering orgs — and a lead writes and reviews many
of them. A good doc gets read, gets useful feedback, and gets to a decision; a bad
doc is a wall of text nobody finishes, that generates bikeshedding instead of
substantive review, and that never reaches a clear outcome.

```
   A design doc's job: get to a good DECISION, efficiently.
   ┌──────────────────────────────────────────────────────┐
   │ CONTEXT     — what's the situation, why are we here?  │
   │ PROBLEM     — what exactly are we solving?            │
   │ OPTIONS     — what did we consider? (builds trust)    │
   │ RECOMMEND'N — what we propose, and why                │
   │ RISKS       — what could go wrong, what we're unsure  │
   │ + a one-paragraph SUMMARY at the very top (BLUF)      │
   └──────────────────────────────────────────────────────┘
   written for SKIMMERS: headings carry the argument, so
   someone reading only the headings still gets the story
```

The reusable structure — context → problem → options → recommendation → risks,
with a summary up top — works for almost any design doc or proposal because it
follows how a reader needs to reason: understand the situation, know what's being
solved, see the alternatives were considered, get the recommendation with its
reasoning, and understand the risks. The English-track Phase 7 develops the
*writing* craft; this lesson is about the *structure and purpose* — writing docs
that drive decisions.

---

## How It Works

### The structure follows the reader's reasoning

Each section answers a question the reader has, in the order they have it:
**Context** (why are we even discussing this? what's the background?) grounds the
reader; **Problem** (what precisely are we solving?) — often the most-skipped and
most-important section, because a doc that jumps to solutions without crisply
stating the problem invites solutions to the wrong problem; **Options considered**
(what alternatives did we weigh, and why did they lose?) — this builds trust
(shows the recommendation isn't arbitrary), prevents someone "discovering" an
option you already ruled out, and lets reviewers engage with the real trade-offs
(Lesson 8); **Recommendation** (what we propose, with reasoning); **Risks and open
questions** (what could go wrong, what we're unsure about — including this
signals honesty and invites help on the genuine uncertainties). This order isn't
arbitrary — it's the sequence a reader needs to reason toward the decision.

### Write for skimmers: headings carry the argument

Almost nobody reads a design doc linearly start to finish — they skim, jump to
sections, read the summary and the parts relevant to them. So structure for
skimming: **headings should carry the argument** (someone reading only the section
headings should get the gist — "Problem: our deploy process can't support
independent team releases" is a heading that conveys, "Background" is one that
doesn't), lead each section with its point (BLUF at section level too), use
formatting (bold, bullets, diagrams) to make the structure scannable, and — most
importantly — put a **one-paragraph executive summary at the very top** (the whole
thing in miniature: the problem, the recommendation, the ask — so a reader who
reads only that paragraph understands the essence and can decide whether to read
more). The English track's Lesson 41 develops the exec summary.

### Soliciting review without design-by-committee

A design doc invites feedback — but there's a failure mode where the doc's clear
recommendation gets watered into mush by trying to accommodate every reviewer's
opinion (design-by-committee). The lead's craft: genuinely solicit and incorporate
substantive feedback (the doc should get *better* from review — catch flaws,
consider missed options, sharpen the reasoning), while maintaining a clear
recommendation and not diluting it to please everyone. Distinguish feedback that
improves the decision (incorporate) from feedback that's just a different
preference (acknowledge, but the doc can still recommend one thing — disagree and
commit, Lesson 7). The options-considered section helps: a reviewer's alternative
is addressed there (considered, here's why it lost) rather than derailing the
recommendation. A good doc has a *point of view*, informed by review, not a
committee-averaged mush.

### End with the decision

A proposal's purpose is a decision, so make reaching one easy: state clearly what
decision is being requested and from whom, and — good practice — include a
decision log at the end (what was decided, when, by whom, and the key reasoning —
essentially an ADR, Lesson 7, capturing the outcome). A doc that generates
discussion but never reaches a documented decision has failed at its job; the
structure should drive toward "here's what we're asking you to decide" and then
"here's what was decided."

{: .note }
> **The problem statement is the most-skipped, most-important section</br>**
> The single most common design-doc flaw is jumping to the solution without
> crisply stating the <em>problem</em> — because the author is excited about their
> solution and the problem feels obvious to them (curse of knowledge, Lesson 13).
> But a doc that doesn't nail the problem invites solutions to the wrong problem,
> lets reviewers argue past each other (each solving a different implicit problem),
> and can't be evaluated (you can't judge a solution without knowing what it's
> solving). Spend disproportionate effort on the problem statement — a crisp,
> agreed problem is half the decision, and a doc whose reviewers all agree on the
> problem will reach a good decision far more easily than one where the problem was
> assumed.

---

## Lab — Scenario

**The situation:** A senior engineer on your team has written a design doc for a
significant new system, and asked for your review. It's six pages, and it's a
mess: it dives straight into the proposed technical solution on page one with no
clear statement of what problem it solves; the alternatives are mentioned in
passing but never really laid out; there's no summary; the reasoning for the
recommendation is scattered throughout; and there's a long section of
implementation detail that buries the key decision. The underlying idea seems
sound, but the doc won't get good review or reach a clean decision in this state.

**Restructure it: write the outline (the sections and what goes in each) the doc
should have, and write the missing one-paragraph executive summary (invent
plausible specifics based on "a significant new system").** Then note what your
restructuring fixes.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong response applies the standard structure, fixes the specific problems (no
summary, no problem statement, buried options and reasoning, implementation detail
crowding the decision), and produces a genuine skimmable outline plus an exec
summary. Example (with invented specifics — say the system is a new event-
processing pipeline):
<br><br>
<strong>Executive summary (one paragraph, at the very top):</strong> "We need a new
event-processing pipeline because our current approach can't handle the volume and
latency our growth demands — events are processed too slowly and we're hitting
scaling limits that will cause failures within [timeframe]. This doc recommends
building a [streaming-based] pipeline using [approach X], chosen over [Y and Z]
because it best balances our latency needs, operational simplicity, and the team's
existing skills. The main risks are [operational maturity of the new tooling] and
[migration complexity], addressed in the risks section. We're asking for a decision
to proceed with this approach and allocate [resources] for [timeframe]." — This is
the whole doc in miniature: problem, recommendation, why, main risk, and the ask,
so a reader who reads only this understands the essence and can decide whether to
engage further.
<br><br>
<strong>The restructured outline:</strong>
<br>
<strong>1. Summary</strong> (the paragraph above) — BLUF, at the top.
<br><strong>2. Context</strong> — what's the current situation and why we're
addressing it now (our current event processing, how it grew, what's changed to
make this urgent).
<br><strong>3. Problem</strong> — the crisp statement of what we're solving: "our
current pipeline can't handle [volume] at [latency], and will fail at projected
growth within [timeframe]; specifically [the concrete limitations]." (The
most-important section, currently missing entirely — the doc jumped to the
solution.)
<br><strong>4. Requirements / Goals</strong> — what a solution must achieve
(latency targets, volume, operational constraints, timeline) — the criteria the
options will be judged against.
<br><strong>5. Options considered</strong> — the alternatives laid out properly:
Option X (recommended), Option Y, Option Z, each with its approach, how it meets
the requirements, and its trade-offs — and why the non-recommended ones lost. (The
options were only "mentioned in passing" — this section makes the recommendation
trustworthy and lets reviewers engage the real trade-offs.)
<br><strong>6. Recommendation</strong> — Option X, with the reasoning gathered in
one place (currently "scattered throughout") — why it best meets the requirements
given the trade-offs.
<br><strong>7. Risks and open questions</strong> — what could go wrong (the new
tooling's operational maturity, migration complexity), how we'd mitigate, and what
we're still unsure about (inviting input on the genuine uncertainties).
<br><strong>8. Implementation plan</strong> — the detail (currently burying the
decision on the early pages) moved to the <em>end</em>, where it's available for
those who want it but doesn't crowd out the decision-relevant content. Ideally
high-level here with detail in an appendix.
<br><strong>9. Decision requested</strong> — clearly: "we're asking [whom] to
decide whether to proceed with Option X and allocate [resources]," plus a
decision-log space for the outcome.
<br><br>
<strong>What the restructuring fixes:</strong> (1) The <strong>missing summary</strong>
— now a reader gets the essence in one paragraph and can decide how deeply to
engage (respects the skimmer, Lesson 13's BLUF). (2) The <strong>missing problem
statement</strong> — now the doc states crisply what it's solving <em>before</em>
the solution, so reviewers evaluate the solution against a clear, agreed problem
(the note's point — the most important fix) rather than arguing past each other. (3)
The <strong>buried options</strong> — now laid out properly, making the
recommendation trustworthy (shows alternatives were weighed), preventing "did you
consider Y?" derailment (Y is there with its reasoning), and letting reviewers
engage the real trade-offs. (4) The <strong>scattered reasoning</strong> — now
gathered in the recommendation section, so the case is coherent and evaluable. (5)
The <strong>implementation detail crowding the decision</strong> — moved to the end,
so the decision-relevant content (problem, options, recommendation, risks) comes
first and the detail is available but not in the way. (6) The <strong>headings now
carry the argument</strong> — someone skimming just the section titles gets the
story (problem → options → recommendation → risks → decision). Net: the doc goes
from a wall of text that would generate bikeshedding and never reach a clean
decision, to a skimmable, well-structured proposal that gets good review (reviewers
engage the real trade-offs against a clear problem) and drives to a documented
decision. The underlying idea (sound, per the scenario) can now actually succeed
because it's presented in a form that lets people evaluate and decide on it. When
giving this feedback to the engineer, frame it as the doc's <em>effectiveness</em>
(this restructuring will get you better review and a cleaner decision), not as
criticism of the idea (which is sound) — and use it as a teaching moment on doc
structure (Lesson 9's design-review-as-teaching). Common mistakes in the response:
(1) fixing the prose but not the structure (the problem is structural — missing
summary/problem, buried options/decision — not wording); (2) keeping the
implementation detail up front (it crowds the decision); (3) not adding a problem
statement (the most important fix); (4) writing an outline whose headings don't
carry the argument (so it's still not skimmable).
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Design doc structure & purpose | Google's design doc culture — <https://www.industrialempathy.com/posts/design-docs-at-google/> |
| Writing for skimmers; the pyramid | *The Pyramid Principle*, Barbara Minto |
| Executive summaries & document craft | English for Work track, Phases 7 (Lessons 38–43) |
| Proposals that get decisions | Amazon's 6-pager / narrative memo culture |
| Options-considered & ADRs | Lesson 7 (ADRs); Lesson 8 (trade-offs) |

---

## Checkpoint

**Q1.** Why is the "options considered" section so valuable in a design doc, and
what goes wrong in a doc that presents only the recommendation?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The options-considered section is valuable for several reinforcing reasons. (1)
<strong>It builds trust in the recommendation</strong> — showing that alternatives
were genuinely weighed (and why they lost) demonstrates the recommendation is the
product of real analysis, not the author's first idea or pet preference; a doc that
presents only the recommendation looks like a decree or an unexamined default,
which reviewers rightly distrust ("did they even consider other approaches?"). (2)
<strong>It prevents derailment by "did you consider X?"</strong> — without the
section, reviewers will inevitably raise alternatives ("why not use Y?"), and each
one either derails the discussion (now debating an option the author may have
already ruled out) or, worse, reopens the whole decision; <em>with</em> the section,
those alternatives are already addressed (considered, here's why it lost), so
reviewers engage with the actual reasoning rather than re-litigating from scratch —
their alternative is <em>there</em>, respectfully handled. (3) <strong>It lets
reviewers engage the real trade-offs</strong> — the genuine substance of most
technical decisions is the trade-offs between viable options (Lesson 8), and a doc
that shows the options lets reviewers reason about those trade-offs (maybe they
weight them differently, maybe they see one the author missed) — which is exactly
the substantive review you want; a recommendation-only doc gives them nothing to
engage but the conclusion, so review becomes either rubber-stamping or vague
objection. (4) <strong>It makes dissenters feel heard</strong> — someone whose
preferred option is in the section, addressed with an honest reason it lost, feels
their view was considered (even if not chosen), which is important for
disagree-and-commit (Lesson 7); if their option isn't mentioned, they feel
dismissed and resist. (5) <strong>It documents the reasoning for the future</strong>
— like an ADR (Lesson 7), it preserves why the roads-not-taken were rejected, so
future engineers don't "rediscover" a ruled-out option and reopen a settled
question. What goes wrong with recommendation-only: the doc reads as a decree
(distrusted), invites endless "why not X?" derailment (nothing addresses the
alternatives), gives reviewers nothing substantive to engage (so review is
shallow — bikeshedding on details or vague pushback), leaves dissenters feeling
unheard (their option ignored), and loses the reasoning for the future (the
alternatives and why they lost aren't recorded). The recommendation looks
<em>weaker</em> and gets <em>worse</em> review, paradoxically, for showing only the
conclusion — because the conclusion's credibility and the review's quality both
depend on the alternatives being visible. The deeper point: a design decision's
value is largely in the <em>reasoning about trade-offs among options</em>, so a doc
that hides the options hides the substance, reducing itself to "trust me, this is
right" — which neither earns trust nor enables good review. The options-considered
section is where the real thinking is shown and the real review happens, which is
why it's a hallmark of a strong design doc and its absence a reliable sign of a
weak one.
</details>

**Q2.** "Write for skimmers — headings carry the argument." Explain what this means
concretely and why it matters, given how design docs are actually read.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
It matters because of how design docs are <em>actually</em> read, which is almost
never linearly start to finish: readers skim, jump to the sections relevant to
them, read the summary and headings to decide what to engage with, and often make
their assessment from a partial read (a busy reviewer, an exec, a stakeholder with
five other docs). Given that reality, a doc structured for a hypothetical
cover-to-cover reader fails the actual readers — they miss the point, engage the
wrong parts, or bounce off the wall of text. "Write for skimmers" means structuring
the doc so it works for how it's really consumed. Concretely: (1) <strong>Headings
carry the argument</strong> — the section titles, read alone, should convey the
doc's story. "Problem: our deploy process blocks independent team releases" is a
heading that <em>communicates</em> (a skimmer reading just the headings learns the
problem); "Background" or "Overview" or "Section 3" convey nothing. If someone reads
only your headings — which many will — they should get the gist: the problem, that
options were considered, the recommendation, the risks. This means writing headings
as mini-statements of each section's point, not as generic labels. (2) <strong>Lead
each section with its point</strong> (BLUF at section level) — a skimmer reading the
first sentence of each section gets the substance without reading the whole section.
(3) <strong>A one-paragraph executive summary at the very top</strong> — the whole
doc in miniature, so a reader who reads <em>only</em> that understands the essence
and can decide whether to go deeper (respecting the many who will read only the
summary). (4) <strong>Scannable formatting</strong> — bold key points, bullets for
lists, diagrams for structure, short paragraphs — so the eye can find what it needs
and the structure is visible at a glance, rather than a dense block that must be
read word-by-word to extract anything. (5) <strong>Front-load the decision-relevant
content</strong> and push detail to the end/appendix (the lab's implementation-
detail fix), so the parts that matter for the decision come first where skimmers
reach them. Why this matters: a doc's job is to get read, get good feedback, and
reach a decision (this lesson's opening), and a doc that only works if read
linearly cover-to-cover will get none of those from the busy, skimming, partial-
reading audience it actually has — it'll be abandoned, mis-engaged, or bounced
off. Writing for skimmers isn't dumbing down or sacrificing rigor; it's designing
the doc's <em>information architecture</em> so that the substance is accessible at
every level of reading depth — the skimmer gets the story from headings, the
moderate reader gets the argument from section-leads and summary, and the deep
reader gets the full detail — so the doc serves all its real readers rather than
only the mythical linear one. It's audience-first (Lesson 13) applied to document
structure: shape the doc for how the receiver actually consumes it. The practical
test: read only your headings and summary — do you get the doc's whole argument? If
not, the headings aren't carrying the argument, and most of your readers (the
skimmers) will miss it.
</details>

---

## Homework

Take a design doc or proposal you've written (or need to write, or one you're
reviewing). Apply this lesson's structure: does it have a one-paragraph summary at
the top? A crisp problem statement before the solution? A real options-considered
section? Reasoning gathered in one place? Detail pushed appropriately back? Do the
headings carry the argument (read only the headings — do you get the story)?
Restructure it, and write or sharpen the executive summary. Then reflect: what was
weakest (probably the problem statement or the summary), and how does the
restructured version's likely reception (review quality, path to decision) differ
from the original's?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the doc-structuring skill against the common failure modes, and
usually reveals that the weakest parts are the problem statement and the summary —
the two most-skipped, most-important elements. A strong response honestly assesses
the doc against each structural element and produces a genuinely restructured
version with a sharp exec summary. The common findings: (1) <strong>The problem
statement was weak or missing</strong> — the doc jumped toward the solution because
the problem felt obvious to the author (curse of knowledge, Lesson 13), and forcing
a crisp problem statement is both the hardest part and the highest-value fix
(often it reveals the author hadn't fully articulated the problem even to
themselves — the same clarifying effect as the two-sentence version in Lesson 14).
(2) <strong>The summary was missing or buried the point</strong> — writing a true
one-paragraph BLUF summary (problem, recommendation, why, ask) forces distilling
the essence, and many docs don't have one, so most readers never got the gist. (3)
<strong>The options were thin</strong> — either absent (recommendation-only, Q1's
failure) or mentioned without real analysis, so the recommendation looked
unexamined and reviewers had nothing substantive to engage. (4) <strong>The
headings didn't carry the argument</strong> — reading only the headings gave a
skimmer nothing (generic labels like "Overview," "Details"), so the doc only worked
for a linear reader who doesn't exist (Q2). The restructuring's payoff, on
reflection: the restructured doc will get <em>better review</em> (reviewers engage
the real trade-offs against a clear problem, rather than bikeshedding or arguing
past each other) and reach a <em>cleaner decision</em> (the structure drives toward
"here's what we're asking you to decide," and the options-considered prevents
derailment) — versus the original, which would generate scattered discussion, "did
you consider X?" derailment, unheard dissenters, and no clean decision. The meta-
skill: a design doc is a <em>decision-driving instrument</em>, and its
effectiveness at driving a good decision efficiently depends heavily on its
structure — the reusable skeleton (summary → context → problem → options →
recommendation → risks → decision), written for skimmers with argument-carrying
headings, is not a formality but the architecture that lets the doc do its job. A
lead writes and reviews many docs, so this structural skill compounds: better docs
mean better decisions reached faster, and modeling/teaching good doc structure
(giving this feedback in reviews — Lesson 9) raises the whole team's ability to
drive decisions through writing. The English track's Phase 7 develops the writing
<em>craft</em> (plain language, flow, exec summaries) that complements this
structural skill — together they make a lead's written communication a powerful
decision-driving and influence tool (which, since a lead spends more time
communicating than coding, is among the highest-return skills to develop). If your
doc was already well-structured (summary, crisp problem, real options, argument-
carrying headings) — good, that's a strong foundation; the ongoing practice is
maintaining that structure under time pressure (when it's tempting to dump the
solution without the problem statement or skip the summary) and helping your team
adopt it.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 17 — Presenting and Demos →](lesson-17-presenting){: .btn .btn-primary }
