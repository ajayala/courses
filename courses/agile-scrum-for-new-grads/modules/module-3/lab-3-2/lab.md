# Retrospectives and Thriving in Your First 90 Days

In this final lab you will learn how to run and contribute to a Sprint Retrospective, give and receive feedback professionally without burning bridges, and build a personal 30-60-90 day plan that sets you up to grow quickly on your new team. The skills in this lab are less about Scrum and more about being the kind of engineer people want to work with.

**Prerequisites:** Lab 3-1 completed; `agile-journal/sprint-plan.md` exists.

---

## Step 1 — The Retrospective: The Team's Learning Loop

The **Sprint Retrospective** is the most undervalued ceremony in Scrum. Teams that skip or rush it accumulate process debt the same way teams that skip refactoring accumulate technical debt.

A Retro has three phases:

```
Phase 1 — Set the stage (5 min)
  Remind everyone: this is a safe space. The goal is improvement, not blame.
  A good facilitator (usually the Scrum Master) may read the Prime Directive:
  "Regardless of what we discover, we understand and truly believe that
  everyone did the best job they could, given what they knew at the time."

Phase 2 — Gather data (20 min)
  Everyone adds cards/notes to three columns:
  ┌─────────────┬──────────────┬────────────────┐
  │    START    │     STOP     │    CONTINUE    │
  │ Things we   │ Things that  │ Things working │
  │ should try  │ hurt us      │ well — keep!   │
  └─────────────┴──────────────┴────────────────┘

Phase 3 — Decide on actions (20 min)
  The team votes on which items to act on this Sprint.
  Each action item gets an owner and a deadline.
  Unacted-on items from the last Retro are reviewed first.
```

Create your retrospective document:

```bash
cd agile-journal
cat > retrospective.md << 'EOF'
# Sprint Retrospective

Sprint number: [fill in]
Date: [fill in]
Facilitator: [Scrum Master's name or "rotating"]
Participants: [list team members]

## Start
Things we should begin doing:
-

## Stop
Things that are hurting us and should stop:
-

## Continue
Things that are working well:
-

## Action Items
| Action | Owner | Due |
|--------|-------|-----|
|        |       |     |
EOF
```

---

## Step 2 — How to Contribute as a New Joiner

New graduates are often the *quietest* people in a Retro, and the most valuable contributors — because they see the team clearly, before familiarity makes the problems invisible.

**How to frame observations without sounding critical:**

| Instead of... | Try... |
|--------------|--------|
| "The code review process is too slow" | "I wasn't sure how to get a review — could we document the process?" |
| "The planning meeting is a waste of time" | "I found planning hard to follow — could we share the agenda beforehand?" |
| "Nobody explained the system to me" | "I'd love a regular chat with a senior to ask questions — would that work?" |

**The rule:** Talk about the *system*, not the *person*. "PRs sit unreviewed for days" is a process observation. "Bob never reviews PRs" is a personal accusation.

```bash
# Add observations to your retrospective
cat >> retrospective.md << 'EOF'

## My Observations (as a new joiner)
Things that surprised me this Sprint:
-

Things I found hard to navigate:
-

Things that genuinely helped me:
-
EOF
```

---

## Step 3 — Giving and Receiving Feedback

Retros are not the only place feedback happens. Code reviews, one-on-ones, and ad-hoc Slack messages are all feedback channels. Handling them well is a professional skill.

**Giving feedback:**

Use the **SBI model** (Situation, Behaviour, Impact):

```
"In yesterday's PR review [Situation],
you asked clarifying questions before making suggestions [Behaviour] —
that helped me understand the reasoning rather than just following instructions [Impact]."
```

```
"In this morning's planning [Situation],
the technical constraints were not mentioned upfront [Behaviour],
which meant we estimated the story too low and now it feels risky [Impact]."
```

**Receiving feedback:**

New graduates sometimes react defensively to code review feedback. This is natural — your code feels personal. But treating review comments as attacks rather than collaboration will slow your growth.

