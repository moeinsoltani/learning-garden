---
title: "Lesson 39 — Plain Language"
nav_order: 2
parent: "Phase 7: Design Docs & Proposals"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 39: Plain Language

## Concept

Many people — especially those anxious about their writing — think impressive writing means
big words, long sentences, and formal phrasing. The opposite is true: **the best professional
writing is plain, clear, and simple.** Clear writing signals clear thinking; complicated
writing signals confusion (or hiding it). Your goal is never to sound smart — it's to be
understood with the least effort from the reader. This is *especially* freeing for a
non-native speaker: plain language is easier to write correctly and lands better.

```
   TRYING TO SOUND SMART            PLAIN & CLEAR (better)
   ────────────────────            ──────────────────────
   "utilize"                       "use"
   "in order to"                   "to"
   "at this point in time"         "now"
   "we endeavored to facilitate"   "we tried to help"
   "It is imperative to note that" "Note that" / (just say it)

   Write to be UNDERSTOOD, not to sound impressive.
```

The reframe: **plain writing is a sign of strength, not simplicity of mind.** The clearest
thinkers write the most simply — they've done the work of understanding something well enough
to explain it plainly. Reaching for fancy words and convoluted sentences usually makes writing
worse and often hides muddy thinking. For you, this is liberating: you don't need impressive
English; you need clear, simple English, which you can absolutely write.

---

## How It Works

### Prefer the simple word

Choose the plain word over the fancy one: **use** not "utilize", **to** not "in order to",
**help** not "facilitate", **now** not "at this point in time", **enough** not "sufficient",
**about** not "regarding/with respect to", **start** not "commence", **end** not "terminate".
The simple word is clearer, and it's what good writers actually use. Fancy synonyms don't add
value — they add friction.

### Cut the filler

