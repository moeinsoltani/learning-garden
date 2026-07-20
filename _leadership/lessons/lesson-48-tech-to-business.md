---
title: "Lesson 48 — Connecting Technical Decisions to Business Outcomes"
nav_order: 4
parent: "Phase 9: Business & Product Thinking"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 48: Connecting Technical Decisions to Business Outcomes

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **translation table** — the mental mapping from technical properties (latency, uptime, debt) to business outcomes (conversion, churn, cost).
> - **conversion** — the share of visitors who become paying users.
> - **uptime** — the fraction of time a service works ("99.9%"); its opposite is **downtime**.
> - **time-to-market** — how fast an idea becomes a shipped product.
> - **loaded cost** — an employee's true cost: salary plus benefits, equipment, office, taxes.
> - **engineer-month / engineer-quarter** — units of effort: one engineer working one month/quarter; convertible to dollars.
> - **expected value** — probability × impact; how to price a risk rationally.
> - **business case** — the numbers-and-assumptions argument for an investment (Lesson 11).
> - **labeled assumptions** — stating openly which numbers are estimates, so the case is honest and checkable.
> - **payback** — how long until an investment has returned its cost.
> - **gold-plating** — polish beyond what pays off (Lesson 11).

## Concept

To get engineering investments approved and to make good decisions, you must **justify significant
technical work in business language** — because leadership decides in terms of business impact (revenue,
cost, risk, time), not technical merit. The skill is the **translation**: connecting technical things
(latency, reliability, tech debt) to their business consequences (conversion, churn, velocity/time-to-
market), treating engineering time as a real cost, and being honest — including when the honest answer is
"this doesn't pay off."

```
   THE TRANSLATION TABLE (technical → business)
   ┌──────────────────────────────────────────────────────┐
   │ latency          → conversion / user satisfaction       │
   │ reliability/uptime → churn / trust / revenue            │
   │ tech debt        → velocity → time-to-market / cost     │
   │ security         → risk (breach cost, compliance)       │
   │ observability    → incident cost / time-to-resolve      │
   │ ENG TIME = a real $ cost (salaries × time)              │
   │ RISK in expected-value terms (probability × impact)     │
   └──────────────────────────────────────────────────────┘
```

The reframe: **translate technical work into business impact — leadership funds business outcomes, not
technical elegance — and be honest, including when the answer is "it doesn't pay off."** An engineer
argues "we should improve reliability because it's the right thing"; a lead argues "improving reliability
from 99.5% to 99.9% would reduce the churn that's costing us ~$X/year, for an investment of Y — a clear
win." The translation (technical → business) is what gets engineering investments understood, funded, and
prioritized correctly.

---

## Going Deeper

### The translation table — technical to business

The core skill is a mental **translation table** from technical properties to business outcomes: (1)
**latency → conversion / satisfaction** (slower = fewer conversions, less satisfaction — often
quantifiable: "every 100ms costs X% conversion"); (2) **reliability/uptime → churn / trust / revenue**
(outages and bugs drive churn and erode trust, directly hitting revenue in a subscription business); (3)
**tech debt → velocity → time-to-market / cost** (debt slows the team, which delays features and raises
cost — debt's business cost is the slower delivery it causes); (4) **security → risk** (a breach's cost,
compliance requirements); (5) **observability → incident cost / resolution time** (better observability =
faster/cheaper incident resolution). Learning to translate any technical thing into its business
consequence is what lets you justify it in the terms leadership weighs.

### Engineering time as a real cost

A crucial reframe: **engineering time is a real, quantifiable cost** — engineers' salaries are large, so a
"2-engineer-quarter" project is a real dollar amount (2 engineers × 3 months × loaded cost). Treating
engineering time as the significant cost it is (not free) lets you do real cost-benefit analysis: is this
investment (X engineer-months = $Y) worth its benefit (the business value it creates)? This grounds
engineering decisions in economics — you're spending real money (engineering time), so the return should
justify it. Many engineering "should we do this?" questions become clear when you put a real cost on the
engineering time and compare it to the benefit.

### Risk in expected-value terms

Frame **risk** in **expected-value** terms: probability × impact. A risk (a potential outage, a security
vulnerability, a scaling failure) has an expected cost = (probability it happens) × (cost if it does). This
lets you reason about risk-mitigation investments rationally: spending $X to mitigate a risk is worth it if
X < the expected cost reduced (the reduction in probability × impact). "There's a 30% chance this scaling
issue causes a major outage next year, and an outage costs ~$Z, so the expected cost is 0.3 × Z — worth a
mitigation costing less than that." Expected-value thinking turns vague "we should be safer" into a rational
investment decision.

