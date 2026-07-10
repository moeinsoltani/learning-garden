---
title: "Lesson 04 — Time and Leverage"
nav_order: 4
parent: "Phase 1: The Transition"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 04: Time and Leverage

## Concept

You have the same 40-ish hours you had as a senior engineer, but the job changed
underneath them. Lesson 03 got you off the critical path; this lesson is about
what to *do* with the time — and the answer is: spend it on **leverage**, the
activities that multiply your team's output rather than add to it directly.

Two ideas govern how a lead's time works:

**Maker's schedule vs manager's schedule** (Paul Graham). Engineers run on the
*maker's schedule*: long uninterrupted blocks, because deep technical work needs
flow and a single meeting mid-afternoon can destroy a whole day's productivity.
Leaders run on the *manager's schedule*: the day sliced into slots, context-
switching between people and problems, availability as a feature. The painful
truth: **you're now on the manager's schedule, and part of your job is protecting
the maker's schedule for your team.**

```
   MAKER'S SCHEDULE (your ICs)          MANAGER'S SCHEDULE (you now)
   ─────────────────────────            ────────────────────────────
   ┌──────────────────────┐             ┌──┬──┬──┬──┬──┬──┬──┬──┐
   │   deep work block    │             │1 │1 │  │mtg│ │1 │  │  │
   │   (4 hours, flow)    │             │:1│:1│rev│  │pl│:1│..│..│
   │                      │             └──┴──┴──┴──┴──┴──┴──┴──┘
   └──────────────────────┘             fragmented — and that's OK,
   one interruption = day ruined        because the WORK is different.
                                         Your job: guard THEIR blocks.
```

**Leverage** (Andy Grove): a lead's output isn't hours worked — it's the *impact
of what those hours enable*. A one-hour decision that unblocks five engineers for
a week is astronomically high-leverage; two hours coding a task someone else
could do is low-leverage. The whole game is spending your fragmented, meeting-
heavy time on the highest-leverage activities and ruthlessly cutting the rest.

---

## How It Works

### What high-leverage activities actually are

Grove's insight: managerial leverage = impact per unit of your time. The highest-
leverage activities share a property — they *shape or unblock the work of many
people*:

- **Decisions that unblock multiple people** — making a call five engineers are
  waiting on; a one-hour decision worth a week of collective progress.
- **Setting direction and priorities** — a planning doc or a clear priority
  ranking that keeps a whole team pointed the right way for a quarter (Lesson 06,
  Phase 10).
- **Design reviews** — an hour that catches a flaw worth weeks of rework, and
  teaches the team simultaneously (Lesson 09).
- **1:1s and growing people** — investments whose payoff is the compounding
  improvement of your engineers (Phase 5).
- **Removing recurring blockers** — fixing the thing that repeatedly stalls the
  team (a broken process, a missing tool, a dependency) once, so it stops costing
  everyone.

The lowest-leverage things a lead does are usually the ones that *feel* most
productive: coding a delegable task, attending meetings that don't need them,
answering things others could answer.

### The calendar is your primary tool

As a lead, your calendar *is* your strategy made visible — where your time goes
is what you actually prioritize, regardless of what you say. So audit it:

- **Look at last week honestly.** How much time on leverage work (decisions,
  direction, growing people) vs low-leverage (delegable coding, unnecessary
  meetings, reactive firefighting)? The gap is usually alarming.
- **Cut ruthlessly.** Decline meetings you don't add value to (or send a
  delegate). Kill recurring meetings that have outlived their purpose (Lesson
  15). Batch shallow work.
- **Protect blocks — yours and theirs.** Reserve blocks for the leverage work
  that never happens if you stay purely reactive (the planning doc that's always
  "later"). And protect your team's maker-blocks: cluster the meetings *you* need
  from them, shield them from the org's meeting sprawl, be the buffer.

### Batch interrupts; be available without being always-on

The tension: your team needs you available (that's part of the job — an
unblocking lead is high-leverage), but constant availability destroys your own
ability to do focused leverage work and models bad boundaries. The resolution:
batch. Office-hours blocks, a "grab me anytime" norm for true blockers but async-
first for everything else, clustering your reactive work rather than letting it
fragment every hour. You're optimizing for *the team's* flow, and sometimes
that means absorbing interruptions into yourself to protect theirs.

