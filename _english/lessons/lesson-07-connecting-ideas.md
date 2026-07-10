---
title: "Lesson 07 — Connecting Ideas Without Run-ons"
nav_order: 7
parent: "Phase 1: Sentence Mechanics"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 07: Connecting Ideas Without Run-ons

## Concept

When you have two related thoughts, you need to join them *correctly* — and the
most common mistake is the **comma splice**: joining two complete sentences with
just a comma. This is one line away from clear or confusing.

```
   TWO complete thoughts:
     "I deployed the fix."   +   "It broke the build."

   ✗ COMMA SPLICE:  "I deployed the fix, it broke the build."
                    (two full sentences glued by only a comma — wrong)

   THREE correct fixes:
   ✓ Two sentences:  "I deployed the fix. It broke the build."
   ✓ A joining word: "I deployed the fix, but it broke the build."
   ✓ A semicolon:    "I deployed the fix; it broke the build."
```

The single most useful move for a non-native writer: **when in doubt, make two
short sentences.** Short, clear sentences are easier to write correctly and easier
to read than long joined ones — so if you're unsure how to connect two thoughts,
just use a period. You rarely go wrong with shorter sentences.

---

## How It Works

### The comma splice and its three fixes

A comma alone cannot join two complete sentences (two things that could each stand
alone). Three ways to fix it:

1. **Period** (make two sentences): "The API is slow. We should add caching." —
   the safest, clearest choice.
2. **Joining word** (comma + and/but/so/because/although...): "The API is slow, **so**
   we should add caching." — connects the ideas and shows the relationship.
3. **Semicolon** (for closely-related ideas): "The API is slow; caching would
   help." — correct but formal; use sparingly in chat.

### The joining words and what they signal

The small joining words carry meaning — they tell the reader how the two ideas
relate:

- **and** — addition ("I fixed the bug **and** added a test")
- **but** — contrast ("It works locally **but** fails in CI")
- **so** — result ("The server was down **so** requests timed out")
- **because** — reason ("It failed **because** the config was wrong")
- **although / though** — concession ("It's slow, **although** it works")

Note the comma placement: with *and/but/so* joining two full sentences, put the
comma *before* the joining word ("It works locally, **but** it fails in CI"). With
*because*, usually no comma when it comes in the middle ("It failed **because** the
config was wrong").

### Starting with But / So / And is fine

You may have learned "never start a sentence with *but*, *so*, or *and*." In modern
writing — especially chat — this is fine and natural: "The deploy failed. **But** I
found the cause." / "**So** here's the plan." Don't avoid it; it often makes writing
clearer and more conversational (which is good for Slack).

### The default: shorter is safer

When you're unsure how to connect ideas, **break them into shorter sentences.**
Long sentences with multiple joined clauses are where run-ons, comma splices, and
dropped words happen — and they're harder to read. Two clean short sentences almost
always beat one long tangled one. This is the single best habit for clear
writing as a non-native speaker: prefer short sentences.

{: .note }
> **The two-second run-on check**
> Before sending a longer message, look for sentences with a comma in the middle
> and ask: "are both sides complete sentences on their own?" If yes, and they're
> joined by only a comma, it's a comma splice — fix it with a period, a joining
> word (and/but/so), or a semicolon. And the easy default: if a sentence feels
> long or tangled, split it into two shorter ones — you almost never make writing
> worse by shortening it.

---

## Lab — Rewrite Drill

Each message is a run-on or comma splice. Untangle each into clear, correct
sentences — usually the simplest fix is a period or a joining word.

**Your rewrite (fix each):**

1. "I deployed the fix, it broke the build, I'm rolling back now."
2. "The API is slow, we should add caching, the users are complaining."
3. "I checked the logs, the error is a timeout, the server was overloaded."
4. "It works on my machine, it fails in CI, I'm not sure why yet."
5. "Can you review the PR, I need to merge it today, it's blocking the release."

<details>
<summary>Show Model Rewrites</summary>
<br>
<strong>1.</strong> "I deployed the fix, it broke the build, I'm rolling back now."
→ <strong>"I deployed the fix, but it broke the build. I'm rolling back now."</strong>
(Three complete thoughts glued by commas. Use "but" to show the contrast between the
first two, then a period before the third. Or all periods: "I deployed the fix. It
broke the build. I'm rolling back now." — also clear.)
<br><br>
<strong>2.</strong> "The API is slow, we should add caching, the users are
complaining." → <strong>"The API is slow and users are complaining, so we should add
caching."</strong> (Shows the relationships: slow + complaints → therefore caching.
Or simpler: "The API is slow and users are complaining. We should add caching.")
<br><br>
<strong>3.</strong> "I checked the logs, the error is a timeout, the server was
overloaded." → <strong>"I checked the logs — the error is a timeout because the
server was overloaded."</strong> (The last two are cause-and-effect: timeout
<em>because</em> overloaded. Or: "I checked the logs. The error is a timeout — the
server was overloaded.")
<br><br>
<strong>4.</strong> "It works on my machine, it fails in CI, I'm not sure why yet."
→ <strong>"It works on my machine but fails in CI. I'm not sure why yet."</strong>
("but" for the contrast (works vs fails), period before the third thought. Note you
can drop the repeated "it": "works on my machine but fails in CI.")
<br><br>
<strong>5.</strong> "Can you review the PR, I need to merge it today, it's blocking
the release." → <strong>"Can you review the PR? I need to merge it today — it's
blocking the release."</strong> (The first is a question (needs "?" not a comma).
Then the reason: "I need to merge it today because it's blocking the release" also
works.)
<br><br>
The pattern across all five: when you have multiple complete thoughts, don't glue
them with commas — use periods (make short sentences), or joining words (and/but/so/
because) that show how the ideas relate. And notice that the shorter, period-
separated versions are perfectly clear — when in doubt, just use periods.
</details>

