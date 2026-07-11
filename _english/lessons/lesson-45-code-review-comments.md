---
title: "Lesson 45 — Code-Review Comments"
nav_order: 2
parent: "Phase 8: Feedback & Hard Conversations"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 45: Code-Review Comments

## Concept

Code review is where engineers give and receive feedback *daily* — so how you write review
comments shapes both the code and the team's culture more than almost anything else you write. A
good comment teaches, improves the code, and keeps the author motivated; a bad one (curt, nitpicky,
or condescending) demoralizes and breeds defensive, adversarial reviews. As a lead, your review
comments set the tone the whole team copies.

```
   BLUNT / STINGING              TEACHING / WARM
   ───────────────              ───────────────
   "This is wrong."             "I think this might miss the empty-list case —
                                 what happens if `items` is []? Maybe guard it?"
   "Don't do this."             "Small suggestion: extracting this into a helper
                                 would make it easier to test. Optional though!"
   "Why no tests??"             "Could we add a test for the new endpoint? Happy
                                 to help if useful."

   Explain the WHY · suggest, don't command · mark severity · stay kind
```

The reframe: **a code-review comment is a chance to teach and collaborate, not to judge** — the
same note framed as a shared improvement ("could we…", "what about…") rather than a verdict ("this
is wrong", "don't do this") gets the fix AND keeps the author learning and motivated. Small
wording choices make the difference between a review that builds people up and one that grinds them
down.

---

## How It Works

### Explain the *why*, not just the *what*

A comment that says "don't do this" teaches nothing; one that explains the reasoning helps the
author learn and agree. "This could deadlock if two threads hit it — consider a lock-free approach
or a single mutex" tells them *why*, so they understand and improve, and can apply it next time. The
why turns a correction into a lesson (the teaching-vs-solving idea, Lesson 24, in reviews).

### Suggest, don't command

Frame changes as suggestions/questions, not orders: "what about extracting this into a helper?" /
"could we guard the empty case here?" rather than "extract this" / "guard this." Suggestions invite
the author's judgment (they might have a reason, or a better idea) and feel collaborative;
commands feel like being bossed around. You're reviewing together, not dictating.

### Mark severity — nit vs blocking

Signal how much each comment matters, so the author can prioritize and doesn't treat every note as
a demand. Conventions: **`nit:`** (minor/style, optional), **`suggestion:`** (worth considering),
**`question:`** (just asking), **`blocking:`** (must resolve before merge). "nit: extra blank line
here" tells the author it's trivial; without the marker, they might agonize over it equally with a
real bug. Explicit severity respects their time and reduces friction.

### Praise the good, too

Note what's well done, not only what needs fixing: "nice clean solution here" / "good catch adding
that test." Positive comments make reviews feel collaborative rather than adversarial, encourage
good patterns, and make the critical comments easier to receive. A review that's all red marks
feels like an attack even if each is fair; sprinkling genuine praise balances it.

### Assume competence, ask don't accuse

Assume the author had reasons (they usually did): "was there a reason for the direct query here? I
might be missing context" beats "why didn't you use the ORM?" (which reads as an accusation).
Curiosity ("what's the thinking here?") is collaborative; interrogation ("why did you…?") is
adversarial. You're often the one missing context — asking respects that.

### Know when to move off the comment thread

Some things don't belong in written comments: a big design disagreement, a pile of back-and-forth,
anything getting tense. When a thread is going in circles or heating up, switch to a call or a
pairing session — "this is a bigger discussion, want to hop on a quick call?" Text amplifies
friction; a two-minute conversation resolves what ten comments can't. Knowing when to leave the
thread is a key review skill.

{: .note }
> **A review comment teaches and collaborates — it doesn't judge**
> Code review is daily feedback, so how you write comments shapes the code and the team's culture
> more than almost anything. The reframe: a comment is a chance to teach and improve together, not
> to judge — so explain the <em>why</em> (not just "don't do this"), suggest rather than command
> ("what about…?" not "do X"), mark severity (nit vs blocking, so not everything reads as a
> demand), praise the good (not only the problems), assume competence (ask, don't accuse), and move
> off the thread when it's tense or circular. Small wording choices — "this is wrong" vs "I think
> this might miss the empty case, what do you think?" — decide whether a review builds people up or
> grinds them down. As a lead, your comments set the tone the whole team copies, so warm, teaching
> comments are high-leverage: better code AND a healthier review culture.

---

## Lab — Rewrite Drill

Turn each harsh review comment into a warm, teaching one.

**Your answer:**

1. "This is wrong."
2. "Don't use a global here."
3. "Why are there no tests??"
4. "This whole approach is overcomplicated. Redo it." (a real design concern — but it's demoralizing
   and command-y; also consider whether it belongs in a comment at all)

<details>
<summary>Show Model Answers</summary>
<br>
<strong>1. →</strong> "I think this might miss the empty-list case — what happens if <code>items</code>
is <code>[]</code>? Looks like it'd throw on the <code>[0]</code> access. Maybe add a guard? (Let me
know if I'm misreading it.)" (Explains the specific issue and why, frames as a question, leaves room
to be wrong — versus a bare "this is wrong" that teaches nothing and stings.)
<br><br>
<strong>2. →</strong> "suggestion: could we avoid the global here and pass it in instead? It'd make
this easier to test and reason about. Not blocking, but I think it'd help." (Marks severity
(suggestion, not blocking), explains the why (testability), frames as "could we" — versus the curt
command "don't use a global here.")
<br><br>
<strong>3. →</strong> "Could we add a couple of tests for the new endpoint? Mainly the happy path
and the error case, I think. Happy to pair on it if that helps!" (Asks collaboratively, is specific
about what's needed, offers help — versus "why are there no tests??", which reads as an exasperated
accusation.)
<br><br>
<strong>4. →</strong> This probably shouldn't be a comment at all — a "redo the whole approach"
design concern is too big and too easily demoralizing for a review thread. Better: "I have some
bigger-picture thoughts on the overall approach here — I think there might be a simpler structure,
but it's more than a comment can cover. Could we hop on a quick call to talk it through? Want to
make sure I understand your reasoning too." (Moves the big disagreement off the thread to a
conversation, frames it as a shared exploration (not "you're wrong, redo it"), and respects that
they had reasons — versus a demoralizing "overcomplicated, redo it" verdict dropped in a comment.)
<br><br>
The patterns: explain the why, suggest/ask rather than command, mark severity, offer help, assume
reasons, and move big or tense things off the thread to a conversation.
</details>

