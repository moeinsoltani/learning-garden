---
title: "Lesson 46 — Customer Needs and Product Strategy"
nav_order: 2
parent: "Phase 9: Business & Product Thinking"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 46: Customer Needs and Product Strategy

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **jobs-to-be-done (JTBD)** — the lens that customers "hire" a product to accomplish a *job* (the underlying need), not for its features ("they don't want a drill, they want a hole").
> - **segment** — a group of customers with similar needs; **persona** — an invented archetypal user representing a segment.
> - **monolithic** — treated as one uniform block; "the customer" isn't.
> - **choosing what to disappoint** — strategy's essence: deciding which customers and requests you deliberately *won't* serve.
> - **bloated** — swollen with too many features; **incoherent** — without a unifying logic.
> - **roadmap** — the planned sequence of what to build; good ones follow strategy, bad ones are a **grab-bag** (random assortment).
> - **differentiator** — what makes your product meaningfully different from competitors.
> - **product thinking** — the PM's discipline of deciding what's worth building and why.
> - **fluff** — impressive-sounding filler without substance (Lesson 06).
> - **lens** — a way of looking at something that reveals particular aspects.

## Concept

To be a real partner to your PM (and to make good engineering-informed product decisions), you need to
understand the **product thinking** they do — not to become a PM, but to speak the language and
contribute meaningfully. Two core ideas: **jobs-to-be-done** (people "hire" a product to do a job — focus
on the underlying need, not the features) and **product strategy as choosing what to disappoint** (a
strategy is defined as much by what you *won't* do and who you *won't* serve as by what you will).

```
   PRODUCT THINKING (enough to be a real partner)
   ┌──────────────────────────────────────────────────────┐
   │ JOBS-TO-BE-DONE: people "hire" a product for a JOB      │
   │   (the underlying need), not for features               │
   │   "they don't want a drill, they want a hole"           │
   │ STRATEGY = CHOOSING WHAT TO DISAPPOINT                   │
   │   a strategy is defined by what you WON'T do & who you   │
   │   WON'T serve (saying yes to everything = no strategy)   │
   └──────────────────────────────────────────────────────┘
```

The reframe: **product strategy is about focus — choosing which customers and needs to serve (and which
to deliberately disappoint) — and understanding the underlying job, not just the requested features.** A
product that tries to serve everyone and do everything has no strategy (and becomes an incoherent mess);
a good strategy makes hard choices about who and what to focus on. And understanding customers means
understanding the <em>job</em> they're trying to get done (the real need), not just the features they
request.

---

## How It Works

### Jobs-to-be-done — the underlying need

**Jobs-to-be-done (JTBD)** is a lens: people "hire" a product to do a **job** — to make progress on
something they're trying to accomplish — not for its features per se. The classic line: "people don't
want a quarter-inch drill, they want a quarter-inch hole" (and really, they want to hang the picture).
Focusing on the <em>job</em> (the underlying need/progress the customer wants) rather than the features
leads to better products: you solve the real need (which might be met differently than the requested
feature), you understand <em>why</em> customers use you (and what would make them switch), and you avoid
building features that don't actually serve a real job. It connects to translating complaints into signal
(Lesson 42) — understand the job behind the request.

### Segments and personas — without the fluff

Products serve different **segments** (groups of customers with different needs) and **personas**
(archetypal users). The useful core (without the marketing fluff): different customers have different
needs, and you can't serve all of them equally — so understanding your segments (who your customers are,
what different groups need) lets you make deliberate choices about who to focus on. The practical value is
recognizing that "the customer" isn't monolithic — a feature great for one segment may be irrelevant or
harmful for another — so product decisions involve choosing which segments to prioritize.

### Product strategy = choosing what to disappoint