### Building a business case — numbers and labeled assumptions

To justify a significant investment, **build a business case** with numbers and clearly-labeled
assumptions: state the cost (engineering time as $), the benefit (the business impact, quantified via the
translation table), and the assumptions behind the numbers (labeled as assumptions, so they're honest and
checkable). "This observability investment costs 2 engineer-quarters (~$X); it should cut our incident
resolution time by ~50%, saving ~Y engineer-hours and ~$Z in incident impact per year [assuming current
incident rate and cost]; payback in ~N months." Numbers (even estimated) make it concrete and comparable;
labeling assumptions makes it honest (you're not pretending precision you don't have) and lets others
engage with the reasoning. A business case with real numbers and honest assumptions is far more persuasive
than "we should invest in observability because it's good practice."

### When the honest answer is "it doesn't pay off"

Crucially, the translation is **honest** — sometimes the analysis shows a technical investment **doesn't
pay off**, and you should say so. Not every technically-appealing thing is worth the business cost (a
rewrite that costs 6 engineer-months for marginal benefit, gold-plating with no business return, a
premature optimization). Being willing to conclude "this doesn't pay off — the cost exceeds the benefit"
(and not doing it) is part of business-honest engineering leadership. It builds credibility (you're not
just advocating for engineering's wish list; you apply honest cost-benefit even against engineering's
preferences) and it's the right call (spending engineering resources where they don't pay off is waste).
The translation cuts both ways: it justifies worthwhile investments AND rules out non-worthwhile ones.

{: .note }
> **Translate technical work into business impact — with real numbers, honest assumptions, both ways</br>**
> Leadership funds business outcomes, not technical elegance, so justify significant technical work in
> business language via the translation table: latency → conversion, reliability → churn/revenue, tech debt
> → velocity → time-to-market, security → risk, observability → incident cost. Treat <em>engineering time
> as a real dollar cost</em> (enabling genuine cost-benefit analysis), frame <em>risk in expected-value</em>
> terms (probability × impact), and build a <em>business case with real numbers and labeled assumptions</em>
> (concrete and honest). Crucially, the translation is honest and cuts both ways — sometimes the answer is
> "this doesn't pay off," and saying so builds credibility and is the right call. Justifying engineering in
> business terms is what gets worthwhile investments funded and prioritized correctly (and rules out the
> unworthwhile) — and it's what makes you a business-credible technical leader.

---

## Lab — Scenario

**The situation:** Your team keeps getting burned by production incidents that take a long time to diagnose
— you're flying blind without good observability (metrics, tracing, logging). You believe investing in a
proper observability setup would pay off, and you want to propose it. The investment: about **2
engineer-quarters** (2 engineers for a quarter) to build it out. You need to make the business case to get
it approved.

