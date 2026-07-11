---
title: "Lesson 38 — Aligning Teams Around a Direction"
nav_order: 4
parent: "Phase 7: Influence Without Authority"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 38: Aligning Teams Around a Direction

## Concept

Getting people to *agree* in a meeting is easy; getting them to stay **aligned** after they leave — so
they actually build compatible things toward a shared direction — is hard. Misalignment is silent and
expensive: teams nod along to "we're going multi-region," then each builds something different because
they interpreted it differently, and nobody notices until the incompatible pieces collide. The skill is
creating alignment that **survives after you leave the room** — primarily by **writing the direction
down** (the memo is the tool) and by detecting silent misalignment early.

```
   ALIGNMENT ≠ AGREEMENT
   ┌──────────────────────────────────────────────────────┐
   │ AGREEMENT: nodding in the meeting ("yes, multi-region")│
   │ ALIGNMENT: everyone building COMPATIBLE things toward   │
   │            the SAME understood direction                │
   │ → the gap: people "agree" but interpret it differently  │
   │   (silent misalignment) → incompatible work → collision │
   └──────────────────────────────────────────────────────┘
   TOOLS: write it down (the memo) · "say it back" test ·
          disagree-and-commit · re-alignment cadence
```

The reframe: **alignment isn't agreement in a meeting — it's a shared, durable understanding that
survives after the room, which you create by writing the direction down and checking for silent
misalignment.** Verbal agreement is cheap and evaporates (people forget, or "agreed" to different
interpretations); a written direction (a memo) creates a durable, shared, referenceable understanding —
and the "say it back" test surfaces the silent misalignment (different interpretations of the same
words) before it causes collisions.

---

## How It Works

### Alignment ≠ agreement

A critical distinction: **agreement** is people saying "yes" in a meeting; **alignment** is people
actually sharing the same understanding and building compatible things toward it, durably. They're not
the same — people can "agree" (nod along) while interpreting the direction completely differently (silent
misalignment), which surfaces later as incompatible work. So getting agreement in the room isn't the
goal; getting genuine, shared, durable alignment is — which requires more than a nod. The failure mode is
mistaking the easy agreement (nods) for the hard alignment (shared understanding that holds).

### Write the direction down — the memo is the tool

