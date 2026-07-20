---
title: "Lesson 26 — Announcements to Wide Channels"
nav_order: 6
parent: "Phase 4: Slack Communication"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 26: Announcements to Wide Channels

## Concept

Sometimes you have to write a message many people will read — a deploy notice, an
incident update, a change that affects a whole team. These high-visibility messages need
a clear structure so busy readers instantly know what's happening and what (if anything)
they need to do. And the tone must match the situation — calm for a routine change,
appropriately serious for an incident.

```
   THE ANNOUNCEMENT SHAPE:
   ┌──────────────────────────────────────────────┐
   │ WHAT is changing/happening (the headline)     │
   │ WHEN (timing)                                  │
   │ WHAT YOU NEED TO DO (action needed, if any)   │
   │ WHERE to ask / get more info                   │
   └──────────────────────────────────────────────┘

   Match the TONE to severity:
   routine maintenance ≠ live incident ≠ all-clear
```

The reader of a wide announcement is scanning — they need to know instantly: what's
happening, when, whether it affects them / what to do, and where to ask. Lead with the
what, be clear about action needed, and match the tone to how serious it is (a
maintenance notice is calm; an incident is focused and serious but not panicked).

---

## Going Deeper

### The announcement structure

For any wide announcement:

- **What**: the headline — what's changing or happening, stated clearly first ("🚀
  Deploying v2.4 to production").