The deepest idea: **a product strategy is defined by what you choose NOT to do and who you choose NOT to
serve** — strategy is about focus and trade-offs, so "choosing which customers to disappoint" is the
essence of it. A product that says yes to every customer request and tries to serve every segment has
<em>no</em> strategy — it becomes an incoherent, bloated mess that serves no one well (trying to please
everyone pleases no one). A real strategy makes hard choices: <em>these</em> customers and <em>these</em>
needs are our focus; <em>those</em> we deliberately won't serve well (we'll disappoint them). This focus
is what makes a product coherent and excellent for its target — and the discipline to say no (to
customers, segments, features outside the strategy) is what protects it. Understanding this helps you (and
your team) see why the answer to "can we add X?" is often (correctly) "no — that's not our strategy."

### Roadmap logic — why these, in this order

A **roadmap** should follow from the strategy — the sequence of what to build reflects the strategic
priorities (serving the target customers' most important jobs first, building toward the strategic goals).
Understanding roadmap logic means seeing <em>why</em> certain things are prioritized (they serve the
strategy/the key jobs) and others aren't (they don't, even if some customer asked) — rather than a roadmap
being a random list of requested features. A good roadmap has a logic (this before that, because...); a
bad one is a grab-bag. As an eng lead, understanding the logic lets you contribute to and question the
sequence intelligently.

### Where engineering insight changes product strategy

Engineering isn't just a consumer of product strategy — **engineering insight can change it.** Engineers
see things product can't: what's newly <em>possible</em> (a technical capability that opens a new product
direction), what's <em>cheap vs expensive</em> (a requested feature that's trivial, or one that's
enormously costly — which changes its priority), where the <em>real technical leverage</em> is (a platform
investment that would enable many future features). A good eng lead surfaces these — "we could do X, which
isn't on the roadmap but is now easy and could be big," or "that feature is 10x more expensive than the
alternative that serves the same job" — genuinely informing product strategy with engineering reality.
This is the payoff of understanding product thinking: you can contribute engineering insight <em>to the
strategy</em>, not just execute it.

