---
title: "Lesson 35 — Paraphrasing"
nav_order: 2
parent: "Phase 6: Spoken Fluency"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 35: Paraphrasing

## Concept

Paraphrasing — saying something back in your own words — is a quietly powerful speaking
skill with two big uses. First, it **confirms understanding**: "so what you're saying is…"
lets you check you got it right (and shows the other person you listened). Second, it's your
**escape hatch when a word won't come**: instead of freezing because you can't recall the
exact term, you describe the idea in simpler words and keep going. For a non-native speaker,
paraphrasing is one of the highest-value fluency tools.

```
   TWO USES OF PARAPHRASING:

   1. CONFIRM     "So if I understand right, you want X before Y — correct?"
                  → checks understanding, shows you listened

   2. WORK AROUND a word you can't recall:
      (can't remember "idempotent") → "so it's safe to run more than once,
       calling it twice does the same as once — you know the property I mean?"
      → describe the idea in simple words; keep talking, don't freeze
```

The reframe: **you never need a specific word to express an idea** — you can always describe
it in simpler terms. So a word you can't recall (or don't know) is never a dead end; you
paraphrase around it. This single habit removes one of the biggest fears of speaking a second
language: getting stuck on a missing word.

---

## How It Works

### Paraphrasing to confirm understanding

Say back what you understood, in your own words, and check:

- "So if I've got this right, you want the cache invalidated on every write — is that
  correct?"
- "Let me make sure I follow — you're proposing we do X, then Y. Yeah?"

This catches misunderstandings before they cause problems, shows the speaker you were
listening (which builds rapport), and — when English is a barrier — is your safeguard
against mis-hearing. It's the same move as confirming in a meeting (Lesson 29).

### Paraphrasing as a word-recovery escape hatch

When a specific word won't come, **don't freeze — describe the idea in simpler words**:

- Can't recall "idempotent"? → "it's safe to call more than once — running it twice does the
  same thing as once."
- Can't recall "throttle"? → "we'd limit how many requests go through per second."
- Can't recall "deprecate"? → "we'd mark it as old and tell people to stop using it."

You keep talking and get the idea across; nobody minds that you used more words. Often the
listener even supplies the word ("oh, idempotent?" — "yes, that!"), which is a normal, easy
exchange.

### Signposting the work-around

You can openly flag that you're reaching for a word — it's completely normal:

- "I forget the exact term, but it's the thing where…"
- "What's the word… anyway, the idea is…"
- "You know the property where running it twice is the same as once?"

Signposting keeps you fluent (you don't stall) and often prompts the listener to fill in the
word for you. Native speakers do this all the time ("what's it called…").

### Simplifying is a strength, not a compromise

Describing an idea in simpler words isn't a lesser version — it's often *clearer*. Plain
descriptions ("safe to run twice") are more universally understood than jargon
("idempotent"), so paraphrasing frequently communicates better, not worse. You're not
settling for a worse expression; you're often improving it. (This connects to plain-language
writing — Lesson 39.)

{: .note }
> **You never need a specific word — you can always describe the idea**
> Paraphrasing removes one of the biggest fears of speaking a second language: getting stuck
> because a word won't come. The reframe: any idea can be expressed by describing it in
> simpler words, so a missing or forgotten word is never a dead end — you paraphrase around
> it and keep going ("it's the thing where running it twice does the same as once"). Its
> other use is confirming understanding ("so what you're saying is…"), which catches
> misunderstandings and shows you listened — especially valuable when English is a barrier.
> Both uses make paraphrasing one of the highest-value fluency tools: it keeps you talking
> when a word escapes you, and it keeps everyone aligned. And simpler descriptions are often
> clearer than the jargon anyway.

---

## Lab — Say Drill

Practice both uses of paraphrasing.

