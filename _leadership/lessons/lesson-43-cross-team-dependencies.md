---
title: "Lesson 43 — Other Engineering Teams and Dependencies"
nav_order: 4
parent: "Phase 8: Stakeholder Management"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 43: Other Engineering Teams and Dependencies

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **dependency** — work of yours that cannot proceed until another team delivers theirs.
> - **dependency contract** — the written agreement: *what* they'll deliver, *when*, with what *interface*.
> - **interface** — the exact technical boundary (API, schema, spec) two teams meet at.
> - **incentives** — what actually drives a team's choices: their goals, metrics, mandate — not kindness.
> - **goodwill** — friendly willingness to help; real but unreliable when priorities compete.
> - **burning bridges** — winning today in a way that destroys the relationship you'll need tomorrow.
> - **over someone's head** — going to a person's boss without trying them first.
> - **platform team** — a team whose product is tooling and infrastructure for other teams; you are its **customer**.
> - **mandate** — what a team is officially chartered to do.
> - **reciprocal** (rih-SIP-roh-kul) — flowing both ways over time; cross-team relationships are.

## Concept

Your work often depends on **other engineering teams** — a platform team, an API you need changed, a
service another team owns — who don't report to you and have their **own roadmap and priorities.** You
can't order them, and their kindness alone won't reliably get you what you need. The skills: making
**dependency contracts** explicit (what, when, interface — in writing), understanding their
**incentives** (not relying on goodwill), escalating **without burning bridges**, and being a **good
customer** to platform teams. Getting things from teams that don't work for you is a core leadership
challenge.

```
   DEPENDENCY REALITY
   ┌──────────────────────────────────────────────────────┐
   │ • they have their OWN roadmap & priorities (not yours) │
   │ • you can't ORDER them → influence, not authority      │
   │ • DEPENDENCY CONTRACT: what + when + interface, WRITTEN │
   │ • understand their INCENTIVES (why would THEY help?)   │
   │ • escalate WITHOUT burning bridges (you'll need them    │
   │   again)                                                │
   └──────────────────────────────────────────────────────┘
```

