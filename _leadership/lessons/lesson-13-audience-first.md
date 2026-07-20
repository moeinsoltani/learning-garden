---
title: "Lesson 13 — Audience-First Communication"
nav_order: 1
parent: "Phase 3: Communication Foundations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 13: Audience-First Communication

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **BLUF** — "Bottom Line Up Front": state your conclusion first, support it after.
> - **sender-first / audience-first** — organizing a message around what *you* want to say vs what the *receiver* needs to hear.
> - **curse of knowledge** — once you know something deeply, you can't remember what not-knowing felt like, so you skip the context others need.
> - **triage** (TREE-ahzh) — quickly sorting by importance before acting; here: audience → purpose → medium.
> - **medium** — the channel a message travels in (Slack, email, doc, meeting).
> - **calibrate** — to adjust to fit ("calibrate detail to the audience").
> - **takeaway** — the one thing the audience should remember.
> - **jargon** — specialist vocabulary outsiders don't share.
> - **spin** — dishonest framing that misleads; audience-shaping with consistent facts is *not* spin.
> - **executive summary** — the few lines at the top that carry the whole message for busy readers.

## Concept

The single most common communication mistake — and the one that separates
effective communicators from ineffective ones — is communicating from the
*sender's* perspective instead of the *receiver's*. You know what you want to
say; the skill is shaping it around what the audience needs to hear, in a form
they can absorb.

```
   SENDER-FIRST (the default, and the mistake)
   "here is everything I know / did / think, in the order I think it"
        → the audience has to work to extract what matters TO THEM
        → often tunes out, misunderstands, or misses the point

   AUDIENCE-FIRST (the skill)
   "what does THIS audience need, care about, and already know —
    and what's the ONE thing they should take away?"
        → shaped for the receiver; lands; gets the response you wanted
```

