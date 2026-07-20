---
title: "Lesson 41 — Executive Summaries"
nav_order: 4
parent: "Phase 7: Design Docs & Proposals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 41: Executive Summaries

## Concept

Busy leaders don't read your whole doc — they read the **summary** and decide. So the few
lines at the top often matter more than the twenty pages below: they determine whether the
reader grasps your point, trusts it, and acts. Writing a summary that captures the essence in a
few lines — for someone who won't read the rest — is one of the highest-value writing skills for
a lead, and a distinct skill from writing the doc itself.

```
   THE EXEC SUMMARY (3-6 lines) ANSWERS:
   ┌──────────────────────────────────────────────────────┐
   │ • WHAT are we proposing / what's the situation?       │
   │ • WHY does it matter (impact / cost of inaction)?     │
   │ • WHAT do we recommend / decide?                      │
   │ • WHAT do we need from the reader (decision/ask)?     │
   └──────────────────────────────────────────────────────┘

   Written for someone who reads ONLY this. Self-contained.
```

The reframe: **the summary must stand completely on its own** — a reader who reads only the
summary (which is most senior readers) should come away knowing the point, why it matters, the
recommendation, and what's being asked. It's not a teaser for the doc; it's the doc's essence,
complete in itself. Nailing this is a skill worth deliberate practice.

---

## Going Deeper

### Write for someone who reads only the summary

Assume the reader reads the summary and nothing else — because senior people often do. So the
summary must be **self-contained**: it delivers the full essential message on its own, not "read
the doc for the point." A summary that says "this doc explores our caching options" fails — the
reader learns nothing actionable. One that says "We recommend adopting Redis to fix checkout
slowdowns; it's a one-sprint change cutting p99 latency ~40%; we need your approval to proceed"
gives them everything.

### Lead with the bottom line (BLUF)

State the conclusion/recommendation first, not a build-up. Executives want the answer, then
(optionally) the reasoning — the reverse of a mystery novel. "We should adopt Redis" comes
first; the why and how follow. Burying the recommendation under context wastes the one thing you
know they'll read.

### Answer the four questions

A strong exec summary answers: **What** (are we proposing / is the situation), **Why** (does it
matter — the impact, or the cost of doing nothing), **What we recommend** (the decision/
direction), and **What we need** (from the reader — a decision, approval, awareness). In 3–6
lines. If a busy leader gets those four, they can act.

### Speak the reader's language — impact, not implementation

Executives care about **impact** (cost, revenue, risk, time, users), not technical detail. Frame
the summary in those terms: "reduces checkout latency, which is costing us conversions" lands
better with a leader than "adds an LRU cache layer with a 5-minute TTL". Translate the technical
into the business/impact terms your reader thinks in (a leadership-track skill — connecting tech
to business). Save the implementation for the body.

### Quantify where you can

Numbers make a summary concrete and credible: "cuts latency ~40%", "saves ~$5k/month", "two weeks
of work", "affects all checkout users". Vague claims ("improves performance", "not too much
effort") are weak; a specific figure (even estimated, flagged as such) is far more convincing and
actionable. Round numbers a leader can hold onto.

### Write it last, cut it ruthlessly

Write the summary after the doc, when you know what matters most — then cut it down. The
discipline is fitting the essence into a few lines, which means ruthless prioritization (what are
the 3–4 things that truly matter?). If your summary is a page, it's not a summary. Distilling is
the skill; resist including everything.

{: .note }
> **The summary must stand alone — most senior readers read only it**
> Busy leaders read the summary and decide, so those few lines often matter more than the whole
> doc. Write the summary for someone who reads <em>only</em> it: self-contained, delivering the
> full essential message (not a teaser). Lead with the bottom line (BLUF — the recommendation
> first), answer the four questions (what / why it matters / what we recommend / what we need),
> frame it in the reader's terms (impact, not implementation), and quantify where you can. Write
> it last and cut it ruthlessly — distilling the essence into a few lines is the skill. Nail the
> summary and your idea gets grasped, trusted, and acted on by the people who decide; bury the
> point or make it a vague teaser and even a great doc fails at the top. This is one of the
> highest-value writing skills for a lead.

