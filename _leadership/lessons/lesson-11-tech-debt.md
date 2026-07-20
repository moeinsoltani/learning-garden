---
title: "Lesson 11 — Managing Technical Debt Strategically"
nav_order: 6
parent: "Phase 2: Technical Leadership"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 11: Managing Technical Debt Strategically

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **technical debt** — code whose current state makes future change slower, riskier, or more expensive (like interest on a loan).
> - **quadrant** (KWOD-rant) — one of four squares in a 2×2 diagram.
> - **prudent** (PROO-dent) — careful and wise; **reckless** — carelessly risky; **inadvertent** (in-ad-VER-tent) — unintentional.
> - **portfolio** (port-FOH-lee-oh) — a managed collection of investments; here: treating debts as items to weigh, not sins to purge.
> - **paydown** — reducing debt deliberately; **stabilization** — a focused period of fixing rather than building.
> - **churn** — how often code changes (Lesson 06); high-churn debt costs the most.
> - **gold-plate** — to add unnecessary polish beyond what's needed.
> - **opportunity cost** — the value of the best alternative you gave up.
> - **cycle time** — how long a piece of work takes from start to shipped.
> - **big-bang rewrite** — rebuilding a system from scratch in one giant risky effort; the **strangler-fig** incremental path is the usual alternative (Lesson 06).
> - **business case** — a pitch justifying work by its return (money, risk, speed), not its virtue.

## Concept

Engineers tend to treat technical debt as a moral failing — messy code someone
should feel bad about and "clean up." Leads have to treat it as what it actually
is: a **portfolio to manage**, with debt that's worth carrying and debt that's
worth paying down, evaluated in terms of business risk and velocity, not
cleanliness.

The foundational taxonomy is Martin Fowler's debt quadrant:

```
                 DELIBERATE                    INADVERTENT
              ┌──────────────────────┬──────────────────────┐
   RECKLESS   │ "no time for design, │ "what's layering?"   │
              │  just ship it"       │ (didn't know better) │
              ├──────────────────────┼──────────────────────┤
   PRUDENT    │ "we'll ship now and  │ "now we know how we  │
              │  deal with the       │  should have done it" │
              │  consequences" ← OK! │ (learned in hindsight)│
              └──────────────────────┴──────────────────────┘
```

