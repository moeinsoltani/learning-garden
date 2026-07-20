---
title: "Lesson 34 — Succession and the Bus Factor"
nav_order: 5
parent: "Phase 6: Delegation & Growing the Team"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 34: Succession and the Bus Factor

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **bus factor** — how many people could be lost ("hit by a bus") before critical knowledge is gone (Lesson 03).
> - **SPOF (single point of failure)** — the one person or component whose loss breaks everything.
> - **de-risk** — to reduce a risk deliberately before it bites.
> - **redundancy** — having more than one person able to do each critical thing; here framed as *freedom*, not expendability.
> - **knowledge mapping** — listing each critical system and asking "who understands this?"
> - **runbook** — the written step-by-step guide for operating a system ("what to do when X breaks").
> - **tribal knowledge** — critical information living only in heads (Lesson 03).
> - **brown-bag** — an informal lunchtime teaching session (named for the lunch bag).
> - **hoards context** — keeps knowledge to themselves, deliberately or not, because being irreplaceable feels safe.
> - **expendable / disposable** — easily discarded; the threatening misreading of "not irreplaceable."
> - **on call** — carrying the pager: responsible for responding to incidents.

## Concept

The **bus factor** is the number of people who'd have to be "hit by a bus" (leave, go on vacation, be
out sick) before a critical piece of knowledge or capability is lost. A bus factor of 1 — only one
person understands the billing system, the deploy process, the legacy service — is a serious risk:
you're one departure, illness, or vacation away from being stuck. The skill is **de-risking single
points of failure** — including *yourself* — by spreading knowledge, without making anyone feel
replaceable or threatened.

```
   BUS FACTOR = how many people can be lost before you're stuck
   ┌──────────────────────────────────────────────────────┐
   │ bus factor 1: only ONE person knows X                  │
   │   → fragile: their vacation/illness/exit = crisis      │
   │ DE-RISK: docs · pairing · rotation · knowledge-sharing  │
   │   → spread knowledge so no one (incl. YOU) is a SPOF    │
   │ FRAME as FREEDOM (vacations!), not threat/replaceability │
   └──────────────────────────────────────────────────────┘
```

The reframe: **making people non-irreplaceable is protecting the team (and freeing the person), not
diminishing anyone — frame redundancy as freedom, not threat.** The fear is that "you're not
irreplaceable" sounds like "you're expendable," and that the hero who hoards context resists sharing
it (their irreplaceability feels like security). But redundancy is what lets people take vacations,
get promoted, and not be on call forever — it's freedom, not a threat — and de-risking single points
of failure (including your own) is a core responsibility of a lead.

---

## How It Works

### Knowledge mapping — find the bus factor 1s

