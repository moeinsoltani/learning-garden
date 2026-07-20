---
title: "Lesson 47 — Metrics and KPIs"
nav_order: 3
parent: "Phase 9: Business & Product Thinking"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 47: Metrics and KPIs

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **KPI** — Key Performance Indicator: a metric an organization steers by.
> - **north-star metric** — the single outcome metric that captures the value you create; **input metrics** — the controllable drivers that feed it.
> - **DORA metrics** — the four research-backed delivery metrics: deployment frequency, lead time for changes, change failure rate, time to restore.
> - **cycle time** — how long a piece of work takes start to finish (Lesson 11).
> - **velocity / story points** — agile's estimate-based output measures; notoriously gameable.
> - **gaming (a metric)** — improving the number without improving the reality it's meant to measure.
> - **vanity metric** — a number that looks impressive but informs no decision (total registered users); **actionable metric** — one that reflects real value and guides action.
> - **Goodhart's Law** (GOOD-hart) — "when a measure becomes a target, it ceases to be a good measure."
> - **proxy** — a stand-in measured *instead of* the real thing; all metrics are proxies.
> - **instrument** (verb) — to add measurement to a system so you can actually see what's happening.

## Concept

Leaders think and communicate in **metrics** — and a lead needs to speak them natively: both **business
metrics** (revenue, retention, engagement) and **engineering metrics** (cycle time, incidents, DORA),
honestly connected. But metrics are dangerous: they get **gamed**, they mislead (vanity metrics), and
they're often mistaken for the thing they measure. The skill is using metrics well — instrumenting what
you actually care about, knowing their limits, and balancing metrics with judgment — rather than being
ruled or fooled by them.

```
   USING METRICS WELL
   ┌──────────────────────────────────────────────────────┐
   │ • NORTH-STAR vs INPUT metrics (the outcome vs the       │
   │   drivers you can act on)                              │
   │ • ENGINEERING metrics leaders watch (DORA, cycle time,  │
   │   incidents) — and how they're GAMED                   │
   │ • VANITY metrics (look good, mean nothing)              │
   │ • measure what you CLAIM to care about                  │
   │ • METRICS + JUDGMENT (Goodhart: a measure that becomes  │
   │   a target stops being a good measure)                 │
   └──────────────────────────────────────────────────────┘
```

The reframe: **metrics are tools for insight and alignment, not the goal itself — use them to inform
judgment, and beware that any metric made a target gets gamed (Goodhart's Law).** Metrics are essential
(you can't manage what you don't measure, and leaders communicate in them), but they're imperfect proxies:
they can be gamed, they can be vanity (impressive but meaningless), and optimizing a metric can diverge
from the real goal it was meant to represent. So speak metrics natively, but use them wisely — as inputs
to judgment, not replacements for it.

---

## Going Deeper

### North-star vs input metrics

Distinguish the **north-star metric** (the single key outcome that captures the value you're creating —
e.g., weekly active users, revenue retention) from **input metrics** (the drivers you can directly act on
that feed the north star — e.g., signup rate, activation rate, feature adoption). The north star is the
outcome you care about but can't directly control; input metrics are the levers you <em>can</em> move that
influence it. Good metric-thinking connects them: identify the north star, then the input metrics that
drive it, and work the inputs. This gives a clear line from your team's work (moving an input metric) to
the outcome that matters (the north star).

### Engineering metrics leaders watch — and how they're gamed

Leaders watch certain **engineering metrics**: **DORA metrics** (deployment frequency, lead time for
changes, change failure rate, time to restore — the research-backed delivery metrics), **cycle time** (how
long work takes), **incident counts/severity**, etc. Know these (they're the language leaders use to assess
engineering health). But critically, know **how they're gamed**: (1) deployment frequency can be inflated
by trivial deploys; (2) cycle time can be gamed by splitting tickets artificially; (3) incident counts can
be reduced by not reporting incidents (worse!); (4) "velocity" (story points) is notoriously gameable
(inflate estimates). Every metric can be gamed, and optimizing the metric rather than the underlying reality
is a real failure mode. So use these metrics as signals to investigate, not targets to hit — and watch for
gaming (including your own team's, under pressure).

### Vanity metrics vs actionable metrics

**Vanity metrics** look impressive but don't inform decisions or reflect real value — total registered
users (when most are inactive), total page views, cumulative downloads. They go up and to the right and
feel good, but they don't tell you if you're succeeding or what to do. **Actionable metrics** reflect real
value and inform decisions — active users, retention, conversion, revenue per user. The test: does this
metric reflect real value, and does it help me decide something? If it just looks good but doesn't inform
action or reflect real success, it's vanity. Beware presenting (or being impressed by) vanity metrics.

### Instrument what you claim to care about

A discipline: **measure what you actually claim to care about.** If you say quality matters, do you measure
it (defect rates, escaped bugs)? If you say the team's health matters, do you measure it (or just claim
it)? Often people claim to care about things they don't measure (so it's not really managed) and optimize
things they do measure (so they get optimized whether or not they matter). Aligning what you measure with
what you claim to value ensures the important things are actually tracked and managed — and prevents the
trap of optimizing the measurable at the expense of the important-but-unmeasured.

