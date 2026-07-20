---
title: "Lesson 03 — Articles II: When to Use Nothing"
nav_order: 3
parent: "Phase 1: Sentence Mechanics"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 03: Articles II — When to Use Nothing

## Concept

Lesson 2 covered *a/an* vs *the*. This lesson covers the third option that causes
the most errors: **no article at all** (the "zero article"). Many non-native
writers add *the* or *a* where English uses nothing — "the production," "a
feedback," "the management" — and these small errors are very noticeable.

```
   THREE choices, not two:
   a/an  → introducing one countable thing        "a bug", "an error"
   the   → a specific known thing                  "the bug we found"
   ∅     → NO article                              "tests are failing",
           (general statements, uncountable nouns,  "I need access",
            proper names)                            "in production"
```

The two big rules for using no article: (1) **plural and uncountable nouns making
general statements** take no article ("tests are failing" — tests in general, not
specific ones; "code should be reviewed"), and (2) many **tech/abstract nouns are
uncountable** — you can't put *a* in front of them or make them plural
("feedback," "information," "access," "code," "software," "progress").

---

## Going Deeper

### General statements: no article

When you talk about something *in general* (all of them, the concept), not
specific ones, use no article — with plurals and uncountables:

- ✓ "**Tests** are failing." (tests in general — the failing ones, but as a
  general statement) — NOT "the tests are failing" *unless* you mean specific,
  known tests.
- ✓ "**Code** should be reviewed before merging." (code in general — the concept)
- ✓ "We deploy to **production** on Fridays." (production as a place/concept — no
  "the")
- ✓ "**Bugs** happen." (bugs in general)

The contrast: "**The** tests are failing" is correct when you mean *specific*
tests you both know about ("the tests [for the new feature] are failing"). No
article = general; *the* = specific/known. Both are right in different situations —
the error is using *the* when you mean the general case.

### Uncountable nouns: no *a*, no plural

Some nouns can't be counted, so they never take *a/an* and never become plural.
Many common tech words are uncountable — this is the source of the classic errors:

- **feedback** — ✗ "a feedback," "feedbacks" → ✓ "feedback," "some feedback," "a
  piece of feedback"
- **information** — ✗ "an information," "informations" → ✓ "information," "some
  information"
- **access** — ✗ "an access" → ✓ "access," "I need access"
- **code** — ✗ "a code" (meaning source code), "codes" → ✓ "code," "some code"
- **software, hardware, progress, advice, research, work** — all uncountable

To count them, use a "unit" word: "a **piece of** feedback," "a **line of** code,"
"a **bit of** advice."

### Proper names and "production"

Product names, company names, and specific service names take no article: ✓ "We
use **Postgres**," "deploy to **Kubernetes**," "**Slack** is down." And words like
**production, staging, prod, dev** used as environment names take no article: ✓
"It's broken **in production**" — NOT "in the production."

{: .note }
> **Your top offenders</br>**
> The single most common article errors for non-native engineers are:
> "<strong>the</strong> production" (should be "production" — no article for the
> environment name), "<strong>a</strong> feedback" / "feedback<strong>s</strong>"
> (feedback is uncountable — "feedback" or "some feedback"), and "<strong>an</strong>
> information" / "information<strong>s</strong>" (also uncountable). If you fix just
> these three, your writing gets noticeably more accurate. Put them on your
> checklist: <em>production</em> (no "the"), <em>feedback</em> (no "a", no "s"),
> <em>information</em> (no "a", no "s").

---

## Lab — Rewrite Drill