The key insight: **prudent, deliberate debt is a legitimate tool**, not a
failure. Shipping a simpler version now and refactoring later — knowingly, as a
conscious trade — is often exactly right (Lesson 8's speed-vs-quality trade). The
debt to worry about is the *reckless* kind (skipping design out of laziness) and
the *accumulating unmanaged* kind (nobody's tracking it, it compounds silently
until the system is unworkable). And the framing that matters most for a lead:
debt has to be communicated to non-engineers in *their* language — risk and
velocity and cost, not "the code is ugly."

---

## Going Deeper

### Debt is about the future cost, not the aesthetics

Technical debt isn't "ugly code" — it's *code whose current state makes future
change slower, riskier, or more expensive*. That reframe matters because it makes
debt *measurable in business terms*: this debt costs us X because every feature
in this area takes twice as long, or because this fragile component causes Y
incidents a quarter, or because onboarding to this system takes months. Ugly code
that's stable and rarely touched isn't urgent debt (it's not costing you much);
clean-looking code in a critical, high-churn area with a subtle fragility might be
expensive debt. Evaluate debt by its *future cost given how the code is actually
used*, not by how it looks — which also tells you which debt to pay first (the
high-cost, high-churn areas, not the ugliest).

### Making debt visible to non-engineers

The lead's hardest debt task is getting *non-engineers* (PMs, execs) to fund
paying it down — and that requires translating out of engineer-language. "The code
is a mess and we need to refactor" is unpersuasive and sounds like engineers
wanting to gold-plate. The persuasive version speaks their language: **risk**
("this component causes 40% of our incidents; each outage costs us customer trust
and $X"), **velocity** ("features in this area take 3x longer than they should;
we're leaving speed on the table every sprint"), and **opportunity cost**
("because of this, we can't build [thing the business wants] without a rewrite").
Quantify where you can (incident rates, cycle time, the time features take).
You're making a *business case for an investment*, not asking permission to clean
up — Lesson 48 (tech-to-business translation) develops this fully.

### The 20% myth vs debt budgets

A common but weak approach: "we'll spend 20% of every sprint on tech debt." It
sounds disciplined but often fails — the 20% gets raided under deadline pressure,
or spent on low-value cleanup because it's undirected. Better: treat debt paydown
as *prioritized work competing on its merits* — the highest-cost debt (by the
risk/velocity measure) gets funded like any other high-value work, justified by
its business case, and scheduled deliberately (sometimes a focused
"stabilization" period, sometimes woven into feature work in the affected area —
"we're building feature X here anyway, so we pay down this area's debt as part of
it"). The goal is *strategic* paydown of the debt that matters, not a ritual
percentage.

### When to rewrite (rarely) vs strangle

The most dangerous debt response is the **big-bang rewrite** — "this is so bad
we should rebuild it from scratch." Rewrites are seductive and usually a trap:
they take far longer than estimated, deliver no value until done (a multi-quarter
period of pure risk with nothing shipped), often reintroduce old bugs while
missing the accumulated fixes and edge-cases the old system handled, and
frequently fail entirely (Joel Spolsky's famous "never rewrite" warning). The
usually-better path is **incremental**: the strangler-fig pattern (Lesson 6) —
build the new alongside the old, migrate piece by piece, deliver value
continuously, and keep the option to stop. Reserve full rewrites for the rare
cases where the old system genuinely can't be incrementally improved (a dead
platform, a fundamentally wrong core). The lead's instinct should be "how do we
improve this incrementally?" not "let's rebuild it."

{: .note }
> **"The code is ugly" is not a business case</br>**
> The single most common reason tech-debt work doesn't get funded is that
> engineers pitch it in engineer terms — cleanliness, elegance, "the right way" —
> to people who (rightly) don't care about code aesthetics, only business
> outcomes. Reframe every debt pitch as risk, velocity, or cost: not "this module
> is a mess" but "this module causes a third of our incidents and doubles the time
> for any change in it — investing two weeks now saves us [quantified] and reduces
> [risk]." The debt is the same; the framing is the difference between funded and
> dismissed.

---

## Lab — Scenario

**The situation:** Your team is slowed by a legacy component — the payments
integration layer. It's fragile (causes several production incidents a quarter),
undocumented, and every feature that touches payments takes roughly three times
longer than it should because engineers have to work around its quirks. You want
to spend about a quarter of the team's capacity over the next three months
stabilizing and refactoring it. Your PM, Jordan, is skeptical — Jordan is under
pressure to ship customer-facing features and sees a quarter of "refactoring" as
a quarter of not delivering value.

**Write the pitch you'd make to Jordan to fund this work — in business terms, not
engineer terms.**

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong pitch translates entirely out of engineer-language into risk, velocity,
and cost — and frames the work as an <em>investment with a return</em>, not a
cleanup. It also respects Jordan's real constraint (shipping pressure) rather than
dismissing it. Example:
<br><br>
"Jordan, I want to make a business case for investing in the payments layer over
the next quarter — and I want to frame it in terms of what it costs us <em>not</em>
to. Right now this component is costing us in three concrete ways: <strong>Risk</strong>
— it causes [N] production incidents a quarter; each one is a payments outage,
which directly hits revenue during the outage and erodes customer trust in the
part of our product they care about most (their money). <strong>Velocity</strong>
— every feature that touches payments takes about 3x longer than it should because
engineers spend most of their time working around this thing's fragility and
quirks. That means every payments-related feature on your roadmap is silently
costing us 3x — we're paying that tax on every sprint, whether we acknowledge it
or not. <strong>Opportunity cost</strong> — there are things you want to build
[name them] that we effectively can't do at reasonable cost until this is fixed.
<br><br>
So the investment: about a quarter of the team's capacity for three months to
stabilize and refactor this. The return: after it, payments features ship roughly
3x faster (so the roadmap items behind this go from 'expensive/slow' to
'normal'), incidents drop [toward zero], and we unlock the [features] we can't
reasonably do now. Concretely, if we have [X] payments features coming up, the
speedup pays back the investment within [timeframe] and keeps paying after. This
isn't cleanup for its own sake — it's removing a tax we're paying on every
payments feature and a risk we're carrying on every deploy.
<br><br>
I also hear your constraint — you're under pressure to ship, and a quarter feels
like a lot. So a few things: we can <em>phase</em> it (start with the highest-risk
fragility — the incident sources — for quick risk reduction, then the velocity-
draining parts), we can weave some of it into payments features you want anyway
(we're in that code regardless), and we can keep shipping the highest-priority
customer work in parallel with a portion of the team. Let's look at the roadmap
together and figure out the sequencing that gets you the most value while we
address this — but I do think ignoring it means paying the 3x tax and the incident
risk indefinitely, which is the more expensive path."
<br><br>
<strong>Why this works:</strong> Every argument is in Jordan's terms (revenue,
customer trust, roadmap velocity, opportunity cost) with quantification where
possible — not a single mention of "clean code," "the right way," or "it's ugly."
It frames the work as an <em>investment with a computable return</em> (3x speedup,
incident reduction, unlocked features) rather than as a request to indulge
engineers. It makes the <em>cost of inaction</em> visible (the 3x tax is being paid
right now, silently, on every feature — a powerful reframe because Jordan is
already paying it without realizing). It respects Jordan's constraint (phasing,
weaving into feature work, parallel shipping) rather than presenting an all-or-
nothing ultimatum. And it invites collaboration on sequencing rather than
demanding a block of time — turning Jordan from an adversary into a partner
deciding how to invest. This is Lesson 48's tech-to-business translation applied,
and it's the difference between "engineers want to refactor" (dismissed) and "a
funded investment with a clear ROI" (approved). Common mistakes: (1) pitching it as
"the code is a mess and we need to clean it up" (engineer-language, sounds like
gold-plating, gets dismissed); (2) presenting it as all-or-nothing (a full quarter,
no shipping) rather than showing you understand Jordan's constraint and can phase/
parallelize; (3) failing to quantify (vague "it's slow and buggy" is far weaker
than "3x slower, N incidents a quarter"); (4) not naming the cost of <em>inaction</em>
(the strongest argument — you're already paying this tax, the only question is
whether to keep paying it).
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| The technical debt quadrant | Martin Fowler — <https://martinfowler.com/bliki/TechnicalDebtQuadrant.html> |
| Why rewrites usually fail | Joel Spolsky — "Things You Should Never Do" <https://www.joelonsoftware.com/2000/04/06/things-you-should-never-do-part-i/> |
| Strangler-fig incremental migration | Martin Fowler — <https://martinfowler.com/bliki/StranglerFigApplication.html> |
| Making the business case for debt | *The Manager's Path*, Camille Fournier |
| Debt as risk/velocity (tech→business) | Lesson 48; *An Elegant Puzzle*, Will Larson |

---

## Checkpoint

**Q1.** Explain why "prudent, deliberate" technical debt is a legitimate tool
rather than a failure — and what distinguishes it from the debt a lead should
actually worry about.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Prudent, deliberate debt (Fowler's quadrant) is a <em>conscious trade-off made
with awareness of the consequences</em> — "we'll ship the simpler version now and
deal with the refactoring later, knowingly, because shipping now is worth more to
us than the cleaner implementation." It's legitimate because it's exactly the
speed-vs-quality trade-off of Lesson 8, and often the <em>right</em> call: getting
a product to market to prove it, hitting a real deadline, deferring polish on
something that might change anyway — all can justify taking on debt deliberately,
the same way financial debt is a legitimate tool for funding something valuable now
against future repayment. The key is that it's <em>deliberate</em> (a choice, not
an accident) and <em>prudent</em> (you understand the cost you're deferring and
have a plan or intent to manage it). This is debt as a tool, not a moral failing —
and treating all debt as shameful "messy code" misses that the team that never
takes on prudent debt is probably over-engineering and shipping too slowly
(Lesson 8's premature-quality version of the same mistake). What a lead should
actually worry about is the <em>other</em> quadrants and the accumulation problem:
(1) <strong>Reckless debt</strong> ("no time for design, just hack it in" out of
laziness or lack of discipline, or "what's layering?" out of ignorance) — debt
taken without the conscious trade or the skill to know better, which tends to be
gratuitous (you got the downside without a real upside — you could have done it
right for little extra cost). (2) <strong>Unmanaged accumulation</strong> — the
real killer: debt that nobody is tracking, that compounds silently across many
small "just ship it" decisions until the system becomes slow, fragile, and
eventually unworkable, with no one having ever <em>decided</em> to let it get that
bad (it happened by a thousand un-tracked cuts). The distinction that matters for
the lead: prudent deliberate debt is <em>on the balance sheet</em> — chosen,
understood, and managed (you know it's there, why you took it, and roughly when/
whether you'll pay it); dangerous debt is <em>off the books</em> — taken
recklessly or accumulated invisibly, unmanaged, compounding. So the lead's job
isn't to eliminate debt (impossible and counterproductive — you'd never ship) but
to <em>manage the portfolio</em>: take prudent debt deliberately when the trade is
worth it, avoid reckless debt (build the discipline/skill to not create gratuitous
debt), and — most importantly — <em>track</em> the debt so it doesn't accumulate
invisibly, paying down the high-cost pieces strategically (by the risk/velocity
measure) before they compound into an unworkable system. Debt is a tool and a
portfolio, not a sin — and the leads who either treat all debt as shameful (and
so over-engineer) or ignore debt entirely (and so let it accumulate to
catastrophe) both mismanage it; the skill is the deliberate, business-informed
middle.
</details>

**Q2.** A senior engineer proposes rewriting a painful legacy system from scratch.
Why should your default be skepticism, and what's the usually-better alternative —
with the rare exception where a rewrite is justified?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Default skepticism because big-bang rewrites are one of the most reliable ways to
waste enormous effort and often fail outright — the seductive "let's just rebuild
it right" hides several severe traps. (1) <strong>They take far longer than
estimated</strong> — always; the old system's true complexity is invisible until
you try to replicate it, and the estimate never accounts for the accumulated edge-
cases and fixes the old code silently handles. (2) <strong>They deliver no value
until done</strong> — a multi-quarter (often multi-year) period where the team is
pouring effort into rebuilding what already exists, shipping nothing new, while the
business waits and competitors move; pure risk with no incremental return. (3)
<strong>They reintroduce solved problems</strong> — the old system, however ugly,
encodes years of bug fixes, edge-case handling, and hard-won production knowledge
(the "cruft" is often battle scars, not just mess); a rewrite starts fresh and
rediscovers those bugs one by one in production. (4) <strong>They frequently fail
entirely</strong> — abandoned half-done when the timeline blows out and the
business loses patience, or shipped and worse than what they replaced (Joel
Spolsky's famous warning that rewriting from scratch is "the single worst
strategic mistake" — Netscape's rewrite is the canonical cautionary tale). And
(5) the pain that motivated the rewrite often <em>isn't</em> actually best solved
by a rewrite — it's specific hot-spots (the fragile bits, the slow bits) that can
be fixed in place. The usually-better alternative is <strong>incremental
improvement</strong>, especially the strangler-fig pattern (Lesson 6): build the
new alongside the old, migrate piece by piece behind stable interfaces, deliver
value continuously (each migrated piece is a real improvement shipped), keep the
old system running the parts not yet migrated, and — crucially — retain the
<em>option to stop</em> at any point with value already delivered (vs the rewrite's
all-or-nothing bet). This attacks the highest-pain pieces first (the risk/velocity
hot-spots), so you get most of the benefit early, and it's far lower-risk (no
big-bang cutover, no long value-less period). The rare exceptions where a rewrite
<em>is</em> justified: (a) the old system is built on a genuinely dead or
unsupportable platform (a language/framework that's EOL with no path forward,
hardware that's disappearing) so incremental improvement isn't possible; (b) the
core architecture is fundamentally wrong for what's now needed in a way that can't
be incrementally reshaped (rare — most architectures can be strangled); (c) the
system is small enough that a rewrite is genuinely quick and low-risk (a small
component, not a core system). But these are the exception, and the bar should be
high — the engineer proposing a rewrite should have to show why <em>incremental</em>
won't work, not just that the current system is painful (painful is normal and
usually incrementally fixable). The lead's instinct should be "how do we improve
this incrementally, highest-pain-first?" and the rewrite reserved for the genuine
dead-end cases — because the enthusiasm for rewrites vastly exceeds the situations
where they're actually the right call, and a lead who greenlights rewrites easily
will preside over expensive failures, while one who defaults to incremental
improvement delivers the same relief with a fraction of the risk.
</details>

---

## Homework

Inventory your team's (or a system you know's) technical debt as a portfolio. List
the significant debt items, and for each, assess: which Fowler quadrant is it
(deliberate/inadvertent × reckless/prudent)? What's its actual *cost* in
risk/velocity terms (not aesthetics) — how much does it slow work or cause
incidents, weighted by how much that code is actually touched? Rank them by cost.
Then pick the top item and draft the business-terms pitch you'd make to fund
paying it down. Finally: is anything on your list a candidate someone might want to
"rewrite," and would incremental improvement work instead?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the portfolio-management mindset that separates strategic debt
handling from either ignoring debt or moralizing about it. A strong response
produces a real inventory with three shifts in thinking. (1) <strong>Cost over
aesthetics</strong>: ranking debt by <em>risk/velocity cost weighted by churn</em>
rather than by ugliness usually reorders the list surprisingly — the ugliest code
might be stable and rarely-touched (low actual cost, low priority), while a
less-obviously-messy but fragile, high-churn, incident-prone component is the most
expensive debt (high priority). This reveals that "which debt to pay first" is a
business-value question (highest cost, highest churn), not an aesthetic one, and
that some debt isn't worth paying at all (stable, rarely-touched — leave it). (2)
<strong>The quadrant assessment</strong> often reveals that much of the worst debt
was <em>accumulated unmanaged</em> (nobody tracked it; it compounded through many
small decisions) rather than deliberately taken — which is the dangerous kind, and
the exercise of tracking it is itself the first fix (debt on the books can be
managed; off-the-books debt compounds invisibly). (3) The <strong>business-terms
pitch</strong> for the top item forces the tech→business translation (Q from the
scenario): quantifying the cost in incidents/velocity/opportunity and framing it as
an investment with a return — practicing the skill that determines whether debt
work gets funded. The <strong>rewrite check</strong> is a valuable reality-test:
almost every team has something someone would love to rewrite, and honestly asking
"would incremental (strangler-fig) work instead?" usually reveals that it would
(the pain is specific hot-spots fixable in place, not a true dead-end requiring a
rebuild) — building the default-skepticism-toward-rewrites instinct. The meta-
outcome: you leave with a <em>prioritized, business-justified debt backlog</em>
rather than a vague sense that "the code needs cleanup" — debt items ranked by real
cost, each with a business case, and a clear-eyed read on which are worth paying
down (the high-cost/high-churn ones), which to leave (stable/low-cost), and which
tempting rewrite to resist in favor of incremental improvement. That's managing
debt as a portfolio: not eliminating it (impossible, and prudent debt is a tool),
not ignoring it (it compounds), and not moralizing about it (aesthetics aren't the
measure) — but tracking it, costing it in business terms, and strategically paying
down what actually matters. If the exercise reveals your team already tracks and
prioritizes debt this way and funds it with business cases — excellent, that's a
mature engineering culture; more commonly it reveals debt that's untracked,
mis-prioritized by ugliness rather than cost, and pitched (when at all) in
engineer-terms that don't get funded — which is exactly the gap this lesson closes.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 12 — How Much Should a Lead Still Code? →](lesson-12-how-much-to-code){: .btn .btn-primary }
