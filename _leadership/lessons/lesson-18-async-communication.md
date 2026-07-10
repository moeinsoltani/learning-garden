---
title: "Lesson 18 — Async and Written Communication"
nav_order: 6
parent: "Phase 3: Communication Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 18: Async and Written Communication

## Concept

Increasingly, leadership happens through text — Slack, docs, code review comments,
written updates. A distributed, async-heavy world means much of a lead's influence,
alignment, and decision-making flows through written channels, and doing it well is
a distinct skill from talking. The stakes are high because text has a property that
speech doesn't: **tone is far easier to misread**, and a message that would be fine
spoken can land as cold, harsh, or dismissive in writing.

```
   WRITTEN/ASYNC LEADERSHIP — the surfaces:
     Slack messages  ·  design docs  ·  code review comments
     ·  written decisions  ·  status updates  ·  announcements

   THE TEXT-TONE PROBLEM:
     spoken: tone, warmth, and intent carried by voice + face
     written: NONE of that — the reader supplies the tone,
              and under ambiguity they often supply a WORSE one
        → over-signal warmth; write decisions where they'll be found
```

Two ideas anchor written leadership. **When async beats sync** (and vice versa) —
knowing which channel a communication belongs in is itself a skill: async (written,
read-when-convenient) is better for information transfer, thoughtful input, and
respecting focus (Lesson 4/15); sync (meeting, call) is better for real-time
discussion, sensitive/emotional matters, and building trust. **Tone in text
requires deliberate warmth** — because text strips the warmth that voice and face
carry, and readers under ambiguity often assume the worse interpretation, a lead
must *over-signal* warmth and clarity in writing to land as they intend. This
pairs directly with the [English for Work]({{ '/english/learning-plan.html' | relative_url }})
track (Phase 3: Tone & Warmth, Phase 4: Slack), which teaches the *language*; this
lesson is the leadership *judgment*.

---

## How It Works

### When async beats sync

Match the channel to the communication: **async** (written) is right for
information transfer (updates, FYIs, documentation), things that benefit from
thoughtful composition and thoughtful response (a considered decision, complex
input), things that should be findable later (decisions, context — write them where
they'll be searched for), and anything that doesn't need real-time back-and-forth
(most things — the meeting-vs-async judgment of Lesson 15). **Sync** (meeting/call)
is right for real-time discussion and rapid iteration (a live decision needing
back-and-forth, brainstorming), sensitive or emotional matters (feedback, conflict,
bad news — where tone and reading the person matter, and text's tone-ambiguity is
dangerous — Phase 4), building relationships and trust (which text does poorly),
and situations where the nuance and immediacy of live interaction genuinely help.
The default should lean async (it respects focus and scales), reserving sync for
what genuinely benefits from it.

### Tone in text: over-signal warmth

The core hazard: text carries no vocal tone or facial warmth, so the reader
supplies the tone — and under ambiguity, people tend to read neutral text as
slightly negative and terse text as cold or annoyed (the brain's negativity bias).
A message you'd say warmly ("can you take a look at this?") can read as a curt
command in text. So a lead must *deliberately over-signal* warmth and good intent
in writing: a bit more warmth than feels necessary (because it'll read as less than
you put in), softeners and human touches where appropriate (Phase 3 of English
track), explicit positive framing, and care especially in feedback and disagreement
(where cold text does real damage). This isn't fluff — it's compensating for the
information (tone) that the medium strips, so your message lands as you intend
rather than colder. The rule: in text, add warmth beyond what you'd need in speech,
because the medium subtracts it.

### Write decisions and context where they'll be found

A specific high-value practice: when a decision is made or important context
established (in a meeting, a thread, a call), *write it down where people will find
it* — because knowledge that lives only in someone's head, a call, or a buried
Slack thread effectively doesn't exist for the team (it can't be referenced, gets
forgotten, and has to be re-derived). The lead who documents decisions (the ADR-
style record, Lesson 7), writes down the "why" behind context, and puts these in
findable places (the repo, a docs system, a pinned channel — not a random DM)
creates durable, referenceable alignment. This is especially important async and
at scale: written, findable decisions are how a distributed team stays aligned
without everyone being in every conversation.

### Landing a decision in a runaway thread

A common async leadership situation: a decision thread going in circles — 40
messages, competing views, no convergence, everyone slightly frustrated. The lead's
job is to *land it*: summarize the discussion fairly (so everyone feels heard —
"here's what I'm hearing: [the positions and their merits]"), state the decision
clearly with reasoning ("we're going to do X, because [the deciding factors,
acknowledging the trade-offs]"), and close it (with next steps and an owner —
disagree-and-commit, Lesson 7). The written decision-landing message is a distinct
skill: it must be fair (dissenters feel heard), clear (the decision and reasoning
unambiguous), and firm (it ends the debate — the thread is closed) — all in a tone
that's warm enough not to feel like a heavy-handed shutdown.

