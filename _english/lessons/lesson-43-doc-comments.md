---
title: "Lesson 43 — Doc Comments and Review Notes"
nav_order: 6
parent: "Phase 7: Design Docs & Proposals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 43: Doc Comments and Review Notes

## Concept

Commenting on someone's design doc or leaving notes on their work is some of the most *frequent*
writing you do — and it's where clarity and warmth meet. A good comment is clear about what you
mean and kind about how you say it; a blunt one ("this is wrong") lands as an attack even when
you're right. As a lead, your written comments carry weight and set the tone, so getting them
warm-and-clear matters a lot. (Code-review comments get their own lesson — 45 — this one is
about doc/design comments.)

```
   A BLUNT COMMENT              A WARM, CLEAR COMMENT
   ──────────────              ─────────────────────
   "This won't scale."         "One thing I'm wondering about: how does this
                                hold up under peak load? I'm worried the sync
                                call might bottleneck — could we consider a
                                queue here? What do you think?"

   Clear about the concern · warm about the delivery · a question, not a verdict
```

The reframe: **a written comment lacks tone of voice, so it reads harsher than you mean** — the
same words that would be fine said warmly in person can feel cold or critical in text. So written
comments need *extra* warmth to land as intended: soften, frame as questions, acknowledge what's
good, and be specific about the concern. Clear + warm is the goal for every comment.

---

## Going Deeper

### Written comments read harsher — add warmth

Text strips tone, so a neutral-in-your-head comment often reads as critical or curt to the
receiver. "This is wrong" — which you meant matter-of-factly — lands as harsh. Compensate by
adding warmth you might not need face-to-face: a softener, a question form, a bit of
acknowledgment. This isn't being fake; it's restoring the warmth the medium removed (Phase 3, in
writing). Err warmer than feels necessary.

### Frame concerns as questions

A question invites dialogue where a verdict invites defensiveness (Lesson 30, in writing):

- ✗ "This won't scale." (verdict — feels like an attack)
- ✓ "How does this handle peak load? I'm a bit worried the sync call could bottleneck." (question
  + concern — opens a conversation)

