---
title: "Lesson 21 — Message Shapes: Lead With the Ask"
nav_order: 1
parent: "Phase 4: Slack Communication"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 21: Message Shapes — Lead With the Ask

## Concept

The single most useful Slack skill: **structure your message so a busy reader gets it
in one pass.** The most common mistake is burying the ask — making the reader work to
figure out what you actually want. Lead with the ask, then give context.

```
   BURIED ASK (reader has to dig)      LEAD WITH THE ASK (one pass)
   ──────────────────────────────      ────────────────────────────
   "Hey, so I was looking at the        "Could you review PR #42 today?
    payment flow and I noticed some      It fixes the login bug and is
    weird behavior with the tokens,      blocking the release. 🙏"
    and I've been debugging it, and
    I think I found a fix, and I made
    a PR, so I was wondering if maybe
    you could take a look at it?"        ↑ ask first, context second
```

The reader — often busy, skimming — should know *what you want from them* in the
first line, then get the context if they need it. This connects to BLUF (Bottom
Line Up Front) from the leadership track: lead with the point. For Slack
specifically, this means: the ask first, one topic per message, and formatting that
makes it scannable.

---

## Going Deeper

### Ask first, context second

Put what you want from the reader at the top; supporting context below:

- ✗ [three lines of background] ... "so could you review it?" (ask buried at the end)
- ✓ "Could you review PR #42 when you get a chance? Context: it fixes the login bug
  that's blocking the release." (ask first, context after)

The reader gets the ask immediately and decides how much context to read. Even better,
make the ask visually prominent (bold, or on its own first line).

### One message, one topic

Don't cram multiple unrelated things into one message — the reader will address one
and miss the others. Send separate messages (or clearly separate points) for separate
topics:

- ✗ "Could you review my PR, and also do you know when the migration is happening,
  and by the way I think the staging server is down?" (three things — some will get
  lost)
- ✓ Three separate, focused messages (or a clearly numbered list if they belong
  together).

One topic per message means each gets addressed, and the reader isn't juggling.

### The no-hello rule

Don't send "hi" or "hey" alone and wait for a reply before stating your point — it
wastes time (the reader sees "hi", replies "hi", and waits for your actual message).
Put your greeting and your point in the *same* message:

- ✗ "Hi!" [reader replies "hey!"] ... [then you type the actual question]
- ✓ "Hi! Quick question — could you tell me where the deploy config lives?"