{: .note }
> **Response-time norms and the after-hours message</br>**
> Two async-etiquette points that matter for leads. (1) <em>Set and honor
> response-time norms</em> — teams work better when it's clear that not everything
> needs an instant reply (async means read-when-convenient); a lead who expects or
> models instant responses destroys focus and creates always-on pressure (Lesson
> 4). (2) <em>The after-hours message</em> — when you (as the lead) send messages
> at night or on weekends, even without expecting a response, it signals an
> always-on expectation and pressures people to be on; use scheduled-send to
> deliver during work hours, so your off-hours working doesn't leak into an
> off-hours expectation for the team. A lead's communication habits set the team's
> norms, so model the boundaries you want (English track Lesson 25 develops async
> etiquette).

---

## Lab — Scenario

**The situation:** A decision thread in your team's Slack channel has gone in
circles. The question: should the team adopt a new testing framework (migrating
from the current one). Over 40 messages across two days, three engineers have
argued strongly for the new framework (better DX, more features), two have pushed
back (migration cost, the current one works fine), and it's become a bit heated —
some messages are getting terse, people are repeating themselves, and no decision
is emerging. As the lead, you need to land this.

**Write the Slack message you'd post to land the decision.** (Assume that, weighing
the input, you've concluded the migration isn't worth it right now — the current
framework works, and the migration cost outweighs the DX benefits given other
priorities — but reasonable people could disagree.) Then note what your message
does and why.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong decision-landing message summarizes fairly (everyone heard), states the
decision and reasoning clearly, closes the debate firmly, and does it warmly — all
in text, where tone must be over-signaled. Example:
<br><br>
"Thanks everyone for the thoughtful discussion on this — I've read through the whole
thread and want to land a decision so we can move forward.
<br><br>
Here's what I'm hearing: there's a real case for [new framework] — [names] make
good points about the better developer experience and the features we'd gain, and I
don't doubt those benefits are real. On the other side, [names] have raised the
migration cost and the fact that our current setup is working, which are also fair.
So this is a genuine trade-off with reasonable arguments on both sides, not a case
where one side is obviously right.
<br><br>
Weighing it, my call is that <strong>we're going to stay on [current framework] for
now</strong> — here's my reasoning: the migration would be a meaningful investment
(rewriting a lot of existing tests + the team ramp), and given our current
priorities [the roadmap pressure / the other work], I don't think the DX gains
justify that cost right now. The current framework is working and not blocking us.
This isn't a 'no forever' — if the migration cost comes down (better tooling), the
DX pain gets worse, or our priorities shift so we have room for infrastructure
investment, we should revisit. I'd suggest we note the specific pain points people
are hitting with the current one, so if we do revisit, we have the real evidence.
<br><br>
I know a few of you were hoping to migrate, and I genuinely appreciate the case you
made — this was a close call and your arguments moved my thinking even though I
landed the other way this time. Let's put this one to rest and keep the energy for
[the priorities that matter most right now]. Thanks all. 🙏"
<br><br>
<strong>What this does and why:</strong> (1) <strong>Summarizes fairly</strong> —
it restates both sides' positions with their genuine merits and names the people,
so <em>everyone feels heard</em> (especially the pro-migration side who "lost" —
they see their case was understood and valued, not dismissed); this is essential
for disagree-and-commit (Lesson 7) — people commit to a decision they lost if they
felt heard, and resist one where they didn't. (2) <strong>States the decision
clearly with reasoning</strong> — the decision is unambiguous (bolded, direct: stay
on current) and the reasoning is explicit (migration cost vs DX benefit given
priorities), so it's a <em>reasoned call</em>, not a decree, and people understand
<em>why</em> even if they disagree. (3) <strong>Acknowledges it's a genuine
trade-off</strong> — "reasonable arguments on both sides," "close call," "your
arguments moved my thinking" — this validates the dissenters (their view wasn't
wrong, just outweighed this time) and shows the decision was made thoughtfully, not
arbitrarily or by fiat. (4) <strong>Closes the debate firmly but warmly</strong> —
"let's put this one to rest" ends the thread (the decision is made, stop
relitigating — disagree-and-commit), but the warmth ("appreciate the case you
made," the 🙏) means it lands as a considered close, not a heavy-handed shutdown. (5)
<strong>Leaves a door open with a condition</strong> — "not a no forever, revisit if
[specific triggers]" — so the dissenters don't feel permanently overruled, and it's
honest (the decision is "not now," not "never"), plus the suggestion to note pain
points gives them a constructive path. (6) <strong>Over-signals warmth
throughout</strong> — because it's text (where tone is stripped and a terse decision
would read as cold/authoritarian), the message is deliberately warm (thanks,
appreciation, acknowledging the close call, the emoji) to land as a fair, human
close rather than a cold ruling; a curt "we're staying on the current framework,
decision made" would be technically the same decision but would feel dismissive and
breed resentment. It embodies the lesson's core: land the decision (summarize fair,
decide clear, close firm) in text with deliberately over-signaled warmth. Common
mistakes: (1) just stating the decision without summarizing the input (dissenters
feel unheard → resentment, not commitment); (2) not giving the reasoning (a decree,
distrusted); (3) landing it too coldly/curtly (text strips tone — a terse decision
reads as authoritarian even if the decision is right); (4) not actually closing it
(leaving it open perpetuates the circular thread); (5) being so warm/hedged that the
decision isn't clear (must be firm — the point is to <em>land</em> it); (6) framing
it as one side being wrong rather than a genuine trade-off (alienates the dissenters
unnecessarily when reasonable people could disagree).
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Written & async communication craft | English for Work track, Phase 3 (Tone) & Phase 4 (Slack) |
| Async-first culture | GitLab handbook; *Remote*, Fried & Hansson |
| Tone in text / negativity bias | *The Culture Map*, Erin Meyer (communication across contexts) |
| Writing decisions down (ADRs) | Lesson 7; Lesson 16 (design docs) |
| Leading distributed/remote teams | LeadDev — <https://leaddev.com/remote> |

