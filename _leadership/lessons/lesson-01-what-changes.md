---
title: "Lesson 01 — What Actually Changes"
nav_order: 1
parent: "Phase 1: The Transition"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 01: What Actually Changes

## Concept

The single most important idea in this entire course, and the one most new
leads get wrong for months: **your output is no longer what you personally
produce. It's what your team produces.**

As a senior engineer, your value was clear and measurable — you shipped
features, fixed hard bugs, made good technical calls. Your hands on the
keyboard created the value. The mental model was simple:

```
   SENIOR ENGINEER                    LEAD
   ──────────────                     ────
   your value = what YOU build        your value = what your TEAM builds
                                                    + teams you influence

   a great week = shipped the         a great week = your team shipped,
   thing, solved the hard problem     unblocked each other, made good calls,
                                       and grew — often WITHOUT you touching
                                       the code
```

This is a genuinely disorienting shift, because the thing that made you
successful (personal technical output) is now, at best, a small part of the
job — and at worst, a trap. The hours you spend heads-down coding are hours you
*aren't* doing the multiplier work only you can do: setting direction,
unblocking people, growing the team, making the calls that let five engineers
work well instead of one engineer working brilliantly.

Andy Grove (Intel's CEO) put it as a formula in *High Output Management*: a
manager's output is the output of their team plus the output of neighboring
teams they influence. Your leverage — the multiplier on your effort — is now
enormous *and* indirect. One good architectural decision, one unblocked
engineer, one well-run design review shapes weeks of others' work. But you
won't *feel* that value the way you felt shipping code, which is why this
transition is as much emotional as practical (Lesson 05).

---

## How It Works

### "Success this week" is redefined

The concrete change is in what you optimize for hour to hour. A senior
engineer's good day is measured in personal progress. A lead's good day is
measured in the *team's* progress and health — and the highest-value things you
do often leave no artifact with your name on it:

- You spotted that two engineers were about to build incompatible things, and a
  five-minute conversation saved a week of rework. (No commit. Enormous value.)
- You noticed a junior was stuck and afraid to ask, and unblocked them. (No
  commit. Kept the project on track.)
- You said no to a poorly-scoped request, protecting the team's focus. (No
  commit. Prevented weeks of waste.)

None of these show up in a git log, which is exactly why new leads, hungry for
the old feeling of productivity, drift back to coding — where they can *see*
their output. Resisting that pull is the core discipline of the role.

### Your old strengths can become traps

The skills that got you promoted don't disappear, but their *application*
changes:

- **Being the best coder** → the temptation to do the hard tasks yourself,
  which caps your team at your personal throughput and stops others from
  growing (Lesson 03).
- **Deep focus** → the maker's schedule, which fragments the moment you have a
  team depending on your availability (Lesson 04).
- **Knowing the answer** → jumping in with solutions instead of growing your
  team's ability to find them (Lesson 26, coaching vs mentoring).

The reframe: your technical skill is now *leverage for judgment and teaching*,
not for personal output. You use it to make better decisions, review designs,
grow people, and stay credible — not to out-code your team.

{: .note }
> **The discomfort is the job, not a sign you're failing**
> Nearly every new lead feels, for the first months, like they're "not doing
> real work" — because the real work no longer looks like the work they know.
> That feeling is the transition, not a warning. The engineers who struggle
> most are often the strongest ICs, precisely because they have the most to
> unlearn. Lesson 05 handles this directly; for now, just know it's universal.

---

## Lab — Scenario

> Read the situation, write your response in the blank, *then* reveal the model
> answer. The struggle to decide is where the learning is — don't skip it.

**The situation:** It's your first week as team lead. A production bug lands in
a subsystem you know intimately — you could fix it in about two hours. A newer
engineer on your team, Priya, also *could* fix it, but it would take her most
of the day and she'd need to ask you questions along the way. Separately, you
have a planning document due Friday that will shape the team's next quarter, and
you've barely started it. It's Tuesday.

