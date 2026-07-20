---
title: "Lesson 10 — Driving Technical Standards"
nav_order: 5
parent: "Phase 2: Technical Leadership"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 10: Driving Technical Standards

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **paved road / guardrail / gate** — the easy supported default / the warning that catches danger / the hard block (the lesson defines them fully).
> - **decree** (dih-KREE) — an order imposed from above without consultation.
> - **malicious compliance** — obeying an order's exact letter in a way that defeats its purpose.
> - **linter / CI check / formatter** — automated tools that enforce code rules impersonally.
> - **style police** — the resented person who nitpicks everyone's code against personal taste; **nitpick** — to criticize tiny unimportant details.
> - **flag day** — an abrupt everyone-switches-at-once change (risky; opposite of gradual).
> - **early adopters** — the willing first users of a change, whose success persuades the rest.
> - **social proof** — people adopting something because respected peers already did.
> - **ratchet** (RATCH-et) — a mechanism that only turns one way; "ratcheting" = tightening a standard gradually, never loosening.
> - **scaffold / generator / template** — tooling that creates new projects pre-configured the standard way.
> - **buy-in** — genuine agreement and support, not just compliance.

## Concept

Teams need shared standards — consistent testing, error handling, code style,
architecture patterns — or the codebase becomes N different codebases and every
context-switch between areas is a fresh learning curve. But new leads who try to
*impose* standards by decree ("from now on, everyone will...") get resentment,
malicious compliance, or quiet ignoring. The skill is raising the bar without
becoming the style police.

The key distinction is *how* a standard is enforced:

```
   PAVED ROAD   the easy, supported, default path — do it the standard way
                and everything just works (tooling, templates, docs).
                Deviating is allowed but you're on your own.
                → adoption by making the right way the EASY way

   GUARDRAIL    catches you before you go off a cliff — warns/blocks on the
                genuinely dangerous, but leaves room within the safe zone.
                → automated where possible (linters, CI checks)

   GATE         a hard requirement that blocks (CI fails, PR can't merge).
                → reserve for things that MUST hold; overuse breeds resentment
```

The most effective standards work by making the standard path the *path of least
resistance* (paved roads) — engineers follow it because it's easier, not because
they're forced. The least effective is decree without support (a document saying
"you must write tests" with no tooling, examples, or reason). And automation
(linters, CI, formatters) is the lead's friend: a standard enforced by a tool is
impersonal and consistent, removing the lead from the position of nagging
enforcer.

---

## Going Deeper

### Which standards are worth having (and which aren't)

Not every consistency is worth enforcing. Good standards target things where
consistency has real value: **safety** (error handling, security patterns —
where deviation causes real harm), **maintainability at scale** (architecture
patterns, testing — where the codebase's long-term health depends on it), and
**reducing friction** (formatting, structure — where consistency lets anyone work
anywhere without re-learning). Bad standards enforce personal preference dressed
as principle (tabs vs spaces beyond "pick one and automate it"; naming bikeshedding;
"the way I like it"). The test: does this standard serve the team's real goals
(safety, maintainability, velocity), or is it enforcing taste? Enforce the former,
let go of the latter — and where it genuinely doesn't matter which choice, *pick
one and automate it* so no one wastes energy debating it.

### Paved roads over mandates

The highest-leverage way to drive a standard is to make it the easy default: a
project template that has testing set up, a shared library that handles the error
pattern correctly, a generator that scaffolds the standard structure, good docs
and examples. Now doing it the standard way is *easier* than not — and engineers
adopt it because it saves them effort, not because they're told to. This is how
platform teams and good tooling drive consistency at scale: not by policing, but
by making the right thing the low-friction thing. Google's engineering practices,
the internal-platform movement, and "developer experience" work are all this idea.

### Getting adoption without decree

When you can't just build a paved road (a behavioral standard like "write tests"),
adoption is a *change-management* problem (Phase 7, driving change), not a decree:
build buy-in (why does this standard matter — connect it to pain the team feels),
start with willing early adopters and let success spread, make it easy (examples,
pairing, tooling), and use social proof and gradual ratcheting rather than a
flag-day mandate. A standard the team *agreed* is worth having (because they
understand the value and had input) sticks; a standard imposed on them is resented
and evaded. The lead's move is usually to facilitate the team *choosing* the
standard, not to hand it down.

### Automate the enforcement, humanize the intent

Where a standard can be checked by a machine (formatting, linting, test coverage
thresholds, security scans), automate it — the CI check is an impersonal,
consistent enforcer that never plays favorites and removes the lead from the
nagging-enforcer role (nobody resents the linter the way they resent a person
constantly telling them their code is wrong). But keep the *intent* human: the
tool enforces the rule, the lead explains the *why* and handles the judgment calls
the tool can't (when to grant an exception, when a rule is doing more harm than
good). Automate the mechanical, reserve the human attention for the judgment.

