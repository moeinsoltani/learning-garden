---
title: "Lesson 14 — Explaining Technology to Non-Technical People"
nav_order: 2
parent: "Phase 3: Communication Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 14: Explaining Technology to Non-Technical People

## Concept

A huge part of a lead's job is translating between the technical world and the
people who fund, prioritize, and depend on it — executives, PMs, sales,
customers. These people make decisions that depend on understanding the technical
situation, but they don't share your vocabulary or mental models. The skill is
making them genuinely *understand* (or at least *feel*) the technical reality
without the jargon — so they can make good decisions.

```
   THE WRONG WAY                          THE RIGHT WAY
   ─────────────                          ─────────────
   "we need to refactor the auth          "right now, every change to how
    service because the coupling           people log in risks breaking the
    creates cascading failures and         whole system, which is why it's slow
    the tech debt is slowing velocity"     and risky — like a house where you
        → they nod, understand nothing,    can't fix one room without the others
          make a bad decision              collapsing"
                                            → they GET it, decide well
```

Two moves make this work: **translate into their units** (money, risk, time,
customer impact — the things they actually reason about), and **use analogies**
that map the technical situation onto something they already understand. The goal
isn't to dumb it down (which is condescending and loses real meaning) — it's to
convey the *essential reality* in a form they can reason with.

---

## How It Works

### Quantify in their units

Non-technical decision-makers reason in business units: money, risk, time,
customer impact, competitive position. So translate technical facts into those:
not "high latency" but "pages load in 4 seconds, and we lose ~7% of users for
every second — that's costing us [X]"; not "tech debt" but "features in this area
take 3x longer, and it causes [N] outages a quarter" (Lesson 11's whole approach);
not "we need better testing" but "we ship bugs to customers [N] times a month, and
each erodes trust and costs support time." The technical fact becomes a business
fact they can weigh against other business facts. Quantify wherever you can —
numbers in their units are far more persuasive and understandable than technical
adjectives.

### Analogies that carry weight (and their limits)

A good analogy maps an unfamiliar technical situation onto something the audience
already understands, letting them reason about it: technical debt as financial
debt (borrow speed now, pay interest later); a monolith's coupling as a house
where you can't renovate one room without the others collapsing; scaling as a
restaurant that works with 10 tables but breaks at 100; a cache as keeping
frequently-used things on your desk instead of walking to the filing cabinet.
Analogies are powerful because they transfer intuition. But they have limits:
every analogy breaks down if pushed too far (don't let the audience over-extend
it), and a bad analogy actively misleads (maps to the wrong intuition). Use them
to convey the *core* idea, signal where they stop ("the analogy breaks down
here"), and don't rely on them for precision.

### Answer the question they actually have

When an exec asks "why is it late/slow/hard?", they usually want to understand the
*situation and what to do about it*, not a technical lecture. Answer the real
question: what's going on (in their terms), what it means for them (impact,
options, trade-offs), and what you recommend or need. "It's late because the
integration turned out to require rebuilding a component we thought we could
reuse — that's the [X weeks]. The options are [ship reduced scope on time /
full scope late / add people with diminishing returns], and I recommend [Y]
because [business reason]." That's what they can act on — not the technical
details of why the component couldn't be reused.

### The two-sentence version

A useful discipline: be able to explain any technical situation in *two
sentences* for a non-technical audience — the essential what and the essential
so-what. "Our login system is built in a way where any change risks breaking
everything, which is why it's slow and fragile. Investing [X] to restructure it
would let us ship login features [3x faster] and cut the outages that are hurting
customer trust." If you can't compress it to that, you don't yet understand the
audience's version of it well enough — the compression forces you to find the
core that matters to them.

{: .note }
> **Don't confuse "simple" with "dumbed down"</br>**
> Explaining technology simply to a non-technical audience is not talking down to
> them or hiding complexity — it's <em>respecting</em> them enough to convey the
> real situation in terms they can reason with. Executives and PMs are smart
> people making important decisions; they just don't share your technical
> vocabulary. Giving them jargon isn't "treating them as equals" — it's failing to
> communicate, and it leads to bad decisions made on non-understanding. Giving
> them a clear, quantified, analogy-supported explanation of the real situation
> respects both their intelligence and their need to actually understand. The
> condescension is in the jargon (which excludes them), not in the clarity.

---

## Lab — Scenario

