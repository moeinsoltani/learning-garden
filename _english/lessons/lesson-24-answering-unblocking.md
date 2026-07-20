---
title: "Lesson 24 — Answering and Unblocking Others"
nav_order: 4
parent: "Phase 4: Slack Communication"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 24: Answering and Unblocking Others

## Concept

The other side of asking for help is *answering* — and being the person whose answers
actually unblock people is a valuable, well-liked trait. A good answer gives the
person what they need to move forward, warmly; a poor answer is vague, or answers a
different question, or leaves them still stuck.

```
   WEAK ANSWER                        STRONG ANSWER
   ───────────                        ─────────────
   "check the docs"                   "You'll want the `retryPolicy` setting —
                                       it's in config.yaml, line ~40. Here's an
                                       example: [code]. Let me know if that
                                       doesn't work!"

   answer the QUESTION first, then add context. Unblock them.
```

The core skill: **answer the question first, then add context.** Lead with the direct
answer (so they're unblocked immediately), then any elaboration. And if you don't
fully know, a partial answer with a pointer ("I'm not sure, but X might help / ask
Y") beats silence. Being genuinely helpful — and warm about it — makes you the person
people want to work with.

---

## Going Deeper

### Answer the question first

Lead with the direct answer to what they asked, then add context/caveats:

- ✗ [three paragraphs of background] ... "so the answer is X." (answer buried)
- ✓ "Yes — the answer is X. Context: [why], and watch out for [caveat]." (answer
  first, then elaboration)

They asked a question; give them the answer up front (lead with the point, Lesson 21),
so they're unblocked immediately, then any useful context.

### "I don't know, but…" beats silence

If you don't know the full answer, a partial answer or a pointer is far better than not
responding:

- "I'm not sure exactly, but I think it's related to the retry config — [name] would
  know for sure." (points them somewhere)
- "I don't know the answer, but here's how I'd find out: [approach]." (helps them
  self-serve)

Silence leaves them stuck; a partial answer or pointer keeps them moving.

### Partial answers with a timeline

If you can help but not right now, say so with a timeline rather than going silent:

- "Short answer: yes, that should work. I'll give you the fuller explanation after
  standup." (immediate partial answer + when the rest is coming)
- "I can help with this — give me ~30 min, then I'll dig in with you." (commits to help,
  sets expectations)

This unblocks them partially now and tells them when they'll get more.

### Link instead of retyping

If the answer already exists (docs, a previous thread, a runbook), link to it — but add
a pointer to the relevant part ("it's in the deploy runbook, under 'rollback' — here:
[link]"), not just a bare link (a bare link makes them hunt).

### The teaching answer vs the fast answer

Sometimes the best answer just solves their problem (fast — they're blocked, unblock
them); sometimes it's better to help them understand or find it (teaching — they'll
learn and not need to ask again). For an urgent block, give the fast answer; for a
learning opportunity (a junior, a recurring question), consider guiding them. Both are
helpful; match to the moment (the coaching-vs-mentoring idea from the leadership track,
at Slack scale).

{: .note }
> **Being genuinely helpful makes you the person people want to work with**
> The person who reliably unblocks others — answers clearly, points them the right way
> when unsure, does it warmly — builds enormous goodwill and a great reputation.
> Helping others is also reciprocal: people help those who help them. So answering
> well (answer first, partial-answer-beats-silence, warm) isn't just nice — it makes
> you valued and well-liked. For your goal, answering warmly and helpfully is a big
> part of being the warm, collaborative colleague you want to be — and it's a natural
> place to practice the warmth tools (Phase 3).

---

## Lab — Rewrite Drill

Write a warm, helpful answer to each of these four situations.

**Your answer (write each answer):**

1. **You know it:** Someone asks "where's the config for the retry logic?" You know it's in `config.yaml` around line 40.
2. **You half-know it:** Someone asks why the payment webhook is failing. You're not sure, but you suspect a signature-verification issue, and you know Priya set up that system.
3. **The wrong question:** Someone asks "how do I increase the timeout to fix these slow requests?" but you realize the real problem is probably an N+1 query, not the timeout.
4. **Asked to you, but should be public/someone else:** Someone DMs you a general question about the deploy process that the whole team would benefit from, and that Sam actually owns.

<details>
<summary>Show Model Answers</summary>
<br>
<strong>1. You know it:</strong> "It's in <code>config.yaml</code>, around line 40 —
look for <code>retryPolicy</code>. Here's roughly what it looks like: [example]. Let me
know if you need anything else! 🙏" (Answer first (exact location), a helpful example,
warm offer — unblocks them immediately.)
<br><br>
<strong>2. You half-know it:</strong> "I'm not 100% sure, but my guess is it's a
signature-verification issue — a common cause of webhook failures. Priya set up that
system, so she'd know for sure — worth pinging her. Happy to help dig in too if you get
stuck!" (Partial answer + pointer to who knows + offer to help — better than silence.)
<br><br>
<strong>3. The wrong question:</strong> "You could bump the timeout, but I have a hunch
the real issue might be an N+1 query rather than genuine slowness — increasing the
timeout would just hide it. Could you check if [endpoint] is making a query per row? If
so, batching it would fix the root cause. Happy to look together!" (Gently redirects to
the likely real problem — the question behind the question — done warmly.)
<br><br>
<strong>4. Should be public/someone else:</strong> "Great question! This is actually
something the whole team runs into — do you mind if I answer in #eng-help so it's
searchable for others? And Sam owns the deploy process, so they'd have the fullest
answer — I'll tag them there. 🙂" (Routes to public (searchable, benefits everyone) and
the right owner, warmly — being <em>more</em> helpful by routing it well.)
<br><br>
The patterns: answer first (1), partial answer + pointer when unsure (2), answer the real
question behind the question (3), route to public/the right person warmly (4) — all
keeping them moving and doing it warmly.
</details>

