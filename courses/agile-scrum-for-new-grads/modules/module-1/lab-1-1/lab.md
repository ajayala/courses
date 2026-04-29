# From Waterfall to Agile — Why Teams Work This Way

In this lab you will learn why the software industry moved away from traditional "plan everything upfront" development and how Agile emerged as the answer. You will read the four core values of the Agile Manifesto, understand how they translate into daily work, and write a short personal reflection — the first artefact in your professional learning journal.

**Prerequisites:** No prior industry experience required. An open mind and a text editor.

---

## Step 1 — The Problem Agile Was Trying to Solve

Before Agile, most software was built using a process called **Waterfall** — a sequential approach borrowed from construction and manufacturing. Every phase had to be 100% complete before the next one began.

```
Requirements → Design → Build → Test → Deploy
    (months)     (months)  (months) (weeks) (one day)
```

This sounds logical. The problem: software is not a building. Requirements change. Users do not know what they want until they see it. Technology shifts. A year-long project planned in January is often solving the wrong problem by December.

A classic real-world failure pattern:

| Phase | What happened |
|-------|--------------|
| Requirements | Business team wrote a 200-page spec |
| Design | Architects designed a system for the spec |
| Build | Developers built what was designed |
| Test | Testers found the spec was wrong from the start |
| Deploy | A product no one needed shipped 18 months late |

> **University parallel:** Imagine being given a group project brief in Week 1, not being allowed to show your lecturer *anything* until Week 12, and only finding out then that you misunderstood the brief. That is Waterfall.

Create your workspace for this course:

```bash
mkdir agile-journal
cd agile-journal
```

---

## Step 2 — The Agile Manifesto: Four Values

In 2001, seventeen software practitioners met in Utah and wrote the **Agile Manifesto** — a single page that changed how the industry builds software. It defines four values, each stated as a preference rather than an absolute rule.

```
We are uncovering better ways of developing software by doing it
and helping others do it. Through this work we have come to value:

  Individuals and interactions   over   processes and tools
  Working software               over   comprehensive documentation
  Customer collaboration         over   contract negotiation
  Responding to change           over   following a plan

That is, while there is value in the items on the right,
we value the items on the left more.
```

Create a notes file to capture your understanding:

```bash
cat > agile-manifesto-reflection.md << 'EOF'
# Agile Manifesto Reflection

## Value 1: Individuals and interactions over processes and tools
My understanding:

## Value 2: Working software over comprehensive documentation
My understanding:

## Value 3: Customer collaboration over contract negotiation
My understanding:

## Value 4: Responding to change over following a plan
My understanding:
EOF
```

> **Key insight:** The manifesto does not say "ignore documentation" or "ignore plans." It says: when there is a conflict, people and working code win. Documentation and plans are still valuable — just not at the expense of the things on the left.

---

## Step 3 — The Four Values in Plain English

Reading the manifesto once is not enough. Here is what each value means day-to-day in a real team.

**Value 1 — Individuals and interactions over processes and tools**

Your team's daily conversations, pair programming sessions, and informal Slack messages matter more than any ticket system or process document. When a process is slowing the team down, the team should change the process — not blindly follow it.

**Value 2 — Working software over comprehensive documentation**

A feature that is live and helping users is worth more than a perfect 50-page design document describing a feature that has not been built yet. Documentation is written to serve the software — not the other way around.

**Value 3 — Customer collaboration over contract negotiation**

Instead of agreeing on a fixed scope at the start and fighting about changes later, Agile teams invite the customer (or product owner) into the process continuously. Requirements evolve, and that is fine.

**Value 4 — Responding to change over following a plan**

Plans are useful starting points, not sacred documents. When new information arrives — a market shift, a user interview, a technical discovery — the team updates the plan rather than defending it.

```bash
# Fill in your understanding for each value in agile-manifesto-reflection.md
# Open the file in your editor and add 1-2 sentences per value in your own words
```

---

## Step 4 — Agile, Scrum, Kanban — Clearing Up the Confusion

New joiners often use these terms interchangeably. They are not the same thing.

```
Agile
└── is a philosophy / set of values and principles
    │
    ├── Scrum
    │   └── a specific framework that implements Agile
    │       (sprints, roles, ceremonies)
    │
    ├── Kanban
    │   └── a visual flow-based method (continuous delivery, no sprints)
    │
    └── XP (Extreme Programming)
        └── engineering-focused practices (TDD, pair programming, CI)
```

| | Agile | Scrum | Kanban |
|--|-------|-------|--------|
| What it is | Philosophy | Framework | Method |
| Defines roles? | No | Yes (PO, SM, Dev) | No |
| Time-boxed iterations? | Not specified | Yes (Sprints) | No |
| Primary artifact | — | Sprint Backlog | Kanban Board |
| Best for | Any team | Product teams | Operations, support |

**The most important thing to know:** When your employer says "we do Agile," they almost certainly mean Scrum. This course covers Scrum — the framework you will encounter in your first job.

```bash
echo "## Terminology clarification" >> agile-manifesto-reflection.md
echo "" >> agile-manifesto-reflection.md
echo "Agile is the philosophy. Scrum is the framework my team will likely use." >> agile-manifesto-reflection.md
```

---

## Step 5 — What This Looks Like from a New Grad's Seat

Here is the honest picture of what Agile/Scrum feels like in your first weeks at a company.

**What you will experience:**
- A two-week rhythm of planning, building, and reviewing
- Short daily meetings (standups) where everyone shares what they are doing
- Work tracked on a digital board (Jira, Linear, Trello, GitHub Projects)
- Your work reviewed by teammates before it ships
- Regular team conversations about how to improve

**Common new-grad misconceptions:**

| Misconception | Reality |
|--------------|---------|
| "Agile means no planning" | Agile means *continuous* planning, not *no* planning |
| "Standups are status reports to management" | Standups are coordination between teammates |
| "I should know exactly what I'm doing" | Uncertainty is normal; asking questions is the job |
| "The process is fixed" | The team owns the process and changes it regularly |

> **You are not expected to know everything.** Agile teams are designed for learning. Asking "why do we do it this way?" is encouraged, not embarrassing.

```bash
echo "" >> agile-manifesto-reflection.md
echo "## My questions for my first week" >> agile-manifesto-reflection.md
echo "- " >> agile-manifesto-reflection.md
```

---

## Step 6 — Complete Your Reflection

Open `agile-manifesto-reflection.md` and complete every section. Write in your own words — not textbook definitions. Future-you reading this in six months should recognise your voice.

```bash
# Final structure your file should contain:
cat agile-manifesto-reflection.md
```

Your file should have all four value sections filled in, the terminology note, and at least one question for your first week. Here is a completed example for Value 1 to guide your style:

```markdown
## Value 1: Individuals and interactions over processes and tools
My understanding: This means talking to my teammates directly when something
is unclear, rather than filing a ticket and waiting. If a process is getting
in the way, we fix the process — we don't work around people to follow rules.
```

Save the file. In the next lab you will move from theory into the rhythm of a Scrum team — learning what happens in each ceremony and, critically, what your role is in each one as a new joiner.