The same information gets communicated completely differently depending on who's
receiving it. A production incident is described one way to your engineers (root
cause, technical detail, what to fix), another way to your director (impact,
status, what you need from them), another way to the affected customer (what
happened in their terms, what you're doing, reassurance). Same facts, three
different messages — because the audiences need different things.

Two foundational principles run through all workplace communication: **lead with
the conclusion** (BLUF — Bottom Line Up Front: state the point first, then
support it, because busy readers need the takeaway immediately, not after three
paragraphs of buildup), and **one message, one point** (each communication should
have a single clear thing you want the audience to know or do — burying it among
five other things means it gets lost).

---

## Going Deeper

### The curse of knowledge

The root cause of sender-first communication is the *curse of knowledge*: once you
know something deeply, it's genuinely hard to remember what it's like not to know
it, so you skip the context the audience needs, use jargon they don't share, and
assume understanding they don't have. Fighting it requires deliberately modeling
the audience: what do they already know? what do they care about? what's their
context? A senior engineer explaining to another senior engineer can assume a
lot; the same person explaining to a PM, an exec, or a new hire must supply what
those audiences lack — and the curse of knowledge makes this genuinely effortful,
which is why it's a skill to practice, not a thing that comes naturally.

### Triage: audience, purpose, medium

Before communicating anything important, three quick questions: **Audience** — who
is receiving this, what do they know, what do they care about? **Purpose** — what
do I want to happen as a result (inform? get a decision? get help? align?)? —
because the purpose shapes everything (a message meant to get a decision looks
different from one meant to inform). **Medium** — is this a Slack message, an
email, a doc, a meeting, a presentation? (matching medium to message matters —
Lesson 15/18). This triage takes seconds and prevents the most common failures
(wrong content for the audience, no clear ask, wrong channel).

### BLUF and calibrating detail

**Lead with the conclusion.** Busy people read the first sentence and decide
whether to keep reading — so the takeaway goes first, support after. "The
migration will slip three weeks; here's why and what I need" beats three
paragraphs of context ending in the point (which the reader may never reach).
Then **calibrate detail to the audience**: engineers want the technical depth;
executives want the impact and the ask with the detail available if they want it
(an executive summary up top, detail below — Lesson 41 in the English track).
Giving an exec engineering detail loses them; giving an engineer only the
high-level frustrates them. Match the depth to who's reading.

{: .note }
> **Same facts, different message — that's not "spin"</br>**
> Shaping a message for its audience sometimes feels like being inconsistent or
> "political" — but it's neither, as long as the <em>facts</em> are consistent
> across versions. Telling your team the technical root cause and telling the
> customer "we've identified and fixed the issue" aren't contradictory — they're
> the same truth at the right resolution for each audience. What <em>would</em> be
> wrong is telling different audiences contradictory things (saying it's fixed to
> the customer while telling the team it isn't). Audience-first communication is
> selecting <em>what to emphasize and how to frame</em> for the receiver's needs —
> honest and appropriate — not saying different things to different people.

---

## Lab — Scenario

**The situation:** Your team just completed a significant piece of work: you
migrated the authentication system to a new provider. It took six weeks (two
longer than estimated), improves login reliability and security, and required a
brief maintenance window last weekend that a few customers noticed. You need to
communicate this to three audiences.

**Write the same update three ways:** (1) to your **team** (in your team channel);
(2) to your **director** (who cares about delivery, impact, and risks); (3) to an
affected **customer** who noticed the maintenance window and emailed asking what
happened. Then note what changed between the three and why.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
The three versions should share the facts but differ sharply in emphasis, detail,
framing, and what they lead with — shaped by each audience's needs. Examples:
<br><br>
<strong>To the team (team channel):</strong> "🎉 Auth migration is done and live —
nice work everyone, especially [names] on the tricky session-migration piece. We're
now fully on [new provider]; login reliability and security are both improved, and
the weekend maintenance window went smoothly. A few learnings for next time: the
session migration was the part that ran long (the two-week slip was mostly there —
we underestimated the edge cases around existing tokens), so let's capture that in
the retro. Monitoring looks clean so far; ping me if you see anything odd in the
auth metrics this week." — <em>Audience needs:</em> recognition (they did the work),
technical specifics (what's live, the session-migration detail), learnings (the
retro framing on the slip — honest, blameless, forward-looking), and a technical
heads-up (watch the metrics). Leads with celebration/credit; includes the
technical depth engineers want.
<br><br>
<strong>To the director:</strong> "<strong>Auth migration shipped — improved login
reliability and security, live as of the weekend.</strong> It ran two weeks over
(6 vs 4) due to more edge cases in session migration than estimated — I've noted
the estimation learning for our planning. Customer impact was minimal: a brief
planned maintenance window Saturday that a few customers noticed; we communicated
it in advance and I'm handling the couple of inbound questions. No ongoing risks —
monitoring is clean. Net: we're now on a more reliable, more secure auth platform,
which also unblocks [the SSO feature / whatever it enables]." — <em>Audience needs:</em>
BLUF (the outcome first), delivery honesty (the slip, owned, with the learning —
directors value leads who surface slips honestly, Lesson 41), impact and risk (the
customer impact and that it's handled, no ongoing risk), and business connection
(what this enables). No engineering detail (the director doesn't need session-token
specifics); focused on delivery, impact, risk, and what it unblocks.
<br><br>
<strong>To the customer:</strong> "Hi [name] — thanks for reaching out, and sorry
for any inconvenience from Saturday's maintenance. We performed a planned upgrade
to our authentication system to improve login reliability and security. The brief
maintenance window was part of that upgrade, and everything is now running normally
— in fact, you should experience more reliable logins going forward. If you have
any trouble signing in or notice anything unusual, please let me know directly and
I'll make sure it's handled right away. Thanks for your patience!" — <em>Audience
needs:</em> what happened <em>in their terms</em> (an upgrade that benefits them —
not "we migrated auth providers"), reassurance (it's done, running normally,
benefits you), an apology for the inconvenience (they were affected), and a path if
there's a problem. No internal detail (the slip, the provider name, the technical
specifics are all irrelevant and would only confuse or concern them); framed around
<em>their</em> experience and benefit.
<br><br>
<strong>What changed and why:</strong> The <em>facts</em> are consistent (an auth
upgrade, done, improves reliability/security, brief maintenance window) but each
version is shaped for its audience: the <em>team</em> version leads with credit and
includes technical depth and learnings (they did the work and need the details);
the <em>director</em> version leads with the outcome (BLUF), honestly surfaces the
slip with its learning, and focuses on impact/risk/business-value (what a director
manages); the <em>customer</em> version is entirely in the customer's terms (an
upgrade that benefits them), reassuring, apologetic for the inconvenience, with a
support path (what a customer cares about — their experience, not your internals).
Note what's <em>included</em> vs <em>omitted</em> differs by relevance: the slip is
central for the director (delivery), a learning for the team (retro), and entirely
absent for the customer (irrelevant to them); the technical specifics are detailed
for the team, summarized for the director, and gone for the customer. This is
audience-first communication: same truth, three resolutions, each shaped by what
that receiver needs, cares about, and can use — <em>not</em> spin (the facts don't
contradict), but appropriate framing. Common mistakes: (1) writing one version and
sending it to all three (engineering detail bores the director and confuses the
customer; customer-reassurance framing frustrates the team); (2) burying the point
(not leading with the outcome for the busy director); (3) hiding the slip from the
director (dishonest and erodes trust — Lesson 41 — surface it, owned); (4) giving
the customer internal detail (the provider name, the slip) that only worries or
confuses them.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| The curse of knowledge & making ideas stick | *Made to Stick*, Chip & Dan Heath |
| BLUF / leading with the conclusion | *The Pyramid Principle*, Barbara Minto |
| Audience-first technical communication | *The Manager's Path*, Camille Fournier |
| Writing for different audiences | English for Work track, Phases 5 & 7 |
| Communication as a leadership skill | LeadDev — <https://leaddev.com/communication> |

---

## Checkpoint

**Q1.** What is the "curse of knowledge," why does it make you a worse
communicator, and what's the practical defense against it?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The curse of knowledge is the cognitive bias that once you know something deeply,
you can't easily reconstruct what it was like to <em>not</em> know it — expertise
makes it genuinely hard to imagine the audience's state of not-understanding. It
makes you a worse communicator because it causes you, without realizing it, to:
skip the context and background the audience needs (it's so obvious to you that you
don't see it needs saying); use jargon and shorthand the audience doesn't share
(the terms are second-nature to you); assume knowledge and mental models the
audience lacks (you can't feel the gap); and structure the explanation around
<em>your</em> understanding rather than the path a newcomer needs. The result is
communication that makes sense to you and to people who already understand, but
loses the actual audience — which is invisible to you precisely because of the
curse (it feels clear to you, so you can't tell it's unclear to them). It's why
experts are often bad at explaining things to non-experts, why the person who wrote
the code writes the worst documentation for newcomers, and why a lead can give a
technically perfect explanation that a PM or exec doesn't follow at all. The
practical defense is to <strong>deliberately model the audience</strong> — since
you can't <em>feel</em> their gap, you have to consciously reason about it: before
communicating, explicitly ask "what does this specific audience already know? what
do they NOT know that I'm assuming? what jargon am I using that they don't share?
what context do they need that's obvious to me?" — and shape the message to supply
what they lack. Concrete tactics: get feedback from someone closer to the
audience's level (does this land for you?), watch for the audience's confusion
signals and adjust, test explanations on a real member of the target audience,
and default to over-supplying context for unfamiliar audiences (better slightly
too much than losing them). The meta-move is <em>audience-first thinking</em>
(this whole lesson): consciously starting from the receiver's state rather than
your own — which is effortful precisely because the curse of knowledge makes your
own state feel like the natural starting point, so good communicators build the
deliberate habit of asking "what does <em>this audience</em> need" every time,
rather than trusting that what's clear to them will be clear to others (it won't
be, and they can't feel that it isn't — which is the whole trap).
</details>

**Q2.** "Lead with the conclusion" (BLUF) runs against the instinct to build up to
your point. Why is leading with the conclusion usually right for workplace
communication, and when might building up be better?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Leading with the conclusion is usually right because <strong>workplace readers are
busy, skimming, and deciding moment-to-moment whether to keep engaging</strong> —
they read the first sentence (or the subject line, or the first slide) and, if the
point isn't there, they may stop, skim past it, or miss it entirely; they don't
have the patience (or often the time) for a build-up that delays the takeaway. If
the conclusion is at the end, the busy director scanning their inbox reads three
paragraphs of context, doesn't reach the point, and moves on — or reads the point
but resents the time it took to extract. Leading with the conclusion respects the
reader's time and attention: they get the takeaway immediately, can decide whether
they need the supporting detail, and if they read no further they've still got the
essential message. It also aids comprehension — knowing the conclusion first gives
the reader a frame to organize the supporting details around (they read the
"because" knowing what it's explaining), whereas a build-up forces them to hold
context without knowing what it's building toward. This is why BLUF is standard in
executive communication, the military (where it originated), journalism (the
"inverted pyramid" — most important first), and effective business writing (Minto's
Pyramid Principle). The instinct to build up comes from academic writing and
storytelling (where suspense and a logical build have value) and from a natural
desire to justify before concluding (show your work first) — but workplace
communication optimizes for the reader getting the point efficiently, not for
suspense or for the writer's comfort. When <em>might</em> building up be better?
(1) <strong>When you need to bring a resistant audience along</strong> — if the
conclusion will trigger immediate rejection (a controversial recommendation, bad
news that needs framing), sometimes establishing shared context and reasoning
<em>first</em> makes the conclusion landable, whereas leading with it provokes a
defensive shutdown before they've heard the why (though even here, a brief framing
then the point usually beats a long build-up). (2) <strong>Narrative/persuasive
contexts where the journey matters</strong> — a keynote, a story meant to inspire,
a case being built where the emotional or logical arc is the point (Lesson 17's
presenting has some of this). (3) <strong>When the reasoning IS the value</strong>
— a teaching explanation where understanding the derivation matters more than the
answer. But these are exceptions; the default for updates, proposals, requests,
status, and most workplace communication is BLUF — state the point, then support
it — because the reader's time and attention are the scarce resources, and burying
the conclusion squanders both. The practical rule: lead with the conclusion unless
you have a specific reason the build-up serves the reader better, and even then,
keep any build-up short and signal where it's going.
</details>

---

## Homework

Take one real communication you need to make (an update, a request, a status, a
proposal) and consciously apply audience-first thinking. First triage: who's the
audience, what's the purpose, what medium? Then draft it leading with the
conclusion, calibrated to what that audience knows and cares about. If it goes to
multiple audiences, write the distinct versions. Then reflect: how different is
this from your default (probably sender-first) draft, and what did modeling the
audience change? Bonus: find a recent message you sent that landed poorly and
diagnose it — was it sender-first, buried its point, or mis-calibrated for the
audience?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the deliberate audience-first habit against the sender-first
default. A strong response shows the triage actually shaping the message (the
audience/purpose/medium analysis changing what's included, emphasized, and led
with) and produces a message noticeably different from the instinctive draft — most
people, comparing their audience-first version to what they'd have sent by default,
find the default was sender-first (organized around what <em>they</em> wanted to
say, in their order, with their assumed context and jargon) and the audience-first
version is reorganized around what the <em>receiver</em> needs (led with the point,
context supplied or trimmed for that audience, framed around their concerns). The
common realizations: (1) the default buried the point (the actual ask or takeaway
was somewhere in the middle or end — moving it to the front is the single biggest
improvement); (2) the default assumed context or used jargon the audience doesn't
share (the curse of knowledge — modeling the audience reveals what needs supplying
or cutting); (3) the default optimized for completeness (everything the sender
knows) rather than relevance (what this audience needs) — the audience-first version
is often shorter because it cuts what doesn't serve the receiver. The multi-audience
version (if applicable) drives home that the <em>same information</em> becomes
genuinely different messages for different receivers (the lab's core lesson). The
diagnosis of a poorly-landed past message is especially valuable — most communication
failures trace to one of a few audience-first violations: it was sender-first
(organized around the sender's perspective, so the receiver had to work to extract
their part), it buried the point (the reader missed or never reached the takeaway),
it was mis-calibrated (too much detail for the audience, or too little, or the wrong
kind), or it had no clear purpose/ask (the reader didn't know what they were
supposed to do with it). Naming which failure occurred builds the diagnostic eye
for your own communication. The meta-skill: communication is a <em>design</em>
problem where the user is the receiver — you're not "expressing what you think," you're
"engineering a message that produces the right understanding and response in a
specific audience" — and the discipline is starting every important communication
from the receiver's needs (audience-first) rather than your own content
(sender-first), leading with the conclusion, and calibrating to what they know and
care about. This is effortful (the curse of knowledge fights you) but it's the
highest-leverage communication habit, and since a lead spends more time
communicating than coding (this phase's premise), getting it right compounds across
everything you do — every update, request, proposal, and decision lands better when
it's shaped for its audience rather than transmitted from yours. If your
audience-first version was barely different from your default, either you're
already a strong audience-first communicator (good) or the message was to an
audience very like you (where sender and audience-first converge — the test is a
message to a <em>different</em> audience, like an exec or customer, where the gap
shows).
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 14 — Explaining Technology to Non-Technical People →](lesson-14-explaining-tech){: .btn .btn-primary }
