---
title: "Lesson 45 — How Your Company Makes Money"
nav_order: 1
parent: "Phase 9: Business & Product Thinking"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 45: How Your Company Makes Money

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **business model** — how a company creates and captures value: revenue in, costs out.
> - **revenue model** — the specific way money comes in: **SaaS/subscriptions** (recurring fees), **transactions** (a cut of each sale), ads, licenses, usage-based.
> - **margin** — revenue minus costs, usually as a percentage; **cost structure** — where the money goes.
> - **cost center** — a part of the company that spends money but doesn't directly earn it.
> - **unit economics** — whether one average customer is profitable: **CAC** (customer acquisition cost), **LTV** (lifetime value — total revenue a customer brings), **payback period** (months until a customer covers their CAC).
> - **churn** — customers leaving (Lesson 42); the enemy of subscription businesses.
> - **take-rate** — the percentage cut a marketplace keeps of each transaction.
> - **growth stage vs profitability stage** — grabbing market share fast (funded by investors) vs optimizing margins; each implies different engineering priorities.
> - **earnings call** — a public company's quarterly presentation of results to investors.
> - **read like an investor** — understanding the whole company's economics, not just your slice.

## Concept

Most engineers never really understand how their company makes money — and it limits them. A lead who
grasps the **business model** (how revenue is made, where the money goes, what stage the company is at)
can make and justify engineering decisions in business terms, prioritize what actually matters, and be a
real partner to the business. The shift: **read your company like an investor, not an employee** —
understanding the economics, not just your slice of the work.

```
   READ YOUR COMPANY LIKE AN INVESTOR
   ┌──────────────────────────────────────────────────────┐
   │ • REVENUE MODEL: how does money come in? (subscriptions,│
   │   transactions, ads, enterprise licenses, usage...)    │
   │ • COST STRUCTURE & MARGINS: where does money go? where  │
   │   does engineering sit? (a cost center? the product?)  │
   │ • UNIT ECONOMICS: does each customer make money? (CAC,  │
   │   LTV, payback)                                         │
   │ • STAGE: growth (grab market) vs profitability (margins)│
   │   → implies very different engineering priorities       │
   └──────────────────────────────────────────────────────┘
```

