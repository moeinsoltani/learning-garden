---
title: "Lesson 42 — Customers and Users"
nav_order: 3
parent: "Phase 8: Stakeholder Management"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 42: Customers and Users

## Concept

Engineering exists to serve the humans on the other end — customers and users — but teams easily lose
touch with them, building based on assumptions, specs, and internal opinions rather than real user
reality. A lead's job includes **keeping the team connected to the actual humans** they're building
for: getting engineers direct exposure to users, translating complaints into engineering signal, and
navigating the hard cases (an enterprise escalation, saying no to a customer, weighing an anecdote
against data).

```
   DISCONNECTED TEAM               CONNECTED TEAM
   ─────────────────               ──────────────
   builds from specs &             engineers hear/see real users
   internal assumptions             (support rotations, calls)
   "users probably want X"         → real understanding of the pain
   never talks to a user           → build the right thing
        ↑ builds the wrong thing,       ↑ translates complaints into
          loses touch with reality        engineering signal
```

The reframe: **keep the team connected to real users — direct exposure beats second-hand
assumptions.** Engineers who never encounter real users build based on guesses ("users probably want
X"), which are often wrong; engineers with direct exposure (support rotations, customer calls, watching
real usage) develop genuine understanding of the actual pain, which leads to building the right thing.
The lead's job is to create that connection and to help translate what users say (complaints,
requests) into what engineering should actually do.

---

## How It Works

### Direct customer exposure for engineers

The most powerful practice: give engineers **direct exposure to users** — not just filtered second-hand
through product. Mechanisms: **support rotations** (engineers do a stint handling support tickets —
seeing real problems firsthand), **customer calls** (engineers join calls with users), **watching real
usage** (user research sessions, session recordings, usage data). Direct exposure builds genuine empathy
and understanding of the actual pain (which filtered summaries lose), motivates engineers (they see the
real humans they help), and leads to better decisions (grounded in reality, not assumptions). A team that
never touches a real user builds from guesses; one with direct exposure builds from understanding.

### Translating complaints into engineering signal

Users express problems in <em>their</em> terms — complaints, feature requests, confusion — which need
**translating into engineering signal** (the real underlying problem). A user saying "I want a button
here" is expressing a need (they're struggling to do X), not necessarily specifying the right solution
(the button might not be the best fix). The skill is hearing past the literal complaint/request to the
underlying problem (the interests-behind-positions idea, Lesson 37) — "what are they actually trying to
do, and what's the real pain?" — and then finding the right engineering response (which may differ from
what they literally asked for). Complaints are valuable data about real problems, but they need
interpretation, not literal implementation.

### Anecdote vs data

A key judgment: **weigh anecdote against data.** A single loud customer's complaint (an anecdote) is
vivid and compelling but may not represent the broader user base; aggregate data (usage metrics, how
many users hit the issue) shows the real prevalence. Both matter: anecdotes provide rich qualitative
understanding (the <em>why</em> and the human texture) that data lacks, while data provides the scale
(the <em>how many</em>) that anecdotes lack. The mistake is over-indexing on either — building for one
loud customer (anecdote) while ignoring what most users need (data), or drowning in metrics without the
qualitative understanding of <em>why</em>. Use anecdotes for depth and data for breadth, together.

### The enterprise-customer escalation