{: .note }
> **Product strategy is choosing what to disappoint — and understanding the job, not just the features</br>**
> To be a real partner to your PM, understand the product thinking: <em>jobs-to-be-done</em> (people
> "hire" a product for a job — the underlying need — not for features; solve the real job, not the literal
> request) and <em>strategy as choosing what to disappoint</em> (a strategy is defined by what you won't do
> and who you won't serve — saying yes to everything is no strategy and makes an incoherent mess; focus is
> what makes a product excellent for its target). Understand segments (customers aren't monolithic) and
> roadmap logic (the sequence follows the strategy). And crucially, engineering insight can <em>change</em>
> product strategy — you see what's newly possible, what's cheap vs expensive, where the real leverage is —
> so understanding product thinking lets you contribute to the strategy, not just execute it. This makes
> you a genuine partner in deciding what to build.

---

## Lab — Scenario

**The situation:** Your PM has a roadmap with **ten features** for the next two quarters. You have a
strategy memo (below) that states the product's focus. Your job is to think critically about the roadmap
against the strategy — which features truly matter, and which actively <em>hurt</em>.

**The strategy memo (provided):** "Our strategy is to be the best tool for <strong>small engineering
teams (5-20 people)</strong> who want <strong>simple, fast CI/CD without complexity</strong>. We win by
being radically simpler and faster to set up than the complex enterprise tools. We are NOT trying to serve
large enterprises with complex compliance needs, and we compete on <strong>simplicity</strong>, not
feature-completeness."

**The ten features (examples):** (1) one-click setup for common stacks; (2) an advanced enterprise RBAC/
compliance system; (3) faster build times; (4) a complex, highly-configurable pipeline builder with 50
options; (5) great docs and onboarding; (6) SSO/SAML enterprise auth; (7) a clean, simple default that
works out of the box; (8) a marketplace of 200 plugins; (9) clear error messages when builds fail; (10) a
super-configurable enterprise audit-logging system.

**Argue which three matter most and which three actively hurt** — using the strategy. Then note the
principles and mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong analysis judges each feature against the strategy (simple, fast, for small teams; NOT enterprise;
compete on simplicity), recognizing that features contradicting the strategy actively hurt.
<br><br>
<strong>Three that matter most (serve the strategy — simplicity, speed, small teams):</strong> <br>•
<strong>(1) One-click setup for common stacks</strong> — directly serves "radically simpler and faster to
set up," the core differentiator. <br>• <strong>(7) Clean, simple default that works out of the box</strong>
— the essence of the simplicity strategy; this <em>is</em> the product's value. <br>• <strong>(3) Faster
build times</strong> (or 5, great docs/onboarding, or 9, clear errors — all reinforce simple+fast for small
teams) — I'd pick faster builds as it directly serves "fast," a core promise. <em>(These three most directly
advance the strategy: simple, fast, easy for small teams.)</em>
<br><br>
<strong>Three that actively HURT (contradict the strategy — enterprise complexity):</strong> <br>•
<strong>(4) Complex, highly-configurable pipeline builder with 50 options</strong> — directly contradicts
"simple, without complexity"; adding complexity is the <em>opposite</em> of the strategy, and it makes the
product worse for the target (small teams who want simplicity). This actively hurts. <br>• <strong>(2)
Advanced enterprise RBAC/compliance</strong> — serves the segment the strategy explicitly says NOT to serve
(large enterprises with compliance needs); it adds complexity for a non-target segment, hurting focus and
the simple experience. <br>• <strong>(10) Super-configurable enterprise audit-logging</strong> — same:
enterprise complexity for a non-target segment, contradicting the simplicity focus. <em>(These three pull
the product toward enterprise complexity — exactly what the strategy rejects — so they don't just fail to
help, they actively harm the product's coherence and its value for the target.)</em>
<br><br>
<strong>The key insight:</strong> Features that <em>contradict</em> the strategy (add complexity, serve
non-target segments) don't just fail to help — they <strong>actively hurt</strong>, because they erode the
product's focus and its core value (simplicity), make it worse for the target customers, and pull it toward
being an unfocused enterprise tool (competing on the wrong axis). This is "strategy as choosing what to
disappoint" in action: the enterprise features (2, 6, 8, 10) should be deliberately <em>not</em> built —
disappointing enterprise customers is the correct strategic choice, because serving them would ruin the
product for the target small teams. (Note: 6 SSO/SAML and 8 the 200-plugin marketplace are also
strategy-contradicting — enterprise auth and complexity — arguably worse picks than some; the point is
recognizing the enterprise/complexity features as harmful.)
<br><br>
<strong>Principles:</strong> (1) <strong>Judge features against the strategy</strong> — not "is this a good
feature?" but "does this serve <em>our</em> strategy (simple, fast, small teams)?" (2) <strong>Features
contradicting the strategy actively hurt</strong> — adding complexity/serving non-target segments erodes
focus and core value, not just "doesn't help." (3) <strong>Strategy = choosing what to disappoint</strong>
— deliberately NOT building the enterprise features (disappointing enterprises) is correct, because it
protects the product for the target. (4) <strong>Simplicity is the differentiator</strong> — so features
that add complexity attack the very thing they compete on. (5) <strong>Engineering insight</strong> — as
eng lead, you might also note which are cheap/expensive to reinforce the argument. <strong>Mistakes to
avoid:</strong> (1) <strong>judging features in isolation</strong> ("RBAC is a good feature") rather than
against the strategy (it's good for a product this one explicitly isn't); (2) <strong>"more features =
better"</strong> — the feature-completeness trap the strategy explicitly rejects; (3) <strong>not
recognizing active harm</strong> — treating off-strategy features as merely low-priority rather than
harmful (they erode the core value); (4) <strong>trying to serve everyone</strong> — wanting to do the
enterprise features <em>too</em> (which is exactly the no-strategy mess); (5) <strong>ignoring the "NOT"
in the strategy</strong> — the strategy explicitly says not-enterprise, so enterprise features are
off-strategy by definition.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Jobs-to-be-done | Clayton Christensen's JTBD; *Competing Against Luck* |
| Product strategy | *Good Strategy/Bad Strategy*, Rumelt; *Inspired*, Cagan |
| Strategy as choosing what not to do | Michael Porter, "What Is Strategy?" |
| Eng insight into product | Lesson 40 (the triad); *EMPOWERED*, Cagan |

---

## Checkpoint

**Q1.** What is "jobs-to-be-done," and why is "product strategy = choosing what to disappoint" a powerful
reframe?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Jobs-to-be-done (JTBD)</strong> is the lens that <strong>people "hire" a product to do a job — to
make progress on something they're trying to accomplish — rather than wanting the product's features for
their own sake</strong>. The classic illustration: "people don't want a quarter-inch drill, they want a
quarter-inch hole" (and really, they want to hang the picture / accomplish the underlying goal). The
customer's real motivation is the <em>job</em> — the progress or outcome they're trying to achieve — and the
product is just a means to it. Why JTBD matters: focusing on the job (the underlying need) rather than the
features leads to better products because (1) <strong>you solve the real need</strong> — understanding the
job the customer is trying to do lets you serve that need well, which might be met differently (and better)
than the specific feature they requested (they asked for a drill, but a better hole-making tool might serve
the job better); (2) <strong>you understand why customers use you</strong> — knowing the job they hire you
for tells you what actually creates value for them (and what would make them switch to a competitor that
does the job better); and (3) <strong>you avoid building features that don't serve a real job</strong> —
features that don't advance any real customer job are waste. It connects to translating complaints into
signal (Lesson 42): understand the job behind the request, not just the literal feature asked for. So JTBD
keeps product thinking focused on the real customer need (the job) rather than features in isolation, which
leads to solving what customers actually need. Why <strong>"product strategy = choosing what to disappoint"</strong>
is a powerful reframe: <strong>because it captures that strategy is fundamentally about focus and trade-offs
— a real strategy is defined by what you deliberately choose NOT to do and who you choose NOT to serve, and
a product that tries to serve everyone and do everything has no strategy and becomes an incoherent mess</strong>.
The common misconception is that a good product strategy means serving as many customers and building as
many good features as possible — but this is backwards. Strategy is about <em>focus</em>: choosing a target
(specific customers, specific needs) and committing to serving them excellently, which necessarily means
<em>not</em> serving others as well (deliberately disappointing them). A product that says yes to every
customer request and tries to serve every segment (never disappointing anyone) ends up: (1) <strong>incoherent
and bloated</strong> — accreting features and use cases with no unifying focus, becoming a mess that does
many things poorly rather than a few things excellently; (2) <strong>excellent for no one</strong> — trying
to please everyone, it pleases no one (a tool that serves both simplicity-seeking small teams and
complexity-needing enterprises serves neither well, because their needs conflict); (3) <strong>competing on
the wrong axes</strong> — without focus, it can't be the best at anything. So "choosing what to disappoint"
reframes strategy correctly: <strong>the essence of strategy is the hard choices about who and what NOT to
serve</strong> — <em>these</em> customers and needs are our focus (served excellently); <em>those</em> we
deliberately won't serve well (disappointing them is the correct choice, because serving them would ruin the
product for our target). This focus is what makes a product coherent and excellent for its target, and the
discipline to say no (to customers, segments, and features outside the strategy) is what protects it. The
reframe is powerful because it (a) corrects the "more is better" instinct (revealing that saying yes to
everything destroys strategy), (b) makes the value of focus and saying-no clear (they're not limitations but
the essence of a good strategy), and (c) gives a practical test for decisions: "does this serve our target/
strategy, or does it dilute our focus to serve someone we've chosen not to prioritize?" — which correctly
identifies off-strategy features as things to deliberately <em>not</em> build (even if some customer wants
them). For an eng lead, this understanding is valuable: it explains why the answer to "can we add X?" is
often correctly "no — that's not our strategy," and it helps the team resist the feature-creep and
serve-everyone pressures that destroy focus. Both JTBD (focus on the real job, not features) and
strategy-as-choosing-what-to-disappoint (focus on the target, deliberately not serving others) reflect that
good product thinking is about <em>focus</em> — on the real underlying needs, and on a chosen target served
excellently — rather than maximizing features and customers, which produces an unfocused mess.
</details>

