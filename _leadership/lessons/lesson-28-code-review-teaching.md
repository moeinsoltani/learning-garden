---
title: "Lesson 28 — Code Review as Teaching"
nav_order: 5
parent: "Phase 5: 1:1s, Coaching & Mentoring"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 28: Code Review as Teaching

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **gate / gating** — acting only as an approve-or-block filter, without teaching.
> - **the pattern / principle** — the reusable general rule behind a specific fix; teaching it is this lesson's core move.
> - **comment tiers** — labelling each review comment's weight: **blocking** (must fix), **suggestion** (author's call), **nit** (minor, take it or leave it).
> - **nit / nitpick** — a tiny style-level comment.
> - **wall of red marks** — a review that's nothing but corrections; demoralizing even when each is fair.
> - **magic number** — an unexplained literal value in code (why 86400?) that should be a named constant.
> - **separation of concerns** — the design principle that one function should do one thing.
> - **edge case** — an unusual input (empty list, zero, huge value) that breaks naive code.
> - **PR (pull request)** — a proposed code change submitted for review.
> - **pair instead** — replacing a huge comment pile with a live working session; warmer and faster for big rework.

## Concept

Code review is the **highest-volume teaching channel** most leads have — you review far more code
than you have 1:1s or mentoring sessions, so how you review shapes your engineers' growth more than
almost anything. But most reviews miss the teaching opportunity: they gate the code (approve/block)
without teaching the *pattern* behind the feedback, or they're a pile of nitpicks that exhaust
rather than develop. The skill is reviewing so that people *learn* — teaching the reusable principle,
not just fixing this instance — while keeping the review efficient and the author motivated.

```
   GATE-ONLY REVIEW                 TEACHING REVIEW
   ───────────────                  ───────────────
   "change this to X"               "I'd extract this into a helper — the
   (fixes the instance,              principle is that functions doing two
    teaches nothing)                 things are hard to test and reuse. Here
   30 nitpicks, no priority          it's doing validation AND saving. Worth
   (exhausting, demoralizing)        splitting? (nit — take it or leave it)"
                                          ↑ teaches the PATTERN, labeled,
                                            so they learn AND aren't exhausted
```

The reframe: **review the code's risk, but teach the pattern.** A review that just says "change
this" fixes one line but develops no one; a review that explains the *reusable principle* behind the
change ("functions doing two things are hard to test — here's why") teaches something the author
applies everywhere after. Since you review constantly, making reviews teach — efficiently, without
exhausting — compounds into large developmental impact.

---

## Going Deeper

### Review the risk, teach the pattern

Two jobs in a review: (1) protect the codebase (catch the real risks — bugs, security,
maintainability), and (2) *teach* (help the author grow). The key teaching move: when you flag
something, explain the **reusable pattern/principle**, not just the specific fix. "Change this to X"
fixes the instance; "extract this into a helper — the principle is that a function doing two things
is hard to test and reuse" teaches something the author applies everywhere after. Teaching the *why*
and the general pattern turns each review comment into a lesson that compounds.

### Comment tiers — label them (blocking / suggestion / nit)

Label the *severity* of each comment so the author can prioritize and isn't exhausted or confused
about what's required: **blocking** (must fix before merge — a real bug/risk), **suggestion** (worth
considering, author's call), **nit** (minor/style, take it or leave it). Without labels, the author
can't tell a critical bug from a style preference (so they either agonize over everything or miss the
important one), and a pile of unlabeled comments feels like a wall of demands. Labeling respects the
author's time and makes the review navigable. (This mirrors the English track's Lesson 45.)

### Questions over commands

Frame feedback as questions/suggestions, not orders — "what do you think about extracting this?" /
"could this deadlock if two threads hit it?" rather than "extract this" / "add a lock." Questions (a)
teach (they prompt the author to think, not just comply), (b) invite dialogue (the author may have a
reason, or a better idea), and (c) are warmer (collaborative, not bossy). Especially for teaching, a
question that leads the author to see the issue themselves develops them more than a command that just
tells them what to do.

