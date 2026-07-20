---
title: "Lesson 08 — Evaluating Trade-offs"
nav_order: 3
parent: "Phase 2: Technical Leadership"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 08: Evaluating Trade-offs

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **trade-off** — a choice where gaining one thing costs another; there is no free option.
> - **implicit vs explicit** — unstated and hidden vs openly named; the lesson's core move is making trade-offs explicit.
> - **premature** — done too early, before it's needed ("premature scalability").
> - **headroom** — spare capacity above current needs.
> - **cost of delay** — what shipping later actually costs the business (sometimes everything, sometimes nothing).
> - **the 10x test** — "would this survive ten times the load — and do we actually *expect* ten times?"
> - **"what would have to be true?"** — asking under what conditions an option would be right, instead of arguing which is better.
> - **product-market fit (PMF)** — proof that people genuinely want your product; the early startup's only goal.
> - **build vs buy** — writing it yourself vs paying for an existing product.
> - **debiasing** (dee-BY-us-ing) — correcting a systematic thinking error.
> - **context** — here: the business situation (stage, scale, timeline, team) that decides which option wins.

## Concept

Almost every meaningful technical decision is a trade-off: speed vs quality,
simple vs scalable, build vs buy, consistency vs availability. Junior engineers
argue about which option is "better" in the abstract; leads recognize that
*there is no better in the abstract* — there's only better **for a given
context**, and the whole skill is making the trade-off *explicit* and letting the
context decide.

```
   THE JUNIOR FRAME               THE LEAD FRAME
   ────────────────               ──────────────
   "microservices are better"     "microservices trade operational complexity
   "we should use Kafka"           for independent scaling and deployment —
   "this needs to scale"           is that trade worth it for OUR situation?"

   argues options in the abstract  makes the trade-off visible, then asks
   → unresolvable, or resolved      what the CONTEXT (scale, timeline, team,
     by whoever's loudest           business stage) implies about the choice
```

The most common failure is **implicit** trade-offs — decisions made without
anyone naming what's being traded, so the team optimizes for the wrong thing
(builds for a scale they'll never hit, or ships fast and drowns in the debt
later) without ever having *decided* to. The lead's job is to surface the
trade-off, name what each side costs and buys, and connect it to the actual
situation.

A key debiasing tool: **premature scalability is debt too.** Engineers reflexively
treat "scalable" as the responsible choice, but building for 10x load you don't
have — and may never have — costs real time and complexity now, often more than
you'll ever recoup. Over-engineering for imagined future scale is as much a
mistake as under-engineering for real present needs; both are trade-offs made
badly because the context wasn't honestly assessed.

---

## How It Works

### Make the trade-off visible

The single highest-value move is to *name the trade-off explicitly* instead of
arguing options: "Option A ships in two weeks but strains above ~10x current
load; Option B takes eight weeks but scales to 100x. We're trading eight weeks of
delay and complexity for headroom we may or may not need." Once it's stated this
way, the decision becomes tractable — you can reason about whether the headroom is
worth the delay *given what you actually know about the business*. Implicit
trade-offs can't be reasoned about because nobody's said what's being traded;
explicit ones can.

### "What would have to be true?"

A powerful framing (from strategy consulting): instead of arguing whether an
option is right, ask *what would have to be true for it to be the right choice?*
"Building for 100x scale is right **if** we expect to 10x within a year and
re-architecting later would be very expensive." Now you have a testable
condition: do we actually expect that growth? Is re-architecting actually that
expensive? This converts an unresolvable "is it better?" debate into concrete
questions about the world you can actually assess — and often reveals that the
conditions for the "obvious" choice don't hold.

### The 10x test, and cost of delay

Two practical lenses: the **10x test** — "would this design survive 10x the
current scale? and do we actually expect 10x?" — forces you to separate real
scaling needs from imagined ones (if you don't expect 10x, building for it is
premature-scalability debt). And **cost of delay** — what does it cost to ship
later? For a market-timing-critical feature, eight weeks of delay might cost the
whole opportunity, making "ship fast, refactor later" clearly right; for
foundational infrastructure with no deadline, the delay is cheap and getting it
right matters more. The same technical trade-off resolves oppositely depending on
these business facts — which is the whole point.

{: .note }
> **The same decision, opposite answers**
> "Should we ship the fast-but-limited version or the slow-but-scalable one?" has
> no universal answer. For a startup racing to prove a market before the money
> runs out: ship fast, the scale problem is a <em>good</em> problem to have later
> (and you might pivot anyway, wasting the scalability work). For a bank building
> a payments core that must handle known high volume and is expensive to change:
> build it right. Same trade-off, opposite resolution — because the
> <em>context</em> (business stage, cost of delay, cost of later change,
> likelihood of the scale) decides, not the technical merits alone. A lead who
> gives the same answer regardless of context hasn't understood trade-offs.

