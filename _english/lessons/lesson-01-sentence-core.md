---
title: "Lesson 01 — The Sentence Core"
nav_order: 1
parent: "Phase 1: Sentence Mechanics"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 01: The Sentence Core

## Concept

Every clear English sentence has a simple skeleton: **one subject, one main verb,
and (usually) an object.** The subject is who or what does the action; the verb is
the action; the object is what the action is done to.

```
   SUBJECT   +   VERB     +   OBJECT
   ────────      ────         ──────
   I             deployed     the fix.
   The tests     are failing.
   The build     broke        the pipeline.
   who/what      the action   what receives it
```

English carries meaning through **word order** — unlike many languages, English
usually keeps subject → verb → object. "The bug broke the build" and "The build
broke the bug" mean different things, only because of the order.

When you write quickly (especially in Slack), two problems appear: (1) a
**fragment** — a "sentence" missing its verb or subject, so it isn't a complete
thought; and (2) words that **drop out** — the subject or verb goes missing
because you were thinking faster than you typed. The fix is a two-second habit:
before you hit send, find the *subject* and the *verb*. If you can't find both,
the sentence isn't finished.

---

## Going Deeper

### Find the subject and the verb

The reliable check for any sentence: ask **"who or what does what?"** If you can
answer both — who/what (subject) and does what (verb) — the sentence has its core.
If either is missing, it's a fragment.

- "Fixed the login bug." → *Who* fixed it? The subject is missing. In chat this
  is sometimes OK (see the note), but in most writing you need "**I** fixed the
  login bug."
- "The deployment to production." → This *does what*? There's no verb — it's just
  a noun phrase, not a sentence. → "The deployment to production **failed**."
- "Because the server was down." → This is only half a thought (a *because* clause
  needs a main clause). → "The request timed out **because the server was down**."

### Word order carries the meaning

Because English relies on order, keep subject → verb → object and don't let the
pieces scatter. Non-native writers sometimes move the verb or object to a
position that another language allows but English doesn't:

- ✗ "I to the meeting will come late." → ✓ "I will come late to the meeting."
- ✗ "Yesterday fixed I the bug." → ✓ "I fixed the bug yesterday." (time words like
  *yesterday* go at the start or end, not between subject and verb.)

### The dropped-word trap

When you type fast, small but essential words disappear — often the subject
(*I*, *it*, *we*), the verb *to be* (*is*, *are*, *was*), or the article. Your
brain filled them in; the reader's doesn't. This is one of your highest-value
things to catch, so the pre-send check specifically looks for a missing subject
or a missing verb.

