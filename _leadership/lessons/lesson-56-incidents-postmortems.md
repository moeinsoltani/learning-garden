---
title: "Lesson 56 — Incidents and Blameless Postmortems"
nav_order: 7
parent: "Phase 10: Project Leadership"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 56: Incidents and Blameless Postmortems

{: .note }
> **Words to know** *(simple definitions for this lesson's jargon)*
> - **incident** — a live production emergency (outage, data risk); **severity** — its declared seriousness level, which drives the response.
> - **incident command(er)** — the person running the response: roles, decisions, communications — *not* debugging.
> - **communications lead** — the role that updates stakeholders on a regular **cadence** so responders aren't interrupted.
> - **postmortem** (post-MOR-tum) — the structured review after an incident (medical term: examination after death).
> - **blameless** — focused on the *systemic* factors that allowed the incident, never on punishing the person who triggered it.
> - **systemic** — belonging to the system (missing review, no rollback) rather than to an individual.
> - **root cause** — the underlying cause; "root cause = the person" is the anti-pattern.
> - **scapegoat** — a person unfairly given all the blame.
> - **rollback / staged rollout** — undoing a change fast / releasing gradually to limit damage.
> - **near miss** — something that *almost* caused an incident; mature teams report and learn from these.
> - **action items that ship** — postmortem fixes that are owned, tracked, and actually done — not filed and forgotten.

## Concept

When things break — the site's down, data's at risk — the team learns **who you really are as a leader.**
Incidents test you: leading during the fire (calm coordination, not panic) and after it (a **blameless
postmortem** that actually makes things better). The two core principles: during an incident, **the lead
coordinates, doesn't debug** (your job is command, not fixing); and after, **postmortems are blameless** —
focused on the systemic factors that let it happen, not on blaming a person, because blame kills the honesty
that prevents recurrence.

```
   INCIDENTS: DURING & AFTER
   ┌──────────────────────────────────────────────────────┐
   │ DURING (incident command):                             │
   │   • the LEAD coordinates, does NOT debug (stay above it)│
   │   • clear ROLES, comms cadence, honest severity        │
   │ AFTER (blameless postmortem):                          │
   │   • BLAMELESS: systemic factors, not "root cause = the  │
   │     person who pushed the button"                      │
   │   • action items that SHIP; normalize near-miss reporting│
   └──────────────────────────────────────────────────────┘
```

The reframe: **during an incident, lead (coordinate, communicate) rather than dive into debugging; after,
run a blameless postmortem focused on systemic fixes, not blame — because blame destroys the honesty needed
to actually prevent recurrence.** The instinct during a fire is to jump into fixing it yourself (you're a
good engineer); but the lead's job is coordination (roles, comms, decisions) — someone needs to be above the
firefight. And the instinct after is to find who caused it; but blame makes people hide information, so a
blameless focus on <em>why the system allowed it</em> is what actually improves things.

---

## How It Works

### Incident command — the lead coordinates, does NOT debug

During an incident, there's a strong pull for the lead to dive into debugging (you're a capable engineer,
and fixing feels productive). Resist it: **the lead's job is to coordinate (incident command), not to
debug.** Someone needs to be <em>above</em> the firefight — running the response: (1) establishing clear
**roles** (who's the incident commander, who's investigating, who's communicating — so it's not chaos); (2)
maintaining a **communications cadence** (regular updates to stakeholders — Lesson from English track 26 —
so people know what's happening without interrupting the responders); (3) making **decisions** (do we roll
back? escalate? call in more help?); and (4) keeping the response organized and calm. If the lead is
head-down debugging, no one is commanding (the response is chaotic, comms lapse, decisions don't get made).
So the lead stays out of the debugging and runs the incident — coordination is the higher-value role during
a fire, and it's what only the lead can do.

### Severity honesty and clear roles

Two incident-command basics: (1) **honest severity** — assess and declare the incident's severity honestly
(don't under-call it to avoid alarm, or over-call everything) — because severity drives the response (how
many people, how urgent, who's notified), and mis-calling it means the wrong response. (2) **clear roles** —
a defined **incident commander** (who runs the response and makes decisions), **investigators/responders**
(who debug), and a **communications lead** (who updates stakeholders) — so the response is organized, not a
scrum where everyone debugs and no one coordinates or communicates. Clear roles and honest severity turn a
chaotic scramble into an organized response.

### Blameless postmortems — systemic factors, not blaming a person

After the incident, the **postmortem** — and the crucial principle is that it's **blameless.** A blameless
postmortem focuses on <strong>the systemic factors that allowed the incident</strong> (why did the system
let this happen? what conditions, processes, gaps made it possible?), <em>not</em> on blaming the individual
who triggered it ("root cause: the engineer who pushed the bad config"). Why blameless: (1) **blame kills
honesty** — if people are blamed for incidents, they hide information, avoid reporting problems, and get
defensive — so you <em>lose the information</em> needed to understand and prevent recurrence (the opposite of
what a postmortem needs). A blameless environment makes people share openly (what really happened, what they
were thinking), which is what lets you actually learn. (2) **incidents are almost always systemic, not
individual** — when an engineer's config change takes down the site, the real question isn't "why did they
make the mistake?" (humans make mistakes — that's a given) but "why did the <em>system</em> let a single
config change take down the site?" (no review? no staged rollout? no automated check? no easy rollback?) —
the systemic factors that allowed a normal human error to become an incident. Fixing the person ("be more
careful") doesn't prevent recurrence (the next person will also err); fixing the <em>system</em> (add review,
staging, checks, rollback) does. So blameless postmortems focus on the systemic contributing factors (not
root-cause-the-individual) because that's both what people will be honest about (no blame) and what actually
prevents recurrence (fix the system, not scapegoat the person).

### Action items that ship, and near-miss reporting

A postmortem is only valuable if it **changes things** — so it must produce **action items that actually
ship** (specific, owned, tracked improvements — not a document that's filed and forgotten). The systemic
fixes (add the review gate, the staged rollout, the automated check, the better observability) must be
turned into real, owned, tracked work that gets done — otherwise the postmortem is theater and the same
incident recurs. And a mature incident culture **normalizes near-miss reporting** — encouraging people to
report the near-misses (things that almost caused an incident) and the small problems, because those are
free lessons (you learn without the full cost of an incident). Normalizing near-miss reporting (which
requires the same blamelessness) surfaces problems before they become incidents.

### Handling the "accountability" pressure

After a visible incident, leadership often wants **"accountability"** — which can mean pressure to blame/
punish the person responsible. A key leadership skill is <strong>redirecting that toward systemic
accountability</strong>: explain that blaming the individual is counterproductive (it kills the honesty
needed to prevent recurrence, and the real fault is systemic), and that true accountability is fixing the
<em>system</em> so it can't happen again (the owned action items). Protect your engineer from being
scapegoated (a config change taking down the site is a systemic failure — no review/staging/rollback — not
just the engineer's mistake), while genuinely holding the team accountable for the systemic fixes. This
protects your people (building deep trust and loyalty), maintains the blameless culture (so honesty
continues), and actually prevents recurrence (systemic fixes) — versus scapegoating, which destroys trust
and fixes nothing.

{: .note }
> **Lead the incident (coordinate, don't debug); run a blameless postmortem (systemic fixes, not blame)</br>**
> Incidents test who you are as a leader. <em>During</em>: the lead coordinates (incident command — roles,
> comms cadence, decisions, honest severity), does NOT debug (someone must be above the firefight; if the
> lead is head-down fixing, no one's commanding). <em>After</em>: run a <em>blameless</em> postmortem —
> focused on the systemic factors that let the incident happen (why did the system allow it?), not on blaming
> the person who triggered it — because blame kills the honesty needed to prevent recurrence, and incidents
> are almost always systemic (fix the system, not scapegoat the human, who will err again). Produce action
> items that <em>ship</em> (owned, tracked systemic fixes — not a filed-and-forgotten doc), normalize
> near-miss reporting, and redirect "accountability" pressure toward systemic fixes (protecting your
> engineer from scapegoating). Leading incidents well — calm command, blameless learning — is where teams
> learn to trust you.

---

## Lab — Scenario

**The situation:** One of your engineers, **Sam**, made a **configuration change** that took **the site
down for 2 hours** — a serious, visible incident. It's resolved now. Sam feels terrible. And leadership is
asking for **"accountability"** — there's pressure to hold someone responsible, and the implication is that
Sam is the one to blame. You need to run the postmortem and write your reply to leadership.

**Run the postmortem (its approach and key findings) and write your reply to leadership.** Then note the
principles and mistakes to avoid.

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
A strong response runs a blameless postmortem (systemic focus, protecting Sam) and redirects leadership's
"accountability" toward systemic fixes. Example:
<br><br>
<strong>The postmortem approach (blameless, systemic):</strong> "Run it blamelessly — the goal is to
understand why the <em>system</em> let a config change take down the site, not to blame Sam. Make it safe
(especially for Sam) so we get the honest picture. The key question isn't 'why did Sam make the change?'
(config changes are normal; humans err) but <strong>'why did our system allow a single config change to take
the whole site down for 2 hours?'</strong> — which surfaces the real, systemic contributing factors:" <br>
• Was there no <strong>review</strong> of the config change before it went out? <br>
• Was there no <strong>staged/canary rollout</strong> (so a bad change would've been caught on a small slice,
not the whole site)? <br>
• Was there no <strong>automated validation</strong> of the config? <br>
• Why did it take <strong>2 hours to recover</strong> — was rollback hard/slow (no easy rollback)? <br>
• Was there adequate <strong>observability</strong> to diagnose it fast? <br>
<em>(These systemic gaps — no review, no staging, no easy rollback — are the real causes: they let a normal
human error become a 2-hour outage.)</em>
<br><br>
<strong>The action items (systemic fixes that ship):</strong> "Owned, tracked fixes: add review for config
changes; add staged/canary rollout so a bad change hits a small slice first; add automated config validation;
make rollback fast and easy; improve observability. These prevent <em>any</em> engineer's config error from
becoming an outage — fixing the system, not Sam." <em>(Real, owned action items — the point of the postmortem.)</em>
<br><br>
<strong>The reply to leadership (redirect 'accountability' to systemic):</strong> "Hi [leadership] — I've run
the postmortem on the 2-hour outage, and I want to share what we found and how we're ensuring it doesn't
recur. <br>
<strong>What happened</strong>: a config change went out that took the site down; it took 2 hours to recover.
<br>
<strong>The real cause is systemic, not one person's mistake</strong>: the change was a normal kind of change
that any of us could have made — the real issue is that our system had no safeguards to catch it: no review
of config changes, no staged rollout (so it hit 100% immediately), and no fast rollback (which is why
recovery took 2 hours). A single config change should never be able to take down the whole site — that it
could is a systemic gap, which is on our engineering practices, not on the individual. <br>
<strong>True accountability = fixing the system</strong>: I want to be clear that blaming the engineer would
be counterproductive — it would make people hide problems (killing the honesty we need to prevent
recurrence), and it wouldn't fix anything (the next person could hit the same gap). Real accountability is
fixing the system so this <em>can't</em> happen again. Here's what we're doing [the owned action items:
review, staged rollout, validation, fast rollback, observability], with owners and dates. <br>
I take responsibility as the lead for the systemic gaps, and we're closing them. Happy to walk through the
details." <em>(Redirects 'accountability' from blaming Sam to fixing the system; protects Sam; the lead takes
systemic responsibility; concrete owned fixes.)</em>
<br><br>
<strong>And support Sam</strong>: "Privately, reassure Sam — this was a systemic failure, not their fault;
config changes are normal and the system should have caught it; they're not being blamed. Protecting Sam
here builds deep trust (and models the blameless culture for everyone watching)."
<br><br>
<strong>Principles:</strong> (1) <strong>Blameless — systemic, not the person</strong> — ask why the system
allowed it (no review/staging/rollback), not why Sam erred (humans err; that's a given). (2) <strong>Blame
kills honesty</strong> — blaming Sam would make everyone hide problems, losing the information needed to
prevent recurrence. (3) <strong>Fix the system, not the person</strong> — systemic fixes (review, staging,
rollback) prevent recurrence; "be more careful" doesn't. (4) <strong>Action items that ship</strong> — owned,
tracked systemic fixes (the point of the postmortem). (5) <strong>Redirect 'accountability' to systemic</strong>
— explain blaming is counterproductive; real accountability is fixing the system. (6) <strong>Protect your
engineer</strong> — Sam isn't scapegoated (systemic failure); this builds trust and models blamelessness.
(7) <strong>The lead takes responsibility</strong> for the systemic gaps. <strong>Mistakes to avoid:</strong>
(1) <strong>blaming/scapegoating Sam</strong> — caving to the 'accountability' pressure by hanging the
individual (destroys trust, kills honesty, fixes nothing — the next person hits the same gap); (2)
<strong>"root cause: human error"</strong> — the lazy, wrong conclusion that blames the person and misses the
systemic gaps; (3) <strong>no systemic fixes / "be more careful"</strong> — a non-fix that doesn't prevent
recurrence; (4) <strong>a postmortem that's filed and forgotten</strong> — no owned, shipped action items
(theater); (5) <strong>caving to leadership's blame framing</strong> — not redirecting 'accountability' to
systemic (a key leadership job — educate them); (6) <strong>not supporting Sam</strong> — leaving them feeling
blamed/terrible (demoralizing, and signals to everyone that mistakes get you blamed → people hide problems);
(7) <strong>defensiveness / hiding the incident</strong> — rather than transparent, honest, learning-focused.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Blameless postmortems | Google SRE book (postmortem culture); Etsy's blameless postmortems (John Allspaw) |
| Incident command | PagerDuty incident response docs; ICS (incident command system) |
| Human error is systemic | Sidney Dekker, *The Field Guide to Understanding Human Error* |
| Incident communication (the language) | English for Work track, Lesson 26 (announcements) |

---

## Checkpoint

**Q1.** Why should the lead "coordinate, not debug" during an incident, and why must postmortems be
blameless?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The lead should "coordinate, not debug" during an incident because <strong>someone needs to be above the
firefight running the response — establishing roles, maintaining communications, and making decisions — and
that coordination is the higher-value role only the lead can fill, whereas if the lead is head-down debugging,
no one is commanding and the response becomes chaotic</strong>. During an incident, there's a strong pull for
the lead to dive into debugging (you're a capable engineer, fixing feels productive and urgent). But the
lead's job is <strong>incident command</strong>, not fixing: (1) someone must establish and hold clear
<strong>roles</strong> (who's the incident commander, who's investigating, who's communicating) so the
response is organized rather than a chaotic scrum where everyone debugs and nothing is coordinated; (2)
someone must maintain the <strong>communications cadence</strong> (regular updates to stakeholders) so people
know what's happening without constantly interrupting the responders (and so the org isn't in the dark); (3)
someone must make <strong>decisions</strong> (do we roll back? escalate? call in more help? what's the
priority?) — the judgment calls that steer the response; and (4) someone must keep the response
<strong>organized and calm</strong> (preventing panic and thrash). These coordination functions are
essential and are what the <em>lead</em> is positioned to do (they require the overview and authority the lead
has). If the lead abandons this to debug alongside everyone else, <em>no one is commanding</em>: roles are
unclear (chaos), communications lapse (stakeholders in the dark, or responders interrupted), decisions don't
get made (or get made ad hoc), and the response is disorganized and slower. Meanwhile, the debugging itself
doesn't need the lead specifically — the responders can debug (often better than the lead, being closer to
the systems). So the lead coordinating (not debugging) is the higher-value allocation: the lead does the
command role only they can do (and that's essential), while the responders do the debugging (which they can
do). This is a specific instance of the lead's job being to enable and organize the team's work, not to do
the individual work themselves — and during an incident, the command role (coordination, comms, decisions)
is what the lead uniquely provides, so diving into debugging (however tempting) leaves that gap. Why
<strong>postmortems must be blameless</strong>: <strong>because blame kills the honesty needed to understand
and prevent recurrence, and incidents are almost always systemic (not individual) — so a blameless focus on
systemic factors is what actually improves things, while blaming the individual destroys the information and
fixes nothing</strong>. Two connected reasons: (1) <strong>blame kills honesty</strong> — if people are
blamed for incidents, they become defensive, hide information, and avoid reporting problems (to protect
themselves), so you <em>lose the honest picture</em> of what happened and why — which is exactly what a
postmortem needs to learn and prevent recurrence. A blameless environment (where the goal is understanding,
not punishment) makes people share openly (what really happened, what they were thinking, what they saw),
which is what enables genuine learning. So blame is self-defeating for a postmortem — it destroys the honesty
the postmortem depends on. (2) <strong>incidents are almost always systemic, not individual</strong> — when
an engineer's action triggers an incident (a config change takes down the site), the real question isn't "why
did the person make the mistake?" (humans make mistakes — that's a constant, unavoidable given) but "why did
the <em>system</em> let a normal human error become an incident?" (no review? no staged rollout? no automated
check? no fast rollback?) — the systemic conditions that allowed a routine error to cause damage. Blaming the
individual ("be more careful," "root cause: the engineer") misidentifies the cause and fails to prevent
recurrence: the person being more careful doesn't fix the systemic gap (the next person will also err, and
hit the same gap), whereas fixing the <em>system</em> (add review, staging, checks, rollback) prevents
<em>any</em> future error from becoming an incident. So the systemic focus is what actually prevents
recurrence, while person-blame fixes nothing. Combining these: postmortems are blameless because (a)
blamelessness gets the honesty needed to understand what happened (blame would make people hide it), and (b)
the systemic focus (why the system allowed it) targets the real, fixable causes (whereas blaming the
individual misses them). A blameless postmortem — focused on the systemic contributing factors, not
scapegoating the person — is thus what makes postmortems actually work: people are honest (so you learn), and
you fix the system (so it doesn't recur). Blame breaks both (people hide, and you fix the wrong thing). Both
points reflect leading incidents well: coordinate during (the command role only the lead provides, versus
diving into debugging and leaving no one in charge), and run blameless postmortems after (systemic focus and
psychological safety, versus blame that destroys honesty and fixes nothing) — which is what makes incident
response organized and incident learning effective, and, crucially, is where the team learns whether you're a
leader who commands calmly and protects/learns, or one who panics and scapegoats.
</details>

**Q2.** Why does blaming the individual after an incident fix nothing, and how should a lead handle
leadership's demand for "accountability"?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Blaming the individual after an incident fixes nothing because <strong>incidents are systemic — a human
error becomes an incident only because the system allowed it — so "fixing" the person (blame, "be more
careful") doesn't address the systemic gap that let the error cause damage, meaning the next person will hit
the same gap and the incident recurs</strong>. When an engineer's action triggers an incident (a config
change takes down the site), the person's error is usually a <em>normal, expected human mistake</em> — humans
make mistakes; that's an unavoidable constant, not something you can eliminate by blaming or exhorting
people. The reason the mistake became a <em>2-hour site outage</em> (rather than a caught, harmless error) is
<strong>systemic</strong>: there was no review to catch the change, no staged/canary rollout to limit its
blast radius, no automated validation, no fast rollback. These systemic gaps are what turned a routine human
error into an incident. So blaming the individual ("be more careful," "root cause: the engineer who pushed
the button") fixes nothing, because: (1) it doesn't address the systemic gaps — the lack of review, staging,
rollback remain, so the system is just as vulnerable; (2) "be more careful" doesn't work — the person (and
everyone else) will still make mistakes eventually (humans do), and the next mistake will hit the same
unguarded gap and cause the same incident; (3) it targets the wrong cause — the mistake was the trigger, but
the <em>cause</em> of the <em>incident</em> was the systemic vulnerability that let the trigger cause damage.
Only fixing the <em>system</em> (adding review, staging, checks, fast rollback) prevents recurrence — because
then <em>any</em> future error (which will happen) gets caught or contained before it becomes an incident. So
blame fixes nothing (the system stays vulnerable, the next error recurs the incident), while systemic fixes
actually prevent recurrence — which is why blameless-systemic is the right focus. (Plus, blame destroys the
honesty needed to even understand the incident — Q1 — and demoralizes/scapegoats a person for a systemic
failure.) How a lead should handle leadership's demand for <strong>"accountability"</strong>: <strong>redirect
it from blaming/punishing the individual toward systemic accountability — fixing the system — while protecting
the engineer from scapegoating and genuinely holding the team accountable for the fixes</strong>. After a
visible incident, leadership often wants "accountability," which can carry an implication of blaming or
punishing whoever's responsible. The lead's job is to <em>redirect</em> this: (1) <strong>explain that
blaming the individual is counterproductive</strong> — it kills the honesty needed to prevent recurrence (Q1:
people hide problems if blamed), and it doesn't fix anything (the systemic gap remains, the incident can
recur); so scapegoating actively harms the goal (preventing recurrence) that "accountability" presumably
serves. (2) <strong>reframe true accountability as fixing the system</strong> — real accountability is
ensuring it <em>can't happen again</em>, which means closing the systemic gaps (the owned, tracked action
items: review, staging, rollback, observability) — that's the meaningful accountability (the org is genuinely
held responsible for preventing recurrence), versus the hollow, harmful accountability of punishing a person.
(3) <strong>protect the engineer from scapegoating</strong> — make clear (to leadership and the engineer)
that a config change taking down the site is a <em>systemic</em> failure (no review/staging/rollback), not
just the engineer's fault — the engineer made a normal mistake that the system should have caught. Protecting
them is right (it wasn't their fault systemically), and it builds deep trust and loyalty (the team sees the
lead has their back) and maintains the blameless culture (so honesty continues — if the engineer is
scapegoated, everyone learns that mistakes get you blamed, so they'll hide problems). (4) <strong>the lead
takes responsibility</strong> for the systemic gaps (as the lead, the practices are your responsibility) and
commits to the fixes. So the lead handles the "accountability" demand by educating leadership that
scapegoating is counterproductive and real accountability is systemic fixing, protecting the engineer,
and demonstrating genuine accountability through the owned systemic action items. This satisfies the
legitimate need behind "accountability" (ensuring it doesn't recur, taking it seriously) in the way that
actually works (systemic fixes) rather than the way that fails and harms (scapegoating) — while protecting
the people and the blameless culture. Both points reflect the blameless-systemic principle: blame fixes
nothing (the incident is systemic, so only systemic fixes prevent recurrence) and destroys trust/honesty, so
a lead redirects "accountability" from punishing the person to fixing the system — protecting the engineer,
maintaining the honesty that prevents recurrence, and delivering the real accountability (systemic fixes that
ship). This is a defining leadership moment: how you handle the blame/accountability pressure after an
incident is where the team learns whether you protect them and fix systems, or scapegoat them under
pressure — which profoundly shapes trust.
</details>

---

## Homework

Reflect on how you lead incidents and postmortems. (1) During an incident, do you coordinate (roles, comms,
decisions) or dive into debugging? Practice staying in the command role. (2) Are your postmortems blameless
(systemic factors — "why did the system allow it?") or do they land on "human error / who's to blame"? (3) Do
your postmortems produce action items that actually ship (owned, tracked systemic fixes), or documents that
get filed and forgotten? (4) When there's pressure to blame someone, do you redirect it to systemic
accountability and protect your engineer? Reflect: do you lead incidents with calm command and blameless
learning, and what's one thing (command discipline, blamelessness, shipping fixes, protecting people) to
improve?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds incident leadership — leading during fires and running blameless postmortems, where teams
learn who a leader really is. A strong response checks whether they coordinate vs. debug during incidents,
whether postmortems are blameless (systemic) vs. blame-focused, whether action items actually ship, and
whether they redirect blame pressure toward systemic accountability while protecting their engineers. The
realizations: (1) <strong>the lead coordinates, doesn't debug</strong> — someone must be above the firefight
(roles, comms cadence, decisions, honest severity); if the lead is head-down debugging, no one's commanding
and the response is chaos; (2) <strong>postmortems must be blameless</strong> — blame kills the honesty needed
to learn (people hide problems), and incidents are systemic (a human error becomes an incident because the
system allowed it), so focus on "why did the system let this happen?" (no review/staging/rollback), not "who's
to blame"; (3) <strong>blame fixes nothing</strong> — "be more careful" doesn't address the systemic gap, so
the next person hits it and the incident recurs; only systemic fixes prevent recurrence; (4) <strong>action
items must ship</strong> — owned, tracked systemic fixes (not a filed-and-forgotten doc), or the postmortem is
theater; (5) <strong>redirect "accountability" to systemic and protect your engineer</strong> — explain
scapegoating is counterproductive (kills honesty, fixes nothing), real accountability is fixing the system,
and protecting the engineer (a systemic failure, not their fault) builds trust and maintains the blameless
culture. On reflection, people commonly find they dive into debugging during incidents (leaving no one
commanding), that their postmortems sometimes land on human error (rather than systemic factors), that action
items don't always ship, and that they feel pressure to satisfy blame demands. The highest-value change is
usually: stay in the command role during incidents, keep postmortems rigorously blameless-systemic (with
shipping action items), and redirect blame pressure toward systemic fixes while protecting people. The
meta-point: incidents test who you are as a leader — during (calm command: coordinate, don't debug — roles,
comms, decisions) and after (blameless postmortems: systemic factors not blame, because blame kills honesty
and incidents are systemic; action items that ship; protect your engineer from scapegoating and redirect
"accountability" to fixing the system). Leading incidents well — calm command and blameless learning — is
where teams learn to trust you (you command calmly under fire and protect/learn rather than panic and
scapegoat), which profoundly shapes the culture and their loyalty. It connects to blameless culture (Lesson
63 psychological safety), incident communication (English Lesson 26), and the systemic-thinking of good
engineering. This completes Phase 10 (Project Leadership): planning, estimation, risk, dependencies,
prioritization, slip recovery, and incidents — the craft of delivering projects and leading through the
crises that test a leader. Delivering reliably and leading through fires is much of what a lead is judged on
and where trust is built. The next phase (People Management) turns to the EM path specifically — the people-
management responsibilities (performance, hiring, motivation, org design, psychological safety) that come
with managing people, not just leading projects.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 11 — People Management (Lesson 57: Performance Management) →](lesson-57-performance-management){: .btn .btn-primary }