---

## Phrase Bank

The joining words and what each signals:

| To show… | Use | Example |
|---|---|---|
| Addition | **and** | "I fixed the bug **and** added a test." |
| Contrast | **but** | "It works locally **but** fails in CI." |
| Result | **so** | "The server was down, **so** requests failed." |
| Reason | **because** | "It failed **because** the config was wrong." |
| Concession | **although / though** | "It's slow, **though** it works." |
| Two full sentences | **. (period)** | "The API is slow. We should add caching." |

---

## Further Reading

| Topic | Source |
|---|---|
| Comma splices and run-ons | Purdue OWL — <https://owl.purdue.edu/owl/general_writing/punctuation/independent_and_dependent_clauses/index.html> |
| Coordinating conjunctions (and/but/so) | British Council — <https://learnenglish.britishcouncil.org/grammar/> |
| Sentence variety and length | *The Sense of Style*, Steven Pinker |

---

## Checkpoint

**Q1.** What is a comma splice, and what are the three ways to fix it? Which is the
safest default for a non-native writer?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A comma splice is joining two <strong>complete sentences</strong> (two thoughts that
could each stand alone) with only a <strong>comma</strong> — for example, "I deployed
the fix, it broke the build," where "I deployed the fix" and "it broke the build" are
each complete sentences, so a comma alone can't hold them together. The three fixes:
(1) <strong>Period</strong> — make two separate sentences: "I deployed the fix. It
broke the build." (2) <strong>Joining word</strong> — comma + a conjunction that
shows the relationship (and/but/so/because): "I deployed the fix, <strong>but</strong>
it broke the build." (3) <strong>Semicolon</strong> — for closely related ideas: "I
deployed the fix; it broke the build" (correct but formal — use sparingly in casual
chat). The safest default for a non-native writer is the <strong>period — make two
shorter sentences</strong>. It's the choice you can almost never get wrong: two
clear, short, complete sentences are easy to write correctly, easy to read, and avoid
the run-on/splice/dropped-word problems that longer joined sentences invite. The
joining-word option (fix #2) is also good and often better because it shows how the
ideas relate (but = contrast, so = result), so use it when you know the relationship
you want to signal. But when in doubt — when you're unsure how to connect two thoughts
correctly — just use a period and write two short sentences. You rarely make writing
worse by shortening it, and shorter sentences are one of the single best habits for
clear writing as a non-native speaker: they're easier to get right and easier to
read. The semicolon (fix #3) is correct but formal and easy to misuse, so it's the
least necessary for everyday work writing — a period does the same job more simply.
</details>

**Q2.** Is it wrong to start a sentence with "But," "So," or "And"? What's the
practical guidance for workplace/Slack writing?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>No, it's not wrong</strong> — despite the old school rule that you should
"never start a sentence with a conjunction," starting sentences with But, So, and
And is completely acceptable in modern writing, and especially natural in workplace
and Slack communication. The old rule was a simplification taught to young students
(to stop them writing endless "and... and... and..." sentences), not an actual rule
of English — good writers have always started sentences with these words, and style
guides explicitly permit it. The practical guidance for workplace/Slack writing:
<strong>use it freely — it often makes your writing clearer and more
conversational</strong>. Starting with "But" cleanly signals a contrast at the start
of a new thought ("The deploy failed. But I found the cause." — clearer and punchier
than forcing it into one sentence). Starting with "So" naturally introduces a result
or a conclusion ("So here's the plan" / "So we should roll back"). Starting with
"And" adds a related point. These make your writing flow the way people actually
think and talk, which suits the conversational tone of Slack. The benefit for a
non-native writer specifically: it lets you break ideas into shorter sentences (the
recommended default from Q1) while still showing how they connect — instead of
struggling to join two thoughts into one correct long sentence, you can write two
short sentences and start the second with But/So/And to show the relationship. So
"It works locally. But it fails in CI." is a perfectly good (and easy) alternative
to "It works locally but it fails in CI." — both are correct, and the two-sentence
version is often easier to get right. The one caveat: don't overuse it (starting
every sentence with "And" gets repetitive), and it's slightly informal (fine for
Slack and most work writing, occasionally worth avoiding in very formal documents).
But for everyday workplace communication, starting sentences with But/So/And is
natural, clear, and encouraged — don't avoid it out of a rule that isn't real.
</details>

