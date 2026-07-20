---
title: "Lesson 25 — Async Etiquette"
nav_order: 5
parent: "Phase 4: Slack Communication"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 25: Async Etiquette

## Concept

Beyond individual messages, teams have *unwritten rules* for Slack — and following
them makes you a coworker people appreciate, while breaking them (even with good
intentions) quietly annoys everyone. These norms are about respecting people's time,
attention, and focus in a shared async space.

```
   THE UNWRITTEN RULES:

   • use THREADS for replies (don't clutter the main channel)
   • @channel / @here sparingly (you're pinging everyone)
   • don't expect instant replies (async = read when convenient)
   • schedule-send off-hours messages (don't pressure people)
   • react with emoji to acknowledge (👍 instead of a "got it" message)
```

None of these are written down, but violating them — @channel-ing for a non-urgent
thing, sending messages at midnight, expecting instant replies — creates friction and
resentment. Following them marks you as a considerate, low-friction colleague.

---

## Going Deeper

### Use threads

Reply in **threads** to keep the main channel clean. Threading keeps related discussion
together and doesn't push unrelated messages down for everyone. Posting replies to the
main channel clutters it and makes conversations hard to follow. Thread your replies;
use the main channel for new topics.

### @channel and @here sparingly

`@channel` notifies *everyone* (even offline); `@here` notifies everyone *currently
online*. These are interruptions for many people — use them only when something
genuinely needs everyone's attention (an incident, a critical announcement). For a
normal message, just post it or @ the specific people. Overusing @channel trains people
to ignore your pings and annoys everyone.

### Don't expect instant replies

Async means people read and respond when convenient, not instantly. Don't expect (or
pressure for) immediate replies, and don't follow up impatiently after a few minutes. If
something is genuinely urgent, say so ("urgent — the site is down") or use a faster
channel. For normal things, let people respond in their own time; if you need a reply by
a certain time, say so kindly ("no rush, but I'd love your thoughts by end of day if
possible").

### The after-hours message

Sending messages at night or on weekends — even without expecting a response — signals
an always-on culture and can pressure people. Use **schedule-send** to deliver during
work hours, so your off-hours working doesn't leak into an off-hours expectation for
others. (If you're a lead, your habits set the team's norms.) A small courtesy that
protects everyone's boundaries.

### Emoji reactions as lightweight acknowledgment

Use emoji reactions (👍 ✅ 👀) to acknowledge messages without posting a reply — a
lightweight "got it / seen / agree" that doesn't clutter the channel or notify everyone.
Reacting 👍 to "I'll deploy at 3pm" acknowledges it without a "sounds good" message that
adds noise. Efficient and still warm — the async equivalent of a nod.

{: .note }
> **These unwritten rules make teams love or resent a coworker**
> The etiquette is invisible until you break it — nobody writes down "don't @channel
> for non-urgent things," but violating these quietly frustrates people (the coworker
> who @channels constantly, expects instant responses, sends midnight messages, and
> clutters the main channel is subtly resented, even if nice). Following them marks you
> as a considerate, low-friction colleague. For your goal of good relationships, the
> async etiquette is as important as the warmth in individual messages — it's warmth at
> the level of respecting everyone's shared time and attention.

---

## Lab — Judgment Calls

For each situation, decide the right async-etiquette move and explain briefly.

**Your answer (decide + explain each):**