The single most powerful alignment tool: **write the direction down.** A clear written memo (the
direction, the reasoning, the key decisions, what's in/out of scope) creates alignment that verbal
discussion can't, because: (1) it's a **single shared reference** — everyone can point to the same words
(vs. everyone's fuzzy memory of a meeting); (2) it **forces precision** — writing it down surfaces the
ambiguities and disagreements that a verbal discussion glosses over (you have to actually specify what
"multi-region" means, which reveals people interpreted it differently); (3) it's **durable** — it
persists and can be referenced, so alignment doesn't decay as memories fade; and (4) it's **checkable** —
people can read it and confirm (or reveal) their understanding. The memo is the tool because it converts a
fuzzy verbal "agreement" into a precise, shared, durable, checkable direction.

### The "say it back" test — detecting silent misalignment

Silent misalignment (people "agree" but understand different things) is the core danger, and it's
detectable with the **"say it back" test**: ask people to articulate, in their own words, what the
direction means and what they'll do — "so what does this mean for your team? what are you going to build?"
When they say it back, the differences surface: you discover team A thinks "multi-region" means active-
active, team B thinks it means a DR backup, team C thinks it means just deploying to two regions. Those
divergent interpretations — invisible when everyone just nodded — become visible when they say it back, so
you can correct them <em>before</em> they build incompatible things. Never assume nodding = alignment;
test it by having people articulate their understanding.

### Disagree-and-commit — done honestly

Real alignment doesn't require everyone to <em>agree</em> — it requires everyone to <em>commit</em>.
**Disagree-and-commit** (Lesson from Phase 2 / English track): people voice their disagreement, a decision
is made, and then everyone commits to and executes it — even those who disagreed. Done honestly, this
means (1) people genuinely got to voice their dissent (and felt heard), (2) a clear decision was made, and
(3) everyone — including dissenters — genuinely gets behind it (not grudgingly sabotaging). This creates
alignment (everyone rowing the same direction) without requiring unanimous agreement (which is often
impossible). The "honestly" matters: fake disagree-and-commit (people say they'll commit but quietly
resist) isn't alignment — real alignment needs genuine commitment after genuine voice.

### Re-alignment cadence — alignment decays

Alignment isn't set-once — it **decays** over time (circumstances change, details drift, new people join,
memories fade), so it needs a **re-alignment cadence**: periodically revisiting the direction (is it still
right? are we still aligned? has drift crept in?) and re-establishing shared understanding. Especially for
long or evolving efforts, checking alignment regularly (not assuming it holds from the kickoff) catches
drift early. Alignment is maintained, not achieved once.

{: .note }
> **Alignment isn't agreement — it's a shared, durable understanding, created by writing it down and testing it</br>**
> Getting agreement (nods) in a meeting is easy; getting alignment (everyone building compatible things
> toward the same understood direction, durably) is hard — and the gap is silent misalignment (people
> "agree" but interpret differently), which surfaces later as incompatible work. Create durable alignment
> by <em>writing the direction down</em> (the memo forces precision, is a single shared reference, and is
> durable and checkable), and detect silent misalignment with the <em>"say it back" test</em> (have people
> articulate what it means for them — divergent interpretations surface before they cause collisions). Use
> honest disagree-and-commit (commitment, not unanimous agreement), and maintain a re-alignment cadence
> (alignment decays). The core: don't mistake nodding for alignment — build shared understanding that
> survives after the room, in writing, and keep verifying it.

---

## Lab — Scenario

**The situation:** Leadership has declared "we're going multi-region" (for resilience and lower latency),
and three teams are now working toward it. But you've noticed something alarming: they each seem to
interpret "multi-region" differently. Team A is building active-active (traffic served from both regions
simultaneously); Team B is building an active-passive disaster-recovery setup (one region normally, failover
to the other); Team C is just deploying their service to a second region with no data replication plan. They
all "agreed" to "go multi-region" — but they're building **incompatible** things, and if this continues,
the pieces won't fit together. You need to re-align them.

**Design the re-alignment** — how you'd surface and fix the misalignment. Then note the principles and
mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong re-alignment surfaces the divergent interpretations (say-it-back), then creates a precise shared
direction in writing that everyone confirms. Example:
<br><br>
<strong>First — surface the misalignment (the say-it-back test):</strong> "Get the teams together (or
1:1 first) and have each articulate what they're building and what they think 'multi-region' means for
them. This immediately surfaces the divergence: 'Team A, walk us through your approach' → active-active;
'Team B' → active-passive DR; 'Team C' → just a second deployment. Now the silent misalignment is visible
and undeniable — everyone sees the teams interpreted the same words three different ways and are building
incompatible things." <em>(The say-it-back test makes the silent misalignment visible — the essential first
step, because until it's surfaced, people think they're aligned.)</em>
<br><br>
<strong>Then — establish the actual direction, precisely, in writing:</strong> "The problem is 'multi-
region' was never actually specified — it's ambiguous, and each team filled the gap differently. So nail
down the real direction: What are we actually trying to achieve (resilience? latency? both)? What
architecture does that require (active-active? active-passive?)? What does each team need to build for it
to fit together? Get the deciders to make the actual architectural decision (it needs a real answer, not
'multi-region'), then <strong>write it down</strong>: a clear memo specifying the target architecture, what
'multi-region' concretely means, the data strategy, and what each team builds. This becomes the single
shared reference." <em>(Write the precise direction down — the memo forces the specificity that was missing
and gives everyone one reference.)</em>
<br><br>
<strong>Then — confirm alignment with say-it-back again:</strong> "Once the memo exists, have each team say
back what it means for <em>them</em> and what they'll build — confirming they now share the same
understanding, and catching any remaining divergence. Don't assume the memo alone aligns them — verify."
<em>(Re-test alignment after establishing the direction — nodding at the memo isn't enough; have them
articulate it.)</em>
<br><br>
<strong>Then — set a re-alignment cadence:</strong> "Given this is a big cross-team effort, set up periodic
check-ins (are we still aligned? has drift crept in? did new questions arise?) so alignment is maintained,
not assumed from this one fix."
<br><br>
<strong>Principles:</strong> (1) <strong>Surface the misalignment first (say-it-back)</strong> — having
each team articulate their interpretation makes the silent misalignment visible and undeniable (until then,
everyone thinks they agree). (2) <strong>The direction was under-specified</strong> — "multi-region" was
ambiguous, so teams filled the gap differently; the fix is precision (what it actually means
architecturally). (3) <strong>Write it down</strong> — a precise memo forces the specificity and creates
the single shared reference that verbal agreement lacked. (4) <strong>A real decision is needed</strong> —
someone has to actually decide the architecture (not leave "multi-region" ambiguous). (5) <strong>Re-test
alignment</strong> — say-it-back again after the memo to confirm shared understanding. (6) <strong>Re-
alignment cadence</strong> — maintain alignment over the effort. <strong>Mistakes to avoid:</strong> (1)
<strong>assuming they're aligned because they all 'agreed'</strong> — the whole problem is that agreement ≠
alignment; the divergence was silent; (2) <strong>not surfacing the interpretations</strong> — without the
say-it-back, you don't see the misalignment clearly (or teams keep building incompatible things); (3)
<strong>re-stating 'go multi-region' without specifying it</strong> — repeating the ambiguous direction
just reproduces the divergent interpretations; you must make it precise; (4) <strong>not writing it
down</strong> — a verbal re-alignment decays and stays fuzzy; the memo is the tool; (5) <strong>not
confirming with say-it-back after</strong> — assuming the memo aligned everyone (test it); (6) <strong>one-
time fix</strong> — not setting a cadence, so drift returns; (7) <strong>blaming the teams</strong> — the
misalignment came from an under-specified direction (a leadership/communication failure), not the teams
being dumb — frame it as clarifying, not blaming.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Writing to align (the memo) | Amazon's memo culture; *Working Backwards*, Bryar & Carr |
| Alignment & disagree-and-commit | *The Making of a Manager*, Zhuo; Amazon leadership principles |
| Written direction & docs | Lesson 16 (design docs); English track Lesson 38 (doc structure) |
| Detecting misalignment | *Crucial Conversations* (mutual understanding) |