**What do you do, and why? Walk through your actual decision — not the
textbook one.**

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
The instinct — fix it yourself, it's faster — is exactly the trap this lesson
is about, and giving in to it in week one sets the tone for your whole tenure.
The lead move is to <strong>give the bug to Priya</strong> and spend your time
on the planning doc, while making yourself available for her questions.

The reasoning, in leverage terms: fixing it yourself saves ~half a day <em>this
week</em> but produces nothing lasting — and it costs you the planning doc time
(the highest-leverage thing on your plate: it shapes a <em>quarter</em> of five
people's work), signals to the team that hard problems flow to you (so they
stop growing and you become the bottleneck — Lesson 03), and denies Priya a
real growth opportunity in a subsystem she needs to learn. Priya taking the day
"costs" more hours <em>this week</em> but: she levels up in that subsystem
(compounding — next time she's faster and independent), the bug still gets
fixed, and you protect your leverage work. That's the whole reframe — you
optimized for the team's output and growth over your personal throughput.

The nuance that separates a good answer from a naive one: <strong>severity and
context matter</strong>. If this were a Sev-1 outage costing money every minute,
"fastest fix wins" is correct — you fix it (or pair on it live), because
right-now impact outweighs growth; then you make it a teaching moment
afterward. The lesson isn't "never touch code" — it's "default to leverage and
growth, override only when immediate impact genuinely demands it." A first-week
non-emergency bug is squarely in the delegate-and-coach zone. Also good:
telling Priya explicitly "I'm giving you this because it's a great way to learn
X, and I'm here for questions — take the time you need" (framing it as
investment, not dumping — Lesson 30), and time-boxing your own availability so
the planning doc still gets done.

Common mistakes: (1) fixing it yourself and rationalizing it as "just this
once, I'm new and want a quick win" — the quick win is a step backward; (2)
delegating it but then hovering/taking over the moment Priya struggles (you've
now spent the time <em>and</em> undermined her); (3) delegating it but failing
to make yourself available, so she's stuck and resentful. The craft is
delegating the <em>outcome</em> while providing a safety net — which the whole
of Phase 6 develops.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| The manager's job & output | *High Output Management*, Andy Grove |
| The IC→manager transition | *The Manager's Path*, Camille Fournier (ch. on tech lead / new manager) |
| Multiplier leadership | *Multipliers*, Liz Wiseman |
| Staff/lead leverage & scope | StaffEng.com — <https://staffeng.com/guides/> |
| Engineering leadership community | LeadDev — <https://leaddev.com/> |

---

## Checkpoint

**Q1.** Your manager asks "what did you get done this week?" and you realize you
personally shipped no code. How do you think about — and answer — that
question?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The reframe first: "what did I get done" is now the wrong internal question —
the right one is "what did my <em>team</em> get done, and what did I do to make
that happen." Your no-code week may have been highly productive: you unblocked
two engineers, ran a design review that caught a costly flaw, had a career
conversation that re-engaged someone who was drifting, and wrote the plan that
directs next quarter — none of which is a commit, all of which is real output
(the multiplier work). To your manager, answer in <em>those</em> terms and, ideally,
in terms of team outcomes: "The team shipped X and unblocked Y; I spent my time
on the planning doc, unblocking Priya on the payments bug, and a 1:1 that
surfaced a retention risk we should discuss." This does two things: it trains
your manager (and yourself) to value the right output, and it demonstrates you
understand the role. The trap to avoid is feeling defensive and either
manufacturing code to "have something to show" or apologizing for not coding —
both signal you haven't internalized the shift. If your manager genuinely
expects you to still be a heavy individual contributor, that's a role-
expectations conversation to have explicitly (are you a tech lead who still
codes 50%, or an EM? — Lesson 02) — but the framing of your value in team terms
is correct regardless.
</details>