---

## Lab — Rewrite Drill

Fix each weak summary.

**Your answer:**

1. **A teaser, not a summary:** "This document explores the various options for improving our
   caching strategy and discusses the trade-offs of each approach, with a recommendation at the
   end." Rewrite it to stand alone.
2. **Too technical for an exec:** "We propose introducing a Redis-backed LRU cache with a
   5-minute TTL and write-through invalidation on the product-catalog read path." Reframe for a
   VP who cares about impact.
3. **Vague, no numbers, no ask:** "Our system has some performance problems that we think we can
   improve with some caching work, which should help things and shouldn't take too long. Let us
   know what you think." Rewrite as a strong 3–4 line exec summary (invent reasonable numbers).

<details>
<summary>Show Model Answers</summary>
<br>
<strong>1. Teaser → self-contained:</strong> "We recommend adopting Redis caching to fix our
checkout slowdowns. It's the best of the three options we evaluated — proven, low-risk, and fast
to build (about one sprint) — and should cut checkout latency significantly. We're asking for
approval to proceed." (Delivers the actual recommendation, why, effort, and ask — stands alone —
versus "this document explores options… recommendation at the end," which forces the reader into
the doc to learn anything.)
<br><br>
<strong>2. Technical → impact:</strong> "We can noticeably speed up checkout by adding a caching
layer to our slowest database queries. This should reduce the lunchtime slowdowns that are likely
costing us conversions, at about one sprint of work and low risk." (Frames it in impact terms —
faster checkout, fewer slowdowns, protecting conversions, effort, risk — that a VP cares about;
the "Redis-backed LRU cache with 5-minute TTL" detail belongs in the body, not the summary.)
<br><br>
<strong>3. Vague → strong summary:</strong> "<strong>Checkout is slow at peak times</strong>
(p99 latency ~2s at lunch), which is likely costing us conversions. <strong>We recommend adding a
Redis cache</strong> to the hot queries, expected to cut peak latency by ~40% (to ~1.2s).
<strong>Effort:</strong> about one sprint, low risk. <strong>We need:</strong> your approval to
start next sprint." (What/why with a number, the recommendation with a quantified expected impact,
effort, and a clear ask — concrete, credible, actionable — versus the vague "some performance
problems… some caching… shouldn't take too long.")
<br><br>
The patterns: make the summary self-contained (deliver the recommendation, not a teaser); lead with
the bottom line; frame in impact terms for the audience; quantify (real or flagged-estimate
numbers); and end with a clear ask.
</details>

---

## Phrase Bank

| Element | Phrase |
|---|---|
| Bottom line first | "We recommend [X] to [solve Y]." |
| The situation | "Today, [problem], which is [impact]." |
| Why it matters | "This is costing us [conversions / $X / time / risk]." |
| Quantified impact | "Expected to [cut latency ~40% / save ~$5k/mo]." |
| Effort/risk | "About [one sprint] of work, [low] risk." |
| The ask | "We need your [approval / decision] to [proceed / choose]." |
| Impact framing | "In business terms, this means…" |

---

## Further Reading

| Topic | Source |
|---|---|
| Connecting tech to business | Leadership track Lesson 48 (translating tech to business) |
| Exec summaries & BLUF | Leadership track Lesson 13 (audience-first); Lesson 16 (design docs) |
| Persuasive proposals | English track Lesson 42 (next lesson) |

---

## Checkpoint