**Build the business case for the 2-engineer-quarter observability investment** — with numbers and clearly
labeled assumptions. Then note the principles and mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong business case states the cost (eng time as $), the quantified benefit (via the translation table),
labeled assumptions, and a payback — honestly. Example:
<br><br>
<strong>The cost:</strong> "2 engineer-quarters = 2 engineers × 3 months. At a loaded cost of ~$75k/engineer/
quarter [assumption: loaded cost including salary, benefits, overhead], that's <strong>~$150k</strong> of
engineering investment." <em>(Engineering time as a real dollar cost.)</em>
<br><br>
<strong>The benefit (quantified via translation):</strong> "The problem: we're getting burned by incidents
that take a long time to diagnose without observability. Currently [assumptions, labeled]: we have ~2 major
incidents/month, each taking ~4 hours to diagnose and resolve, involving ~3 engineers = ~24 engineer-hours
per incident, plus customer impact. <br>
• <strong>Faster incident resolution</strong>: good observability should cut diagnosis time by ~50%
[assumption], saving ~12 engineer-hours/incident × 2 incidents/month × 12 months = ~288 engineer-hours/year
(~$X in engineering time). <br>
• <strong>Reduced incident impact / churn</strong>: faster resolution means less customer-facing downtime.
If our incidents currently cause ~[N hours] of degraded service/year, halving resolution time reduces the
customer impact — in a subscription business, downtime drives churn, so reducing it protects revenue
[reliability → churn → revenue; quantify if we have churn-from-incidents data]. <br>
• <strong>Prevented incidents</strong>: better observability also catches issues before they become
incidents (proactive), reducing incident frequency, not just resolution time [harder to quantify, labeled as
upside]." <em>(Benefit quantified via the translation table — incident cost, engineering time saved, churn/
revenue protected — with assumptions labeled.)</em>
<br><br>
<strong>The payback:</strong> "Rough payback: the engineering-time savings alone (~288 eng-hours/year, ~$X)
plus the reduced incident impact should recover the ~$150k investment within [~N months/quarters]
[assumption-dependent]. And there's significant upside I haven't fully quantified (prevented incidents,
faster feature debugging, developer productivity)." <em>(A payback estimate, honest about what's quantified
vs. upside.)</em>
<br><br>
<strong>Honest framing:</strong> "I've labeled my assumptions so you can pressure-test them — if our
incident rate or cost is different, the numbers shift. But even conservatively, I think this pays off,
mainly through faster incident resolution and reduced customer impact. If you think the assumptions are off,
let's refine them together." <em>(Honest about assumptions, open to scrutiny — builds credibility.)</em>
<br><br>
<strong>Principles:</strong> (1) <strong>Engineering time as a real $ cost</strong> — the 2 eng-quarters
becomes ~$150k, enabling real cost-benefit. (2) <strong>Quantify the benefit via translation</strong> —
incident resolution time → engineering hours saved + customer impact/churn → revenue, with numbers. (3)
<strong>Label assumptions</strong> — every number's assumption stated, so it's honest and checkable (not
false precision). (4) <strong>Payback / ROI</strong> — compare cost to benefit over time. (5) <strong>Honest
about quantified vs. upside</strong> — clear on what's estimated vs. harder-to-quantify upside. (6)
<strong>Open to scrutiny</strong> — invite pressure-testing the assumptions (credibility). <strong>Mistakes
to avoid:</strong> (1) <strong>"we should invest in observability because it's good practice"</strong> — no
business case, just engineering preference (leadership can't weigh it); (2) <strong>not costing the
engineering time</strong> — treating eng time as free (it's ~$150k); (3) <strong>no numbers / all
qualitative</strong> — "it'll help a lot" isn't a business case; quantify via the translation; (4)
<strong>false precision / hidden assumptions</strong> — presenting made-up numbers as fact rather than
labeling assumptions (dishonest and non-credible); (5) <strong>over-claiming</strong> — inflating the
benefit or ignoring uncertainty (be honest, including the upside you can't fully quantify); (6) <strong>not
being willing to conclude it doesn't pay off</strong> — if the honest analysis showed the cost exceeded the
benefit, say so (here it pays off, but the discipline is honest either way — which builds the credibility
that gets your <em>worthwhile</em> cases approved).
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Business cases for eng investment | *An Elegant Puzzle*, Will Larson; *StaffEng* |
| Latency → business impact | Amazon/Google latency studies (100ms → conversion) |
| Cost of delay / tech debt economics | *Principles of Product Development Flow*, Reinertsen |
| Expected value & decisions | *Thinking in Bets*, Annie Duke; decision analysis |

---

## Checkpoint

**Q1.** Why must significant technical work be justified in business language, and what is the "translation
table"?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Significant technical work must be justified in business language because <strong>leadership decides on
investments in terms of business impact (revenue, cost, risk, time), not technical merit — so to get
engineering work understood, funded, and prioritized correctly, you must express it in the terms leadership
weighs</strong>. Leadership (who approve investments and set priorities) evaluate everything through a
business lens: does this make/save money, reduce risk, or serve the business's goals, and is it worth the
cost? They generally don't (and shouldn't have to) evaluate technical merit directly — "this is better
architecture" or "this is the right thing to do technically" doesn't tell them the business value, so they
can't weigh it against other investments. If you argue for engineering work in purely technical terms ("we
should improve reliability because it's the right thing," "we should pay down this tech debt because it's
bad"), leadership can't assess its business worth, so it competes poorly against things expressed in
business impact (and often loses funding/priority), even if it's genuinely valuable. If instead you
<em>translate</em> the technical work into its business impact ("improving reliability from 99.5% to 99.9%
would reduce the churn that's costing ~$X/year, for an investment of ~$Y — a clear win"), leadership can
understand and weigh it (compare the $Y cost to the $X benefit, against other options), so worthwhile
engineering investments get funded and prioritized correctly. So justifying in business language is what
connects engineering work to the terms decisions are made in — enabling it to be understood, funded, and
prioritized appropriately — versus staying in technical terms that leadership can't weigh (and that lose to
business-framed alternatives). This is essential for a lead: much of getting engineering investments
approved (reliability work, observability, tech-debt paydown, platform investment) depends on making the
business case, and a lead who can't translate technical value into business impact can't effectively
advocate for the engineering work that matters. The <strong>translation table</strong> is the mental mapping
from technical properties to their business consequences — the tool for doing this translation: (1)
<strong>latency → conversion / satisfaction</strong> (slower response = fewer conversions and less
satisfaction, often quantifiable — "every 100ms costs X% conversion"); (2) <strong>reliability/uptime →
churn / trust / revenue</strong> (outages and bugs drive customers away and erode trust, directly hitting
revenue — especially in subscription businesses where churn is the enemy); (3) <strong>tech debt → velocity
→ time-to-market / cost</strong> (debt slows the team, which delays features and raises cost — debt's
business cost is the slower, more expensive delivery it causes); (4) <strong>security → risk</strong> (a
breach's cost, compliance requirements — the expected cost of security failures); (5) <strong>observability
→ incident cost / resolution time</strong> (better observability means faster, cheaper incident resolution
and fewer incidents). The translation table captures that <em>every technical property has a business
consequence</em> — latency affects conversion, reliability affects churn, debt affects velocity and
time-to-market — and learning to map any technical thing to its business impact is the core skill that lets
you justify engineering work in the business terms leadership weighs. Using it, you translate "we should
reduce latency" into "reducing latency would improve conversion, worth ~$X" — turning a technical argument
into a business case. The translation table is thus the essential tool for connecting technical decisions to
business outcomes: it's how a lead expresses engineering work in the language of business impact, which is
what gets that work understood, funded, and prioritized — the difference between advocating for engineering
in terms leadership can weigh (business impact) versus terms they can't (technical merit).
</details>

**Q2.** Why treat engineering time as a real cost and frame risk in expected-value terms, and why is being
willing to say "this doesn't pay off" important?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Treating engineering time as a real cost</strong> matters because <strong>engineering time is
expensive (large salaries), so putting a real dollar figure on it enables genuine cost-benefit analysis —
grounding engineering decisions in economics rather than treating engineering effort as free</strong>.
Engineers' salaries are large, so a project measured in engineering time (a "2-engineer-quarter" effort) is
a substantial real dollar amount (2 engineers × 3 months × loaded cost ≈ a six-figure sum). Treating that
time as the significant cost it is (not free, not just "some engineering work") lets you do real
cost-benefit analysis: you can compare the investment (X engineer-months = $Y) against its benefit (the
business value it creates), and ask whether the return justifies the cost. Many engineering "should we do
this?" questions become clear when you put a real cost on the engineering time: a rewrite that "seems worth
it" might obviously not be when you price it at 6 engineer-months (~$X) against its marginal benefit; a
proposed feature's cost becomes weighable against its value. Without pricing engineering time, you can't do
real economics (everything seems potentially worth doing because the cost is invisible); with it, you spend
engineering resources (real money) where the return justifies it — which is the disciplined, business-honest
way to make engineering investment decisions. <strong>Framing risk in expected-value terms</strong>
(probability × impact) matters because <strong>it lets you reason about risk and risk-mitigation investments
rationally, turning vague "we should be safer" into a quantified decision</strong>. A risk (a potential
outage, a security vulnerability, a scaling failure) has an <em>expected cost</em> = (probability it happens)
× (cost if it does). This framing enables rational decisions about mitigation: spending $X to mitigate a
risk is worth it if X is less than the expected cost it reduces (the reduction in probability × impact).
"There's a ~30% chance this scaling issue causes a major outage next year, an outage costs ~$Z, so the
expected cost is 0.3 × Z — a mitigation costing less than that is worth it." This turns risk from a vague
"we should be careful / this feels risky" into a quantified, rational investment decision — you can weigh
the cost of mitigation against the expected cost of the risk, and decide accordingly (invest in mitigating
high-expected-cost risks, accept low-expected-cost ones). It avoids both over-investing in unlikely/
low-impact risks (wasting resources on remote possibilities) and under-investing in likely/high-impact ones
(ignoring real dangers) — by reasoning about expected value rather than fear or hand-waving. Why being
willing to say <strong>"this doesn't pay off"</strong> is important: <strong>because the honest cost-benefit
translation cuts both ways — not every technically-appealing thing is worth its business cost — and being
willing to conclude (and act on) "the cost exceeds the benefit" is what makes your analysis honest and
builds the credibility that gets your worthwhile cases approved</strong>. The translation from technical to
business isn't just a tool to justify engineering's wish list — it's an honest analysis, and honest analysis
sometimes shows a technical investment <em>doesn't</em> pay off: a rewrite costing 6 engineer-months for
marginal benefit, gold-plating with no business return, a premature optimization, an appealing-but-
unjustified project. Being willing to say "this doesn't pay off — the cost exceeds the benefit" (and not
doing it) is important for two reasons: (1) <strong>it's the right call</strong> — spending scarce
engineering resources where they don't pay off is waste (opportunity cost — that time could go to something
that does pay off), so honestly declining non-worthwhile investments is good economics; and (2) <strong>it
builds credibility</strong> — if you only ever use the translation to advocate <em>for</em> engineering
work (always concluding "yes, this pays off"), leadership learns you're just an advocate for engineering's
preferences (biased), and discounts your business cases. But if you apply honest cost-benefit even against
engineering's wishes — sometimes concluding "this doesn't pay off, let's not do it" — you demonstrate that
your analysis is genuine and unbiased, which makes leadership <em>trust</em> your business cases (including
the "yes" ones), so your <em>worthwhile</em> proposals get approved. The willingness to say no to
non-worthwhile engineering work is what makes your yes credible. So all three reflect business-honest
engineering leadership: price engineering time (real economics), frame risk in expected value (rational risk
decisions), and apply the cost-benefit honestly both ways (funding what pays off, declining what doesn't) —
which produces good decisions (spending where the return justifies it) and builds the credibility that makes
you an effective, trusted advocate for the engineering investments that genuinely matter. The translation is
a tool for honest decision-making, not just advocacy — and its honesty (including "no") is what makes it
powerful.
</details>

---

## Homework

Practice connecting technical decisions to business outcomes. (1) Take a technical investment you want (or a
past one) and build the business case: cost (engineering time as a real $ figure), benefit (quantified via
the translation table — latency→conversion, reliability→churn, debt→velocity, etc.), and labeled
assumptions. (2) Practice the translation table — for a few technical properties (latency, reliability, tech
debt, observability), state the business consequence. (3) Frame a risk in expected-value terms (probability
× impact) and reason about mitigation. (4) Honestly assess a technical investment you'd like — does it
actually pay off, or is it engineering preference? Reflect: do you justify engineering in business terms or
just technical merit, and what's one investment you should make the business case for (or honestly conclude
doesn't pay off)?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the tech-to-business translation skill — justifying engineering in business terms, which
gets it funded and prioritized. A strong response builds a real business case (cost as $, benefit quantified
via the translation table, labeled assumptions), practices the translation table, frames a risk in expected
value, and honestly assesses whether an investment pays off. The realizations: (1) <strong>justify in
business language</strong> — leadership decides in business terms (revenue, cost, risk, time), not technical
merit, so translate technical work into business impact via the translation table (latency→conversion,
reliability→churn, debt→velocity→time-to-market, security→risk, observability→incident cost) to get it
understood and funded; (2) <strong>engineering time is a real cost</strong> — pricing it (2 eng-quarters ≈
$150k) enables genuine cost-benefit analysis and grounds decisions in economics; (3) <strong>risk in
expected-value terms</strong> — probability × impact turns vague "be safer" into rational mitigation
decisions; (4) <strong>business case with numbers and labeled assumptions</strong> — concrete (numbers) and
honest (labeled assumptions), far more persuasive than "it's good practice"; (5) <strong>be honest, both
ways</strong> — sometimes the answer is "this doesn't pay off," and saying so is the right call (avoiding
waste) and builds the credibility that gets your worthwhile cases approved. On reflection, people commonly
find they argue for engineering work in technical terms (which leadership can't weigh), don't price
engineering time, and don't build real business cases — and sometimes advocate for technically-appealing
things that don't actually pay off. The highest-value change is usually: translate engineering work into
business impact with real numbers (making the business case), and apply honest cost-benefit both ways. The
meta-point: to get engineering investments approved and make good decisions, justify significant technical
work in business language — leadership funds business outcomes, not technical elegance — via the translation
table (technical → business consequence), treating engineering time as a real cost and risk in expected-value
terms, with honest numbers and assumptions. And the translation is honest, cutting both ways: it justifies
worthwhile investments AND rules out non-worthwhile ones, which is what makes you a business-credible
technical leader (trusted because your analysis is genuine, not just advocacy). This is the culmination of
the business/product thinking — connecting engineering to business impact — and it feeds the final phase-9
lesson, "should we build this?", which applies this business lens to the fundamental question of whether to
build something at all. The next lesson covers that shift — from "how do we build it" to "should we, and is
this the best way" — the lead's constant opportunity-cost lens.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 49 — "Should We Build This?" →](lesson-49-should-we-build-this){: .btn .btn-primary }