**Q2.** Name three high-value things a lead can do in a week that produce no
artifact with their name on it — and explain why the "no artifact" property is
exactly what makes them psychologically hard to keep doing.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Three (there are many): (1) <strong>Unblocking</strong> — spotting that someone
is stuck (technically, or on a dependency, or afraid to ask) and clearing it,
keeping their work and often the whole project moving. (2) <strong>Preventing
waste</strong> — catching a bad direction early (two engineers building
incompatible things, a poorly-scoped feature, a design that won't scale) with a
conversation or a design-review comment, saving days or weeks that now simply…
don't get wasted (invisible by nature — you can't point at the disaster that
didn't happen). (3) <strong>Growing people</strong> — a coaching conversation, a
well-chosen stretch assignment, feedback that changes someone's trajectory —
whose payoff is weeks or months out and shows up as <em>their</em> improved work,
not yours. Others: setting direction, making a decision that unblocks five
people, protecting the team's focus by saying no. Why "no artifact" makes them
hard to sustain: humans are motivated by visible progress and feedback (the same
reason a green test suite or a merged PR feels good) — and these activities give
none. The bug you prevented leaves no trace; the engineer you unblocked ships
<em>their</em> work; the disaster you averted is, definitionally, something that
didn't happen. So the lead gets none of the dopamine of "done," and under the
natural hunger for that feeling, drifts back toward coding, where output is
immediate and visible. Recognizing this is the defense: you have to
<em>deliberately</em> value and track the invisible work (keep a running note of
what you unblocked/prevented/grew), because the environment won't reward it
automatically the way it rewarded your commits. This is Lesson 05's psychology
and why measuring yourself weekly-in-team-terms rather than daily-in-personal-
terms is the survival skill of the role.
</details>

---

## Homework

Track your own week honestly, as if you were already a lead (or, if you are one,
this week). Each day, write down the single highest-value thing you did — and
force yourself to include at least three things that produced *no artifact*
(no commit, no doc, no ticket) but genuinely helped the team or a person. At the
end, answer: which felt more valuable in the moment (the artifact work or the
invisible work), which was *actually* more valuable to the team's outcomes, and
what that gap tells you about the discipline you'll need to build.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise's payoff is in confronting the gap you'll almost certainly find:
the artifact work (the code you wrote, the doc you finished) <em>felt</em> more
satisfying and "productive" in the moment — it gave you the closure and visible
progress your engineer-brain is wired to crave — while several pieces of the
invisible work (unblocking someone whose task was on the critical path, killing
a bad direction before it consumed a week, a conversation that re-motivated a
drifting teammate) were, on honest reflection, worth far more to what the team
actually accomplished and where it's heading. A strong reflection names that
inversion specifically and draws the conclusion: your <em>felt</em> sense of
productivity is now a misleading signal — it rewards the low-leverage familiar
work and starves the high-leverage unfamiliar work — so you cannot manage
yourself by how productive you feel day to day. The discipline this implies:
(1) deliberately schedule and protect the invisible/leverage work (1:1s,
unblocking rounds, planning) rather than treating it as interruption of the
"real" work; (2) track it explicitly so it becomes visible to you (the running
note of what you unblocked/prevented/grew — which also answers your manager's
"what did you do" question, Q1, and builds the case for your own performance
review, Lesson 05's homework); (3) recalibrate your internal scorecard from
daily-personal-output to weekly-team-output. If you found the invisible work
easy to identify and clearly more valuable, you're ahead; if you struggled to
find three no-artifact items or they felt like "not real work," that's
diagnostic — it's the strongest signal of how much the IC mindset still has you,
and exactly the thing Phase 1 (and Lesson 05's psychology) exists to shift. The
new leads who thrive are the ones who make peace with trading the reliable
dopamine of "I built this" for the delayed, diffuse, but far larger reward of
"my team is better and shipped more because of choices I made."
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 2 — Lead vs EM vs Staff: the Tracks →](lesson-02-lead-em-staff){: .btn .btn-primary }