- **When**: the timing ("at 3pm today" / "starting now" / "over the next hour").
- **What you need to do**: the action required, if any ("no action needed" / "please
  hold deploys until it's done" / "restart your local env after"). Be explicit — people
  need to know if this affects them.
- **Where to ask**: where to go for questions or more info ("questions in this thread"
  / "updates in #incidents").

Lead with the what (Lesson 21), keep it scannable (short, maybe bulleted), and make the
action-needed unmistakable.

### Match tone to severity

The tone should fit how serious the situation is:

- **Routine** (planned maintenance, a normal deploy): calm, matter-of-fact, even
  friendly. "🔧 Quick heads up: deploying the new billing service at 3pm — brief
  ~2-min blip possible, no action needed. Questions here!"
- **Incident** (something is broken, live): serious and focused, but *not* panicked — a
  calm, clear incident update reassures people that it's being handled. "🔴 Investigating:
  the payment API is returning errors as of 2:15pm. We're on it — updates every 15 min
  in this thread. Please hold payment-related deploys."
- **All-clear** (resolved): clear resolution + brief cause. "✅ Resolved: the payment
  API is back to normal as of 2:45pm. Cause was a bad config; we've reverted it and are
  monitoring. Thanks for your patience!"

Over-alarming a routine change wastes attention and creates false panic; under-playing a
real incident leaves people uninformed. Match the tone to reality.

### The incident update rhythm

During an incident, provide *regular updates* — even "no new update yet" is an update.
People watching an incident want to know it's being handled and what's happening; regular
updates ("still investigating, next update in 15 min") reassure them and prevent a flood
of "any update?" questions. State the cadence ("updates every 15 min") and keep to it.
Silence during an incident is worse than "still working on it."

### Pre-write and the all-clear

For planned announcements (a big deploy, a maintenance window), pre-write the message and
maybe have someone review it (a wide announcement is high-visibility — worth getting
right). And always send the **all-clear / follow-up** — after an incident or a change,
confirm it's resolved/done, with the cause if relevant. People who saw the problem need
to see the resolution; leaving them hanging (no all-clear) leaves them uncertain whether
it's still ongoing.

{: .note }
> **Clarity and the right tone are what wide announcements need</br>**
> A wide announcement is read by many busy, scanning people, so it must be instantly
> clear: what's happening, when, what they need to do, where to ask. And the tone must
> match severity — calm for routine, serious-but-not-panicked for incidents, clear for
> the all-clear. The two failure modes: (1) unclear structure (people can't tell what's
> happening or if they need to act) and (2) mismatched tone (panicking over a routine
> change, or being too casual about a real incident). Get the structure clear and the
> tone right, and your announcements inform and reassure rather than confuse or alarm —
> which is especially important because these are high-visibility (many people see how
> you communicate).

---

## Lab — Rewrite Drill

Write a clear, appropriately-toned announcement for each of these three situations.

**Your answer (write each announcement):**

1. **Planned maintenance:** You're deploying a new version of the billing service at 3pm today. There may be a brief (~2 min) disruption to billing. No action needed from others.
2. **A live incident:** It's 2:15pm and you've just discovered the payment API is returning errors. You're investigating. You want people to hold payment-related deploys, and you'll update regularly.
3. **The all-clear:** It's 2:45pm and you've resolved the incident from #2 — the cause was a bad config change, which you've reverted, and you're monitoring.

<details>
<summary>Show Model Answers</summary>
<br>
<strong>1. Planned maintenance (calm, matter-of-fact):</strong> "🔧 <strong>Heads up:
deploying billing service v2.4 at 3pm today.</strong> There may be a brief (~2 min)
disruption to billing during the deploy. No action needed on your end — just a heads up
in case you see a blip. I'll confirm when it's done. Questions? Drop them in this thread!"
(What (billing deploy) + when (3pm) + impact (~2 min blip) + action (none) + where (this
thread) + a friendly, calm tone appropriate to routine maintenance. Bolded the headline
for scanning.)
<br><br>
<strong>2. Live incident (serious, focused, not panicked):</strong> "🔴
<strong>Investigating: the payment API is returning errors as of 2:15pm.</strong> We're
on it and digging into the cause now. <strong>Please hold any payment-related deploys</strong>
until we give the all-clear. I'll post updates here every 15 minutes. Thanks for your
patience." (What (payment API errors) + when (2:15pm) + status (investigating) + action
needed (hold payment deploys — bolded, so it's unmissable) + update cadence (every 15
min) + where (here). Serious and clear, but calm — "we're on it," regular updates —
which reassures people it's being handled rather than spreading panic.)
<br><br>
<strong>3. All-clear (clear resolution + cause):</strong> "✅ <strong>Resolved: the
payment API is back to normal as of 2:45pm.</strong> Cause was a bad config change,
which we've reverted. We're monitoring to make sure it stays healthy, but you're clear to
resume payment deploys. Thanks so much for your patience, everyone! 🙏" (Clear resolution
(back to normal) + when (2:45pm) + brief cause (bad config, reverted) + status
(monitoring) + action (clear to resume) + warm thanks. Confirms it's over so people who
saw the incident know the resolution — never leave them hanging.)
<br><br>
The patterns: each leads with the what (bolded headline for scanning), states when,
makes the action-needed unmistakable (especially the incident's "hold deploys"), says
where to ask/get updates, and — crucially — matches the tone to severity (calm/friendly
for routine, serious-but-calm for the incident, clear-and-warm for the all-clear). And
the incident has an update cadence; the all-clear confirms resolution.
</details>

---

## Phrase Bank

| Element | Phrase |
|---|---|
| Routine headline | "🔧 Heads up: [what] at [when]." |
| Incident headline | "🔴 Investigating: [what] as of [when]." |
| All-clear headline | "✅ Resolved: [what] back to normal as of [when]." |
| Impact | "brief (~X min) disruption to [Y]" |
| Action needed | "No action needed" / "Please hold [X] until [Y]" (bold it) |
| Update cadence | "Updates every [X] min in this thread." |
| Where to ask | "Questions? Drop them here." |
| Cause (all-clear) | "Cause was [X]; we've [fixed it] and are monitoring." |

---

## Further Reading

| Topic | Source |
|---|---|
| Incident communication | Leadership track Lesson 56 (incidents & postmortems) |
| Announcement structure (lead with what) | English track Lesson 21 (message shapes) |
| Tone by severity | English track Lesson 14 (register); Lesson 8 (tone) |

---

## Checkpoint

**Q1.** What should a wide announcement include, and why does leading with "what's
happening" matter for a high-visibility message?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
A wide announcement should include: (1) <strong>What</strong> — the headline: what's
changing or happening, stated clearly first ("Deploying v2.4 to production," "Payment API
returning errors"); (2) <strong>When</strong> — the timing ("at 3pm," "as of 2:15pm,"
"over the next hour"); (3) <strong>What you need to do</strong> — the action required, if
any ("no action needed," "please hold deploys," "restart your env after") — made explicit
and unmistakable, so people know whether and how it affects them; (4) <strong>Where to
ask/get updates</strong> — where to go for questions or more info ("questions in this
thread," "updates in #incidents"). This structure gives the many busy readers everything
they need: what's happening, when, whether/how it affects them, and where to follow up.
Why leading with "what's happening" matters especially for a high-visibility message:
<strong>the reader is one of many people scanning, and the first line determines whether
they instantly grasp the message or have to work for it</strong>. A wide announcement is
read by lots of people, most of whom are busy and skimming — they see the message, and in
the first line they need to know immediately "what is this about, and does it affect me?"
If you lead with the what ("🔴 Investigating: payment API returning errors as of 2:15pm"),
every reader instantly knows what's happening and can decide if they need to read more or
act; if you bury the what under preamble or context, the many readers each have to dig for
the point, multiplying the wasted effort across everyone (a buried point in a message to
one person wastes one person's time; in a wide announcement, it wastes many people's). So
leading with the what is even more important for wide announcements than for individual
messages, because the cost of a buried point is multiplied by the audience size, and
because high-visibility messages are exactly where clarity matters most (many people see
how you communicate, and an unclear announcement about something important — an incident,
a change — causes widespread confusion). Leading with the what (plus the clear structure)
ensures the announcement does its job: informing many people quickly and clearly about
what's happening and what they need to do. The bolded headline (for scanning) and the
scannable structure reinforce this — a wide announcement should be graspable at a glance,
because that's how it's read. Combined with matching the tone to severity (Q2), the clear
structure makes wide announcements inform and reassure rather than confuse — which is
important both for the immediate purpose (people need to know) and for how you're
perceived (high-visibility communication shapes your reputation).
</details>

**Q2.** Why must an announcement's tone match the severity of the situation, and what
goes wrong if it doesn't (in both directions)?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
An announcement's tone must match severity because <strong>the tone itself communicates
how seriously to take the situation, and a mismatch either misleads people or undermines
the message</strong>. Readers take cues from tone: a calm, matter-of-fact tone signals
"this is routine, no big deal"; a serious, focused tone signals "this matters, pay
attention"; a panicked tone signals "emergency." So the tone needs to accurately convey
the actual severity, or it miscommunicates. What goes wrong in both directions: (1)
<strong>Over-alarming a routine situation</strong> — using an urgent, panicked, or
heavily-serious tone (or @channel, red alerts) for a routine change (a normal deploy,
planned maintenance) creates false alarm: people drop what they're doing thinking
something's wrong, get anxious, and waste attention on a non-event — and it erodes your
credibility (if you sound the alarm for routine things, people learn to discount your
alarms, so real emergencies get less attention — crying wolf, like @channel misuse). It
also just makes you look like you can't calibrate — over-reacting to normal things. (2)
<strong>Under-playing a real incident</strong> — using a too-casual, breezy tone for a
genuine incident (the payment system is down, but you post "hey, payments are acting a
bit weird 🙂") under-communicates the seriousness: people don't realize it's a real
problem, don't take the needed action (hold deploys, escalate), and are caught off guard
by the impact — the casual tone failed to convey "this is serious, act accordingly." It
can also seem tone-deaf or unprofessional (being flippant about a real problem). So both
mismatches fail: over-alarming causes false panic and erodes credibility; under-playing
leaves people uninformed and unprepared, and seems unserious. The right tone matches
reality: <strong>calm and matter-of-fact for routine</strong> (a planned deploy: "heads
up, deploying at 3pm, brief blip possible, no action needed" — friendly and low-key,
appropriate to a non-event); <strong>serious and focused but NOT panicked for an
incident</strong> (a live outage: "🔴 Investigating: payment API errors as of 2:15pm,
we're on it, hold deploys, updates every 15 min" — conveys real seriousness and the
needed action, while the calm "we're on it" and regular updates reassure people it's
being handled rather than spreading panic); <strong>clear and warm for the all-clear</strong>
("✅ Resolved, cause was X, thanks for your patience"). Note the incident tone is a
balance — serious enough to convey it matters and prompt action, but calm enough to
reassure rather than panic (a panicked incident message makes everyone more anxious and
less effective; a calm-but-serious one signals competent handling). Getting the tone
right — matched to severity, and for incidents, serious-but-calm — makes announcements
both accurate (people understand the real severity and act appropriately) and reassuring
(people trust it's being handled well). For your goal, matching tone to severity is a
professionalism-and-warmth point: it shows good judgment (you calibrate appropriately),
respects people's attention (no false alarms), and — for incidents — the calm-competent
tone reassures the team, which is a form of warmth under pressure. High-visibility
announcements are where tone calibration is most visible and most consequential, so
getting it right (clear structure + tone matched to severity) is a high-value skill for
both communication effectiveness and how you're perceived.
</details>

---

## Homework

Next time you need to write a wide announcement (a deploy notice, a change affecting
others, an incident update — or write a practice one for a hypothetical), use the
structure: what (headline first, bolded) + when + what-you-need-to-do (explicit) + where
to ask. Match the tone to severity (calm for routine, serious-but-not-panicked for
incidents, clear for all-clear). For incidents, set an update cadence and always send the
all-clear. If it's high-stakes, pre-write and have someone review it. Add to your
checklist: "announcement — what/when/action/where, tone matched to severity, all-clear
sent?"

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the wide-announcement skill — a high-visibility communication where
clarity and tone matter most. A strong response uses the structure (what-headline-first,
when, explicit action-needed, where to ask) and matches the tone to severity (calm for
routine, serious-but-calm for incidents, clear-and-warm for the all-clear), with an update
cadence for incidents and an all-clear to confirm resolution. The realizations: (1)
<strong>the structure makes announcements instantly graspable</strong> — leading with the
what (bolded), stating when, making the action-needed unmistakable, and pointing to where
to ask means the many scanning readers immediately know what's happening and whether they
need to act; (2) <strong>matching tone to severity is crucial and often gets missed</strong>
— it's easy to over-alarm (making routine things sound urgent) or under-play (being too
casual about a real incident), and both mislead; the right tone (calibrated to reality,
and for incidents serious-but-calm) informs accurately and reassures; (3) <strong>the
incident rhythm and all-clear matter</strong> — regular updates during an incident reassure
people it's handled (and prevent "any update?" floods), and the all-clear confirms
resolution so people who saw the problem aren't left uncertain. The meta-point: wide
announcements are high-visibility (many people read them, and see how you communicate), so
getting them right — clear structure, tone matched to severity, incident rhythm, all-clear
— matters both for the immediate purpose (informing/coordinating many people) and for your
reputation (clear, well-calibrated announcements build trust and make you look
competent and composed, especially the calm-competent incident handling that reassures a
team under pressure). This completes Phase 4 (Slack Communication): message shapes (lead
with the ask), status updates (specific, done/doing/blocked), asking for help (make it
easy), answering/unblocking (answer first, warm), async etiquette (the unwritten rules),
and announcements (clear + tone-matched). Together, Phase 4 applied the accuracy/
naturalness/warmth foundation (Phases 1–3) to the specific patterns of Slack, where most
of your work communication happens — so mastering it directly improves your day-to-day
communication and collaboration, and makes you the kind of clear, warm, considerate,
low-friction teammate people genuinely like working with. Add the announcement checklist
item. The remaining phases move beyond Slack to spoken/meeting communication (Phase 5
meetings, Phase 6 spoken fluency), written documents (Phase 7 design docs), feedback
(Phase 8), and the capstone (Phase 9) — broadening from chat to the full range of
workplace communication, all built on the accurate, natural, warm foundation you've
developed.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Phase 5 — Meetings (Lesson 27: Agendas and Opening a Meeting) →](lesson-27-agendas-opening){: .btn .btn-primary }