---

## Phrase Bank

| Situation | Phrase |
|---|---|
| Direct answer | "It's [X] — here's [pointer/example]." |
| Don't fully know | "Not sure, but I think [X] — [person] would know for sure." |
| Partial + timeline | "Short answer: yes. Fuller explanation after standup." |
| Link with a pointer | "It's in [doc], under [section]: [link]." |
| Redirect the question | "You could do X, but the real issue might be Y —" |
| Route to public/owner | "Great question — mind if I answer in [channel]? [owner] knows this best." |
| Warm offer | "Let me know if that doesn't work!" / "Happy to dig in together." |

---

## Further Reading

| Topic | Source |
|---|---|
| Answering the real question | Leadership track Lesson 25 (listening & questions) |
| Coaching vs answering (teach vs solve) | Leadership track Lesson 26 |
| Being helpful & reciprocity | *Give and Take*, Adam Grant |

---

## Checkpoint

**Q1.** Why should you "answer the question first, then add context," and why does a
partial answer or pointer beat silence when you don't fully know?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You answer the question first because <strong>the person asked because they're blocked,
and the fastest way to help is to give the answer immediately</strong> — rather than
making them read through your context and reasoning to find it. Burying the answer under
background forces the blocked person to wade through it to get unblocked (slow and
frustrating when they just need the answer); leading with the direct answer unblocks
them right away, with context available after if they want it. This is lead-with-the-point
(Lesson 21) applied to answering. Why a partial answer or pointer beats silence when you
don't fully know: <strong>the goal is to keep the person moving, and even incomplete help
does that, while silence leaves them stuck and uncertain</strong>. Tempted to say nothing
(not wanting to give an incomplete or wrong answer), you'd leave them worst off — still
blocked, and unsure anyone will help. A best-guess partial answer gives them a lead; a
pointer to who knows routes them to the answer; a pointer to how to find out helps them
self-serve. All keep them progressing. The reframe: <strong>you don't need the complete
answer to be helpful</strong> — partial knowledge, a flagged guess, or a pointer is
genuinely valuable and far better than silence, and it's low-cost (you're not committing
to a full solution). So: know it → answer first (then context); half-know it → share
your partial knowledge + a pointer; can help but not now → say so with a timeline; only
stay silent if you truly have nothing (rare). This makes you reliably helpful — a valued,
well-liked trait — and it's warm and considerate (actively helping rather than leaving
people stuck).
</details>

