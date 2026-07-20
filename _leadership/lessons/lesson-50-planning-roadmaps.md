---
title: "Lesson 50 — Planning and Roadmaps"
nav_order: 1
parent: "Phase 10: Project Leadership"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 50: Planning and Roadmaps

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **milestone** — a marked point of progress; good ones *prove* something, not just mark dates.
> - **risk retirement** — removing an unknown by proving the risky thing works (or doesn't) early.
> - **slack / buffer** — deliberately unscheduled time that absorbs the inevitable surprises; a feature, not padding.
> - **brittle vs robust** — shatters at the first surprise vs absorbs surprises and holds.
> - **work backward** — planning from the target date to today, to see what must be true when.
> - **now / next / later** — a roadmap format whose precision honestly decreases with distance.
> - **Gantt chart** (GANT) — the classic bar-chart project schedule that pretends to know far-future dates.
> - **false precision** — exact-looking numbers that imply certainty nobody has.
> - **cadence** — the repeating rhythm of planning and reality-checking (Lesson 38).
> - **stale artifact** — a document nobody updates or believes anymore.

## Concept

Plans are essential and yet almost always wrong — "no plan survives contact with reality." The goal
isn't a perfect plan (impossible) but a plan that's **useful despite being wrong** — one that helps you
navigate, adapt, and retire risk, rather than a rigid schedule that shatters on first contact. The key
ideas: **milestones as risk-retirement** (each milestone proves something / removes a risk, not just
marks calendar time), **slack as a feature** (build in buffer — plans with no slack always slip), and a
**planning cadence** (plan, then reality-check regularly).

```
   MILESTONES = RISK RETIREMENT (not calendar decoration)
   ┌──────────────────────────────────────────────────────┐
   │ BAD milestone: "week 4: 50% done" (calendar, meaningless)│
   │ GOOD milestone: "week 4: proven the risky integration   │
   │   works end-to-end" (retires a RISK)                    │
   │ • work BACKWARD from the date                           │
   │ • SLACK is a feature (buffer for the unknowns)          │
   │ • CADENCE: plan (quarterly) + reality-check (weekly)    │
   └──────────────────────────────────────────────────────┘
```

The reframe: **plan to retire risk and to adapt, not to predict — milestones prove things (retire
risks), slack absorbs the inevitable unknowns, and you reality-check the plan continuously.** A plan
that's just a calendar of "% done by date" is useless (it doesn't help you know if you're actually on
track or handle surprises); a plan built around retiring the real risks (proving the hard things work
early) and with slack for the unknowns is genuinely useful — it helps you navigate reality rather than
being broken by it.

---

## How It Works

### Milestones as risk-retirement, not calendar decoration