### Metrics + judgment — Goodhart's Law

The deepest point: **metrics inform judgment; they don't replace it — and any metric that becomes a target
gets gamed** (Goodhart's Law: "when a measure becomes a target, it ceases to be a good measure"). When you
turn a metric into <em>the</em> goal (hit this number), people optimize the metric — often in ways that
diverge from or undermine the real thing it was meant to represent (gaming, or genuine but misguided
optimization). So metrics should <em>inform</em> your judgment about reality (a signal to investigate,
understand, and act on with judgment), not <em>become</em> the reality you optimize blindly. Balance the
metric with judgment: use it as evidence, but understand what's really going on (the metric is a proxy, not
the truth), and don't let hitting a number replace achieving the actual goal. This is the crucial wisdom
about metrics — they're powerful tools that become dangerous when treated as the goal rather than a proxy
for it.

{: .note }
> **Speak metrics natively, but use them as inputs to judgment — and beware gaming (Goodhart)</br>**
> Leaders think and communicate in metrics, so a lead needs to speak both business metrics (revenue,
> retention, engagement) and engineering metrics (DORA, cycle time, incidents) fluently and honestly
> connected. But metrics are imperfect proxies: distinguish the <em>north star</em> (the outcome) from
> <em>input metrics</em> (the levers you act on); know the engineering metrics leaders watch <em>and how
> they're gamed</em>; avoid <em>vanity metrics</em> (impressive but meaningless) in favor of actionable
> ones; <em>measure what you claim to care about</em> (or it's not managed); and above all, use metrics to
> <em>inform judgment</em>, not replace it — because any metric made a target gets gamed (Goodhart's Law).
> Metrics are powerful tools for insight and alignment, dangerous when treated as the goal. Use them wisely:
> as evidence to understand reality and inform action, not as numbers to hit blindly.

---

## Lab — Scenario

**The situation:** Your director comes to you and says: **"I want a dashboard proving the team is
productive."** They're under pressure from <em>their</em> leadership to show engineering is delivering, and
they want metrics to demonstrate productivity. But you know "proving productivity with a dashboard" is
fraught — engineering productivity is notoriously hard to measure, the obvious metrics (lines of code,
story points, commits) are gameable and misleading, and a "productivity dashboard" can drive bad behavior.
You want to help your director (they have a real need) while pushing back on the problematic framing.

**Write how you'd respond** — what you'd build and what you'd push back on. Then note the principles and
mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong response helps the director's real need (showing engineering is delivering value) while pushing
back on the naive/gameable "productivity dashboard" framing. Example:
<br><br>
<strong>Understand and validate the real need:</strong> "First, I get it — you need to show your leadership
that engineering is delivering, and that's completely reasonable. Let me help with that. But I want to be
honest about a trap and propose something better than a naive 'productivity dashboard,' because I don't want
to give you numbers that could backfire." <em>(Validates the real need — showing delivery — while flagging
the problem.)</em>
<br><br>
<strong>Push back on the problematic framing:</strong> "Here's my concern with 'proving productivity with a
dashboard': engineering productivity is genuinely hard to measure, and the obvious metrics are misleading
and gameable. Lines of code rewards bloat; story points/velocity get inflated (Goodhart's Law — once it's a
target, it's gamed); commits/PRs reward splitting work artificially. Worse, if we make individual
'productivity metrics' the target, we'll drive bad behavior (gaming, individualism, gaming incident counts
by not reporting) and hurt the actual work. So I don't want to build a dashboard that measures the wrong
things and creates bad incentives." <em>(Honest pushback on the naive framing, with the specific reasons —
gameability, Goodhart, bad incentives.)</em>
<br><br>
<strong>Propose what to build instead — outcomes and healthy delivery metrics:</strong> "Here's what I'd
build instead, which actually shows we're delivering value: <br>
• <strong>Outcomes/impact</strong>: what the team has <em>delivered</em> and its business impact — features
shipped and their results (adoption, the metrics they moved), problems solved, value created. This shows
real productivity (delivering value), not activity. <br>
• <strong>Healthy delivery metrics (DORA)</strong>: deployment frequency, lead time, change failure rate,
time to restore — the research-backed metrics of a healthy, high-performing delivery process (used as
health signals, at the <em>team</em> level, not individual targets). <br>
• <strong>Reliability/quality</strong>: incident trends, uptime — showing we deliver reliably. <br>
These show the team is delivering valuable work reliably and efficiently — which is what 'productive' really
means — without the gameable individual-activity metrics." <em>(Proposes outcome/impact + healthy team-level
delivery metrics — showing real value delivery, not gameable activity.)</em>
<br><br>
<strong>Frame it for the director's need:</strong> "This actually gives you a <em>stronger</em> story for
your leadership — 'here's the value engineering delivered and how reliably/efficiently we deliver' is more
compelling than 'here's how many story points,' and it won't backfire. And I'd pair the metrics with the
narrative — the metrics support the story of what we delivered, not stand alone." <em>(Shows this serves the
director better, with metrics + narrative.)</em>
<br><br>
<strong>Principles:</strong> (1) <strong>Validate the real need</strong> — the director genuinely needs to
show delivery; help with that. (2) <strong>Push back on the gameable framing</strong> — naive productivity
metrics (LOC, velocity, commits) are misleading, gameable (Goodhart), and drive bad behavior. (3)
<strong>Measure outcomes/value, not activity</strong> — what was delivered and its impact shows real
productivity, not gameable output counts. (4) <strong>Team-level healthy delivery metrics (DORA)</strong> —
as health signals, not individual targets. (5) <strong>Metrics + narrative</strong> — the numbers support
the story of delivered value. (6) <strong>Serve the director better</strong> — this gives a stronger,
backfire-proof story. <strong>Mistakes to avoid:</strong> (1) <strong>just building the naive dashboard</strong>
(LOC, velocity, individual metrics) — gameable, misleading, drives bad behavior; (2) <strong>flatly refusing</strong>
("productivity can't be measured, no") — unhelpful; the director has a real need; (3) <strong>individual
productivity metrics</strong> — especially harmful (gaming, individualism, surveillance feel); keep it
team-level and outcome-focused; (4) <strong>measuring activity, not outcomes</strong> — output counts
(commits, points) reward activity, not value; measure delivered value; (5) <strong>metrics without
narrative</strong> — numbers alone (especially eng metrics) mislead; pair with the story; (6) <strong>not
explaining the why to the director</strong> — pushing back without helping them understand the trap and
offering a better option (be a partner, not a blocker).
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| DORA metrics | *Accelerate*, Forsgren, Humble, Kim; DORA reports |
| Goodhart's Law | [Goodhart's law (Wikipedia)](https://en.wikipedia.org/wiki/Goodhart%27s_law) |
| Vanity vs actionable metrics | *The Lean Startup*, Ries; *Lean Analytics* |
| Measuring eng productivity (carefully) | McKinsey debate + Kent Beck/Gergely Orosz responses; SPACE framework |

---

## Checkpoint

**Q1.** What is Goodhart's Law and why does it matter for metrics, and what's the difference between vanity
and actionable metrics?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Goodhart's Law</strong> states: <strong>"when a measure becomes a target, it ceases to be a good
measure."</strong> The idea: a metric works as a useful proxy for something you care about <em>as long as
people aren't optimizing for the metric itself</em> — but once you make the metric a <em>target</em> (hit
this number), people start optimizing the metric directly, often in ways that diverge from or undermine the
real thing it was meant to represent, so it stops being a good measure of that real thing. Why it matters
for metrics: it reveals a fundamental danger in using metrics as goals. When you turn a metric into <em>the</em>
objective, people (rationally, under pressure) find ways to move the number — and because the metric is only
a <em>proxy</em> for the real goal (not the goal itself), optimizing the proxy can (a) <strong>game it</strong>
— move the number without improving (or while harming) the real thing (e.g., reducing "incident count" by
not reporting incidents — the number improves, reality worsens; inflating "velocity" by padding estimates —
points go up, actual output doesn't; boosting "deployment frequency" with trivial deploys); or (b) <strong>optimize
the proxy at the expense of the goal</strong> — genuinely improve the metric in ways that don't serve (or
undermine) the actual objective (optimizing lines of code produces bloat; optimizing narrow metrics can hurt
the broader goal). So Goodhart's Law matters because it warns that <strong>metrics made into targets get
gamed or misoptimized, diverging from the real goal — so you can't just set a metric target and optimize it
blindly</strong>. The practical implication: use metrics to <em>inform judgment</em> about reality (as
signals to investigate and understand), not as targets to hit — because the moment a metric becomes the
target, it degrades as a measure of the underlying reality. This is why "metrics + judgment" is essential:
the metric is a proxy, not the truth, so you use it as evidence while understanding what's really going on,
rather than letting the number become the goal. The difference between <strong>vanity and actionable
metrics</strong>: <strong>vanity metrics look impressive but don't reflect real value or inform decisions,
while actionable metrics reflect real value and help you decide something</strong>. <strong>Vanity
metrics</strong> — total registered users (when most are inactive), total page views, cumulative downloads,
total commits — are metrics that (1) go "up and to the right" and <em>feel</em> good/impressive, but (2)
don't reflect real success (total registered users doesn't tell you if the product is working — most might
be inactive; cumulative downloads keeps rising even if the product is dying) and (3) don't inform any
decision (knowing total page views doesn't tell you what to do). They're impressive-looking but hollow —
you can show them off, but they don't tell you if you're succeeding or what to change. <strong>Actionable
metrics</strong> — active users, retention, conversion rate, revenue per user — (1) reflect <em>real
value/success</em> (active users and retention actually indicate whether the product is working and valued)
and (2) <em>inform decisions</em> (a drop in retention tells you something's wrong and to investigate; a
conversion rate tells you where to improve). The test to distinguish them: <strong>does this metric reflect
real value, and does it help me decide something?</strong> If a metric just looks good but doesn't reflect
real success or inform action (like cumulative totals that only ever rise), it's vanity; if it reflects real
value and drives decisions, it's actionable. Why the distinction matters: vanity metrics are seductive
(they feel good and impress) but dangerous — presenting or being impressed by them creates a false sense of
success and doesn't guide action, so teams can feel good about vanity metrics while the real (actionable)
metrics are flat or declining. Beware presenting vanity metrics (to leadership, or yourself) or being
fooled by them; focus on actionable metrics that reflect real value and inform decisions. Both concepts —
Goodhart's Law and vanity-vs-actionable — reflect the core wisdom that metrics are imperfect proxies to be
used with care: Goodhart warns that made-into-targets they get gamed (so inform judgment, don't optimize
blindly), and vanity-vs-actionable warns that some metrics look good without reflecting real value (so
choose metrics that reflect value and inform decisions). Together they guide using metrics wisely — as
honest signals about reality that inform judgment — rather than being ruled or fooled by them.
</details>

**Q2.** Why should you "measure what you claim to care about" and use metrics as inputs to judgment rather
than replacements for it?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You should <strong>measure what you claim to care about</strong> because <strong>what gets measured gets
managed and optimized, while what's claimed-but-unmeasured tends to be neither — so a mismatch between what
you say you value and what you measure means the important things aren't actually tracked, and the measured
things get optimized whether or not they matter</strong>. There's a common, damaging gap: people <em>claim</em>
to care about things (quality, team health, long-term maintainability, customer satisfaction) but don't
<em>measure</em> them, while they <em>do</em> measure other things (velocity, output, whatever's easy to
count). The consequences: (1) <strong>the claimed-but-unmeasured things aren't really managed</strong> — if
you say quality matters but never measure it (defect rates, escaped bugs), then quality isn't actually
tracked, so it isn't managed, so it drifts (you can't manage what you don't measure, and a stated value that
isn't measured is just words); (2) <strong>the measured things get optimized regardless of importance</strong>
— what you measure and watch gets attention and optimization (people optimize toward the numbers being
tracked), so if you measure velocity but not quality, velocity gets optimized (possibly at quality's expense
— the classic "measured speed, unmeasured quality suffers"). So the mismatch means you optimize the
measurable (whether or not it's what matters) while neglecting the important-but-unmeasured (which you
claimed to value). Aligning what you measure with what you claim to value fixes this: it ensures the things
you genuinely care about are actually tracked and managed (so they don't drift), and it prevents the trap of
optimizing the merely-measurable at the expense of the important-but-unmeasured. The discipline is: if you
say something matters, measure it (so it's real, not just a claim) — and be honest about whether your
measurements reflect your actual priorities. Why use metrics as <strong>inputs to judgment rather than
replacements for it</strong>: <strong>because metrics are imperfect proxies for reality — they capture part
of the picture but not all of it, can be gamed (Goodhart), and can mislead — so treating a metric as the
truth (optimizing it blindly) diverges from the real goal, whereas using it as evidence to inform your
judgment about reality lets you understand what's actually going on</strong>. Metrics are valuable (they
provide signal, enable communication and alignment, and reveal things), but they're <em>proxies</em>, not
the reality itself — any single metric is a partial, imperfect representation of a complex reality. If you
<strong>replace judgment with the metric</strong> (make the number the goal, optimize it blindly), several
failures occur: (1) <strong>Goodhart's Law</strong> — the metric-as-target gets gamed or misoptimized,
diverging from the real goal; (2) <strong>you miss what the metric doesn't capture</strong> — the metric
measures one aspect, but reality has many (a "productivity" metric misses quality, morale, long-term health,
etc.), so optimizing it blindly ignores the unmeasured important things; (3) <strong>you lose sight of the
real goal</strong> — the metric was meant to represent something you care about (the real goal), and
optimizing the proxy isn't the same as achieving the goal (hitting the number ≠ succeeding). If instead you
<strong>use the metric as an input to judgment</strong> — as evidence/signal that informs your understanding
of reality, which you interpret with judgment (what's really going on? does this metric reflect the real
situation? what does it miss?) — then you get the metric's value (the signal) without its danger (blind
optimization): the metric points you to investigate, you understand the underlying reality (using judgment
and context the metric lacks), and you act on that understanding of reality, not on the number alone. So a
drop in a metric is a signal to investigate and understand (judgment), not automatically a fire to fix by
moving the number (which might game it). This "metrics inform judgment, don't replace it" wisdom is what lets
you use metrics powerfully (as evidence and signal) while avoiding being ruled or fooled by them (Goodhart,
vanity, missing what they don't capture). Both points reflect using metrics wisely: measure what you actually
value (so the important things are tracked and managed, and you don't just optimize the convenient), and use
metrics as inputs to judgment about reality (so you get their signal without letting the imperfect proxy
become the blindly-optimized goal). Metrics are powerful tools for insight and alignment, but they serve
judgment about the real goal — they don't replace it — and aligning measurement with real priorities and
holding the metric-judgment balance is how a lead uses them well rather than being managed by the numbers.
</details>

---

## Homework

Sharpen how you use metrics. (1) For something your team is measured on (or that you track), ask: is it a
vanity metric (looks good, doesn't inform/reflect value) or actionable? Is it being gamed (or would it be if
targeted)? (2) Do you measure what you claim to care about? Find something you say matters (quality, team
health) but don't measure — and consider how you'd track it. (3) For any metric that's become a target, watch
for Goodhart effects (gaming, optimizing the number over the goal). (4) Practice connecting an engineering
metric to a business outcome honestly (e.g., cycle time → time-to-market → competitiveness). Reflect: are you
using metrics as signals to inform judgment, or being ruled/fooled by them — and what's one metric you should
add, drop, or use more carefully?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds metric-literacy — speaking metrics natively while using them wisely. A strong response
examines whether a tracked metric is vanity or actionable (and gameable), checks whether they measure what
they claim to value, watches for Goodhart effects on any metric-target, and practices connecting an
engineering metric to a business outcome. The realizations: (1) <strong>speak both business and engineering
metrics</strong> — leaders think and communicate in metrics (revenue/retention/engagement and DORA/cycle
time/incidents), so a lead needs fluency in both, honestly connected; (2) <strong>vanity vs actionable</strong>
— vanity metrics look impressive but don't reflect value or inform decisions (cumulative totals), while
actionable ones reflect real value and drive decisions (active users, retention) — favor the latter and
beware being fooled by the former; (3) <strong>Goodhart's Law</strong> — any metric made a target gets gamed
(reducing incident counts by not reporting, inflating velocity), so use metrics as signals to investigate,
not targets to hit blindly, and watch for gaming (including your team's under pressure); (4) <strong>measure
what you claim to care about</strong> — what's measured gets managed/optimized and what's claimed-but-
unmeasured doesn't, so align measurement with real priorities (or you optimize the measurable and neglect the
important); (5) <strong>metrics inform judgment, don't replace it</strong> — metrics are imperfect proxies,
so use them as evidence to understand reality, not as the goal to optimize blindly. On reflection, people
commonly find they track some vanity or gameable metrics, that they claim to value things they don't measure
(quality, team health), and that they sometimes treat metrics as targets rather than signals. The
highest-value change is usually: measure what actually matters (align measurement with real priorities), and
use metrics as signals informing judgment rather than targets to hit. The meta-point: leaders think and
communicate in metrics, so a lead must speak them natively (business and engineering, honestly connected) —
but metrics are dangerous (gamed via Goodhart, misleading via vanity, mistaken for the reality they proxy),
so the skill is using them well: distinguish north-star from input metrics, know the engineering metrics
leaders watch and how they're gamed, favor actionable over vanity, measure what you claim to value, and above
all use metrics as inputs to judgment rather than replacements for it. Metrics are powerful tools for insight
and alignment, dangerous as goals — use them wisely rather than being ruled or fooled by them. This connects
business and engineering (the phase's theme) and feeds directly into the next lesson — connecting technical
decisions to business outcomes, which uses metrics to justify engineering investments in business terms.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 48 — Connecting Technical Decisions to Business Outcomes →](lesson-48-tech-to-business){: .btn .btn-primary }
