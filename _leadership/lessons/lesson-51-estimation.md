---
title: "Lesson 51 — Estimation — and Why It Fails"
nav_order: 2
parent: "Phase 10: Project Leadership"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 51: Estimation — and Why It Fails

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **estimate / target / commitment** — honest guess / hoped-for goal / firm promise: three different things never to conflate (the lesson's core).
> - **conflate** — to blur two different things into one.
> - **unknown unknowns** — the problems you can't foresee because you don't know they exist.
> - **optimism bias / planning fallacy** — the systematic human tendency to imagine the happy path and underestimate work.
> - **the happy path** — the scenario where nothing goes wrong.
> - **pressure-contaminated** — an estimate unconsciously shrunk to please whoever wants it smaller.
> - **point estimate vs range** — a single number ("3 weeks") vs an honest spread ("2–5 weeks").
> - **reference-class forecasting** — estimating from how long *similar past projects actually took* (the "outside view") instead of imagining this one's steps (the "inside view").
> - **conservative** — deliberately cautious; commitments should be.
> - **re-estimate** — updating the estimate as you learn; honest, not shameful.

## Concept

Estimation is one of the hardest and most-failing parts of engineering leadership — estimates are almost
always wrong (usually too optimistic), and the surprises they cause damage trust and plans. But you can't
avoid estimating (people need to plan around your work), so the goal isn't perfect estimates (impossible)
but **estimating honestly, communicating uncertainty, and never being surprised the same way twice.** The
key discipline: **never conflate estimate, commitment, and target** — three different things that get
dangerously confused.

```
   THREE THINGS PEOPLE CONFUSE (keep them separate!)
   ┌──────────────────────────────────────────────────────┐
   │ ESTIMATE:   honest best guess of how long (uncertain!) │
   │ TARGET:     what we'd LIKE (a goal, aspiration)         │
   │ COMMITMENT: what we PROMISE (should be conservative)    │
   │ → conflating them = "3 weeks-ish" (estimate) becomes    │
   │   "3 weeks" (commitment) told to sales → disaster       │
   │                                                        │
   │ RANGES beat points · reference-class > gut · re-estimate│
   └──────────────────────────────────────────────────────┘
```

The reframe: **estimates are uncertain guesses (communicate the uncertainty as a range), and estimate ≠
target ≠ commitment (never conflate them).** The disaster case is when an uncertain estimate ("about 3
weeks, maybe") gets treated as a firm commitment ("3 weeks") that others promise externally — so the
uncertainty is lost and the inevitable miss becomes a broken promise. Estimating well means communicating
uncertainty honestly (ranges), grounding estimates in reality (reference-class), and keeping the three
concepts distinct.

---

## Going Deeper

### Why estimates fail

Estimates fail for structural reasons, not just carelessness: (1) **unknown unknowns** — you can't
estimate the problems you don't know exist yet (and every project has them), so estimates systematically
miss the surprises; (2) **optimism bias** — people (especially engineers) are systematically optimistic,
imagining the happy path and underweighting the things that go wrong (the planning fallacy); (3)
**pressure-contaminated estimates** — when there's pressure for a short estimate (a deadline, an eager
stakeholder), estimates get unconsciously (or consciously) shrunk to please, contaminating them with
wishful thinking rather than honest assessment. Understanding <em>why</em> estimates fail (structural, not
just "try harder") is the first step — it means you build in buffer for unknown unknowns, correct for
optimism, and protect estimates from pressure.

### Ranges beat points

A single-point estimate ("3 weeks") falsely implies precision that doesn't exist. **Ranges** ("2 to 5
weeks, most likely 3") honestly convey the uncertainty — which is real and important information. A range
communicates that it could be as short as X or as long as Y, so people plan accordingly (not treating the
midpoint as certain). Ranges also resist the conflation problem — a range is obviously an estimate (with
uncertainty), harder to mistake for a firm commitment than a single number. When you must give a single
number for a commitment, lean toward the conservative end of the range (under-promise). Always
communicate the uncertainty; a point estimate hides it.

### Reference-class forecasting — how long did similar things take?

The most reliable estimation technique: **reference-class forecasting** — instead of estimating from the
inside (imagining the steps and summing, which is optimism-prone), look at **how long similar things
actually took** ("the last three migrations like this took 2-4 months, so this will probably be similar").
The outside view (actual history of similar projects) is far more accurate than the inside view (imagining
this specific project going well), because it implicitly includes all the surprises and delays that
similar projects hit (which you'd underweight from the inside). Whenever possible, ground estimates in the
actual track record of comparable work, not just gut feel about this project.

### Estimate ≠ target ≠ commitment — never conflate

The most important discipline: keep three distinct concepts separate: (1) an **estimate** is your honest
best guess of how long something will take (inherently uncertain); (2) a **target** is what you'd
<em>like</em> — a goal or aspiration (which may be more ambitious than the estimate); (3) a **commitment**
is what you <em>promise</em> to deliver (which should be conservative — something you're confident you can
hit). These get dangerously conflated: an uncertain estimate ("about 3 weeks") gets treated as a
commitment ("we'll deliver in 3 weeks") and promised externally, or a target ("we want it in 3 weeks")
gets confused with an estimate ("it'll take 3 weeks"). The disaster: the uncertainty in the estimate is
lost when it becomes a commitment, so the inevitable variance becomes a broken promise. Always be explicit
about which you're giving — "my <em>estimate</em> is 2-5 weeks; if you need a <em>commitment</em>, I'm
confident in 6 weeks; the <em>target</em> of 3 weeks is aggressive but possible if things go well" — so
nobody mistakes an uncertain estimate for a firm promise.

### Re-estimating without shame

As you learn more (the project progresses, unknowns surface), your estimates should **change** — and
re-estimating is honest and correct, not a failure. Early estimates are made with the least information
(so they're the most uncertain); as you learn, you can estimate better, and updating the estimate reflects
that learning. Treating re-estimation as shameful ("you said 3 weeks!") pressures people to stick to wrong
estimates (or not update), which is worse. Normalize re-estimating as new information arrives (and
communicate the updates early — Lesson 55's early slip detection). The estimate at the start is a guess
with high uncertainty; refining it as you learn is good practice, not backpedaling.

{: .note }
> **Estimates are uncertain guesses — communicate ranges, and never conflate estimate/target/commitment</br>**
> Estimates almost always fail (unknown unknowns, optimism bias, pressure contamination), so the goal isn't
> perfect estimates but honest ones with communicated uncertainty. Use <em>ranges</em> (not false-precision
> points) to convey the real uncertainty. Ground estimates in <em>reference-class forecasting</em> (how long
> did similar things actually take — the outside view beats optimistic inside-view guessing). Above all,
> keep <em>estimate</em> (honest guess, uncertain), <em>target</em> (what we'd like), and <em>commitment</em>
> (what we promise — conservative) distinct — the disaster is an uncertain estimate becoming a firm external
> commitment, so the uncertainty is lost and the miss becomes a broken promise. And re-estimate without
> shame as you learn. Estimating well is about honesty and communicating uncertainty, not false precision —
> which is how you stop being surprised (and surprising others) the same way over and over.

---

## Lab — Scenario

**The situation:** Classic broken telephone. Your team, asked how long a feature will take, says **"3
weeks-ish"** (an off-the-cuff, uncertain estimate — they mean "roughly, maybe, if things go well"). Your
director hears **"3 weeks"** (drops the "-ish"), treats it as firm, and **tells sales**, who **promise a
customer** the feature in 3 weeks. Now there's an external commitment based on a fuzzy estimate — and when
it inevitably takes 5 weeks, it's a broken promise to a customer, and everyone's upset. You need to repair
this communication chain and fix the process so it stops happening.

**Write how you'd repair the immediate situation and fix the process.** Then note the principles and
mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong response repairs the immediate miscommunication and fixes the underlying estimate/commitment
conflation. Example:
<br><br>
<strong>Repair the immediate situation (early, before the miss):</strong> "As soon as I realize the chain
happened, flag it immediately (don't wait for the miss — Lesson 55/41): 'I need to correct something before
it causes a problem. The team's "3 weeks-ish" was a rough estimate, not a commitment — realistically it's
more like 3-5 weeks with the usual uncertainty, and I wouldn't commit externally to less than 5. I don't
think we should have promised a customer 3 weeks firm on a rough estimate. Let's fix the customer
expectation now — I'd rather under-promise and over-deliver. Can we tell them ~5 weeks (or a range), so we
don't break a promise?' Better to correct the expectation early than break it late." <em>(Fix the external
commitment early — reset to a conservative, honest number — rather than letting the broken promise happen.)</em>
<br><br>
<strong>Fix the process (the real problem — estimate/commitment conflation):</strong> "The root cause is
that an uncertain estimate got turned into a firm external commitment, with the uncertainty lost at each
hop. Fix the process: <br>
• <strong>Always distinguish estimate vs commitment</strong>: when the team gives a number, be explicit —
'this is a rough <em>estimate</em> (2-5 weeks), not a <em>commitment</em>.' And establish that external
commitments (what sales tells customers) come from conservative commitments, not raw estimates. <br>
• <strong>Give ranges, not points</strong>: 'the estimate is 3-5 weeks' resists being flattened to '3
weeks' and carries the uncertainty. <br>
• <strong>A clear path for external commitments</strong>: nothing gets promised to a customer without an
explicit, conservative commitment from engineering (not a fuzzy estimate overheard and passed along). Sales
should ask 'what can we commit to?' and get a conservative answer, not grab an estimate. <br>
• <strong>Buffer for external promises</strong>: external commitments should be conservative (under-promise)
— the estimate might be 3-5, but we commit to 6 externally (Lesson 44's under-promise calibration). <br>
• <strong>Educate the chain</strong>: help the director and sales understand estimate ≠ commitment, so they
stop flattening '-ish' estimates into firm promises." <em>(Fixes the structural conflation that caused it —
distinguishing estimate/commitment, ranges, a clear commitment path, buffer, and educating the chain.)</em>
<br><br>
<strong>Principles:</strong> (1) <strong>Repair early</strong> — correct the external expectation before
the miss (under-promise/over-deliver beats breaking a promise). (2) <strong>The root cause is estimate/
commitment conflation</strong> — a rough estimate became a firm commitment; fix that. (3) <strong>Ranges,
not points</strong> — '3-5 weeks' resists flattening. (4) <strong>External commitments are conservative
and explicit</strong> — nothing promised to customers without a deliberate conservative commitment (not an
overheard estimate). (5) <strong>Buffer external promises</strong> — under-promise. (6) <strong>Educate the
chain</strong> — help director/sales understand the distinction. <strong>Mistakes to avoid:</strong> (1)
<strong>letting the miss happen</strong> — waiting silently, then breaking the customer promise (fix the
expectation early); (2) <strong>blaming</strong> — the director/sales/team (it's a process failure —
estimate/commitment conflation — not individual fault); (3) <strong>just re-committing to 3 weeks</strong>
under pressure (perpetuating the problem — the estimate is 5, don't commit to 3); (4) <strong>not fixing the
process</strong> — repairing this instance but leaving the conflation to recur; (5) <strong>giving point
estimates</strong> going forward (they flatten and mislead — use ranges); (6) <strong>no clear commitment
path</strong> — letting estimates keep leaking into external promises without a deliberate conservative
commitment step.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Why estimates fail / planning fallacy | *Thinking, Fast and Slow*, Kahneman (planning fallacy) |
| Reference-class forecasting | Flyvbjerg; Kahneman (outside view) |
| Estimate vs commitment vs target | *Software Estimation*, Steve McConnell |
| Communicating uncertainty | #NoEstimates debate; ranges & confidence |

---

## Checkpoint

**Q1.** Why do estimates systematically fail, and why is it critical to never conflate estimate, target,
and commitment?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Estimates systematically fail (usually too optimistic) for <strong>structural reasons, not just
carelessness</strong>: (1) <strong>unknown unknowns</strong> — you can't estimate problems you don't yet
know exist, and every project has them (surprises, unforeseen complications, things that turn out harder
than they looked), so estimates systematically miss the time these consume (you estimate the work you can
see, but the unseen surprises aren't in the estimate); (2) <strong>optimism bias / the planning fallacy</strong>
— people (especially engineers) are systematically optimistic when estimating, imagining the happy path
(everything goes smoothly) and underweighting the things that go wrong; the planning fallacy is the
well-documented tendency to underestimate how long tasks take even when we know similar tasks took longer;
(3) <strong>pressure-contaminated estimates</strong> — when there's pressure for a short estimate (an eager
stakeholder, a desired deadline), estimates get unconsciously (or consciously) shrunk to please or to fit
the pressure, so the number reflects wishful thinking rather than honest assessment. These are
<em>structural</em> causes (built into how estimation works and human psychology), which means estimates
fail predictably and systematically (toward optimism) — not because people are careless, but because of
unknown unknowns, cognitive bias, and social pressure. Understanding this matters because it changes the
response: since the failure is structural, "try harder to estimate accurately" doesn't work; instead you
build in buffer for unknown unknowns, deliberately correct for optimism (add contingency, use reference-
class forecasting), and protect estimates from pressure (give honest ranges, distinguish estimate from
commitment). You can't make estimates accurate (the structural causes remain), but you can account for their
systematic failure. Why it's critical to <strong>never conflate estimate, target, and commitment</strong>:
<strong>because they're three different things with different levels of certainty, and confusing them —
especially treating an uncertain estimate as a firm commitment — loses the uncertainty and turns the
inevitable variance into a broken promise</strong>. The three are distinct: an <strong>estimate</strong> is
your honest best <em>guess</em> of how long (inherently uncertain — it will be wrong, as above); a
<strong>target</strong> is what you'd <em>like</em> (a goal/aspiration, possibly more ambitious than the
estimate); a <strong>commitment</strong> is what you <em>promise</em> to deliver (which should be
conservative — something you're confident you can hit). They have fundamentally different certainty: an
estimate is a wide-uncertainty guess, a commitment is a promise you should be able to keep. The danger is
<strong>conflation</strong> — treating one as another, especially an estimate as a commitment: an uncertain
estimate ("about 3 weeks, maybe") gets treated as a firm commitment ("we'll deliver in 3 weeks") and
promised externally (to a customer, in a plan). When this happens, the <em>uncertainty is lost</em> — the
"about, maybe, if things go well" gets dropped, and the fuzzy estimate becomes a hard promise. Then, when
the work takes longer than the optimistic estimate (as it systematically will — see above), the miss isn't
just "the estimate was off" (expected — estimates are uncertain) but "we broke our commitment" (a promise
failed), which damages trust, plans, and relationships (the customer was promised 3 weeks and it's 5). The
scenario (the "3 weeks-ish" → "3 weeks" → sales promise → broken commitment chain) is exactly this: an
uncertain estimate flattened into a firm external commitment, so the inevitable variance became a broken
customer promise. Keeping the three distinct prevents this: by being explicit about which you're giving
("my <em>estimate</em> is 2-5 weeks; my <em>commitment</em>, if you need one, is 6 weeks conservative; the
<em>target</em> of 3 is aggressive"), you ensure an uncertain estimate is never mistaken for a firm promise
— external commitments come from deliberate conservative commitments (with buffer), not from raw optimistic
estimates that lose their uncertainty in transmission. This is critical because the systematic optimism of
estimates (they'll usually run long) means that committing to an estimate is committing to something you'll
usually miss — so the estimate/commitment distinction (commit conservatively, not to the estimate) is what
lets you make promises you can keep despite estimates being unreliable. Together: estimates fail
structurally (so account for it — buffer, ranges, reference-class), and estimate ≠ commitment (so never
promise the uncertain estimate — commit conservatively) — which is how you estimate honestly and avoid the
broken-promise disasters that come from treating uncertain guesses as firm commitments.
</details>

**Q2.** Why do ranges beat point estimates, and why is reference-class forecasting more accurate than
estimating from the inside?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Ranges beat point estimates</strong> because <strong>a range honestly communicates the real
uncertainty (which is important information), while a single-point estimate falsely implies a precision
that doesn't exist — and a point estimate is easily mistaken for a firm commitment</strong>. Estimates are
inherently uncertain (they'll be wrong, per Q1), so the honest representation of an estimate includes its
uncertainty — how much it could vary. A <strong>range</strong> ("2 to 5 weeks, most likely 3") conveys
this: it tells people it could be as short as 2 or as long as 5, so they understand the uncertainty and can
plan accordingly (not treating any single number as certain — building in appropriate buffer, not making
firm promises on the optimistic end). A <strong>point estimate</strong> ("3 weeks") hides the uncertainty —
it presents a single number as if it were precise and certain, when it's actually a wide-uncertainty guess.
This is misleading in two ways: (1) it <em>implies false precision</em> — "3 weeks" sounds definite, so
people plan as if it's certain (and are surprised when it's 5, though 5 was well within the real
uncertainty); (2) it's <em>easily mistaken for a commitment</em> — a single number ("3 weeks") is readily
flattened into a firm promise (as in the scenario), where a range ("2-5 weeks") is obviously an uncertain
estimate, harder to mistake for a commitment (the range visibly carries the uncertainty). So ranges are
better because they communicate the true uncertainty (important for planning honestly), resist the false-
precision trap, and resist the estimate-becomes-commitment conflation. (When you must give a single number
for a commitment, lean to the conservative end of the range — under-promise.) Always communicating the
uncertainty (via a range) rather than hiding it (in a point) is more honest and leads to better planning
and fewer broken commitments. Why <strong>reference-class forecasting</strong> (how long did similar things
take) is more accurate than estimating from the inside: <strong>because the outside view (actual history of
similar projects) implicitly includes all the surprises, delays, and complications that similar projects
hit — which the inside view (imagining this project's steps) systematically underweights</strong>. There
are two ways to estimate: (1) the <strong>inside view</strong> — imagine this specific project's steps and
sum them up ("first we'll do X (2 days), then Y (3 days)...") — which is how people naturally estimate, but
it's optimism-prone: you imagine the work going well (the happy path), and you can't imagine the unknown
unknowns and surprises (you don't know what they'll be), so you systematically underestimate (the planning
fallacy — the inside view produces optimistic estimates because it models the visible work going smoothly);
(2) the <strong>outside view / reference-class forecasting</strong> — look at how long <em>similar</em>
projects actually took ("the last three migrations like this took 2-4 months") and estimate based on that
track record. The outside view is more accurate because <strong>the actual history of comparable projects
already includes all the surprises, delays, and complications those projects hit</strong> — even though you
can't predict <em>which</em> surprises this project will hit, the reference class tells you that similar
projects <em>in aggregate</em> took a certain amount of time (including their surprises), which is a much
better predictor than imagining this one going smoothly. The outside view captures the base rate (how these
things actually go), while the inside view captures only the optimistic plan (how you hope this one goes,
missing the surprises). Empirically, the outside view is far more accurate — because it's grounded in what
actually happened to similar work (surprises included), not in an optimistic imagination of this specific
work. So whenever possible, ground estimates in reference-class forecasting (the actual track record of
comparable projects) rather than inside-view gut feel — "the last similar migrations took 2-4 months, so
this will probably be similar" beats "I imagine this taking 6 weeks if it goes well." Both points improve
estimation honesty and accuracy: ranges communicate the real uncertainty (rather than false-precision
points that hide it and get mistaken for commitments), and reference-class forecasting grounds estimates in
what similar work actually took (the outside view, which includes the surprises the optimistic inside view
misses) — together producing more honest, more accurate estimates with communicated uncertainty, which is
the best you can do given that estimates are inherently uncertain and systematically optimistic.
</details>

---

## Homework

Improve your estimation practice. (1) For your next estimate, give a <em>range</em> (not a point) that
honestly conveys the uncertainty — and be explicit whether it's an estimate, a target, or a commitment
(never let them blur). (2) Try reference-class forecasting — instead of estimating from the inside, ask "how
long did similar things actually take?" (3) Watch for the estimate → commitment conflation in your org
(estimates being flattened into firm promises) — and establish that external commitments come from
conservative commitments, not raw estimates. (4) Normalize re-estimating as you learn (without shame).
Reflect: how do your estimates fail (optimism? pressure? unknown unknowns?), and what's one practice (ranges,
reference-class, the estimate/commitment distinction) that would reduce your surprises?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds estimation skill — honest estimation with communicated uncertainty. A strong response
gives ranges (explicitly labeled estimate/target/commitment), tries reference-class forecasting, watches for
the estimate→commitment conflation, and normalizes re-estimating. The realizations: (1) <strong>estimates
fail structurally</strong> — unknown unknowns, optimism bias, and pressure contamination make estimates
systematically too optimistic, so the response is to account for it (buffer, ranges, reference-class), not
"try harder"; (2) <strong>ranges beat points</strong> — a range honestly conveys the real uncertainty and
resists both false precision and being flattened into a commitment, where a point estimate hides the
uncertainty and gets mistaken for a firm number; (3) <strong>reference-class forecasting beats inside-view
guessing</strong> — how long similar things actually took (the outside view) includes the surprises that
optimistic step-by-step imagining (the inside view) misses; (4) <strong>never conflate estimate/target/
commitment</strong> — the disaster is an uncertain estimate becoming a firm external commitment (uncertainty
lost, miss becomes broken promise), so keep them distinct and commit conservatively; (5) <strong>re-estimate
without shame</strong> as you learn. On reflection, people commonly find they give point estimates (which get
treated as commitments), estimate from the inside (optimistically), and let estimates leak into firm promises
without a deliberate conservative commitment step. The highest-value change is usually: give ranges and keep
estimate/target/commitment distinct (especially: external commitments are conservative, not raw estimates),
plus use reference-class forecasting. The meta-point: estimation is one of the hardest, most-failing parts of
engineering leadership — estimates are almost always wrong (structurally optimistic) — so the goal isn't
perfect estimates but honest ones with communicated uncertainty: ranges (not false-precision points),
reference-class grounding (outside view), and never conflating estimate (uncertain guess), target (what we'd
like), and commitment (conservative promise). Estimating well is about honesty and communicating uncertainty,
not false precision — which is how you stop being surprised, and surprising others, the same way repeatedly.
This is central to project leadership (plans rest on estimates) and connects to no-surprises (Lesson 41) and
renegotiating early (Lesson 44). The next lesson turns to finding what will actually kill a project — risk
management — identifying and retiring the real risks early rather than being blindsided by them.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 52 — Risk Management →](lesson-52-risk-management){: .btn .btn-primary }
