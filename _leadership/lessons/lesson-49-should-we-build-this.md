---
title: "Lesson 49 — \"Should We Build This?\" — The Question That Changes"
nav_order: 5
parent: "Phase 9: Business & Product Thinking"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 49: "Should We Build This?" — The Question That Changes

## Concept

The biggest mindset shift from senior engineer to lead is the question you ask. A senior engineer asks
**"how do we build this?"** — assuming the thing should be built, focusing on building it well. A lead
asks **"*should* we build this, and is this the best way?"** — questioning whether it's worth doing at
all, and whether building it (vs. buying, or not doing it) is the right approach. This is powered by
**opportunity cost** as a constant lens: every "yes" to one thing is a "no" to everything else that
engineering time could do.

```
   THE QUESTION THAT CHANGES
   ┌──────────────────────────────────────────────────────┐
   │ SENIOR ENGINEER: "how do we build this?"               │
   │   (assumes it should be built → builds it well)        │
   │ LEAD: "SHOULD we build this? Is this the best way?"    │
   │   (questions if it's worth doing → opportunity cost)   │
   │                                                        │
   │ • OPPORTUNITY COST: every yes = a no to everything else │
   │ • BUILD vs BUY vs OPEN-SOURCE vs DON'T                  │
   │ • cheapest test first (walking skeleton)               │
   │ • SUNK COST: past investment doesn't justify continuing │
   │ • KILL projects well when they don't pay off            │
   └──────────────────────────────────────────────────────┘
```

The reframe: **shift from "how do we build it" to "should we build it, and is this the best way" —
holding opportunity cost as a constant lens.** The senior engineer's job was to build things well; the
lead's job includes questioning whether things should be built at all, because engineering time is scarce
and every project chosen means others foregone. This means considering build-vs-buy, testing cheaply
before committing, ignoring sunk costs, and being willing to kill projects — all in service of spending
scarce engineering effort where it creates the most value.

---

## How It Works

### Opportunity cost — the lead's constant lens

The foundational concept: **opportunity cost** — the value of the best alternative you give up. Engineering
capacity is scarce, so **every "yes" to one project is a "no" to everything else that engineering time
could have done.** A lead holds this lens constantly: the question isn't just "is this project good?" but
"is this the <em>best</em> use of this engineering time, compared to the alternatives?" A project that's
good in isolation might be a poor choice if a more valuable one is foregone to do it. Thinking in
opportunity cost shifts you from "should we do this good thing?" (usually yes, in isolation) to "is this
the best thing to do with this scarce capacity?" (a much more disciplined question that prioritizes ruthlessly).

### Build vs buy vs open-source vs don't