The reframe: **get things from other teams through explicit contracts and understanding their
incentives — not by assuming their goodwill will prioritize your need over their roadmap.** Another
team has their own priorities; your need competes with theirs. Relying on kindness ("they'll help
because we're all one company") is unreliable — they're rationally focused on their own goals. So you
make dependencies explicit (contracts), align your need with their incentives (why would helping you
serve <em>them</em>?), and escalate skillfully when needed — all while preserving the relationship for
next time.

---

## Going Deeper

### Dependency contracts — what, when, interface, in writing

The foundation of managing a cross-team dependency: an **explicit contract** — agree, <em>in writing</em>,
on **what** they'll deliver, **when**, and the **interface** (the exact API/contract/spec). Vague verbal
agreements ("yeah, we'll get you that API") fail — they're forgotten, deprioritized, or interpreted
differently, and you discover the gap too late. A written dependency contract (what exactly, by when,
with what interface) creates clarity and accountability: both sides know precisely what's committed,
you can plan around it, and there's a reference if it slips. Nail down dependencies explicitly and in
writing, early — the fuzzy dependency is the one that blows up your timeline.

### Their incentives, not their kindness

The crucial mindset shift: **understand and align with the other team's incentives, don't rely on their
kindness.** Another team helps you reliably when helping you serves <em>their</em> goals (their
priorities, their metrics, their mandate) — not out of pure goodwill (which is unreliable when your need
competes with their roadmap). So ask: <em>why would this team want to help me? what's in it for them?</em>
— and frame your need in terms of their incentives ("this helps you hit the adoption target you care
about," "this is exactly the platform use case you're mandated to support"). If there's genuine
alignment, surface it; if not, you may need to make it worth their while, find another path, or escalate
to someone who can reprioritize. Relying on kindness alone (expecting them to drop their priorities for
yours because you asked nicely) is naive; aligning with incentives is how you reliably get cooperation.

### Escalating without burning bridges

Sometimes you need to **escalate** — the other team can't/won't prioritize your need at the working
level, and it needs a decision from higher up (their manager, a shared manager). Escalate **without
burning bridges**: (1) **try the working level first** (escalating prematurely, over people's heads,
damages the relationship); (2) **give warning** (tell the other team you're going to escalate — don't
blindside them: "I don't think we can resolve the priority conflict at our level, so I'm going to raise
it with our managers — wanted you to know"); (3) **frame it neutrally** as a prioritization decision, not
a complaint about them (present both sides' needs and ask for a decision); and (4) **preserve the
relationship** (you'll need this team again — escalating a genuine conflict is fine, but doing it
adversarially or behind their back poisons future cooperation). Escalation is a legitimate tool for
genuine priority conflicts, done in a way that doesn't make an enemy.

### Platform teams — being a good customer

For **platform/infrastructure teams** (whose job is to serve other teams), you're a **customer** — and
being a **good customer** gets you better service: (1) come with clear, well-specified needs (not vague
demands); (2) understand their constraints and roadmap (work with them, not against); (3) be reasonable
(don't demand everything urgently); (4) give feedback constructively and appreciate their work; (5) help
them (adopt their stuff, give them the usage/feedback that helps them improve). A team that's a good
customer (clear, reasonable, appreciative, collaborative) gets prioritized and helped; one that's a
demanding, vague, unappreciative customer gets deprioritized. The platform team relationship is a
long-term one — invest in being the customer they want to help.

### The vendor-team relationship — reciprocity over time

Cross-team relationships are **long-term and reciprocal** — you'll depend on each other repeatedly over
time. So treat them as ongoing relationships (Lesson 35's build-before-you-need-them): help other teams
when you can (building goodwill and reciprocity), be reliable and reasonable (so they want to work with
you), and don't burn bridges over any single conflict (you'll need them again). The team that's a good
partner over time — helpful, reliable, reasonable — gets cooperation when they need it; the team that's
purely extractive or adversarial finds doors closed. Invest in the cross-team relationships as
long-term reciprocal partnerships, not one-off transactions.

{: .note }
> **Get things from other teams via explicit contracts and their incentives — not their kindness</br>**
> Your work depends on teams that don't report to you and have their own roadmaps — you can't order them,
> and their goodwill alone won't reliably prioritize your need over theirs. Make dependencies explicit:
> a written <em>dependency contract</em> (what + when + interface) creates clarity and accountability
> (the fuzzy dependency is the one that blows up). Align with their <em>incentives</em> (why would helping
> you serve <em>them</em>?) rather than relying on kindness. <em>Escalate without burning bridges</em>
> (working level first, warn them, frame neutrally, preserve the relationship — you'll need them again). Be
> a <em>good customer</em> to platform teams (clear, reasonable, appreciative — which gets you prioritized).
> And treat cross-team relationships as long-term reciprocal partnerships (help others, build goodwill,
> don't burn bridges). Getting things from teams you don't control is influence, not authority.

---

## Lab — Scenario

**The situation:** Your launch (an important, time-sensitive feature) needs an **API change from a
platform team** — a modification to their service that only they can make. You've asked, but their
roadmap is full, and their manager has said **"next quarter"** — about three months out. But you need it
in **six weeks** to hit your launch. You have no authority over the platform team; their manager
outranks or equals yours. You need to get this API change in six weeks without having authority and
without burning the bridge (you depend on this team regularly).

**Write how you'd approach getting the API change in time.** Then note the principles and mistakes to
avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong approach understands their incentives and constraints, looks for creative options, makes the
business case, and escalates well if needed — all while preserving the relationship. Example:
<br><br>
<strong>First — understand their situation and the real constraint:</strong> "Talk to the platform team
(the engineers and manager) to understand: is 'next quarter' about capacity (no one to do it), priority
(other things rank higher), or complexity (it's a big change)? And clarify exactly what I need (maybe a
smaller version suffices). Understanding the real blocker tells me which lever to pull." <em>(Understand
their incentives/constraints before pushing.)</em>
<br><br>
<strong>Look for creative options that lower the cost to them:</strong> "Depending on the blocker: (a)
<strong>Can we do the work?</strong> If it's a capacity problem, could my team implement the change (with
their review/guidance) rather than them doing it — a common way to unblock a busy platform team? (b)
<strong>A smaller/interim version?</strong> Maybe a minimal change gets me what I need now, with the full
version next quarter. (c) <strong>Reprioritize a piece?</strong> Is there a small slice they could fit in?
Lowering the cost/effort for them makes 'yes' easier." <em>(Reduce the burden on them — often the key to
getting a busy team to help, especially offering to do the work.)</em>
<br><br>
<strong>Make the business case in their terms and to the right level:</strong> "Frame why this matters —
not just 'I need it' but the business impact (this launch is important to [company goal/revenue], and the
API change is the blocker). If it's genuinely important, make that case to the people who own the priority
trade-off (my manager + their manager, or product leadership) — this is a prioritization decision above
the working level. Align it with their incentives if possible (does this API change also serve their
platform goals / other teams?)." <em>(Business case + align with their incentives + surface to the right
decision level.)</em>
<br><br>
<strong>Escalate well if needed (without burning bridges):</strong> "If we can't resolve it at the working
level, escalate — but well: (1) tell the platform team first ('I think this needs to go up to our managers
to sort out the priority — wanted you to know, not going around you'); (2) frame it neutrally as a
prioritization conflict (my launch's importance vs. their full roadmap — a real trade-off for leadership),
not a complaint about them; (3) bring options (my team does the work, an interim version) so it's
constructive; (4) accept the outcome gracefully (if leadership says their roadmap wins, that's the call —
don't poison the relationship)." <em>(Escalate as a service, with warning, neutrally, preserving the
relationship.)</em>
<br><br>
<strong>Preserve the relationship throughout:</strong> "Be a good customer — reasonable, appreciative,
collaborative — because I'll depend on this team again. Even if I have to escalate, do it in a way that
keeps us good partners."
<br><br>
<strong>Principles:</strong> (1) <strong>Understand their constraint/incentives first</strong> — capacity
vs. priority vs. complexity determines the approach. (2) <strong>Lower the cost to them</strong> — offer
to do the work, accept a smaller/interim version — making 'yes' easy (often the key with a busy team). (3)
<strong>Make the business case, aligned with their incentives</strong> — why it matters, in terms that
move them and the deciders. (4) <strong>Escalate to the right level</strong> — a genuine priority conflict
between teams is a leadership prioritization decision, surfaced with framing and options. (5) <strong>Escalate
without burning bridges</strong> — warn them, frame neutrally, bring options, accept the outcome. (6)
<strong>Preserve the long-term relationship</strong> — you'll need them again; be a good customer/partner.
<strong>Mistakes to avoid:</strong> (1) <strong>relying on kindness</strong> — assuming they'll just help
because you asked (their roadmap is full — you need incentive alignment or a lower cost or a priority
decision); (2) <strong>demanding / going over their heads immediately</strong> — escalating without trying
the working level first, or blindsiding them, poisons the relationship; (3) <strong>not offering to lower
their cost</strong> — not exploring 'my team does the work' or an interim version (often the unlock); (4)
<strong>making it a complaint about them</strong> when escalating (vs. a neutral priority trade-off); (5)
<strong>no business case</strong> — just 'I need it' without why it matters (doesn't move deciders); (6)
<strong>burning the bridge</strong> — winning this one adversarially and losing the long-term relationship
(you depend on them regularly); (7) <strong>not accepting a legitimate 'no'</strong> — if leadership
genuinely decides their roadmap wins, fighting it or sabotaging poisons everything (accept it, adjust your
plan — Lesson 44's renegotiation).
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Cross-team dependencies | *StaffEng* (working across teams); dependency management |
| Incentives & influence | Lesson 35 (sources of influence); *Influence Without Authority* |
| Escalation done well | Lesson 37 (escalation as a service); Lesson 44 (negotiation) |
| Platform-team / good-customer relationship | Team Topologies (platform as a product) |

---

## Checkpoint

**Q1.** Why is a written "dependency contract" (what/when/interface) important, and why should you rely on
the other team's "incentives, not their kindness"?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A written <strong>dependency contract</strong> (what + when + interface, in writing) is important because
<strong>it creates the clarity and accountability that vague verbal agreements lack — so both sides know
precisely what's committed, you can plan around it, and there's a reference if it slips</strong>. When a
cross-team dependency is handled with a vague verbal agreement ("yeah, we'll get you that API sometime"),
it fails in predictable ways: (1) it's <strong>forgotten or deprioritized</strong> — a casual verbal
commitment isn't tracked, so it slips down the other team's priorities without anyone noticing; (2) it's
<strong>interpreted differently</strong> — "that API" might mean different things to each side (different
scope, interface, timing), so you build assuming one thing while they deliver another; (3) you <strong>discover
the gap too late</strong> — the fuzzy dependency's problems surface near your deadline (when they haven't
delivered, or delivered the wrong thing), blowing up your timeline when it's too late to recover. An
explicit written contract fixes these: specifying <strong>what</strong> exactly they'll deliver (the precise
scope), <strong>when</strong> (the committed date), and the <strong>interface</strong> (the exact API/spec/
contract) creates (a) <strong>clarity</strong> — both sides know precisely what's committed, eliminating the
different-interpretations problem; (b) <strong>accountability</strong> — a written commitment is tracked and
referenceable, so it's harder to silently deprioritize, and if it slips there's a clear record; and (c)
<strong>plannability</strong> — you can plan your work around a specific committed deliverable and date,
rather than a fuzzy hope. So nailing down dependencies explicitly and in writing, early, is how you avoid
the fuzzy-dependency-blows-up-the-timeline failure — the written contract is the tool for managing what you
depend on but don't control. Why rely on the other team's <strong>incentives, not their kindness</strong>:
<strong>because another team has their own roadmap and priorities, so they reliably help when helping you
serves <em>their</em> goals — whereas relying on pure goodwill is unreliable when your need competes with
their priorities</strong>. The key reality: another engineering team isn't focused on your goals; they're
(rationally) focused on their own roadmap, metrics, and mandate. Your need competes with their priorities
for their limited capacity. If you rely on <strong>kindness</strong> — assuming they'll help because "we're
all one company" or because you asked nicely — you're expecting them to deprioritize their own goals for
yours out of goodwill, which is unreliable: when push comes to shove, they'll (reasonably) prioritize their
own committed work over a favor for another team, so goodwill-based requests get deprioritized when they
conflict with the other team's priorities (which is often). Relying on kindness alone is therefore naive —
it works for small, low-cost favors but fails when your need genuinely competes with their roadmap. Aligning
with <strong>incentives</strong> is more reliable because it makes helping you serve <em>their</em> goals:
if you can frame your need so that helping you also advances their priorities/metrics/mandate ("this helps
you hit the adoption target you care about," "this is exactly the platform use case you're mandated to
support"), then helping you is in their interest, so they'll do it reliably (not as a favor competing with
their goals, but as something that advances their goals). So you ask "why would this team want to help me —
what's in it for them?" and align your request with their incentives; if there's genuine alignment, surface
it (making the case in their terms); if not, you may need to make it worth their while (offer to do the work,
reduce their cost), find another path, or escalate to someone who can reprioritize (a priority decision
above the working level). The mindset shift is from "they should help because it's nice / we're all on the
same side" (kindness — unreliable) to "they'll help if it serves their incentives, so align my need with
those (or make it worth their while, or get it reprioritized)" (incentives — reliable). This isn't cynical;
it's realistic — teams rationally pursue their own priorities, so getting reliable cooperation means working
with their incentives, not against them or in spite of them. Both points reflect managing cross-team
dependencies realistically: make the dependency explicit and accountable (the written contract, so it's
clear and tracked, not fuzzy and forgotten), and get cooperation through incentive alignment (so helping
you serves them, reliably) rather than goodwill (unreliable when needs compete) — which is how you get what
you need from teams you don't control.
</details>

**Q2.** Why must you "escalate without burning bridges," and what makes a team a "good customer" to a
platform team?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You must "escalate without burning bridges" because <strong>escalation is sometimes necessary for genuine
priority conflicts, but cross-team relationships are long-term and reciprocal — you'll depend on this team
again — so escalating in a way that makes an enemy poisons future cooperation, which costs you far more than
any single win</strong>. Escalation (taking a priority conflict to higher management for a decision) is a
legitimate tool when the working level can't resolve it — but <em>how</em> you escalate matters enormously,
because the other team isn't a one-off: you'll have repeated dependencies on them over time, so the
relationship is an ongoing asset you need to preserve. Escalating <em>badly</em> — going over their heads
without warning (blindsiding them), framing it as a complaint about them (making them look bad), or doing it
adversarially — damages the relationship: the other team feels attacked, undermined, or betrayed, and
becomes reluctant or hostile to help you in the future (they remember, and cooperation dries up). So a
poorly-handled escalation might win the immediate battle but lose the long-term war (the ongoing
cooperation you need). Escalating <em>well</em> — without burning bridges — preserves the relationship while
still resolving the conflict: (1) <strong>try the working level first</strong> (escalating prematurely, over
people's heads, before genuinely trying to resolve it directly, damages the relationship and is seen as
going around them); (2) <strong>give warning</strong> — tell the other team you're going to escalate ("I
don't think we can resolve this priority conflict at our level, so I'm going to raise it with our managers —
wanted you to know"), so you're not blindsiding them (which feels like a betrayal); (3) <strong>frame it
neutrally</strong> as a prioritization decision (present both teams' legitimate needs and ask leadership to
decide), not as a complaint about the other team (so they don't look bad and it's not personal); and (4)
<strong>accept the outcome gracefully</strong> (if leadership decides against you, don't poison the
relationship over it). Escalation done this way — working level first, with warning, framed neutrally,
accepting the outcome — resolves genuine conflicts while keeping the relationship intact for the next time
you need each other. The principle: <strong>escalate the genuine conflict, but do it in a way that treats
the other team as a long-term partner you'll need again, not an enemy to defeat</strong>. What makes a team
a <strong>good customer</strong> to a platform team: <strong>being clear, reasonable, appreciative, and
collaborative — which gets you prioritized and helped, where being demanding, vague, and unappreciative gets
you deprioritized</strong>. Platform/infrastructure teams (whose job is to serve other teams) have many
"customers" and limited capacity, so they (like anyone) prefer to help the customers who are good to work
with — and being a good customer gets you better service. A good customer: (1) <strong>comes with clear,
well-specified needs</strong> — clear requirements and use cases (not vague demands like "make it faster"),
so the platform team can actually help efficiently; (2) <strong>understands their constraints and
roadmap</strong> — works <em>with</em> their reality (their priorities, their capacity) rather than against
it, and doesn't expect them to drop everything; (3) <strong>is reasonable</strong> — doesn't demand
everything urgently, doesn't treat every request as a P0, respects their time and priorities; (4) <strong>gives
feedback constructively and appreciates their work</strong> — treats them as valued partners (thanks them,
acknowledges their work) rather than a service to be complained at; and (5) <strong>helps them</strong> —
adopts their platform, gives them the usage and feedback that helps them improve, is a partner in their
success (a platform team's success <em>is</em> teams using and benefiting from their stuff). Being this kind
of customer (clear, reasonable, appreciative, collaborative) makes the platform team <em>want</em> to help
you and prioritize your needs — you become a customer they enjoy serving. Being a bad customer (vague,
demanding, entitled, unappreciative, treating them as a punching bag) makes them deprioritize you (they'll
help the pleasant, clear customers first). Since the platform-team relationship is long-term (you'll depend
on them ongoing), being a good customer is an investment in reliably good service over time. Both points
reflect that cross-team relationships are <strong>long-term reciprocal partnerships</strong>, not one-off
transactions: you'll depend on these teams repeatedly, so you preserve the relationship even through
conflicts (escalate without burning bridges) and invest in being someone they want to help (a good
customer) — because the team that's a good long-term partner (reasonable, appreciative, non-bridge-burning)
gets cooperation when they need it, while the extractive or adversarial team finds doors closed. Managing
dependencies well is as much about the long-term relationship as any single request.
</details>

---

## Homework

Reflect on how you manage cross-team dependencies. (1) Are your dependencies explicit — written contracts
(what/when/interface) — or vague verbal agreements that could blow up? Nail down one fuzzy dependency in
writing. (2) Are you relying on other teams' kindness, or aligning with their incentives (why would helping
you serve them)? For a dependency, identify their incentives and how to align. (3) When you've escalated (or
need to), are you doing it without burning bridges (working level first, warning, neutral framing)? (4) With
platform teams, are you a good customer (clear, reasonable, appreciative)? Reflect: how well do you get
things from teams you don't control, and what's one dependency you should make explicit or one relationship
to invest in?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds cross-team-dependency skill — getting things from teams that don't report to you and
have their own roadmaps. A strong response checks whether dependencies are explicit (written contracts) or
fuzzy, whether they align with other teams' incentives (vs. relying on kindness), whether escalation
preserves relationships, and whether they're a good customer to platform teams. The realizations: (1)
<strong>make dependencies explicit — written contracts</strong> (what/when/interface) — vague verbal
agreements get forgotten, deprioritized, or misinterpreted and blow up the timeline; a written contract
creates clarity, accountability, and plannability; (2) <strong>rely on incentives, not kindness</strong> —
other teams have their own priorities and reliably help when it serves their goals, so align your need with
their incentives (or lower their cost, or get it reprioritized) rather than expecting goodwill to
deprioritize their roadmap for you; (3) <strong>escalate without burning bridges</strong> — escalation is
legitimate for genuine priority conflicts, but do it well (working level first, warn them, frame neutrally,
accept the outcome) because you'll need the team again; (4) <strong>be a good customer to platform teams</strong>
— clear, reasonable, appreciative, collaborative — which gets you prioritized, where being demanding and
vague gets you deprioritized; (5) <strong>treat cross-team relationships as long-term reciprocal
partnerships</strong> — help others, build goodwill, don't burn bridges. On reflection, people commonly find
their dependencies are often vague/verbal (a timeline risk), that they lean on goodwill rather than incentive
alignment, and that they could be better customers to platform teams. The highest-value change is usually:
make dependencies explicit in writing (what/when/interface), and align requests with the other team's
incentives (or lower their cost) rather than relying on kindness. The meta-point: your work depends on teams
you don't control and who have their own roadmaps, so getting what you need is influence, not authority —
through explicit dependency contracts (clarity and accountability), incentive alignment (reliable
cooperation), skillful escalation (without burning bridges), and being a good long-term partner (good
customer, reciprocity). This applies the influence skills (Lesson 35) to the specific challenge of
cross-team work, which is much of how larger efforts actually get done. The last Phase 8 lesson brings
together the stakeholder skills into negotiation and expectation management — negotiating scope, time, and
staffing, which is now a core part of the job.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 44 — Negotiation and Expectation Management →](lesson-44-negotiation-expectations){: .btn .btn-primary }