Include the ask with the greeting. (See nohello.net — it's a known etiquette point.)

### Format for scanning

Make longer messages scannable so the structure is visible at a glance:

- **Bold the ask** or the key point.
- **Bullet points** for lists (multiple items, options, steps).
- **Code blocks** for code, errors, or commands (` ``` `).
- **A TL;DR** at the top for long messages: "TL;DR: I need X by Friday. Details
  below." — so busy readers get the essence and can skip the rest.

{: .note }
> **The one-pass test</br>**
> Before sending a Slack message, ask: <strong>"if the reader reads only the first
> line, do they know what I want from them?"</strong> If the ask is buried below
> context, move it up. Busy people skim, and the first line often decides whether
> they engage now or later (or miss the point) — so the ask (or the main point) goes
> first, context second. This one habit — lead with the ask — makes your messages
> dramatically easier to act on, and it's the foundation of good Slack
> communication. Warm framing (Phase 3) still applies — lead with the ask
> <em>warmly</em> ("Could you...? 🙏") — but the ask comes first.

---

## Lab — Rewrite Drill

Restructure these buried-ask messages so the ask/point comes first and they're
scannable — keeping the warmth.

**Your rewrite (lead with the ask):**

1. "Hey, so I've been working on the analytics dashboard for the past few days and I ran into an issue with the data pipeline, I think there might be a permissions problem, and I was trying to figure it out but I'm stuck, so I was wondering if you could maybe help me?"
2. "Hi!" (then waiting to type the real question)
3. "Could you review my PR, and also I wanted to check if the release is still Thursday, and one more thing — do we have a runbook for the payment service?"
4. "So the deploy failed and I looked at the logs and there's a timeout error and I restarted the service but it happened again and I checked the database and that seems fine and I'm not sure what else to try, can you take a look?"

<details>
<summary>Show Model Rewrites</summary>
<br>
<strong>1.</strong> "Hi [name]! Could you help me with a permissions issue on the
analytics data pipeline? 🙏 I'm stuck — it looks like a permissions problem but I
can't crack it. Happy to share what I've tried." (Ask first ("could you help me with
X"), the specific problem named, context offered but not dumped. The reader knows what
you want in the first line.)
<br><br>
<strong>2.</strong> "Hi! Quick question — could you point me to where the deploy
config lives? Thanks!" (Greeting + the actual ask in ONE message — no lone "hi" that
makes them wait.)
<br><br>
<strong>3.</strong> Split into focused messages (or number them): <strong>"Hi! Three
quick things: (1) could you review PR #42 when you get a chance? (2) is the release
still Thursday? (3) do we have a runbook for the payment service? Thanks!"</strong> —
or better, three separate messages so each gets addressed. (If together, number them
so none gets lost.)
<br><br>
<strong>4.</strong> "Could you take a look at the failed staging deploy with me? 🙏
It's throwing a timeout error. What I've tried so far: restarted the service (failed
again), checked the DB (looks fine). Not sure what else to try." (Ask first, the error
named, then a clean scannable list of what you've tried — much easier to act on than
the run-on narrative. This also previews Lesson 23's how-to-ask-for-help.)
<br><br>
The pattern: the ask or main point comes first (so the reader knows immediately what
you want), one topic per message (or clearly numbered), no lone "hi," and formatting
(lists, brevity) that makes it scannable — all while keeping the warmth (Hi!, 🙏,
Thanks!).
</details>

---

## Phrase Bank

| Move | Example |
|---|---|
| Lead with the ask | "Could you [X]? Context: [Y]." |
| Greeting + ask together | "Hi! Quick question — [ask]?" |
| TL;DR for long messages | "TL;DR: I need [X] by [when]. Details below." |
| Multiple items | "Two things: (1) … (2) …" |
| Flag it's quick | "Quick one —" / "Small ask —" |
| Code/error | use a ``` code block ``` |

---

## Further Reading

| Topic | Source |
|---|---|
| Lead with the conclusion (BLUF) | Leadership track Lesson 13 (audience-first) |
| The no-hello rule | <https://nohello.net/> |
| Structuring messages for skimmers | Leadership track Lesson 16 (design docs) |

---

## Checkpoint

**Q1.** Why should you lead with the ask rather than building up to it, and what's the
"one-pass test"?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You should lead with the ask because <strong>the reader is busy and skimming, and the
first line often decides whether and how they engage</strong>. When you build up to the
ask (three lines of context, then "so could you...?" at the end), the reader has to
read through all the context to discover what you actually want from them — and busy
people often don't: they skim the first line, and if it's not clear what's being asked,
they may deprioritize it, misunderstand, or miss the ask entirely (it's buried below).
Leading with the ask ("Could you review PR #42?") means the reader knows immediately
what you want, can decide right away how to respond (do it now, later, or ask a
question), and can read the context below only if they need it. This respects the
reader's time and attention (they get the essential thing first) and makes your message
far easier to act on. It's the same principle as BLUF (Bottom Line Up Front) from the
leadership track — busy readers need the point first, support second — applied to
Slack, where skimming is the norm and the first line is what gets read. The
<strong>one-pass test</strong> is the practical check: before sending, ask <strong>"if
the reader reads only the first line, do they know what I want from them?"</strong> If
yes, the ask is well-placed (leading). If no — if the ask is buried below context, or
the first line is just background — move the ask up so it's the first thing they see.
The test is called "one-pass" because a well-structured message should be actionable
in a single skim: the reader passes their eyes over it once and immediately knows the
ask (and can get context if they want). A buried ask requires multiple passes or
careful reading to extract the point, which busy readers won't do. So the one-pass test
enforces lead-with-the-ask: it fails any message where the first line doesn't convey
what you want, prompting you to front-load the ask. This is the single most useful
Slack habit because it makes every request, question, and message easier to act on —
and since so much work coordination happens via Slack asks, getting the message shape
right (ask first, context second, scannable) has a big cumulative effect on how
effectively you communicate. Note that leading with the ask doesn't mean being cold or
abrupt — you still frame it warmly (Phase 3): "Could you review PR #42 when you get a
chance? 🙏" leads with the ask AND is warm. The ask comes first; the warmth wraps it.
Lead with the ask, warmly.
</details>

**Q2.** What's the problem with cramming multiple topics into one Slack message, and
what's the fix?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The problem with cramming multiple topics into one message is that <strong>the reader
tends to address one and miss or forget the others</strong>. When you send "could you
review my PR, and also when's the release, and by the way is staging down?", the reader
reads it, latches onto one item (usually the first, or the most salient), responds to
that, and the other items get lost — either forgotten in the moment, or buried when the
conversation moves on. People process one thing at a time in a quick Slack exchange,
and a single message with three asks overloads that — some will inevitably slip through
the cracks, leaving you chasing the unanswered ones later (or not getting them
answered at all). It also makes the message harder to respond to (the reader has to
juggle multiple things) and harder to track (was item 2 answered? which item is this
reply about?). The fix has two forms depending on whether the topics are related: (1)
<strong>For unrelated topics, send separate messages</strong> — one message per topic,
each focused, so each gets its own attention and response, and each can be tracked
independently (and threaded separately if the conversation develops). This is the
cleaner approach for genuinely separate things (a PR review, a release question, a
staging alert — three different topics that deserve three messages). (2) <strong>For
related items that belong together, use a clear numbered list</strong> — "Three quick
things: (1)... (2)... (3)..." — so the reader sees there are multiple items, can
address each, and can reference them by number in their reply ("re: 2, yes it's still
Thursday"). The numbering makes the multiplicity visible and trackable, reducing the
chance of missing one. The underlying principle is <strong>one message, one topic</strong>
(or, if grouped, clearly-delineated topics) — matching the reader's one-thing-at-a-time
processing, so nothing gets lost. This pairs with lead-with-the-ask (Q1): each focused
message (or each numbered item) leads with its own ask, so the whole thing is scannable
and actionable. The practical habit: before cramming several things into one message,
ask "are these really one topic, or several?" — if several, split them (or number
them), so each gets addressed rather than one getting attention and the rest getting
lost. This is a small discipline that noticeably improves how reliably your asks get
answered, which matters because unaddressed asks mean chasing, delays, and things
falling through — all avoidable by structuring messages as one-topic-each.
</details>

---

## Homework

For a few days, apply lead-with-the-ask to your Slack messages: put what you want from
the reader in the first line, context below; send one topic per message (or number
related items); include your greeting and ask in the same message (no lone "hi"); and
format longer messages to be scannable (bold the ask, use lists, add a TL;DR). Run the
one-pass test before sending important messages. Notice: do your messages get responded
to faster and more completely? Add to your checklist: "does the first line say what I
want? one topic per message? scannable?"

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the lead-with-the-ask habit and the associated message-shape
discipline (one topic, no-hello, scannable formatting), which together make Slack
messages much easier to act on. A strong response applies the moves and notices the
effect — messages with the ask up front tend to get responded to faster and more
completely, because the reader immediately knows what's wanted and can act, rather than
having to dig for the point or missing buried asks. The common realizations: (1)
<strong>your asks may have been getting buried</strong> — the natural tendency (often
from wanting to provide context or ease into the request) is to build up to the ask,
which makes busy readers work to find it; leading with it is a small change with a big
effect on responsiveness. (2) <strong>multi-topic messages were losing items</strong> —
splitting or numbering ensures each gets addressed. (3) <strong>the no-hello habit
speeds things up</strong> — including the ask with the greeting avoids the pointless
"hi"/"hey" round-trip. (4) formatting (bold ask, lists, TL;DR) makes longer messages
scannable, so busy readers get the essence. The one-pass test is the practical enforcer:
checking "does the first line say what I want?" before sending catches buried asks. The
meta-point: Slack is where most work coordination happens, and message shape (ask first,
one topic, scannable, warm) significantly affects how effectively that coordination
works — well-shaped messages get acted on quickly and completely, poorly-shaped ones
cause delays, missed asks, and chasing. So mastering message shape is high-leverage for
day-to-day effectiveness, not just tone. Importantly, lead-with-the-ask combines with
the warmth of Phase 3 — you lead with the ask <em>warmly</em> ("Could you...? 🙏"),
getting both the clarity (ask first) and the warmth (friendly framing) — so this isn't a
trade-off against your warm-tone goal; it's structuring your warm messages so they're
also clear and actionable. Add the message-shape checks to your checklist. This is the
foundation of the Slack phase; the next lessons apply it to specific message types —
status updates (answer questions before they're asked), asking for help (make helping
you easy), answering/unblocking others, async etiquette, and announcements — each a
common Slack situation where good structure and warmth matter.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 22 — Status Updates →](lesson-22-status-updates){: .btn .btn-primary }