{: .note }
> **When fragments are fine (and when they're not)**
> In casual Slack, short fragments are normal and natural: "On it!", "Done.",
> "Good catch." — nobody expects a full sentence for a quick reply, and forcing
> one would sound stiff. The rule isn't "always write full sentences"; it's
> "when you're making a real statement, make sure it has a subject and a verb."
> "Fixed it" as a quick reply is fine; "The reason for the outage" as your
> explanation of what happened is a fragment that leaves the reader waiting for
> the rest.

---

## Lab — Rewrite Drill

Here are ten real Slack-style messages with fragments or word-order errors. For
each, write a corrected version, then reveal the model rewrite.

**Your rewrite (fix each):**

1. "The reason the tests are failing."
2. "Will deploy after lunch."
3. "Because the API is slow."
4. "I to you send the doc now."
5. "Fixed. The bug in the payment flow which was causing the errors."
6. "The meeting moved I think to 3pm?"
7. "Not sure why happening this."
8. "The new feature almost done, just need review."
9. "Yesterday deployed we the hotfix and everything working now."
10. "Can you the PR review when you get a chance?"

<details>
<summary>Show Model Rewrites</summary>
<br>
<strong>1.</strong> "The reason the tests are failing" → <strong>"The tests are
failing because the database connection times out."</strong> (The original is a
fragment — a noun phrase with no main verb about the reason. Give it a verb, or
just state the point directly.)
<br><br>
<strong>2.</strong> "Will deploy after lunch." → <strong>"I'll deploy after
lunch."</strong> (Missing subject — <em>who</em> will deploy? In casual chat "will
deploy after lunch" can pass, but adding "I'll" is clearer and more natural.)
<br><br>
<strong>3.</strong> "Because the API is slow." → <strong>"The page loads slowly
because the API is slow."</strong> (A <em>because</em> clause is only half a
thought — it needs a main clause. What is happening because the API is slow?)
<br><br>
<strong>4.</strong> "I to you send the doc now." → <strong>"I'll send you the doc
now."</strong> (Word order — English is subject → verb → object: "I send you," not
"I to you send.")
<br><br>
<strong>5.</strong> "Fixed. The bug in the payment flow which was causing the
errors." → <strong>"Fixed — the bug in the payment flow was causing the
errors."</strong> or <strong>"I fixed the payment-flow bug that was causing the
errors."</strong> (The second "sentence" is a fragment: "The bug... which was
causing the errors" has no main verb — <em>which was causing</em> is a
description, not the main statement. Give the bug a main verb, or attach it to
"Fixed.")
<br><br>
<strong>6.</strong> "The meeting moved I think to 3pm?" → <strong>"I think the
meeting moved to 3pm?"</strong> (Word order — "I think" belongs at the front, not
dropped into the middle. Keep the core "the meeting moved to 3pm" together.)
<br><br>
<strong>7.</strong> "Not sure why happening this." → <strong>"I'm not sure why
this is happening."</strong> (Missing subject "I", missing verb "is", and word
order — "why this is happening," not "why happening this." A dropped-word classic:
"I'm" and "is" both vanished.)
<br><br>
<strong>8.</strong> "The new feature almost done, just need review." →
<strong>"The new feature is almost done — it just needs a review."</strong>
(Missing verb "is" — "the new feature is almost done." And "just need review"
dropped its subject and article: "it just needs a review.")
<br><br>
<strong>9.</strong> "Yesterday deployed we the hotfix and everything working now."
→ <strong>"We deployed the hotfix yesterday, and everything is working now."</strong>
(Word order — "We deployed... yesterday," not "Yesterday deployed we." And missing
verb "is" — "everything is working.")
<br><br>
<strong>10.</strong> "Can you the PR review when you get a chance?" → <strong>"Can
you review the PR when you get a chance?"</strong> (Word order — the verb "review"
must come before its object "the PR": "review the PR," not "the PR review." A very
common pattern to watch — in English the verb comes before what it acts on.)
<br><br>
The theme across all ten: find the <em>subject</em> and the <em>verb</em>, keep
them in order (subject → verb → object), and make sure neither dropped out. Reading
your message back once, asking "who does what?", catches nearly all of these.
</details>

---

## Phrase Bank

Complete, correct sentence cores you can reuse — notice each has a clear subject
and verb:

| Instead of a fragment… | Write the full core |
|---|---|
| "The reason for X." | "X happened because [reason]." |
| "Almost done." | "It's almost done — I just need to [X]." |
| "Not sure why." | "I'm not sure why this is happening." |
| "Will do after lunch." | "I'll do it after lunch." |
| "Fixed the bug." (as a real statement) | "I fixed the bug in [area]." |
| "The PR review can you do?" | "Can you review the PR?" |

---

## Further Reading

| Topic | Source |
|---|---|
| Sentence structure (subject–verb–object) | <https://en.wikipedia.org/wiki/Subject%E2%80%93verb%E2%80%93object_word_order> |
| Sentence fragments | Purdue OWL — <https://owl.purdue.edu/owl/general_writing/mechanics/sentence_fragments.html> |
| English word order | British Council — <https://learnenglish.britishcouncil.org/grammar/> |
| Plain, clear sentences | *The Sense of Style*, Steven Pinker |

---

## Checkpoint