{: .note }
> **The style-police trap</br>**
> The fastest way to make yourself resented and to make standards counter-
> productive is to become the person who constantly nitpicks everyone's code
> against your preferences. It positions you as an adversary, wastes your leverage
> on trivia, and teaches the team that standards are about your ego, not their
> benefit. Escape it two ways: <em>automate</em> the mechanical standards (let the
> formatter/linter enforce style so you never have to), and <em>reserve your
> personal attention</em> for the standards that genuinely matter and require
> judgment (architecture, safety) — delivered as shared team norms the team bought
> into, not as your personal decrees.

---

## Lab — Scenario

**The situation:** Your team of eight is split on testing. Four engineers write
tests religiously — good coverage, they catch bugs before production. Four write
few or no tests — they ship fast but their code causes a disproportionate share
of production incidents, and the well-tested engineers are frustrated at cleaning
up after them. There's no team testing standard. You want to establish a real one.
Critically: **you cannot simply decree it** (you tried "everyone must write tests"
in a meeting once and it changed nothing — the non-testers nodded and continued
as before).

**Design your path to a real, adopted testing standard. What do you actually do,
in what order?**

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong response treats this as a change-management and buy-in problem (not a
decree problem — decree already failed), and uses the paved-road + gradual-
ratchet + automate approach rather than mandating. A sequence:
<br><br>
<strong>1. Diagnose why the non-testers don't test (before prescribing).</strong>
Don't assume laziness. Talk to them: is it that they don't see the value (they
think their code is fine)? Is testing hard in this codebase (no test
infrastructure, painful to set up, slow)? Is there time pressure (they're rewarded
for shipping fast, and tests feel like they slow that)? Different causes need
different fixes — and often the non-testers have a real point (maybe testing
<em>is</em> genuinely painful here, which is fixable and would help everyone). This
is Rumelt's diagnose-first (Lesson 6) applied to a standards problem.
<br><br>
<strong>2. Build buy-in by connecting to shared pain.</strong> Make the cost
visible and shared: the non-testers' code causes a disproportionate share of
incidents (get the data — which areas/authors correlate with production issues),
and the whole team (including the testers, who clean up) feels that pain. Frame
the standard not as "you must test because I say so" but as "we have a real
problem — these incidents are costing us, disrupting everyone, and eroding trust —
and testing is how we fix it." Get the team to <em>agree there's a problem worth
solving</em> before proposing the solution. Ideally, let the team <em>define the
standard together</em> (what level of testing, for what kinds of code) — a
standard the team chose sticks; one imposed is evaded (the failed decree proved
this).
<br><br>
<strong>3. Make testing easy (the paved road).</strong> If part of the reason is
that testing is painful here, <em>fix that</em> — invest in test infrastructure,
fast test runs, good examples/templates, helpers that make writing a test quick.
Make the tested path the low-friction path. This addresses a root cause and
removes the non-testers' most legitimate objection ("testing is a pain here"),
while helping everyone. This is often the highest-leverage move and the one decree
skips entirely.
<br><br>
<strong>4. Start with willing adopters and social proof.</strong> The four testers
are your allies — amplify what they do (share how their testing caught a bug before
prod; have them pair with or mentor the non-testers). Change spreads better from
peers demonstrating value than from the lead mandating (Phase 7). Let the standard
grow from a coalition, not a command.
<br><br>
<strong>5. Ratchet gradually, then automate the floor.</strong> Rather than a
flag-day "100% coverage now" (which would trigger resistance and gaming), ratchet:
start with "new code in high-risk areas gets tests," expand over time. Once there's
buy-in and a paved road, <em>automate the floor</em> — a CI check requiring tests
for new code (or coverage not decreasing), so the standard is enforced by an
impersonal tool, not by you nagging. The gate comes <em>after</em> buy-in and
tooling, not before (a gate imposed on resentful engineers without support gets
gamed — they write meaningless tests to pass the check).
<br><br>
<strong>6. Address the incentive if needed.</strong> If the non-testers are testing
less because the team/org <em>rewards</em> shipping speed and ignores the resulting
incidents, no standard survives that incentive — the lead may need to change what's
recognized (make reliability and testing visibly valued, count the incident cost
against the "fast" shipping, recognize the testers' quality work). Standards fail
when they fight the incentives; align them.
<br><br>
<strong>Why this beats the failed decree:</strong> the decree failed because it
provided no <em>reason</em> the non-testers accepted, no <em>ease</em> (testing
stayed painful), no <em>social proof</em>, and no <em>enforcement</em> — just words
in a meeting. This path provides all four: shared understanding of the problem
(buy-in), a paved road (easy), peer-driven adoption (social proof), and eventual
automation (enforcement) — plus it addresses root causes (painful testing,
misaligned incentives) rather than just demanding behavior. It's slower than a
decree but it actually <em>works</em>, because standards are adopted through
buy-in + ease + enforcement, not through authority. The meta-point: "decree
doesn't change behavior" is one of the most important lessons of the lead role —
you have far less unilateral power than you think, and driving change is a
persuasion-and-enablement problem (Phase 7), not an order-giving one.
<br><br>
Common mistakes: (1) re-issuing the decree more forcefully (it already failed;
volume won't fix it); (2) jumping to a CI gate first, before buy-in or a paved
road (gets gamed — meaningless tests to pass the check — and breeds resentment);
(3) assuming the non-testers are just lazy/wrong without diagnosing (they may have
a real point about painful testing or misaligned incentives); (4) making it a
you-vs-them enforcement battle rather than a team-owned shared standard.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Engineering practices at scale | Google Engineering Practices — <https://google.github.io/eng-practices/> |
| Paved roads / golden paths | Netflix, Spotify engineering blogs; platform-engineering literature |
| Driving standards through developer experience | *Team Topologies*, Skelton & Pais (platform teams) |
| Change management for standards | *Switch*, Chip & Dan Heath |
| Testing culture | *Software Engineering at Google*, Winters et al. (testing chapters) |

---

## Checkpoint

**Q1.** Distinguish paved roads, guardrails, and gates as ways to drive a
standard. Why is "make the right way the easy way" usually more effective than a
mandate?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Paved road</strong>: the standard way is the easy, supported, default path
— good tooling, templates, libraries, and docs mean doing it the standard way
"just works" and requires the least effort; deviating is allowed but you're on your
own (more work, less support). Drives adoption by making the right way the
<em>low-friction</em> way. <strong>Guardrail</strong>: catches you before genuine
danger — warns or blocks on the truly risky, but leaves freedom within the safe
zone; often automated (a linter warning, a CI check on the dangerous patterns).
Drives safety without over-constraining. <strong>Gate</strong>: a hard requirement
that blocks (CI fails, the PR can't merge until it's met) — reserved for things
that <em>must</em> hold (security requirements, tests for critical code); powerful
but resented if overused (every gate is friction and a message of distrust). Why
"make the right way the easy way" (paved road) usually beats a mandate: a mandate
fights human nature — you're telling people to do something that's <em>harder</em>
or less convenient than the alternative, relying on compliance and enforcement,
which produces resentment, evasion, malicious compliance (technically meeting the
letter while missing the point), and constant policing effort from the lead; the
moment enforcement lapses, people revert. A paved road <em>aligns</em> with human
nature — people naturally take the path of least resistance, so if the standard
path <em>is</em> the least-resistance path (it's easier to use the template that
has testing set up than to build from scratch, easier to use the shared library
that handles errors right than to write your own), people follow it because it
serves <em>them</em>, needing no enforcement and generating no resentment. The
standard becomes self-sustaining (adopted because it's genuinely easier) rather
than requiring perpetual policing. The deeper principle: you drive far more
consistent behavior by <em>changing the incentives and friction</em> (making the
desired behavior the easy/rewarding one) than by <em>demanding the behavior</em>
against the existing friction — which is why the highest-leverage standards work is
building tooling and paved roads (a platform-engineering investment) rather than
writing policy documents and enforcing them. Gates and mandates have their place
(the genuinely must-hold things, enforced impersonally by automation), but they're
the exception for the critical few, not the primary mechanism — and even those work
best <em>after</em> a paved road makes compliance easy, so the gate catches slips
rather than fighting an uphill battle against a standard that's harder than the
alternative.
</details>

**Q2.** Why is a technical standard imposed by decree usually resented and evaded,
while the same standard can succeed if approached differently? What's the
difference in the two paths?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A decreed standard is resented and evaded because it has none of the ingredients
that actually produce adoption — it's just an assertion of authority. Specifically,
the decree typically provides: no <em>reason</em> the affected people accept (they
don't understand or agree with the value, so they experience it as arbitrary
imposition — "why should I, my code is fine"); no <em>ease</em> (the standard is
often harder than what people do now, and the decree does nothing to reduce that
friction, so following it is pure cost to them); no <em>input</em> (it was handed
down, not chosen, so there's no ownership — people defend what they helped create
and resist what's imposed on them); and often no real <em>enforcement</em> (a
statement in a meeting that nobody checks, so ignoring it is costless — the failed
"everyone must write tests" decree). The result: people nod and continue as before
(the lab's exact scenario), or comply minimally and resentfully (malicious
compliance — meaningless tests to satisfy the letter), or evade when unwatched.
The successful path provides what the decree lacks: (1) <strong>buy-in / shared
understanding</strong> — the team genuinely understands and ideally agrees the
standard solves a real problem they feel (connecting it to shared pain like the
incidents), so they follow it because they see the value, not because they're
told; (2) <strong>input / ownership</strong> — the team helps define the standard
(what testing, for what code), so it's <em>theirs</em>, and people uphold what they
chose; (3) <strong>ease / paved road</strong> — the standard path is made low-
friction (tooling, examples, fixing whatever made it painful), so following it
costs little; (4) <strong>social proof</strong> — early adopters demonstrate the
value peer-to-peer, and change spreads through the team rather than being pushed
from above; (5) <strong>aligned incentives</strong> — what's rewarded matches the
standard (reliability is valued, not just shipping speed), so the standard doesn't
fight the incentives; (6) <strong>appropriate enforcement</strong> — automated,
impersonal, and applied <em>after</em> buy-in and ease exist (a CI check that
catches slips, not a mandate imposed on resentful people). The core difference:
the decree treats adoption as an <em>authority</em> problem ("I'm the lead, I said
so") and the successful path treats it as a <em>change-management</em> problem
(persuade, enable, involve, ease, reinforce — Phase 7). The lead's realization that
must underlie this: you have far less unilateral power to change behavior than the
title suggests — engineers do what they're persuaded and enabled to do, not what
they're ordered to do — so driving a standard is a campaign of buy-in and
enablement, not an act of decree, and leads who lean on authority get compliance-
theater while leads who build buy-in and paved roads get genuine, durable adoption.
Same standard, opposite outcomes, entirely because of <em>how</em> it's pursued.
</details>

---

## Homework

Pick a standard your team lacks but would benefit from (testing, error handling,
a code pattern, documentation, whatever's genuinely missing). Design the adoption
campaign — NOT the decree. Specify: how you'd diagnose why it's not already
happening; how you'd build buy-in (what shared pain connects to it); what paved
road (tooling, template, examples) would make the right way the easy way; how you'd
use early adopters and social proof; and what, if anything, you'd eventually
automate as enforcement — and in what order. Then reflect: how does this differ
from your instinct, and what does the difference reveal about how much power a
lead actually has to change behavior?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise's value is confronting the gap between the instinct (usually: "hold a
meeting, explain the standard, tell everyone to follow it" — a decree with extra
steps) and what actually works (a multi-step campaign of diagnosis, buy-in, ease,
social proof, and eventual automation). A strong response produces a real campaign:
a genuine diagnosis of why the standard isn't already happening (often revealing
the behavior has a rational cause — it's painful, unrewarded, or the value isn't
seen — not just negligence), a buy-in strategy that connects the standard to pain
the team actually feels (so they <em>want</em> it, not just tolerate it), a
concrete paved road that reduces the friction (the specific tooling/template/
examples that make the standard path easier than the current one — usually the
highest-leverage and most-overlooked piece), a plan to leverage willing adopters
for peer-driven spread, and a <em>later</em> automation step (a CI check as the
floor, imposed after buy-in and ease exist, not before). The ordering matters:
diagnose → buy-in → ease → social proof → automate — with enforcement last, because
a gate imposed before the others gets gamed. The reflection is the real payoff, and
it should produce a genuine recalibration: most leads' instinct is far closer to
decree than they realize (even "explain it well and then require it" is mostly
decree), and doing the exercise reveals how much <em>more</em> is required to
actually change behavior — and, underneath that, how much <em>less unilateral
power</em> a lead has than the title implies. This is one of the most important and
humbling realizations of the role: you cannot simply order engineers to change how
they work and expect it to happen (the failed decree proves it); behavior change
requires persuasion (they must understand and accept the value), enablement (the
new way must be made feasible/easy), involvement (they must have ownership), and
reinforcement (incentives and enforcement aligned) — a campaign, not a command. The
leads who internalize this stop trying to drive change by authority (which produces
compliance-theater and resentment) and start driving it by buy-in and paved roads
(which produces genuine adoption) — and they redirect their energy from writing and
enforcing policy to the far higher-leverage work of diagnosing root causes, making
the right thing easy (tooling investment), and building the coalition that carries
the change. If your instinct was <em>already</em> the campaign approach rather than
the decree, excellent — that means you've internalized the "influence not
authority" reality that Phase 7 develops fully; the ongoing discipline is
resisting the decree temptation when you're under pressure and a mandate feels
faster (it isn't — it fails, and then you've spent the authority and still don't
have the standard).
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 11 — Managing Technical Debt Strategically →](lesson-11-tech-debt){: .btn .btn-primary }