{: .note }
> **"Your calendar is your real priorities"**
> The gap between what a lead <em>says</em> matters and where their <em>time</em>
> actually goes is where good intentions die. You say growing the team matters,
> but your calendar shows zero time on 1:1s and career conversations, and forty
> hours of reactive firefighting and delegable coding. The calendar doesn't lie.
> Auditing and deliberately reshaping it — protecting time for the leverage work
> that's always "later" — is one of the highest-return habits a new lead can
> build.

---

## Lab — Scenario

**The situation:** Here's a week from the calendar of a struggling new team lead
(let's call him Sam). Sam feels constantly busy and constantly behind, the
team's quarterly planning doc is three weeks overdue, and two engineers have
mentioned they "never get time with him." His typical week:

- **Daily:** 9:00 standup (30 min, whole team, Sam runs it); ~3 hours scattered
  coding on a feature Sam owns (on the critical path); ~2 hours of Slack/email
  answered reactively throughout the day, fragmenting everything.
- **Recurring meetings:** a 1-hour weekly "sync" with an adjacent team (Sam
  mostly listens); a 1-hour weekly all-hands (org-wide); a 90-min weekly
  "architecture review" that's become a status meeting; a 30-min daily leadership
  check-in.
- **What's NOT on the calendar:** no 1:1s with his engineers, no protected time
  for the planning doc, no design-review time.

**Audit Sam's week: what do you cut, delegate, batch, or add — and justify each
move in leverage terms?**

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
Sam's calendar perfectly diagnoses his problems: it's full of low-leverage
reactive work and completely missing the high-leverage work that would fix what
he's struggling with. The audit:
<br><br>
<strong>CUT / DELEGATE (reclaim time from low-leverage activities):</strong>
(1) <em>The ~3 hours/day of critical-path coding</em> — this is the biggest
single problem and Lesson 03's exact trap. Sam is on the critical path, which is
why he's a bottleneck and has no time for leadership. Delegate the feature (with
coaching/pairing to set the owner up), reclaiming ~15 hours/week — the single
highest-return move. (2) <em>The adjacent-team sync Sam "mostly listens" to</em> —
if he adds no value, either send someone else, get the notes async, or drop it
(low leverage; an hour back). (3) <em>Standup at 30 min daily run by Sam</em> —
tighten to 10–15 min and rotate who runs it (grows the team, frees Sam, and a
30-min daily standup is usually bloated). (4) <em>The reactive Slack/email
fragmenting the whole day</em> — batch it into 2–3 defined blocks instead of
answering continuously (protects focus; models good boundaries), while keeping a
"ping me for true blockers" norm.
<br><br>
<strong>FIX (reclaim a broken meeting):</strong> The 90-min "architecture review"
that's become a status meeting — either restore it to actually reviewing
designs (high leverage — Lesson 09) or, if status is all that's needed, replace
it with an async written update (Lesson 15's kill-the-zombie-meeting) and reclaim
90 minutes. A 90-min weekly meeting that's degraded into status is pure waste.
<br><br>
<strong>ADD (the missing high-leverage work — this is what fixes Sam's actual
problems):</strong> (1) <em>Weekly 1:1s with each engineer</em> (30 min each) —
this directly addresses "two engineers never get time with him" and is among the
highest-leverage things a lead does (Phase 5); it's <em>non-negotiable</em> and
the fact that it's missing entirely is the clearest sign Sam hasn't made the
transition. (2) <em>A protected block for the planning doc</em> — it's three
weeks overdue precisely because it's never scheduled and always loses to reactive
work; the fix is to <em>put it on the calendar as a protected block</em> (the
planning doc shapes a quarter of the team's work — enormous leverage, and its
absence is why the team lacks direction). (3) <em>Design-review time</em> —
scheduled space to review the team's designs (catches costly flaws early, grows
people).
<br><br>
<strong>The leverage logic tying it together:</strong> Sam feels busy-and-behind
because his time goes to low-leverage work (his own coding, reactive comms,
status meetings) while the high-leverage work that would actually help (1:1s,
planning, direction, design review) gets zero time. The audit reclaims ~20+
hours from the low-leverage activities and spends them on the leverage work —
and notably, doing so fixes <em>all three</em> of Sam's stated problems at once:
delegating the coding un-bottlenecks him and creates time; adding 1:1s addresses
the neglected engineers; protecting the planning block gets the overdue doc done
and gives the team direction. The meta-lesson: Sam's calendar was the diagnosis
<em>and</em> the prescription — "constantly busy and constantly behind" almost
always means time is going to low-leverage work while high-leverage work is
starved, and the fix is a deliberate reshaping of where the hours go, not
working more hours (Sam doesn't need more time, he needs to spend the time he has
on what multiplies rather than what adds).
<br><br>
Common mistakes in the audit: (1) telling Sam to "manage his time better" or
"be more disciplined" without changing what's ON the calendar (the problem is
structural, not willpower); (2) cutting meetings but not adding the missing
leverage work (you've freed time but not redirected it — he'll just fill it with
more reactive work or coding); (3) keeping Sam on the critical-path coding
because "the feature needs to ship" (that's the hero trap that <em>caused</em>
the bottleneck — the fix is delegation, Lesson 03).
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Managerial leverage | *High Output Management*, Andy Grove |
| Maker's schedule vs manager's schedule | Paul Graham — <https://www.paulgraham.com/makersschedule.html> |
| Calendar auditing & focus | *Deep Work*, Cal Newport |
| Protecting team focus / meeting hygiene | LeadDev — <https://leaddev.com/> |
| Time management for engineering leaders | *The Manager's Path*, Camille Fournier |

