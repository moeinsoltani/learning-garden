---
title: "Lesson 54 — Prioritization and Trade-offs"
nav_order: 5
parent: "Phase 10: Project Leadership"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 54: Prioritization and Trade-offs

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **the one ranked list** — a single visible priority order; makes every "no" a transparent trade-off instead of a personal rejection.
> - **cost of delay** — how much value is lost per week of postponing something; value × urgency.
> - **WSJF** — Weighted Shortest Job First: cost of delay ÷ effort; do high-value, cheap, urgent things first.
> - **urgent vs important** — the Eisenhower distinction: deadline-pressured vs genuinely valuable; the urgent crowds out the important unless you protect it.
> - **the tyranny of the urgent** — the pattern where fires consume all capacity forever.
> - **interrupt budget** — a fixed share of team capacity reserved for unplanned work (support, bugs).
> - **on-call / support rotation** — one person absorbing the interrupts so the rest keep focus (Lesson 34/42).
> - **"not now" vs "no"** — deferred vs declined; most good ideas are "not now."
> - **strings along** — keeps someone hoping with vague maybes (Lesson 42).
> - **who shouts loudest** — the anti-method: prioritizing by noise instead of value.

## Concept

As a lead, you **say no (or "not now") all day** — there's always more demand than capacity, and your job
is to decide what the team does and (mostly) doesn't do. Done badly, this makes you a bottleneck or a
source of frustration; done well, it focuses the team on what matters while keeping requesters trusting
you. The keys: **making prioritization visible** (one ranked list everyone can see), using **cost of
delay** to prioritize by value-and-urgency, protecting an **interrupt budget** for the unplanned, and
saying no in a way that stays trusted.

```
   PRIORITIZATION (saying no all day, staying trusted)
   ┌──────────────────────────────────────────────────────┐
   │ • ONE ranked list, VISIBLE (prioritization is public)  │
   │ • COST OF DELAY: value × urgency (what's costly to      │
   │   delay ranks higher) — WSJF-lite                      │
   │ • URGENT vs IMPORTANT trap (urgent crowds out important)│
   │ • INTERRUPT BUDGET for support/bugs (protect focus)     │
   │ • say NO / "not now" clearly, with the WHY — stay trusted│
   └──────────────────────────────────────────────────────┘
```

The reframe: **prioritization is deciding what NOT to do, made visible and value-based — and saying no
well keeps you trusted.** Everything can't be top priority (if everything's a priority, nothing is), so
prioritization is fundamentally about the trade-offs — what to do <em>instead of</em> other things. Making
the priorities visible (one ranked list) turns "no" from a personal rejection into a transparent trade-off;
prioritizing by cost of delay focuses on value; and saying no clearly and kindly (with the why) keeps
requesters trusting you even when you decline them.

---

## How It Works

### Making prioritization visible — the one ranked list