---

## Lab — Scenario

**The situation:** Your team must choose between two designs for a new feature:

- **Design A:** Ships in ~2 weeks. Simple, uses your existing stack. Will handle
  current load comfortably but starts to strain around 10x current traffic —
  above that it would need significant rework.
- **Design B:** Takes ~8 weeks. More complex (new infrastructure the team must
  learn and operate). Scales cleanly to 100x current traffic.

The engineers are split and arguing about which is "the right architecture."

**Work through the decision for three different contexts, showing how the same
trade-off resolves differently:** (1) an early-stage startup racing to prove
product-market fit before funding runs out; (2) an established company adding a
feature to a product with steady, predictable growth of ~20%/year; (3) a company
that just signed a huge enterprise customer whose launch in 3 months will bring
~50x current traffic.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
The whole point is that there's no context-free answer — and a strong response
makes the trade-off explicit first, then shows the context deciding, using the
lenses (cost of delay, 10x test, "what would have to be true").
<br><br>
<strong>The explicit trade-off (before any context):</strong> Design A trades
scale-headroom for speed (2 weeks, simple, but caps around 10x); Design B trades
6 extra weeks + operational complexity for scale (100x). The question is whether
the headroom is worth the delay and complexity — which depends entirely on how
likely and how soon the scale is, and how costly the delay is.
<br><br>
<strong>Context 1 — early-stage startup racing for PMF:</strong> Design A,
clearly. Cost of delay is existential (6 extra weeks could mean missing the
window or running out of money), and the scale problem is <em>speculative</em> —
most startups never hit 10x this feature's load because they pivot, fail, or
change the product first. "What would have to be true for B?" — you'd need high
confidence you'll survive AND hit 10x on this exact feature soon AND that
re-architecting later is prohibitive — none of which a pre-PMF startup can claim.
Building for 100x scale you may never reach is premature-scalability debt paid in
the currency you have least of (time). Ship A; the scale problem, if you're lucky
enough to have it, is a <em>good</em> problem you'll gladly solve with the
resources success brings.
<br><br>
<strong>Context 2 — established company, steady ~20%/year growth:</strong> Design
A, but it's closer, and the reasoning is different. At 20%/year it takes over a
decade to reach 10x — far beyond the horizon where this design or even this
feature will still exist unchanged. So the scale headroom of B buys protection
against a scenario that won't arrive for years (by which time you'll have
rebuilt for other reasons). Cost of delay is moderate (no existential clock, but
6 weeks of engineering time has real opportunity cost — other work not done). "What
would have to be true for B?" — you'd need to expect much faster growth than 20%,
or this specific component to be exceptionally expensive to change later. Absent
that, A is right, and B is over-engineering. (Caveat: if the component is truly
foundational and genuinely expensive to re-architect — a one-way door, Lesson 7 —
the calculus shifts toward more investment; but "scales to 100x" for a feature
that won't need it for a decade is usually premature.)
<br><br>
<strong>Context 3 — signed enterprise customer, 50x traffic in 3 months:</strong>
Design B (or a variant). Here the scale is <em>not</em> speculative — it's
contractually committed and arriving in 3 months, and Design A would <em>fail</em>
at 50x (it strains at 10x), which would mean a broken launch for a major customer:
a business disaster. The 10x test is decisively passed (you expect 50x, soon,
with certainty). Cost of delay on B (8 weeks) fits within the 3-month window. "What
would have to be true for A?" — you'd need the 50x traffic to not materialize,
which contradicts a signed contract. So B's scale is exactly what's required, and
A's speed advantage is irrelevant because A doesn't meet the actual requirement.
(A sophisticated answer might explore whether a middle path exists — ship A-ish
fast to derisk, with a parallel effort on the scaling, or negotiate the launch
timeline — but absent that, B is the choice the context forces.)
<br><br>
<strong>The meta-point:</strong> the <em>same</em> technical trade-off — speed vs
scale — resolved three different ways (A, A, B) purely because the context
(likelihood and timing of scale, cost of delay, cost of later change) differed.
The engineers arguing about "the right architecture" in the abstract were asking
an unanswerable question; the lead's move is to make the trade-off explicit and
route the decision through the business context, using cost-of-delay, the 10x
test, and "what would have to be true?" to convert the debate into concrete
assessable questions. And notably, the "responsible/scalable" choice (B) is right
in only one of three contexts — a direct rebuttal to the reflex that scalable is
always the mature choice; premature scalability is debt too.
<br><br>
Common mistakes: (1) giving a context-free answer ("B, always build it right" or
"A, always ship fast") — the whole lesson is that context decides; (2) failing to
make the trade-off explicit and instead arguing the designs' technical merits; (3)
treating "scalable" as automatically responsible and missing that building for
unneeded scale is its own waste; (4) in context 3, still leaning toward A because
"ship fast is agile" — ignoring that A doesn't meet the actual, certain
requirement.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| "What would have to be true?" | *Playing to Win*, Lafley & Martin |
| Cost of delay | Don Reinertsen — *The Principles of Product Development Flow* |
| Premature optimization / scalability | Martin Fowler, and the classic Knuth caution |
| Build vs buy & technical trade-offs | *The Staff Engineer's Path*, Tanya Reilly |
| Reversible vs irreversible decisions | Lesson 07 (ADRs) and Bezos's doors |

---

## Checkpoint

**Q1.** Two engineers argue: one insists "we have to build this to scale," the
other says "that's over-engineering." How do you, as the lead, move them from an
abstract argument to a resolvable decision?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Stop the abstract argument and <strong>make the trade-off explicit, then route it
through the context</strong> using concrete, assessable questions. The move: (1)
<em>Name what's actually being traded</em> — "Building to scale costs us [X weeks
+ operational complexity] now; not building to scale risks [rework/failure] if we
hit [Y load]. So the real question isn't 'is scalable better' — it's 'is that
future headroom worth that present cost, given our situation?'" This reframes an
unanswerable abstract debate ("scale good" vs "over-engineering bad") into a
tractable one. (2) <em>Apply the 10x test</em> — "do we actually expect 10x this
load, and if so, when?" This is the crux: the "build to scale" engineer is often
assuming growth that isn't established, and the "over-engineering" engineer is
often assuming it won't come — but neither has checked. Force the question:
what's our real expected growth, and what's our evidence? (3) <em>Ask "what would
have to be true?"</em> — "For building-to-scale to be right, what would have to be
true? We'd need to expect [10x within some horizon] AND that re-architecting
later is expensive. Do those hold?" This converts opinion into testable
conditions about the actual world. (4) <em>Weigh cost of delay</em> — "what does
shipping the scalable version later cost us? Is there a deadline/opportunity that
the extra time threatens?" Together these turn "scalable vs over-engineered" (an
unresolvable values clash, usually decided by who's more senior or louder) into a
set of concrete questions — expected growth, cost of later change, cost of delay —
that have actual answers the team can find or estimate, and that <em>decide</em>
the trade-off. Often the exercise reveals that neither engineer had the facts: the
expected growth is far lower (or higher) than assumed, tilting the decision
clearly. The lead's contribution isn't picking a side in the abstract argument —
it's changing the <em>question</em> from "which is better" (unanswerable) to "what
does our specific context imply" (answerable), and insisting the trade-off be made
explicitly and connected to real business facts rather than left implicit and
decided by conviction. And the lead should name the debiasing point directly:
"scalable" isn't automatically the responsible choice — building for scale you
won't need is real waste (premature-scalability debt), just as building too small
for scale you will need is; the mature move is to assess honestly which situation
you're actually in, not to reflexively favor either side.
</details>