---

## Homework

Find three of your recent longer messages. Check each for comma splices (two complete
sentences joined by only a comma) and run-ons (multiple thoughts glued together). Fix
any by splitting into shorter sentences or adding a joining word (and/but/so/because).
Then, as an experiment, take your longest recent message and rewrite it as several
short sentences — notice whether it's clearer. Add to your checklist: do you tend to
run sentences together (comma splices), and does "just use a period" help you?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the run-on/comma-splice check and, more importantly, the
short-sentence habit that's one of the highest-value moves for a non-native writer. A
strong response finds any comma splices (two complete sentences joined by only a
comma — "I did X, it broke Y") and fixes them, and the rewrite-as-short-sentences
experiment usually reveals that the shorter version is clearer, easier to read, and
easier to have written correctly. The common realization: <strong>your longer
sentences are where most of your errors hide</strong> — comma splices, run-ons,
dropped words (Lesson 1), agreement errors in the distance trap (Lesson 5) all
cluster in long, multi-clause sentences, while short sentences are simply easier to
get right. So the single best habit that comes out of this lesson is
<strong>preferring shorter sentences</strong>: when you have several thoughts, use
periods to make several short sentences rather than trying to join them all into one
long correct sentence. This isn't a stylistic preference — for a non-native writer
it's an accuracy strategy: shorter sentences have fewer places to go wrong, and
"just use a period" is a reliable default whenever you're unsure how to connect
ideas. The joining words (and/but/so/because) are worth using when you want to show
how ideas relate (and you can start a sentence with them — Q2 — which lets you keep
sentences short while still connecting them), but the foundational move is not being
afraid to use periods. Add your pattern to the checklist — if you tend toward comma
splices and run-ons, the checklist item is "any long sentence: could this be two
short sentences? and is any comma actually joining two complete sentences?" The
reassuring meta-point: you don't need to master complex sentence construction to
write well at work — clear short sentences are not just acceptable, they're often
<em>better</em> than long complex ones (even for native speakers, plain short
sentences are a hallmark of good writing), so leaning into shorter sentences plays to
your strengths rather than exposing your weaknesses. This is the last of the core
mechanics; the next lesson covers punctuation for controlling tone in chat, and then
the error clinic (Lesson 9) assembles your personal checklist from everything you've
identified across Phase 1.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 8 — Punctuation for Chat →](lesson-08-punctuation-chat){: .btn .btn-primary }
