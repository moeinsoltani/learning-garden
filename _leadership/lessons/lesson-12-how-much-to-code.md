---
title: "Lesson 12 — How Much Should a Lead Still Code?"
nav_order: 7
parent: "Phase 2: Technical Leadership"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 12: How Much Should a Lead Still Code?

## Concept

This is the question every new technical leader wrestles with, and the honest
answer frustrates people who want a number: **it depends** — on your role, your
team's size, and what the team actually needs from you. But "it depends" isn't a
dodge; it's a real framework, and getting your coding level wrong in either
direction is a common, costly mistake.

The two failure modes bracket the answer:

```
   TOO MUCH CODING                        TOO LITTLE CODING
   ────────────────                       ──────────────────
   the hero trap (Lesson 3):              the lost-touch trap (Lesson 3):
   on the critical path, bottleneck,      can't review designs credibly,
   team can't grow, leverage work         makes disconnected decisions,
   undone                                  loses the team's respect

        ← the sustainable level lives in between, and MOVES →
              with role, team size, and situation
```

The level isn't fixed — it shifts with your role (a tech lead of a small team
codes far more than an EM of two teams) and with the situation (more hands-on
during a crunch or a critical incident, less during a period of heavy people-work
or planning). The skill is finding *your* sustainable level for *your* situation,
staying technically credible without becoming a bottleneck — and adjusting it
deliberately as things change, rather than defaulting to either extreme.

---

## How It Works

### It scales inversely with role scope

The rough rule: the more people and scope you're responsible for, the less you
code, because the leverage math shifts. A **tech lead of a 4-person team** might
code 30–50% — the team is small enough that your hands add meaningful throughput,
and staying deep is part of the role. An **EM of an 8-person team** might code
10–20% (or less) — with more people to grow, more coordination, and more
leverage-work, your coding time is better spent elsewhere, and being on the
critical path across a bigger team is riskier. An **EM of two teams / a senior
EM** might code ~0% on the critical path — their leverage is entirely in people,
direction, and cross-team work, and any coding is exploratory/learning, not
delivery. As scope grows, the opportunity cost of your coding rises (the leverage
work you're not doing is worth more) and the risk of bottlenecking grows (more
people depend on your availability) — so coding time compresses.

### Code review as your main technical surface

Whatever your coding level, **code review is how a lead stays technically
connected without owning delivery** (Lesson 28 develops this). Reviewing keeps you
intimately aware of the codebase, the patterns, the quality, and the team's work —
so your technical judgment stays sharp and your decisions stay grounded — while
requiring no critical-path ownership. For many leads (especially at larger scope),
review is the *primary* technical activity, and it's high-leverage: it spreads
standards, grows people, and catches problems, all while keeping you credible.
The lead who reviews thoughtfully can stay technically excellent while coding very
little.

### Keeping depth without owning delivery

