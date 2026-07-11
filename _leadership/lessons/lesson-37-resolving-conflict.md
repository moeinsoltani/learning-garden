---
title: "Lesson 37 — Resolving Conflict"
nav_order: 3
parent: "Phase 7: Influence Without Authority"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 37: Resolving Conflict

## Concept

Conflict is inevitable on any team — and a lead's job isn't to eliminate it (some conflict is
healthy) but to keep it **productive**: moving disputes from *people vs. people* (destructive) to
*people vs. problem* (constructive). The key distinctions are between **task conflict** (disagreement
about the work — often healthy) and **relationship conflict** (personal friction — corrosive), and
between **positions** (what people say they want) and **interests** (why they want it — where
resolution usually lives).

```
   TASK CONFLICT (healthy)          RELATIONSHIP CONFLICT (corrosive)
   ──────────────────────          ────────────────────────────────
   disagreeing about the WORK       personal friction, it's-about-them
   (the design, the approach)       (dislike, resentment, ego)
   → surfaces better ideas          → damages the team, no upside
   KEEP it task, don't let it       → the lead's job: keep conflict on
   become personal                     the task, not the people

   POSITIONS ("we must use X") vs INTERESTS (WHY they want X)
   → resolution usually lives in the interests, not positions
```

The reframe: **move conflict from people-vs-people to people-vs-problem, and look for the interests
behind the positions.** Task conflict (about the work) is healthy and produces better outcomes —
you don't want to suppress it. Relationship conflict (personal) is corrosive and you want to defuse
it. And most resolvable conflicts get stuck at the level of *positions* (incompatible demands) when
the resolution is found by digging to the underlying *interests* (which often <em>are</em>
compatible).

---

## How It Works

### Task conflict vs relationship conflict

A crucial distinction: **task conflict** (disagreement about the work itself — the design, the
approach, the priority) is often **healthy** — it surfaces different perspectives, pressure-tests
ideas, and produces better outcomes; a team with no task conflict is probably not challenging each
other's thinking (groupthink). **Relationship conflict** (personal friction — dislike, resentment,
ego, it's-about-the-person) is **corrosive** — it has no upside, damages the team, and makes everything
worse. The lead's job is to **encourage healthy task conflict while preventing/defusing relationship
conflict** — and, critically, to keep task conflict *from becoming* relationship conflict (a
disagreement about the design curdling into "I don't like working with them"). Keep the conflict on
the task, not the people.

### Interests behind positions