Each message has an article error — an extra *the*/*a* that should be nothing, or
an uncountable noun made countable. Fix each.

**Your rewrite (fix each):**

1. "Can you give me a feedback on my PR?"
2. "The tests are failing." (context: you mean testing in general is broken across the repo, no specific set)
3. "I deployed it to the production this morning."
4. "I need more informations about the outage."
5. "Thanks for the feedbacks, I'll make the changes."
6. "We should write a documentation for this."
7. "The code should be reviewed before we merge."  (context: general principle)
8. "I don't have an access to the staging environment."
9. "Let me do a research and get back to you."
10. "The bugs happen when you deploy on Friday." (context: general statement)

<details>
<summary>Show Model Rewrites</summary>
<br>
<strong>1.</strong> "Can you give me a feedback on my PR?" → <strong>"Can you give
me feedback on my PR?"</strong> (or "some feedback"). Feedback is uncountable — no
"a," no plural. To count it: "a piece of feedback."
<br><br>
<strong>2.</strong> "The tests are failing." → <strong>"Tests are failing across
the repo."</strong> (For the general meaning, drop "the." "The tests are failing"
would mean specific known tests — correct in a different context, but here you mean
tests in general.)
<br><br>
<strong>3.</strong> "I deployed it to the production this morning." → <strong>"I
deployed it to production this morning."</strong> (No "the" before the environment
name "production." Same for "staging," "prod," "dev.")
<br><br>
<strong>4.</strong> "I need more informations about the outage." → <strong>"I need
more information about the outage."</strong> (Information is uncountable — no plural
"informations," no "an information.")
<br><br>
<strong>5.</strong> "Thanks for the feedbacks, I'll make the changes." →
<strong>"Thanks for the feedback, I'll make the changes."</strong> (Feedback is
uncountable — "the feedback" is fine here since it's specific/known, but never
"feedbacks." Note "the changes" is correct — changes IS countable and these are
specific.)
<br><br>
<strong>6.</strong> "We should write a documentation for this." → <strong>"We
should write documentation for this."</strong> (or "some docs"). Documentation is
uncountable — no "a." (Casually, "write docs" — "docs" is countable and common.)
<br><br>
<strong>7.</strong> "The code should be reviewed before we merge." → <strong>"Code
should be reviewed before we merge."</strong> (For the general principle, drop
"the." "The code should be reviewed" is correct if you mean specific code — but as
a general rule, no article.)
<br><br>
<strong>8.</strong> "I don't have an access to the staging environment." →
<strong>"I don't have access to the staging environment."</strong> (Access is
uncountable — no "an." Note "the staging environment" is correct — a specific known
one.)
<br><br>
<strong>9.</strong> "Let me do a research and get back to you." → <strong>"Let me
do some research and get back to you."</strong> (Research is uncountable — no "a,"
no plural. Use "some research" or "look into it.")
<br><br>
<strong>10.</strong> "The bugs happen when you deploy on Friday." → <strong>"Bugs
happen when you deploy on Friday."</strong> (For the general statement, drop "the."
"The bugs" would mean specific known bugs.)
<br><br>
The two patterns: (a) drop <em>the</em> for general statements with plurals/
uncountables (tests, code, bugs <em>in general</em>) and for environment names
(production, staging); (b) don't put <em>a</em> or a plural on uncountable nouns
(feedback, information, access, research, documentation) — use "some" or a unit
word ("a piece of feedback").
</details>

---

## Phrase Bank

The uncountables you'll use most, done right:

| Wrong | Right |
|---|---|
| "a feedback" / "feedbacks" | "feedback" / "some feedback" / "a piece of feedback" |
| "an information" / "informations" | "information" / "some information" |
| "an access" | "access" / "I need access" |
| "in the production" | "in production" |
| "a research" | "some research" / "research" |
| "a documentation" | "documentation" / "docs" |
| "a progress" | "progress" / "some progress" |
| "an advice" | "advice" / "a piece of advice" |

---

## Further Reading

| Topic | Source |
|---|---|
| Zero article / no article | British Council — <https://learnenglish.britishcouncil.org/grammar/b1-b2-grammar/no-article> |
| Countable and uncountable nouns | Cambridge — <https://dictionary.cambridge.org/grammar/british-grammar/countable-and-uncountable-nouns> |
| Common uncountable nouns | Purdue OWL — <https://owl.purdue.edu/owl/> |

---

## Checkpoint

**Q1.** When do you use *no* article (rather than *a* or *the*)? Give the two main
rules with an example each.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Two main rules. (1) <strong>General statements with plural or uncountable nouns
take no article</strong> — when you're talking about something in general (the
concept, all of them), not specific known ones: "Tests are failing" (tests in
general — a general statement), "Code should be reviewed before merging" (code as a
concept), "Bugs happen" (bugs in general). The contrast is with <em>the</em>, which
you'd use for <em>specific, known</em> ones: "The tests [for the new feature] are
failing" (specific). So no article = general; <em>the</em> = specific — and the
common error is using <em>the</em> when you mean the general case. (2)
<strong>Uncountable nouns never take <em>a/an</em> and never become plural</strong>
— many common tech and abstract words are uncountable: feedback, information,
access, code, software, research, documentation, progress, advice. You can't say
"a feedback" or "feedbacks" — just "feedback" or "some feedback" (and to count it,
use a unit word: "a piece of feedback"). Example: "Can you give me feedback?" (not
"a feedback"); "I need information" (not "informations"). A third practical case:
<strong>proper names and environment names take no article</strong> — "deploy to
production" (not "the production"), "we use Postgres," "Slack is down." The
underlying idea: <em>a/an</em> and <em>the</em> are for specific countable things
(one bug, the bug), while no-article covers the general concept, the uncountable
mass, and the named thing — three situations where English simply uses nothing.
For a non-native writer, the two highest-value fixes are dropping <em>the</em> from
environment names ("in production," not "in the production") and never adding
<em>a</em>/plural to uncountables ("feedback," not "a feedback" or "feedbacks") —
these are the most common and most noticeable article errors.
</details>

**Q2.** "Thanks for the feedbacks" has two things to check. What's wrong, and what
about "the changes" in "I'll make the changes" — is that also wrong?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
"Thanks for the feedback<strong>s</strong>" has one clear error: <strong>"feedbacks"
is wrong because feedback is uncountable</strong> — it never takes a plural "s."
The correct form is "the feedback" (singular). Note that "<strong>the</strong>
feedback" here is actually fine — <em>the</em> is correct because you're referring
to specific feedback you received (known from context), and uncountable nouns
<em>can</em> take <em>the</em> when they're specific ("the feedback you gave me").
What they can't take is <em>a/an</em> or a plural "s." So the fix is just dropping
the "s": "Thanks for the feedback." (Or "Thanks for your feedback.") Now, "I'll
make <strong>the changes</strong>" — is that wrong? <strong>No, it's correct.</strong>
This is the important contrast: "changes" is a <em>countable</em> noun (you can have
one change, two changes — they're discrete, countable things), so it <em>can</em>
be plural, and "the changes" is right because you mean specific changes (the ones
from the feedback — known from context). So the sentence "Thanks for the feedback,
I'll make the changes" is correct: <em>feedback</em> stays singular (uncountable),
<em>changes</em> is plural (countable, and specific so it takes <em>the</em>). The
lesson: you have to know which nouns are countable (bug, error, change, ticket, PR
— can be plural, take a/an) and which are uncountable (feedback, information, access,
code, research — no a/an, no plural), because they behave differently. The tech
uncountables (feedback, information, access, code, software, documentation,
progress, research, advice) are the ones to memorize, because they're common and
the "a feedback / feedbacks" error is very noticeable — everything else is usually
countable and behaves normally.
</details>

---

## Homework

Scan your recent messages for the specific uncountable-noun offenders: *feedback,
information, access, code, research, documentation, progress*. Did you ever write
"a [one of these]" or add an "s"? Also check for "the production" / "the staging"
(should be no "the"). Fix any you find. Then, from Lessons 2 and 3 together, write
your personal article rule in one line — the check you'll run before sending. Which
of the three article errors is most yours: dropping articles, wrong a/the, or
adding article to uncountables/environments?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise pins down which of the three article-error types is <em>yours</em>,
which is what makes your checklist efficient. The three types, from Lessons 2–3:
(1) <strong>dropping articles</strong> where English needs one ("I found bug" →
"I found a bug"); (2) <strong>wrong a/the</strong> — the new-vs-known choice made
wrong ("I fixed a bug" when you mean the specific one just mentioned → "the bug");
(3) <strong>adding an article where English uses none</strong> — "the production,"
"a feedback," "an information" (the zero-article errors of this lesson). Most people
lean toward one or two of these, and knowing which is yours lets you build a
targeted check rather than trying to master all of English article usage (which is
genuinely hard and partly done by feel even by natives). A strong response
identifies the personal pattern honestly and writes a one-line rule — for example:
"Before sending, check: (a) did I put 'the' before an environment name or an
uncountable? (production, feedback, information — drop it), and (b) did I add 'a' or
's' to feedback/information/access/code/research? (remove it)." — if your pattern is
the zero-article errors; or "Before sending, check every noun has its article
(don't drop 'a'/'the')" — if your pattern is dropping them; or the "which one?"
check — if your pattern is a/the confusion. The reassuring meta-point (worth
internalizing across this whole phase): you do <em>not</em> need to master English
articles comprehensively — that's a multi-year project and partly intuition. You
need to catch <em>your</em> specific recurring article error, which is usually just
one or two patterns, with a quick pre-send check. For most non-native engineers, the
highest-value single fix is the uncountable/environment errors from this lesson
("production" not "the production"; "feedback" not "a feedback/feedbacks") because
they're both common and very noticeable — so if those are yours, targeting them
gives the biggest visible improvement. Add your identified pattern to the growing
checklist (Lesson 9). Articles will never be effortless, but the targeted check
makes your writing markedly more accurate, and you've now covered the three main
error types — the rest is practice noticing your pattern until the check becomes
automatic. Next lesson moves from articles to verb tenses — the three tenses that
cover almost all workplace writing.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 4 — Verb Tenses for Work →](lesson-04-verb-tenses){: .btn .btn-primary }