**Q2.** "Premature scalability is debt too." Explain why over-engineering for
imagined future scale is a mistake, not just conservative good practice — and how
a lead should think about when scaling investment is actually warranted.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Engineers reflexively code "scalable" as the responsible, mature, safe choice —
building for future scale feels like professionalism, and under-building feels
like cutting corners. But that reflex ignores that <em>scalability isn't free</em>:
building for scale you don't have costs real time (weeks/months that could ship
other value), real complexity (more infrastructure to build, learn, operate,
debug — every added component is ongoing operational burden and a source of bugs),
and real opportunity (the effort spent on speculative scale is effort not spent on
things that matter now). When that investment is made for scale you <em>never
reach</em> — because you pivot, the feature changes, growth doesn't materialize,
or you rebuild for unrelated reasons before ever hitting the load — it's pure
waste: you paid the full cost and got none of the benefit. That's why it's debt,
not prudence: like other debt, it's a present cost taken against a future that may
not arrive, and worse than most debt because it often <em>can't be recouped</em>
(you can't get the weeks back, and the complexity keeps costing you via operational
overhead even while unused). It also compounds: over-engineered systems are harder
to change, so the premature complexity makes you <em>slower</em> at the pivots and
adaptations you actually need — the opposite of the flexibility "scalability" was
supposed to buy. The symmetry is the key insight: under-engineering for scale you
<em>will</em> need is a mistake (you'll hit the wall and pay a painful emergency
rework), but over-engineering for scale you <em>won't</em> need is <em>equally</em>
a mistake (you paid up front for nothing and slowed yourself down) — both are
trade-offs made badly because the actual future demand wasn't honestly assessed.
How a lead should think about when scaling investment is warranted: (1)
<strong>Is the scale real or speculative?</strong> — committed/contracted/strongly-
evidenced future load (like the enterprise customer, context 3) warrants building
for it; "we might grow" without evidence usually doesn't (the 10x test — do we
actually expect it, and when?). (2) <strong>How expensive is it to add scaling
later?</strong> — if the architecture makes future scaling a reasonable
incremental change (a two-way door — Lesson 7), you can defer and add it when
needed (build for now, scale when the need is real); if it's a one-way door where
retrofitting scale is prohibitively expensive, that raises the case for investing
earlier. (3) <strong>What's the cost of the investment now vs the cost of the
rework later, weighted by probability?</strong> — an expected-value calc: high
probability of needing scale + expensive-to-add-later + affordable-now argues for
building it; low probability + cheap-to-add-later + expensive-now argues against.
The disciplined default for most teams most of the time (especially earlier-stage
or uncertain situations) is <em>build for the scale you can see, keep the
architecture from foreclosing future scaling, and add scale when the need becomes
real</em> — because the future is uncertain and premature investment in it is
usually the losing bet. The lead's job is to resist both the "build it huge to be
safe" reflex (premature-scalability debt) and the "never think ahead" recklessness
(real scaling cliffs), and instead assess honestly, per situation, whether the
scale is real enough and soon enough and expensive-enough-to-add-later to warrant
paying now — which is just the explicit trade-off reasoning of this whole lesson,
applied to the specific and very common temptation to over-build for an imagined
future.
</details>