**Q2.** Sometimes the most helpful answer addresses the question *behind* the question
(the real problem), not the literal question. Give an example, and how to do it warmly.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Sometimes the literal question is based on a wrong assumption or aims at a symptom, and
the most helpful thing is to gently address the <em>real</em> issue. Example: someone
asks "how do I increase the timeout to fix these slow requests?" but you realize the real
problem is likely an N+1 query, not genuine slowness. Answering literally ("here's how to
increase the timeout") applies a band-aid that <em>hides</em> the problem; the more
helpful answer addresses the root cause: "You could bump the timeout, but I suspect the
real issue is an N+1 query — that would just mask it. Could you check if [endpoint]
queries per row? Batching it would fix the actual cause." This gives them what they
<em>need</em> (a real fix) rather than just what they <em>asked for</em> (a way to hide
the symptom). Why it's often the most helpful: it solves the actual problem, prevents them
heading down the wrong path (implementing a fix that doesn't fix it), and shows you
understood their situation deeply (the leadership track's listening skills — the real
problem, not the presented one). How to do it warmly (so it doesn't feel like "you're
asking the wrong question"): (1) <strong>acknowledge their question first</strong> — "You
could do X" — so you're not dismissing their approach; (2) <strong>frame it as a humble
hunch</strong> — "I suspect the real issue might be Y" (softeners, Lesson 15) rather than
"you're wrong, it's Y"; (3) <strong>explain why it matters</strong> — "increasing the
timeout would just hide it" — so the redirect clearly helps them; (4) <strong>offer to
help</strong> — "happy to look together." The warm version lands as "here's a more
helpful angle," not "you asked the wrong thing." Answering the question behind the
question, warmly, is a hallmark of a genuinely helpful, expert colleague — you address
what the person actually needs, kindly.
</details>

---

## Homework

For a few days, focus on answering and unblocking others well: answer the question first
(then context), give a partial answer or pointer when you don't fully know (rather than
silence), watch for chances to answer the real question behind the question (warmly), and
close by offering further help. When someone DMs a general question, consider routing it
to a public channel or the right owner. Notice how being reliably helpful affects your
relationships. Add to your checklist: "answering — answer first, partial-beats-silence,
watch for the real question, warm."

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the answering-and-unblocking skill — the other half of the help
exchange, and a valued, relationship-building trait. A strong response answers the
question first (unblocking immediately), gives partial answers/pointers when unsure
(rather than silence), watches for the real question behind the question (redirecting
warmly), and offers further help. The realizations: leading with the direct answer
unblocks faster; a partial answer or pointer keeps people moving (you don't need the
complete answer to help); catching the real question is sometimes the most valuable
answer (done warmly so it helps rather than corrects); routing general questions to
public/owners benefits everyone. The meta-point: being the person who reliably unblocks
others — clearly, pointing the right way when unsure, warmly — builds enormous goodwill
and a great reputation, and it's reciprocal (people help those who help them). So
answering well makes you valued and well-liked, and builds the relationships that make
work better. For your goal, answering warmly and helpfully is a big part of being the
warm, collaborative colleague you want to be — and a natural, frequent place to practice
the warmth tools (Phase 3). This pairs with the previous lesson (asking well) — the two
sides of collaborative help. Add the answering checklist item. The remaining Phase 4
lessons cover async etiquette and announcements — completing the Slack skills that,
combined with warmth, make you a teammate people genuinely like working with.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 25 — Async Etiquette →](lesson-25-async-etiquette){: .btn .btn-primary }
