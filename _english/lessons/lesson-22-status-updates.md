---
title: "Lesson 22 — Status Updates"
nav_order: 2
parent: "Phase 4: Slack Communication"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 22: Status Updates

## Concept

A status update tells people where your work stands — and a good one *answers their
questions before they ask them*. The classic shape is **done / doing / blocked**, and
the key skill (beyond the shape) is being **specific** rather than vague, so the reader
knows exactly where things are.

```
   VAGUE (raises more questions)      SPECIFIC (answers them)
   ────────────────────────────      ───────────────────────
   "Making progress on the API."      "API: 3 of 5 endpoints done. Working on
                                       the last two, should finish tomorrow."

   "Almost done."                     "Done except for the tests — ~2 hours left."

   "Working on the bug."              "Fixed the root cause; now verifying it
                                       doesn't break the payment flow."
```

Vague updates ("making progress," "almost done," "working on it") force the reader to
ask follow-up questions — how much progress? how close? what's the ETA? A specific
update answers those in advance ("3 of 5 done, ETA tomorrow"), which is more useful and
more professional. This uses the verb tenses from Lesson 4 (done = present perfect,
doing = present continuous) doing real work.

---

## How It Works

### The done / doing / blocked shape

The reliable structure for a status update:

- **Done**: what you've completed (present perfect — "I've finished X"). Concrete
  accomplishments.
- **Doing**: what you're working on now, with an ETA if you can (present continuous —
  "I'm working on Y, should be done tomorrow").
- **Blocked**: anything stopping you, and what you need to unblock it (present simple —
  "I'm blocked on Z — I need access to the staging DB").

Not every update needs all three, but this shape ensures you cover what people want to
know: progress (done), current focus + timeline (doing), and any problems + what you
need (blocked).

### Be specific — numbers and concretes

The difference between a weak and a strong update is specificity:

- ✗ "making progress" → ✓ "3 of 5 tests passing now"
- ✗ "almost done" → ✓ "done except the docs — ~1 hour left"
- ✗ "working on the migration" → ✓ "migration is ~60% through; the last tables are the
  tricky ones"

Specifics (numbers, percentages, concrete milestones, ETAs) tell the reader exactly
where things are and answer their implicit questions. Vagueness forces follow-ups and
can hide problems. When you can quantify, do.

### Flag risk early, without alarm

If something might slip or go wrong, say so *early* and *calmly* — don't hide it until
it's a crisis, and don't over-alarm:

- ✓ "Heads up: the migration is trickier than expected — still aiming for Friday, but
  there's some risk it slips to Monday. I'll know more by Wednesday." (Early, honest
  about risk, calm, with a follow-up point.)

Flagging risk early lets people plan; hiding it until the deadline is a nasty surprise
(and erodes trust). But frame it calmly ("some risk," "I'll know more by X") — not as a
panic. This connects to the leadership track's "no surprises" principle.

### The weekly-summary variant

A weekly update is a bigger done/doing/blocked, often to a wider audience (your
manager, the team). Same principles, more roll-up: what shipped this week (done),
what's the focus next week (doing), any risks or needs (blocked), and ideally the
*impact* of what was done (not just "did X" but "did X, which enables Y"). Keep it
scannable (bullets) and specific.