**The situation:** Your team is midway through a critical migration (moving off a
legacy database that's increasingly unreliable and expensive). The migration is
going to need three more months than originally planned — the legacy system's data
turned out to be messier and more entangled than anyone knew, and rushing it risks
data corruption. You need to explain this to the CFO, who controls the budget, is
frustrated about the delay, and is not technical.

**Explain to the CFO why the migration needs three more months — WITHOUT using the
words "refactor," "technical debt," or "legacy" (and no jargon generally).** Make
them genuinely understand and support the decision.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong response translates entirely into the CFO's units (risk, cost, time),
uses an analogy to convey the core problem, answers their real concern (is this
delay justified and what are my options), and respects their intelligence while
avoiding jargon. Example:
<br><br>
"I want to walk you through why we need three more months, and give you the
options so you can decide. Here's the situation in plain terms: we're moving all
our data from an old system that's becoming unreliable and expensive — you're
right that this is overdue and I share the frustration about the timeline. What we
discovered partway through is that the data in the old system is far messier and
more tangled than anyone realized — records that reference each other in
inconsistent ways, duplicates, gaps that built up over years. Think of it like
moving a warehouse where we assumed the inventory was neatly labeled and
palletized, but when we got in there, half of it is loose, mislabeled, and some
boxes are wired to others in ways that break if you move them wrong. We can't just
truck it over — we have to carefully untangle and verify it as we go.
<br><br>
Here's why that matters to us specifically: this is <em>customer and financial
data</em>. If we rush it and get it wrong, we risk corrupting records — which
could mean customers seeing wrong information, billing errors, or data loss. The
cost of that — customer trust, potential financial and compliance exposure,
and the cleanup — would dwarf the cost of the extra three months. So the three
months isn't padding; it's the time to move this safely without risking the kind
of data problem that would be far more expensive than the delay.
<br><br>
Your options as I see them: (1) take the three months and do it safely — my
recommendation, because the downside risk of rushing is severe and the old system,
while degrading, is holding for now; (2) rush it in the original timeline — which
I'd strongly advise against, because the risk of corrupting customer/financial
data is real and the potential cost is much higher than the savings; (3) if the
timeline is truly critical for a business reason I'm not seeing, we could discuss
adding specialized help, though data-untangling work like this doesn't parallelize
well, so it'd buy less than you'd hope. I recommend option 1 — the three months is
the responsible investment to avoid a much more expensive problem. What questions
can I answer, and is there a business constraint driving the timeline that I should
factor in?"
<br><br>
<strong>Why this works:</strong> It's entirely in the CFO's units — the core
argument is <em>risk vs cost</em> (rushing risks corrupting customer/financial
data, whose cost dwarfs the delay), which is exactly how a CFO reasons; no jargon
("refactor/tech debt/legacy" all avoided, replaced with plain descriptions). The
<em>warehouse analogy</em> conveys the essential problem (we assumed clean data,
found tangled data that breaks if moved carelessly) in a way the CFO can picture
and reason about — transferring the intuition for <em>why</em> it takes longer
without any technical detail. It answers the CFO's <em>real</em> question (is this
delay justified, and what are my options) with a clear recommendation and honest
options, rather than a technical lecture. It respects the CFO (acknowledges their
frustration as legitimate, gives them the decision and the reasoning, invites
their business context) rather than either talking down or drowning them in
detail. And it frames the three months as an <em>investment against a larger
risk</em> (Lesson 11's reframe), which is persuasive to someone who thinks in
cost/risk. Note the two-sentence core is available inside it: "The old data is far
messier than we knew, and rushing to move it risks corrupting customer and
financial data. The three extra months is the time to do it safely and avoid a
problem that would cost us far more than the delay." Common mistakes: (1) using
jargon or technical detail the CFO can't parse (loses them, or makes them feel
they're being snowed); (2) being defensive about the delay rather than
explaining and owning it with options; (3) not connecting to the CFO's actual
concern (risk/cost) — e.g., explaining the <em>technical</em> reasons the data is
messy rather than the <em>business</em> stakes of rushing; (4) no clear
recommendation (dumping options without guidance leaves the CFO to decide on
non-understanding); (5) an analogy that misleads or that you over-extend.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Explaining complex ideas simply | *Made to Stick*, Chip & Dan Heath (analogies, concreteness) |
| Communicating with executives | *The Manager's Path*, Camille Fournier |
| Translating tech to business value | Lesson 48 (tech→business); *An Elegant Puzzle*, Will Larson |
| Analogies and their limits | *The Sense of Style*, Steven Pinker |
| Talking to non-technical stakeholders | LeadDev — <https://leaddev.com/communication> |

---

## Checkpoint

**Q1.** Why is translating technical facts into the audience's units (money, risk,
time, customer impact) more effective than explaining them accurately in technical
terms — even though the technical version is "more correct"?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Because communication's purpose isn't to be technically correct — it's to produce
<em>understanding and good decisions in the receiver</em>, and a technically
accurate explanation the audience can't parse produces neither. The technical
version ("high coupling causes cascading failures and the tech debt slows
velocity") is precise, but to a non-technical decision-maker it's opaque: they
don't have the mental models to reason with those concepts, so they can't weigh the
situation, can't compare it to their other priorities, and end up either deferring
blindly to the engineer (abdicating a decision they own) or deciding on
non-understanding (badly). The units-translated version ("features in this area
take 3x longer and it causes N outages a quarter, costing us [X] and eroding
customer trust") maps the technical reality onto the dimensions the audience
<em>actually reasons in</em> — money, risk, time, customer impact — so they can
genuinely understand the situation, weigh it against their other concerns, and make
a good decision. The key insight is that "correctness" in communication is measured
by the understanding produced in the receiver, not by the technical precision of
the words: a "more correct" explanation that lands as noise is <em>less</em>
correct in the way that matters (it fails to inform the decision), while a
"simplified" explanation in the right units that produces accurate understanding
and a good decision is <em>more</em> correct in the way that matters. Non-technical
stakeholders don't need to understand the <em>mechanism</em> (why coupling causes
cascading failures) — they need to understand the <em>implications</em> for the
things they're responsible for (cost, risk, delivery, customers), because those are
what their decisions turn on. Translating into their units also enables them to do
their actual job: a CFO can't decide whether three months is worth it if the case
is made in technical terms, but can decide it easily when it's framed as
risk-vs-cost (the thing CFOs are expert at weighing). So you're not "dumbing it
down" or sacrificing correctness — you're conveying the essential reality in the
form that lets the audience reason accurately about it, which is the entire point
of communicating to them. The technical version optimizes for the wrong thing
(precision of expression) at the expense of the right thing (understanding and
decision quality in the audience); the units-translated version optimizes for what
actually matters. (This connects to audience-first, Lesson 13: the technical version
is sender-first — organized around what's true in the engineer's world; the
units-translated version is audience-first — organized around what the receiver
needs to understand and decide.)
</details>

**Q2.** Analogies are powerful for explaining technology to non-technical people,
but also risky. Explain both their power and their danger, and how to use them
well.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>The power:</strong> a good analogy transfers <em>intuition</em> — it maps
an unfamiliar technical situation onto something the audience already deeply
understands, so they can reason about the unfamiliar thing using their existing
mental models. Technical debt as financial debt lets a CFO immediately grasp
"borrow speed now, pay interest later, and if you never pay it down it compounds
until it cripples you" — they already understand debt, so the analogy hands them a
whole reasoning framework for a concept they'd otherwise find abstract. A monolith's
coupling as a house where you can't renovate one room without the others collapsing
conveys the <em>essential problem</em> (changes are risky and entangled) in a
single image anyone can picture. This is why analogies are one of the most
effective tools for cross-domain explanation: they don't just describe the
situation, they <em>equip the audience to think about it</em>. <strong>The
danger:</strong> analogies are always imperfect maps — they're similar to the
target in some ways and different in others — so they carry two risks. (1)
<strong>They break down when pushed too far</strong>: every analogy holds only
within a certain range, and if the audience (or you) extends it beyond that range,
it starts implying false things (the financial-debt analogy breaks down if someone
concludes "so we should never take on any technical debt," missing that prudent
debt is a legitimate tool — Lesson 11; or the house analogy breaks if someone
thinks you can "hire more builders" to parallelize the way you would a renovation).
(2) <strong>A bad analogy actively misleads</strong>: if the analogy maps to the
<em>wrong</em> intuition (e.g., describing a probabilistic system with a
deterministic analogy), it gives the audience a confidently wrong mental model,
which is worse than no analogy — they now reason incorrectly with conviction. How
to use them well: (a) <strong>choose analogies that map the core correctly</strong>
— the essential feature you're trying to convey should correspond accurately, even
if peripheral details don't; (b) <strong>signal where they break down</strong> —
explicitly say "the analogy holds for [X] but breaks down at [Y]" so the audience
doesn't over-extend it (e.g., "like a renovation, but unlike a renovation, adding
more people doesn't speed it up because the work is sequential"); (c) <strong>use
them to convey the core idea, not for precision</strong> — the analogy gets the
audience to the right intuition, then you supply the specific facts/numbers for the
actual decision (the analogy is the on-ramp, not the whole road); (d) <strong>test
them</strong> — a good analogy resonates ("oh, I get it") and a bad one produces
confusion or wrong conclusions; watch the audience's reaction and adjust; (e)
<strong>don't rely on a single analogy for a complex situation</strong> — different
aspects may need different analogies, and no one analogy captures everything. The
meta-principle: analogies are a tool for building the audience's intuition
efficiently, extremely powerful when the mapping is right and bounded, actively
harmful when it's wrong or over-extended — so use them deliberately (the right
analogy for the core point, its limits signaled) rather than casually (grabbing
whatever comparison comes to mind, which may mislead). The lead who explains
technology well has a repertoire of good, tested analogies for the recurring
situations (debt, coupling, scaling, caching, reliability) and deploys them with
awareness of where each stops.
</details>

