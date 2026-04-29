# Sprints, Ceremonies, and Your Role as a New Joiner

In this lab you will learn the four Scrum ceremonies — the regular meetings that give a team its rhythm — and understand exactly what is expected of you in each one. New graduates often find ceremonies uncomfortable at first because they feel put on the spot. This lab demystifies each meeting and gives you concrete things to say and do.

**Prerequisites:** Lab 1-1 completed; `agile-journal/` directory exists.

---

## Step 1 — What Is a Sprint?

A **Sprint** is a fixed-length period — almost always one or two weeks — during which the team builds a usable, potentially shippable increment of the product. Every Sprint follows the same shape:

```
Sprint (2 weeks)
├── Day 1:    Sprint Planning        (~2–4 hours)
├── Days 2–9: Development + Daily Standups (~15 min/day)
├── Day 10:   Sprint Review          (~1 hour)
└── Day 10:   Sprint Retrospective   (~1 hour)
```

At the end of each Sprint, the team should have something working that they can show to stakeholders. Not a document. Not a design. Working software.

> **Why fixed length?** A fixed Sprint length creates a predictable heartbeat. Everyone knows when planning happens, when demos happen, when to expect feedback. Predictability reduces anxiety — especially useful when you are new.

Create your ceremony reference file:

```bash
cd agile-journal
cat > sprint-ceremonies.md << 'EOF'
# Sprint Ceremonies Reference

Sprint length: [fill in your team's sprint length]
Team size: [fill in]

## Sprint Planning
When:
Purpose:
My role:
What to prepare:

## Daily Standup
When:
Purpose:
My role:
What to say:

## Sprint Review
When:
Purpose:
My role:
What to prepare:

## Sprint Retrospective
When:
Purpose:
My role:
What to say:
EOF
```

---

## Step 2 — Sprint Planning: Where the Work Gets Chosen

**Sprint Planning** happens at the start of every Sprint. The Product Owner presents the highest-priority items from the backlog, and the team decides what they can realistically complete in the upcoming Sprint.

```
Sprint Planning agenda:
1. Product Owner presents top backlog items (30 min)
2. Team discusses each item, asks questions (60 min)
3. Team estimates effort and commits to a Sprint goal (30–60 min)
4. Result: the Sprint Backlog — the list of work for this Sprint
```

**Your role as a new joiner:**

You are not expected to drive Sprint Planning. Your job in the first few Sprints is to:
- Listen and absorb how the team thinks about problems
- Ask "can you explain what this means?" — this is valuable, not embarrassing
- Volunteer for one or two tasks rather than staying silent

> **The most common new-grad mistake in planning:** Saying "yes" to too much because you want to impress people. It is far better to under-commit and over-deliver than the reverse. Your tech lead knows this — be honest about uncertainty.

Fill in the Sprint Planning section of your reference file:

```bash
# Open sprint-ceremonies.md and add:
# When: First day of the sprint (e.g. Monday morning)
# Purpose: Decide what the team will build this sprint
# My role: Ask questions, volunteer for 1-2 tasks, be honest about capacity
# What to prepare: Read the top backlog items the day before
```

---

## Step 3 — The Daily Standup: Your Daily Communication Touchpoint

The **Daily Standup** (also called Daily Scrum) is a 15-minute meeting held at the same time every day. It is *not* a status report to your manager — it is coordination between teammates.

The classic three questions:

```
1. What did I do yesterday that helped the team reach the Sprint Goal?
2. What will I do today toward the Sprint Goal?
3. Is there anything blocking me?
```

**What good standup answers look like:**

| Weak answer | Strong answer |
|-------------|--------------|
| "I was working on the login page" | "I finished the form validation; today I'll wire it to the API" |
| "Nothing blocking" (when there is) | "I'm stuck on the database timeout — can someone pair with me after standup?" |
| "I don't know what I'm doing" | "I'm picking up ticket PROJ-42, planning to start with the data model" |

**What standup is not for:**
- Detailed technical discussions (take those offline: "let's chat after standup")
- Reporting to management (your manager is a participant, not the audience)
- Problem-solving (flag the problem; solve it in a smaller group)

