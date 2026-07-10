---
title: "Lesson 09 — Running Design Reviews"
nav_order: 4
parent: "Phase 2: Technical Leadership"
---

[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }

---

# Lesson 09: Running Design Reviews

## Concept

A design review can be one of two things: a gauntlet where senior engineers
prove how smart they are by tearing designs apart, or the team's single best
venue for catching risk, spreading knowledge, and growing people. Which one it is
depends almost entirely on how the lead runs it.

The reframe: a design review has three goals, and their *order* matters:

```
   1. CATCH RISK    — find the flaws, gaps, and risks before they're built
                      (the obvious goal, but not the only one)
   2. SPREAD CONTEXT — everyone leaves understanding the design and the system
                      better; knowledge doesn't stay siloed in one head
   3. GROW PEOPLE   — the author and the reviewers both level up their design
                      judgment; it's a teaching venue, not just a filter
```

Most people think design review is only goal 1 (catch the flaws). But a review
run purely as flaw-hunting becomes adversarial — authors get defensive, juniors
stop presenting (too scary), and the team learns that design review is where you
get attacked. A review run for all three goals — catch risk *and* teach *and*
build shared understanding — becomes the highest-leverage technical venue the
team has, because it improves the design, the system knowledge, and the people,
all at once.

The lead's role is to *set the tone* (curious, not gatekeeping), *protect the
author* (especially juniors) from pile-ons, and *steer* toward the important
issues rather than bikeshedding — while occasionally exercising the judgment to
overrule when needed.

---

## How It Works

### The reviewer's stance: curious, not gatekeeping