---

## Checkpoint

**Q1.** Why is "alignment ≠ agreement," and why is writing the direction down (the memo) such a powerful
alignment tool?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
"Alignment ≠ agreement" because <strong>agreement is people saying "yes" in a meeting, while alignment is
people actually sharing the same understanding and building compatible things toward the direction, durably
— and people can agree (nod) while interpreting the direction completely differently (silent
misalignment)</strong>. The two are distinct: agreement is a verbal/social act (nodding along, saying "yes,
let's go multi-region"), which is easy to get; alignment is a genuine shared understanding that holds after
the meeting and translates into compatible action. The dangerous gap between them is <strong>silent
misalignment</strong>: people "agree" to a direction while each understanding it differently — they all nod
at "we're going multi-region," but one thinks active-active, another thinks DR failover, another thinks just
a second deployment. They genuinely believe they agree (and did, verbally), but they're aligned on the
<em>words</em>, not the <em>meaning</em> — so they go off and build incompatible things, and the misalignment
only surfaces later when the pieces collide. This is why getting agreement in the room isn't the goal (it's
cheap and can mask misalignment); getting genuine, shared, durable alignment is. Mistaking agreement (nods)
for alignment (shared understanding) is the core failure — the nods feel like success but hide the divergent
interpretations that cause expensive collisions later. Why writing the direction down (the memo) is such a
powerful alignment tool: <strong>because writing forces the precision that surfaces ambiguity and
disagreement, creates a single shared durable reference, and is checkable — converting a fuzzy verbal
"agreement" into a precise, shared, lasting, verifiable direction</strong>. Specifically: (1) <strong>it
forces precision</strong> — to write the direction down, you have to actually specify what it means (what
"multi-region" concretely is — active-active or DR, the data strategy, what each team builds), and this
act of specifying surfaces the ambiguities and divergent interpretations that a verbal discussion glosses
over (in conversation, "multi-region" can stay comfortably vague, with everyone filling the gap
differently; in writing, you must pin it down, which exposes the differences). So writing it down catches
the silent misalignment by forcing the specificity that reveals it. (2) <strong>it creates a single shared
reference</strong> — everyone can point to the same words (the memo), rather than relying on their own
fuzzy, diverging memories of a meeting; there's one authoritative statement of the direction, so people are
aligned on the same actual text, not their individual recollections. (3) <strong>it's durable</strong> — a
written direction persists and can be referenced over time, so alignment doesn't decay as memories fade
(verbal agreement evaporates; a memo remains). (4) <strong>it's checkable</strong> — people can read it and
confirm (or reveal) their understanding against it, enabling verification of alignment (and combined with
the say-it-back test, surfacing remaining divergence). So the memo is powerful because it addresses exactly
what verbal agreement lacks: precision (forcing specification that surfaces hidden differences), a shared
reference (one text vs. many memories), durability (persists vs. fades), and checkability (verifiable vs.
assumed). It converts the fuzzy, ephemeral, individually-interpreted verbal "agreement" into a precise,
shared, lasting, verifiable direction — which is what actual alignment requires. This is why "the memo is
the tool" for alignment: writing the direction down is the single most effective way to create the shared,
durable understanding that survives after the room, rather than the cheap agreement that masks silent
misalignment.
</details>