The most powerful prioritization tool: **one visible, ranked list** of what the team is working on and
what's queued, in priority order, that everyone can see. This transforms prioritization: (1) it makes
trade-offs <em>explicit</em> — "we can do X <em>or</em> Y, and X is ranked higher" (so saying no to Y is a
visible trade-off, not a personal rejection); (2) it lets requesters see <em>where their thing ranks</em>
(and why) rather than feeling arbitrarily declined; (3) it forces real prioritization (a single ranked
list means things must be ordered — you can't have ten "top priorities"); and (4) it enables the "if you
want X higher, what comes down?" conversation (making the trade-off concrete). Visible prioritization turns
prioritization from opaque, individual "no"s into a transparent, shared understanding of the trade-offs.

### Cost of delay — prioritize by value and urgency

A useful prioritization lens: **cost of delay** — how much value is lost by delaying something. This
combines <em>value</em> (how important/valuable) and <em>urgency</em> (how time-sensitive — does delaying
it cost more over time?). Something high-value <em>and</em> urgent (costly to delay) ranks above something
high-value but not urgent (fine to delay) or low-value-but-urgent. **WSJF** (Weighted Shortest Job First,
"lite") refines this: prioritize by cost-of-delay divided by effort (do the high-cost-of-delay, low-effort
things first — best value per effort). The point isn't a precise formula but the lens: prioritize by what's
costly to delay (value × urgency) relative to effort, not by who shouts loudest or what's newest. Cost of
delay grounds prioritization in value rather than noise.

### The urgent-vs-important trap — at team scale

A classic trap (Eisenhower), acute at team scale: **urgent things crowd out important ones.** Urgent things
(a support ticket, an interrupt, a fire) demand immediate attention and get done; important-but-not-urgent
things (the strategic work, the tech-debt paydown, the big initiative) have no deadline pressure, so they
keep getting deferred for the urgent — and never happen. As a lead, you must <em>protect</em> the important-
but-not-urgent work from being perpetually crowded out by the urgent — by deliberately allocating capacity
to it (not just filling the team's time with whatever's urgent), because the important work is usually where
the real value is, and it only happens if protected. Don't let the tyranny of the urgent consume all the
capacity that should go to the important.

### Interrupt budgets — protect focus from the unplanned

Unplanned work (support, bugs, urgent requests) is inevitable, and if unmanaged it destroys the planned
work (constant interrupts mean the committed work doesn't get done). The tool: an **interrupt budget** —
deliberately allocate a portion of the team's capacity to the unplanned (support/bugs/interrupts), and
protect the rest for planned work. Often via an **on-call/support rotation** (one person handles the
interrupts, shielding the others' focus). This (1) accounts for the inevitable unplanned work (so it
doesn't blow the plan — it's budgeted), (2) protects the majority's focus (they're not all constantly
interrupted), and (3) makes the trade-off explicit (X% goes to interrupts, so plan the rest accordingly).
Budgeting for interrupts is how you protect planned work from being consumed by the unplanned.

### Saying no / "not now" — clearly, with the why, staying trusted

You'll say no (or "not now") constantly, and <em>how</em> determines whether you stay trusted. Say it: (1)
<strong>clearly</strong> (a clear "no" or "not now" beats a vague "maybe/we'll see" that strings people
along — Lesson 42/English track); (2) <strong>with the why</strong> (explain the trade-off — "we're
prioritizing X because Y, so this is 'not now'" — so it's a reasoned decision, not an arbitrary rejection);
(3) <strong>often as "not now," not "never"</strong> (many things are worth doing later, just not now —
"not now" is honest and less final); and (4) <strong>with the visible list</strong> (point to where it
ranks and why — the transparent trade-off). Said this way — clear, reasoned, transparent — a "no" keeps the
requester trusting you (they see it's a fair trade-off, not a brush-off), where a vague, unexplained, or
inconsistent "no" frustrates and erodes trust. And **disagreeing with priorities you must still execute**:
if leadership sets a priority you disagree with, voice it (respectfully), then execute it (disagree-and-
commit) — you can push back on priorities but still own executing the decision.

{: .note }
> **Prioritization is deciding what NOT to do — make it visible, value-based, and say no while staying trusted</br>**
> A lead says no all day (demand always exceeds capacity), so prioritization — deciding what the team does
> and doesn't do — is core. Make it <em>visible</em> (one ranked list everyone sees — turning "no" into a
> transparent trade-off, and forcing real ordering). Prioritize by <em>cost of delay</em> (value × urgency,
> relative to effort — WSJF-lite), not by who shouts loudest. Beware the <em>urgent-vs-important trap</em>
> (protect the important-but-not-urgent from being crowded out by the urgent). Budget for <em>interrupts</em>
> (allocate capacity to the inevitable unplanned work, protecting planned focus). And say <em>no / "not now"
> clearly, with the why</em> (a reasoned, transparent trade-off keeps requesters trusting you). And when you
> disagree with a priority you must execute, disagree-and-commit. Prioritizing well — visibly, by value, saying
> no while staying trusted — is how a lead focuses the team on what matters.

---

## Lab — Scenario

**The situation:** Your team's sprint is planned and committed — everyone knows what they're working on.
Then **Monday morning, three things land at once**: (1) the **CEO** has a "quick" pet request (a small
feature they personally want — not strategically critical, but it's the CEO); (2) a **customer P1** — a
major customer is hitting a serious bug that's blocking them; and (3) a **security patch** — a
vulnerability was disclosed that needs patching. All three want to jump the queue, and your committed sprint
work is now under threat. You need to re-prioritize (publicly) and communicate with the three requesters.

**Re-rank the work publicly and write the messages to the three requesters.** Then note the principles and
mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong response prioritizes by real urgency/impact (not by who's most senior), makes the re-rank visible,
protects some committed work, and communicates clearly with each requester. Example:
<br><br>
<strong>Re-rank by cost of delay / real urgency (not by seniority of requester):</strong> <br>
1. <strong>Security patch — first.</strong> A disclosed vulnerability is genuinely urgent and high-impact
(security risk, possibly exploitable now) — highest cost of delay. Do it immediately. <br>
2. <strong>Customer P1 — second (roughly parallel).</strong> A major customer blocked by a serious bug is
urgent and high-impact (customer at risk, revenue/relationship). Get someone on it right away. <br>
3. <strong>CEO's pet request — scheduled, not jumped.</strong> Despite being from the CEO, it's <em>not</em>
strategically critical or urgent (a nice-to-have feature). It should be prioritized on its merits, not by the
requester's seniority — so it goes into the queue at its real priority (soon, but not ahead of the security
and customer fires, and probably after the committed sprint work it'd displace). <br>
<em>(Prioritize by real urgency × impact — cost of delay — not by who asked; the CEO's rank doesn't make a
minor feature urgent.)</em>
<br><br>
<strong>Make the re-rank visible and protect committed work:</strong> "Update the visible priority list:
security and the P1 jump in (they're genuine fires); to absorb them, [specific committed work] slips or the
interrupt-budget/on-call person takes the P1 while the rest protect the sprint. Show the trade-off publicly —
'security + P1 came in, so X moves' — so it's a transparent re-prioritization, not chaos."
<br><br>
<strong>Message to the CEO (the delicate one):</strong> "Hi [CEO] — thanks for this, I like the idea. Quick
heads up on sequencing: we had two urgent fires land this morning (a security vulnerability and a major
customer outage) that we have to handle first. I've put your request in the queue and we should be able to
get to it [timeframe] — I'll keep you posted. Wanted to be transparent about why it's not immediate. Happy to
discuss if it's more urgent than I'm reading it." <em>(Respectful, clear it's queued (not ignored), explains
the why (real fires first), and opens the door if the CEO genuinely thinks it's more urgent — but doesn't
just drop everything for seniority.)</em>
<br><br>
<strong>Message to the customer P1 requester:</strong> "We're on it — I've got [engineer] looking at the
[customer] issue right now as top priority, and I'll update you within [X hours] with a status. Sorry for the
disruption to them; we're treating this as urgent." <em>(Fast, clear ownership, a timeline — appropriate for a
real P1.)</em>
<br><br>
<strong>Message re: the security patch:</strong> "Prioritizing the [vulnerability] patch immediately —
[engineer] is on it, targeting [timeframe]. Will confirm when it's deployed." <em>(Immediate action on a
genuine security fire.)</em>
<br><br>
<strong>To the team (protecting focus):</strong> "Security + the P1 are today's fires — [names] on those.
Everyone else, stay on the committed sprint work; [committed item X] will slip to absorb this, which I'll
communicate. Let's not let the whole sprint get derailed." <em>(Interrupt budget — a few take the fires, the
rest protect planned work; explicit about what slips.)</em>
<br><br>
<strong>Principles:</strong> (1) <strong>Prioritize by real urgency × impact (cost of delay), not by
requester seniority</strong> — the security fire and customer P1 are genuinely urgent; the CEO's minor
feature isn't (its priority is its merits, not the CEO's rank). (2) <strong>Make the re-rank visible</strong>
— transparent trade-off, not chaos. (3) <strong>Protect committed work / interrupt budget</strong> — a few
handle the fires, the rest protect the sprint; be explicit about what slips. (4) <strong>Say no/not-now
clearly with the why</strong> — especially to the CEO (respectful, queued, explained). (5) <strong>Fast clear
ownership on real fires</strong> — security and P1 get immediate action and timelines. (6) <strong>Stay
trusted</strong> — everyone (including the CEO) sees a fair, transparent, reasoned re-prioritization.
<strong>Mistakes to avoid:</strong> (1) <strong>dropping everything for the CEO</strong> because they're the
CEO (prioritizing by seniority, not merit — the minor feature isn't urgent, and this trains the org that
shouting/rank jumps the queue); (2) <strong>ignoring the CEO / a curt no</strong> (the other extreme — be
respectful and transparent, queue it, open the door); (3) <strong>letting all three derail the whole
sprint</strong> — not protecting any committed work (interrupt budget); (4) <strong>invisible re-prioritization</strong>
— just shuffling silently (chaos, no transparency); (5) <strong>vague responses</strong> ("we'll try to get
to it") — be clear (immediate / queued for X); (6) <strong>not communicating what slips</strong> — the
committed work that's displaced needs its own no-surprises heads-up; (7) <strong>treating the P1 or security
as routine</strong> — they're genuine fires needing immediate action.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Cost of delay / WSJF | Don Reinertsen, *Principles of Product Development Flow*; SAFe WSJF |
| Urgent vs important | Eisenhower matrix; *7 Habits*, Covey |
| Saying no / prioritization | *Essentialism*, McKeown; English track Lesson 18 (saying no) |
| Interrupt budgets / on-call | Google SRE book (toil, on-call); protecting focus |

---

## Checkpoint

**Q1.** Why is making prioritization visible (one ranked list) so powerful, and why prioritize by "cost of
delay" rather than who asks?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Making prioritization visible (one ranked list everyone can see) is powerful because <strong>it transforms
prioritization from opaque individual "no"s into a transparent, shared understanding of trade-offs — which
makes "no" a visible trade-off rather than a personal rejection, forces real ordering, and enables concrete
trade-off conversations</strong>. Several specific benefits: (1) <strong>it makes trade-offs explicit</strong>
— a single ranked list shows that resources are finite and things must be ordered ("we can do X <em>or</em>
Y, and X ranks higher"), so declining Y is visibly a <em>trade-off</em> (we chose X over Y given limited
capacity) rather than a personal rejection of Y or its requester; the "no" is depersonalized into a
transparent prioritization decision. (2) <strong>requesters see where their thing ranks and why</strong> —
instead of feeling arbitrarily declined (which breeds resentment and repeated asking), they can see their
request's position in the ranked list and the reasoning, so even a "not now" feels fair (they understand
it's ranked below other things for reasons, not ignored). (3) <strong>it forces real prioritization</strong>
— a single ranked list means things must actually be <em>ordered</em> (you can't have ten "top priorities" —
the list forces a real sequence), which prevents the everything-is-priority-so-nothing-is failure. (4)
<strong>it enables the trade-off conversation</strong> — with a visible ranked list, "if you want your thing
higher, what comes down?" becomes a concrete conversation (you can point to what would have to be
deprioritized to elevate their request), making the cost of re-prioritizing explicit rather than letting
people demand elevation without acknowledging the trade-off. So visible prioritization turns prioritization
into a transparent, shared, ordered understanding — which makes saying no a fair, visible trade-off (keeping
requesters trusting you), forces genuine ordering, and grounds re-prioritization requests in real trade-offs.
It's far more powerful than opaque, case-by-case "no"s (which feel arbitrary and personal, breed resentment,
and don't force real prioritization). Why prioritize by <strong>cost of delay</strong> rather than who asks:
<strong>because cost of delay (value × urgency) prioritizes by what actually matters most to the outcome,
while prioritizing by who asks (seniority, volume) prioritizes by noise and politics — which optimizes the
wrong thing and trains bad behavior</strong>. Cost of delay — how much value is lost by delaying something,
combining value (how important) and urgency (how time-sensitive) — grounds prioritization in the actual value
and time-sensitivity of the work: the things most costly to delay (high value AND urgent) rank highest,
which correctly focuses the team on what matters most to the outcome. Prioritizing by <strong>who asks</strong>
instead (the most senior requester, the loudest voice, the newest request) has serious problems: (1) <strong>it
optimizes the wrong thing</strong> — the CEO's minor pet request isn't more valuable or urgent than a
customer outage just because the CEO asked; prioritizing by requester seniority (or volume) means the team
works on what powerful/loud people want rather than what's most valuable, misallocating effort away from what
matters. (2) <strong>it trains bad behavior</strong> — if shouting loudest or pulling rank jumps the queue,
people learn to shout and pull rank (and to escalate everything as urgent), so prioritization becomes a
politics/volume contest rather than a value assessment, and the loudest/most-senior always win regardless of
merit. (3) <strong>it's unfair and erodes trust</strong> — prioritizing by who-asks means requests are judged
by the requester's power, not the work's value, which is unfair (a valuable request from a junior loses to a
trivial one from a senior) and erodes trust in the prioritization. Prioritizing by cost of delay (value ×
urgency, relative to effort — WSJF-lite) instead focuses on the merits of the work (what's genuinely valuable
and urgent), which correctly allocates effort to what matters most, treats requests fairly (by their value,
not the requester's rank), and doesn't reward shouting/rank. Even for a high-status requester (the CEO), the
discipline is to prioritize their request on its <em>merits</em> (its real value and urgency), not to jump
it because of their status — which keeps prioritization value-based and fair. (You handle the status
diplomatically — respectfully, transparently — but you don't let rank override value.) Both points make
prioritization better: visible (transparent, ordered trade-offs — fair and forcing real prioritization) and
value-based (cost of delay — focusing on what matters, not who shouts) — together focusing the team on the
most valuable work while keeping the prioritization fair and trusted, versus opaque, politics-driven
prioritization that misallocates effort and erodes trust.
</details>

**Q2.** What is the "urgent-vs-important trap" and the interrupt budget, and why does saying no "clearly,
with the why" keep you trusted?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The <strong>urgent-vs-important trap</strong> is that <strong>urgent things (which demand immediate
attention) crowd out important-but-not-urgent things (which have no deadline pressure), so the important work
keeps getting deferred for the urgent and never happens</strong>. Work varies on two axes: urgency (how
time-pressured) and importance (how valuable). The trap is that <em>urgent</em> things — a support ticket, a
fire, an interrupt — demand immediate attention and so get done (their urgency forces action), while
<em>important-but-not-urgent</em> things — the strategic initiative, the tech-debt paydown, the big
long-term investment — have no deadline pressure, so they can always be deferred "until the urgent stuff
calms down" (which never happens, because there's always more urgent stuff). The result: the team's capacity
gets consumed by the endless stream of urgent things, and the important-but-not-urgent work (which is usually
where the <em>real value</em> is — the strategic, high-impact work) perpetually gets crowded out and never
happens. This is a trap because it feels productive (you're always busy handling urgent things) while the
most valuable work silently doesn't get done. The escape: a lead must <em>deliberately protect</em> the
important-but-not-urgent work — allocating capacity to it explicitly (not just filling all time with whatever's
urgent), because it only happens if protected from the tyranny of the urgent. The <strong>interrupt budget</strong>
is the tool for managing the urgent/unplanned work without letting it consume everything: <strong>deliberately
allocate a portion of the team's capacity to the inevitable unplanned work (support, bugs, interrupts), and
protect the rest for planned work</strong> — often via an on-call/support rotation where one person handles
the interrupts, shielding the others' focus. This (1) <strong>accounts for the inevitable unplanned work</strong>
— unplanned work <em>will</em> happen, so budgeting capacity for it (X% of the team) means it doesn't blow the
plan (it's expected and absorbed) rather than being an unaccounted surprise that derails everything; (2)
<strong>protects the majority's focus</strong> — with one person (on-call) taking the interrupts, the rest of
the team isn't constantly interrupted, so they can do focused planned work (versus everyone being interrupted,
which destroys everyone's productivity); and (3) <strong>makes the trade-off explicit</strong> — allocating
X% to interrupts means planning the rest accordingly (honest capacity planning). The interrupt budget is how
you protect planned work (including the important-but-not-urgent) from being consumed by the unplanned/urgent
— budgeting for the urgent so it doesn't eat everything. Why saying no <strong>"clearly, with the why"</strong>
keeps you trusted: <strong>because a clear, reasoned "no" is experienced as a fair, honest trade-off (which
maintains trust), while a vague, unexplained, or inconsistent "no" feels arbitrary and dismissive (which
erodes trust)</strong>. You say no constantly, and how you do it determines whether requesters keep trusting
you: (1) <strong>clearly</strong> — a clear "no" or "not now" (vs. a vague "maybe/we'll see" that strings
people along) is honest and respectful — people can plan around a clear no, whereas a vague non-answer leaves
them hanging and erodes trust when it turns out to be a no anyway (false hope). (2) <strong>with the why</strong>
— explaining the trade-off ("we're prioritizing X because Y, so your thing is not now") makes the no a
<em>reasoned decision</em> rather than an arbitrary rejection; the requester understands it's a fair
prioritization (their thing ranked below other things for real reasons), which they can accept, versus an
unexplained no that feels dismissive or arbitrary (breeding resentment). (3) <strong>often as "not now" not
"never"</strong> — honest (many things are worth doing later) and less final (the requester's thing isn't
rejected forever, just not now), which is easier to accept. (4) <strong>with the visible list</strong> —
pointing to where it ranks and why (the transparent trade-off) reinforces that it's fair. Said this way —
clear, reasoned, transparent, "not now" — a no keeps the requester trusting you: they see it's a fair
trade-off given real constraints and priorities, not a personal brush-off, so they trust that you're
prioritizing sensibly and treating their request fairly (even though you declined it). A vague, unexplained,
or inconsistent no does the opposite — it feels arbitrary, dismissive, or political, eroding trust and
breeding frustration. So how you say no is what lets you decline people all day (which you must) while
keeping them trusting you — the clear, reasoned, transparent, "not now" no maintains the relationship and
trust even in decline. All three points serve prioritizing well: protect the important from the urgent (don't
let the tyranny of the urgent crowd out the valuable), budget for interrupts (absorb the unplanned without it
consuming planned work), and say no clearly-with-the-why (decline while staying trusted) — together focusing
the team on what matters and keeping requesters trusting your prioritization, which is the essence of the
lead's constant job of deciding what the team does and doesn't do.
</details>

---

## Homework

Sharpen your prioritization. (1) Do you have one visible, ranked priority list, or opaque case-by-case
decisions? Make your priorities visible. (2) Are you prioritizing by cost of delay (value × urgency) or by who
asks (seniority/volume)? Check a recent prioritization. (3) Are you protecting important-but-not-urgent work,
or is the urgent crowding it out? Is there an interrupt budget protecting planned focus? (4) When you say no,
is it clear, with the why, and "not now" where honest — or vague? Practice a clear, reasoned no. Reflect: are
you prioritizing by value and staying trusted when you say no, and what's one thing (visible list, cost of
delay, interrupt budget, better no's) that would improve how you prioritize?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds prioritization — deciding what the team does and doesn't do, visibly and by value, while
staying trusted. A strong response makes priorities visible (one ranked list), prioritizes by cost of delay
(not who asks), protects important-but-not-urgent work with an interrupt budget, and says no clearly with the
why. The realizations: (1) <strong>make prioritization visible</strong> — one ranked list turns "no" into a
transparent trade-off (not personal rejection), forces real ordering, lets requesters see where they rank, and
enables the "what comes down?" conversation; (2) <strong>prioritize by cost of delay, not who asks</strong> —
value × urgency focuses on what matters, where prioritizing by seniority/volume optimizes noise and trains bad
behavior (even the CEO's request is prioritized on merits, handled diplomatically); (3) <strong>escape the
urgent-vs-important trap</strong> — deliberately protect important-but-not-urgent work (the real value) from
being crowded out by the endless urgent, via an interrupt budget that absorbs the unplanned while protecting
planned focus; (4) <strong>say no clearly, with the why, as "not now"</strong> — a reasoned, transparent
trade-off keeps requesters trusting you, where a vague or arbitrary no erodes trust. On reflection, people
commonly find their prioritization is opaque (case-by-case, not a visible list), that they sometimes prioritize
by who asks (especially caving to seniority), that the urgent crowds out the important (no protected capacity
for strategic work), and that their no's are sometimes vague. The highest-value change is usually: make
priorities visible (one ranked list), prioritize by cost of delay, and protect important work with an interrupt
budget. The meta-point: a lead says no all day (demand always exceeds capacity), so prioritization — deciding
what the team does and doesn't do — is core, and doing it well means making it visible (transparent trade-offs,
forcing real ordering), value-based (cost of delay, not who shouts), protecting the important from the urgent
(interrupt budget), and saying no clearly-and-kindly (staying trusted). And when you disagree with a priority
you must execute, disagree-and-commit. Prioritizing well focuses the team on what matters while keeping
requesters trusting you even when declined. It's central to project leadership and connects to negotiation
(Lesson 44) and saying no (English track Lesson 18). The next lesson addresses what happens when, despite good
planning and prioritization, a project slips anyway — recovering slipping projects with your credibility
intact.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 55 — When Projects Slip →](lesson-55-when-projects-slip){: .btn .btn-primary }