{: .note }
> **A good status update answers questions before they're asked</br>**
> The test of a status update: after reading it, does the reader still have obvious
> questions (how much? how close? when? what's wrong?)? A vague update ("making
> progress") leaves all of those unanswered, forcing a back-and-forth. A specific
> done/doing/blocked update ("3 of 5 done, finishing tomorrow, blocked on X which I
> need Y for") answers them in advance — which is more useful, more professional, and
> saves everyone the follow-up questions. Specificity and the done/doing/blocked shape
> are what make an update genuinely informative rather than just a check-in.

---

## Lab — Rewrite Drill

Part A: turn these vague updates into specific ones (invent plausible specifics).
Part B: write a done/doing/blocked update and a weekly summary for a given situation.

**Your answer:**

*Part A — make specific:*
1. "Making good progress on the payment feature."
2. "Almost done with the migration."
3. "Still working on the bug."

*Part B — write these:*
4. A daily update for: you finished the API endpoints, you're writing tests now (should be done tomorrow), and you're blocked because you need database access from the infra team.
5. A one-line risk flag for: the migration is harder than expected and might slip from Friday to Monday.

<details>
<summary>Show Model Answers</summary>
<br>
<strong>Part A — specific:</strong>
<br>1. "Payment feature: the core flow is done and working in staging. Now adding the
error handling and edge cases — should be ready for review by tomorrow."
<br>2. "Migration: ~80% done — the last few tables have some tricky foreign keys, so
I'm being careful. Aiming to finish today, tomorrow morning at the latest."
<br>3. "Bug update: I've found the root cause (a race condition in the token refresh).
Now working on the fix — should have a PR up this afternoon."
<br><br>
<strong>Part B:</strong>
<br>4. Daily update: <strong>"Daily update on the [feature]:
<br>✅ Done: all 5 API endpoints are built and passing manual tests.
<br>🔨 Doing: writing the automated tests now — should be done tomorrow.
<br>🚧 Blocked: I need read access to the staging database to finish testing — could
someone on infra help me get that? 🙏"</strong>
<br>(Done/doing/blocked, specific, with the blocker clearly stated AND what's needed
to unblock — so someone can act on it.)
<br><br>
<br>5. Risk flag: <strong>"Heads up on the migration: it's turning out trickier than
expected (some messy foreign-key relationships). I'm still aiming for Friday, but
there's a real chance it slips to Monday. I'll have a clearer picture by Wednesday and
will update then."</strong> (Early, honest about the risk, calm (not a panic), and
gives a follow-up point so people know when they'll hear more — the "no surprises"
principle done warmly.)
<br><br>
The patterns: specificity (numbers, concrete milestones, ETAs) answers the reader's
questions in advance; the done/doing/blocked shape covers what they want to know; and
risk flags come early and calm, with a next-update point.
</details>

---

## Phrase Bank

| Element | Phrase |
|---|---|
| Done | "✅ Done: [specific accomplishment]" / "I've finished X" |
| Doing | "🔨 Doing: [X] — should be done [ETA]" |
| Blocked | "🚧 Blocked on [X] — I need [Y] to unblock" |
| Specific progress | "3 of 5 done" / "~80% through" / "done except [X]" |
| ETA | "should be ready by [when]" / "~2 hours left" |
| Risk flag | "Heads up: [X] is trickier than expected — some risk it slips to [when]." |
| Next update | "I'll know more by [when] and will update then." |

---

## Further Reading

| Topic | Source |
|---|---|
| Verb tenses (done/doing/fact) | English track Lesson 4 |
| Status communication & no-surprises | Leadership track Lessons 41 (managing up), 55 (when projects slip) |
| Specific vs vague reporting | any project-communication guide |

---

## Checkpoint

**Q1.** Why is a specific status update ("3 of 5 done, ETA tomorrow") better than a
vague one ("making progress"), and how does the done/doing/blocked shape help?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A specific status update is better because it <strong>answers the reader's questions in
advance, rather than forcing them to ask follow-ups</strong>. When someone reads a
status update, they have implicit questions: how much progress exactly? how close to
done? when will it be finished? is anything wrong? A vague update ("making progress,"
"almost done," "working on it") answers none of these — it tells the reader that work is
happening but nothing about <em>where it actually stands</em>, so they either have to
ask follow-up questions (a back-and-forth that wastes both people's time) or they're
left uncertain (which is worse — they can't plan, and "making progress" could mean 10%
or 90%). A specific update ("3 of 5 endpoints done, finishing the last two tomorrow")
answers the questions directly: the reader knows exactly how far along you are (3 of 5),
what's left (2 endpoints), and when it'll be done (tomorrow) — no follow-up needed, they
can plan around it, and it's obvious the work is genuinely on track (or, if there's a
problem, that's visible too). Specificity — numbers, percentages, concrete milestones,
ETAs — is what makes an update <em>informative</em> rather than just a check-in; it
transforms "something is happening" into "here's exactly where it stands." It's also
more professional (it shows you know precisely where your work is and can communicate it
clearly) and more honest (vagueness can hide problems — "making progress" might mask
being stuck; specifics surface reality). How the <strong>done/doing/blocked shape</strong>
helps: it ensures you cover the three things people want to know about your work, in a
clear structure. <strong>Done</strong> (what you've completed) shows concrete progress —
the accomplishments, so people see what's actually finished. <strong>Doing</strong>
(what you're working on now, with an ETA) shows the current focus and timeline — so
people know what's happening and when to expect it. <strong>Blocked</strong> (anything
stopping you and what you need) surfaces problems and — crucially — <em>what you need to
unblock</em>, so someone can actually help (a blocker stated without the ask for help is
just a complaint; "blocked on X, I need Y" is actionable). Together, the shape covers
progress, timeline, and problems — the complete picture of where your work stands — in a
structure the reader can quickly parse. Combined with specificity (concrete details in
each part), the done/doing/blocked update answers the reader's questions before they ask
them, which is the mark of a genuinely useful update. This matters because status
updates are frequent (daily standups, weekly reports, ad-hoc "where are we on X"), and a
specific, well-shaped update saves everyone the follow-up questions, surfaces problems
early (blocked), lets people plan (ETAs), and makes you look on-top-of-your-work — while
a vague update forces back-and-forth, hides issues, and looks unclear. The habit:
structure updates as done/doing/blocked, and make each part specific (numbers, ETAs,
what-you-need) — so the update actually informs rather than just checking in.
</details>

**Q2.** Why should you flag risk early rather than hiding it until the deadline, and
how do you flag it without causing alarm?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
You should flag risk early because <strong>early warning lets people plan and adjust,
while hiding it until the deadline creates a nasty surprise that's worse for everyone
and erodes trust</strong>. If a deadline might slip or something might go wrong, and you
flag it early ("there's some risk the migration slips from Friday to Monday"), the
people depending on it can react while there's still time — re-plan the release, adjust
dependent work, set expectations with stakeholders, or help you mitigate the risk. If
instead you hide the risk and only reveal it at the deadline ("the migration isn't done
and won't be until Monday" — announced Friday afternoon when everyone expected it done),
it's a surprise with no time to react: the release plan is blown, dependent work is
stuck, stakeholders are blindsided, and the cost of the slip is much higher because
nobody could prepare for it. Worse, hiding it erodes trust — people learn they can't
rely on your updates to surface problems, so they can't trust your "on track" (it might
be hiding a slip), which makes all your communication less useful and damages your
credibility. This is the "no surprises" principle (from the leadership track, Lesson 41):
the people you work with can handle bad news (a slip, a problem) far better when it comes
early with time to react than when it's sprung on them at the last moment — surprises are
what damage trust and cause the most disruption, so surfacing problems early (even when
uncertain) is a core professional habit. How to flag risk without causing alarm: (1)
<strong>Frame it calmly and proportionally</strong> — "there's some risk it slips" or "a
chance it moves to Monday," not "everything is falling apart" or panic; match the alarm
level to the actual risk (a possible one-day slip isn't a crisis). (2) <strong>Be honest
about the uncertainty</strong> — "I'm still aiming for Friday, but there's real risk it
slips" acknowledges it might be fine while flagging the risk, rather than either
false-confidence ("definitely Friday") or doom ("it's definitely going to slip"). (3)
<strong>Give a next-update point</strong> — "I'll have a clearer picture by Wednesday
and will update then" — which reassures people that you're on top of it and they'll know
more soon, so they don't have to worry or chase; it turns an open-ended worry into a
managed situation with a check-in. (4) <strong>Where possible, note what you're doing
about it or what would help</strong> — showing it's being managed, not just reported. So
a good risk flag: "Heads up: the migration is trickier than expected — still aiming for
Friday, but some risk it slips to Monday. I'll know more by Wednesday and will update
then." — early (gives time to react), honest (real risk, but might be fine), calm (not
alarmist), and managed (next-update point). This gives people the information they need
to plan without spreading panic, and it builds trust (they know your updates surface
problems honestly and early). The balance is the key: flag risk early and honestly
(don't hide it — that's the bigger failure), but calmly and with a plan/follow-up (don't
over-alarm) — informative and reassuring rather than either falsely-confident or
panicked. For your goal, this is also a warmth-and-professionalism point: surfacing risk
early and calmly is considerate (it helps others) and builds trust, which strengthens
working relationships — whereas surprises damage them.
</details>

---

## Homework

Write your next status update (daily or weekly) using done/doing/blocked, and make every
part specific (numbers, concrete milestones, ETAs — no "making progress" or "almost
done"). If there's any risk of a slip or problem, flag it early and calmly, with a
next-update point. For the blocked part, state clearly what you need to unblock (so
someone can act). Notice whether the specific update gets fewer follow-up questions. Add
to your checklist: "status update — done/doing/blocked, specific (numbers/ETAs), risk
flagged early, blocker states what I need?"

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the status-update skill — a frequent and professionally-important
communication. A strong response uses done/doing/blocked with genuine specificity
(replacing "making progress" with "3 of 5 done, ETA tomorrow"-style concretes) and, if
relevant, flags risk early and calmly with a next-update point. The common realization:
<strong>your updates may have been vaguer than useful</strong> — "making progress,"
"almost done," "working on it" feel like updates but leave the reader's real questions
(how much? when? what's wrong?) unanswered, forcing follow-ups; specific updates answer
those in advance and get fewer follow-up questions (a sign they're actually
informative). The done/doing/blocked shape ensures completeness (progress, timeline,
problems), and stating what you need for the blocker makes it actionable (someone can
help). The risk-flagging habit (early, calm, with a follow-up point) is a professional
and trust-building move — surfacing potential problems early lets people plan and shows
you're on top of your work, whereas hiding them until the deadline creates surprises
that damage trust. The meta-point: status updates are a frequent, visible communication
(standups, weekly reports, "where are we on X"), and doing them well — specific,
well-shaped, honest about risk — makes you look on top of your work, saves everyone the
follow-up back-and-forth, surfaces problems early, and builds trust; doing them poorly
(vague, incomplete, hiding risk) forces follow-ups, hides issues, and looks unclear. So
the status-update skill is high-value for how effective and trustworthy you appear.
This lesson also puts several earlier skills to work: the verb tenses (Lesson 4 —
done=present perfect, doing=present continuous), lead-with-the-point/specificity (Lesson
21), and warm framing (Phase 3 — the blocker's ask is a warm request, the risk flag is
calm and considerate). Add the status-update checklist item. The next lessons continue
the Slack phase with more specific situations — asking for help well (make helping you
easy), answering and unblocking others, async etiquette, and announcements — each a
common Slack communication where structure, specificity, and warmth combine.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 23 — Asking for Help →](lesson-23-asking-for-help){: .btn .btn-primary }