**Q1.** What are the two things you should check for before sending any real
statement, and what's the one question that finds them both?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Check that the sentence has a <strong>subject</strong> (who or what) and a
<strong>main verb</strong> (the action) — the two pieces of the sentence core. The
one question that finds them both: <strong>"who or what does what?"</strong> If you
can answer both halves (who/what = the subject, does what = the verb), the sentence
has its core and is a complete thought. If you can't find one of them, it's a
fragment — a word or two dropped out, or you wrote a description instead of a
statement. This two-second check, done before you hit send, catches the most common
accuracy problems: missing subjects (especially "I", "it", "we"), missing verbs
(especially "is/are/was"), and fragments that leave the reader waiting for the rest
of the thought. (Quick replies like "On it!" are the fine exception — the rule is
for real statements, where a missing subject or verb genuinely confuses the reader.)
</details>

**Q2.** Rewrite this and explain the fix: "The deployment to the staging server
which failed twice already."

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Rewrite: <strong>"The deployment to the staging server failed twice
already."</strong> The fix: the original is a <strong>fragment</strong> — it looks
long, but it has no main verb making a statement. "The deployment to the staging
server" is the subject, and "which failed twice already" is only a <em>description</em>
of it (a relative clause), not the main statement — so the whole thing is just a big
noun phrase, leaving the reader waiting for "...which failed twice already [did
what?]." The fix is to turn the description into the main verb: remove "which" so
"failed" becomes the sentence's main verb — "The deployment... <strong>failed</strong>
twice already." Now it's a complete thought (subject: the deployment; verb: failed).
This is a common trap: a "which/that" clause can make a fragment feel like a full
sentence because it's long and has a verb inside it — but that verb belongs to the
description, not to the main statement. The check catches it: "who or what does
what?" — the deployment does... nothing, in the original (failed is trapped inside
"which failed"); removing "which" frees the verb to be the main action.
</details>

---

## Homework

For one day, before you send each Slack message that makes a real statement (not a
quick "ok" or "thanks"), pause for two seconds and find the subject and the verb.
Notice how often a word almost dropped out — a missing "I", "is", or "it". Keep a
small tally of how many you caught. Then write down the two or three patterns you
catch yourself doing most (dropped subject? dropped "is/are"? verb in the wrong
place?) — these become the first entries in your personal error checklist (Lesson
9).

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the pre-send habit that catches most sentence-core errors, and
the tally makes visible how often small essential words nearly drop out when you
type fast — which is usually more than people expect. The value is twofold. First,
the <strong>habit</strong>: pausing to find the subject and verb before sending
becomes automatic with practice, and it's the single highest-return editing move
for accuracy (it catches fragments, dropped subjects, and dropped verbs — the most
common and most confusing errors). Second, the <strong>self-diagnosis</strong>:
noticing <em>your</em> specific recurring patterns (many non-native speakers find
they drop "is/are/was" — the verb <em>to be</em> — most often, or drop the subject
"I/it/we", or occasionally put the verb after its object as in "the PR review can
you") tells you exactly what to watch for, which is far more useful than a generic
grammar rule. These patterns become the foundation of your personal error checklist
(Lesson 9) — the point of this whole phase is not to memorize every grammar rule,
but to identify <em>your</em> handful of recurring errors and build a quick
pre-send check for them. Most people have only 3–5 patterns they repeat, so
catching those handles the large majority of their errors. The tally also shows
progress: as the habit builds, you'll catch errors before sending rather than after
(or never), and your sent messages get noticeably more accurate — the whole goal of
Phase 1. If you found you rarely dropped words, great — your sentence cores are
already solid, and you can focus your checklist on other patterns (articles, tenses)
that later lessons cover. The meta-point: accurate writing isn't about knowing more
rules, it's about a fast, reliable self-check for the few errors you actually make —
and this homework starts building both the check and the knowledge of what to check
for.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 2 — Articles I: a/an vs the →](lesson-02-articles-a-the){: .btn .btn-primary }