1. You have a question for the whole team, and it's not urgent. @channel or not?
2. You reply to a message in a busy channel. Main channel or thread?
3. You send someone a question at 9pm (you're working late). What do you do?
4. Someone posts "I'll deploy the fix at 3pm." You want to acknowledge you saw it. How?
5. You asked a non-urgent question 20 minutes ago and no reply. What do you do?
6. The production site just went down and everyone needs to know NOW. @channel?

<details>
<summary>Show Model Answers</summary>
<br>
<strong>1. Team question, not urgent:</strong> Don't @channel — just post it (people
with time will answer), or @ the specific people most likely to know. @channel notifies
everyone including offline; save it for genuinely everyone-needs-this things.
<br><br>
<strong>2. Reply in a busy channel:</strong> Use a thread — it keeps the discussion
together and doesn't push unrelated messages down for everyone.
<br><br>
<strong>3. Question at 9pm:</strong> Schedule-send it for the next morning. Sending at
9pm — even without expecting a reply — signals an always-on expectation and can pressure
them. (If genuinely urgent, that's different — say so.)
<br><br>
<strong>4. Acknowledge "I'll deploy at 3pm":</strong> React with an emoji (👍 or ✅) — a
lightweight acknowledgment without a "sounds good" message that adds noise. The async
nod.
<br><br>
<strong>5. Non-urgent, no reply after 20 min:</strong> Wait. Async means people reply
when convenient — 20 minutes is nothing for a non-urgent question. Don't follow up
impatiently (it pressures them and reads as annoyed). If it becomes time-sensitive later,
a gentle bump with the reason is fine (Lesson 17), but not after 20 minutes.
<br><br>
<strong>6. Production down, everyone needs to know NOW:</strong> Yes, @channel/@here —
this is exactly what it's for. A genuine incident where everyone needs immediate
awareness is the legitimate use. (Don't overuse it for non-urgent things, but DO use it
when something truly needs everyone now.)
<br><br>
The theme: respect shared time and attention — thread to keep channels clean, @channel
only for genuine everyone-now things, don't pressure for instant replies or send
off-hours (schedule-send), and use reactions for lightweight acks.
</details>

---

## Phrase Bank

| Situation | Move |
|---|---|
| Non-urgent team question | post it (or @ specific people) — not @channel |
| Replying to a message | use a thread |
| Off-hours message | schedule-send for work hours |
| Acknowledge without noise | react with 👍 / ✅ / 👀 |
| Genuinely urgent | say "urgent —" and/or @here; or call |
| Need a reply by a time | "no rush, but I'd love this by [time] if possible" |
| Genuine incident | @channel / @here (this is what it's for) |

---

## Further Reading

| Topic | Source |
|---|---|
| Async communication norms | GitLab async handbook; *Remote*, Fried & Hansson |
| Response-time norms & boundaries | Leadership track Lesson 18 (async communication) |
| Slack etiquette | your team's norms (observe them); general Slack guides |

---

## Checkpoint

**Q1.** Why should you use @channel/@here sparingly, and what makes something genuinely
warrant it?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You should use @channel/@here sparingly because <strong>they interrupt everyone</strong>
— @channel notifies every member (even offline), @here everyone currently online — so
each use is a push notification to potentially dozens of people. @channel-ing for
something that doesn't need everyone's immediate attention (a question only a few can
answer, a non-urgent update, an FYI) interrupts many people unnecessarily, pulling them
from focus for something irrelevant to most of them. Two bad effects: (1) it's
inconsiderate (you cost many people's attention for your convenience); (2) it's
self-defeating — frequent non-urgent @channels train people that your @channels aren't
important, so they start ignoring or muting them, and when you have a <em>real</em>
emergency, it doesn't get the attention it needs (crying wolf). What genuinely warrants
it: <strong>something that truly needs everyone's immediate awareness</strong> — a
production incident, a critical time-sensitive announcement (an emergency change, a
security issue, "everyone stop deploying"). The test: does <em>everyone</em> genuinely
need to see this <em>now</em>? If yes (an incident, a critical time-sensitive
announcement), @channel is correct — it's what it's for. If no (only some need it, or it's
not time-sensitive, or it's an FYI), don't @channel — post it normally, @ the specific
people, or thread it. The distinction is <strong>everyone + now</strong>: @channel is for
things both broadly relevant AND immediately important; most things aren't both, so most
shouldn't @channel. Using it correctly (rarely, for genuine everyone-now situations) keeps
it effective (people attend because your @channels are always important) and keeps you
from being the annoying over-pinger — a form of warmth at the team level (respecting the
team's shared attention).
</details>

**Q2.** Why is following the unwritten async rules important even though nobody writes
them down — and how does it relate to your warm-communication goal?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Following the unwritten async rules matters precisely <em>because</em> they're unwritten:
they're the invisible norms a considerate colleague follows and a frustrating one
violates, and violating them — even with good intentions — quietly creates friction and
resentment people rarely voice but definitely feel. Nobody writes down "use threads,"
"don't @channel for non-urgent things," "don't expect instant replies," "don't send
midnight messages" — but these govern how pleasant and low-friction you are in the shared
async space. The coworker who clutters the channel (no threads), @channels constantly,
pressures for instant replies, sends off-hours messages, and posts "got it" instead of
reacting is subtly resented — each behavior disrespects others' time, attention, or
boundaries, accumulating into "exhausting to work with in Slack" even if they're nice in
individual messages. Since these rules are invisible, you have to learn them deliberately
and follow them consciously — violating them out of ignorance still causes friction. How
it relates to your warm-communication goal: <strong>async etiquette is warmth at the team
level — respecting everyone's shared time, attention, and boundaries</strong>. Individual-
message warmth (Phase 3) makes each message friendly; async etiquette makes your overall
<em>presence</em> considerate. Both matter for being someone people like working with. A
person can write warm individual messages but be exhausting async (constant @channels,
instant-reply pressure, midnight pings) — the individual warmth doesn't compensate for
disrespecting the team's shared attention. So genuine warmth requires both: warm messages
AND good async etiquette. The etiquette is arguably a deeper consideration — not being
nice to one person in one message, but respecting the whole team's focus, time, and
off-hours, affecting everyone all the time. For your goal of being a warm, well-liked
colleague, mastering the async etiquette is as important as individual-message warmth —
it's the difference between someone warm-in-messages-but-friction-generating and someone
warm AND low-friction, which is who people genuinely enjoy working with.
</details>

---

## Homework

For a week, deliberately follow the async etiquette: thread your replies, @channel only
for genuine everyone-now things, don't expect/pressure for instant replies (and
schedule-send any off-hours messages), and use emoji reactions to acknowledge instead of
posting "got it" messages. Observe your team's specific norms and match them. Notice
whether being a lower-friction presence changes how people respond to you. Add to your
checklist: "async — thread it? @channel only if everyone-now? schedule-send off-hours?
react instead of clutter?"

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds async-etiquette awareness — the invisible norms that determine how
pleasant you are to work with in the shared space. A strong response deliberately follows
the rules and observes the team's specific norms (which vary — some teams @channel more,
some are heavy emoji users, some have strict off-hours cultures; matching the local norms
is part of it). The realizations: (1) you may have been violating some rules without
realizing (not threading, expecting quick replies, off-hours messages), and correcting
them makes you lower-friction; (2) the etiquette is invisible but real — nobody tells you
"you @channel too much" directly, but it quietly affects how people feel about working
with you; (3) being low-friction is appreciated even though rarely said. The meta-point:
async etiquette is warmth and consideration at the team level, complementing individual-
message warmth (Phase 3). Being genuinely warm and well-liked requires both — a warm
messenger who's exhausting async undercuts their warmth. So mastering the unwritten rules
is key to being the considerate, low-friction presence you want to be. Observe your
team's norms, follow the etiquette consciously, and you complement your warm messages with
a considerate overall presence. Add the async checks to your checklist. This is the fifth
Slack lesson; the last (announcements) covers high-visibility messages, completing the
Slack skills that, combined with warmth, make you a teammate people genuinely enjoy
working with.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 26 — Announcements to Wide Channels →](lesson-26-announcements){: .btn .btn-primary }
