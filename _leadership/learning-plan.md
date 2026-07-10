---
title: Learning Plan
nav_order: 2
---

# Engineering Leadership Learning Plan

A curriculum for the Senior Developer → Lead Developer / Engineering Manager
transition. It covers the eight competency areas that define the role: technical
leadership, communication, stakeholder management, project leadership, mentoring
& coaching, influence without authority, business & product thinking, and people
management.

**Lab format:** this track has no terminal. Each lab is a **realistic scenario
exercise** — a situation you will actually face ("your strongest engineer wants
to rewrite the codebase in Rust", "your PM promised a date you can't hit") —
where you write your response before revealing a model answer that explains the
reasoning, common mistakes, and useful phrasing. Checkpoints and homework work
the same way. The [English for Work]({{ '/english/learning-plan.html' | relative_url }})
track pairs with this one: it teaches the *language*, this track teaches the
*judgment*.

**Core references** (cited throughout the Further Reading tables):
*The Manager's Path* (Fournier), *Staff Engineer* (Larson), *An Elegant Puzzle*
(Larson), *The Making of a Manager* (Zhuo), *Crucial Conversations* (Patterson
et al.), *Turn the Ship Around!* (Marquet), plus StaffEng.com, LeadDev, the
Google Engineering Practices documentation, and the Manager Tools podcasts.

---

## [Phase 1: The Transition](lessons/phase-01-transition.html)

### [Lesson 1: What actually changes](lessons/lesson-01-what-changes.html)
**Goal:** Redefine your output: from code you write to outcomes your team produces.
**Topics:** The IC→lead shift; multiplier vs contributor; why your old strengths can become traps; what "success this week" now means; the discomfort is the job.
**Scenario:** Your first week as lead: a gnarly bug you could fix in 2 hours, or a planning doc due Friday. Walk through the real decision.

### [Lesson 2: Lead vs EM vs Staff — the tracks](lessons/lesson-02-lead-em-staff.html)
**Goal:** Understand the roles you're choosing between so you aim deliberately.
**Topics:** Tech Lead, Team Lead, EM, Staff/Principal IC — how responsibilities differ; the dual-track ladder; the pendulum (Fournier's engineer/manager oscillation); what your company's version actually looks like.
**Scenario:** Map your own org: who holds which of these roles, and where do the boundaries blur?

### [Lesson 3: Letting go of the code](lessons/lesson-03-letting-go-of-code.html)
**Goal:** Stop being the bottleneck without abandoning technical credibility.
**Topics:** Why hoarding the hard tasks caps your team; the "hero" anti-pattern; staying hands-on the safe ways (spikes, tooling, pairing — not the critical path); grief is normal.
**Scenario:** A critical feature is late and you *know* you'd finish it fastest yourself. Decide, and defend the second-order effects.

### [Lesson 4: Time and leverage](lessons/lesson-04-time-leverage.html)
**Goal:** Restructure your calendar around leverage, not tasks.
**Topics:** Maker vs manager schedule; leverage (Grove: output = your team's output + neighboring teams you influence); calendar audit; batching interrupts; protecting deep-work blocks for your ICs.
**Scenario:** Audit a (provided) chaotic lead's calendar week — cut, delegate, batch, and justify each move.

### [Lesson 5: Identity, doubt, and the psychology of the switch](lessons/lesson-05-identity-psychology.html)
**Goal:** Handle the internal side: impostor feelings, loss of flow, slower feedback loops.
**Topics:** Where your sense of progress goes; measuring yourself weekly not hourly; impostor syndrome in new leads; energy management; when the discomfort is a signal vs just growth.
**Scenario:** Three months in, you feel useless — no code shipped "by you". Write your own performance review for those months honestly.

---

## [Phase 2: Technical Leadership](lessons/phase-02-technical-leadership.html)

### [Lesson 6: Technical vision and strategy](lessons/lesson-06-tech-vision.html)
**Goal:** Give the team a direction that survives contact with quarterly chaos.
**Topics:** Vision vs strategy vs roadmap; writing a 1-page tech vision; diagnosing before prescribing (Rumelt's kernel); strategy as what you *won't* do; keeping it alive.
**Scenario:** Write a 1-page technical vision for a (provided) team drowning in a legacy monolith.

### [Lesson 7: Architecture decisions and ADRs](lessons/lesson-07-adrs.html)
**Goal:** Make big decisions legible, reviewable, and durable.
**Topics:** Architecture Decision Records; context/decision/consequences structure; reversible vs one-way doors; deciding *when* to decide; disagree-and-commit.
**Scenario:** Write an ADR for a real decision from your past team — then compare against the model's structure.

### [Lesson 8: Evaluating trade-offs](lessons/lesson-08-tradeoffs.html)
**Goal:** Reason explicitly about speed vs quality, simple vs scalable, build vs buy.
**Topics:** Making trade-offs visible instead of implicit; cost of delay; "what would have to be true"; the 10x-growth test; premature scalability as debt too.
**Scenario:** Two designs: ships in 2 weeks and strains at 10x load, or 8 weeks and scales. The business context decides — work through three different contexts.

### [Lesson 9: Running design reviews](lessons/lesson-09-design-reviews.html)
**Goal:** Turn design review from a gauntlet into the team's best learning venue.
**Topics:** Review goals (catch risk, spread context, grow people — in that order); async-first review; the reviewer's stance (curious, not gatekeeping); handling the senior steamroller; when to overrule.
**Scenario:** A junior's design has a real flaw and two seniors are piling on in comments. Moderate the thread — write your replies.

### [Lesson 10: Driving technical standards](lessons/lesson-10-technical-standards.html)
**Goal:** Raise the bar without becoming the style police.
**Topics:** Standards worth having (and not); paved roads vs guardrails vs gates; linting/CI as automated standards; getting adoption without decree; Google's engineering practices as reference.
**Scenario:** Half the team writes tests religiously, half doesn't. Design your path to a real testing standard — decree is not available.

### [Lesson 11: Managing technical debt strategically](lessons/lesson-11-tech-debt.html)
**Goal:** Treat debt as a portfolio you manage, not a moral failing you lament.
**Topics:** Debt taxonomy (deliberate/accidental, reckless/prudent); making debt visible to non-engineers (risk & velocity language, not code language); the 20% myth vs debt budgets; when to rewrite (rarely) vs strangle.
**Scenario:** Convince a (provided) skeptical PM to fund a quarter of refactoring — write the pitch in business terms.

### [Lesson 12: How much should a lead still code?](lessons/lesson-12-how-much-to-code.html)
**Goal:** Find your sustainable technical involvement level.
**Topics:** The honest answer: it depends on team size and role; safe contributions vs critical-path traps; code review as your main technical surface; keeping depth via spikes and incidents; the credibility question.
**Scenario:** Design your own weekly technical-involvement budget for three situations: 4-person team as tech lead, 8-person team as EM, two teams as senior EM.

---

## [Phase 3: Communication Foundations](lessons/phase-03-communication.html)

### [Lesson 13: Audience-first communication](lessons/lesson-13-audience-first.html)
**Goal:** Stop transmitting; start landing. Shape every message by receiver, not sender.
**Topics:** The curse of knowledge; audience/purpose/medium triage; lead with the conclusion (BLUF); one message, one point; calibrating detail by audience.
**Scenario:** Take one technical update and write it three ways: to your team, to your director, to a customer.

### [Lesson 14: Explaining technology to non-technical people](lessons/lesson-14-explaining-tech.html)
**Goal:** Make execs and PMs *feel* the technical situation without the jargon.
**Topics:** Analogies that carry weight (and when they mislead); quantifying in their units (money, risk, time); answering "why is it slow/late/hard" honestly; the two-sentence version of anything.
**Scenario:** Explain to a CFO why the migration needs three more months — without the words "refactor", "tech debt", or "legacy".

### [Lesson 15: Running effective meetings](lessons/lesson-15-effective-meetings.html)
**Goal:** Make every meeting you run worth its (salary × attendees) cost.
**Topics:** The no-agenda-no-meeting rule; decision vs status vs brainstorm meetings need different shapes; timeboxing and parking lots; ending with owners and dates; killing recurring zombies; async alternatives.
**Scenario:** You inherit a weekly 10-person, 60-minute "sync" everyone hates. Redesign it — including what stops being a meeting at all.

### [Lesson 16: Writing design documents and proposals](lessons/lesson-16-design-docs.html)
**Goal:** Write docs that get read, get useful feedback, and get to a decision.
**Topics:** Context → problem → options → recommendation; the one-pager vs the deep doc; writing for skimmers (headings carry the argument); soliciting review without design-by-committee; the decision log at the end.
**Scenario:** A (provided) rambling 6-page design doc: restructure its outline and write its missing one-paragraph summary.

### [Lesson 17: Presenting and demos](lessons/lesson-17-presenting.html)
**Goal:** Present work — yours and your team's — so it lands and gets credited.
**Topics:** Story shape (situation → tension → resolution); slides as evidence, not script; demoing without a net (and with one); making your *team's* work visible upward; handling hostile questions.
**Scenario:** Prepare the 5-minute quarterly review of your team's work for the VP — outline, opening line, and the one slide you'd show.

### [Lesson 18: Async and written communication](lessons/lesson-18-async-communication.html)
**Goal:** Lead a team through text: Slack, docs, and code review comments as leadership surfaces.
**Topics:** When async beats a meeting (and reverse); writing decisions down where they'll be found; tone in text is 2x easier to misread — over-signal warmth; response-time norms; the daily written standup pattern.
**Scenario:** A decision thread is going in circles across 40 Slack messages. Write the message that lands the decision.

---

## [Phase 4: Feedback & Difficult Conversations](lessons/phase-04-feedback.html)

### [Lesson 19: Feedback that lands — SBI](lessons/lesson-19-sbi-feedback.html)
**Goal:** Give behavioral feedback that changes behavior instead of raising shields.
**Topics:** Situation-Behavior-Impact; observation vs judgment ("you're careless" vs "the last two PRs shipped without tests"); timeliness; feedback ratios; asking before telling.
**Scenario:** An engineer dominates every meeting and interrupts juniors. Write the SBI feedback, word for word.

### [Lesson 20: Praise that lands](lessons/lesson-20-praise.html)
**Goal:** Make recognition specific, deserved, and motivating — it's half the feedback job.
**Topics:** Specific beats generic ("great job" is empty); public vs private calibration; praising process and growth, not just wins; recognition as retention; passing credit upward accurately.
**Scenario:** Three teammates contributed differently to a launch (architecture, grind, unblocking). Write each one's recognition — no copy-paste.

### [Lesson 21: Receiving feedback as a lead](lessons/lesson-21-receiving-feedback.html)
**Goal:** Make it safe to tell you the truth — your information supply depends on it.
**Topics:** Power distorts honesty; asking for specific critique ("what should I stop doing?"); the flinch and how it silences teams for months; thanking, digesting, closing the loop; skip-level signals.
**Scenario:** In retro, a junior says your late-night Slack messages stress people out. Everyone watches your face. Respond — then design the follow-up.

### [Lesson 22: Crucial conversations](lessons/lesson-22-crucial-conversations.html)
**Goal:** Stay effective when stakes are high, opinions differ, and emotions run.
**Topics:** The Crucial Conversations toolkit: safety first, mutual purpose, mastering your story (facts vs the story you tell about them); dialogue vs silence/violence; contrasting statements.
**Scenario:** Your peer lead publicly blamed your team for a slipped deadline in front of leadership — and it's partly true. Have the conversation.

### [Lesson 23: Defusing emotional situations](lessons/lesson-23-defusing-emotions.html)
**Goal:** Be the calmest person in the room, usefully.
**Topics:** Name-it-to-tame-it; listening before solving; when to pause a meeting; anger vs distress vs frustration need different responses; your own triggers; following up after the storm.
**Scenario:** An engineer breaks down in a 1:1 — burnout, family stress, and a looming deadline all at once. Navigate the next ten minutes.

---

## [Phase 5: 1:1s, Coaching & Mentoring](lessons/phase-05-coaching.html)

### [Lesson 24: 1:1s that aren't status meetings](lessons/lesson-24-one-on-ones.html)
**Goal:** Run the highest-leverage 30 minutes of your week.
**Topics:** The 1:1 is *their* meeting; cadence and non-negotiability; question banks that open people up; tracking themes across weeks; the status-report failure mode; Manager Tools' classic format.
**Scenario:** Your 1:1s with one engineer have become "everything's fine" in five minutes. Design your next three 1:1s to break the pattern.

### [Lesson 25: Listening and powerful questions](lessons/lesson-25-listening-questions.html)
**Goal:** Extract the real problem, which is rarely the presented one.
**Topics:** Levels of listening; the advice trap (Stanier's *The Coaching Habit*); "what's the real challenge here for you?"; silence as a tool; summarize-and-check.
**Scenario:** An engineer asks to switch teams. Five questions to understand before you react — write them, and what each might reveal.

### [Lesson 26: Coaching vs mentoring](lessons/lesson-26-coaching-vs-mentoring.html)
**Goal:** Know when to give answers (mentor) and when to grow the answer-finder (coach).
**Topics:** The distinction and why it matters; GROW model; defaulting to coaching for growth, mentoring for onboarding/crisis; the leading-question anti-pattern; Marquet's "leader-leader" intent model.
**Scenario:** Same question — "should we use Kafka or a cron job?" — from a junior in week 2 and a senior gunning for staff. Respond to each.

### [Lesson 27: Mentoring engineers](lessons/lesson-27-mentoring-engineers.html)
**Goal:** Grow juniors and mid-levels deliberately, not by osmosis.
**Topics:** Scaffolding (stretch + safety net); teaching judgment via "what would you do first?"; apprenticeship patterns; mentoring outside your team; being mentored yourself, always.
**Scenario:** Design a 90-day growth arc for a bootcamp grad joining your team: week-by-week responsibilities, checkpoints, and failure signals.

### [Lesson 28: Code review as teaching](lessons/lesson-28-code-review-teaching.html)
**Goal:** Turn your review comments into your highest-volume teaching channel.
**Topics:** Review the code's risk, teach the pattern; comment tiers (blocking/suggestion/nit — label them); questions over commands; praising in review; Google's code review guidelines; when to pair instead.
**Scenario:** A (provided) PR from a junior: functionally fine, structurally naive. Write the review — it must teach without exhausting.

### [Lesson 29: Sponsorship](lessons/lesson-29-sponsorship.html)
**Goal:** Spend your credibility on other people's visibility — the growth tool nobody tells you about.
**Topics:** Mentoring advises, sponsorship *acts* (staffing them on the visible project, saying their name in the room); who you're accidentally not sponsoring; stretch assignments as sponsorship; tracking whose career you've moved.
**Scenario:** A visible cross-team project needs an owner. Your safest pick is the person who always gets picked. Decide who, and handle the person not picked.

---

## [Phase 6: Delegation & Growing the Team](lessons/phase-06-delegation.html)

### [Lesson 30: The delegation ladder](lessons/lesson-30-delegation-ladder.html)
**Goal:** Delegate outcomes, not tasks — at the right rung for each person.
**Topics:** Levels from "do exactly this" to "own this area"; matching rung to competence+confidence; the monitoring contract (check-ins without hovering); delegating things you're good at (that's the point); the boomerang problem.
**Scenario:** Delegate "own our flaky CI" to a mid-level engineer: write the actual handoff conversation, including the safety net and check-in plan.

### [Lesson 31: Assigning for growth, not speed](lessons/lesson-31-assigning-for-growth.html)
**Goal:** Use the work itself as your main people-development tool.
**Topics:** The fastest-person trap and how it calcifies teams; stretch assignments with contained blast radius; rotating the glue work fairly; balancing delivery pressure vs growth (you must do both); tracking who's growing stale.
**Scenario:** Plan next sprint's assignments for a (provided) 5-person team where the same senior always gets the interesting work.

### [Lesson 32: Career development conversations](lessons/lesson-32-career-conversations.html)
**Goal:** Know where each person wants to go and bend their work toward it.
**Topics:** Career conversations ≠ performance reviews; the "where do you want to be in 3 years (and it's fine not to know)" conversation; ladders and legible promotion criteria; the promotion packet you build all year; managing the not-promoted conversation.
**Scenario:** An engineer expects a promotion this cycle; they're not ready and it isn't happening. Have the conversation — and the 6-month plan that follows.

### [Lesson 33: Growing seniors into leads](lessons/lesson-33-growing-leads.html)
**Goal:** Build your own succession — and multiply yourself.
**Topics:** Spotting lead potential (hint: it's not the best coder); giving away pieces of your own job deliberately; letting them fail safely at leading; the "acting lead" trial; when someone wants your actual job.
**Scenario:** You're going on 4 weeks of leave. Design who covers what — as a growth plan, not a coverage rota.

### [Lesson 34: Succession and the bus factor](lessons/lesson-34-succession-bus-factor.html)
**Goal:** Make no one irreplaceable — including you — without making anyone feel replaceable.
**Topics:** Knowledge mapping (who's the only one who knows X?); docs, pairing, and rotation as de-risking; the hero who hoards context; your own bus factor as lead; framing redundancy as freedom (vacations!) not threat.
**Scenario:** One engineer solely understands the billing system and likes it that way. De-risk it within a quarter without alienating them.

---

## [Phase 7: Influence Without Authority](lessons/phase-07-influence.html)

### [Lesson 35: Where influence actually comes from](lessons/lesson-35-sources-of-influence.html)
**Goal:** Build the currency you'll spend for the rest of your career.
**Topics:** Credibility (track record), relationships (built before you need them), and framing; reciprocity; why title-based authority stops working one level up; the trust battery; influence audits.
**Scenario:** Map your real influence network: for a change you care about, list who decides, who they listen to, and where you stand with each.

### [Lesson 36: Building buy-in for ideas](lessons/lesson-36-building-buy-in.html)
**Goal:** Get to "yes" before the meeting where "yes" happens.
**Topics:** Socializing ideas 1:1 first (never surprise a decision-maker in public); co-ownership beats ownership; the pilot/experiment framing; objection inventory; knowing when you've won (stop selling).
**Scenario:** You want the org to adopt trunk-based development. Three key people are skeptical for three different reasons. Plan the campaign.

### [Lesson 37: Resolving conflict](lessons/lesson-37-resolving-conflict.html)
**Goal:** Move conflicts from people-vs-people to people-vs-problem.
**Topics:** Task vs relationship conflict (one is healthy!); interests behind positions; the mediator stance for fights inside your team; escalation as a service, not a failure; when to let a disagreement stand.
**Scenario:** Two seniors are in a slow-burning war over code style that's now leaking into review tone. Mediate — write the joint conversation.

### [Lesson 38: Aligning teams around a direction](lessons/lesson-38-aligning-teams.html)
**Goal:** Create alignment that survives after you leave the room.
**Topics:** Alignment ≠ agreement; writing the direction down (the memo is the tool); disagree-and-commit done honestly; re-alignment cadence; detecting silent misalignment early (the "say it back" test).
**Scenario:** Three teams each interpret "we're going multi-region" differently and are building incompatible things. Design the re-alignment.

### [Lesson 39: Driving change across an organization](lessons/lesson-39-driving-change.html)
**Goal:** Land a change bigger than any team you control.
**Topics:** Change curves and why resistance is information; early adopters as beachheads; making the new way the easy way (paved roads again); visible wins early; sustaining past the honeymoon; Kotter, compressed.
**Scenario:** Migrate 12 teams from a shared staging server everyone hates-but-uses to ephemeral environments. Year-long campaign plan, one page.

---

## [Phase 8: Stakeholder Management](lessons/phase-08-stakeholders.html)

### [Lesson 40: Working with product managers and designers](lessons/lesson-40-pms-designers.html)
**Goal:** Form the triad that ships the right thing, not just the thing.
**Topics:** The PM/design/eng triad and healthy tension; engaging at problem-definition, not ticket-receipt; pushing back with alternatives, not vetoes; sharing the "why" downward; when eng should drive product ideas.
**Scenario:** The PM hands you a fully-specced feature that solves the wrong problem. Push back — without the relationship taking damage.

### [Lesson 41: Managing up](lessons/lesson-41-managing-up.html)
**Goal:** Make your manager effective on your behalf — and never surprise them.
**Topics:** What your boss actually needs from you (no surprises, options not problems, brevity); status that answers before it's asked; disagreeing upward safely; using your skip-level; managing a weak or overloaded manager.
**Scenario:** The project will slip 3 weeks and your director finds out Thursday at the exec review — unless you tell them today. Write the message.

### [Lesson 42: Customers and users](lessons/lesson-42-customers.html)
**Goal:** Keep the team connected to the humans on the other end.
**Topics:** Direct customer exposure for engineers (support rotations, calls); translating complaints into engineering signal; the enterprise-customer escalation; saying no to a customer; anecdote vs data.
**Scenario:** A major customer demands a feature that contradicts your product direction, and sales already half-promised it. Navigate the triangle.

### [Lesson 43: Other engineering teams and dependencies](lessons/lesson-43-cross-team-dependencies.html)
**Goal:** Get things from teams that don't work for you and have their own roadmap.
**Topics:** Dependency contracts (what/when/interface — in writing); their incentives, not their kindness; escalating without burning bridges; platform teams and being a good customer; the vendor-team relationship.
**Scenario:** Your launch needs an API change from a platform team whose roadmap is full. Their manager says "next quarter." You need it in six weeks.

### [Lesson 44: Negotiation and expectation management](lessons/lesson-44-negotiation-expectations.html)
**Goal:** Negotiate scope, time, and staffing like it's your job — it is now.
**Topics:** Interests vs positions (again — it's everywhere); BATNA in engineering negotiations; the iron triangle conversation (scope/time/people — pick two); under-promise calibration; renegotiating early beats apologizing late.
**Scenario:** Leadership wants the feature by the conference, full scope, no new people. Run the negotiation — with the trade-off menu you'd bring.

---

## [Phase 9: Business & Product Thinking](lessons/phase-09-business.html)

### [Lesson 45: How your company makes money](lessons/lesson-45-business-model.html)
**Goal:** Read your company like an investor, not an employee.
**Topics:** Revenue model, margins, and where engineering sits in the cost structure; unit economics basics; what the company's stage (growth vs profitability) implies for engineering choices; reading your own company's public numbers or board narrative.
**Scenario:** For your actual company: write one paragraph on how it makes money and one on what that means your team should optimize for.

### [Lesson 46: Customer needs and product strategy](lessons/lesson-46-product-strategy.html)
**Goal:** Understand the product thinking your PM does, well enough to be a real partner.
**Topics:** Jobs-to-be-done in one lesson; segments and personas without the fluff; product strategy as choosing which customers to disappoint; roadmap logic; where engineering insight changes product strategy.
**Scenario:** Your PM's roadmap has ten features. Using a (provided) strategy memo, argue which three matter and which three actively hurt.

### [Lesson 47: Metrics and KPIs](lessons/lesson-47-metrics-kpis.html)
**Goal:** Speak metrics natively: business ones and engineering ones, honestly connected.
**Topics:** North-star vs input metrics; engineering metrics that leaders watch (DORA, incident counts, cycle time) and how they're gamed; vanity metrics; instrumenting what you claim to care about; the metric-vs-judgment balance.
**Scenario:** Your director asks for "a dashboard proving the team is productive." Respond — including what you'd build and what you'd push back on.

### [Lesson 48: Connecting technical decisions to business outcomes](lessons/lesson-48-tech-to-business.html)
**Goal:** Justify every significant technical investment in business language.
**Topics:** The translation table (latency→conversion, reliability→churn, debt→velocity→time-to-market); cost of engineering time as a real number; risk in expected-value terms; when the honest answer is "this doesn't pay off."
**Scenario:** Build the business case for a 2-engineer-quarter observability investment — numbers included, assumptions labeled.

### [Lesson 49: "Should we build this?" — the question that changes](lessons/lesson-49-should-we-build-this.html)
**Goal:** Graduate from "how do we build it" to "should we, and is this the best way."
**Topics:** Opportunity cost as the lead's constant lens; build vs buy vs open-source vs don't; the walking skeleton / cheapest-test-first habit; sunk cost in engineering; killing projects well.
**Scenario:** Three months into a six-month internal tool build, a vendor launches 80% of it for $30k/year. Recommend — kill, continue, or pivot — and write the memo.

---

## [Phase 10: Project Leadership](lessons/phase-10-projects.html)

### [Lesson 50: Planning and roadmaps](lessons/lesson-50-planning-roadmaps.html)
**Goal:** Plan so that the plan survives its collision with reality.
**Topics:** Milestones as risk-retirement, not calendar decoration; working backward from the date; slack is a feature; now/next/later roadmaps; the planning cadence (quarterly + weekly reality checks).
**Scenario:** Turn a (provided) vague quarterly goal into a milestone plan with explicit risks retired at each milestone.

### [Lesson 51: Estimation — and why it fails](lessons/lesson-51-estimation.html)
**Goal:** Estimate honestly, communicate uncertainty, and stop being surprised.
**Topics:** Why estimates fail (unknown unknowns, optimism, pressure-contaminated estimates); ranges beat points; reference-class forecasting ("how long did similar things take"); estimate vs commitment vs target — never conflate; re-estimating without shame.
**Scenario:** The team says "3 weeks-ish"; your director hears "3 weeks" and tells sales. Repair the communication chain — and fix the process so it stops happening.

### [Lesson 52: Risk management](lessons/lesson-52-risk-management.html)
**Goal:** Find the thing that will actually kill the project — in week one, not week ten.
**Topics:** Pre-mortems ("it's six months later and we failed — why?"); risk registers that aren't theater (likelihood × impact × *owner*); derisking order (hardest/scariest first); the difference between a risk and an issue; watching for the quiet risks (people, dependencies).
**Scenario:** Run a written pre-mortem on a (provided) migration project; extract the top three risks and the week-one derisking actions.

### [Lesson 53: Dependency management](lessons/lesson-53-dependency-management.html)
**Goal:** Stop dependencies from being the reason your projects slip.
**Topics:** Mapping the critical path; dependency kickoffs (align before you're blocked); interface-first contracts so teams work in parallel; tracking external promises like your own tasks; the buffer-or-decouple decision.
**Scenario:** Your 8-week project depends on three teams. Design the coordination structure: what's written down, who syncs when, and the early-warning tripwires.

### [Lesson 54: Prioritization and trade-offs](lessons/lesson-54-prioritization.html)
**Goal:** Say no (or "not now") all day and stay trusted.
**Topics:** Cost of delay and WSJF-lite; the urgent-vs-important trap at team scale; making prioritization visible (the one ranked list); interrupt budgets for support/bugs; who owns the "no" — and disagreeing with priorities you must still execute.
**Scenario:** Sprint is committed; then the CEO's pet request, a customer P1, and a security patch all arrive Monday. Re-rank publicly and write the messages to the three requesters.

### [Lesson 55: When projects slip](lessons/lesson-55-when-projects-slip.html)
**Goal:** Recover slipping projects with your credibility intact — no death marches.
**Topics:** Detecting slip early (velocity vs status-green lies); Brooks's law (adding people makes it later); the honest recovery menu (scope, date, people, quality — mostly scope); communicating the slip (Lesson 41's no-surprises rule); the crunch decision and its real costs.
**Scenario:** Halfway through, you realize the deadline is 40% too optimistic. Write the recovery plan and the stakeholder message — same day.

### [Lesson 56: Incidents and blameless postmortems](lessons/lesson-56-incidents-postmortems.html)
**Goal:** Lead during and after fires — where teams learn who you really are.
**Topics:** Incident command basics (roles, comms cadence, the lead does *not* debug); severity honesty; blameless postmortems that actually change things (contributing factors, not root-cause-the-intern); action items that ship; normalizing near-miss reporting.
**Scenario:** Your engineer's config change took the site down for 2 hours; leadership wants "accountability." Run the postmortem and write your reply to leadership.

---

## [Phase 11: People Management (the EM path)](lessons/phase-11-people-management.html)

### [Lesson 57: Performance management](lessons/lesson-57-performance-management.html)
**Goal:** Evaluate fairly, document honestly, and never let a review surprise anyone.
**Topics:** Continuous evaluation vs review-season scramble; calibration meetings and how ratings really happen; writing evidence-based reviews (SBI at scale); the halo/recency effects; goals that mean something.
**Scenario:** Write the annual review for a (provided) engineer file: strong output, corrosive collaboration. Rating included — defend it for calibration.

### [Lesson 58: Underperformance and PIPs](lessons/lesson-58-underperformance.html)
**Goal:** Handle the hardest management duty early, fairly, and lawfully.
**Topics:** Diagnosing first (skill? will? role fit? life? you?); the feedback→expectations→support escalation ladder; PIPs — honest tool vs exit paperwork, and telling the difference; working with HR; the cost of *not* acting (your best people notice); parting ways with dignity.
**Scenario:** A once-strong engineer has been quietly underperforming for two months. Plan the arc: first conversation through 8-week resolution, both endings.

### [Lesson 59: Hiring and interviewing](lessons/lesson-59-hiring-interviewing.html)
**Goal:** Build the pipeline and interview craft to hire above your current bar.
**Topics:** Writing job descriptions that filter honestly; structured interviews (same questions, anchored rubrics) vs vibes; what actually predicts (work samples > puzzles); interviewer calibration and shadowing; bias traps ("culture fit" as mirror test); candidate experience as brand.
**Scenario:** Design a full interview loop for a senior backend role: stages, one anchored rubric, and the debrief format.

### [Lesson 60: Evaluating and closing candidates](lessons/lesson-60-evaluating-closing.html)
**Goal:** Decide well under uncertainty, then actually land the person.
**Topics:** The debrief (evidence before opinions, hire-bar not compare-to-team); no weak yeses; references done usefully; the close (what candidates actually decide on — growth, manager, mission, money); compensation conversations without games; the reneged-offer postmortem.
**Scenario:** Split panel on a candidate: two strong-hires, two no-hires, evidence on both sides. Run the debrief and make the call.

### [Lesson 61: Motivation](lessons/lesson-61-motivation.html)
**Goal:** Understand what actually drives engineers — and stop accidentally killing it.
**Topics:** Autonomy/mastery/purpose applied to engineering work; motivation is diagnosed per-person, not team-wide; the demotivators you control (churn, micromanagement, invisible work, broken promises); burnout signals and real responses; re-engaging a coasting senior.
**Scenario:** Three disengaged engineers, three different causes (stalled growth, meaningless project, personal life). Diagnose from the (provided) 1:1 notes and plan each response.

### [Lesson 62: Org design and team topologies](lessons/lesson-62-org-design.html)
**Goal:** Shape team boundaries deliberately — structure eats process for breakfast.
**Topics:** Conway's law used on purpose; Team Topologies vocabulary (stream-aligned, platform, enabling); team size and cognitive load limits; reorgs — when they're needed and their real cost; splitting a team well.
**Scenario:** Your 12-person team owns too much and is drowning. Design the split: boundaries, staffing, migration of ownership, and the announcement.

### [Lesson 63: Psychological safety and culture](lessons/lesson-63-psychological-safety.html)
**Goal:** Build the environment where problems surface early and people do their best work.
**Topics:** What psych safety is (and isn't — it's not comfort); Edmondson's research and Google's Project Aristotle; how leads destroy it in small ways (flinches, blame, interrupting); culture is what you tolerate; rituals that compound (demos, retros, failure parties); onboarding as culture transmission.
**Scenario:** In your new team, nobody asks questions in meetings and mistakes surface weeks late. Diagnose and write your 90-day culture plan.

---

## [Phase 12: Your Path](lessons/phase-12-your-path.html)

### [Lesson 64: Your first 90 days as a Lead](lessons/lesson-64-first-90-days-lead.html)
**Goal:** Start the tech-lead role deliberately instead of just "senior with extra meetings."
**Topics:** Listen-first (the 30-day no-big-changes rule); building your map (people, systems, bodies buried); early wins that build trust without stepping on toes; renegotiating your IC workload explicitly; the relationships to build in week one.
**Scenario:** Write your own 30/60/90 plan for becoming lead of your current team — specific names, systems, and risks.

### [Lesson 65: Your first 90 days as an EM](lessons/lesson-65-first-90-days-em.html)
**Goal:** Survive the bigger jump — especially if you now manage former peers.
**Topics:** The peer-to-boss transition (renegotiating every relationship, the friend problem); inheriting a team (don't trash the predecessor); 1:1s with everyone, first two weeks; finding the real performance picture; your new manager peer group; what to stop doing from your old job.
**Scenario:** You're now managing the team you were on — including your closest work friend and the senior who also wanted the job. First-week plan and both hard conversations.

### [Lesson 66: Building your support system](lessons/lesson-66-support-system.html)
**Goal:** Stop learning leadership alone — it's the slowest possible way.
**Topics:** Peer groups inside and outside (other leads, communities like LeadDev/Rands Leadership Slack); finding mentors who've done *your next* job; StaffEng and Manager Tools as ongoing curricula; therapy/coaching without stigma; keeping a decision journal.
**Scenario:** Design your personal board of advisors: the four seats, who could fill each (real names where possible), and the ask that gets each one.

### [Lesson 67: Choosing — and re-choosing — your path](lessons/lesson-67-choosing-your-path.html)
**Goal:** Make the Lead vs EM vs Staff decision with real information, and keep it revisable.
**Topics:** The honest differences in dailies (calendar screenshots, not job descriptions); which energizes you vs which flatters you; trying before committing (acting roles, internships); the pendulum is normal and healthy; exit criteria — how you'll know in a year; capstone: your written decision memo.
**Scenario:** Capstone — write the decision memo: your choice, your evidence from this course's 66 scenarios, your 12-month plan, and your re-evaluation date.

---

*All 67 lessons written = track complete. Update CLAUDE.md's index as each lands.*