**Your answer (write what you'd say):**

1. **Confirm understanding:** A colleague explains a complex requirement quickly: "We need
   the webhook to retry with backoff, but only for 5xx errors, and it must be idempotent."
   Paraphrase it back to confirm.
2. **Word won't come (idempotent):** You want to say the API is "idempotent" but the word
   escapes you mid-sentence. Work around it.
3. **Word won't come (bottleneck):** You're explaining a performance issue and can't recall
   "bottleneck." Describe the idea instead.
4. **Signpost a reach:** You're mid-explanation and blank on a term. Signpost it gracefully
   and keep going.

<details>
<summary>Show Model Answers</summary>
<br>
<strong>1. Confirm understanding:</strong> "Let me make sure I've got this — the webhook
should retry if it fails, waiting longer between each try, but only when the error is a 5xx
(a server error, not a client one), and it needs to be safe to call multiple times without
double-processing. Is that right?" (Says it back in own words — confirming the requirement,
catching any misunderstanding, and showing you listened. Note it even paraphrases
"idempotent" into "safe to call multiple times without double-processing," which is clearer.)
<br><br>
<strong>2. Work around "idempotent":</strong> "The API is — what's the word — it's safe to
call more than once; calling it twice does the same thing as calling it once, no double
effect. You know the property I mean?" (Describes the idea in simple words and keeps going;
the listener will likely supply "idempotent," but even if not, the meaning is fully clear —
arguably clearer than the jargon.)
<br><br>
<strong>3. Work around "bottleneck":</strong> "The database is the — it's the part that's
slowing everything else down; even though the rest is fast, everything has to wait on the
database, so it caps our overall speed." (Describes a bottleneck perfectly without the word.
Plain and clear; the listener understands completely and may offer "bottleneck.")
<br><br>
<strong>4. Signpost a reach:</strong> "We'd want to — I forget the exact term — basically
mark the old endpoint as outdated and tell everyone to move to the new one over time." (Openly
flags the reach ("I forget the exact term"), then paraphrases and continues — staying fluent,
not freezing. Someone may say "deprecate?" — "yes, exactly." Completely normal.)
<br><br>
The patterns: paraphrase back to confirm understanding (catches errors, shows listening);
and when a word won't come, describe the idea in simple words (and optionally signpost the
reach) — never freeze. The plain description is often clearer than the jargon anyway.
</details>

---

## Phrase Bank

| Use | Phrase |
|---|---|
| Confirm understanding | "So if I've got this right, you're saying… — correct?" |
| Confirm (check) | "Let me make sure I follow — you want X, then Y. Yeah?" |
| Work around a word | "It's the thing where…" / "You know the property where…?" |
| Signpost a reach | "What's the word… anyway, the idea is…" |
| Signpost (flag) | "I forget the exact term, but basically…" |
| Invite the word | "…you know the one I mean?" |
| Describe an idea | "Basically it [does / means / is where]…" |

---

## Further Reading

| Topic | Source |
|---|---|
| Confirming understanding in meetings | English track Lesson 29 (interrupting & clarifying) |
| Plain language over jargon | English track Lesson 39 (plain language) |
| Active listening (reflecting back) | Leadership track Lesson 25 (listening & questions) |

---

## Checkpoint

