---
title: "Lesson 23 — Asking for Help"
nav_order: 3
parent: "Phase 4: Slack Communication"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 23: Asking for Help

## Concept

Everyone needs help sometimes — and *how* you ask determines both how easily people can
help you and how professional you look. A good help request makes it easy to help you
(you've given the right information) and shows you tried first (you look capable, not
lost). A bad one dumps a vague problem and makes the helper do all the work.

```
   BAD ASK (vague, effortless)         GOOD ASK (easy to help)
   ───────────────────────────         ───────────────────────
   "the deploy isn't working,           "Deploy failing with a timeout error.
    can you help?"                       What I've tried: restarted (failed
                                         again), checked the DB (fine). Any
                                         idea what else to look at? 🙏"

   THE SHAPE:  goal · what I tried · exact error · specific question
```

The good-ask shape: state your **goal** (what you're trying to do), **what you've
tried** (shows effort + saves the helper suggesting things you did), the **exact
error/symptom** (specific, not "it's broken"), and a **specific question** (what
exactly you need). This makes helping you fast and easy — and makes you look like
someone who did their homework.

---

## How It Works

### The help-request shape

Include these, briefly:

- **Goal**: what you're trying to accomplish ("I'm trying to deploy the payment
  service to staging").
- **What you tried**: the things you've already attempted ("I've restarted it and
  checked the logs") — this shows effort and prevents the helper suggesting what you
  did.
- **Exact error/symptom**: the specific error message or behavior, ideally pasted in a
  code block ("it fails with `ConnectionTimeout: could not reach db:5432`") — not "it's
  broken."
- **Specific question**: what exactly you need ("any idea what would cause the timeout,
  or where else I should look?").

This gives the helper everything they need to help quickly, instead of a vague problem
they have to interrogate you about.

### Don't "ask to ask"

Don't send "can I ask you a question?" and wait — just ask the question (with context).
"Can I ask you something?" makes the person reply "sure" and wait, wasting a round-trip
(same as the no-hello rule, Lesson 21). Ask the actual question directly:

- ✗ "Hey, can I ask you a quick question?" [waits] → ✓ "Hey! Quick question — do you
  know why the staging deploy might time out? [context]"

### Timebox before asking (and say so)

Show you tried before asking — spend a reasonable amount of time attempting it yourself
first, and mention that you did: "I've been debugging this for ~30 min and I'm stuck —"
signals you didn't immediately run for help, which is professional. (But don't
over-timebox: struggling alone for hours on something someone could answer in two
minutes is also a mistake — there's a balance between trying first and wasting time.)

### Ask in public channels when you can

Prefer asking in a public/team channel over a DM (when appropriate): others can help
(not just the one person), the answer helps anyone with the same question later
(searchable), and it doesn't put all the burden on one person. DM for genuinely
person-specific or sensitive things; public channel for general technical questions.

### Close the loop

After you get help, **close the loop**: say it worked and, ideally, what the answer was
("That fixed it — it was a missing env var. Thanks so much! 🙏"). This thanks the helper
(Lesson 19), confirms the resolution, and — in a public channel — leaves the answer for
the next person. Not closing the loop leaves the helper wondering if they helped and
loses the searchable answer.

{: .note }
> **A good ask makes you look capable, not lost</br>**
> Asking for help well doesn't make you look incapable — the opposite. A well-shaped
> ask (goal, what you tried, exact error, specific question) shows you did your
> homework, thought about the problem, and know exactly what you're stuck on — which
> reads as capable and professional. A vague ask ("it's broken, help") makes you look
> lost and makes the helper do all the diagnostic work. So don't avoid asking (out of
> fear of looking incapable) or ask badly (dumping a vague problem) — ask <em>well</em>,
> and you both get help faster and look good doing it. And asking well is warm too: it
> respects the helper's time by making it easy for them.

---

## Lab — Rewrite Drill

Rewrite these two bad help requests into good ones (invent plausible details). Note
what each was missing.

**Your rewrite (fix each):**

1. **Zero context:** "hey the tests are failing, do you know why?"
2. **Life-story version (too much, no clear ask):** "So I've been trying to get the local dev environment set up for the past two hours, first I cloned the repo and then I tried to run the setup script but it gave me an error, so I looked it up online and tried a few things from Stack Overflow, and then I updated my dependencies, and then I tried again and got a different error, and I've been really struggling with this and I'm not sure what to do, it's been so frustrating..."

<details>
<summary>Show Model Rewrites</summary>
<br>
<strong>1. Zero context → good ask:</strong> "Hey! I'm getting test failures on the
payment module and I'm stuck — could you help? 🙏 The failing test is
<code>test_refund_flow</code>, and the error is <code>AssertionError: expected 200,
got 500</code>. I've checked that the DB is seeded and the other tests pass. Any idea
what might cause just this one to 500?" (Added: the goal (payment module tests), the
specific failing test and exact error (in code format), what I tried (checked DB, other
tests pass), and a specific question. Now the helper can actually help — the original
gave them nothing to work with.)
<br><br>
<strong>2. Life-story → good ask:</strong> "Hey! I'm stuck setting up the local dev
environment (been at it ~2 hours) — could you help? 🙏 The setup script fails with:
<code>Error: cannot find module 'X'</code>. I've tried updating dependencies and a
couple of Stack Overflow suggestions, no luck. Have you seen this before, or is there a
setup step I might be missing?" (Cut the long narrative to the essentials: goal (dev
env setup), timeboxed effort (~2 hours — shows I tried), the exact current error (in
code), what I tried (deps, SO), and a specific question. The original had all the
frustration but no clear error or ask — the helper couldn't act on it.)
<br><br>
The two failures fixed: #1 had <em>no context</em> (no goal, error, or what-was-tried —
the helper would have to interrogate you), and #2 had <em>too much narrative but no
clear error or ask</em> (a wall of frustration the helper can't act on). Both become
good asks with the shape: goal + what I tried + exact error + specific question — brief,
specific, and easy to help with.
</details>

---

## Phrase Bank

| Element | Phrase |
|---|---|
| State the goal | "I'm trying to [X] and I'm stuck —" |
| Show effort (timebox) | "I've been at this for ~[time] —" |
| The exact error | "the error is: ``` [paste it] ```" |
| What you tried | "What I've tried: [X], [Y] — no luck." |
| Specific question | "Any idea what would cause [X]?" / "Where else should I look?" |
| Close the loop | "That fixed it — it was [cause]. Thanks so much! 🙏" |

---

## Further Reading

| Topic | Source |
|---|---|
| How to ask good questions | "How To Ask Questions The Smart Way" — <http://www.catb.org/~esr/faqs/smart-questions.html> |
| Don't ask to ask | <https://dontasktoask.com/> |
| Closing the loop / thanking | English track Lesson 19 (thanks) |

---

## Checkpoint

**Q1.** What's the shape of a good help request, and why does asking well make you look
*more* capable rather than less?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The shape of a good help request has four brief parts: (1) <strong>the goal</strong> —
what you're trying to accomplish ("I'm trying to deploy the payment service to
staging"); (2) <strong>what you've tried</strong> — the things you've already attempted
("I've restarted it and checked the logs"), which shows effort and prevents the helper
from suggesting things you already did; (3) <strong>the exact error or symptom</strong>
— the specific error message or behavior, ideally pasted in a code block ("it fails with
<code>ConnectionTimeout: could not reach db:5432</code>"), not a vague "it's broken";
and (4) <strong>a specific question</strong> — exactly what you need ("any idea what
would cause the timeout, or where else I should look?"). Together, these give the helper
everything they need to help you <em>quickly</em>, rather than a vague problem they have
to interrogate you about (what error? what have you tried? what exactly do you need?).
Why asking well makes you look <em>more</em> capable rather than less: many people avoid
asking for help, or ask vaguely, because they fear that needing help makes them look
incapable or lost. But a well-shaped ask signals the opposite — it demonstrates that you
(a) <strong>understood the problem</strong> (you can state the goal and the specific
error clearly, which requires actually grasping what's happening), (b) <strong>tried to
solve it yourself</strong> (the "what I've tried" shows you didn't just immediately run
for help — you attempted it, which is professional and capable), and (c) <strong>know
exactly what you're stuck on</strong> (the specific question shows you've isolated the
gap in your understanding, not that you're generally lost). This reads as capable,
thoughtful, and professional — someone who did their homework and hit a specific
obstacle, not someone flailing. A <em>vague</em> ask ("the deploy isn't working, can you
help?"), by contrast, makes you look lost (you can't even state the problem specifically)
and makes the helper do all the diagnostic work (they have to extract the goal, the
error, what you tried — turning a quick answer into a lengthy interrogation). So the
counterintuitive truth is that asking <em>well</em> improves how you're perceived: it
shows competence and effort, while asking badly (or avoiding asking and struggling
uselessly) is what actually looks bad. This matters because it removes the fear that
drives two bad behaviors — avoiding asking (struggling alone for hours on something
someone could answer in two minutes, wasting time and looking like you can't collaborate)
and asking vaguely (dumping an unstructured problem). The professional move is to ask
<em>well</em>: try first (timebox), then ask with the full shape (goal, what you tried,
exact error, specific question), which gets you help fast AND makes you look capable. And
it's warm/considerate too — a well-shaped ask respects the helper's time by making it
easy for them, rather than making them do all the work. So there's no downside to asking
well: faster help, better impression, and kinder to the helper.
</details>

**Q2.** Why is it better to ask in a public channel than a DM (when appropriate), and
why should you "close the loop" after getting help?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Asking in a public channel (vs a DM) is better, when appropriate, for several
reasons:</strong> (1) <strong>More people can help</strong> — in a public/team channel,
anyone who knows the answer can respond, not just the one person you'd DM; this often
gets you a faster answer (someone available/knowledgeable jumps in) and doesn't depend
on a single person being free. (2) <strong>The answer helps others later</strong> — a
public question and its answer are searchable, so the next person with the same problem
finds it (rather than asking again); over time, this builds a useful searchable
knowledge base and reduces repeated questions. (3) <strong>It distributes the burden</strong>
— DMing one person puts all the load on them (and if they're busy, you're stuck); asking
publicly spreads it, and nobody feels singled out or obligated. (4) <strong>It's more
transparent</strong> — public questions and answers keep knowledge visible to the team
rather than siloed in DMs. So the default for general technical questions should be a
public channel; reserve DMs for genuinely person-specific things (only that person can
answer), sensitive matters, or when you specifically need one person. (Some people default
to DMs out of a feeling that public questions are "bothering everyone" or exposing
ignorance — but a good public question is normal, helps others, and doesn't single anyone
out; the norm on healthy teams is public-by-default.) <strong>Why close the loop after
getting help:</strong> "closing the loop" means, after the help solves your problem,
posting that it worked and ideally what the answer was ("That fixed it — it was a missing
env var. Thanks so much! 🙏"). This matters for several reasons: (1) <strong>It thanks the
helper</strong> (Lesson 19) — acknowledging that their help worked is warm and
appreciative, and it's satisfying for them to know they helped (rather than being left
wondering if their suggestion worked). (2) <strong>It confirms the resolution</strong> —
it closes the thread clearly (the problem is solved), so nobody wonders if you're still
stuck or keeps trying to help. (3) <strong>In a public channel, it leaves the answer for
the next person</strong> — the searchable thread now includes both the question AND the
solution, so someone with the same problem later finds a complete answer (a question with
no recorded resolution is much less useful — the searcher finds the question but not the
fix). (4) <strong>It's good collaborative citizenship</strong> — closing loops keeps
channels clean and useful, and it's the courteous, professional thing to do. Not closing
the loop (getting help, solving the problem, and going silent) leaves the helper
unsure if they helped, leaves the thread ambiguous, and — in public — loses the valuable
recorded answer. So closing the loop is a small but important habit: it thanks the helper,
confirms resolution, and preserves the answer for others. Together, these two practices —
ask publicly (when appropriate) and close the loop — make your help-seeking benefit the
whole team (more helpers, searchable answers) rather than just you, and they're warm and
considerate (thanking helpers, not burdening one person, leaving answers for others) —
which builds goodwill and a good collaborative reputation. For your goal, asking well +
publicly + closing the loop is both effective (you get help) and warm/professional (you
respect helpers' time, thank them, and contribute to the team's shared knowledge).
</details>

---

## Homework

Next time you need help, ask using the full shape: goal + what you tried + exact error
(in a code block) + specific question. Try first (timebox ~15–30 min), and mention you
did. Ask in a public channel if it's a general question. After you get help, close the
loop (say it worked and what the fix was, and thank them). Notice: do you get help faster
and more easily? And add to your checklist: "help request — goal, what I tried, exact
error, specific question; ask publicly; close the loop."

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the help-request skill, which affects both how easily you get help
and how professional you appear. A strong response uses the full shape (goal, what tried,
exact error in a code block, specific question), timeboxes and mentions the effort, asks
publicly for general questions, and closes the loop afterward. The common realizations:
(1) <strong>well-shaped asks get help dramatically faster</strong> — giving the helper the
goal, error, and what-you-tried means they can answer immediately rather than having to
extract those through back-and-forth; a vague ask turns a two-minute answer into a
ten-minute interrogation. (2) <strong>asking well makes you look capable, not lost</strong>
(Q1) — the fear that needing help looks bad is backwards; a good ask shows you understood
the problem, tried to solve it, and isolated exactly what you're stuck on, which reads as
competent. (3) <strong>public asking + closing the loop benefits the team</strong> — more
helpers, searchable answers, and good collaborative citizenship. The meta-point: asking
for help is a frequent and skill-dependent activity, and doing it well is a professional
superpower — you get unblocked faster, look capable, respect helpers' time, and contribute
to shared knowledge, all at once. It also removes two common failure modes: avoiding asking
(struggling uselessly for hours, which wastes time and looks like you can't collaborate)
and asking vaguely (which frustrates helpers and looks lost). The balance to strike is
try-first-then-ask-well: timebox a reasonable attempt (showing effort), but don't
over-timebox (struggling alone on something someone could answer quickly is also a mistake
— knowing when to ask is part of the skill). For your goal, asking well is also a warmth
point: a well-shaped ask respects the helper's time (making it easy for them), and closing
the loop thanks them and leaves value for others — so good help-seeking is both effective
and considerate, building goodwill and a reputation as a good collaborator. Add the
help-request checklist item. This lesson (asking well) pairs with the next one (answering
and unblocking others well) — the two sides of the help exchange — and both are core to
being an effective, warm collaborator on a team, which is much of what good Slack
communication is about.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 24 — Answering and Unblocking Others →](lesson-24-answering-unblocking){: .btn .btn-primary }