The difference between a good and bad review culture is the stance reviewers
take. **Gatekeeping** stance: "here's what's wrong with this," "why didn't you
consider X," "this won't work" — adversarial, positional, makes the author defend.
**Curious** stance: "help me understand why you chose X over Y," "what happens if
Z?", "have you considered..." — collaborative, inquisitive, invites the author to
think rather than defend. The curious stance catches just as many real problems
(often more, because the author isn't defensive and engages honestly) while
building rather than eroding the culture. The lead models this stance and gently
corrects reviewers who slip into gatekeeping.

### Async-first

The best design reviews often happen *async* before any meeting: the author
circulates the design doc, reviewers comment in writing over a day or two, and the
synchronous meeting (if needed) is reserved for the genuinely contentious points
that written back-and-forth couldn't resolve. This respects the maker's schedule
(Lesson 4 — no meeting fragmenting everyone's day), gives thoughtful reviewers
time to reason (vs the person who thinks fastest out loud dominating a live
meeting), and creates a written record. Reserve live time for real disagreement,
not for reading the doc together.

### Handling the senior steamroller and the pile-on

Two dynamics the lead must manage. The **senior steamroller** — a senior engineer
whose strong opinions and status shut down discussion, so the review becomes
"whatever they think," and juniors' good ideas and the author's reasoning get
flattened. The lead intervenes: draw out other voices ("before we settle on
Sara's approach — [author], walk us through your reasoning again," "[quieter
person], what's your read?"), and privately coach the steamroller. The
**pile-on** — several reviewers piling criticism on one author (especially a
junior), which is demoralizing and unproductive regardless of whether the
criticisms are valid. The lead moderates: consolidate the feedback ("it sounds
like the core concern is X — let's focus there"), protect the author's dignity,
and if needed take it to a smaller conversation rather than a public gang-up.

### When to overrule

Usually the lead facilitates and lets the team reach the decision. But sometimes
the lead must overrule — when the team is converging on a genuinely bad choice, or
is deadlocked and someone needs to decide (disagree-and-commit, Lesson 7). Do it
sparingly (overruling often kills the team's ownership and engagement) and
transparently (explain the reasoning, acknowledge the view you're overriding) —
and prefer to guide the team to the right answer rather than impose it, reserving
the overrule for when guidance genuinely fails and a decision is needed.

{: .note }
> **Reviews are a teaching venue — protect the learning**
> The single biggest waste in design review is treating it as pure quality-
> control and ignoring the teaching opportunity. Every review is a chance for the
> author to build design judgment and for reviewers (including juniors watching)
> to learn how good engineers reason about systems. That's why the <em>stance</em>
> matters so much: a curious, explain-your-reasoning review teaches; an
> adversarial flaw-hunt catches some bugs but teaches only that design review is
> scary. The lead who optimizes reviews for learning — not just filtering — gets
> a team whose design judgment compounds over time.

---

## Lab — Scenario

**The situation:** A junior engineer, Dev, has posted a design doc for a new
feature in the team's review channel. The design has a real flaw — a data-
consistency issue under concurrent access that would cause bugs in production.
Two senior engineers, Alex and Sam, have already left comments — and they're
piling on: Alex wrote "This won't work, you clearly didn't think about
concurrency" and listed four problems in a dismissive tone; Sam replied "+1, also
the whole data model is wrong, we should just do it the way we did in the old
service." Dev hasn't responded and, you notice, seems to have gone quiet.

**You're the lead. Write your actual replies in the thread — moderate this so the
real flaw gets fixed, Dev isn't crushed, and the review culture is protected.**

**Your response:**

<details>
<summary>Show Model Answer</summary>
<br>
The situation has three problems to solve at once: the real technical flaw needs
addressing, Dev is being piled on and has gone quiet (a culture and a
person problem), and the seniors' behavior needs correcting without a public
scolding that creates new drama. A strong response threads all three. Example
replies:
<br><br>
<strong>First, reframe and protect (public, in-thread):</strong> "Thanks for
putting this up, Dev — reviewing design before building is exactly right, and
this is a solid start on a genuinely tricky feature. Alex and Sam have flagged an
important area: the concurrency/consistency behavior. Dev, let's dig into that
together — <strong>what happens in your design when two requests update the same
record at the same time?</strong> Walking through that case will tell us whether
we've got a real gap to close." — This does several things: it thanks and
validates Dev (countering the pile-on's demoralizing effect and re-engaging
someone who'd gone quiet), reframes the criticism as "an important area to dig
into" (collaborative, not "you failed"), and — crucially — leads Dev to
<em>discover</em> the flaw through a curious question rather than being told "this
won't work" (teaching, not just correcting — Dev will understand and remember the
concurrency issue far better for having reasoned to it). It also surfaces the real
technical problem (the concurrency flaw is legitimate and must be fixed — you're
not softening the substance, just the delivery).
<br><br>
<strong>Redirect the "do it the old way" comment:</strong> "Sam, on the data
model — can you say more about what specifically the old-service approach handled
that this doesn't? Want to make sure we're comparing the actual trade-offs rather
than defaulting to the familiar." — This doesn't dismiss Sam, but it stops "the
whole data model is wrong, do it the old way" from steamrolling (Lesson 8: "the
old way" isn't automatically right; make the trade-off explicit) and models the
curious stance. It also protects Dev from a vague, deflating "your whole approach
is wrong" by forcing it into a specific, discussable point.
<br><br>
<strong>Handle the seniors privately (NOT in the thread):</strong> Separately,
1:1 with Alex (and Sam): "Your technical points about the concurrency issue were
right and important — but the delivery landed as a pile-on, and Dev went quiet.
'You clearly didn't think about concurrency' shreds someone; 'what happens under
concurrent writes?' surfaces the same issue and teaches. I need our reviews to
catch flaws <em>and</em> keep people engaged enough to keep posting designs —
especially juniors. Can you help me set that tone?" — The private coaching is
essential: correcting them publicly would create new drama and defensiveness, and
the goal is to change future behavior, not to win the thread. You affirm the
substance (their technical points were valid — you're not saying "be nice at the
expense of rigor") while addressing the delivery (Lesson 19's feedback skills).
<br><br>
<strong>Why this is the right shape:</strong> The flaw gets fixed (you led Dev
straight to the concurrency case — it will get addressed, with Dev understanding
<em>why</em>); Dev isn't crushed (validated, re-engaged, taught rather than
attacked — they'll post the next design instead of dreading review); and the
culture is protected (you modeled the curious stance publicly and coached the
gatekeeping privately, so future reviews trend collaborative). Note what you
<em>didn't</em> do: you didn't let the flaw slide to spare Dev's feelings (the
consistency bug is real and must be fixed — kindness isn't avoiding the substance),
and you didn't publicly reprimand the seniors (that fixes nothing and creates
drama). The craft is separating the <em>valid technical content</em> of the
criticism (keep it, address it) from the <em>harmful delivery</em> (fix it,
privately) — getting the design right AND growing Dev AND protecting the review
culture, which are the three goals of design review this lesson names.
<br><br>
Common mistakes: (1) jumping in to fix the flaw yourself or telling Dev the answer
("you need optimistic locking here") — solves the immediate problem but misses the
teaching and doesn't help Dev's judgment; (2) publicly scolding Alex and Sam
("guys, be nicer") — creates drama, makes them defensive, and doesn't change the
underlying behavior; (3) siding with the pile-on ("yeah Dev, this needs work") —
completes the crushing; (4) protecting Dev's feelings by downplaying the real flaw
— lets a genuine production bug through and teaches that review is toothless.
</details>

---

## Further Reading

| Topic | Source |
|---|---|
| Code/design review culture | Google Engineering Practices — <https://google.github.io/eng-practices/review/> |
| Psychological safety in technical discussion | *The Fearless Organization*, Amy Edmondson |
| Running effective technical reviews | *The Manager's Path*, Camille Fournier |
| The curious vs gatekeeping stance | *Debugging Teams*, Fitzpatrick & Collins-Sussman |
| Facilitation & drawing out quiet voices | *The Art of Gathering*, Priya Parker |

---

## Checkpoint

**Q1.** A design review has three goals — catch risk, spread context, grow people.
Why does running it as *only* flaw-hunting (goal 1) undermine even that goal over
time?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
Running review as pure flaw-hunting is self-defeating even at catching flaws,
through several compounding mechanisms. (1) <strong>It makes authors defensive</strong>
— when review is an attack to survive rather than a collaboration to improve, the
author's energy goes to defending their design, not honestly engaging with the
concerns; they argue back, minimize problems, or dig in — so real flaws get
<em>less</em> scrutiny, not more, because the author isn't a willing participant in
finding them (a curious "help me understand X" gets an honest "oh, actually X is a
problem," while an adversarial "this is wrong" gets "no it isn't"). (2) <strong>It
scares people out of presenting</strong> — especially juniors, who learn that
design review is where you get torn apart, so they either stop posting designs
(building without review — the flaws now ship uncaught, the exact opposite of the
goal) or post only safe, un-ambitious designs they're sure will survive (so the
review never sees the risky, interesting work that most needs reviewing). (3)
<strong>It kills the teaching that improves future designs</strong> — a review
that only hunts flaws catches <em>this</em> design's bugs but teaches nobody how to
design better, so the same classes of flaws recur; a review that explains reasoning
and grows judgment means fewer flaws in <em>future</em> designs (the compounding
effect — you're not just filtering bad designs, you're producing engineers who
make fewer bad designs). (4) <strong>It drives away good reviewers and good
contributors</strong> — an adversarial review culture is unpleasant, and people
disengage from it (reviewers stop bothering, or the culture selects for the
combative and repels the collaborative), degrading the review pool. So the flaw-
hunting-only approach, pursued for the sake of catching flaws, ends up catching
<em>fewer</em> flaws over time: defensive authors, absent juniors, recurring
mistakes, and a hollowed-out review culture. The paradox is that optimizing narrowly
for goal 1 (catch risk) undermines goal 1, while optimizing for all three (catch
risk in a way that also teaches and builds shared understanding, via the curious
stance) catches <em>more</em> risk — because engaged authors surface their own
problems, people bring their real designs, and the team's design judgment improves
so there's less to catch. This is why the lead invests in the <em>stance</em> and
the <em>safety</em> of review, not because catching flaws is secondary, but
because a psychologically safe, teaching-oriented review is the more effective way
to catch flaws — the "soft" culture work is in service of the "hard" goal.
</details>

**Q2.** Contrast the "gatekeeping" and "curious" reviewer stances with concrete
phrasings, and explain why the curious stance often catches *more* real problems
despite sounding softer.

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
<strong>Gatekeeping</strong> phrasings are positional and adversarial, asserting
judgment: "This won't work." "You didn't consider concurrency." "This is wrong,
do it the other way." "Why would you do it like this?" (rhetorical, implying the
choice was dumb). They put the reviewer above the author and the author on
defense. <strong>Curious</strong> phrasings are inquisitive and collaborative,
inviting reasoning: "Help me understand why you chose X over Y." "What happens
under concurrent writes?" "Have you considered the case where Z?" "Walk me through
how this handles [scenario]." They put reviewer and author side by side, looking
at the design together. The curious stance catches <em>more</em> real problems
despite sounding softer for several reasons. (1) <strong>It engages the author as
an ally rather than an adversary</strong> — asked "what happens under concurrent
writes?", the author actually thinks it through and often discovers the flaw
themselves ("oh — yeah, that would double-count, that's a real problem"); told
"you didn't handle concurrency," they get defensive and argue, so the same flaw
gets contested rather than fixed. Honest engagement surfaces more than defensive
argument. (2) <strong>It surfaces the author's reasoning</strong> — "help me
understand why X over Y" reveals <em>whether</em> the author considered Y and
their reasoning, which often uncovers either a good reason the reviewer missed (so
the "flaw" wasn't one — avoiding a wrong correction) or a gap in the author's
thinking (a real issue) — either way you learn more than an assertion does. (3)
<strong>It avoids false positives from reviewer overconfidence</strong> — the
gatekeeping "this is wrong" is sometimes itself wrong (the reviewer missed context
the author had), and its adversarial framing makes it hard to walk back; the
curious question lets the truth emerge without anyone losing face. (4) <strong>It
keeps the channel open</strong> — the author stays engaged and forthcoming (sharing
the messy edges and uncertainties where flaws hide), rather than clamming up and
presenting only the polished, defended parts. So "softer" is a misread: the curious
stance isn't softer on the <em>substance</em> (it pursues the same technical
concerns, and often more rigorously because the author cooperates) — it's softer
on the <em>author's ego</em>, which is precisely what makes the substantive
inquiry more productive. The gatekeeping stance trades away the author's honest
engagement for the reviewer's feeling of authority, and that's a bad trade for
finding problems. The lead models curious phrasing and gently redirects gatekeeping
("let's ask Dev what happens under concurrency rather than telling them it's
broken") — not to be nice, but because it's the more effective way to run a review,
and because it builds the safe, teaching-oriented culture that makes <em>all</em>
future reviews better (Q1).
</details>

---

## Homework

Observe (or recall) your team's last few design reviews with this lesson's lens.
Assess: what's the dominant reviewer stance — curious or gatekeeping? Is there a
senior steamroller or a pile-on pattern? Do juniors present their designs, or
avoid it? Are reviews teaching venues or just filters? Then pick one concrete
change to make in the next review you run or participate in — a specific phrasing
shift, a way to draw out a quiet voice, a move to protect an author, an async-first
change — and try it. What did you notice about how the review's dynamics and
outcomes changed?

**Your answer:**

<details>
<summary>Show Answer</summary>
<br>
The exercise builds the diagnostic eye for review culture, which most teams never
examine explicitly. A strong response honestly assesses the current state — and
many leads, looking closely, find their reviews trend more gatekeeping than they'd
assumed (the senior engineers, including possibly the lead, assert judgments more
than they ask questions; there's a dominant voice others defer to; juniors are
quieter or absent from presenting; reviews catch flaws but rarely teach). The
value is in the specific, small change tried and its observed effect. Effective
changes and their typical results: (1) <strong>shifting your own phrasing from
assertions to questions</strong> ("what happens if..." instead of "this doesn't
handle...") — often the author engages more openly, sometimes discovers the issue
themselves, and the whole thread's tone softens because the lead modeled it. (2)
<strong>deliberately drawing out a quiet voice</strong> ("[name], you've worked in
this area — what's your read?") — surfaces input that would have been lost and
signals that the review isn't just for the loudest, often noticeably shifting who
participates. (3) <strong>protecting an author under pile-on</strong> (consolidating
scattered criticism into "the core concern seems to be X, let's focus there," or
validating before critiquing) — visibly re-engages an author who'd gone
defensive/quiet and keeps the review productive. (4) <strong>going async-first</strong>
(circulating the doc for written comments before a meeting) — often produces more
thoughtful input (people who don't think fast out loud contribute) and shorter,
more focused meetings. (5) <strong>privately coaching a steamroller</strong> — the
slowest to show effect but the highest-leverage for a persistent dynamic. What
you're likely to notice: even a small stance shift changes the emotional
temperature of the review (less defensive, more collaborative) and often the
<em>output</em> (the author engages more honestly, so more real issues surface and
get genuinely addressed rather than argued) — demonstrating the lesson's core claim
that how the review is run, not just who's in it, determines whether it catches
flaws, teaches, and keeps people engaged. The meta-skill this builds: seeing design
review (and by extension code review, meetings, any technical discussion) as a
<em>culture you actively shape</em> through your own modeled behavior and your
moderation, not a fixed thing that just happens — the lead sets the tone, and
small deliberate moves (a question instead of an assertion, drawing out a voice,
protecting an author) compound into a review culture that's the team's best venue
for catching risk and growing people, or into one that's an adversarial gauntlet
people dread. If your reviews are already curious, safe, and teaching-oriented with
broad participation — excellent, that's a real achievement worth protecting (it
erodes easily as teams grow and pressure rises); the ongoing work is maintaining it
and coaching new/senior members into the stance rather than letting it drift toward
gatekeeping.
</details>

---

<!-- nav-next -->
[← Home]({{ '/' | relative_url }}){: .btn .btn-outline }
[Next: Lesson 10 — Driving Technical Standards →](lesson-10-technical-standards){: .btn .btn-primary }