Start by **mapping the knowledge**: for each critical system, process, and area, ask *who understands
it?* — and identify the **bus factor 1s** (things only one person knows: the billing system, the
deploy magic, the legacy service, the customer-specific hacks). This map surfaces the single points
of failure — the risks you need to de-risk. Include the non-obvious ones (tribal knowledge, "ask
Dave," undocumented processes). You can't de-risk what you haven't identified, so mapping the
knowledge and its concentration is the first step.

### De-risking tools: docs, pairing, rotation

Three main tools to spread concentrated knowledge: (1) **Documentation** — write down the critical
knowledge (how the system works, the runbook, the context) so it's not only in someone's head. (2)
**Pairing / knowledge-sharing** — have the sole expert pair with or teach others (brown-bags,
pairing on the system), transferring the knowledge through people. (3) **Rotation** — rotate people
through the critical areas (don't let one person own billing forever), so more people gain
experience. Combined, these raise the bus factor (more people know each thing) — docs capture it,
pairing/teaching transfers it, rotation builds broad experience. It's ongoing work, not a one-time fix.

### The hero who hoards context

A specific challenge: the **hero who hoards context** — the person who's the sole expert on something
and (consciously or not) keeps it that way, because being irreplaceable feels like job security and
status. They may resist documenting or sharing ("it's faster if I just do it," "it's too complex to
explain"). This is a real risk (the team stays fragile, dependent on them) and must be addressed —
but carefully: (1) **reframe it for them** — being the sole expert is a *burden* (they can never fully
switch off, take a real vacation, or move to new work — they're chained to it), so sharing is
liberating; (2) **make sharing part of their role and valued** — recognize knowledge-sharing as
important work (not a threat to them), and expect it; (3) **don't attack their value** — frame it as
de-risking the team and freeing them, not as diminishing them. The hoarding often comes from insecurity;
addressing that (their value isn't their hoarded knowledge) helps.

### Your own bus factor as lead

Critically: **you** are often a bus factor 1 too — the lead who's the only one who knows the
stakeholders, the context, the decisions, how to run things. De-risking your *own* single points of
failure is part of the job (and connects to growing successors, Lesson 33): document your context,
grow people who can cover for you (acting leads), spread your knowledge. A lead who's irreplaceable is
a risk (and can't take vacation or get promoted — you can't move up if no one can replace you). Model
the de-risking by starting with yourself.

### Frame redundancy as freedom, not threat

The key to doing this without alienating people: **frame it as freedom, not replaceability.** The
message isn't "we need to be able to replace you" (threatening — sounds like "you're expendable");
it's "we want you to be able to take a real vacation, switch off on weekends, move to exciting new
work, and get promoted — none of which is possible if you're the only one who can do X." Redundancy
*serves* the person (freedom to be off, to grow, to advance) and the team (resilience) — it's not
about anyone being disposable. Framed as freedom (vacations! promotions! not being on call forever!),
people embrace de-risking; framed as replaceability, they resist it.

{: .note }
> **De-risk single points of failure (including yourself) — and frame redundancy as freedom, not threat</br>**
> The bus factor is how many people can be lost before critical knowledge is gone; a bus factor of 1
> (only one person knows the billing system, the deploy process — or the lead knows all the context) is
> a serious fragility. De-risk it: map the knowledge (find the bus-factor-1s), then spread it via docs,
> pairing/teaching, and rotation. Address the hero who hoards context carefully (reframe sole-expertise
> as a burden, make sharing valued, don't attack their value). De-risk your <em>own</em> single points of
> failure too (grow successors, document your context). And crucially, frame redundancy as <em>freedom</em>
> — being non-irreplaceable is what lets people take real vacations, switch off, move to new work, and get
> promoted — not as "you're replaceable/expendable" (which people resist). De-risking single points of
> failure serves both the team (resilience) and the people (freedom), and it's a core lead responsibility.

---

## Lab — Scenario

**The situation:** One engineer on your team, Dave, is the *sole* person who understands the billing
system — a critical, complex, somewhat legacy piece that everything depends on. And Dave... likes it
that way. Being the billing expert gives him status and a sense of job security; he's subtly resistant
to documenting it or teaching others ("it's really complex, faster if I just handle it"). This is a
serious bus factor 1 (if Dave's out sick, on vacation, or leaves, you're in trouble), but Dave's
resistance and the sensitivity mean you can't just order him to "document everything." You need to
de-risk billing within a quarter without alienating him.

**Write your plan to de-risk the billing knowledge within a quarter without alienating Dave.** Then
note the principles and mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong plan reframes de-risking as serving Dave (freedom, not threat), makes knowledge-sharing valued
work, and uses concrete tools — carefully, given the sensitivity.
<br><br>
<strong>First — the conversation, reframing it as freedom and valuing him:</strong> "Dave, I want to
talk about the billing system. First — you've done an amazing job owning it; it's genuinely critical and
you're the expert, and I value that a lot. <em>(Affirm his value — don't attack it.)</em> Here's what's
on my mind: right now you're the <em>only</em> person who deeply understands it, and honestly, I think
that's not fair to <em>you</em>. It means you can never fully switch off — you're the one who gets
pinged on vacation, you can't easily move to new exciting work because you're chained to billing, and it
puts a lot of pressure on you. I'd like to spread the billing knowledge — not to diminish your role, but
so <em>you</em> can take real vacations, work on new things, and not be the permanent on-call for
billing. And it de-risks the team. Would you be up for helping lead that?" <em>(Reframes de-risking as
freedom FOR Dave (vacations, new work, less pressure) and team resilience — not "we need to replace
you"; makes him a leader of the effort, not a target of it.)</em>
<br><br>
<strong>Make knowledge-sharing valued, expert-led work:</strong> "I'd like you to <em>lead</em> the
knowledge-sharing — you're the expert, so you're the one to teach it. Let's make this a recognized part
of your work this quarter (not extra, on-top): (1) run a few brown-bag sessions walking the team through
how billing works; (2) pair with [one or two people] on real billing work so they build hands-on
experience; (3) as we go, capture the key things in docs/runbooks — you explaining, maybe someone else
writing it up so it's not all on you." <em>(Positions Dave as the expert LEADING the transfer (status
preserved, even enhanced), makes it valued recognized work, and uses the three tools — teaching, pairing,
docs — with the writing burden shared.)</em>
<br><br>
<strong>Concrete quarter plan:</strong> "Month 1: brown-bags + start pairing [name] on billing tasks +
begin the runbook. Month 2: [name] takes on real billing work with Dave supporting (rotation begins);
continue docs. Month 3: a second person starts; [name] handles some billing independently; docs cover
the critical paths. Goal by end of quarter: at least two people beyond Dave can handle common billing
work, the critical knowledge is documented, and Dave can take a vacation without billing breaking."
<em>(Concrete, time-bound, using pairing + rotation + docs to raise the bus factor to 3.)</em>
<br><br>
<strong>Recognize it:</strong> "And I'll make sure your work teaching and de-risking this is recognized —
it's valuable leadership, and it'll reflect well on you (it's the kind of thing that matters for growth)."
<em>(Recognition — makes sharing rewarded, not threatening; even ties it to Dave's own advancement.)</em>
<br><br>
<strong>Principles:</strong> (1) <strong>Frame as freedom for Dave, not threat</strong> — the whole
approach centers on how de-risking <em>serves</em> Dave (vacations, new work, less pressure), not "we
need to replace you." (2) <strong>Affirm his value, don't attack it</strong> — he's valued and made the
leader of the transfer, so it's not diminishing. (3) <strong>Make sharing valued, expert-led work</strong>
— Dave leads the teaching (status preserved), it's recognized (rewarded, not threatening). (4) <strong>Use
the three tools</strong> — teaching (brown-bags), pairing, rotation, plus docs (writing shared). (5)
<strong>Concrete, time-bound plan</strong> — a real quarter plan raising the bus factor to 3+. (6)
<strong>Address the insecurity</strong> — his value isn't his hoarded knowledge; recognizing his
teaching/leadership gives him status through sharing, not hoarding. <strong>Mistakes to avoid:</strong>
(1) <strong>ordering him to "document everything"</strong> — triggers resistance, feels like a threat to
his status/security; (2) <strong>framing it as replaceability</strong> ("we need to be able to replace
you") — threatening, makes him resist harder; (3) <strong>attacking his hoarding directly</strong> —
defensive; instead reframe and address the underlying insecurity; (4) <strong>not making it valued
work</strong> — if sharing feels like a threat with no upside for him, he won't engage; make it
recognized and tie it to his growth; (5) <strong>trying to bypass Dave</strong> (having others reverse-
engineer billing without him) — slow, adversarial, wastes his expertise; (6) <strong>not being concrete/
time-bound</strong> — "spread the knowledge sometime" won't happen; a real plan with milestones does; (7)
<strong>ignoring the human sensitivity</strong> — this needs care (his identity/security is tied to being
the expert), so the reframe-as-freedom and valuing-him are essential, not optional.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Bus factor / knowledge risk | Bus factor (Wikipedia) — <https://en.wikipedia.org/wiki/Bus_factor> |
| Knowledge sharing & docs | Lesson 33 (successors); *The Manager's Path* (team resilience) |
| The hero / context hoarding | *StaffEng* on avoiding heroism; blameless knowledge-sharing culture |
| Framing & the hard conversation | English for Work track, Lesson 44 (feedback); Lesson 47 (framing) |

---

## Checkpoint

**Q1.** What is the "bus factor," why is a bus factor of 1 a serious risk (including for the lead), and
what are the main tools to de-risk it?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The <strong>bus factor</strong> is <strong>the number of people who would have to be lost (hit by a bus,
metaphorically — leave, go on vacation, be out sick) before a critical piece of knowledge or capability
is lost to the team</strong>. It measures how concentrated critical knowledge is: a high bus factor
(many people know something) is resilient; a low one (few people) is fragile. A <strong>bus factor of
1</strong> — only one person understands a critical system, process, or area (the billing system, the
deploy magic, the legacy service, the stakeholder relationships) — is a serious risk because <strong>the
team is one departure, illness, or even vacation away from being stuck or in crisis</strong>. If that
single person leaves, gets sick, or takes a vacation, the critical knowledge goes with them, and the team
can't maintain, fix, or operate that thing — a single point of failure that can cause an outage, block
work, or create a crisis at the worst time (things often break exactly when the expert is unavailable).
It's fragile: the team's ability to function depends on one person's continuous availability, which is a
risk you can't control (people do leave, get sick, take vacations). This applies to the lead too — often
<strong>the lead is a bus factor 1</strong>: the only one who knows the stakeholders, the context, the
history, the decisions, how to run things. A lead who's irreplaceable is a risk to the team (if they're
out, the team is stranded) and a problem for themselves (they can't take a real vacation, and they can't
get promoted — you generally can't move up if no one can replace you). So de-risking single points of
failure includes the lead's own — a core part of the job, connected to growing successors (Lesson 33).
The main tools to de-risk (raise the bus factor): (1) <strong>Documentation</strong> — write down the
critical knowledge (how the system works, runbooks, context, the "why" behind decisions) so it exists
outside one person's head and others can access it. (2) <strong>Pairing / knowledge-sharing</strong> —
have the sole expert pair with, teach, or run brown-bags for others, transferring the knowledge through
direct people-to-people learning (which conveys the tacit knowledge docs miss). (3) <strong>Rotation</strong>
— rotate people through the critical areas (don't let one person own billing forever), so multiple people
gain hands-on experience over time. Combined, these raise the bus factor: docs capture the knowledge,
pairing/teaching transfers it, and rotation builds broad experience — so more than one person can handle
each critical thing. It's ongoing work (not a one-time fix), and it applies to the lead's own knowledge
too (document your context, grow people who can cover for you). The goal is that no critical
knowledge or capability depends on any single person's continuous availability — so the team is resilient
to the normal, inevitable events (vacations, illness, departures) that a bus-factor-1 turns into crises.
Mapping the knowledge first (identifying the bus-factor-1s — who's the only one who knows each critical
thing, including the non-obvious tribal knowledge) is the prerequisite, since you can't de-risk what you
haven't identified.
</details>

**Q2.** Why is framing redundancy as "freedom, not threat" important, and how do you handle the hero who
hoards context?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Framing redundancy as "freedom, not threat" is important because <strong>de-risking can easily be
misheard as "you're replaceable/expendable," which threatens people and makes them resist — whereas
framing it as freedom (what redundancy lets them do) makes them embrace it, and it's actually true:
redundancy serves the person</strong>. The fear is that "we need to be able to do without you" or "you're
not irreplaceable" sounds like "you're disposable, we're reducing your value" — which is threatening
(people's sense of security and worth feels attacked), so they resist de-risking (why help make yourself
replaceable?). But this framing is both counterproductive and wrong. The reframe: <strong>redundancy is
freedom, not replaceability</strong> — being the sole person who can do something isn't security, it's a
<em>trap</em>: you can never fully switch off (you get pinged on vacation, you're the permanent on-call),
you can't easily move to new/exciting work (you're chained to the thing only you can do), and you carry
constant pressure (everything depends on you). Redundancy <em>frees</em> the person: it lets them take a
real vacation (someone else can cover), switch off on weekends, move to new work (they're not
irreplaceable on the old thing), and get promoted (you can't advance if no one can replace you). So the
honest message is "I want you to be able to take real vacations, work on new things, and grow — none of
which is possible if you're the only one who can do X" — which is true and serves the person, versus "we
need to be able to replace you" (threatening and makes them resist). Framed as freedom (vacations!
promotions! not being chained to one thing!), people embrace de-risking because it genuinely benefits
them; framed as replaceability, they resist because it feels like a threat. The framing matters because
de-risking requires people's cooperation (especially the experts), and whether they cooperate or resist
depends heavily on whether they experience it as serving them or threatening them. How to handle the
<strong>hero who hoards context</strong> (the sole expert who keeps it that way because irreplaceability
feels like security/status, resisting documenting or sharing): (1) <strong>reframe sole-expertise as a
burden, not a benefit</strong> — help them see that being the only expert is a <em>trap</em> (can't
switch off, can't take real vacations, can't move to new work, constant pressure), so sharing is
<em>liberating</em> for them (the freedom framing applied to their specific situation); (2) <strong>make
knowledge-sharing valued, expert-led work</strong> — position them as the expert <em>leading</em> the
transfer (teaching, running brown-bags), which preserves and even enhances their status (they're the
respected teacher, not a target), and recognize this work as valuable (Lesson 20), so sharing is rewarded
rather than threatening; (3) <strong>don't attack their value or the hoarding directly</strong> — the
hoarding usually comes from insecurity (their value feels tied to their irreplaceable knowledge), so
attacking it (ordering them to document, criticizing the hoarding) triggers defensiveness; instead affirm
their value and address the underlying insecurity — show that their value is recognized (and even grows)
through their expertise and teaching, not their hoarding, so they don't need to hoard to feel secure; (4)
<strong>tie it to their growth</strong> — knowledge-sharing and de-risking are valuable leadership work
that reflects well on them and aids their advancement, so it serves their career (not threatens it). The
key is that hoarding is often driven by the (false) belief that irreplaceability = security/value, so you
counter it by (a) showing sole-expertise is actually a burden (freedom framing), (b) giving them status
and value through sharing/teaching (so their worth isn't tied to hoarding), and (c) doing it with care
(affirming, not attacking) given the sensitivity. Both points reflect the human dimension of de-risking:
it requires people's cooperation, which depends on them experiencing it as serving them (freedom, valued
work, preserved status) rather than threatening them (replaceability, diminished value) — so the framing
and the careful handling of the hoarding-hero aren't soft niceties but the essential mechanics of
actually getting de-risking done without alienating the very people whose knowledge you need to spread.
</details>

---

## Homework

Assess and reduce your team's (and your own) bus factor. (1) Map the knowledge: for your critical
systems, processes, and areas, who understands each? Identify the bus-factor-1s (things only one person
knows — including the non-obvious tribal knowledge). (2) Pick the most critical single point of failure
and plan to de-risk it (docs, pairing, rotation) — framing it as freedom for the person, not threat. (3)
Assess your <em>own</em> bus factor — what do only you know/do as lead? Plan to spread it (grow
successors, document your context). (4) If there's a context-hoarding hero, plan how you'd approach them
(reframe as freedom, value their teaching, don't attack). Reflect: where is your team (and you) most
fragile to a single person being lost, and what's the first de-risking step?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds bus-factor awareness and de-risking — making no one (including the lead) an
irreplaceable single point of failure, done without alienating anyone. A strong response maps the
knowledge (finding the bus-factor-1s, including tribal knowledge), plans to de-risk the most critical one
(docs, pairing, rotation, framed as freedom), assesses the lead's own bus factor, and plans how to
approach any context-hoarding hero. The realizations: (1) <strong>bus factor 1 is a serious fragility</strong>
— when only one person knows a critical thing, the team is one vacation/illness/departure from crisis,
including when the lead is the sole holder of context; (2) <strong>de-risk with docs, pairing, and
rotation</strong> — map the knowledge first (can't de-risk what you haven't identified), then spread it
through documentation, teaching/pairing, and rotation, ongoing; (3) <strong>frame redundancy as freedom,
not threat</strong> — being non-irreplaceable is what lets people take real vacations, switch off, move
to new work, and get promoted; framed as freedom people embrace it, framed as replaceability they resist;
(4) <strong>handle the hoarding hero carefully</strong> — reframe sole-expertise as a burden, make sharing
valued expert-led work, don't attack their value (the hoarding comes from insecurity); (5) <strong>de-risk
your own bus factor</strong> — the lead is often a single point of failure too (and can't vacation or get
promoted if irreplaceable). On the assessment, people commonly find several bus-factor-1s (critical
systems only one person knows, and often the lead holding all the context/stakeholder knowledge), and that
they haven't deliberately spread this knowledge. The highest-value first step is usually: identify the
most critical single point of failure and start de-risking it (docs + pairing + rotation), framed as
freedom for the person — and begin de-risking the lead's own context by growing successors. The meta-
point: a bus factor of 1 is a serious, common fragility that turns normal events (vacations, illness,
departures) into crises, and de-risking single points of failure — including the lead's own — is a core
leadership responsibility. The key to doing it without alienating people is framing redundancy as freedom
(it serves the person: vacations, growth, promotion) rather than replaceability (which threatens), and
handling the context-hoarding hero with care (reframe, value their teaching, address the insecurity). This
completes Phase 6 (Delegation & Growing the Team): delegating outcomes at the right rung, assigning for
growth, career conversations, growing seniors into leads, and de-risking the bus factor — the mechanisms
by which a lead scales themselves and grows a resilient, developing team. De-risking (this lesson) is the
resilience counterpart to growing people (the rest of the phase): you both develop people <em>and</em>
ensure no one (including you) is an irreplaceable single point of failure — which together make a team
that's both growing and robust. The next phase (Influence Without Authority) shifts from developing your
own team to a different core leadership skill: getting things done through people you don't manage, using
influence rather than authority.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 7 — Influence Without Authority (Lesson 35: Sources of Influence) →](lesson-35-sources-of-influence){: .btn .btn-primary }
