---
title: "Lesson 06 — Technical Vision and Strategy"
nav_order: 1
parent: "Phase 2: Technical Leadership"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 06: Technical Vision and Strategy

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **vision / strategy / roadmap** — destination / route and choices / turn-by-turn steps (the lesson defines them fully).
> - **kernel** (Rumelt's) — the three-part core of real strategy: diagnosis, guiding policy, coherent actions.
> - **diagnosis** — naming what is *actually* wrong before choosing a cure.
> - **coherent / cohere** — parts that fit together and reinforce each other.
> - **monolith** — one big codebase deployed as a single unit; **microservices** — many small independently deployed services.
> - **strangler fig** — replacing an old system piece by piece while it keeps running (named after the tree that slowly envelops its host); opposite of a **big-bang rewrite**.
> - **seam** — a natural boundary in code where you can safely split it.
> - **churn** — how often a piece of code changes; high-churn areas matter most.
> - **fluff** — impressive-sounding words with no content.
> - **wish-list** — a list of desires with no choices or trade-offs behind it.
> - **adjudicate** — to settle a dispute by acting as the judge.

## Concept

Without a technical direction, a team makes locally-reasonable decisions that add
up to an incoherent mess — each engineer optimizes their piece, and the whole
drifts. A lead's job is to provide the direction that makes the individual
decisions cohere. But "vision," "strategy," and "roadmap" get used
interchangeably and mean different things:

```
   VISION     where we're going and why — the destination
              "a platform where any team can ship a service in a day"
                        │  (aspirational, stable, ~yearly)
   STRATEGY   HOW we'll get there — the approach, and what we WON'T do
              "consolidate on one deployment path; stop supporting the
               three legacy ones; invest in self-service tooling first"
                        │  (a set of choices, ~quarterly)
   ROADMAP    WHAT and WHEN — the concrete sequence
              "Q1: unify CI. Q2: self-service envs. Q3: deprecate legacy."
                        │  (specific, changes often)
```

Vision is the destination, strategy is the route and the deliberate choices
(including what to sacrifice), roadmap is the turn-by-turn directions. New leads
often produce a roadmap (a list of things to build) and call it strategy —
missing that strategy is fundamentally about **choices and trade-offs**, not a
to-do list.

Richard Rumelt (*Good Strategy/Bad Strategy*) gives the sharpest framing: real
strategy has a **kernel** of three parts — a *diagnosis* (what's actually the
problem?), a *guiding policy* (the overall approach to the problem), and
*coherent actions* (steps that reinforce each other). "Bad strategy" is
fluff, goals-mistaken-for-strategy ("we will be the best!"), or a wish-list with
no diagnosis. The discipline is to diagnose before you prescribe, and to make
strategy about the hard choices — especially **what you won't do**.

---

## How It Works

### Diagnose before you prescribe

The most common strategy failure is jumping to solutions before understanding the
problem. Rumelt's kernel starts with *diagnosis*: what is actually going wrong,
at the root? A team drowning in a legacy monolith might diagnose "we can't ship
independently because everything is coupled and every deploy risks everything" —
that diagnosis then <em>implies</em> the guiding policy (decouple along the lines
that let teams ship independently) and coherent actions (extract the highest-
churn, highest-risk seams first; invest in the testing that makes extraction
safe). Skip the diagnosis and you get a wish-list ("migrate to microservices!")
disconnected from the real problem, which is why so many rewrites fail.

### Strategy is what you WON'T do

A strategy that says yes to everything is not a strategy — it's a list. The
essence is choosing: this approach *over* that one, this investment *instead of*
that one, these customers *and not* those. The hard, valuable part is the
sacrifice — "we will consolidate on one deployment path, which means we stop
supporting the two others, which means some teams will be inconvenienced, and
we're accepting that because independent shipping matters more." A lead who
won't make those cuts produces mush that gives the team no real guidance
(everything is a priority, so nothing is).

### One page, and keep it alive

Write the vision/strategy short — one page — because a strategy nobody can hold
in their head doesn't guide decisions. And it's not a document you write once and
file; it's a *living* reference you keep invoking: when a design decision comes
up, you point back to the strategy ("our guiding policy is X, so we should do
A not B"); when priorities are contested, the strategy adjudicates. A strategy
that isn't referenced in real decisions is dead — the test of a good one is
whether it actually changes what the team does.

{: .note }
> **Goals are not strategy**
> "Increase reliability to 99.99%," "ship faster," "be the best platform" —
> these are <em>goals</em> (or worse, aspirations), not strategy. Strategy is
> the <em>diagnosis of why you're not there and the coherent approach to
> getting there</em>. "We will improve reliability" tells the team nothing
> about how; "our incidents cluster in the deploy path, so we're investing in
> progressive rollouts and automated rollback before adding features" is
> strategy — a diagnosis and a coherent set of choices. When your "strategy"
> is a restatement of the goal, you haven't done the work yet.

---

## Lab — Scenario

**The situation:** You've just become tech lead of a team responsible for a
seven-year-old monolith. The symptoms: deploys are scary (any change can break
anything, so the team deploys once a week in a tense two-hour window); onboarding
takes months (the codebase is a maze); three teams all work in the monolith and
constantly step on each other; and leadership is pushing for faster feature
delivery, which the current state can't support. The team is demoralized and
keeps talking vaguely about "we should move to microservices."

**Write a one-page technical vision and strategy for this team.** Use Rumelt's
kernel: a diagnosis, a guiding policy, and coherent actions — and be explicit
about what you *won't* do.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong response resists the team's instinct ("move to microservices!" — a
solution with no diagnosis, the classic bad-strategy trap) and instead does the
diagnostic work first. An example one-pager:
<br><br>
<strong>Diagnosis (the real problem):</strong> The pain isn't "we have a
monolith" — it's that the monolith is <em>coupled such that no team can ship
independently or safely</em>. Every change risks everything (no isolation), three
teams contend for one deploy (no independence), and the weak test coverage makes
every change scary (no safety). "Faster features" is impossible because the
bottleneck is <em>fear and contention in the deploy path</em>, not coding speed.
The root causes: tight coupling across team boundaries, and insufficient
automated testing to make change safe.
<br><br>
<strong>Guiding policy (the approach):</strong> Decouple along <em>team
boundaries</em> so each team can build, test, and deploy its own area
independently and safely — prioritizing the seams that cause the most contention
and risk, and building the testing/observability that makes decoupling safe
<em>first</em>. We will do this <em>incrementally</em> (strangler-fig — extract
piece by piece behind stable interfaces), not as a big-bang rewrite. Crucially:
we are decoupling for <em>independent, safe shipping</em> — not for
"microservices" as an end in itself. Some things stay in the monolith
indefinitely if they're not causing pain.
<br><br>
<strong>Coherent actions (reinforcing steps):</strong> (1) Invest first in the
deploy safety net — automated testing on the highest-risk paths, progressive
rollout, fast rollback — so that <em>any</em> change becomes less scary
immediately (a quick win that helps before any extraction). (2) Identify the 2–3
seams where teams contend most and risk is highest; extract those first behind
clean interfaces, giving those teams independent deploy. (3) Establish clear
ownership boundaries in the codebase so teams stop stepping on each other even
before full extraction. (4) Measure and publish deploy frequency and lead time
so progress is visible and tied to the leadership goal.
<br><br>
<strong>What we will NOT do (the essential sacrifices):</strong> We will
<em>not</em> do a big-bang rewrite to microservices (high risk, long time-to-
value, the graveyard of teams like ours). We will <em>not</em> extract everything
— only the pieces causing real pain; low-churn stable code stays in the monolith.
We will <em>not</em> add major new features to the worst-coupled areas until
they're stabilized (we're accepting slightly slower feature delivery <em>this
quarter</em> to enable much faster delivery after — and we'll make that trade-off
explicit to leadership rather than pretending we can do both at once).
<br><br>
<strong>Vision (the destination, one sentence):</strong> "Each team can safely
ship its own part of the product multiple times a day, without fear and without
waiting on other teams."
<br><br>
Why this is good strategy and the naive version isn't: it <em>diagnoses</em> (the
problem is coupling+safety, not "monolith bad"), makes real <em>choices</em>
(incremental over big-bang, decouple-what-hurts over decouple-everything, safety-
first over features-first this quarter), and states explicit <em>sacrifices</em>
(no rewrite, slower features now for faster later) — so it actually guides
decisions ("should we extract module X? only if it's causing contention/risk —
our policy"). The naive "let's do microservices" has no diagnosis, no sacrifice,
and no coherence — it's a solution in search of a problem, which is why it would
likely become a multi-year rewrite that delivers little and possibly fails (as
these commonly do). It also connects to the leadership goal honestly (faster
features come <em>through</em> safe independent shipping, with an explicit
short-term trade), rather than either ignoring the business pressure or
pretending the rewrite is free.
<br><br>
Common mistakes: (1) producing a roadmap ("Q1 do this, Q2 do that") and calling
it strategy — missing the diagnosis and the choices; (2) adopting the team's
"microservices" solution without diagnosing whether coupling or something else is
the real problem; (3) refusing to state what you <em>won't</em> do, so the
strategy gives no real guidance; (4) making it aspirational fluff ("we'll be the
best-architected team!") with no coherent actions.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| The kernel of strategy (diagnosis/policy/actions) | *Good Strategy/Bad Strategy*, Richard Rumelt |
| Technical vision & strategy for engineers | *The Staff Engineer's Path*, Tanya Reilly |
| Writing engineering strategy | Will Larson — <https://lethain.com/eng-strategies/> |
| Strangler-fig / incremental migration | Martin Fowler — <https://martinfowler.com/bliki/StranglerFigApplication.html> |
| Setting technical direction | *An Elegant Puzzle*, Will Larson |

---

## Checkpoint

**Q1.** Distinguish vision, strategy, and roadmap — and explain why "we're going
to migrate to microservices" is a bad answer to "what's your strategy?"

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Vision</strong> is the destination and why it matters — where you're
going, aspirational and relatively stable (e.g., "any team ships safely, many
times a day"). <strong>Strategy</strong> is <em>how</em> you'll get there: the
overall approach and the deliberate choices, <em>including what you won't do</em>
— it's fundamentally about diagnosis and trade-offs (Rumelt's kernel: diagnose
the real problem, set a guiding policy, take coherent reinforcing actions).
<strong>Roadmap</strong> is the concrete what-and-when: the specific sequence of
work, which changes frequently. Vision = destination, strategy = route and
choices, roadmap = turn-by-turn. "We're going to migrate to microservices" fails
as a strategy on several counts: (1) <strong>it's a solution with no diagnosis</strong>
— it doesn't say what problem microservices solve <em>for this team</em>, so it
can't tell you whether it's even the right approach (maybe the real problem is
weak testing or unclear ownership, which microservices wouldn't fix and might
worsen); (2) <strong>it has no sacrifice/choice</strong> — real strategy states
what you're giving up and why, and "do microservices" chooses nothing (it doesn't
say which parts, in what order, at the cost of what); (3) <strong>it's actually a
roadmap item (a thing to build) masquerading as strategy</strong> — it's the
"what," not the "how and why"; (4) <strong>it doesn't guide decisions</strong> —
faced with "should we extract module X?", it gives no principle to decide by,
whereas a real strategy ("decouple the seams that cause the most team contention
and risk, incrementally, safety-first") does. It's the classic bad-strategy
pattern: a fashionable solution adopted before understanding the problem, which
is how teams end up in multi-year rewrites that deliver little because they never
diagnosed what was actually wrong. The fix is Rumelt's discipline: <em>diagnose
first</em> (what's the real problem — coupling? safety? ownership? speed?), then
derive the approach and the coherent actions from that diagnosis, and state the
trade-offs — which produces a strategy that actually directs the team's
decisions rather than a slogan that sounds like direction but provides none.
</details>

**Q2.** A good strategy states what you *won't* do. Why is the sacrifice the
hard, essential part — and what happens to a team whose "strategy" says yes to
everything?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Sacrifice is essential because strategy is fundamentally about <em>allocating
finite resources against competing options</em> — and if you don't choose, you
haven't strategized, you've just listed. Saying "we'll do X" only becomes
strategy when it implies "instead of Y" — the choice to consolidate on one
deployment path <em>means</em> not supporting the others; investing in the
testing safety net first <em>means</em> not shipping features as fast this
quarter. Those trade-offs are hard because each thing you cut has real advocates
and real costs (the team using the deprecated path is inconvenienced; leadership
wanting features now is disappointed), and it's emotionally easier to avoid the
conflict by saying "we'll do all of it." But that avoidance is exactly the
failure. A team whose "strategy" says yes to everything gets: (1) <strong>no
real guidance</strong> — if everything is a priority, nothing is, so engineers
have no principle to decide by and fall back to local optimization (the
incoherent drift the whole lesson is about); (2) <strong>spread-too-thin
execution</strong> — finite people and time divided across every initiative
means nothing gets the focus to actually succeed (the reliability work, the
feature work, and the migration all limp along half-done); (3) <strong>no
coherence</strong> — the "coherent actions" that reinforce each other (Rumelt)
require choosing a direction they can reinforce; a everything-list has actions
pulling in different directions. The lead's job — the hard part — is to make the
cuts and <em>own them explicitly</em>: "we're not doing Z this quarter, here's
why, here's what it costs and why it's worth it." That takes courage (you'll
disappoint people and defend the choice) and clarity (you have to actually
decide what matters most), which is precisely why weak leads avoid it and produce
mush. The tell of a real strategy is that reasonable people could <em>disagree</em>
with it (because it made genuine choices with genuine costs); a strategy nobody
could object to has made no choices and will guide nothing. Saying no — to
initiatives, to stakeholders, to good-but-not-most-important ideas — is the
substance of strategy, not a regrettable side effect of it.
</details>

---

## Homework

Write a one-page technical vision and strategy for a team you know well (your
current team, ideally). Force yourself through Rumelt's kernel: a real diagnosis
(what's the actual root problem, not the symptoms or the fashionable solution), a
guiding policy (the overall approach), coherent actions (2–4 reinforcing steps),
and — the hard part — an explicit list of what you *won't* do and why. Then test
it: would it actually guide a real decision your team faces? Could a reasonable
person disagree with it (proving it made real choices)? If it reads like a
goal-restatement or a wish-list, you haven't diagnosed yet — go back to the
problem.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise's value is in the discipline it forces, and the two tests at the end
are the graders. A strong response has a <em>diagnosis</em> that names a
non-obvious root cause rather than restating symptoms — the difference between
"we ship slowly" (symptom) and "we ship slowly because our release process
requires manual coordination across three teams and there's no automated safety
net, so every release is a scary human-intensive event" (diagnosis that implies
a solution). The <em>guiding policy</em> should follow from that diagnosis (not
be a fashionable solution bolted on), the <em>actions</em> should reinforce each
other and the policy, and the <em>won't-do list</em> should contain real
sacrifices that will disappoint someone (if it doesn't, you haven't made real
choices). The two final tests are what separate a real strategy from fluff: (1)
<strong>"would it guide a real decision?"</strong> — take an actual pending
decision your team faces and check whether your strategy tells you what to do; if
it's too vague to adjudicate a real choice ("be more reliable" can't decide
"should we build feature A or fix the deploy pipeline?"), it's not operative. (2)
<strong>"could a reasonable person disagree?"</strong> — if your strategy is so
unobjectionable that no sensible person could argue against it ("we'll improve
quality and ship faster"), it's made no real choices and is therefore useless as
guidance; a real strategy took a position that has genuine trade-offs and
alternatives someone could prefer. If your draft fails these — reads like a goal
("increase velocity"), a wish-list (everything's included), or a slogan (no
diagnosis) — the fix is to go back to the diagnosis: you jumped to solutions
before understanding the problem. The meta-skill this builds is the single most
valuable thing in technical leadership: the ability to look at a messy situation,
diagnose the <em>actual</em> problem beneath the symptoms and the fashionable
non-solutions, and articulate a coherent direction with honest trade-offs that a
team can actually follow. Most "strategy" in the wild is bad strategy (Rumelt's
whole thesis) — goals mistaken for plans, solutions with no diagnosis,
everything-lists that guide nothing — so a lead who can produce genuine strategy
(diagnose, choose, sacrifice, cohere) is disproportionately valuable, and this
one-pager, done honestly and pressure-tested against the two questions, is how
you practice it. Keep the page, reference it in real decisions, and revise it as
the diagnosis sharpens — a living strategy that changes what the team does is the
goal; a filed document nobody invokes is a strategy that failed regardless of how
well-written it was.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 7 — Architecture Decisions and ADRs →](lesson-07-adrs){: .btn .btn-primary }