---

## Homework

Take a technical situation on your team that non-technical stakeholders need to
understand (a debt problem, a delay, an architecture decision, a risk). Write the
explanation for a specific non-technical audience (your PM, a director, an exec):
translate it entirely into their units (money/risk/time/customer impact), find an
analogy that conveys the core (and note where the analogy breaks down), and produce
the two-sentence version. Ban yourself from the jargon you'd normally use. Then
test it: would this audience genuinely understand the situation and be able to make
a good decision from it? What did removing the jargon and finding the analogy force
you to clarify in your own understanding?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the translation skill and often reveals something about your
<em>own</em> understanding. A strong response translates fully into the audience's
units (no technical adjectives standing in for business facts — "slow" becomes
"features take 3x longer, costing X"; "fragile" becomes "causes N outages, eroding
customer trust"), finds an analogy that maps the core accurately and names its
limits (showing you understand both what the analogy conveys and where it would
mislead), and compresses to two sentences (the essential what and so-what). The
jargon ban is the forcing function — most technical people lean on jargon as a
shortcut that hides fuzzy thinking, and being unable to use it forces you to
actually articulate the underlying reality in plain terms, which frequently reveals
that you understood the <em>mechanism</em> but hadn't clearly articulated the
<em>implications</em> (the thing the audience needs). The common realization: "I
say 'we need to refactor this' all the time, but forcing myself to explain to a CFO
what that actually means for the business — the cost we're paying, the risk we're
carrying, what the investment would return — made me realize I hadn't crisply
thought through the business case myself; I'd been operating on a technical
intuition that 'this is bad and should be fixed' without translating it into why
it matters to the people who'd fund fixing it." That's the deeper value: explaining
technology to non-technical people well requires <em>you</em> to understand the
business implications of the technical situation, not just the technical
mechanism — and the translation exercise surfaces where your own understanding was
technical-only. The two-sentence compression is especially diagnostic: if you can't
get it to two sentences, you don't yet have the core clear (the compression forces
you to identify what actually matters to this audience and discard the rest). The
analogy-and-its-limits part builds the discipline of using analogies deliberately
(the right one for the core, its breakdown signaled) rather than casually. The
test — "would this audience genuinely understand and be able to decide well?" —
is the real measure: not "is it technically accurate" but "does it produce
accurate understanding and enable a good decision in this specific audience"
(Q1's reframe). The meta-skill this builds is one of the highest-value things a
lead does: being the <em>translation layer</em> between the technical world and the
people who fund, prioritize, and depend on it — because those people make crucial
decisions that depend on understanding the technical reality, and if the lead can't
convey that reality in terms they can reason with, they decide badly (over- or
under-investing, mis-prioritizing, misunderstanding risk). The lead who can make
executives and PMs genuinely <em>get</em> the technical situation — through units-
translation, well-chosen analogies, answering their real question, and the
two-sentence core — dramatically improves the decisions made about their team's
work, and becomes trusted and influential with the stakeholders who matter (Phase
8). If your explanation was already jargon-free, units-translated, and
decision-enabling — excellent, that's a rare and valuable skill; the practice is
maintaining it as a default rather than slipping into the technical shorthand
that's natural among engineers but opaque to the stakeholders a lead must reach.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 15 — Running Effective Meetings →](lesson-15-effective-meetings){: .btn .btn-primary }