---

## Checkpoint

**Q1.** You have two hours free this afternoon. Option A: knock out a coding task
on your team's backlog that you could do well. Option B: spend it on a planning
document that's overdue and a 1:1 with an engineer who seems disengaged. Which is
higher leverage, and how do you tell in general?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Option B, decisively — and the way you tell in general is to ask: <em>which use
of my time multiplies the work of many people or shapes it over a long horizon,
versus adds to it directly and once?</em> Option A (the coding task) is
low-leverage: it produces one unit of work that someone else on the team could
also produce (so your unique contribution is near zero — you're substituting for
an engineer, not doing something only a lead can do), and its impact is bounded
and one-time. Option B is high-leverage on both counts: the <em>planning
document</em> shapes a quarter of the whole team's work (its impact is
multiplied across everyone and stretches over months — a few hours now directs
weeks of collective effort in the right vs wrong direction), and the <em>1:1
with the disengaged engineer</em> addresses a potential retention/performance
problem whose cost, if it goes unaddressed and they leave or check out, dwarfs
the two hours (rehiring, lost knowledge, team morale — and re-engaging them
compounds through all their future work). The tell in general — Grove's leverage
formula: estimate impact-per-unit-of-your-time. Leverage is high when the
activity (a) affects many people rather than one, (b) has lasting rather than
one-time impact, or (c) is something <em>only you</em> can do (a decision only
you have the authority/context for, a relationship only you have). Coding a
delegable task scores low on all three; planning and growing people score high.
The trap that makes A tempting: it <em>feels</em> more productive (visible output,
the satisfying flow of Lesson 01/03) while B feels amorphous and its payoff is
delayed and invisible — which is exactly why leads default to the low-leverage
familiar work and starve the high-leverage work, and why the discipline is to
<em>override the felt sense of productivity</em> and spend your scarce leadership
time on what multiplies. (The one exception: if the coding task is genuinely
on the critical path and blocking the whole team <em>right now</em> and no one
else can do it in time — then unblocking the team is itself high-leverage. But a
backlog task, by definition, isn't that.)
</details>

**Q2.** Explain the maker's-schedule / manager's-schedule distinction, and the
specific obligation it creates for a lead beyond just managing their own time.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Paul Graham's distinction: <strong>makers</strong> (engineers doing deep
technical work) need long uninterrupted blocks because their work requires flow
state, which takes time to enter and is destroyed by interruption — so a single
meeting dropped in the middle of a maker's afternoon doesn't cost one hour, it
can cost the whole afternoon (the meeting fragments the day into two
too-short-to-go-deep pieces, plus the anticipation of it prevents settling into
flow beforehand). <strong>Managers</strong> run on a different schedule: the day
sliced into slots, moving between people and problems, where availability and
context-switching <em>are</em> the work — a meeting isn't a disruption, it's the
job. As a lead you've moved onto the manager's schedule (your work is now
decisions, unblocking, people, meetings — fragmented by nature, and that's
appropriate). The obligation this creates <em>beyond your own time</em>: because
your team is still on the maker's schedule and their deep work is where the
actual engineering output comes from, <strong>you are responsible for protecting
their maker-blocks</strong> — you're now the buffer between them and the
organization's meeting sprawl. Concretely: cluster the meetings <em>you</em> need
from them (don't scatter a standup, a sync, and a review across their day —
batch them at the edges so they get long uninterrupted middles); shield them
from meetings that don't need them (attend org meetings yourself and relay, so
five engineers don't lose an afternoon); default to async communication for
non-urgent things (so a question doesn't interrupt flow); and model and defend
"focus time" as legitimate and protected. This is a genuine and often
overlooked part of the role: a lead who lets the org fragment their team's days
(or who fragments them personally by scheduling meetings and pinging constantly
for non-urgent things) is directly reducing the team's output, no matter how
much they "help" in those interactions. The reframe: you traded your own
maker-schedule for the manager-schedule, and part of what you're now managing
<em>is</em> the makers' schedules — protecting the conditions for deep work is a
core leadership responsibility, not a nicety. The lead who understands this
absorbs interruptions and meeting-load into themselves to keep their team's days
whole — which is itself a high-leverage act (Q1), because the team's protected
deep work is where the real value gets made.
</details>

---

## Homework

Audit your own last week (real calendar + honest accounting of where the time
actually went, including the reactive/unscheduled work). Categorize every block
as high-leverage (decisions, direction, growing people, unblocking many) or
low-leverage (delegable work, unnecessary meetings, reactive firefighting you
could have prevented or batched). Calculate the rough split. Then redesign next
week: what will you cut, delegate, batch, or add — and specifically, what
leverage work that's currently getting zero time will you protect a block for?
If you're not yet a lead, do this for your current week and imagine the redesign
as a lead of your team.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise almost always produces an uncomfortable and useful finding: far
more of the week went to low-leverage work than expected, and the high-leverage
work that would most help is getting little or no protected time. A strong
response is honest and specific — a real accounting ("~14 hours coding a task I
could have delegated; ~6 hours in meetings I added no value to; ~5 hours of
reactive Slack fragmenting everything; ~2 hours in genuinely high-leverage work
— one design review and part of one 1:1") and a rough split that reveals the gap
(e.g., "maybe 15% high-leverage, 85% low-leverage — the opposite of what I'd want
for a lead"). The redesign should be concrete moves, not resolutions: <em>cut</em>
(name the specific meetings to decline or send a delegate to), <em>delegate</em>
(name the coding/tasks to hand off and to whom, with the coaching plan — Lesson
03/30), <em>batch</em> (define the 2–3 blocks for reactive comms instead of
all-day fragmentation), and — most importantly — <em>add and protect</em> the
missing leverage work: the specific block on the calendar for the planning/
direction work that's always "later," the 1:1s that aren't happening, the design-
review time. The key insight the exercise should produce: you probably don't
need more hours — you need to spend the hours you have on what multiplies rather
than what adds (Q1), and the mechanism for that is <em>reshaping the calendar</em>
(making the high-leverage work a scheduled, protected commitment rather than
something you hope to get to), because whatever isn't on the calendar loses to
whatever is. The pattern to internalize: your calendar is a truthful statement of
your real priorities, and if it doesn't show time on the things you claim matter
(growing the team, setting direction), then despite your intentions you're not
actually doing them — the fix is structural (schedule and protect them), not
motivational (try harder to fit them in). If your audit shows you're <em>already</em>
mostly on leverage work with protected blocks and delegated ICs, excellent —
that means you've made the transition, and the ongoing discipline is defending
those blocks against the reactive tide (and your team's maker-blocks against the
org's — Q2). Re-run this audit quarterly; calendar drift toward reactive low-
leverage work is constant, and the periodic re-audit is how you correct it before
you're the busy-and-behind Sam from the lab.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 5 — Identity, Doubt, and the Psychology of the Switch →](lesson-05-identity-psychology){: .btn .btn-primary }