### Praise in review

Note what's *good*, not only what needs fixing — "nice clean solution here", "good call adding that
test." Praise in review (a) reinforces good patterns (teaching what to do, not just what to avoid),
(b) balances the critical comments (so the review isn't a wall of negativity that demoralizes), and
(c) makes critical feedback land better (the author knows you see the good too). A review with only
red marks feels like an attack even if each is fair; genuine praise makes it feel collaborative.

### Don't exhaust — prioritize, and know when to pair

A review that teaches everything at once exhausts and demoralizes. **Prioritize**: focus on the
important things (the real risks and the highest-value teaching), and let minor stuff go or mark it
clearly as optional — a junior's PR with 40 comments is crushing even if each is valid. Teach a *few*
high-value lessons per review, not every possible improvement. And **know when to pair instead**: if
a PR needs extensive rework or a big design change, a pile of comments is the wrong tool — a quick
pairing session or conversation teaches better and faster than 30 comments (and is warmer). Text
amplifies the exhaustion; a conversation resolves it.

{: .note }
> **Code review is your highest-volume teaching channel — teach the pattern, don't just gate</br>**
> You review far more code than you have 1:1s, so how you review shapes engineers' growth more than
> almost anything. Most reviews miss it — gating the code without teaching the reusable pattern, or
> piling on nitpicks that exhaust. Review so people <em>learn</em>: teach the principle behind the fix
> (not just "change this"), label comment severity (blocking/suggestion/nit, so they can prioritize
> and aren't overwhelmed), frame feedback as questions (which teach and invite dialogue), praise the
> good (reinforcing patterns and balancing criticism), prioritize a few high-value lessons per review
> (don't exhaust), and pair instead when a PR needs extensive change. Since you review constantly,
> making reviews teach — efficiently and warmly — compounds into large developmental impact, which is
> why it's the highest-leverage teaching a lead does.

---

## Lab — Scenario

**The situation:** A junior engineer, Amir, submits a PR. It's *functionally* fine — it works and
passes tests — but it's structurally naive: one big function doing several things (validation,
business logic, and database save all mixed together), no separation of concerns, a magic number or
two, and a missing test for an edge case (empty input). It's the kind of PR where you could leave 25
comments, but that would crush him. You want to review it so he *learns* — without exhausting him.

**Write the review** — the actual comments you'd leave (and how you'd structure it). Then note what
makes it teach without exhausting, and the mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong teaching review prioritizes a few high-value lessons, teaches the pattern (not just the fix),
labels severity, uses questions, praises the good, and doesn't pile on. Example:
<br><br>
<strong>Opening comment (set the tone, praise):</strong> "Nice work, Amir — this works and the tests
pass, which is great. I left a few comments; most are learning-oriented (not blockers), and I want to
highlight one or two patterns worth internalizing rather than nitpicking everything. Happy to pair on
any of it!"
<br><br>
<strong>The one high-value teaching comment (the big function):</strong> "The main thing I'd suggest,
and it's a really useful pattern to learn: this function is doing three different things — validating
the input, running the business logic, and saving to the DB. When a function does multiple things, it
gets hard to test each part, hard to reuse, and hard to read. The principle is 'single
responsibility' — each function does one thing. Here, I'd split it into <code>validate()</code>,
the core logic, and <code>save()</code>. Want to give that a try? This one's worth doing (suggestion,
not strictly blocking) because it's a pattern you'll use constantly. Happy to pair if useful." <em>(The
key teaching comment — teaches the reusable principle (single responsibility) with the why, not just
"split this"; framed as a suggestion with an offer to pair.)</em>
<br><br>
<strong>The one blocking comment (the real risk):</strong> "blocking: I think this'll break on empty
input — if <code>items</code> is empty, the <code>items[0]</code> on line 20 will throw. Could you add
a guard and a test for the empty case? Good habit to always test the empty/edge cases." <em>(The
genuine bug — labeled blocking (must fix), explains the issue, and teaches the habit (test edge
cases).)</em>
<br><br>
<strong>A nit or two (labeled, minimal):</strong> "nit (take it or leave it): the <code>86400</code>
on line 30 is a magic number — a named constant like <code>SECONDS_PER_DAY</code> reads more clearly.
Small thing." <em>(One nit, clearly labeled optional, so it doesn't carry the weight of the real
stuff.)</em>
<br><br>
<strong>What I'd deliberately NOT comment on:</strong> other minor style things, additional
improvements, every possible nitpick — because the goal is a few high-value lessons, not exhaustive
critique. If there were many issues, I'd focus on the top 2–3 and possibly pair rather than comment.
<br><br>
<strong>What makes it teach without exhausting:</strong> (1) <strong>Prioritizes a few high-value
lessons</strong> — the single-responsibility pattern (the big one worth learning) + the real bug +
one nit, not 25 comments; a focused review teaches without crushing. (2) <strong>Teaches the pattern,
not just the fix</strong> — the main comment explains the reusable principle (single responsibility,
and <em>why</em>) that Amir will apply everywhere, not just "split this function." (3) <strong>Labels
severity</strong> — blocking (the bug) vs suggestion (the refactor) vs nit (the magic number) — so
Amir knows what's required vs optional and isn't overwhelmed. (4) <strong>Uses questions and
offers</strong> — "want to give that a try?", "happy to pair" — collaborative and teaching, not
commanding. (5) <strong>Praises and sets a warm tone</strong> — opens with genuine praise, frames it
as learning not nitpicking, so it feels supportive not like an attack. (6) <strong>Offers to
pair</strong> — for the bigger refactor, pairing may teach better than comments. <strong>Mistakes to
avoid:</strong> (1) <strong>25 comments</strong> — crushing and demoralizing even if each is valid;
prioritize. (2) <strong>Just saying "split this function"</strong> without teaching the principle —
fixes the instance, teaches nothing reusable. (3) <strong>No severity labels</strong> — Amir can't
tell the real bug from the magic-number nit, so he either agonizes over all or misses the important
one. (4) <strong>Commands not questions</strong> — "extract this, add a lock, rename this" — bossy and
non-teaching. (5) <strong>Only criticism, no praise</strong> — a wall of red feels like an attack;
balance it. (6) <strong>Nitpicking everything to show thoroughness</strong> — exhausts the author and
buries the important lessons among trivia. (7) <strong>Reviewing structure the codebase doesn't
need</strong> or imposing personal style as if it were blocking — distinguish real risk/valuable
teaching from preference. (8) <strong>Not pairing when it'd be better</strong> — if the PR needed a
big rework, a conversation beats a comment-pile.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| The language of review comments | English for Work track, Lesson 45 (code-review comments) |
| Code review standards | [Google's code review guidelines](https://google.github.io/eng-practices/review/) |
| Review as teaching | Lesson 27 (mentoring); Lesson 19 (feedback) |
| Comment tiers convention | Conventional Comments — <https://conventionalcomments.org/> |

---

## Checkpoint

**Q1.** Why is code review a lead's "highest-volume teaching channel," and what does "teach the
pattern, don't just gate" mean?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Code review is a lead's highest-volume teaching channel because <strong>you review far more code than
you have 1:1s, mentoring sessions, or any other developmental interaction — so the cumulative
developmental impact of how you review is larger than almost any other channel</strong>. Consider the
volume: 1:1s happen weekly or biweekly (a few per person per month); dedicated mentoring sessions are
occasional; but code review happens on essentially every change every engineer makes — many reviews
per person per week. So across all that volume, the way you review — whether each review teaches
something or just gates the code — accumulates into a large effect on your engineers' growth. If every
review teaches a little, that compounds into significant development over the hundreds of reviews a
year; if reviews just approve/block without teaching, that enormous channel is developmentally wasted.
This makes review a uniquely high-leverage teaching opportunity precisely because of its frequency:
it's the one developmental touchpoint that happens constantly, so making it teach turns your highest-
volume interaction into your highest-volume teaching. What "teach the pattern, don't just gate"
means: <strong>when you flag something in review, explain the reusable principle/pattern behind the
feedback (so the author learns something they apply everywhere), rather than just correcting this
specific instance (which fixes the code but develops no one)</strong>. A review has two jobs: gate the
code (protect the codebase — catch real risks) and teach (develop the author). Most reviews do only
the first: "change this to X", "extract this", "add a lock" — these fix the specific instance and
protect the code, but they teach nothing reusable (the author complies without learning <em>why</em>,
so they'll make the same class of mistake again). "Teach the pattern" means adding the <em>why</em>
and the general principle: instead of "split this function", say "this function is doing three things
— validation, logic, and saving; the principle is single responsibility (one function, one job),
because multi-purpose functions are hard to test and reuse — worth splitting." Now the author learns
the reusable pattern (single responsibility, and why it matters), which they'll apply to every future
function — the comment taught something that compounds, not just fixed one instance. So "teach the
pattern, don't just gate" is the difference between a review that develops the engineer (teaches the
reusable principle, so they grow and don't repeat the mistake) and one that merely corrects the code
(fixes this instance, teaches nothing, so they keep needing the correction). Given review's high
volume, teaching the pattern in each review compounds into large developmental impact — every review
becomes a small lesson that builds the engineer's judgment and skill over time. This is why review is
such a powerful (and underused) teaching channel: not because any single review teaches a lot, but
because reviews are so frequent that consistently teaching the pattern (rather than just gating)
turns your most common interaction into your most impactful development tool — while gating-only
reviews waste that enormous opportunity. The lead who reviews to teach (pattern + why) develops their
engineers far faster than one who reviews only to gate, simply because of how much review happens.
</details>

**Q2.** Why is labeling comment severity (blocking / suggestion / nit) and not overwhelming the
author with comments important for a review that teaches?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Both matter because <strong>a review only teaches and develops if the author can actually engage with
it constructively — and an unlabeled or overwhelming pile of comments defeats that by confusing
priorities and demoralizing the author</strong>. On <strong>labeling comment severity</strong>
(blocking / suggestion / nit): without severity labels, all comments look equally weighty, so the
author can't tell a critical must-fix bug from a minor style preference — which causes several problems:
(1) they may <strong>spend equal energy on trivial and critical things</strong> (agonizing over a
magic-number nit as much as a real bug); (2) they may <strong>feel every comment is a required
change</strong>, turning a review with a couple of real issues and some optional musings into what
feels like a long list of demands — demoralizing and time-consuming; or (3) they may <strong>miss the
important one</strong> among the pile (the real bug buried among style nits). Labeling fixes this:
"blocking" (must fix before merge — a real bug/risk), "suggestion" (worth considering, your call),
"nit" (minor, take it or leave it) — so the author can immediately see what's required versus optional
and prioritize accordingly (fix the blockers, weigh the suggestions, take or leave the nits). This
(a) respects their time and judgment (they know what actually matters), (b) reduces the overwhelm and
friction (most things are explicitly optional), and (c) makes the review navigable. For teaching
specifically, labeling matters because an author who's overwhelmed or confused about priorities can't
absorb the lessons — they're too busy feeling piled-on; clear severity lets them focus their learning
energy on the high-value comments. On <strong>not overwhelming with comments</strong>: a review that
comments on <em>everything</em> — every possible improvement, every nitpick — exhausts and demoralizes
the author, which defeats teaching in several ways: (1) <strong>it crushes motivation</strong> — a
junior's PR with 25–40 comments feels like an attack and a failure, even if each comment is valid; a
demoralized author isn't in a state to learn (they're discouraged, defensive, or overwhelmed). (2)
<strong>it buries the important lessons</strong> — the few high-value teaching points get lost among
the mass of trivia, so the author can't tell what really matters and doesn't internalize the key
lessons. (3) <strong>it's counterproductive to development</strong> — people learn a few things well at
a time, not 40 things at once; a firehose of feedback teaches less than a focused few high-value
lessons. So the discipline is to <strong>prioritize</strong>: focus on the important things (real
risks + highest-value teaching), teach a <em>few</em> key lessons per review, and let minor stuff go or
mark it clearly optional — so the author can absorb the important lessons without being crushed. And
<strong>know when to pair instead</strong>: if a PR needs extensive rework, a comment-pile is the wrong
tool (overwhelming and inefficient) — a conversation or pairing session teaches better and warmer.
Both points serve the same goal: <strong>a review teaches only if the author can engage with it
constructively</strong>, which requires them to (a) know what actually matters (severity labels, so
they prioritize rather than drown or miss the point) and (b) not be overwhelmed or demoralized (a
focused few lessons, not an exhausting pile, so they stay motivated and can absorb the learning). An
unlabeled, overwhelming review — even if every comment is correct — fails to teach because it confuses
and demoralizes; a labeled, prioritized, focused review develops the author because they can see what
matters, aren't crushed, and can internalize a few high-value lessons. Given that review is the
highest-volume teaching channel, making each review teachable (labeled, prioritized, warm — plus
teaching the pattern and praising the good) is what turns that volume into actual development rather
than exhaustion.
</details>

---

## Homework

Look at your recent code review comments (or your next few reviews) through a teaching lens. (1) Do
your comments teach the <em>pattern</em> (the reusable principle + why) or just fix the instance? Pick
one and rewrite it to teach. (2) Do you label severity (blocking/suggestion/nit)? Start if not. (3)
Are you framing feedback as questions, praising the good, and prioritizing a few high-value lessons —
or piling on nitpicks? (4) Is there a PR that would be better handled by pairing than comments? Reflect:
does your review style develop people or just gate code, and what's one change that would make your
reviews teach more (and exhaust less)?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds review-as-teaching — turning the highest-volume teaching channel into deliberate
development. A strong response examines real review comments: do they teach the pattern or just fix the
instance? are they severity-labeled, question-framed, praising, and prioritized (not a nitpick pile)?
would some be better as pairing? — and rewrites a comment to teach. The realizations: (1) <strong>code
review is the highest-volume teaching channel</strong> — you review far more than you have 1:1s, so how
you review shapes growth more than almost anything, and making each review teach compounds; (2)
<strong>teach the pattern, don't just gate</strong> — explaining the reusable principle + why (not just
"change this") develops the author (they apply it everywhere), where gating-only fixes the instance and
teaches nothing; (3) <strong>label severity and don't overwhelm</strong> — blocking/suggestion/nit lets
the author prioritize and not feel piled-on, and a focused few high-value lessons teach better than 25
comments that crush; (4) <strong>questions, praise, and pairing</strong> — questions teach and invite
dialogue, praise reinforces good patterns and balances criticism, and pairing beats a comment-pile for
big reworks. On reflection, people commonly find their reviews gate more than teach (correcting
instances without the reusable principle), lack severity labels (so authors can't prioritize), and
sometimes over-comment (exhausting juniors). The highest-value change is usually: teach the pattern
(add the why/principle), label severity, and prioritize a few high-value lessons rather than piling on.
The meta-point: because review is so frequent, it's a lead's highest-leverage teaching channel — but
most reviews waste it by gating without teaching or by exhausting with nitpicks. Reviewing to teach
(pattern + why), labeled, question-framed, praising, prioritized, and warm (pairing when better) turns
your most common interaction into your most impactful development tool. This applies the mentoring
(Lesson 27), coaching (Lesson 26), and feedback (Lesson 19) skills to the daily practice of review, and
it pairs with the English track's Lesson 45 (the language of review comments). Developing people through
review, at the volume review happens, is enormous cumulative leverage. The last Phase 5 lesson covers a
different, often-overlooked growth tool — sponsorship — spending your credibility on others' visibility,
which does something mentoring alone can't.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 29 — Sponsorship →](lesson-29-sponsorship){: .btn .btn-primary }