**Q2.** What is the "say it back" test and why does it catch silent misalignment, and why does alignment
need a re-alignment cadence?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The <strong>"say it back" test</strong> is <strong>asking people to articulate, in their own words, what
the direction means and what they'll do about it — "so what does this mean for your team? what are you
going to build?"</strong> — rather than just accepting their nod of agreement. It catches silent
misalignment because <strong>when people put their understanding into their own words, the differences in
interpretation become visible — whereas a nod hides them</strong>. The problem with silent misalignment is
that it's <em>silent</em>: everyone agrees verbally (nods) while interpreting the direction differently, and
because they all think they agree, the divergence is invisible — nobody realizes they understand different
things. A nod conveys "yes" but not <em>what</em> they think they're agreeing to, so divergent
interpretations pass undetected. The say-it-back test surfaces them by making people articulate their actual
understanding: when Team A says "so we'll build active-active," Team B says "we'll build DR failover," and
Team C says "we'll deploy to a second region," the divergence is immediately visible and undeniable — the
different interpretations that were hidden behind identical nods come out when each person expresses what
the direction means to <em>them</em>. This lets you catch and correct the misalignment <em>before</em> people
act on their divergent understandings (before they build incompatible things), rather than discovering it
later when the pieces collide. The principle: <strong>never assume nodding = alignment; verify by having
people articulate their understanding</strong>, because only when people say it back in their own words do
the interpretation differences surface. It's a cheap, powerful check that converts silent (invisible)
misalignment into visible (correctable) misalignment. Why alignment needs a <strong>re-alignment cadence</strong>:
<strong>because alignment decays over time — circumstances change, details drift, new people join, memories
fade, and new questions arise — so a one-time alignment doesn't hold, and it must be periodically
re-established</strong>. Alignment isn't a set-once achievement; it's a state that erodes: (1)
<strong>circumstances and details change</strong> — as work progresses, new decisions and situations arise
that the original direction didn't cover, and teams resolve them independently (potentially divergently),
so drift creeps in; (2) <strong>memories fade and interpretations drift</strong> — even a clear initial
alignment fades from memory over time, and people's understanding subtly drifts; (3) <strong>new people
join</strong> — who weren't part of the original alignment and may not share the understanding; (4)
<strong>the direction itself may need updating</strong> — as things change, the original direction may need
to evolve. So without periodic re-alignment, the alignment established at kickoff decays, and teams
gradually drift out of sync (often silently) again. A re-alignment cadence — periodically revisiting the
direction (is it still right? are we still aligned? has drift crept in? any new divergence?) and
re-establishing shared understanding (with the memo updated and say-it-back re-tested) — catches this drift
early and maintains alignment over the life of the effort. This is especially important for long or evolving
efforts (where there's more time for drift). The principle: <strong>alignment is maintained, not achieved
once</strong> — it decays, so it needs ongoing attention (a cadence), not a one-time kickoff. Both the
say-it-back test and the re-alignment cadence reflect the core insight that alignment is fragile and must be
actively verified and maintained: silent misalignment lurks behind agreement (surfaced by say-it-back), and
alignment decays over time (maintained by a cadence) — so a lead can't assume alignment from a nod or a
kickoff, but must continually test and re-establish the shared understanding that keeps teams building
compatible things toward the same direction. Together with writing the direction down (the memo), these
tools create alignment that survives after the room and holds over time — the durable, shared understanding
that agreement alone can't provide.
</details>

---

## Homework

Assess alignment on something your teams are working toward. (1) Is there genuine alignment (shared
understanding, compatible work) or just agreement (nods that may hide divergent interpretations)? (2) Apply
the say-it-back test — ask a few people to articulate, in their own words, what a direction means and what
they'll do; do their interpretations match, or is there silent misalignment? (3) Is the direction written
down (a memo) precisely, or is it a fuzzy verbal thing people interpret differently? Write or improve one. (4)
Is there a re-alignment cadence, or was alignment assumed from a one-time kickoff? Reflect: where might silent
misalignment be lurking on your teams, and what would writing it down and testing it reveal?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds alignment skill — creating shared understanding that survives after the room. A strong
response distinguishes genuine alignment from mere agreement, applies the say-it-back test to detect silent
misalignment, checks whether the direction is written down precisely (and improves it), and considers a
re-alignment cadence. The realizations: (1) <strong>alignment ≠ agreement</strong> — nods in a meeting are
cheap and can hide divergent interpretations (silent misalignment), which surface later as incompatible work;
real alignment is shared understanding and compatible action, durably; (2) <strong>write the direction down</strong>
— the memo forces the precision that surfaces ambiguity, creates a single shared reference, and is durable and
checkable, converting fuzzy verbal agreement into a precise shared direction; (3) <strong>the say-it-back test
catches silent misalignment</strong> — having people articulate what the direction means for them surfaces the
divergent interpretations that nods hide, before they cause collisions; (4) <strong>alignment decays</strong> —
it needs a re-alignment cadence (circumstances change, details drift, people join, memories fade), so it's
maintained, not achieved once. On the assessment, people commonly discover silent misalignment they hadn't
seen — teams or people interpreting a shared direction differently behind identical agreement — and that their
directions are often fuzzy/verbal rather than written down precisely. The highest-value change is usually:
write directions down precisely (the memo) and use the say-it-back test to verify shared understanding, rather
than assuming nods mean alignment. The meta-point: getting agreement is easy, getting durable alignment is
hard, and the gap (silent misalignment) is expensive — teams building incompatible things because they
interpreted the same words differently. Creating alignment that survives the room requires writing the
direction down (forcing precision, creating a shared durable reference), testing understanding (say-it-back
surfaces silent divergence), using honest disagree-and-commit (commitment without requiring unanimity), and
maintaining a re-alignment cadence (alignment decays). Don't mistake nodding for alignment — build and verify
shared understanding. This is a key influence-without-authority skill (aligning teams you don't control), and
it builds on writing (Lesson 16, English track docs) and disagree-and-commit. The last Phase 7 lesson scales
up to the hardest version — driving change across an entire organization, bigger than any team you control.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 39 — Driving Change Across an Organization →](lesson-39-driving-change){: .btn .btn-primary }