Delete words that don't earn their place: "**It is important to note that** the cache expires
after 5 minutes" → "The cache expires after 5 minutes." Filler phrases — "it should be
mentioned that", "as a matter of fact", "in order to", "the fact that", "at the end of the
day" — pad without adding meaning. If a sentence means the same with a phrase removed, remove
it. (This is Lesson 13's conciseness, at document scale.)

### Short sentences over long ones

Break long, clause-heavy sentences into shorter ones. A sentence with three "which"es and two
"however"s is hard to parse; two or three plain sentences carry the same meaning more clearly.
When a sentence feels tangled, split it. Short sentences are also easier for a non-native
writer to get grammatically right — a double win.

### Active voice, concrete subjects

Prefer "**the service caches the response**" over "the response is cached by the service", and
"**we decided**" over "it was decided". Active voice with a real subject is clearer and shorter,
and it says who does what (passive often hides the actor). Some passive is fine, but default to
active. (Connects to Lesson 01's sentence core.)

### Define or avoid jargon

Jargon is fine among people who share it, but define terms your reader might not know, or use
a plain description instead (Lesson 35's paraphrasing, in writing). "Idempotent (safe to retry
without side effects)" or just "safe to retry" serves more readers than bare "idempotent". Write
for your actual audience — an exec doc needs less jargon than a doc for your immediate team.

### Read it aloud

A quick test: read a sentence aloud. If you run out of breath, stumble, or it sounds like
nobody talks that way, simplify it. Plain writing sounds like a clear person explaining
something — not like a legal contract or a thesaurus. If you wouldn't say it, don't write it.

{: .note }
> **Plain writing is a sign of strength — write to be understood, not to impress**
> The instinct that good writing means big words and long sentences is backwards: the best
> professional writing is plain, clear, and simple, and the clearest thinkers write the most
> simply (they understand the thing well enough to explain it plainly). Reaching for
> impressive-sounding words and convoluted sentences makes writing worse — harder to read, and
> often hiding muddy thinking. Prefer the simple word, cut filler, use short sentences and
> active voice, and define or avoid jargon. Your goal is to be understood with the least effort
> from the reader, never to sound smart. This is especially freeing for a non-native speaker:
> plain language is easier to write correctly AND lands better — so you don't need impressive
> English, just clear, simple English, which you can write well.

---

## Lab — Rewrite Drill

Rewrite each in plain language.

**Your answer:**

1. "We endeavored to utilize the new caching mechanism in order to facilitate a reduction in
   latency."
2. "It is imperative to note that, at this point in time, the aforementioned service is
   experiencing a degradation in performance which is attributable to an insufficiency of
   available memory."
3. "The implementation of the proposed solution will be commenced subsequent to the
   finalization of the requisite approvals by the relevant stakeholders."
4. "A determination was made by the team that the migration should be postponed." (fix the
   passive/hidden-actor problem too)

<details>
<summary>Show Model Answers</summary>
<br>
<strong>1. →</strong> "We used the new cache to reduce latency." (utilize → used; in order to
facilitate a reduction in → to reduce; endeavored to → just "we used". 18 words → 8, and
clearer.)
<br><br>
<strong>2. →</strong> "The service is slow right now because it's low on memory." (Cuts "it is
imperative to note that", "at this point in time", "aforementioned", "experiencing a degradation
in performance", "attributable to an insufficiency of" — all fancy padding. The plain version
says exactly the same thing, faster and clearer.)
<br><br>
<strong>3. →</strong> "We'll start building once the stakeholders approve." (commence → start;
subsequent to the finalization of the requisite approvals by the relevant stakeholders → once
the stakeholders approve. 20+ words → 7.)
<br><br>
<strong>4. →</strong> "The team decided to postpone the migration." (Active voice with a real
subject — "the team decided" — instead of "a determination was made by the team", which hides
the actor behind passive padding. Shorter, clearer, and says who decided.)
<br><br>
The patterns: prefer the simple word (use, to, start), cut filler ("it is imperative to note
that", "at this point in time"), use active voice with a real subject ("the team decided"), and
say the same thing in far fewer words. Plain isn't dumber — it's clearer, and it's what strong
writers do.
</details>

---

## Phrase Bank

| Fancy / padded | Plain |
|---|---|
| utilize / leverage | use |
| in order to | to |
| at this point in time | now |
| subsequent to / prior to | after / before |
| facilitate | help |
| sufficient / insufficient | enough / not enough |
| commence / terminate | start / end |
| it is imperative/important to note that | (just say it) / Note: |
| in the event that | if |
| a determination was made | we decided |

---

## Further Reading

| Topic | Source |
|---|---|
| Conciseness (cutting words) | English track Lesson 13 (concise words) |
| Sentence core & active voice | English track Lesson 01 (sentence core) |
| Plain language | [plainlanguage.gov guidelines](https://www.plainlanguage.gov/guidelines/) |

---

## Checkpoint

**Q1.** Why is "plain writing is a sign of strength, not simplicity of mind" true, and why is
plain language especially freeing for a non-native writer?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
"Plain writing is a sign of strength, not simplicity of mind" is true because <strong>writing
something plainly requires understanding it well enough to explain it simply — which is harder,
and a mark of clearer thinking, than dressing it up in complicated language</strong>. There's a
common but backwards belief that impressive writing means big words, long sentences, and formal
phrasing — that simple writing looks unsophisticated. The reality is the opposite: (1)
<strong>clear writing reflects clear thinking</strong> — to write plainly, you have to actually
understand the idea, know what matters and what doesn't, and find the direct way to say it;
that's cognitively demanding and reflects mastery. Convoluted writing, by contrast, often
reflects <em>muddy</em> thinking — the writer hasn't clarified the idea, so it comes out tangled,
or they use complexity to hide that they're not sure what they mean (fancy words papering over
unclear thought). So plain writing signals you understand the thing; complicated writing often
signals you don't (or are hiding it). (2) <strong>the clearest thinkers write the most
simply</strong> — the experts who understand a topic deeply can explain it plainly (they've done
the work of understanding); it's often those with a shakier grasp who hide behind jargon and
complexity. So plainness is associated with expertise and confidence, not simplicity of mind. (3)
<strong>impressive-sounding writing is usually worse writing</strong> — big words and long
sentences make text harder to read (more friction for the reader) without adding meaning; the
fancy synonym ("utilize") says nothing more than the plain word ("use"), it just adds effort. So
reaching for impressiveness makes writing worse, not better — it fails the actual goal (being
understood) in pursuit of a false goal (sounding smart). The reframe: <strong>the goal of writing
is to be understood with the least effort from the reader, never to sound smart</strong> — and
plain, clear, simple writing achieves that, which is why it's the mark of a strong writer (and
thinker), not a simple one. Why plain language is especially freeing for a non-native writer:
<strong>it removes the pressure to produce impressive, complex English (which is hard and
error-prone) and replaces it with a goal you can absolutely achieve — clear, simple English</strong>.
A non-native writer often feels they need sophisticated vocabulary and complex sentences to be
taken seriously, which is stressful (reaching for fancy words you're unsure of) and
counterproductive (complex sentences are where grammar errors multiply — the long clause-heavy
sentence with multiple tenses and articles is exactly where mistakes happen). Plain language is
freeing on both counts: (1) it's the <em>correct</em> goal anyway (plain beats fancy for
everyone), so you're not settling for a lesser version — you're doing what strong writers do; and
(2) it's <em>easier to write correctly</em> — short, simple sentences with common words have
fewer places to make grammar mistakes (subject-verb agreement, articles, and word choice are all
easier to get right in a short plain sentence than a long ornate one), so plain language plays to
your strengths and minimizes errors. So the non-native writer doesn't need to chase impressive
English; they can write plain, simple, clear English — which is both what actually communicates
best AND what they can produce most accurately. That's doubly liberating: the easier path is also
the better path. Internalizing "plain is strong, not simple" frees you from a false standard
(impressive English) and points you at an achievable, correct one (clear English) — which is
exactly what makes writing less daunting and more effective for a non-native speaker.
</details>

**Q2.** Give three concrete techniques for making writing plainer, and explain how each helps
the reader.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Three concrete techniques for making writing plainer, and how each helps the reader: (1)
<strong>Prefer the simple word over the fancy one</strong> — choose "use" over "utilize", "to"
over "in order to", "help" over "facilitate", "now" over "at this point in time", "start" over
"commence". How it helps the reader: the simple word is instantly understood with no friction,
while the fancy synonym adds processing effort (or even ambiguity) without adding any meaning —
"utilize" says nothing "use" doesn't, it just makes the reader work slightly harder. Simple words
are also the words readers actually know and expect, so the text flows; fancy words make it feel
stilted and slow. So preferring simple words reduces the reader's effort per word, making the
whole text faster and easier to absorb. (2) <strong>Cut filler phrases</strong> — delete words
that don't add meaning: "it is important to note that", "at this point in time", "the fact that",
"in order to", "as a matter of fact". "It is imperative to note that the cache expires after 5
minutes" → "The cache expires after 5 minutes." How it helps the reader: filler pads the text
with words the reader has to process but that carry no information, diluting the actual content
and slowing them down — they wade through padding to reach the meaning. Cutting filler
concentrates the meaning (every remaining word earns its place), so the reader gets the point
faster with less reading. It also sharpens the writing (the message stands out instead of being
buried in verbiage). (3) <strong>Use short sentences and active voice with real subjects</strong>
— break long clause-heavy sentences into shorter ones, and prefer "the service caches the
response" / "we decided" over "the response is cached by the service" / "a determination was
made". How it helps the reader: short sentences are easier to parse — the reader holds one clear
idea at a time, rather than tracking multiple clauses, "which"es, and "however"s across a long
sentence and trying to assemble the meaning (which is effortful and error-prone). Active voice
with a real subject is clearer and shorter, and it tells the reader <em>who does what</em>
(passive often hides the actor — "a determination was made" leaves the reader wondering "by
whom?"), so the reader immediately knows the agent and action. Both make the sentence's meaning
land quickly and unambiguously. (Bonus techniques: define or avoid jargon so readers aren't
blocked by unfamiliar terms, and read aloud to catch anything convoluted.) The common thread
across all three: <strong>each reduces the effort the reader spends to extract the meaning</strong>
— simpler words (less effort per word), less filler (less to wade through), shorter active
sentences (easier to parse, clear actor). Since the goal of writing is to be understood with the
least reader effort, each technique directly serves that goal. And notably, all three are
<em>easier</em> for a non-native writer to execute correctly (simple words you're sure of, fewer
words to get wrong, short sentences with fewer grammar traps), so plain-language techniques both
help the reader and play to your strengths — the better writing is also the more achievable
writing.
</details>

---

## Homework

Take a piece of your own writing (a design doc, a long Slack message, an email) and make it
plainer: replace fancy words with simple ones (utilize → use), cut every filler phrase that
doesn't add meaning, break long sentences into short ones, and switch passive/hidden-actor
sentences to active with real subjects. Read it aloud and simplify anything you stumble on. Count
the words before and after — notice how much you cut while keeping (improving) the meaning. Add to
your habits: "simple words? cut filler? short sentences? active voice? reads aloud cleanly?"

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the plain-language skill — writing to be understood, not to impress, which is
what actually makes writing (and a writer) credible. A strong response takes real writing and
plainifies it: simple words for fancy ones, filler cut, long sentences split, passive/hidden-actor
sentences made active — then reads aloud and simplifies stumbles, and notices the word count drop
alongside improved clarity. The realizations: (1) <strong>plain is stronger, not simpler</strong>
— the clearest thinkers write the most simply; reaching for impressive words and complex sentences
makes writing worse (more friction, often hiding muddy thinking), so the goal is to be understood
with least reader effort, never to sound smart; (2) <strong>concrete techniques make it
achievable</strong> — prefer simple words, cut filler, short sentences, active voice with real
subjects, define/avoid jargon, read aloud; each reduces the reader's effort to extract meaning; (3)
<strong>plain language is especially freeing for a non-native writer</strong> — it's both the
correct goal (plain beats fancy for everyone) AND easier to write correctly (short simple sentences
have fewer grammar traps), so the easier path is also the better path, removing the false pressure
to produce impressive complex English. The meta-point: the belief that good writing means big words
and long sentences is backwards; plain, clear, simple writing is what strong writers and clear
thinkers produce, and it serves the real goal (understanding). For a non-native speaker this is
doubly liberating — you don't need impressive English, just clear English, which you can absolutely
write, and which plays to your strengths (simple correct sentences over ornate error-prone ones).
Add the plain-language checks to your habits. This pairs with document structure (Lesson 38):
structure organizes the doc, plain language makes each sentence land. The next lesson covers
paragraphs that flow — how to connect sentences and ideas so a reader glides through your writing,
the level between the sentence and the whole doc.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 40 — Paragraphs That Flow →](lesson-40-paragraphs-flow){: .btn .btn-primary }