---

## Phrase Bank

| Move | Phrase |
|---|---|
| Raise an issue (question) | "I think this might [issue] — what happens if…?" |
| Suggest | "suggestion: could we…?" / "what about…?" |
| Mark minor | "nit: …" / "totally optional, but…" |
| Mark important | "blocking: this needs [fix] before merge because…" |
| Explain the why | "…because [reasoning], which would help [benefit]." |
| Assume reasons | "Was there a reason for [X]? I might be missing context." |
| Praise | "Nice clean solution here!" / "Good call adding that test." |
| Move off-thread | "This is a bigger one — want to hop on a quick call?" |
| Offer help | "Happy to pair on this if useful." |

---

## Further Reading

| Topic | Source |
|---|---|
| Doc/written comments (foundation) | English track Lesson 43 (doc comments) |
| Code review culture | [Google's code review guidelines](https://google.github.io/eng-practices/review/) |
| Teaching through review | Leadership track Lesson 28 (code review as teaching) |

---

## Checkpoint

**Q1.** Why does how you write code-review comments matter so much for a team, and what are the
main techniques for comments that teach rather than sting?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
How you write code-review comments matters so much because <strong>code review is where engineers
give and receive feedback <em>daily</em>, so review comments shape both the code quality and the
team's culture more than almost anything else you write — and as a lead, your comments set the tone
the whole team copies</strong>. Several reasons: (1) <strong>frequency</strong> — reviews happen
constantly, so the cumulative effect of your comment style is huge; a small amount of harshness or
warmth per comment, multiplied across every review, adds up to a demoralizing or a supportive
culture; (2) <strong>it shapes the code</strong> — comments that teach (explain the why) help
authors learn and write better code over time, not just fix this one thing; comments that just
correct ("do X") fix the instance but don't build the author's skill; (3) <strong>it shapes team
culture and relationships</strong> — reviews are a recurring point of interpersonal contact, and
whether they feel collaborative (we're improving the code together) or adversarial (the reviewer is
judging and attacking) profoundly affects team morale, psychological safety, and how people feel
about their work; a culture of harsh reviews makes people defensive, afraid to submit code, and
resentful, while a culture of warm teaching reviews makes people eager to learn and safe to take
risks; (4) <strong>as a lead, you set the norm</strong> — the team copies how the lead reviews, so
your comment style propagates into the whole team's review culture (warm or harsh). So review
comments are exceptionally high-leverage — they affect code quality, individual growth, team
culture, and relationships, daily. The main techniques for comments that teach rather than sting:
(1) <strong>explain the why, not just the what</strong> — "consider a mutex here because two threads
could deadlock" teaches the reasoning, so the author understands, agrees, and learns for next time,
versus "don't do this" which teaches nothing; (2) <strong>suggest, don't command</strong> — "what
about extracting this into a helper?" invites the author's judgment and feels collaborative, versus
"extract this" which feels like being bossed; (3) <strong>mark severity</strong> — use "nit:",
"suggestion:", "question:", "blocking:" so the author can prioritize and doesn't treat every note as
an equal demand (a trivial style nit shouldn't weigh the same as a real bug); (4) <strong>praise
the good, too</strong> — noting what's well done ("nice clean solution") balances the critical
comments, makes reviews feel collaborative not adversarial, and makes critical notes easier to
receive; (5) <strong>assume competence, ask don't accuse</strong> — "was there a reason for X? I
might be missing context" (curious, collaborative) versus "why didn't you use the ORM?"
(accusatory), respecting that the author usually had reasons and you might lack context; and (6)
<strong>move off the thread when it's big or tense</strong> — take a major design disagreement or a
heating-up thread to a call, because text amplifies friction and a conversation resolves what many
comments can't. The common thread: <strong>treat a comment as a chance to teach and collaborate, not
to judge</strong> — explain, suggest, prioritize, praise, assume the best, and know when to talk
instead. These small wording choices ("this is wrong" vs "I think this might miss the empty case,
what do you think?") decide whether reviews build people up (better code, growth, healthy culture)
or grind them down (defensiveness, fear, resentment) — which, given how daily and formative reviews
are, makes warm teaching comments one of the highest-leverage communication habits for an engineer
and especially a lead.
</details>

**Q2.** Why is marking severity (nit / suggestion / blocking) and moving big or tense discussions
off the comment thread valuable?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Marking severity</strong> is valuable because <strong>it tells the author how much each
comment matters, so they can prioritize appropriately and don't experience every note as an equal
demand — which reduces both wasted effort and the feeling of being piled on</strong>. Without
severity markers, all comments look alike, so the author can't tell the difference between a trivial
style preference and a real must-fix bug — which causes problems: (1) they may spend equal energy on
a trivial nit and a critical issue (wasting time treating "extra blank line" as seriously as "this
deadlocks"); (2) they may feel obligated to address every comment (since each looks like a required
change), turning a review with a few real issues and some minor musings into what feels like a long
list of demands — demoralizing and time-consuming; or (3) they may push back on or agonize over
things you meant as take-it-or-leave-it. Marking severity fixes this: "nit:" signals trivial/
optional (fix if you want, no worries if not), "suggestion:" signals worth considering (your call),
"question:" signals just asking (not even a change request), and "blocking:" signals must-resolve-
before-merge. Now the author can prioritize — fix the blocking issues, weigh the suggestions, take
or leave the nits — spending their energy proportionally and not feeling overwhelmed by a wall of
seemingly-equal demands. It also (a) respects their time and judgment (you're trusting them to
handle nits as they see fit), (b) reduces friction (fewer things feel like fights when most are
explicitly optional), and (c) clarifies what actually gates the merge. So severity-marking makes
reviews more efficient and less adversarial. <strong>Moving big or tense discussions off the
comment thread</strong> is valuable because <strong>written comment threads are a poor medium for
big disagreements or heated exchanges — text amplifies friction and drags out what a conversation
would resolve quickly</strong>. Some things don't work well in comments: (1) <strong>a big design
disagreement</strong> — a fundamental "I think the whole approach should be different" is too large
and nuanced for a comment (it needs real back-and-forth, shared understanding of reasoning, exploring
options), and dropping it as a comment ("this is overcomplicated, redo it") is both demoralizing and
inadequate to the discussion; (2) <strong>a pile of back-and-forth</strong> — when a thread goes many
rounds without converging, each written exchange is slow and easily misread, and the disagreement
festers; (3) <strong>anything getting tense</strong> — text strips tone (Lesson 43), so a heating-up
thread escalates as each terse comment reads harsher than meant, and written back-and-forth entrenches
positions. In all these cases, <strong>a short conversation (a call, a pairing session) resolves what
many comments can't</strong>: real-time dialogue allows quick clarification, tone and warmth come
through (defusing tension), both sides' reasoning surfaces fast, and you reach resolution in minutes
instead of a long, friction-laden thread. Recognizing when to say "this is a bigger one — want to hop
on a quick call?" and leave the thread is a key review skill: it prevents the amplifying friction of
text on exactly the discussions most vulnerable to it, and it's often faster and always warmer.
Together, both techniques improve reviews: severity-marking makes the many small comments manageable
and non-overwhelming (so the routine review is efficient and low-friction), while moving big/tense
things off-thread handles the few discussions that text can't serve (so disagreements resolve quickly
and warmly instead of festering). Both reflect the core principle — reviews should build people up and
improve the code collaboratively, not grind people down — by managing the review's friction: keeping
small comments light and prioritized, and taking the heavy or heated ones to the medium (conversation)
that handles them best.
</details>

---

## Homework

For the next couple of weeks, upgrade every code-review comment you write: explain the why (not just
the what), frame changes as suggestions/questions (not commands), mark severity (nit / suggestion /
question / blocking), praise what's done well (not only problems), assume the author had reasons (ask,
don't accuse), and move any big or tense discussion to a call. Notice how the author responds when
reviews feel collaborative rather than adversarial. If you're a lead, notice the team copying your
tone. Add to your habits: "explained the why? suggested not commanded? marked severity? praised the
good? moved big/tense off-thread?"

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the code-review-comment skill — daily feedback that shapes code, growth, and
team culture more than almost anything you write. A strong response upgrades comments: explaining the
why, suggesting/asking rather than commanding, marking severity, praising the good, assuming reasons,
and moving big/tense discussions off-thread — noticing authors respond better to collaborative
reviews (and, for a lead, the team copying the tone). The realizations: (1) <strong>a comment is a
chance to teach and collaborate, not judge</strong> — explaining the why builds the author's skill,
suggesting invites their judgment, and the framing ("I think this might miss the empty case, what do
you think?" vs "this is wrong") decides whether the review builds people up or grinds them down; (2)
<strong>severity-marking and off-thread moves manage friction</strong> — marking nit vs blocking lets
authors prioritize and prevents the pile-on feeling, while taking big/tense discussions to a call
avoids text amplifying friction on exactly the exchanges most vulnerable to it; (3) <strong>reviews
shape culture, and the lead sets the tone</strong> — because reviews are daily and formative, and the
team copies the lead, warm teaching comments are exceptionally high-leverage for both code quality and
a healthy, safe review culture. The meta-point: code review is where feedback happens most, so
comment style has outsized impact on code, individual growth, morale, and relationships — and small
wording choices make the difference between collaborative and adversarial reviews. This applies the
feedback principles (Lesson 44) and the written-comment warmth (Lesson 43) to the specific daily
practice of code review. As a lead, modeling warm, teaching reviews propagates a healthy culture
across the team. Add the review checks to your habits. The next lesson goes to higher-stakes
territory — performance conversations — the formal, difficult feedback discussions about someone's
overall performance, where the stakes and the need for careful language are highest.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 46 — Performance Conversations →](lesson-46-performance-conversations){: .btn .btn-primary }