Beyond review, the ways to stay technically deep off the critical path (Lesson 3):
**spikes and prototypes** (exploring risky approaches, evaluating tools — deep
work that blocks nobody), **incident response** (jumping into real production
problems keeps you connected to how the system actually behaves and fails),
**pairing to teach** (deep technical engagement that also grows people), and
**tooling/DX work** (improving the team's technical foundation). These let you
keep your hands dirty and your judgment current while your delivery-critical time
goes to leadership. The disqualifier remains: *not the critical path* — the moment
your coding is what a deadline or a teammate waits on, you've re-entered the hero
trap.

### The credibility question

A real consideration: technical credibility matters for a technical leader — the
team follows your technical direction partly because they respect your judgment,
and design reviews, architecture decisions, and technical calls require you to
genuinely understand the work. This argues for staying technical *enough* to
remain credible and make good decisions — which is why "code ~0%" doesn't mean
"disengage from the technology" (that's the lost-touch trap). But credibility comes
from *demonstrated judgment* (good calls, insightful reviews, understanding the
system) more than from *volume of code shipped* — so you can maintain credibility
with relatively little hands-on coding as long as you stay genuinely engaged with
the technical substance through review, design, and staying informed. The mistake
is thinking credibility requires being the top committer; it requires being
technically sharp, which review and engagement sustain.

{: .note }
> **The level moves — adjust it deliberately</br>**
> Your right coding level isn't a fixed personal setting — it shifts with the
> situation, and part of the skill is adjusting it consciously. During a crunch or
> a critical incident, coding more (even on the critical path) can be right —
> temporarily. During a period of heavy hiring, planning, or team turmoil, coding
> drops toward zero because the people-work is where you're needed. A new, small
> team needs more of your hands than a mature, larger one. The failure is being on
> autopilot at either extreme — reflexively coding because it's comfortable
> (drifting to hero), or reflexively refusing because "I'm a manager now"
> (drifting to lost-touch). Read what the situation needs and set your level to
> match, revisiting it as things change.

---

## Lab — Scenario

**The situation:** Design your own weekly technical-involvement budget for three
different situations, and justify each:

1. You're the **tech lead of a 4-person team** building a new product. The team is
   small, the codebase is young, and you're the most experienced engineer.
2. You're the **EM of an 8-person team** on a mature product. You own the team's
   people (1:1s, careers, hiring) as well as delivery.
3. You're a **senior EM responsible for two teams** (~15 people total), with a
   tech lead reporting to you on each.

For each: roughly what % of your time is hands-on coding, what *kind* of coding
(critical-path delivery? review? spikes? incidents?), and how do you stay
technically credible?

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
The point is to show the level scaling down with scope, the <em>kind</em> of
technical work shifting (from delivery toward review/direction), and credibility
maintained differently at each level. A strong response:
<br><br>
<strong>1. Tech lead, 4-person team, new product:</strong> ~30–50% hands-on
coding. At this scope, your hands genuinely add throughput (the team is small, so
you're a meaningful fraction of its capacity), the codebase is young (so
foundational decisions you make in code have outsized influence and set patterns),
and staying deep is core to the tech-lead role. But — even here — be careful about
the <em>critical path</em>: take the risky/foundational/exploratory work (spikes,
proving the architecture, the hard core problems) and the glue/tooling, while
being ready to hand off delivery pieces if you're becoming a bottleneck (Lesson
3). A meaningful chunk should also be code review (staying across everyone's work)
and pairing (growing the newer engineers). Credibility here comes naturally from
being genuinely hands-on and setting the technical direction in code.
<br><br>
<strong>2. EM, 8-person team, mature product:</strong> ~10–20% hands-on, and
mostly <em>not</em> critical-path delivery. With 8 people to grow (1:1s, careers,
performance, hiring — real people-work), coordination, and planning, your leverage
has shifted toward people and direction, and being on the critical path across a
bigger team is riskier (more people wait on you, more meetings pull you away
mid-task). So the coding budget compresses and its <em>kind</em> changes:
primarily <strong>code review</strong> as your main technical surface (staying
across the codebase and the team's work, spreading standards, growing people
through review), plus occasional spikes/prototypes (exploring a risky approach off
the critical path) and incident response (staying connected to how the system
behaves). Little-to-no delivery ownership — you're not the person a feature waits
on. Credibility comes from thoughtful reviews, sound architecture/technical
decisions, and staying genuinely engaged with the technical substance — <em>not</em>
from shipping features yourself (the mistake would be trying to stay a top
committer while also managing 8 people — you'd do both badly, and either become a
bottleneck or neglect the people-work).
<br><br>
<strong>3. Senior EM, two teams (~15 people), tech leads reporting to you:</strong>
~0% critical-path coding, and quite little hands-on at all. Your leverage is now
almost entirely in people (managing managers/leads, careers, hiring, team health),
cross-team direction, stakeholder work, and organizational problems — and you have
tech leads whose <em>job</em> is the hands-on technical leadership of each team
(coding for you would undermine them and duplicate their role). Any coding is
exploratory/learning (a personal spike to stay current, a prototype to understand
a problem) or occasional deep-dive into a critical incident — never delivery.
Credibility at this level comes from <strong>demonstrated judgment</strong> (good
technical decisions, insightful questions in design reviews you attend, the ability
to reason about the technical trade-offs your teams face) and from staying
<em>informed</em> (reading designs, being present in technical discussions,
understanding your systems at the architecture level) rather than from hands-on
coding — a senior EM who tried to code delivery would be neglecting a 15-person
org's real needs and stepping on their tech leads. The risk here is the
<em>lost-touch</em> trap (drifting so far from the technology that decisions become
disconnected) — countered not by coding but by deliberately staying engaged with
the technical substance (design reviews, architecture discussions, incident
retrospectives, occasional deep-dives).
<br><br>
<strong>The pattern across all three:</strong> as scope grows (4 → 8 → 15 people),
hands-on coding drops (30–50% → 10–20% → ~0%), the <em>kind</em> of technical work
shifts from delivery toward review-and-direction, and credibility increasingly
comes from judgment-and-engagement rather than code-volume. The leverage math
drives it: at small scope your hands add real throughput and the opportunity cost
is low; at large scope your hands add little relative to the leverage work you'd
displace, the bottleneck risk is high, and you have leads whose role is the
hands-on work. And at <em>every</em> level, code review is the throughline that
keeps a lead technically connected without owning delivery — the single most
important technical activity for a leader. A good answer also notes the levels
<em>move with the situation</em> (all three would code more during a crisis, less
during heavy people-work) and that the failure at each level is autopilot at an
extreme (the tech lead who won't delegate delivery becomes the hero bottleneck;
the senior EM who keeps coding neglects the org and undermines their leads; the
EM who disengages technically drifts to lost-touch). Common mistakes: (1) giving
the same % for all three (missing that it scales with scope); (2) keeping
critical-path delivery at higher scopes (the bottleneck/opportunity-cost mistake);
(3) equating credibility with code volume (missing that judgment+engagement
sustain it); (4) at the senior-EM level, coding in a way that undermines the tech
leads whose job it is.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| How much managers should code | *The Manager's Path*, Camille Fournier |
| Technical credibility for leaders | *The Staff Engineer's Path*, Tanya Reilly |
| The tech lead role & coding balance | Pat Kua — "Talking with Tech Leads" |
| Staying technical as you scale | *An Elegant Puzzle*, Will Larson |
| The maker/manager coding tension | Lessons 3 & 4 (hero trap; leverage) |

---

## Checkpoint

**Q1.** Why does the right amount of coding for a lead decrease as their scope
(team size, number of teams) increases? Explain the leverage and risk math.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Two forces both push coding time down as scope grows: rising <em>opportunity cost</em>
and rising <em>bottleneck risk</em>. <strong>Opportunity cost (the leverage math):</strong>
a lead's coding competes for time with their leverage work (decisions, direction,
growing people, unblocking, cross-team work — Lesson 4). At small scope (a
4-person team), there's relatively little leverage work and your hands add a
meaningful fraction of the team's throughput, so an hour coding has a decent
return and a low opportunity cost. As scope grows (8 people, then 15 across two
teams), the leverage work multiplies (more people to grow, more coordination,
more direction, more stakeholders) <em>and</em> your coding adds a smaller fraction
of total throughput (you're one of 15 now, and there are tech leads for the
hands-on work) — so an hour you spend coding is an hour <em>not</em> spent on
increasingly valuable, increasingly abundant leverage work, and it produces
increasingly little relative to the team's total. The opportunity cost of your
coding rises steeply with scope, so the optimal coding time falls. <strong>Bottleneck
risk:</strong> when you're on the critical path (your coding is something others
wait on), you're a single point of failure and a bottleneck — and this gets worse
as more people depend on you. On a 4-person team, if you're pulled into a meeting
mid-task, a small blast radius; across 15 people, your unavailability (which grows
as your leadership demands grow — more meetings, more interruptions) blocks more
people and more work, and the probability you'll be pulled away mid-task is higher.
So critical-path coding becomes progressively more dangerous as scope grows — you
can't reliably deliver code when your calendar is fragmented by leading 15 people
(Lesson 4's maker-vs-manager schedule), and trying to means either you bottleneck
the team or you neglect the leadership. Together: the value of your coding drops
(opportunity cost up, marginal throughput contribution down) while its risk rises
(bottleneck impact up, reliability down) as scope increases — so the rational
coding budget compresses from ~30–50% (small team, tech lead) toward ~0% critical-
path (large scope, senior EM). The <em>kind</em> of technical work also shifts
accordingly: from delivery (which has the bottleneck risk) toward review and
direction (which don't) — so a large-scope leader stays technical through review
and judgment rather than through shipping code. The underlying principle is just
leverage (Grove/Lesson 4) applied to the coding decision: spend your scarce time
where it produces the most, and as scope grows, that's less and less on your own
hands at the keyboard and more and more on multiplying your growing team — with the
important caveat (Q2) that "less coding" must not become "disengaged from the
technology," which is the opposite failure.
</details>

**Q2.** Technical credibility matters for a technical leader — so doesn't coding
very little threaten it? Resolve the apparent tension: how does a lead stay
credible while coding little, and what would actually erode credibility?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The tension dissolves once you separate <em>credibility</em> from <em>code
volume</em> — they're correlated for individual contributors but decouple for
leaders. Credibility for a technical leader comes from <strong>demonstrated
judgment and genuine technical engagement</strong>, not from being the top
committer: the team respects and follows a lead's technical direction because the
lead makes good calls, asks insightful questions, understands the system deeply,
gives valuable design-review feedback, and reasons well about trade-offs — and
<em>all of that can be sustained with little hands-on coding</em>, as long as the
lead stays genuinely engaged with the technical substance. The mechanisms that
maintain credibility while coding little: (1) <strong>code review</strong> as the
primary technical surface (Lesson 28) — reviewing keeps the lead intimately across
the codebase, the patterns, and the team's work, so their judgment stays current
and grounded, and insightful reviews visibly demonstrate technical sharpness; (2)
<strong>design and architecture engagement</strong> — being genuinely present and
valuable in design reviews and architecture decisions, where the lead's judgment
shows; (3) <strong>staying informed</strong> — reading designs, following technical
discussions, understanding incidents and how the system behaves, so the lead can
reason about the real technical situation; (4) <strong>occasional deep-dives</strong>
— spikes, incident response, pairing, which keep the hands and instincts warm. A
lead doing these stays technically excellent and credible on a modest coding diet.
What <em>actually</em> erodes credibility is not low code volume but
<strong>genuine disengagement — the lost-touch trap</strong>: a leader who has
drifted so far from the technology that they no longer understand the work, can't
meaningfully review a design, makes decisions disconnected from reality (unrealistic
timelines, bad technical calls, unable to tell good work from bad), and talks about
the system in ways that reveal they don't really get it anymore. <em>That</em>
destroys credibility — and notably, it's caused by disengaging from the technical
<em>substance</em> (not attending design reviews, not reading the code, not
understanding incidents), not by coding little per se. A lead can code ~0% and stay
highly credible (if they review, engage, and understand deeply), or code a fair
amount and still lose credibility (if they're hands-on on trivia while missing the
big technical picture). So the resolution: coding little doesn't threaten
credibility <em>if</em> the lead stays technically engaged through review, design,
and understanding — because credibility rests on judgment and engagement, which
those sustain. The mistake is conflating the two and concluding "I must keep coding
a lot to stay credible" (which drags the lead toward the hero/bottleneck trap and
neglects leadership) — when the real requirement is to stay technically
<em>engaged and sharp</em>, which is a different and more sustainable thing than
staying a top committer. Guard against lost-touch by deliberately staying engaged
with the substance (review, design, incidents), not by reflexively coding more.
</details>

---

## Homework

Assess your own current (or anticipated) technical involvement. First, honestly
estimate your actual coding %, and what kind (critical-path delivery? review?
spikes? incidents?). Then, given your real role and team size, what <em>should</em>
it be — and which direction are you off (too much, drifting toward hero/bottleneck;
or too little/disengaged, drifting toward lost-touch)? Design your target: the %,
the kinds of technical work you'll prioritize (especially how you'll use code
review as your technical surface), and how you'll stay credible. Finally: what's
your plan to <em>adjust</em> the level as situations change (crunch, hiring, team
changes)?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise forces an honest self-assessment against the framework, and the value
is in identifying which direction you're off and designing a deliberate target
rather than being on autopilot. A strong response has an honest current estimate
(including the <em>kind</em> of coding — many leads discover an uncomfortable share
is critical-path delivery, which is the bottleneck risk), a target calibrated to
actual scope (using the scaling logic — a tech lead of a small team legitimately
codes more than an EM of a larger one), and a clear read on which trap they're
drifting toward. The two common findings: (1) <strong>too much, drifting toward
hero/bottleneck</strong> — you're coding on the critical path more than your scope
warrants, likely because it's comfortable and satisfying (Lessons 1/3), and you're
a bottleneck and/or neglecting leverage work; the fix is delegating delivery
(Lesson 3), shifting your technical time toward review/spikes (off the critical
path), and redirecting the reclaimed time to leadership. (2) <strong>too little/
disengaged, drifting toward lost-touch</strong> — you've pulled back from the
technology to the point where your technical judgment is getting stale and your
decisions less grounded; the fix is <em>not</em> to grab critical-path coding, but
to re-engage with the substance — do more thoughtful code review, attend and
contribute to design reviews, dig into incidents, occasionally spike — staying
technically sharp without owning delivery. The target design should center
<strong>code review as the technical throughline</strong> (the highest-leverage
way to stay connected at any scope) plus the off-critical-path deep work (spikes,
incidents, pairing) appropriate to your level, and a credibility plan based on
judgment-and-engagement rather than code-volume. The adjustment plan is the
sophisticated part: recognizing the level should <em>move</em> deliberately with
the situation (code more during a crunch or critical incident — temporarily; code
less during heavy hiring, planning, or team turmoil where people-work dominates;
more hands-on for a new small team, less as it matures) and committing to set it
consciously rather than autopilot at an extreme. The meta-outcome: you leave with
a <em>deliberate, scope-calibrated, situation-adjustable technical-involvement
strategy</em> instead of either reflexively coding because it's comfortable
(hero drift) or reflexively refusing because "I'm a manager now" (lost-touch
drift) — and with code review identified as the anchor that keeps you credible and
connected at whatever level you're at. If your self-assessment shows you're already
well-calibrated (level matched to scope, credibility via engagement not volume,
consciously adjusting) — good, that's the goal; the ongoing discipline is
resisting the drift (the pull back toward comfortable coding under stress, or the
slow slide toward disengagement as leadership demands grow) and re-checking your
level as your role and situations evolve. This closes Phase 2: you now have the
technical-leadership toolkit — vision/strategy, ADRs, trade-offs, design reviews,
standards, debt, and your own coding balance — and the throughline is that
technical leadership is about <em>judgment and multiplying the team's technical
work</em>, exercised increasingly through review, decisions, and direction rather
than through your own hands on the keyboard, calibrated to your scope and
adjusted to your situation.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 3 — Communication Foundations (Lesson 13: Audience-First Communication) →](lesson-13-audience-first){: .btn .btn-primary }