**Q2.** Why do features that contradict the strategy "actively hurt" (not just fail to help), and how can
engineering insight change product strategy?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Features that contradict the strategy <strong>actively hurt</strong> (not just fail to help) because
<strong>they erode the product's focus and core value, make it worse for the target customers, and pull it
toward being an unfocused product competing on the wrong axes — so they cause real harm, not just neutral
non-contribution</strong>. It's tempting to think an off-strategy feature is merely low-value ("doesn't help
much") but harmless. But a feature that contradicts the strategy causes active damage: (1) <strong>it erodes
the core value / differentiator</strong> — if the strategy is to win on simplicity, a feature that adds
complexity (a 50-option configurable builder) directly attacks the very thing the product competes on,
making it less simple — so it degrades the product's core value, not just adds a marginal unused feature.
(2) <strong>it makes the product worse for the target customers</strong> — off-strategy features (enterprise
complexity, features for non-target segments) clutter and complicate the experience for the actual target
(the small teams who wanted simplicity now face a more complex product), actively harming the people the
product is <em>for</em>. (3) <strong>it dilutes focus and coherence</strong> — each off-strategy feature
pulls the product toward being an unfocused everything-tool, eroding the coherent focus that makes it
excellent for its target and moving it toward the incoherent mess that serves no one well. (4) <strong>it
competes on the wrong axis</strong> — building enterprise features when the strategy is small-team simplicity
pulls the product into competing with enterprise tools (on feature-completeness), an axis it explicitly
chose not to compete on and would lose. (5) <strong>it costs resources that should go to the strategy</strong>
— building off-strategy features consumes engineering effort that should advance the actual strategy
(opportunity cost). So an off-strategy feature doesn't just fail to advance the strategy — it <em>actively
undermines</em> it (eroding core value, harming the target experience, diluting focus, mis-competing,
wasting resources). This is why recognizing active harm matters: off-strategy features must be deliberately
<em>not built</em> (not merely deprioritized), because building them damages the product — "strategy as
choosing what to disappoint" means actively declining these, since serving the non-target (or adding
complexity) harms the target. How <strong>engineering insight can change product strategy</strong>:
engineering isn't just a consumer of product strategy — it can genuinely inform and change it, because
engineers see things product can't: (1) <strong>what's newly possible</strong> — a technical capability or
development (a new technology, an internal platform, something that's become feasible) can open a new product
direction that product wouldn't know to consider ("we could now do X, which isn't on the roadmap but is
newly possible and could be a significant product opportunity"); engineering's view of what's technically
possible can reveal strategic options. (2) <strong>what's cheap vs expensive</strong> — engineering knows
the real cost of features, which should change their priority: a requested feature that's trivially cheap
might be worth doing even if minor, while one that's enormously expensive might not be worth it despite
seeming valuable — and often there's a cheaper way to serve the same job ("that feature is 10x more
expensive than this alternative that serves the same customer need"); this cost reality should inform
strategic prioritization, and only engineering can provide it. (3) <strong>where the real technical leverage
is</strong> — engineering sees where a platform investment or technical capability would enable many future
features or unlock significant value, which is a strategic insight (investing there has outsized future
payoff) that product might not see. So a good eng lead <em>surfaces</em> these insights — what's newly
possible, what's cheap/expensive, where the leverage is — genuinely informing product strategy with
engineering reality, rather than just executing whatever product decides. This is the payoff of understanding
product thinking: you can contribute engineering insight <em>to the strategy</em> (making it better-informed
about technical possibility and cost) rather than being a passive executor — which makes engineering a real
strategic partner (the triad idea, Lesson 40, at the strategy level). Both points reflect deep product-
strategic thinking: off-strategy features actively harm (so focus requires actively declining them, not just
deprioritizing), and engineering insight is a genuine input to strategy (what's possible, what things cost,
where leverage is) — so an eng lead who understands product strategy both helps protect its focus (recognizing
and declining harmful off-strategy features) and improves it (contributing engineering reality), being a
true partner in what gets built and why.
</details>

---

## Homework

Deepen your product partnership. (1) For a feature your team is building (or a customer request), practice
the jobs-to-be-done lens — what <em>job</em> is the customer really trying to do (the underlying need), not
just the requested feature? Would something else serve the job better? (2) Do you understand your product's
strategy — who it's for, and crucially who/what it deliberately does NOT serve? If not, find out (ask your
PM, read the strategy). (3) Look at your roadmap against the strategy — do all the items serve it, or are
some off-strategy (or actively harmful)? (4) Have you surfaced engineering insight that could inform product
strategy (what's newly possible, cheap vs expensive, where the leverage is)? Reflect: how well do you
understand your product's strategy, and where could you contribute engineering insight to make it better?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds product-thinking partnership — understanding product strategy well enough to be a real
partner and to contribute engineering insight. A strong response applies the jobs-to-be-done lens to a real
feature/request (the underlying job, not the literal feature), checks understanding of the product's strategy
(especially who/what it does NOT serve), judges the roadmap against the strategy, and looks for engineering
insight that could inform strategy. The realizations: (1) <strong>jobs-to-be-done</strong> — people hire a
product for a job (the underlying need), not for features, so understanding the real job (not just the
requested feature) leads to better products (solving the real need, which might be met differently); (2)
<strong>strategy = choosing what to disappoint</strong> — a real strategy is defined by what you won't do and
who you won't serve; saying yes to everything is no strategy and makes an incoherent mess, while focus makes
a product excellent for its target; (3) <strong>off-strategy features actively hurt</strong> — they erode the
core value/differentiator, harm the target experience, dilute focus, and mis-compete, so they must be
deliberately not built (not just deprioritized); (4) <strong>engineering insight can change strategy</strong>
— you see what's newly possible, what's cheap vs expensive, where the technical leverage is, so surfacing
these makes you a strategic partner, not just an executor. On reflection, people commonly find they don't
fully know their product's strategy (especially the deliberate "NOT" — who it doesn't serve), that some
roadmap items are off-strategy, and that they haven't surfaced engineering insights that could inform
strategy. The highest-value realization is usually either learning the product strategy (if it was a gap) or
recognizing where engineering insight (cost reality, new possibility, leverage) could genuinely improve
product decisions. The meta-point: to be a real partner to your PM (and make good engineering-informed
product decisions), understand the product thinking — jobs-to-be-done (the real job, not features) and
strategy as focus (choosing what to disappoint) — well enough to judge features against the strategy
(recognizing off-strategy ones as harmful) and to contribute engineering insight (possibility, cost,
leverage) that can change the strategy. This elevates engineering from executor to strategic partner in
deciding what to build and why. It builds on the triad (Lesson 40) and business model (Lesson 45), and feeds
into connecting tech to business (Lesson 48) and "should we build this?" (Lesson 49). The next lesson turns
to metrics — speaking both business and engineering metrics natively, honestly connected, and navigating the
ways metrics get misused.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 47 — Metrics and KPIs →](lesson-47-metrics-kpis){: .btn .btn-primary }