A common hard case: a **big enterprise customer escalates** — demanding a fix or feature, with sales/
account management pushing hard (they don't want to lose the customer). This creates pressure to drop
everything for one customer. Navigate it: (1) understand the real need (translate the demand); (2) weigh
it fairly (one big customer's demand vs. the broader roadmap and other users — anecdote vs. the whole);
(3) recognize the business reality (a major customer's threat to churn is a real business input, not to
be dismissed) while (4) not letting one loud customer hijack the whole roadmap (which happens easily and
harms everyone else). It's a judgment balancing the specific customer's importance against the broader
good — often needing negotiation and trade-offs, not just capitulation or refusal.

### Saying no to a customer

Sometimes you must **say no to a customer** — a request that doesn't fit the product direction, isn't
worth the cost, or would harm other users. Saying no is legitimate and necessary (you can't build
everything every customer asks for, and trying to makes an incoherent product). Do it well: (1) understand
and acknowledge their real need (so they feel heard); (2) be honest and clear (don't string them along
with false "maybe"); (3) explain the why (so it's a reasoned no, not a dismissal); and (4) where possible,
offer an alternative or the underlying-need solution. A good no (heard, honest, explained, with an
alternative) maintains the relationship far better than false promises or a curt dismissal. (English
track Lesson 18 / Lesson 47 for the language.)

{: .note }
> **Keep the team connected to real users — and translate what they say into what to build</br>**
> Engineering serves the humans on the other end, but teams lose touch, building from assumptions rather
> than reality. A lead keeps the team connected: give engineers <em>direct exposure</em> (support
> rotations, customer calls, real usage) — which builds genuine understanding and beats second-hand
> guesses. <em>Translate complaints into engineering signal</em> — hear past the literal request to the
> real underlying problem (the right fix may differ from what they asked for). Weigh <em>anecdote vs
> data</em> (a loud customer vs. the broader base — use anecdotes for depth, data for breadth). Navigate
> the <em>enterprise escalation</em> (balance the customer's importance against the broader good, don't
> let one customer hijack the roadmap) and <em>say no</em> well when needed (heard, honest, explained,
> with an alternative). Keeping the team grounded in real user reality is how you build the right thing.

---

## Lab — Scenario

**The situation:** A major enterprise customer (one of your biggest accounts) is demanding a specific
feature — a custom workflow that fits <em>their</em> unusual process. The problem: it **contradicts your
product direction** (it'd add complexity that hurts the experience for your many other, smaller
customers, and pulls the product toward being a bespoke tool for this one customer). Making it worse:
<strong>sales has already half-promised it</strong> to the customer to close/renew the deal, so the
customer expects it and sales is pushing you hard to build it. You're caught in a triangle: the customer,
sales, and your product's integrity. Navigate it.

**Write how you'd navigate this** — the customer, sales, and the product-direction tension. Then note the
principles and mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong approach digs to the real need, weighs it fairly (one customer vs. the whole), addresses the sales
mis-promise, and looks for a solution that serves the customer without harming the product. Example:
<br><br>
<strong>First — understand the real need (translate the demand):</strong> "Talk to the customer (or via
sales/account manager) to understand the <em>underlying</em> need, not just the literal feature. Why do
they want this custom workflow — what are they actually trying to accomplish? Often the literal request
(a bespoke workflow) is one solution to a real need that could be met another way — maybe a
configuration, an existing feature used differently, or an integration — that doesn't require bending the
product." <em>(Translate the complaint/demand into the real problem — the right solution may differ from
what they literally asked.)</em>
<br><br>
<strong>Weigh it fairly — one customer vs. the whole:</strong> "Assess honestly: this is one (big)
customer's demand, and building it would add complexity that hurts the many other customers and pulls the
product off-direction. That's the anecdote-vs-data / one-customer-vs-the-roadmap judgment — a major
customer matters (real revenue, churn risk), but not at the cost of harming the product for everyone else
and abandoning the direction. I need to balance the customer's importance against the broader good, not
just capitulate (hijacking the roadmap for one customer) or flatly refuse (ignoring a real business input)."
<br><br>
<strong>Look for the win-win:</strong> "Can I serve the customer's real need without harming the product?
Options: (a) meet the underlying need with existing/configurable capability; (b) a more general version of
the feature that helps others too (not bespoke to them); (c) an integration/extension point they can build
on; (d) if truly bespoke and necessary, a contained way that doesn't pollute the core product (a plugin, a
services engagement). Find the solution that serves them AND protects the product." <em>(Seek the option
that satisfies the real need without the product harm — the interests-based resolution.)</em>
<br><br>
<strong>Address the sales mis-promise (internally, honestly):</strong> "Talk to sales directly: 'I
understand you promised this to close the deal, and I want to help keep the customer happy — but building
it as-is would hurt the product for our other customers and take us off-direction. Let's figure out
together how to satisfy the customer's real need in a way that works — here are some options. And going
forward, let's loop engineering in <em>before</em> promising features, so we don't get into this bind.'"
<em>(Collaborative with sales (shared goal: happy customer), honest about the constraint, offers options,
and gently addresses the process problem — promising features without eng — for the future.)</em>
<br><br>
<strong>If it can't be reconciled — escalate the trade-off:</strong> "If there's a genuine conflict
(the customer insists on the bespoke thing, sales insists it's do-or-lose-the-deal, and it genuinely
harms the product), this is a real business trade-off that needs a decision above me — surface it clearly
(the customer value vs. the product cost, with options) to the people who own that trade-off (product
leadership), rather than either caving or refusing unilaterally." <em>(Escalate as a service — a genuine
business trade-off decision — with the framing and options.)</em>
<br><br>
<strong>Principles:</strong> (1) <strong>Translate the demand to the real need</strong> — the literal
feature may not be the only/best way to serve it. (2) <strong>Weigh one customer vs. the whole</strong> —
a big customer matters but shouldn't hijack the roadmap or harm other users (anecdote vs. data). (3)
<strong>Seek the win-win</strong> — serve the real need without the product harm (config, general feature,
integration, contained solution). (4) <strong>Handle sales collaboratively and honestly</strong> — shared
goal (happy customer), honest constraint, options — and fix the process (eng in the loop before promising).
(5) <strong>Escalate genuine trade-offs</strong> — if irreconcilable, surface the business decision to
whoever owns it, with options. <strong>Mistakes to avoid:</strong> (1) <strong>just capitulating</strong>
— building the bespoke feature because a big customer/sales demanded it (harms the product and everyone
else, hijacks the roadmap); (2) <strong>flatly refusing</strong> — dismissing a major customer's need and
a real business input (churn risk is real); (3) <strong>taking the demand literally</strong> — not digging
to the real need (where a non-harmful solution might exist); (4) <strong>fighting sales adversarially</strong>
— they're a partner (shared goal: keep the customer); collaborate; (5) <strong>not addressing the
process</strong> — letting sales keep promising features without eng (the root cause); (6) <strong>deciding
a big business trade-off unilaterally</strong> — if it's genuinely customer-value-vs-product-harm at stake,
that's a decision for product leadership, escalated with framing; (7) <strong>letting the loudest customer
set the roadmap</strong> generally — the anecdote-vs-data trap.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Customer connection / user empathy | *Inspired*, Cagan; support-rotation practices |
| Anecdote vs data | product-sense writing; *The Lean Startup* (validated learning) |
| Saying no (the language) | English for Work track, Lesson 18 (disagreeing & saying no) |
| Enterprise escalations & trade-offs | Lesson 44 (negotiation); Lesson 49 (should we build this?) |