---

## Checkpoint

**Q1.** Why does tone require *more* deliberate attention in written communication
than in speech, and what's the practical rule for a lead writing to their team?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Tone requires more deliberate attention in writing because the medium <em>strips
the channels that carry tone in speech</em> — in a conversation, your intent and
warmth are conveyed by vocal tone, pace, facial expression, body language, and the
immediate back-and-forth that lets you catch and correct a misread; in text, all of
that is gone, and the reader has only the words. This matters because of a specific
asymmetry: <strong>when tone is ambiguous, readers tend to supply a more negative
interpretation than intended</strong> (a negativity bias — neutral text reads as
slightly cool, terse text reads as cold or annoyed, a direct request reads as a
curt command). So a message you'd deliver warmly in person ("hey, can you take a
look at this when you get a chance?") can, stripped to text and read by someone
having a bad day or primed to worry, land as a cold demand — the same words, but the
reader supplied a worse tone than you meant because the medium gave them nothing to
go on and their brain filled the gap negatively. The consequence: written
communication that would be perfectly fine spoken can damage relationships, create
resentment, or cause conflict, entirely through misread tone that neither party
intended. This is especially dangerous for the high-stakes written communication a
lead does — feedback, disagreement, decisions, requests — where a cold-reading text
can hurt. The practical rule: <strong>in text, deliberately over-signal warmth —
add more warmth than feels necessary, because the medium will subtract it</strong>.
Concretely: include the warmth markers that voice would otherwise carry (a friendly
opener, appreciation, acknowledgment of the person, softeners where appropriate —
English track Phase 3), frame positively, take extra care in feedback and
disagreement (where cold text does the most damage — over-signal that you're on
their side), and re-read your message asking "how would this land to someone
reading it uncharitably, or having a rough day?" and adjust. It's not that you
should be saccharine or hedge everything into mush — it's that you should
consciously compensate for the tone the medium strips, so your message lands as
warm-and-clear as you intend rather than colder. The mental model: speech comes
with warmth included; text comes with warmth removed, so you have to add it back
explicitly. A lead who writes with the same terseness they'd use in speech (relying
on tone that isn't there) will repeatedly land colder than intended and slowly erode
relationships through a thousand slightly-cold messages; a lead who over-signals
warmth in text lands as they intend and builds rather than erodes. And because a
lead's written communication is high-volume and high-stakes (much of leadership is
now text — this lesson's premise), getting this right compounds enormously — every
message that lands warm-and-clear rather than cold-and-curt is a small deposit in
the relationship, and every misread-cold message is a withdrawal, so the deliberate
warmth pays off across everything. (The English for Work track's Phase 3 develops
the specific <em>language</em> of warmth in text; this is the leadership judgment of
<em>why</em> and <em>when</em> it matters.)
</details>