The key to resolving many conflicts: distinguish **positions** (what people say they want — "we must
use Kafka") from **interests** (why they want it — "I need reliable async processing," "I'm worried
about operational complexity"). Conflicts get stuck when people argue incompatible *positions*; they
resolve when you dig to the underlying *interests*, which are often compatible or reveal a solution
that serves both. Ask "why?" and "what are you really trying to achieve/avoid?" to surface interests.
Two people arguing positions ("Kafka!" vs "cron!") may have interests (reliable processing, low
complexity) that a third option serves. Resolution usually lives in the interests, not the positions.
(This is from *Getting to Yes* — interest-based negotiation.)

### The mediator stance — for conflicts within your team

When two people on your team are in conflict, you often need to **mediate**. The mediator stance: (1)
stay **neutral** (don't take sides — you're helping them resolve it, not judging who's right); (2)
help each **feel heard** (let both express their view/interests, acknowledge each); (3) **surface the
interests** behind their positions (dig to why); (4) **refocus on the shared problem/goal** (move from
you-vs-them to us-vs-the-problem — "you both want the code to be maintainable; let's find an approach
that gets there"); and (5) help them find a resolution *they* own (rather than imposing one). Often the
biggest thing is just getting them to actually talk (conflicts fester when people avoid each other) and
to hear each other's interests.

### Escalation as a service, not a failure

Sometimes a conflict can't be resolved at your level and needs to go up (to a higher manager, a
decision-maker). Reframe **escalation as a service, not a failure**: escalating a genuine impasse to
someone who can resolve it (a decision that needs a higher authority, a cross-team conflict needing a
shared manager) is the responsible thing, not an admission of defeat. The failure is letting an
unresolvable conflict fester because escalating feels like failing. Escalate well (present it neutrally,
with the interests and options, asking for a decision), and treat it as moving the conflict to where it
can be resolved — a service to everyone stuck in it.

### When to let a disagreement stand

Not every disagreement needs resolution. Sometimes the right move is to **let a disagreement stand** —
people can disagree and still work together (disagree-and-commit, Lesson 38; or simply agreeing to
differ on something that doesn't need consensus). Forcing resolution on every difference is exhausting
and unnecessary; some disagreements are fine to leave unresolved (a matter of taste, a genuine judgment
call where either way works, a philosophical difference that doesn't block the work). The skill includes
knowing which conflicts need resolving (they're blocking or corrosive) and which can simply stand.

{: .note }
> **Move conflict from people-vs-people to people-vs-problem — and dig to interests behind positions</br>**
> Conflict is inevitable; the lead's job is keeping it productive. Distinguish <em>task conflict</em>
> (about the work — healthy, surfaces better ideas; keep it from becoming personal) from <em>relationship
> conflict</em> (personal friction — corrosive, defuse it). Resolve stuck conflicts by digging from
> <em>positions</em> (incompatible demands) to <em>interests</em> (the why behind them — often compatible,
> revealing a solution). For conflicts within your team, take the mediator stance (neutral, help each feel
> heard, surface interests, refocus on the shared problem, let them own the resolution). Treat escalation
> as a service (moving an impasse to where it can be resolved), not a failure. And know when to let a
> disagreement simply stand (not everything needs resolving). The through-line: keep conflict on the
> problem, not the people — which turns it from destructive into a force for better outcomes.

---

## Lab — Scenario

**The situation:** Two of your senior engineers, **Elena** and **Sam**, are in a slow-burning conflict
over code style — specifically, how much abstraction is "right" (Elena favors more abstraction/patterns,
Sam favors simpler, more concrete code). It started as a technical disagreement but has curdled: their
code review comments to each other have gotten terse and pointed, they're clearly annoyed with each
other, and it's starting to affect the team's atmosphere. What began as task conflict is becoming
relationship conflict. You need to mediate.

**Write how you'd mediate the joint conversation** — how you'd set it up and run it, and what you'd aim
for. Then note the principles and mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong mediation stays neutral, surfaces both interests, refocuses on the shared goal, and keeps the
conflict on the task (pulling it back from the personal). Example:
<br><br>
<strong>Setup:</strong> Talk to each 1:1 first (understand each side and defuse a bit), then bring them
together in a private, neutral conversation. Frame it constructively: "I want us to sort out the
abstraction disagreement, because you're both excellent engineers and I don't want a technical difference
to affect how you work together."
<br><br>
<strong>Running it — stay neutral, surface interests, refocus:</strong>
<br><br>
"I want to start by saying I'm not here to declare who's right — you both have strong technical judgment,
and I think this is a real, legitimate disagreement about a genuine trade-off. <em>(Neutral — not taking
sides; validates it as a real technical question, not a personal failing.)</em>
<br><br>
Elena, help me understand what you're optimizing for with more abstraction. [Interests: probably
reusability, DRY, handling future change.] Sam, same for you — what are you optimizing for with simpler
code? [Interests: readability, less indirection, easier for others to follow, avoiding premature
abstraction.] <em>(Surface the interests behind the positions — why each prefers their style.)</em>
<br><br>
So here's what I'm hearing: you both actually want the same thing — <strong>maintainable, quality
code</strong> — you just weigh the trade-offs differently (Elena weighs future flexibility, Sam weighs
present simplicity). That's a real, legitimate tension in engineering, and honestly, <em>both</em> of you
are right in different contexts. <em>(Refocus from you-vs-them to the shared goal (maintainable code) —
people-vs-problem — and reframe it as a legitimate trade-off, not a personal battle.)</em>
<br><br>
Can we agree on some shared principles for when more abstraction is worth it vs. when simpler is better?
[e.g., abstract when there's real, proven repetition or known future change; keep it simple until then —
'avoid premature abstraction' is a shared value.] And can we agree that in reviews, this is a judgment
call to discuss respectfully, not a right/wrong to fight over? <em>(Move to a resolution they co-own — a
shared team standard — and explicitly address the review-tone problem.)</em>
<br><br>
And — I want to name gently — the review comments between you have gotten a bit sharp lately. I get it,
this is frustrating, but let's reset that: you're on the same team, and I'd like the tone to be
collaborative even when you disagree. <em>(Address the relationship-conflict spillover directly but
kindly — pull it back from personal.)</em>"
<br><br>
<strong>Aim for:</strong> a shared standard (or agreement it's a case-by-case judgment call), a reset of
the review tone, and both feeling heard and respected.
<br><br>
<strong>Principles:</strong> (1) <strong>Stay neutral</strong> — don't declare a winner; it's a
legitimate trade-off, and taking sides makes it worse. (2) <strong>Surface interests behind positions</strong>
— what each is optimizing for (flexibility vs. simplicity) reveals they share the goal (maintainable code).
(3) <strong>Refocus people-vs-problem</strong> — "you both want maintainable code" turns a personal battle
into a shared problem. (4) <strong>Pull it back from relationship conflict</strong> — name the sharp review
tone and reset it (keep it task, not personal). (5) <strong>Co-owned resolution</strong> — a shared
principle/standard they agree to, not an imposed verdict. (6) <strong>Both feel heard</strong> — each
explains their view and is acknowledged. <strong>Mistakes to avoid:</strong> (1) <strong>taking sides /
declaring a winner</strong> — makes the "loser" resentful and doesn't resolve the relationship damage; (2)
<strong>staying at the level of positions</strong> ("abstraction vs. simple") without digging to interests
(where the shared goal and resolution live); (3) <strong>ignoring the relationship-conflict spillover</strong>
— resolving the technical point but not the sharp tone/personal friction (which will continue); (4)
<strong>imposing a resolution</strong> — a standard they don't own won't stick; (5) <strong>doing it
publicly</strong> or letting it fester unaddressed (conflicts grow when avoided); (6) <strong>treating a
legitimate trade-off as one person being wrong</strong> — it's a real judgment call; framing it as
right/wrong alienates someone unnecessarily; (7) <strong>not getting them actually talking</strong> — much
of the fix is getting them to hear each other's interests directly (they've been fighting via terse review
comments, not really communicating).
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Interests vs positions | *Getting to Yes*, Fisher & Ury (interest-based negotiation) |
| Task vs relationship conflict | Amy Edmondson; conflict research (De Dreu & Weingart) |
| Defusing emotion (related) | Lesson 23 (defusing emotions); English track Lesson 48 |
| The mediator stance | *Crucial Conversations*; Lesson 22 |

---

## Checkpoint

**Q1.** What's the difference between task conflict and relationship conflict, and why is the lead's job
to keep conflict "people vs. problem"?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Task conflict</strong> is disagreement about the <em>work</em> — the design, the approach, the
priorities, the technical trade-offs. <strong>Relationship conflict</strong> is <em>personal</em> friction
— dislike, resentment, ego, "it's about the person," clashes rooted in personality or animosity rather
than the work. The crucial difference is that they have opposite effects: <strong>task conflict is often
healthy</strong>, while <strong>relationship conflict is corrosive</strong>. Task conflict is healthy
because disagreement about the work surfaces different perspectives, pressure-tests ideas, and produces
better outcomes — a team where people challenge each other's technical thinking (constructively) reaches
better decisions than one where everyone agrees (which risks groupthink and unexamined ideas). So you
<em>want</em> some task conflict — the absence of it usually means people aren't really engaging or
challenging each other. Relationship conflict, by contrast, has no upside: personal friction (people
disliking each other, resentment, ego battles) damages the team, poisons the atmosphere, makes
collaboration hard, and makes everything worse — there's nothing productive in it. So the two need
opposite treatment: encourage/allow healthy task conflict (it's good), and prevent/defuse relationship
conflict (it's harmful). Critically, the lead must also keep task conflict from <em>becoming</em>
relationship conflict — a disagreement about the design curdling into "I don't like working with them"
(as in the code-style example, where a technical disagreement turned into personal friction and sharp
review tone). Task conflict handled badly (getting personal, egos engaged) degrades into relationship
conflict, which is corrosive — so keeping conflict <em>on the task</em> and off the people is central. Why
the lead's job is to keep conflict "people vs. problem": <strong>because framing conflict as people
against a shared problem (rather than people against each other) is what makes it productive rather than
destructive — it channels the disagreement toward solving the issue together instead of fighting each
other</strong>. "People vs. problem" means the parties are collaborators tackling a shared problem
(different views on how to solve it), rather than adversaries opposing each other. This framing: (1)
<strong>keeps the conflict on the task</strong> (the problem to solve) rather than the people (who's right,
who's better) — which keeps it healthy task conflict rather than corrosive relationship conflict; (2)
<strong>makes it collaborative</strong> — if we're both working against the problem, we're on the same
side (allies solving something), not enemies, so we can disagree productively and reach a better solution
together; (3) <strong>preserves the relationship</strong> — people-vs-problem doesn't damage how people
feel about each other (they're collaborating), where people-vs-people erodes the relationship. So the
lead's job is to keep conflict framed as people-vs-problem — encouraging the healthy task conflict (about
the work) while preventing it from becoming people-vs-people (personal, relationship conflict) — because
that framing is what makes conflict a force for better outcomes (the productive clash of ideas) rather
than a destructive force that damages the team. Concretely, this means: welcoming disagreement about the
work, refocusing disputes on the shared goal/problem ("you both want maintainable code"), and pulling
conflict back from the personal whenever it starts to curdle (naming and resetting sharp tone, keeping it
about the task). The distinction and the "people vs. problem" framing together are the core of a lead
managing conflict well: not eliminating conflict (healthy task conflict is valuable) but keeping it
productive (on the problem, not the people).
</details>

**Q2.** Why does resolution often live in the "interests behind the positions," and what is the mediator
stance for conflicts within your team?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Resolution often lives in the "interests behind the positions" because <strong>positions are the specific
(often incompatible) things people demand, while interests are the underlying needs/goals driving those
demands — and the underlying interests are frequently compatible or reveal a solution that serves both,
even when the positions clash</strong>. A <strong>position</strong> is what someone says they want ("we
must use Kafka," "we have to use more abstraction") — a specific stated demand. An <strong>interest</strong>
is <em>why</em> they want it — the underlying need, goal, or concern ("I need reliable async processing,"
"I'm worried about handling future change"). Conflicts get stuck when people argue at the level of
<em>positions</em>: two incompatible demands ("Kafka!" vs "cron!") with no middle ground, so it becomes a
win/lose battle. But when you dig beneath the positions to the <em>interests</em> — why each person wants
what they want — you often find: (1) <strong>the interests are compatible</strong> — the underlying needs
don't actually conflict, even though the stated positions do (they were just reaching for different
solutions to compatible needs); (2) <strong>a third option serves both interests</strong> — once you know
what each really needs (reliable processing, low complexity), you can often find a solution neither
originally proposed that satisfies both; or (3) <strong>the real concern becomes addressable</strong> —
surfacing the interest ("I'm worried about operational complexity") lets you address that specific concern
directly. So the resolution that's invisible at the position level (incompatible demands) becomes visible
at the interest level (compatible needs, or a mutually-satisfying option). The technique (from <em>Getting
to Yes</em>) is to ask "why?" and "what are you really trying to achieve or avoid?" to surface the
interests, then solve for those rather than fighting over positions. Positions are where conflicts get
stuck; interests are where they resolve. The <strong>mediator stance</strong> for conflicts within your
team: when two people on your team are in conflict, you mediate — help <em>them</em> resolve it — using
this stance: (1) <strong>stay neutral</strong> — don't take sides or judge who's right; your role is to
help them resolve it, not to declare a winner (taking sides makes the "loser" resentful and doesn't heal
the relationship). (2) <strong>help each feel heard</strong> — let both express their view and their
interests, and acknowledge each, so both feel understood (feeling heard is often necessary before people
can move toward resolution). (3) <strong>surface the interests behind the positions</strong> — dig to why
each wants what they want (as above), because that's where resolution lives. (4) <strong>refocus on the
shared problem/goal</strong> — move from you-vs-them to us-vs-the-problem by identifying the goal they
share ("you both want maintainable code") and framing the conflict as jointly solving that, rather than
fighting each other (the people-vs-problem reframe). (5) <strong>help them find a resolution they own</strong>
— guide them to a solution they agree on and co-own, rather than imposing one (an imposed resolution
doesn't stick or heal the relationship; a co-owned one does). Additionally, often the biggest thing the
mediator does is simply <strong>get them actually talking</strong> — conflicts fester when people avoid
each other (communicating through terse channels or not at all), so getting them into a real conversation
where they hear each other's interests directly is much of the resolution. The mediator stance is thus
about facilitating <em>their</em> resolution — neutral, helping both feel heard, surfacing the interests,
refocusing on the shared problem, and letting them own the outcome — rather than the lead judging or
imposing. It's the interest-based approach (dig to interests, find the shared goal) applied to helping two
team members resolve their conflict, keeping it people-vs-problem and preserving the relationship. Both
points reflect the core of resolving conflict productively: get beneath the clashing positions to the
underlying interests (where resolution lives), and, as mediator, facilitate the parties to a co-owned
resolution around their shared goal (neutral, both heard, interests surfaced) — rather than fighting over
positions or imposing a verdict, either of which fails to truly resolve the conflict.
</details>

---

## Homework

Reflect on conflict you're dealing with or have seen. (1) Is it task conflict (about the work — often
healthy) or relationship conflict (personal — corrosive)? Is any task conflict curdling into personal?
(2) For a stuck conflict, dig beneath the positions to the interests — what does each person really want/
need? Is there a solution in the interests that the positions obscured? (3) If two people on your team are
in conflict, practice the mediator stance (neutral, both heard, surface interests, refocus on the shared
problem, let them own the resolution). (4) Is there a conflict you should escalate (as a service) or one
you can let stand? Reflect: how do you tend to handle conflict (avoid it? take sides? force resolution?),
and what would keeping it "people vs. problem" change?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds conflict-resolution skill — keeping conflict productive rather than destructive. A
strong response distinguishes task from relationship conflict (and watches for task curdling into
personal), digs beneath positions to interests on a stuck conflict, practices the mediator stance for a
team conflict, and considers escalation-as-service or letting a disagreement stand. The realizations: (1)
<strong>task conflict is healthy, relationship conflict is corrosive</strong> — disagreement about the
work surfaces better ideas (you want it), while personal friction damages the team (defuse it), and the
lead must keep task conflict from becoming personal; (2) <strong>resolution lives in interests, not
positions</strong> — clashing positions (incompatible demands) often hide compatible interests (the why),
so digging beneath positions reveals solutions the positions obscured; (3) <strong>the mediator stance</strong>
— for team conflicts, stay neutral, help both feel heard, surface interests, refocus on the shared problem
(people-vs-problem), and let them own the resolution (rather than judging or imposing); (4) <strong>escalation
is a service, not a failure</strong> (moving an impasse to where it can be resolved), and <strong>not every
disagreement needs resolving</strong> (some can stand). On reflection, people commonly find a default
conflict style: avoiding it (letting it fester — common for conflict-averse people), taking sides (which
resentment), or forcing resolution on everything (exhausting and unnecessary). Recognizing the default is
the first step. The most common realization is that keeping conflict "people vs. problem" (refocusing on
the shared goal, digging to interests, keeping it on the task not the people) transforms it from a
destructive personal battle into a productive joint problem-solving — and that many stuck conflicts resolve
once you get beneath the positions to the interests. The meta-point: conflict is inevitable, and the lead's
job isn't to eliminate it (healthy task conflict is valuable) but to keep it productive — moving it from
people-vs-people to people-vs-problem, digging to interests, mediating neutrally, escalating when needed,
and letting some disagreements stand. This keeps conflict a force for better outcomes rather than a
destroyer of teams. It builds on the influence and de-escalation skills (Lesson 35, Lesson 23), and it's a
core part of leading through people you don't fully control. The next lesson addresses a related challenge —
aligning teams around a direction — creating alignment that survives after you leave the room, so that
teams don't drift into building incompatible things.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 38 — Aligning Teams Around a Direction →](lesson-38-aligning-teams){: .btn .btn-primary }