For any capability, the options aren't just "build it" — they're **build vs. buy vs. use open-source vs.
don't do it at all.** Engineers default to building (it's what we do, and it's more interesting), but
building is often <em>not</em> the best option: (1) **buy** (a vendor/SaaS solution) may be far cheaper and
faster than building (especially for non-core capabilities); (2) **open-source** may give you 80% for free;
(3) **don't** — maybe the capability isn't actually needed, or a manual/simpler workaround suffices.
Building should be reserved for what's genuinely core (your differentiator) and where building is genuinely
better than the alternatives. The default-to-build instinct wastes enormous engineering effort
re-creating things that could be bought, used from open-source, or skipped — so always consider the full
menu (build/buy/OSS/don't), not just how to build.

### The cheapest test first — the walking skeleton

Before committing heavily to building something, **test the riskiest assumptions as cheaply as possible.**
Rather than building the full thing (a big bet) and discovering late it doesn't work/isn't wanted, do the
**cheapest test first**: a prototype, a "walking skeleton" (a minimal end-to-end version), a manual
"wizard-of-oz" version, a spike to de-risk the hard part. This validates the idea (does it work? is it
wanted? is the approach viable?) before the big investment — so if it's wrong, you learn cheaply, and if
it's right, you proceed with confidence. Cheapest-test-first is a habit that prevents expensive commitments
to unvalidated things (connects to the pilot framing, Lesson 36).

### Sunk cost — past investment doesn't justify continuing

A crucial discipline: **ignore sunk costs.** The <strong>sunk cost fallacy</strong> is continuing something
because of what you've already invested ("we've spent 3 months, we can't stop now"). But past investment is
gone regardless of what you do next — the only rational question is <strong>forward-looking</strong>: from
here, is continuing the best use of future resources (compared to alternatives)? "We've already spent 3
months" is irrelevant to whether the <em>remaining</em> work is worth it. Falling for sunk cost keeps teams
pouring resources into failing or superseded projects because stopping "wastes" the past investment (which
is already spent either way). The discipline is to evaluate from the present forward, ignoring what's
already sunk — which sometimes means killing a project you've invested heavily in.

### Killing projects well

Sometimes the right call is to **kill a project** — it's not paying off, a better option emerged, priorities
changed, or it's not going to work. Killing projects is a valuable (and underused) skill, because
continuing bad projects wastes scarce resources. Do it <strong>well</strong>: (1) make the decision on
forward-looking merits (ignoring sunk cost); (2) be honest and clear about why; (3) handle the human side
(the team invested effort — acknowledge it, ensure they don't feel it was their failure, redeploy them to
valuable work); and (4) extract the learning (what did we learn, even from a killed project). A team that
can kill projects well (rather than letting them zombie on) spends its resources far better — but it
requires the discipline to stop, and doing it in a way that doesn't demoralize the people involved.

{: .note }
> **Shift to "should we build this?" — opportunity cost as the constant lens</br>**
> The big shift from senior engineer to lead is the question: from "how do we build this?" (assumes it
> should be built) to "<em>should</em> we build this, and is this the best way?" (questions if it's worth
> doing). It's powered by <em>opportunity cost</em> — engineering capacity is scarce, so every yes is a no
> to everything else, making "is this the <em>best</em> use of this time?" the disciplined question.
> Consider the full menu — <em>build vs buy vs open-source vs don't</em> (the default-to-build instinct
> wastes effort). Test the riskiest assumptions <em>cheapest-first</em> (walking skeleton) before big
> commitments. <em>Ignore sunk costs</em> (past investment doesn't justify continuing — evaluate forward).
> And <em>kill projects well</em> when they don't pay off. This shift — questioning whether and how to
> build, holding opportunity cost constant — is what makes a lead spend scarce engineering effort where it
> creates the most value, rather than just building well whatever comes.

---

## Lab — Scenario

**The situation:** Your team is **three months into a six-month build** of an internal tool (say, a custom
deployment dashboard). It's going okay — about half done, the team's invested, and it'll do what you need
in another three months. But a **vendor just launched a product** that does about **80% of what your tool
would do**, for **$30k/year**. Now you face a decision: kill your build and buy the vendor tool, continue
your build, or pivot somehow. There's real pressure to continue ("we've already spent three months!").

**Recommend — kill, continue, or pivot — and write the memo.** Then note the principles and mistakes to
avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong analysis ignores the sunk cost, evaluates forward-looking with opportunity cost, and (usually)
recognizes buying the vendor tool is likely right — while handling it well. Example reasoning + memo:
<br><br>
<strong>The forward-looking analysis (ignore the sunk 3 months):</strong> "The 3 months already spent is
<em>gone</em> either way — it's irrelevant to the decision (sunk cost). The only question is: from here, what's
the best use of resources? Remaining to finish our build: ~3 more engineer-months (~$X in engineering time —
Lesson 48). The vendor: $30k/year, available now, does ~80% of what we need. So the real comparison is: spend
~3 more engineer-months (~$X) to build 100% of our need, OR pay $30k/year to get 80% now with zero
engineering time (freeing those 3 engineer-months for something else — opportunity cost)."
<br><br>
<strong>The recommendation (likely: buy, unless the 20% is critical):</strong> "My recommendation is to
<strong>buy the vendor tool and stop our build</strong> — unless the missing 20% is genuinely critical and
unachievable otherwise. Reasoning: (1) $30k/year is almost certainly far cheaper than 3 engineer-months of
build <em>plus</em> ongoing maintenance of a custom internal tool forever (internal tools have a long
maintenance tail we'd own). (2) It's available now (vs. 3 more months). (3) Crucially, buying frees ~3
engineer-months to spend on something more valuable than an internal tool — an internal deployment dashboard
is almost certainly not our differentiator or best use of scarce engineering time (opportunity cost). (4)
The 80% likely covers our core need; for the missing 20%, check if it's actually essential — often the 20%
we'd build is gold-plating we don't truly need, or we can work around it, or the vendor will add it. Only if
the 20% is genuinely critical AND unachievable with the vendor would continuing our build make sense."
<br><br>
<strong>The memo (structure):</strong> "<strong>Recommendation: adopt [vendor] and stop our custom
dashboard build.</strong> <br>
<strong>Context</strong>: We're 3 months into a 6-month build of a deployment dashboard. [Vendor] just
launched, covering ~80% of our needs for $30k/year. <br>
<strong>Analysis</strong>: The 3 months spent are sunk (irrelevant going forward). Forward-looking: finishing
costs ~3 more engineer-months (~$X) plus ongoing maintenance; the vendor costs $30k/year, available now, and
frees those engineers for higher-value work. An internal dashboard isn't our differentiator — those 3
engineer-months are better spent on [core product work]. <br>
<strong>The 20% gap</strong>: [Assess — is it critical? workaround-able? on the vendor's roadmap?] I believe
it's [not critical / workable], so 80% now beats 100% in 3 months at high cost. <br>
<strong>Recommendation</strong>: Adopt the vendor, redeploy the team to [higher-value work]. <br>
<strong>The team</strong>: The 3 months weren't wasted — we learned our requirements (which is why we can
evaluate the vendor well), and I'll make sure the team moves to impactful work, not feeling this was a
failure." <em>(Forward-looking, opportunity-cost-driven, honest about the 20%, handles the human side.)</em>
<br><br>
<strong>Principles:</strong> (1) <strong>Ignore the sunk cost</strong> — the 3 months are gone; decide
forward-looking (the "we've already spent 3 months!" pressure is the sunk-cost fallacy). (2) <strong>Opportunity
cost</strong> — the 3 freed engineer-months could do something more valuable than an internal tool (the real
cost of continuing). (3) <strong>Build vs buy</strong> — buying an 80% solution for $30k likely beats
building 100% for 3 engineer-months + perpetual maintenance, for a non-core tool. (4) <strong>Assess the
20% honestly</strong> — is it critical, or gold-plating/workable? (often the latter). (5) <strong>Total cost
of ownership</strong> — the custom tool's maintenance tail (owned forever) vs. the vendor's. (6) <strong>Kill
it well</strong> — honest reasoning, redeploy the team to valuable work, frame the 3 months as learning (not
failure), handle the human side. <strong>Mistakes to avoid:</strong> (1) <strong>continuing because of the
sunk 3 months</strong> — the classic sunk-cost fallacy (the pressure to continue is exactly this); (2)
<strong>ignoring opportunity cost</strong> — not valuing what the freed engineers could do instead; (3)
<strong>defaulting to "we'll just finish ours"</strong> — the default-to-build instinct, ignoring the
cheaper buy option; (4) <strong>ignoring the maintenance tail</strong> — a custom internal tool is owned and
maintained forever (a real ongoing cost vs. the vendor); (5) <strong>overvaluing the 20%</strong> — assuming
we need 100% when 80% likely suffices (gold-plating); (6) <strong>killing it badly</strong> — making the team
feel their 3 months were wasted/a failure (demoralizing) rather than framing it as learning and redeploying
them well; (7) <strong>not writing a clear memo</strong> — the decision needs clear reasoning to get buy-in
(especially against the sunk-cost pressure).
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Opportunity cost & prioritization | *An Elegant Puzzle*, Larson; economics of opportunity cost |
| Sunk cost fallacy | [Sunk cost (Wikipedia)](https://en.wikipedia.org/wiki/Sunk_cost); *Thinking, Fast and Slow* |
| Build vs buy | classic build-vs-buy analysis; total cost of ownership |
| Cheapest test / walking skeleton | *The Lean Startup*, Ries; Alistair Cockburn (walking skeleton) |

---

## Checkpoint

**Q1.** How does the question shift from "how do we build this?" to "should we build this?", and why is
opportunity cost the lead's constant lens?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The question shifts from <strong>"how do we build this?"</strong> (the senior engineer's question) to
<strong>"should we build this, and is this the best way?"</strong> (the lead's question) — a shift from
<em>assuming</em> the thing should be built and focusing on building it well, to <em>questioning</em> whether
it's worth doing at all and whether building it is the right approach. The senior engineer's job is to build
things well: given a thing to build, they focus on how to build it excellently (the design, the
implementation, the quality) — the "should we" is assumed (someone decided it should be built; their job is
the "how"). The lead's job expands to include the "should we": before (or instead of) asking how to build
something well, the lead asks whether it should be built at all (is it worth doing? is it the best use of
our effort?) and whether building it is the right approach (vs. buying, using open-source, or not doing it).
This is "the question that changes" because it's the fundamental mindset shift from IC to lead: the IC
optimizes the execution of chosen work (build it well), while the lead optimizes the <em>choice</em> of work
(should we do this, and how) — a shift from execution to judgment about what's worth executing. It's a bigger
shift than it sounds, because the "how do we build this?" mindset is deeply ingrained (it's what engineers
do and are rewarded for), and questioning whether to build at all (rather than diving into building) requires
consciously stepping back to the prior question. Why <strong>opportunity cost</strong> is the lead's constant
lens: <strong>because engineering capacity is scarce, so every "yes" to one project is a "no" to everything
else that engineering time could have done — which makes "is this the best use of this scarce time?" the
disciplined question that should govern what gets built</strong>. Opportunity cost is the value of the best
alternative foregone. Engineering capacity (people, time) is limited, so the team can only do a fraction of
the possible projects — which means <em>choosing</em> to do one project is <em>choosing not</em> to do
others (the ones that scarce capacity can't also do). So every project has an opportunity cost: the value of
the best alternative project that could have been done with that capacity instead. Holding this lens
constantly changes the question from "is this project good?" (evaluated in isolation — where most reasonable
projects look worth doing) to "is this project the <em>best</em> use of this scarce capacity, compared to the
alternatives?" (a comparative, prioritizing question). This matters because: (1) <strong>in isolation, most
projects seem worth doing</strong> (they have some value), so without opportunity cost, you'd say yes to too
much (everything good) and spread scarce capacity thin or do lower-value things; (2) <strong>with opportunity
cost, you prioritize ruthlessly</strong> — a project that's good in isolation might be a poor choice if a
more valuable one is foregone to do it, so you ask not "is this good?" but "is this the best thing we could
do with this capacity?", which focuses scarce effort on the highest-value work. So opportunity cost is the
lens that turns "should we build this?" from a yes/no-in-isolation question into a prioritization question
("is this the best use of scarce capacity vs. the alternatives?"), which is what disciplines a lead to spend
engineering effort where it creates the <em>most</em> value rather than on everything that has <em>some</em>
value. It's the "constant lens" because scarcity is always present (there's always more that could be done
than capacity allows), so every decision about what to build should be weighed against what else that
capacity could do. The shift to "should we build this?" powered by opportunity cost is thus the core of the
lead's judgment about what's worth doing: not just building chosen things well (the IC's excellence), but
choosing what to build wisely given scarce capacity and the alternatives foregone (the lead's prioritization)
— which is a higher-leverage contribution (choosing the right work matters more than executing the wrong
work well) and the defining mindset shift from senior engineer to lead.
</details>

**Q2.** Why should you consider "build vs buy vs open-source vs don't" (not just build), and why must you
ignore sunk costs?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You should consider <strong>"build vs buy vs open-source vs don't"</strong> (not just how to build) because
<strong>building is often not the best option, and the engineer's default-to-build instinct wastes enormous
effort re-creating things that could be bought, used from open-source, or skipped entirely</strong>. For any
capability, the real options are a menu: (1) <strong>build</strong> it yourself; (2) <strong>buy</strong> a
vendor/SaaS solution; (3) use <strong>open-source</strong>; or (4) <strong>don't</strong> do it at all (it's
not needed, or a simpler/manual workaround suffices). Engineers strongly default to <em>building</em> —
it's what we do, it's more interesting/satisfying, and there's a bias toward creating rather than adopting.
But building is frequently <em>not</em> the best choice: (1) <strong>buying</strong> a vendor solution is
often far cheaper and faster than building (you get a mature product now for a subscription, vs. months of
engineering to build plus perpetual maintenance) — especially for non-core capabilities (internal tools,
commodity functionality); (2) <strong>open-source</strong> may give you 80% of what you need for free
(leveraging others' work rather than re-creating it); (3) <strong>don't</strong> — sometimes the capability
isn't actually needed, or a manual/simpler workaround is fine (building something you don't really need is
pure waste). The default-to-build instinct, unchecked, leads to re-creating things that could be obtained
far more cheaply — spending scarce engineering effort (which has high opportunity cost) building commodity
or non-core things that a vendor, open-source, or a workaround could provide, when that effort should go to
what's genuinely core (the differentiator) and where building is genuinely better than the alternatives. So
considering the full menu (build/buy/OSS/don't) — rather than jumping to "how do we build it" — is essential
to spending engineering effort wisely: reserve building for what's core and genuinely better to build, and
buy/adopt/skip the rest. This is a key application of "should we build this?" — often the answer is "we
shouldn't build it; we should buy it / use open-source / not do it." Why you must <strong>ignore sunk
costs</strong>: <strong>because past investment is gone regardless of what you do next, so it's irrelevant to
the forward-looking decision — and continuing something because of what you've already spent (the sunk-cost
fallacy) keeps teams pouring resources into things that aren't the best use of <em>future</em> resources</strong>.
The sunk-cost fallacy is the tendency to continue something because of what you've already invested ("we've
spent 3 months, we can't stop now"). But this is irrational, because <em>the past investment is already
spent and unrecoverable no matter what you decide going forward</em> — whether you continue or stop, the 3
months are gone. So the past investment provides <em>no</em> information about whether continuing is worth it;
the only rational question is <strong>forward-looking</strong>: from here (this moment forward), is
continuing the best use of future resources, compared to the alternatives? "We've already spent 3 months" is
irrelevant to whether the <em>remaining</em> work (the future 3 months) is worth doing vs. the alternatives
(e.g., buying a vendor tool and freeing those 3 months). Falling for the sunk-cost fallacy causes real harm:
teams continue failing, superseded, or no-longer-worthwhile projects because stopping feels like "wasting"
the past investment (which is already wasted/spent either way) — so they throw good future resources after
sunk past ones, compounding the loss. The discipline is to <strong>evaluate every decision from the present
forward, ignoring what's already sunk</strong> — asking only "given where we are now, what's the best use of
our <em>future</em> resources?" — which sometimes means killing a project you've invested heavily in (because
from here, the remaining investment isn't the best use of resources, regardless of the sunk past investment).
This is hard emotionally (it feels like admitting the past investment was wasted, and like "giving up"), but
it's the rational, resource-optimizing discipline: the past is sunk, so decide based on the future. Both
points serve the "should we build this?" / opportunity-cost discipline: consider the full menu (don't
default to building — buy/OSS/skip may be better uses of scarce effort), and decide forward-looking (ignore
sunk costs — past investment doesn't justify continuing, only future value does). Together they keep a lead
spending scarce engineering resources on the best forward-looking use — building only what's worth building
(and better to build than alternatives), and continuing only what's worth continuing (regardless of past
investment) — which is the ruthless, opportunity-cost-driven prioritization that distinguishes a lead's
judgment about what's worth doing from the IC's focus on doing chosen things well.
</details>

---

## Homework

Practice the "should we build this?" lens. (1) For something your team is building (or about to), ask the
lead's questions: <em>should</em> we build this (is it worth doing, the best use of our scarce time — the
opportunity cost)? And is building the best way (vs. buy, open-source, or don't)? (2) For a new capability,
run the full menu (build/buy/OSS/don't) rather than defaulting to build. (3) Is there a project continuing
partly on sunk cost ("we've invested so much")? Evaluate it forward-looking — from here, is it the best use
of future resources? (4) Practice cheapest-test-first — for a risky idea, what's the cheapest way to validate
it before committing? Reflect: are you asking "how do we build it" or "should we build it," and is there a
project you should question, buy instead of build, or kill?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the "should we build this?" lens — the defining mindset shift from senior engineer to
lead. A strong response applies the lead's questions to real work (should we build this? is building the best
way?), runs the full build/buy/OSS/don't menu, evaluates a project forward-looking (ignoring sunk cost), and
practices cheapest-test-first. The realizations: (1) <strong>the question shifts from "how" to "should
we/what's best"</strong> — from assuming the thing should be built and building it well (IC) to questioning
whether it's worth doing and whether building is the right approach (lead) — a shift from execution to
judgment about what's worth executing; (2) <strong>opportunity cost is the constant lens</strong> —
engineering capacity is scarce, so every yes is a no to everything else, making "is this the best use of
scarce time?" the disciplined question that prioritizes ruthlessly; (3) <strong>consider build vs buy vs
open-source vs don't</strong> — building is often not best, and the default-to-build instinct wastes effort
re-creating what could be bought, adopted, or skipped; reserve building for what's core; (4) <strong>ignore
sunk costs</strong> — past investment is gone regardless, so decide forward-looking (is continuing the best
use of future resources?), which sometimes means killing a heavily-invested project; (5) <strong>test
cheapest-first</strong> — validate risky assumptions cheaply before big commitments; and <strong>kill
projects well</strong> when they don't pay off. On reflection, people commonly find they operate in "how do
we build it" mode (assuming things should be built), default to building rather than considering buy/OSS/
don't, and sometimes continue projects on sunk-cost momentum. The highest-value realization is usually either
a project to question (is it worth doing?), a capability to buy/adopt rather than build, or a project to kill
(evaluated forward-looking). The meta-point: the biggest shift from senior engineer to lead is the question —
from "how do we build this?" (execute chosen work well) to "should we build this, and is this the best way?"
(choose the right work) — powered by opportunity cost (scarce capacity, every yes a no to alternatives). This
means questioning whether to build at all, considering the full menu (build/buy/OSS/don't), testing cheaply
before committing, ignoring sunk costs, and killing projects that don't pay off — all to spend scarce
engineering effort where it creates the most value. Choosing the right work is higher-leverage than executing
the wrong work well, which is why this shift defines the lead's contribution. This completes Phase 9 (Business
& Product Thinking): the business model, product strategy, metrics, connecting tech to business, and "should
we build this?" — the broader thinking that makes a lead a partner in deciding what to build and why, not
just how. Business and product thinking elevates a technical leader into a business leader who directs
engineering effort toward what matters most. The next phase (Project Leadership) turns to executing the work
you've decided is worth doing — planning, estimation, risk, and delivering projects successfully.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 10 — Project Leadership (Lesson 50: Planning and Roadmaps) →](lesson-50-planning-roadmaps){: .btn .btn-primary }