**Q2.** Give three situations where async (written) communication is clearly
better, and three where sync (a meeting or call) is clearly better — and explain
the principle that distinguishes them.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Async is clearly better for:</strong> (1) <strong>Information transfer /
status updates</strong> — an FYI, a status report, documentation: there's no need
for real-time interaction, people can read it when convenient (respecting focus —
Lesson 4/15), and written is often better (skimmable, searchable, findable later).
(2) <strong>Things that benefit from thoughtful composition and response</strong> —
a complex decision or design that people should consider carefully and respond to
thoughtfully: async gives everyone time to reason (vs a meeting where the fastest
talker dominates and considered thinkers can't formulate), and produces a written
record. (3) <strong>Decisions and context that should be findable later</strong> —
writing a decision down where it'll be searched for (the ADR-style record) creates
durable, referenceable alignment that a verbal decision in a call doesn't. (Also:
anything that doesn't need real-time back-and-forth, which is most things — the
default should lean async.) <strong>Sync is clearly better for:</strong> (1)
<strong>Sensitive or emotional matters</strong> — feedback, difficult
conversations, conflict, bad news: text's tone-ambiguity (Q1) is dangerous here,
and you need to read the person's reaction and adjust in real time, convey genuine
warmth (which text strips), and handle emotion — all of which require the live
channel (Phase 4's difficult conversations should almost never be text). (2)
<strong>Real-time discussion / rapid iteration</strong> — a decision needing live
back-and-forth, a brainstorm, working through a complex problem together where the
immediacy and rapid exchange genuinely help (async would be too slow, with each
round taking hours). (3) <strong>Building relationships and trust</strong> — trust,
rapport, and connection are built through live human interaction (a 1:1, a
conversation), which text does poorly; a relationship maintained only through text
stays transactional. The distinguishing principle: <strong>match the channel to
what the communication actually requires — specifically, whether it needs the
things sync provides (real-time interaction, tone/warmth/reading-the-person,
relationship-building, rapid iteration) or benefits from what async provides
(thoughtful composition and response, respecting focus, findability, scale)</strong>.
Sync's costs are real (everyone's time and focus — Lesson 15's meeting cost), so
you reserve it for what genuinely needs it: real-time exchange, emotional/sensitive
nuance, trust-building — the things that lose too much in text or asynchrony.
Async's costs are its lack of real-time nuance and immediacy, so you avoid it for
what needs those: sensitive/emotional matters (where tone and reading the person
matter and text's ambiguity is dangerous), and fast iterative discussion. The
default should lean async (it respects focus, scales, and produces records — and
most communication doesn't need synchrony), with sync reserved for the specific
cases that genuinely benefit from live interaction — which is the same
meeting-vs-async judgment as Lesson 15, applied to the channel choice for any
communication. The common failures are both directions: defaulting to meetings for
things that should be async (wasting everyone's time and focus on information
transfer that a written update would serve — Lesson 15's over-meeting), and using
text for things that need sync (delivering hard feedback or handling conflict over
Slack, where the tone-ambiguity and inability to read the person cause damage — a
serious mistake). The lead's skill is the deliberate channel choice: for each
communication, ask "does this need real-time interaction, emotional nuance, or
relationship-building (→ sync), or is it information/composition/findability that
async serves better (→ async)?" — and choose accordingly, defaulting async but
never forcing async onto what genuinely needs the live channel (the sensitive and
the relational).
</details>

---

## Homework

Review your recent written/async communication as a lead (or in your current role):
Slack messages, docs, code review comments, decisions. Assess: (1) Tone — would any
of these read colder than you intended, given text strips warmth? Find one and
rewrite it with over-signaled warmth. (2) Channel choice — did anything go async
that needed sync (a sensitive matter over text) or sync that should've been async
(a meeting for information transfer)? (3) Findability — are your team's decisions
and context written down where they'll be found, or trapped in threads/heads/calls?
Then practice the decision-landing skill: find a circular thread (past or
hypothetical) and write the message that lands it. What patterns do you notice in
your written-leadership habits, and what would you change?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds self-awareness about written-leadership habits, which most
people never examine even though it's now a huge part of the job. A strong response
honestly audits across the three dimensions and usually finds: (1) <strong>Tone</strong>
— several messages that, re-read uncharitably, would land colder or more curt than
intended (the default engineer/lead terseness that relies on tone the medium
strips) — and the rewrite with over-signaled warmth demonstrates the compensation
skill (adding the warmth markers, framing, and human touches that make text land as
intended). The realization: "I write efficiently/directly, which feels neutral to
me but probably reads as cold to others, especially in feedback and requests — I
need to deliberately add warmth I'd normally rely on my tone to carry." (2)
<strong>Channel choice</strong> — often a case where something sensitive went over
text that should have been a conversation (feedback, a hard message, a conflict —
where the tone-ambiguity caused or risked damage), and/or meetings held for pure
information transfer that should've been async (Lesson 15's over-meeting) — building
the judgment to match channel to communication. (3) <strong>Findability</strong> —
frequently decisions and context that live only in threads, calls, or the lead's
head, not written where they'll be found — so the team can't reference them, they
get forgotten and re-derived, and alignment is fragile; the fix is the habit of
writing decisions down in findable places (the ADR practice, Lesson 7). The
decision-landing practice builds a specific high-value skill (summarize fair,
decide clear, close firm, warmly) that leads need often in async-heavy teams. The
patterns people commonly notice in their written-leadership habits: a default
terseness that reads colder than intended (the biggest and most common — most
technical people write efficiently in a way that lands cold, and don't realize it);
under-documenting decisions (relying on memory and threads rather than findable
records); and sometimes a reluctance to firmly close circular discussions (letting
threads run rather than landing them). The changes that follow: deliberately
over-signaling warmth in text (especially feedback/requests/disagreement), choosing
channels deliberately (sync for sensitive/relational, async for
information/composition), documenting decisions in findable places, and developing
the decision-landing skill. The meta-insight: <strong>written and async
communication is now a primary leadership surface — much of a lead's influence,
alignment, feedback, and decision-making flows through text — and it's a distinct
skill from speaking that most leads never deliberately develop</strong>, so
examining and improving it (tone, channel choice, findability, decision-landing)
is high-leverage. The specific hazard that makes it worth deliberate attention is
text's tone-stripping (Q1): because the medium subtracts warmth and readers supply
worse tone under ambiguity, a lead who writes as they'd speak lands repeatedly
colder than intended and slowly erodes relationships through accumulated
slightly-cold messages — so the discipline of over-signaling warmth, choosing
channels well, and writing decisions down where they're found isn't optional
polish, it's core to leading effectively in a text-heavy world. This closes Phase
3's communication foundations; the [English for Work]({{ '/english/learning-plan.html' | relative_url }})
track develops the language craft (tone, Slack, docs, feedback wording) that makes
this leadership judgment executable in actual words — the two tracks together give
you both the <em>judgment</em> of what to communicate and the <em>language</em> to
do it warmly and clearly. If your audit reveals your written communication is
already warm, well-channeled, and well-documented — good, that's a real and
uncommon strength; the ongoing practice is maintaining it (the terseness-drift under
time pressure is constant) and modeling the norms (response-time expectations, the
after-hours boundary) that shape the whole team's async culture.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 4 — Feedback & Difficult Conversations (Lesson 19: Feedback That Lands — SBI) →](lesson-19-sbi-feedback){: .btn .btn-primary }