The reframe: **understand the economics of your company — how it makes money, where engineering fits,
and what stage implies — so you can prioritize and justify engineering in business terms.** An engineer
who only knows their tickets can't connect their work to what matters; a lead who understands the
business model can (e.g., "we're a high-growth company grabbing market share, so speed-to-market matters
more than perfect efficiency right now"). Business literacy is what lets engineering serve the business
deliberately.

---

## How It Works

### The revenue model — how money comes in

Start with: **how does the company actually make money?** The revenue model — subscriptions (SaaS),
transactions (a cut of each), advertising, enterprise licenses, usage-based, marketplace fees, etc. This
shapes everything: a subscription business cares about retention (churn) and expansion; a transaction
business cares about volume and take-rate; an ads business cares about engagement and users. Knowing the
revenue model tells you what the business fundamentally optimizes for — which should inform what
engineering optimizes for (e.g., in a subscription business, reliability that prevents churn is directly
revenue-relevant).

### Cost structure and margins — where engineering sits

Understand **where the money goes** (the cost structure) and the **margins** (revenue minus costs). Key
question: **where does engineering sit in the cost structure?** Sometimes engineering builds the product
that <em>is</em> the revenue (a software product — engineering is the core value creation); sometimes
engineering is more of a cost center (internal tools, a non-tech company's IT). And infrastructure/
compute costs (which engineering controls) can be a major cost line (in some businesses, cloud spend is a
huge margin factor). Knowing the margins and where engineering fits tells you when engineering's costs
(compute, efficiency) directly matter to the business's profitability, and when engineering's output <em>is</em>
the business's value.

### Unit economics — does each customer make money?

**Unit economics** — the economics of a single customer: does acquiring and serving one customer make
money? Key concepts (lightly): **CAC** (customer acquisition cost — what it costs to get a customer),
**LTV** (lifetime value — the revenue a customer generates over their lifetime), and **payback period**
(how long until a customer becomes profitable). Healthy unit economics (LTV > CAC, reasonable payback) is
what makes a business sustainable. This matters to engineering because engineering affects unit economics
— the cost to serve a customer (infrastructure per customer), the retention (reliability, product
quality), and the efficiency all feed the unit economics. Understanding them shows how engineering's work
connects to whether the business is fundamentally healthy.

### Company stage — growth vs profitability

Crucially, the company's **stage** implies very different engineering priorities: (1) **Growth stage**
(grabbing market share, often pre-profitability, funded by investment) — the priority is speed,
capturing the market, shipping fast; efficiency and perfect quality often matter less than moving fast
(you optimize for growth, accepting some inefficiency/debt). (2) **Profitability stage** (mature, focused
on margins and sustainability) — the priority shifts to efficiency, cost control, reliability, and
margins; now engineering efficiency (compute costs, doing more with less) and reliability (retention)
matter more. The <em>same</em> engineering decision (invest in speed vs. efficiency vs. quality) has
different right answers depending on stage — so knowing your company's stage is essential to prioritizing
correctly. Reading the company's stage (from its funding, its public statements, its focus) tells you
what engineering should optimize for right now.

### Reading the numbers

You can often learn the business model from available sources: a **public company's** financial reports
and earnings calls (revenue, margins, what leadership emphasizes), a **private company's** board
narrative, all-hands messaging, or fundraising story (what they say about how they make money and their
focus), and general industry knowledge. Being curious about these — reading your company like an investor
would — builds the business literacy that most engineers lack and that makes a lead genuinely valuable in
business conversations.

{: .note }
> **Understand how your company makes money — read it like an investor, not an employee</br>**
> Most engineers never grasp their company's business model, which limits them. A lead who understands it
> — the <em>revenue model</em> (how money comes in — subscriptions, transactions, ads...), the <em>cost
> structure and margins</em> (where money goes, where engineering sits — core value or cost center), the
> <em>unit economics</em> (does each customer make money — CAC, LTV, payback), and the <em>stage</em>
> (growth vs profitability — which imply very different engineering priorities) — can prioritize what
> actually matters and justify engineering in business terms. The same engineering decision (speed vs
> efficiency vs quality) has different right answers by stage, so business literacy is what lets you
> prioritize correctly and be a real business partner. Read your company like an investor — the economics,
> not just your tickets — because understanding how the money works is what elevates a technical lead into
> a business-aware leader.

---

## Lab — Scenario

**The exercise:** This one is about <em>your actual company</em> (or, if you prefer, a well-known company
you understand). Apply the lesson to a real business.

**Write two short paragraphs:** (1) **How your company makes money** — its revenue model, roughly where
engineering sits in the cost structure, its stage (growth vs profitability), and what it fundamentally
optimizes for. (2) **What that means your team should optimize for** — given the business model and stage,
what engineering priorities does it imply (speed? efficiency? reliability? cost?).

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
Since this is about your real company, the "answer" is a strong <em>example</em> of the thinking, plus the
criteria for a good response. Example (for a hypothetical B2B SaaS company):
<br><br>
<strong>(1) How it makes money:</strong> "We're a B2B SaaS company — revenue comes from annual
subscriptions, mostly from mid-market and enterprise customers, with expansion revenue as customers add
seats and upgrade tiers. So the core drivers are <em>acquiring</em> customers, <em>retaining</em> them
(churn is the enemy of a subscription business), and <em>expanding</em> them (net revenue retention).
Engineering builds the product that <em>is</em> the revenue (we're a software product — engineering is
core value creation, not a cost center), though our cloud infrastructure costs are a meaningful margin
line as we scale. On stage: we're in growth mode — recently funded, focused on capturing market share in
a competitive space, prioritizing growth over profitability for now. So the business fundamentally
optimizes for growth (new customers), retention (keeping them — churn kills SaaS), and expansion."
<br><br>
<strong>(2) What the team should optimize for:</strong> "Given growth-stage B2B SaaS, my team should
optimize for: (a) <strong>speed to market</strong> — shipping the features that win and retain customers
fast, since we're grabbing market share (some tech debt is an acceptable trade for speed right now); (b)
<strong>reliability</strong> — because churn is the enemy in subscriptions, and outages/bugs drive churn,
reliability directly protects revenue (this is where quality genuinely pays off — it's retention); (c)
<strong>the features that drive acquisition and expansion</strong> — prioritizing work that helps land and
grow customers over internal perfection. What I'd de-prioritize <em>for now</em>: heavy infrastructure
cost-optimization (matters more at profitability stage — though I'd watch the cloud spend as it grows) and
gold-plating quality on things that don't affect retention. If we shift to profitability stage later, I'd
rebalance toward efficiency and margins."
<br><br>
<strong>What makes a good response:</strong> (1) <strong>Correctly identifies the revenue model</strong>
and what it optimizes for (subscription → retention/expansion; transaction → volume; ads → engagement —
each implies different priorities). (2) <strong>Places engineering in the cost structure</strong> — is
engineering the core product/value, or a cost center? Are infra costs a major margin factor? (3) <strong>Identifies
the stage</strong> (growth vs profitability) and its implications. (4) <strong>Derives specific engineering
priorities</strong> from the business model + stage — not generic "build good software," but "reliability
because churn kills subscriptions," "speed because we're grabbing market share," etc. (5) <strong>Connects
engineering choices to business drivers</strong> — showing that what to optimize (speed/efficiency/
reliability/cost) follows from how the company makes money and its stage. <strong>Common mistakes:</strong>
(1) not actually knowing how the company makes money (the whole point — go find out); (2) generic priorities
not derived from the business model; (3) missing the stage implication (optimizing for efficiency in a
growth company, or speed in a profitability-focused one); (4) not connecting engineering work to the real
business drivers (retention, acquisition, margins). The real value is doing this for <em>your</em> company —
if you can't confidently write these two paragraphs, that's the signal to go learn your business model
(read the earnings call, ask your manager, understand the board narrative).
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Business models & unit economics | *The Personal MBA*, Josh Kaufman; a16z on unit economics |
| Reading company financials | public 10-Ks / earnings calls; *Financial Intelligence*, Berman & Knight |
| SaaS metrics (if relevant) | David Skok's "SaaS Metrics 2.0"; churn, LTV/CAC |
| Connecting eng to business (next) | Lesson 48 (tech to business); Lesson 47 (metrics) |

---

## Checkpoint

**Q1.** Why should a lead understand how their company makes money, and what does "read your company like
an investor, not an employee" mean?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A lead should understand how their company makes money because <strong>it's what lets them prioritize what
actually matters, justify engineering decisions in business terms, and be a real partner to the business —
rather than being limited to their slice of the work with no connection to what drives the company</strong>.
Most engineers never grasp the business model, and it limits them: they can build things well, but they
can't connect their work to what the business needs, can't prioritize based on business impact, and can't
participate meaningfully in business-level conversations (where decisions about what to build and why are
made). A lead who understands the business model gains several capabilities: (1) <strong>prioritize what
matters</strong> — knowing how the company makes money (and its stage) tells you what to optimize for (e.g.,
reliability in a subscription business because churn kills revenue; speed in a growth-stage company grabbing
market share), so you prioritize engineering work by real business impact rather than by technical
preference or guesswork; (2) <strong>justify engineering in business terms</strong> — you can make the case
for engineering investments (an observability project, a reliability push, a performance improvement) in the
language of business impact (retention, revenue, cost, growth) that leadership responds to, rather than
purely technical terms they may not weigh; (3) <strong>be a real business partner</strong> — you can
participate in the strategic conversations (what to build, where to invest, how to prioritize) as someone
who understands the economics, not just the engineering — which is what elevates a technical lead into a
business-aware leader and makes your input valued at the level where direction is set. So understanding the
business model is what connects engineering to the business, enabling business-aligned prioritization,
business-language justification, and genuine business partnership — versus being an isolated executor who
builds well but can't connect to or influence what matters. What "read your company like an investor, not an
employee" means: <strong>understand the economics and the whole business (how it makes money, its costs and
margins, its unit economics, its stage and strategy) the way an investor evaluating the company would —
rather than seeing only your own slice of the work the way a typical employee does</strong>. An
<strong>employee</strong> view is narrow and internal: focused on your own tasks, your team, your part of
the org, without much attention to how the overall business works economically — you know your job but not
how the company makes money or where the money goes. An <strong>investor</strong> view is holistic and
economic: an investor deciding whether to put money in studies how the company makes money (revenue model),
whether it's profitable per customer (unit economics), where the money goes (cost structure and margins),
what stage and strategy it's pursuing (growth vs profitability), and what the future economics look like —
they understand the <em>business</em>, not just its operations. Reading your company like an investor means
adopting that economic, whole-business perspective: being curious about and understanding the revenue model,
margins, unit economics, and stage — the actual economics of the business — the way someone with money at
stake would study it. This is valuable because that economic understanding is exactly what enables the
business-aligned prioritization, justification, and partnership above — you can't optimize engineering for
the business, or justify engineering in business terms, or partner on strategy, without understanding the
business economically (the investor view). And the information is often available (public financials,
earnings calls, board narratives, all-hands messaging) to those curious enough to read it. So the reframe
is an invitation to develop business literacy — to understand your company's economics like an investor
would, rather than staying in the narrow employee view of just-my-work — because that economic understanding
is what makes a lead able to connect engineering to what actually drives the company, which is a defining
capability of a business-aware technical leader.
</details>

**Q2.** Why do a company's business model and stage (growth vs profitability) imply different engineering
priorities?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A company's <strong>business model</strong> and <strong>stage</strong> imply different engineering
priorities because <strong>they determine what the business fundamentally needs to optimize for — and
engineering should optimize for what serves those business needs, which differ by model and stage</strong>.
On the <strong>business model</strong>: how a company makes money determines what drives its success, which
determines what engineering work is most valuable. Examples: (1) a <strong>subscription (SaaS) business</strong>
makes money from recurring revenue, so <em>retention</em> (preventing churn) and <em>expansion</em> are
paramount — which means reliability and product quality (which drive retention) are directly revenue-relevant,
so engineering should prioritize reliability and the features that retain/expand customers; (2) a
<strong>transaction business</strong> (a cut of each transaction) cares about <em>volume</em> and
<em>conversion</em>, so engineering should optimize for the things that drive transaction volume and smooth
conversion (performance, reliability at scale, reducing friction); (3) an <strong>ads business</strong>
cares about <em>engagement</em> and <em>users</em>, so engineering optimizes for engagement, growth, and the
scale to serve many users cheaply. Each model optimizes for different things (retention vs volume vs
engagement), so the most valuable engineering work differs — reliability directly protects revenue in
subscriptions (churn), while in an ads business raw engagement/scale might matter more. So knowing the
revenue model tells you which engineering priorities connect to the business's actual value drivers. On the
<strong>stage</strong> (growth vs profitability), the priorities differ even more starkly for the <em>same</em>
company over time: (1) <strong>Growth stage</strong> (grabbing market share, often pre-profitability, funded
by investment) — the priority is <em>speed and capturing the market</em>: shipping fast, getting features
out, winning customers before competitors, growing. Efficiency, cost-optimization, and perfect quality often
matter <em>less</em> right now — the business is optimizing for growth, so it's willing to accept some
inefficiency and technical debt in exchange for speed (moving fast to capture the market is worth more than
being efficient). So engineering should prioritize speed-to-market and the features that drive growth. (2)
<strong>Profitability stage</strong> (mature, focused on margins and sustainability) — the priority shifts
to <em>efficiency, cost control, and margins</em>: now doing more with less, reducing infrastructure/compute
costs (which affect margins), improving reliability (retention), and operating efficiently matter <em>more</em>.
The business is optimizing for profitability, so engineering efficiency and cost discipline that were
de-prioritized in growth stage now become important. So the <em>same engineering decision</em> — invest in
speed vs. efficiency vs. quality — has <em>different right answers by stage</em>: optimizing for compute-cost
efficiency is right at profitability stage but possibly wrong at growth stage (where speed matters more than
saving on infra); accepting some tech debt for speed is right at growth stage but increasingly wrong as the
company matures. This is why knowing the stage is essential to prioritizing correctly — a lead who optimizes
for efficiency in a growth company (slowing down to save costs when the business needs speed to capture the
market) or for speed in a profitability-focused company (racking up costs and debt when the business needs
margins) is optimizing for the wrong thing, misaligned with what the business actually needs. The underlying
principle: <strong>engineering should serve the business's actual needs, and those needs are determined by
the business model (what drives its revenue) and stage (growth vs profitability) — so the right engineering
priorities follow from understanding the model and stage</strong>. The same technical trade-off (speed/
efficiency/quality/cost) is resolved differently depending on what the business needs to optimize for, which
depends on how it makes money and where it is in its lifecycle. This is exactly why business literacy matters
for prioritization: without understanding the model and stage, a lead can't know what the business needs and
so can't prioritize engineering to serve it — they'd optimize by technical preference or generic "good
practices" rather than by what actually matters to <em>this</em> business <em>right now</em>. Understanding
the model and stage lets a lead align engineering priorities with real business needs — which is the
essence of business-aware technical leadership.
</details>

---

## Homework

Build your business literacy about your own company. (1) Write the two paragraphs from the lab for your
<em>actual</em> company — how it makes money (revenue model, cost structure, where engineering sits, stage)
and what that implies your team should optimize for. If you can't confidently write them, that's the
signal — go learn (read the earnings call or board narrative, ask your manager, understand the business).
(2) Identify your company's stage (growth vs profitability) and check: are your team's priorities aligned
with what that stage implies? (3) Notice where engineering's work connects to the business drivers
(retention, acquisition, margins, unit economics). Reflect: how well do you understand your company's
economics, and does your team's engineering prioritization actually align with how the company makes money
and its stage?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds business literacy about one's own company — the foundation for business-aware
engineering leadership. A strong response actually writes the two paragraphs for the real company (revenue
model, cost structure, engineering's place, stage → engineering priorities), identifies the stage and
checks priority alignment, and connects engineering work to business drivers. The realizations: (1)
<strong>understand how the company makes money</strong> — the revenue model (subscription/transaction/ads/
etc.) determines what the business optimizes for (retention/volume/engagement), which should inform
engineering priorities; (2) <strong>know where engineering sits and the margins</strong> — whether
engineering is the core product/value or a cost center, and whether infra costs are a major margin factor,
determines when engineering's costs and output directly matter to the business; (3) <strong>the stage
implies priorities</strong> — growth stage → speed/market-capture (accept some inefficiency/debt),
profitability stage → efficiency/cost/margins — so the same engineering decision has different right answers
by stage, and misaligning (efficiency in a growth company, speed in a profitability one) optimizes for the
wrong thing; (4) <strong>read the company like an investor</strong> — the economic whole-business view (from
earnings calls, board narratives) rather than the narrow employee view. On reflection, many people discover
they <em>can't</em> confidently write how their company makes money — which is exactly the signal to go
learn it (read the financials, ask, understand the business), because that gap is limiting them. The
highest-value realization is usually either learning the business model (if it was a gap) or noticing a
misalignment between the team's engineering priorities and what the company's model/stage actually needs. The
meta-point: most engineers never understand their company's business model, which limits them — a lead who
does can prioritize what actually matters, justify engineering in business terms, and be a real business
partner. The same engineering decision (speed/efficiency/quality/cost) follows different right answers
depending on how the company makes money and its stage, so business literacy is essential to prioritizing
correctly and aligning engineering with the business. Reading your company like an investor (its economics,
not just your tickets) is what elevates a technical lead into a business-aware leader. This is the foundation
of the phase; the next lessons build on it — product strategy (Lesson 46), metrics (47), connecting tech to
business (48), and the "should we build this?" question (49) — all requiring the business understanding this
lesson develops. The next lesson turns to the product side — understanding the product strategy your PM
does, well enough to be a real partner in deciding what to build.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 46 — Customer Needs and Product Strategy →](lesson-46-product-strategy){: .btn .btn-primary }