---

## Checkpoint

**Q1.** Why is direct customer exposure for engineers valuable, and why must complaints be "translated into
engineering signal" rather than implemented literally?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Direct customer exposure for engineers is valuable because <strong>engineers who encounter real users
develop genuine understanding of the actual pain — which leads to building the right thing, motivates them,
and grounds decisions in reality — where engineers who only get filtered second-hand summaries build from
assumptions that are often wrong</strong>. When engineers never touch real users (building only from specs,
internal opinions, and filtered product summaries), they operate on <em>assumptions</em> — "users probably
want X," "this is the problem" — which are frequently wrong, because the reality of how users actually
struggle is hard to convey second-hand and easy to guess incorrectly. Direct exposure — support rotations
(handling real tickets), customer calls, watching real usage — fixes this: (1) <strong>it builds genuine
understanding and empathy</strong> — seeing/hearing real users struggle firsthand conveys the actual pain,
the real context, the human texture, in a way filtered summaries lose (a summary says "users find it
confusing"; watching a real user struggle shows you <em>how</em> and <em>why</em>); (2) <strong>it leads to
better decisions</strong> — grounded in the reality of what users actually need and do, rather than
assumptions, so the team builds the right thing; (3) <strong>it motivates engineers</strong> — seeing the
real humans they're helping (and the difference their work makes) is motivating, connecting the work to
real impact rather than abstract tickets. So direct exposure replaces guesses with understanding, which is
the difference between a team that builds from reality (the right thing) and one that builds from
assumptions (often the wrong thing). The lead's job is to create this connection because it doesn't happen
by default (engineers are often insulated from users). Why complaints must be <strong>translated into
engineering signal</strong> rather than implemented literally: <strong>because users express their problems
in their own terms — often as a specific requested solution — which reflects a real underlying need but is
not necessarily the right solution to build; the skill is hearing past the literal request to the real
problem</strong>. Users don't usually articulate their needs as well-formed engineering problems; they
express them as complaints ("this is frustrating"), confusion, or specific feature requests ("I want a
button here"). A literal request like "I want a button here" is the user's <em>proposed solution</em> to an
underlying need (they're struggling to accomplish X), but: (1) <strong>the user isn't a product/engineering
designer</strong> — their proposed solution (the button) may not be the best or right fix; they know their
<em>problem</em> (the pain) better than the <em>solution</em>; (2) <strong>the literal request may address
a symptom, not the root</strong> — implementing exactly what they asked might not actually solve their real
problem (or might solve it worse than another approach); (3) <strong>the real need might be better served
differently</strong> — once you understand what they're actually trying to do, a different solution (or an
existing capability) might serve it far better than the literal request. So the skill is to <strong>hear
past the literal complaint/request to the underlying problem</strong> (the interests-behind-positions idea,
Lesson 37) — "what are they actually trying to accomplish, and what's the real pain?" — and then find the
right engineering response, which may differ from what they literally asked for. This is "translating
complaints into engineering signal": treating the complaint/request as valuable <em>data about a real
problem</em> (which it is) that needs <em>interpretation</em> to find the right solution, rather than as a
literal spec to implement. Implementing complaints literally leads to building the wrong things (the user's
proposed solution rather than the best solution to their real need), a product that accretes point-solutions
without coherence, and missed opportunities to solve the real problem better. So both points reflect keeping
the team connected to real user reality <em>usefully</em>: direct exposure gives engineers genuine
understanding of the actual pain (vs. assumptions), and translating complaints into engineering signal turns
what users say (in their terms, as proposed solutions) into the right thing to build (addressing their real
underlying need) — together grounding the team in real user needs while applying engineering/product judgment
to serve those needs well, rather than building from guesses or implementing requests literally.
</details>

**Q2.** How should a lead weigh "anecdote vs data," and how do you navigate an enterprise-customer
escalation without letting one customer hijack the roadmap?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A lead should weigh <strong>anecdote vs data</strong> by <strong>using each for what it's good at —
anecdotes for qualitative depth (the why and human texture), data for scale (the how many) — and avoiding
over-indexing on either</strong>. An <strong>anecdote</strong> (a single customer's vivid complaint, one
user's story) is compelling and rich: it provides deep qualitative understanding — the <em>why</em> behind a
problem, the human context, the specific pain — which aggregate numbers can't convey. But it may <em>not
represent the broader user base</em>: one loud customer's issue might be idiosyncratic (unique to them), so
building for it could serve one while ignoring what most users need. <strong>Data</strong> (usage metrics,
how many users hit an issue, aggregate patterns) provides <em>scale and prevalence</em> — how widespread a
problem actually is, what most users do — which anecdotes lack. But data lacks the qualitative <em>why</em>
(metrics show <em>that</em> users drop off, not <em>why</em>). So the mistake is over-indexing on either: (1)
<strong>over-indexing on anecdote</strong> — building for one loud customer (or the most recent vivid
complaint) while ignoring what the data shows most users need (letting a compelling but unrepresentative
story drive decisions); (2) <strong>over-indexing on data</strong> — drowning in metrics without the
qualitative understanding of <em>why</em> (knowing what happens but not why, so you can't fix it well). The
right approach is to <strong>use them together</strong>: anecdotes for depth (rich understanding of the
problem and why) and data for breadth (how prevalent, how many affected) — so you understand both <em>why</em>
a problem occurs (anecdote) and <em>how much it matters</em> across the base (data). A vivid anecdote raises a
hypothesis; the data tells you if it's widespread; together they guide good decisions. How to navigate an
<strong>enterprise-customer escalation without letting one customer hijack the roadmap</strong>: (1)
<strong>understand the real need</strong> — translate the demand (the literal feature) to the underlying
need, because there may be a way to serve it that doesn't require bending the product; (2) <strong>weigh it
fairly — one customer vs. the whole</strong> — this is exactly the anecdote-vs-data / one-customer-vs-the-
roadmap judgment: a big customer's demand is one (loud, important) input, but building a bespoke feature
for them may harm the many other users and pull the product off-direction, so weigh the specific customer's
importance against the broader good rather than automatically capitulating; (3) <strong>recognize the real
business input without capitulating</strong> — a major customer's churn threat is a genuine business
consideration (real revenue at stake — not to be dismissed), but that doesn't mean automatically building
whatever they demand (which would let one customer hijack the roadmap and harm everyone else); it means
weighing it seriously as one factor; (4) <strong>seek the win-win</strong> — find a solution that serves the
customer's real need without harming the product (config, a general feature that helps others, an
integration, a contained solution) — often possible once you understand the real need; and (5)
<strong>escalate genuine trade-offs</strong> — if it truly can't be reconciled (real customer value vs. real
product harm), surface the business trade-off to whoever owns it (product leadership) with clear framing and
options, rather than caving or refusing unilaterally. The key to not letting one customer hijack the roadmap
is the <em>fair weighing</em>: a single big customer is an anecdote (important, but one data point), and
building bespoke things for whoever shouts loudest (or threatens churn) leads to an incoherent product that
serves loud customers while neglecting the silent majority (what the data shows most users need). So you
take the escalation seriously (it's a real input) but weigh it against the whole (the broader user base and
product direction — the data), seek solutions that serve the need without the harm, and escalate genuine
trade-offs — rather than either capitulating (roadmap hijacked) or dismissing (ignoring real business
input). Both points reflect balancing the vivid, loud, specific (anecdote, the escalating customer) against
the broad, aggregate, quiet (data, the whole user base) — using the specific for depth and the aggregate for
scale, and not letting the loud few override what most users need, while still taking real business inputs
seriously.
</details>

---

## Homework

Assess your team's connection to real users. (1) Do your engineers have direct exposure to users (support
rotations, customer calls, watching real usage), or do they build from assumptions and filtered specs? Could
you create more direct exposure? (2) When users complain or request features, are you translating them into
the real underlying problem, or implementing literally? Practice hearing past a request to the real need. (3)
Are you weighing anecdote vs data — using loud individual feedback for depth and aggregate data for breadth,
without over-indexing on either? (4) How do you handle enterprise escalations and saying no — do you cave,
refuse, or navigate thoughtfully? Reflect: how connected is your team to real user reality, and what's one
way to ground them more in the actual humans you build for?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the customer-connection skill — keeping the team grounded in the real humans it builds
for. A strong response assesses engineers' direct exposure to users, whether complaints are translated into
real needs (vs. implemented literally), whether anecdote and data are weighed well, and how escalations and
saying-no are handled. The realizations: (1) <strong>direct exposure beats assumptions</strong> — engineers
who encounter real users (support rotations, customer calls, real usage) develop genuine understanding of the
actual pain, leading to building the right thing, where insulated engineers build from often-wrong guesses;
(2) <strong>translate complaints into engineering signal</strong> — users express needs as complaints or
proposed solutions (in their terms), which reflect a real underlying problem but aren't necessarily the right
solution; hear past the literal request to the real need (interests behind positions); (3) <strong>weigh
anecdote vs data</strong> — use loud individual feedback for qualitative depth (the why) and aggregate data
for scale (how many), without over-indexing on either (one loud customer isn't the whole base); (4)
<strong>navigate escalations and say no thoughtfully</strong> — balance a big customer's real importance
against the broader good (don't let one customer hijack the roadmap or harm other users), seek win-wins, and
say no well (heard, honest, explained, with an alternative) when needed. On reflection, people commonly find
their engineers are fairly insulated from real users (building from specs and assumptions), that they
sometimes take requests literally rather than translating to the real need, and that escalations create
pressure to cave. The highest-value change is usually: create more direct user exposure for engineers
(support rotations, calls) and translate feedback to real needs rather than implementing literally. The
meta-point: engineering serves the humans on the other end, but teams lose touch, building from assumptions
rather than reality — and a lead's job is to keep the team connected (direct exposure), translate what users
say into what to build (real needs, not literal requests), weigh anecdote against data (depth and breadth),
and navigate the hard cases (escalations, saying no) with judgment. Grounding the team in real user reality
is how you build the right thing rather than what you assumed or what the loudest customer demanded. It's a
key stakeholder relationship (the ultimate stakeholder — the user) and connects to product thinking (the
next phase). The next lesson turns to another set of stakeholders — other engineering teams you depend on but
don't control — and getting what you need from teams with their own roadmaps and incentives.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 43 — Other Engineering Teams and Dependencies →](lesson-43-cross-team-dependencies){: .btn .btn-primary }