Questions also leave room for you to be wrong (maybe they've handled it) and make the author a
partner in solving it rather than a defendant.

### Acknowledge what's good

Don't comment only on problems — note what's strong too. "I really like the clean separation
here" / "this section is really clear" balances the critical notes, makes the author feel their
work is seen (not just picked at), and makes the critical comments easier to receive (they know
you're not just hunting for faults). A doc with only critical comments feels like an attack even
if each comment is fair.

### Be specific and actionable

Vague comments frustrate ("this is confusing" — what part? why?); specific ones help ("the term
'session' is used two different ways here — could you clarify which you mean in the second
paragraph?"). Point to the exact spot, explain the issue, and where possible suggest a direction.
Specific + actionable comments respect the author's time and actually move the doc forward.

### Distinguish blocking from optional

Signal how strongly you feel, so the author can prioritize: mark must-fix concerns versus
nice-to-haves versus pure opinion. Conventions like "**nit:**" (minor/optional), "**question:**"
(just asking), "**blocking:**" (needs resolving) or "take it or leave it" help the author know
what truly needs action versus what's a suggestion. Not every comment is a demand; say which are.

### Assume good intent and expertise

Write as if the author is smart and had reasons (they usually did). "Was there a reason for X? I
might be missing context" beats "why did you do X?" (which reads as an accusation). Assuming
competence — and that you might lack context — keeps comments collaborative and humble, and it's
often literally true (they knew something you didn't).

{: .note }
> **Written comments read harsher than you mean — so add warmth, and frame concerns as questions**
> Commenting on docs is frequent, weighty writing (as a lead, your comments set the tone), and its
> challenge is that text strips tone of voice — so a comment that's neutral in your head reads
> harsher to the receiver. Compensate with extra warmth: soften, frame concerns as questions (not
> verdicts), acknowledge what's good, be specific and actionable, distinguish blocking from
> optional, and assume the author is smart and had reasons. "This won't scale" becomes "how does
> this handle peak load? I'm worried the sync call might bottleneck — could we consider a queue?"
> — same concern, warm delivery, opens a dialogue. Clear + warm is the goal: clear so the concern
> is understood, warm so it's received well and the author stays motivated. This is Phase 3's
> warmth applied to the written feedback you give constantly.

---

## Lab — Rewrite Drill

Turn each blunt comment into a warm, clear one.

**Your answer:**

1. "This won't scale."
2. "This section makes no sense."
3. "Why didn't you use a queue here? Obviously that's the right approach."
4. "There are a bunch of problems with this doc." (imagine it's the only comment on an otherwise
   solid doc — fix the balance and specificity)

<details>
<summary>Show Model Answers</summary>
<br>
<strong>1. →</strong> "One thing I'm wondering about: how does this hold up under peak load? I'm a
bit worried the synchronous call could become a bottleneck when traffic spikes — could we consider
a queue here, or is there something I'm missing? Happy to talk it through." (Concern framed as a
question, softened, leaves room to be wrong, invites dialogue — versus the verdict "this won't
scale," which feels like an attack.)
<br><br>
<strong>2. →</strong> "I'm having a bit of trouble following this section — specifically, I wasn't
sure how the auth flow connects to the session handling in the second paragraph. Could you clarify
that link? It might just be me missing context." (Specific about what's confusing (not a blanket
"makes no sense"), owns it partly ("might be me"), and asks for a specific clarification — helpful
rather than dismissive.)
<br><br>
<strong>3. →</strong> "Was there a particular reason you went with the direct call rather than a
queue here? I might be missing some context. My instinct would've been a queue for the buffering,
but curious about your thinking." (Assumes they had reasons (asks rather than accuses), shares your
view as an instinct not a verdict, and stays humble ("might be missing context") — versus "why
didn't you… obviously," which is an accusation that reads as contempt.)
<br><br>
<strong>4. →</strong> "Overall this is a really solid doc — the structure and the problem framing
are clear and I can tell you've thought it through. I left a few specific comments inline: two I
think are worth resolving (the caching consistency question and the rollback plan), and a couple
of smaller nits you can take or leave. Nice work overall!" (Acknowledges the strengths, points to
specific comments, distinguishes blocking from optional, and ends warmly — versus "a bunch of
problems," which is vague, unbalanced, and demoralizing.)
<br><br>
The patterns: frame concerns as questions (not verdicts), be specific about the issue, assume the
author had reasons (ask, don't accuse), acknowledge what's good, and distinguish must-fix from
nit. Clear about the concern, warm in delivery.
</details>

---

## Phrase Bank

| Move | Phrase |
|---|---|
| Concern as a question | "How does this handle [X]? I'm a bit worried about…" |
| Soften a critique | "One thing I'm wondering about…" / "Have we considered…?" |
| Own possible missing context | "I might be missing context, but…" |
| Ask, don't accuse | "Was there a reason for [X]? Curious about your thinking." |
| Acknowledge strengths | "I really like [X here]." / "This section is really clear." |
| Mark severity | "**nit:** …" / "**question:** …" / "**blocking:** …" |
| Take-it-or-leave-it | "Totally optional, but…" / "Take it or leave it." |
| Warm close | "Nice work overall!" / "Happy to talk any of these through." |

---

## Further Reading

| Topic | Source |
|---|---|
| Disagreeing warmly | English track Lesson 18 (disagreeing); Lesson 30 (disagreeing live) |
| Code-review comments | English track Lesson 45 (code-review comments) |
| Feedback that lands | Leadership track Lesson 19 (SBI feedback); Google's code-review guide |

---

## Checkpoint

**Q1.** Why do written comments "read harsher than you mean", and how do you compensate?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Written comments read harsher than you mean because <strong>text strips away tone of voice, facial
expression, and the warmth of in-person delivery — so the receiver reads the bare words without the
softening cues that would accompany them face-to-face, and fills in a harsher tone than you
intended</strong>. When you say something in person, a lot of warmth is carried by <em>how</em> you
say it: a friendly tone, a smile, a collaborative body posture, the back-and-forth that lets you
adjust if it lands wrong. In writing, all of that is gone — the reader gets only the words, and
without the tonal cues, neutral or matter-of-fact words often read as cold, curt, or critical. "This
is wrong" or "this won't scale" — which in your head was a neutral technical observation, and which
said warmly in person would be fine — lands in text as harsh, even hostile, because there's no warm
tone to carry it, and the reader (who can't hear your friendly intent) may read criticism or even
contempt into it. This is compounded by a negativity bias in reading text: people tend to interpret
ambiguous-toned messages more negatively than intended, especially critical ones, and especially
from someone with authority (a lead's comment carries weight, so a curt one stings more). So the
gap between what you meant (neutral/helpful) and what the receiver feels (attacked/criticized) is
wide in written comments — the medium makes you sound harsher than you are. How to compensate:
<strong>add extra warmth you might not need face-to-face, to restore what the medium removed</strong>.
Specifically: (1) <strong>soften</strong> — use hedges and gentle framings ("one thing I'm
wondering about…", "I'm a bit worried that…") that cushion the concern (Phase 3's softeners, in
writing); (2) <strong>frame concerns as questions, not verdicts</strong> — "how does this handle
peak load?" instead of "this won't scale," which invites dialogue rather than feeling like an
attack, and leaves room for you to be wrong; (3) <strong>acknowledge what's good</strong> — note
strengths, not just problems, so the author feels seen and the critical notes are balanced (a doc
with only criticism feels like an attack even if each point is fair); (4) <strong>be specific and
actionable</strong> — point to the exact issue and a direction, so it's helpful rather than a vague
dismissal; (5) <strong>assume good intent and expertise</strong> — "was there a reason for X? I
might be missing context" instead of "why did you do X?" (which reads as an accusation), keeping it
humble and collaborative; and (6) <strong>distinguish blocking from optional</strong> — mark nits
vs. must-fixes so the author isn't overwhelmed by what feels like a pile of demands. The overarching
principle: <strong>err warmer than feels necessary</strong> — because the medium subtracts warmth,
the amount that feels "maybe too much" in your head often lands as just right for the reader. This
isn't being fake or sugar-coating; it's <em>restoring</em> the warmth that in-person delivery would
have carried but text removes, so the comment lands as the helpful, collaborative note you actually
intend rather than the cold criticism the bare words would convey. For a lead especially — whose
comments carry weight and set the team's tone — compensating for the medium's harshness is important:
warm, clear written feedback keeps people motivated and collaborative, while blunt written comments
(even when technically right) demoralize and create defensiveness, undermining both the work and the
relationship.
</details>

**Q2.** Why does framing a concern as a question (rather than a verdict), and acknowledging what's
good, make written feedback more effective?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Both make written feedback more effective by <strong>keeping the author open and motivated to act on
the feedback, rather than defensive or demoralized — which is what determines whether feedback
actually improves the work</strong>. Feedback only works if the receiver takes it in and acts on it;
if they get defensive or discouraged, the feedback fails regardless of how correct it is. On
<strong>framing concerns as questions rather than verdicts</strong>: a verdict ("this won't scale,"
"this is wrong") is a closed judgment that (a) triggers defensiveness — it pronounces the author's
work (and implicitly their competence) deficient, so they defend rather than engage; (b) can be
simply disputed ("yes it will"), creating a standoff rather than progress; and (c) leaves no room
for you to be wrong (if they <em>have</em> handled scaling, your verdict is both wrong and
insulting). A question ("how does this handle peak load? I'm worried the sync call might
bottleneck") instead (a) invites dialogue — it opens a conversation to explore together rather than
a judgment to defend against, so the author engages with the substance; (b) makes the author a
partner in solving the issue rather than a defendant — you're jointly examining a question, not you
attacking and them defending; (c) leaves room for you to be wrong and for them to have context you
lack — maybe they've addressed it, and the question surfaces that gracefully where a verdict
would've been an embarrassing wrong accusation; and (d) is less ego-threatening — a question about
the work doesn't feel like a pronouncement on their competence, so they stay open. So questions
raise the same concern but keep the author engaged and non-defensive, which is what lets the
feedback land. On <strong>acknowledging what's good</strong>: noting strengths, not just problems,
makes feedback more effective because (a) it makes the author feel <em>seen</em> — their work is
recognized, not just picked at, so they don't feel attacked or unappreciated; (b) it balances the
critical notes — a doc with <em>only</em> critical comments feels like an assault (a wall of
negativity) even if each comment is individually fair, which demoralizes the author and makes them
defensive about the whole thing, whereas mixing in genuine appreciation keeps the overall message
balanced and fair-feeling; (c) it makes the critical comments <em>easier to receive</em> — when the
author knows you also see what's good (you're not just hunting for faults), they trust that your
critical notes are fair and well-intentioned, so they take them more readily; and (d) it's
motivating — acknowledgment of what they did well encourages them to keep it up and engage
positively with the improvements, versus pure criticism that discourages. So acknowledging strengths
keeps the author motivated and receptive, which — again — is what lets the feedback improve the work.
The common thread: <strong>the effectiveness of feedback isn't just whether it's correct, but
whether the receiver acts on it — and that depends on keeping them open (not defensive) and
motivated (not demoralized)</strong>. Questions keep them open (dialogue, not attack); acknowledging
what's good keeps them motivated and receptive (seen and fairly treated, not just criticized).
Blunt verdicts and all-negative feedback, by contrast, are often <em>less</em> effective even when
perfectly correct, because they trigger the defensiveness and discouragement that make the author
resist or disengage. This is the Phase 3 warmth insight applied to written feedback: warmth isn't
softness that dilutes the message; it's what makes the message actually land and get acted on. For a
lead giving frequent written feedback, framing concerns as questions and acknowledging strengths are
high-leverage habits — they make your feedback both better-received and more effective at improving
the work, while keeping the author motivated and the relationship strong, rather than winning the
technical point but demoralizing the person.
</details>

---

## Homework

For the next couple of weeks, apply this to every doc comment and review note you leave: frame
concerns as questions (not verdicts), be specific about the issue, assume the author had reasons
(ask, don't accuse), acknowledge what's good (not just problems), distinguish blocking from optional
(nit / question / blocking), and err warmer than feels necessary (the medium strips tone). Re-read
each comment before posting and ask "would this feel warm and helpful to receive?" Add to your
habits: "concerns as questions? specific? acknowledged strengths? marked severity? warm enough for
text?"

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the doc-comment skill — frequent, weighty writing where clarity and warmth meet.
A strong response applies the techniques to real comments: concerns framed as questions, specific and
actionable, assuming the author's competence (asking not accusing), acknowledging strengths,
distinguishing blocking from optional, and erring warmer than feels necessary — with a re-read test
("would this feel warm and helpful to receive?"). The realizations: (1) <strong>written comments read
harsher than you mean</strong> — text strips tone of voice, so neutral-in-your-head words land as
cold or critical; the fix is to add extra warmth that restores what the medium removed (not fakeness
— restoration); (2) <strong>questions beat verdicts, and acknowledgment balances criticism</strong> —
both keep the author open and motivated to act on the feedback, which is what makes feedback actually
improve the work (correct-but-harsh feedback often fails because it triggers defensiveness); (3)
<strong>specificity and severity-marking respect the author</strong> — specific actionable comments
help (vague ones frustrate), and distinguishing nits from must-fixes prevents overwhelm. The
meta-point: commenting on docs is some of the most frequent writing you do, and as a lead your
comments carry weight and set the tone, so getting them clear-and-warm matters a lot. The challenge
is the medium (text strips tone, making comments read harsher), and the compensation is extra warmth
(soften, question-frame, acknowledge, err warmer). This is Phase 3's warmth applied to the written
feedback you give constantly — where warmth isn't softness that dilutes the message but what makes it
land and get acted on. Warm, clear comments keep people motivated and collaborative and get the work
improved; blunt ones (even when right) demoralize and create defensiveness. Add the comment checks to
your habits. This completes Phase 7 (Design Docs & Proposals): document structure, plain language,
paragraphs that flow, executive summaries, persuasive proposals, and doc comments — the longer-form
and feedback writing where a lead's ideas get evaluated and where you shape others' work, all built
on the accurate/natural/warm foundation. The next phase (Feedback & Hard Conversations) goes deeper
into the hardest communication — giving difficult feedback, delivering bad news, and de-escalating —
in both writing and speech, where warmth and clarity matter most.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 8 — Feedback & Hard Conversations (Lesson 44: Feedback Language) →](lesson-44-feedback-language){: .btn .btn-primary }