**Q1.** Why must an executive summary "stand alone", and what four questions should it answer?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An executive summary must stand alone because <strong>most senior readers read only the summary
and decide from it — they don't read the full doc — so the summary has to deliver the complete
essential message by itself, or the reader gets nothing actionable</strong>. Busy leaders
(executives, senior stakeholders, decision-makers) have limited time and many docs; their normal
mode is to read the summary at the top and, based on it, either act (approve, decide) or decide
whether the topic warrants reading further (usually not). So in practice, the summary is often the
<em>entire</em> doc as far as the decision-maker is concerned — the twenty pages below might never
be read by the person who matters most. This means the summary can't be a <em>teaser</em> that
points to the doc for the real content ("this document explores our caching options and gives a
recommendation at the end") — a reader who reads only that learns nothing they can act on, so the
summary has failed at its core job. Instead, the summary must be <strong>self-contained</strong>:
it must deliver the full essential message — the point, why it matters, the recommendation, and
the ask — completely within itself, so that a reader who reads <em>only</em> the summary comes away
knowing everything they need to understand and act on the proposal. The summary is the doc's
essence, complete in itself, not an advertisement for the doc. Since it's what the deciders
actually read, it often matters more than the whole rest of the doc — a great doc with a weak
(teaser or buried) summary fails at the top, while a strong self-contained summary gets the idea
grasped and acted on even by readers who never open the body. The four questions a strong exec
summary should answer (in 3–6 lines): (1) <strong>What</strong> — what are we proposing, or what's
the situation/problem? (the subject); (2) <strong>Why</strong> — why does it matter? the impact,
or the cost of doing nothing (e.g., "checkout is slow, costing us conversions") — this gives the
reader the stakes; (3) <strong>What we recommend</strong> — the actual recommendation/decision/
direction (e.g., "adopt Redis caching"), stated clearly (BLUF — bottom line up front, not buried);
and (4) <strong>What we need</strong> — what's being asked of the reader: a decision, an approval,
awareness (e.g., "we need your approval to proceed"). If a busy leader gets those four —
what, why it matters, the recommendation, and what's needed from them — they have everything
required to act (approve, decide, or ask a question), which is the summary's job. Answering all
four, self-contained, in a few lines, framed in the reader's terms (impact, not implementation)
and quantified where possible, is what makes an exec summary do its job for the people who read
only it — which is why writing it well is one of the highest-value writing skills for a lead.
</details>

**Q2.** Why should an exec summary be framed in "impact" terms (cost, revenue, risk, users) rather
than implementation detail, and why does quantifying matter?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An exec summary should be framed in impact terms rather than implementation detail because
<strong>executives think and decide in terms of business impact — cost, revenue, risk, time,
users — not technical mechanics, so framing the summary in their terms is what lets them
understand the stakes and make the decision</strong>. A leader reading your summary is asking
"why does this matter to the business, and what should I decide?" — they care about outcomes
(will this make/save money, reduce risk, improve the product, take a lot of time?), not the
technical implementation (the cache eviction policy, the TTL, the data structure). If you frame
the summary technically — "we propose a Redis-backed LRU cache with a 5-minute TTL and
write-through invalidation" — you've told the executive <em>how</em> without telling them
<em>why it matters</em>, in language they may not even parse; they can't evaluate the business
case because you've given them mechanics instead of impact. Reframed in impact terms —
"we can speed up checkout, reducing the lunchtime slowdowns that are costing us conversions, at
about one sprint of work and low risk" — the same proposal is now something the executive can
actually evaluate and decide on: they understand the benefit (faster checkout → more
conversions → revenue), the cost (one sprint), and the risk (low), which is exactly the
information a business decision needs. So impact-framing translates your technical work into the
terms your reader thinks in, letting them grasp the value and decide — the implementation detail
belongs in the body for the technical reviewers, not in the summary for the deciders. (This is
the leadership skill of connecting tech to business — you're the translator between the technical
reality and the business meaning.) Why quantifying matters: <strong>numbers make the summary
concrete, credible, and actionable, where vague claims are weak and unconvincing</strong>. Compare
"this improves performance and shouldn't take too long" with "this cuts peak checkout latency
~40% (from ~2s to ~1.2s), at about one sprint of work." The quantified version is: (1) <strong>more
concrete</strong> — "40% faster" and "one sprint" are specific things the reader can grasp and
weigh, whereas "improves performance" and "not too long" are vague and could mean anything (a
2% improvement? a 50% one? two days or two months?); (2) <strong>more credible</strong> — a
specific figure signals you've actually analyzed the impact and know what you're talking about,
whereas vague claims suggest you haven't measured or are hand-waving (which makes a skeptical
executive discount the whole proposal); (3) <strong>more actionable</strong> — a leader deciding
whether to approve needs to weigh benefit against cost, which requires numbers (is a 40% latency
cut worth one sprint? that's a real trade-off they can evaluate; is "some improvement" worth "not
too long"? that's un-decidable). Even estimated numbers (flagged as estimates — "~40%", "roughly
one sprint") are far better than none, because they give the reader something concrete to reason
with, and rounding to figures a leader can hold onto ("~$5k/month", "two weeks") aids that. So
quantifying transforms a vague, weak, hard-to-act-on summary into a concrete, credible,
decision-ready one. Together, impact-framing and quantifying make the summary speak the executive's
language (business outcomes) in the executive's currency (numbers) — which is exactly what enables
the busy leader who reads only the summary to understand the value and make the call. For a lead,
mastering this translation (technical work → quantified business impact) is high-value: it's how
your good technical ideas get understood, funded, and approved by the non-technical people who
decide.
</details>

---

## Homework

Take a doc or proposal you've written (or will write) and craft a standalone exec summary for it:
write it last, lead with the bottom line, answer the four questions (what / why it matters / what
you recommend / what you need), frame it in impact terms (not implementation), quantify where you
can (even estimates), and cut it to 3–6 lines. Test it: would a reader who reads ONLY the summary
know your point and what you're asking? Add to your habits: "summary stands alone? bottom line
first? impact-framed? quantified? clear ask?"

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the exec-summary skill — distilling a whole doc into the few lines a busy
leader reads, one of the highest-value writing skills for a lead. A strong response crafts a
standalone summary: written last, bottom line first, answering the four questions (what / why it
matters / recommendation / ask), framed in impact terms, quantified, cut to a few lines — then
tests whether a summary-only reader would get the point and the ask. The realizations: (1)
<strong>the summary must stand alone</strong> — most senior readers read only it and decide, so
it must deliver the full essential message itself, not tease the doc; a self-contained summary
gets the idea acted on even by those who never open the body; (2) <strong>frame in impact, not
implementation</strong> — executives decide in business terms (cost, revenue, risk, users), so
translate the technical work into the value they care about, saving mechanics for the body; (3)
<strong>quantify</strong> — specific numbers (even estimates) make the summary concrete, credible,
and actionable, where vague claims are weak and un-decidable; (4) <strong>write it last and cut
ruthlessly</strong> — distilling the essence into a few lines is the skill, requiring prioritizing
what truly matters. The meta-point: the summary often matters more than the whole doc, because
it's what the deciders actually read — nail it and your idea gets grasped, trusted, and acted on
by the people who matter; bury the point or make it a vague teaser and even a great doc fails at
the top. This connects to the leadership skill of translating tech to business (you're the
bridge between technical reality and business meaning) and to lead-with-the-point (the summary is
BLUF at document scale). For a lead whose ideas need buy-in from senior, often non-technical
deciders, the standalone, impact-framed, quantified summary is how good technical proposals get
understood and approved. Add the summary checks to your habits. The next lesson covers persuasive
proposals — going beyond a clear summary to actually make the case that convinces people to say
yes, structuring an argument that persuades.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 42 — Persuasive Proposals →](lesson-42-persuasive-proposals){: .btn .btn-primary }