> **First-week tip:** If you genuinely have nothing to report (you just joined), say: *"I'm still onboarding — today I'll be setting up my dev environment and reading the architecture docs."* That is a perfectly valid standup answer.

```bash
echo "## My standup template" >> sprint-ceremonies.md
echo "Yesterday: " >> sprint-ceremonies.md
echo "Today: " >> sprint-ceremonies.md
echo "Blockers: " >> sprint-ceremonies.md
```

---

## Step 4 — Sprint Review: Demonstrating Your Work

The **Sprint Review** (sometimes called the Sprint Demo) happens at the end of the Sprint. The team shows what they built to stakeholders — the product owner, other teams, sometimes customers. Feedback is collected and fed back into the backlog.

```
Sprint Review agenda:
1. Product Owner recaps the Sprint Goal (5 min)
2. Team demos completed work — live, in a real environment (30–45 min)
3. Stakeholders ask questions and give feedback (15 min)
4. Product Owner updates the backlog based on feedback (after the meeting)
```

**The golden rule of Sprint Review: demo working software, not slides.**

Showing a PowerPoint of what you built is not a demo. Clicking through the actual feature in a real (or staging) environment is.

**Your role:**

If you completed a ticket during the Sprint, be ready to demo it. This means:
- Having the feature running in a staging or demo environment before the meeting
- Being able to explain *what* it does and *why* it matters (not just the technical how)
- Handling "what if we…" questions gracefully: "That's a great idea — I'll add it as a follow-up ticket"

> **Nervous about demoing?** Everyone is at first. Practise your demo once beforehand on your own machine. Know the happy path cold. If something breaks live, stay calm and say "let me show you this in a different way" — stakeholders respect composure.

---

## Step 5 — Sprint Retrospective: The Team's Learning Loop

The **Sprint Retrospective** (Retro) is the team's structured opportunity to improve how they work — not what they build, but *how* they build it. It happens after the Sprint Review, before the next Sprint starts.

The most common format: **Start, Stop, Continue**

```
Start:  Things we should begin doing (new practices, tools, habits)
Stop:   Things that are hurting us and should be dropped
Continue: Things that are working well and should be kept
```

**Your role as a new joiner:**

Many new graduates stay quiet in retros because they do not feel they have earned the right to suggest changes. This is wrong — your perspective as someone new is uniquely valuable. You see things that veterans have stopped noticing.

What you *can* say in your first retro:
- "I found it hard to know where to find X — could we document that better?" (Start)
- "The onboarding documentation was really helpful" (Continue)
- "The planning meeting ran long — I wasn't sure what was expected of me" (Start)

What to avoid in your first retro:
- Personal criticism of teammates ("Bob never reviews PRs")
- Sweeping architectural complaints unrelated to how the team works
- Silence — even one observation is a contribution

```bash
echo "" >> sprint-ceremonies.md
echo "## My retro prep template" >> sprint-ceremonies.md
echo "Start: " >> sprint-ceremonies.md
echo "Stop: " >> sprint-ceremonies.md
echo "Continue: " >> sprint-ceremonies.md
```

---

## Step 6 — Build Your Complete Ceremony Reference

Complete every section of `sprint-ceremonies.md` with your own words. Use the context from this lab plus your own honest answers.

```bash
cat sprint-ceremonies.md
```

Your completed file should have:
- All four ceremony sections with `When`, `Purpose`, `My role`, and `What to prepare/say` filled in
- Your standup template (Yesterday / Today / Blockers)
- Your retro prep template (Start / Stop / Continue)

Here is an example of a completed Daily Standup entry to guide your style:

```markdown
## Daily Standup
When: 9:30 AM every weekday
Purpose: Coordinate with teammates — flag blockers before they become problems
My role: Give a clear, brief update focused on the Sprint Goal
What to say:
  Yesterday: Finished the form validation for PROJ-42
  Today: Hooking the form up to the authentication API
  Blockers: None, but I may need a code review later today
```

In the next lab you will learn the skill that drives Sprint Planning: writing user stories — the unit of work in a Scrum backlog — in a way that is clear, testable, and useful to everyone on the team.