**Q1.** How does paraphrasing serve as an "escape hatch" when a word won't come, and why
does the reframe "you never need a specific word" remove a major fear of speaking a second
language?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Paraphrasing serves as an escape hatch when a word won't come because <strong>instead of
freezing because you can't recall (or don't know) a specific term, you describe the idea in
simpler words and keep going</strong> — the missing word stops being a wall and becomes
something you simply talk around. Concretely: if you're mid-sentence and blank on "idempotent,"
you don't stop dead; you say "it's safe to call more than once — running it twice does the same
as once," and the idea is fully communicated. You keep talking, get your point across, and the
listener understands (and often supplies the word — "oh, idempotent?" — "yes, that!", a normal
easy exchange). So the escape hatch is: <em>a word you can't produce is never a dead end,
because you can always express the underlying idea with simpler words you do have</em>. Why the
reframe "you never need a specific word" removes a major fear of speaking a second language:
<strong>one of the biggest anxieties in a second language is getting stuck — reaching for a
word that won't come and freezing, feeling exposed and derailed</strong>. This fear can make
speaking stressful (you're worried mid-sentence that a word will escape you and you'll be
stuck) and can even make you avoid speaking up or choosing to say complex things. The reframe
dissolves the fear at its root: <strong>if any idea can be expressed by describing it in
simpler terms, then no missing word can actually stop you</strong> — the worst case isn't
"freeze and fail," it's just "use a few more, simpler words," which is completely fine and
often clearer. So the situation you were afraid of (a word won't come) has a smooth, reliable
solution (paraphrase around it), which means it's no longer something to fear. This is
liberating because it removes the pressure to have perfect vocabulary recall in real time —
you don't need to know or instantly retrieve every precise term, because you can always fall
back on describing the idea. You can even signpost it openly ("I forget the exact term, but
it's the thing where…"), which native speakers do constantly ("what's it called…") and which
keeps you fluent while often prompting the listener to supply the word. And there's a bonus:
the plain description ("safe to run twice") is frequently <em>clearer</em> than the jargon
("idempotent"), so paraphrasing often communicates better, not worse — it's not a compromise,
it's often an improvement. For a non-native speaker, internalizing "you never need a specific
word" turns speaking from an anxious vocabulary-recall test into a relaxed idea-expression
exercise: you always have a way through, so a missing word is a minor detour, not a
catastrophe. That confidence — knowing you can never truly get stuck — makes speaking far less
stressful and more fluent.
</details>

**Q2.** Beyond word-recovery, why is paraphrasing to confirm understanding ("so what you're
saying is…") valuable — especially when English is a barrier?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Paraphrasing to confirm understanding is valuable because <strong>it verifies that you
correctly understood what was said, catches misunderstandings before they cause problems, and
shows the speaker you were genuinely listening — all of which matter more, not less, when
English is a barrier</strong>. When you say back what you understood in your own words ("so if
I've got this right, you want the cache invalidated on every write — correct?"), several
things happen: (1) <strong>it checks your understanding</strong> — the speaker can confirm
("yes") or correct you ("no, only on updates, not creates"), so any gap between what they
meant and what you understood surfaces immediately, while it's trivial to fix; (2) <strong>it
catches misunderstandings early</strong> — rather than proceeding on a wrong understanding and
discovering the mismatch later (after you've built the wrong thing, or agreed to something you
misread), which is far more costly to unwind; (3) <strong>it shows you listened</strong> —
paraphrasing back demonstrates you actually took in and processed what they said (not just
waited for your turn), which builds rapport and makes the speaker feel heard; (4) <strong>it
clarifies for everyone</strong> — restating a complex point in clearer words often helps the
whole room (and sometimes the speaker) confirm the shared understanding. Why this is
<em>especially</em> valuable when English is a barrier: <strong>a non-native speaker has a
higher risk of mis-hearing or misinterpreting</strong> — fast speech, unfamiliar accents,
idioms, or complex phrasing can all lead to catching the words but misreading the meaning, or
mis-hearing a detail. Paraphrasing back is your safeguard against exactly this: by saying your
understanding out loud and getting confirmation, you catch the mis-hearings and
misinterpretations that are more likely to happen across a language barrier, <em>before</em>
they turn into acting on the wrong information. Without it, a mis-hearing goes undetected and
compounds (you proceed confidently on a wrong understanding); with it, the mismatch surfaces
and gets corrected immediately. So paraphrasing to confirm converts the higher misunderstanding
risk of working in a second language into a manageable, caught-early situation — it's a
reliability tool that lets you participate accurately despite the language barrier, rather than
silently accumulating misunderstandings. It's the same move as confirming in meetings (Lesson
29) and pairs with the word-recovery use to make paraphrasing one of the highest-value fluency
tools: it keeps you unstuck when words escape you, and it keeps your understanding accurate and
aligned — both of which directly address the core challenges of operating in a second language.
</details>

---

## Homework

For the next couple of weeks, practice both uses of paraphrasing. (1) When someone explains
something important, say it back to confirm ("so if I've got this right, you want X — correct?")
— especially anything you're not 100% sure you caught. (2) When a word won't come mid-sentence,
don't freeze — describe the idea in simpler words and keep going (signpost it if you like: "I
forget the term, but it's the thing where…"). Notice that you never actually need the specific
word, and that plain descriptions are often clearer. Add to your habits: "confirmed
understanding by paraphrasing? worked around missing words instead of freezing?"

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds paraphrasing — one of the highest-value fluency tools for a non-native
speaker, with two uses. A strong response practices both: paraphrasing back to confirm
understanding (checking anything not fully caught), and paraphrasing around words that won't
come (describing the idea in simpler words instead of freezing, optionally signposting the
reach). The realizations: (1) <strong>you never actually need a specific word</strong> — any
idea can be described in simpler terms, so a missing/forgotten word is never a dead end,
removing a major fear of speaking a second language (getting stuck); (2) <strong>plain
descriptions are often clearer than jargon</strong> — "safe to run twice" beats "idempotent"
for most listeners, so paraphrasing often communicates better, not worse; (3) <strong>confirming
by paraphrasing catches misunderstandings and shows listening</strong> — especially valuable
across a language barrier where mis-hearing is likelier, it's your safeguard against proceeding
on a wrong understanding. The meta-point: paraphrasing directly addresses the two core
challenges of operating in a second language — getting stuck on missing words (word-recovery
use keeps you unstuck and fluent) and mis-hearing/misunderstanding (confirmation use keeps your
understanding accurate and aligned). Internalizing "you never need a specific word — you can
always describe the idea" turns speaking from an anxious vocabulary-recall test into a relaxed
idea-expression exercise, and confirming by paraphrasing makes you a reliable, accurate
participant despite the language barrier. Both make speaking far less stressful and more
effective. Add the paraphrasing checks to your habits. The next lesson covers asking people to
repeat or slow down — the graceful phrases for when you didn't catch something or someone's
going too fast, without embarrassment.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 36 — Asking People to Repeat or Slow Down →](lesson-36-asking-repeat){: .btn .btn-primary }