The key shift in how to think about milestones: a good milestone **retires a risk or proves something**,
not just marks calendar progress. "Week 4: 50% done" is calendar decoration — meaningless (50% of what?
does it mean we're on track? no way to tell). "Week 4: the risky third-party integration works
end-to-end" is a real milestone — it <em>retires a risk</em> (we now know the scariest part works, or we
found out it doesn't). Structure milestones around <em>de-risking</em>: each one should prove a hard thing
works, remove a major unknown, or validate a key assumption — so that as you hit milestones, the project
gets genuinely de-risked (the remaining unknowns shrink). This makes milestones meaningful (they tell you
about real progress and risk) rather than arbitrary calendar checkpoints.

### Work backward from the date

When there's a target date, **work backward** from it: what has to be true by then, and therefore by
each preceding milestone, to hit it? Working backward (from the goal to the steps) rather than only
forward (from now, hoping it adds up) surfaces whether the date is feasible and what each milestone must
achieve. It reveals early if the date is impossible (the backward math doesn't fit) — a much better time
to find out than at the deadline. Combine with forward planning (what can we actually do) to reconcile
the desired date with reality.

### Slack is a feature

**Build slack (buffer) into plans** — because the unknowns, surprises, and inevitable problems <em>will</em>
consume time, and a plan with no slack (every day scheduled to the max, no buffer) is guaranteed to slip
(the first surprise blows it). Slack isn't waste or padding to hide; it's a deliberate feature that
absorbs the unknowns that always occur, making the plan robust rather than brittle. A plan with realistic
slack is more likely to be met (and more honest) than an aggressive no-slack plan that assumes everything
goes perfectly (which never happens). Resisting the pressure to remove all slack (to look faster) is
important — the no-slack plan just fails later.

### Now / next / later roadmaps

For roadmaps (vs. detailed project plans), a useful format is **now / next / later** — instead of false-
precision dated commitments far out (which will be wrong), express the roadmap as what we're doing
<em>now</em> (committed, specific), <em>next</em> (soon, less certain), and <em>later</em> (directional,
vague). This honestly conveys certainty decreasing with distance (near-term is firm, far-term is
directional), avoids over-committing to specific far-future dates (which reality will break), and is more
truthful than a Gantt chart pretending to know exact dates a year out. Match the precision to the actual
certainty.

### The planning cadence — plan + reality-check

Planning isn't one-time — it's a **cadence**: plan at a longer interval (e.g., quarterly — set direction
and milestones) AND reality-check frequently (e.g., weekly — is the plan still holding? what's changed?
are we hitting milestones? new risks?). The reality-checks catch divergence early (so you adapt while
there's room), keep the plan current (as reality unfolds), and prevent the plan from becoming a stale
artifact everyone ignores. A plan is a living thing you revise as you learn — the cadence (plan +
frequent reality-check) is what keeps it useful, versus a plan made once and never revisited (which
diverges from reality and becomes useless).

{: .note }
> **Plan to retire risk and adapt, not to predict — milestones prove things, slack is a feature</br>**
> No plan survives reality, so the goal is a plan that's useful despite being wrong — one that helps you
> navigate and de-risk, not a rigid schedule. Make <em>milestones retire risks</em> (each proves a hard
> thing works or removes an unknown — "the risky integration works," not "50% done"), <em>work backward</em>
> from the date (to surface feasibility early), build in <em>slack</em> (a feature that absorbs the
> inevitable unknowns — no-slack plans always slip), use <em>now/next/later</em> roadmaps (precision
> matching actual certainty), and run a <em>planning cadence</em> (plan quarterly + reality-check weekly,
> so the plan stays current and divergence is caught early). A plan built around retiring real risks with
> realistic slack, continuously reality-checked, genuinely helps you deliver — versus a calendar of "% done
> by date" that shatters on contact with reality.

---

## Lab — Scenario

**The situation:** Leadership has handed you a vague quarterly goal: **"Modernize our authentication
system this quarter"** (the current auth is old, insecure, and hard to maintain; they want it replaced).
That's it — no plan, no milestones, just the goal and the quarter. You need to turn this into a real
milestone plan.

**Turn the vague goal into a milestone plan with explicit risks retired at each milestone.** Then note the
principles and mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong plan structures milestones around retiring the real risks of an auth modernization, works
backward from the quarter, and builds in slack. Example:
<br><br>
<strong>First — identify the real risks</strong> (what could kill this): (a) the new auth approach might
not handle all our use cases (SSO, existing sessions, edge cases); (b) migrating existing users/sessions
without breaking anyone (risky — auth is critical path); (c) integration with all the services that depend
on auth; (d) security correctness (getting auth wrong is dangerous); (e) unknown dependencies on the old
system. The plan should retire these, hardest/scariest first.
<br><br>
<strong>The milestone plan (each retires a risk):</strong> <br>
• <strong>Milestone 1 (~week 2) — Approach validated:</strong> chosen the new auth approach/technology and
proven (via a spike/prototype) it handles our hardest use cases (SSO, sessions, the edge cases). <em>Risk
retired: "the approach might not work for our needs."</em> <br>
• <strong>Milestone 2 (~week 4) — Core auth working + migration path proven:</strong> new auth works for a
basic flow end-to-end, AND we've proven (on a test) how to migrate existing users/sessions without breaking
them. <em>Risk retired: "we can't migrate existing users safely."</em> <br>
• <strong>Milestone 3 (~week 7) — Integrated with all dependent services (in staging):</strong> all
services that depend on auth work with the new system in staging. <em>Risk retired: "integration with
dependent services breaks."</em> <br>
• <strong>Milestone 4 (~week 9) — Security-reviewed and hardened:</strong> security review passed, edge
cases handled. <em>Risk retired: "we got auth security wrong."</em> <br>
• <strong>Milestone 5 (~week 11) — Rolled out safely:</strong> migrated production with a safe rollout
(gradual, with rollback), monitoring clean. <em>Risk retired: "the production cutover fails."</em> <br>
• <strong>Slack: ~1-2 weeks buffer</strong> (weeks 11-13) for the inevitable surprises — auth is critical
and surprises are guaranteed. <em>(Slack as a feature.)</em>
<br><br>
<strong>Working backward:</strong> "To roll out safely by end of quarter (week 13, with buffer), we need
security review by ~week 9, integration by ~week 7, migration proven by ~week 4, approach validated by ~week
2 — which is why the milestones sequence this way. If the backward math didn't fit the quarter, I'd flag
that the scope/date needs adjusting <em>now</em>, not at the deadline."
<br><br>
<strong>Cadence:</strong> "Weekly reality-checks: are we hitting the milestone (retiring the risk)? what
new risks emerged? is the plan still holding? Adjust as we learn."
<br><br>
<strong>Principles:</strong> (1) <strong>Milestones retire risks</strong> — each proves a hard thing works
(approach, migration, integration, security, rollout), so hitting them genuinely de-risks the project (not
"% done"). (2) <strong>Hardest/scariest first</strong> — validate the risky approach and migration early
(retire the biggest risks first, so if they fail, you find out early). (3) <strong>Work backward</strong>
from the quarter to check feasibility and sequence. (4) <strong>Slack</strong> — buffer for the inevitable
surprises (auth is critical, surprises guaranteed). (5) <strong>Cadence</strong> — weekly reality-checks to
catch divergence early. (6) <strong>Turn vague into concrete</strong> — the vague "modernize auth" becomes
a risk-retiring milestone sequence. <strong>Mistakes to avoid:</strong> (1) <strong>calendar milestones</strong>
("week 4: 50% done") — meaningless; make them retire risks; (2) <strong>easy stuff first, risky stuff
last</strong> — leaves the scariest risks unretired until late (discovering the approach doesn't work in
week 10 is a disaster); do hardest-first; (3) <strong>no slack</strong> — a no-buffer plan for a critical
system with guaranteed surprises will slip; (4) <strong>not working backward</strong> — not checking if the
quarter is even feasible (finding out at the deadline); (5) <strong>plan-and-forget</strong> — no
reality-check cadence, so divergence isn't caught; (6) <strong>false precision</strong> — pretending to know
exact dates for a project full of unknowns (the milestones are ~approximate, adjusted as you learn); (7)
<strong>ignoring the migration/rollout risk</strong> — auth is critical-path; the risky part is often the
safe cutover, not the new code.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Planning & milestones | *Shape Up*, Basecamp; *The Manager's Path*, Fournier |
| Now/next/later roadmaps | Janna Bastow (ProdPad) — now/next/later |
| Slack / buffer | *Slack*, Tom DeMarco; critical chain buffers |
| Risk-based milestones | Lesson 52 (risk management); de-risking order |

---

## Checkpoint

**Q1.** Why should milestones "retire risk" rather than mark calendar progress, and why is slack a feature?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Milestones should "retire risk" rather than mark calendar progress because <strong>risk-retiring
milestones give real, meaningful information about the project's health and progress, while calendar
milestones ("50% done by week 4") are meaningless and don't tell you whether you're actually on track or
what's still uncertain</strong>. A calendar milestone like "week 4: 50% done" is decoration: (1) it's
<em>ambiguous</em> — 50% of what? (effort? features? lines?) — and hard to assess honestly (teams report
"on track / green" while real problems lurk); (2) it doesn't tell you about <em>risk</em> — being "50%
done" says nothing about whether the hard, uncertain parts work (you could be 50% done with all the easy
parts and the scariest risk completely unaddressed, meaning you're actually in great danger despite looking
"on track"); (3) it doesn't help you <em>navigate</em> — it gives no actionable information about what to
worry about or do next. A risk-retiring milestone ("week 4: the risky third-party integration works
end-to-end") is meaningful: (1) it's <em>verifiable</em> — either the integration works or it doesn't (a
clear, honest signal, not a fuzzy "% done"); (2) it tells you about <em>real progress and risk</em> — hitting
it means a major risk is <em>retired</em> (you now know the scariest part works), and missing it means you've
found a real problem early (when you can still respond); (3) it <em>de-risks the project</em> — as you hit
risk-retiring milestones, the remaining unknowns shrink (the project gets genuinely safer), so the milestone
sequence is a de-risking process, not just calendar tracking. Structuring milestones around retiring the
real risks (proving hard things work, validating assumptions, removing unknowns) — ideally hardest/scariest
first — means that hitting milestones actually tells you the project is getting safer, and missing one
surfaces a real risk early (when it's cheap to address). This is far more useful than calendar milestones
that track time-elapsed without revealing whether the risky parts work — the difference between a plan that
genuinely helps you know if you're on track and de-risk, versus one that just marks time and gives false
comfort. Why <strong>slack is a feature</strong>: <strong>because the unknowns, surprises, and inevitable
problems will consume time, so a plan with no slack (every day scheduled to the max) is guaranteed to slip
at the first surprise — while slack absorbs those inevitable unknowns, making the plan robust rather than
brittle</strong>. Every real project encounters surprises: unforeseen problems, things that take longer than
expected, unknown unknowns, dependencies that slip. These <em>will</em> happen (you can't predict which, but
you can be certain some will). A plan with no slack — where every day is fully scheduled assuming everything
goes perfectly — has no capacity to absorb these surprises, so the first one (which is guaranteed) pushes
everything back, and the plan slips. A no-slack plan is essentially a bet that nothing will go wrong, which
never holds. Building in <strong>slack</strong> (buffer time not allocated to specific tasks) makes the plan
<em>robust</em>: when surprises consume time, the slack absorbs them, so the plan can still be met despite
the inevitable problems. Slack isn't waste or padding to hide (it's not laziness or sandbagging) — it's a
<em>deliberate feature</em> that accounts for the reality that unknowns always occur, making the plan honest
(it doesn't assume perfection) and achievable (it can absorb the surprises). The pressure is often to remove
slack (to look faster, to fit an aggressive date), but a no-slack plan just fails later (the surprises come,
and there's no buffer), so removing slack doesn't make the work faster — it makes the plan less honest and
guaranteed to slip. A plan with realistic slack is more likely to be met and more truthful than an aggressive
no-slack plan that assumes everything goes perfectly. So slack is a feature because it's what makes a plan
survive contact with the reality of inevitable surprises — the robustness that comes from not assuming
perfection. Both points reflect planning to handle reality rather than to predict it perfectly: milestones
retire risk (giving real information and de-risking, rather than marking meaningless calendar progress), and
slack absorbs the inevitable unknowns (making the plan robust rather than brittle) — together making a plan
that's useful despite being wrong, helping you navigate and de-risk rather than shattering on first contact
with reality.
</details>

**Q2.** Why work backward from the date, and why is a planning cadence (plan + frequent reality-check)
better than a one-time plan?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Working backward from the date</strong> is valuable because <strong>it reveals whether the target
date is actually feasible and what each milestone must achieve to hit it — surfacing infeasibility early
(when you can still act) rather than at the deadline</strong>. When there's a target date, you can plan two
ways: forward (from now, do the work and hope it adds up to the date) or backward (from the date, what has
to be true by then, and therefore by each preceding step?). Working <em>backward</em> — asking "to hit the
date, what must be true by then, and by each milestone before it?" — has key advantages: (1) <strong>it
tests feasibility</strong> — by mapping backward from the goal to the required intermediate states and their
timing, you find out whether the date is even achievable (does the backward math fit in the available time?);
if it doesn't fit (the required milestones can't be reached in time), you've discovered the date is
infeasible — <em>early</em>, when you can still respond (adjust scope, date, or resources), rather than
discovering it at the deadline (when it's too late and you just miss). (2) <strong>it defines what each
milestone must achieve</strong> — working backward tells you what has to be done by each point to stay on
track for the date, giving each milestone a clear required outcome (rather than vague "make progress"). (3)
<strong>it grounds the plan in the goal</strong> — the plan is structured around reaching the target, not
just doing work and hoping. Forward-only planning, by contrast, risks discovering too late that the work
doesn't fit the date (you kept doing work, and at the deadline realize it's not done). So working backward
(combined with forward planning — what you can actually do — to reconcile the two) surfaces the feasibility
question early and structures the plan around hitting the date. Finding out early that a date is impossible
is enormously valuable (you can renegotiate — Lesson 44 — while there's room), versus finding out at the
deadline (a miss). Why a <strong>planning cadence</strong> (plan + frequent reality-check) is better than a
one-time plan: <strong>because reality unfolds and diverges from any plan, so a plan must be continuously
revised against reality to stay useful — a one-time plan becomes stale and diverges into uselessness</strong>.
No plan survives contact with reality — as the project progresses, things change (surprises occur, estimates
prove wrong, priorities shift, new information emerges), so the plan made at the start increasingly diverges
from what's actually happening. A <strong>one-time plan</strong> (made once, never revisited) becomes a stale
artifact: it reflects the assumptions and knowledge of the start, which reality has since invalidated, so it
becomes inaccurate and eventually ignored (everyone knows it's not real anymore). A <strong>planning
cadence</strong> — planning at a longer interval (quarterly: set direction and milestones) AND reality-
checking frequently (weekly: is the plan holding? are we hitting milestones? what's changed? new risks?) —
keeps the plan useful because: (1) <strong>it catches divergence early</strong> — frequent reality-checks
detect when the project is diverging from the plan (a missed milestone, a new risk, a slipping estimate)
early, when you can still adapt (adjust the plan, the scope, the approach) — rather than divergence
accumulating unnoticed until it's a crisis; (2) <strong>it keeps the plan current</strong> — regularly
revising the plan against reality means it stays an accurate reflection of the actual situation (updated as
you learn), rather than a stale artifact; (3) <strong>it makes the plan a living tool</strong> — a plan
that's continuously reality-checked and revised remains useful for navigation (it reflects current reality
and helps decide what to do next), versus a one-time plan that's abandoned once reality moves past it. The
cadence treats planning as an <em>ongoing process</em> (plan, act, reality-check, revise, repeat) rather than
a one-time event — which matches the reality that plans are wrong and must be adapted. The reality-checks are
where the plan stays connected to reality; without them, the plan diverges and dies. So both points reflect
planning as adaptive navigation rather than one-time prediction: work backward to test feasibility and
structure toward the goal (surfacing problems early), and run a cadence (plan + frequent reality-check) to
keep the plan continuously aligned with unfolding reality (catching divergence early and staying useful) —
together making planning a living process that helps you navigate reality rather than a one-time prediction
that reality invalidates.
</details>

---

## Homework

Improve how you plan. (1) For a project you're planning (or a past one), define milestones that <em>retire
risk</em> (each proves a hard thing works or removes an unknown), hardest/scariest first — rather than
calendar milestones ("% done"). (2) Work backward from the target date — does the backward math fit? If not,
flag it early. (3) Is there slack in your plan, or is it scheduled to the max (guaranteed to slip)? Build in
realistic buffer. (4) Do you have a planning cadence (plan + frequent reality-check), or plan once and forget?
Reflect: do your plans retire risk and adapt, or predict and shatter — and what's one way to make a current
plan more risk-retiring, slack-aware, and continuously reality-checked?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds planning skill — plans that survive contact with reality. A strong response defines
risk-retiring milestones (hardest first), works backward from the date to check feasibility, builds in slack,
and runs a planning cadence. The realizations: (1) <strong>milestones should retire risk, not mark calendar
progress</strong> — a risk-retiring milestone ("the risky integration works") gives real information and
de-risks the project, where "50% done" is meaningless and gives false comfort; sequence hardest/scariest
first so big risks are retired early; (2) <strong>work backward from the date</strong> — to test feasibility
(does the backward math fit?) and define what each milestone must achieve, surfacing infeasibility early
(when you can renegotiate) rather than at the deadline; (3) <strong>slack is a feature</strong> — the
inevitable surprises consume time, so a no-slack plan is guaranteed to slip; buffer absorbs the unknowns,
making the plan robust and honest; (4) <strong>run a planning cadence</strong> — plan + frequent reality-check
keeps the plan current and catches divergence early, where a one-time plan diverges into uselessness. On
reflection, people commonly find their milestones are calendar-based (not risk-retiring), that they don't
work backward to check feasibility, that their plans lack slack (guaranteed to slip), and that they plan once
without reality-checking. The highest-value change is usually: structure milestones around retiring the real
risks (hardest first) and build in slack, plus reality-check regularly. The meta-point: no plan survives
reality, so the goal is a plan that's useful despite being wrong — one that helps you navigate and de-risk,
not a rigid schedule. Milestones that retire risk (giving real information and de-risking), working backward
(testing feasibility early), slack (absorbing inevitable surprises), now/next/later roadmaps (precision
matching certainty), and a planning cadence (staying aligned with reality) together make planning adaptive
navigation rather than doomed prediction. This is the foundation of project leadership — delivering the work
you've decided is worth doing. The next lesson tackles the hardest part of planning — estimation — why it
fails, and how to estimate honestly and communicate uncertainty so you stop being surprised.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 51 — Estimation and Why It Fails →](lesson-51-estimation){: .btn .btn-primary }