| Unhelpful response | Helpful response |
|-------------------|-----------------|
| "That's how I was taught to do it" | "Interesting — can you explain why you prefer the other approach?" |
| Ignoring a comment without reply | "Acknowledged — I've updated the PR" |
| Reopening closed debates | "I see your point. I'll try it your way and see how it reads." |

> **The most career-accelerating thing you can do:** When someone gives you feedback, say "thank you" and act on it. Engineers who receive feedback well get more feedback — and grow faster.

---

## Step 4 — The Three Scrum Roles: Reading the Room

Understanding who does what prevents a lot of frustration in your first months.

```
┌─────────────────────────────────────────────────────┐
│                   SCRUM TEAM                        │
│                                                     │
│  Product Owner       Scrum Master    Developers     │
│  ─────────────       ────────────    ──────────     │
│  Owns the backlog    Serves the      Build the      │
│  Prioritises value   team process    product        │
│  Says WHAT & WHY     Removes blocks  Say HOW & WHEN │
│  Accepts stories     Facilitates     Estimate work  │
│                      ceremonies      Own quality    │
└─────────────────────────────────────────────────────┘
```

**Common tensions new grads encounter:**

| Scenario | What is actually happening |
|----------|--------------------------|
| PO keeps changing requirements | PO's job is to respond to learning — this is *correct* Agile behaviour |
| SM wants more process | SM is trying to protect the team's ability to improve |
| Tech lead pushes back on PO's priorities | Dev team has a right to flag technical risks — this is healthy |
| PO asks for an estimate at a specific hour count | Diplomatically explain story points — "I can give you a range" |

> **A key boundary:** The Product Owner decides *what* to build. The development team decides *how* to build it. Neither can dictate to the other. If your PO is telling you which code to write (not what feature to build), that is a process problem to raise in the Retro.

---

## Step 5 — Your First 30-60-90 Day Plan

Agile is not just for software — it applies to your own growth too. A **30-60-90 day plan** is a personal Sprint plan for your first three months.

```bash
cat > 90-day-plan.md << 'EOF'
# My First 90 Days — Personal Plan

## First 30 Days: Listen and Learn
Goals:
- [ ] Complete onboarding and set up dev environment
- [ ] Ship at least one small ticket end-to-end
- [ ] Learn the team's branching and deployment workflow
- [ ] Attend all four Scrum ceremonies and take notes
- [ ] Schedule 1:1s with every team member

Success looks like:
I can independently pick up a small ticket, implement it, get it reviewed, and deploy it.

## Days 31-60: Contribute Actively
Goals:
- [ ] Own 2-3 stories per Sprint without needing to ask for task breakdown
- [ ] Give at least one code review per Sprint
- [ ] Raise one observation in a Retrospective
- [ ] Identify one area of the codebase I want to understand more deeply

Success looks like:
My teammates trust me to deliver what I commit to in Sprint Planning.

## Days 61-90: Start Leading
Goals:
- [ ] Mentor a teammate on something I have learned
- [ ] Write or refine a user story with the Product Owner
- [ ] Propose one process improvement in a Retrospective
- [ ] Begin a personal learning goal (certification, side project, reading)

Success looks like:
I am bringing new ideas to the team, not just executing existing ones.

## Things I want to ask my manager
-
-

## Things I am worried about
-
-
EOF
```

---

## Step 6 — Write Your Personal Retrospective

Close the loop on this course the same way a Scrum team closes a Sprint — with a retrospective on your own learning.

```bash
cat >> retrospective.md << 'EOF'

## Personal Learning Retrospective (End of Course)

### Start
Things I want to start doing in my new role:
-

### Stop
Habits or assumptions from university that I should leave behind:
-

### Continue
Skills or approaches I have that will serve me well:
-

### My one commitment
The single most important thing I will do differently because of this course:

EOF
```

Fill in every section honestly. This is for you, not for assessment. The engineers who grow fastest are the ones who reflect regularly — and act on what they find.

```bash
cat retrospective.md
cat 90-day-plan.md
```

**Congratulations.** You now understand the Agile mindset, the Scrum framework, how to write user stories with acceptance criteria, how to estimate and plan a Sprint, and how to thrive in a retrospective culture. These are skills your first team will assume you lack — which means you already have an advantage. Use it well.