---

## Homework

Take a real trade-off decision your team is facing (or recently made). Make it
explicit in writing: what each option costs and buys, stated as a trade
("Option A trades X for Y"). Then apply the three lenses: the 10x test (what scale
do we actually expect, with what evidence?), cost of delay (what does shipping
later actually cost?), and "what would have to be true?" for each option to be
right. Finally, state which context-facts are doing the deciding — and whether
your team had actually established those facts or was assuming them. What did
making it explicit reveal that the implicit version hid?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise's value is the near-universal discovery that the implicit trade-off
hid something important — usually that the team was <em>assuming</em> the deciding
facts rather than knowing them. A strong response produces the explicit trade
statement (which itself often clarifies a debate that felt intractable when the
options were argued abstractly), then runs the lenses honestly, and the payoff is
in the last part: identifying which context-facts decide (expected scale, cost of
delay, cost of later change) and confronting whether those facts were
<em>established or assumed</em>. Common revelations: (1) the "we need to build for
scale" position rested on a growth assumption nobody had actually checked (and
when you check, the real expected growth is far lower — or occasionally higher —
than the argument assumed, which changes the answer); (2) the "cost of delay" was
either overstated (there wasn't actually a hard deadline, so the rush was
self-imposed) or understated (there <em>was</em> a real window being missed); (3)
the "what would have to be true?" for the favored option turned out not to hold
(the conditions the "obvious" choice depended on weren't actually met), or the
conditions for the dismissed option <em>did</em> hold. The meta-lesson the
exercise teaches viscerally: <strong>most technical arguments are actually
disagreements about facts-of-the-world (how much will we scale? how much time do
we have? how expensive is it to change later?) disguised as disagreements about
technical merit</strong> — and when you make the trade-off explicit and ask what
would have to be true, you surface those hidden factual assumptions, at which
point the debate often resolves (because you can go find the facts) or at least
becomes productive (you're now arguing about assessable things, not values). The
other common finding: the deciding facts were never explicitly established at all —
the decision was heading toward being made on the intuition of whoever argued
best, with the actual determinants (expected growth, cost of delay) never named or
checked. Making it explicit forces those into the open, where they can be
estimated, researched, or at least acknowledged as assumptions. The habit to build
from this: whenever you catch a team arguing options in the abstract ("X is
better than Y"), interrupt with "let's name what each trades, and ask what would
have to be true for each to be right, and check whether we actually know those
things" — which converts unresolvable merit-debates into tractable fact-questions,
prevents both under- and over-engineering (by honestly assessing the real future
rather than defaulting to a reflex), and models the explicit-trade-off thinking
that is one of the most valuable things a technical leader does. If your analysis
reveals the team <em>had</em> established the deciding facts and reasoned
explicitly, excellent — that's a sign of a mature technical culture; more often,
the exercise reveals how much rides on unexamined assumptions, which is exactly
why making trade-offs explicit is worth the effort.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 9 — Running Design Reviews →](lesson-09-design-reviews){: .btn .btn-primary }
