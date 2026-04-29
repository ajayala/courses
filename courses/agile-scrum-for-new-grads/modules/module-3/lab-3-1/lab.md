# Estimation, Velocity, and Sprint Planning

In this lab you will learn why software estimation is notoriously difficult, how story points work and why they beat hour-estimates, and how to participate in Planning Poker without feeling like a fraud. You will finish by building a realistic sprint plan from a backlog — the same skill your team will use every fortnight.

**Prerequisites:** Lab 2-2 completed; `agile-journal/backlog.md` exists with at least two user stories.

---

## Step 1 — Why Estimation Is Hard (and That Is Normal)

Before your first Sprint Planning, someone will ask you to estimate how long a ticket will take. This is one of the most anxiety-inducing moments for new graduates — because at university, you knew the scope of every assignment. In the industry, you rarely do.

Here is the honest truth: **experienced engineers are also bad at estimation in hours.** The research is clear: humans systematically underestimate effort, especially for unfamiliar work.

Common reasons estimates go wrong:

| Cause | Example |
|-------|---------|
| Hidden complexity | "Simple" API call requires auth token refresh logic |
| Unclear requirements | "Make it fast" — but how fast? |
| Unknown unknowns | The library you planned to use does not support your use case |
| Interruptions | Standups, reviews, deployment issues consume time |
| Optimism bias | "This will only take an hour" (famous last words) |

> **The right mindset:** Estimates are forecasts, not promises. A missed estimate is not a personal failure — it is information the team uses to improve future planning.

Create your sprint planning file:

```bash
cd agile-journal
cat > sprint-plan.md << 'EOF'
# Sprint Plan

Sprint number: [fill in]
Sprint goal: [One sentence: what does success look like for this Sprint?]
Sprint duration: [e.g. 2 weeks, 10 working days]
Team capacity: [Total story points the team can typically complete]

## Committed Stories
| Story | Points | Owner | Status |
|-------|--------|-------|--------|

## Stretch Goals (if time allows)
| Story | Points |
|-------|--------|

## Sprint Goal Definition
We will know this Sprint was successful when:
-
EOF
```

---

## Step 2 — Story Points: Relative Effort, Not Hours

A **story point** is a unit of relative effort — not a unit of time. Teams use story points because relative estimation is far more accurate than absolute estimation.

The key insight: it is easier to say *"this is twice as hard as that"* than to say *"this will take exactly 4.5 hours."*

Story points use the **Fibonacci sequence** (1, 2, 3, 5, 8, 13, 21) to reflect the fact that uncertainty grows with size:

```
1 point:  Very small, well understood, minimal risk
          Example: Change a button label

3 points: Small, mostly clear, low risk
          Example: Add a form field with validation

5 points: Medium, some unknowns, moderate risk
          Example: Integrate a third-party payment API

8 points: Large, significant unknowns, higher risk
          Example: Migrate authentication to a new provider

13 points: Very large — consider splitting this story
21 points: Too big for one Sprint — this is an epic, not a story
```

> **Story points are team-relative.** A "3" for your team might be a "5" for another team. What matters is *internal consistency* — that your 5s are consistently harder than your 3s over time. Do not compare story points across teams.

| Common mistake | Why it hurts |
|---------------|-------------|
| "1 point = 1 hour" | Destroys the benefit of relative estimation |
| Estimating alone without the team | Misses the collective knowledge of who will actually do the work |
| Never revisiting estimates | Velocity becomes useless if the calibration drifts |

---

## Step 3 — Planning Poker: How Estimation Works in Practice

**Planning Poker** is the most common estimation technique in Scrum. It prevents anchoring — the tendency to adjust your estimate toward the first number you hear.

How it works:

```
1. Product Owner reads out a user story
2. Team asks clarifying questions
3. Everyone privately picks a card (1, 2, 3, 5, 8, 13, 21, ?)
4. All cards are revealed simultaneously
5. If estimates match (or are close): accept the estimate
6. If estimates diverge: the highest and lowest estimators explain their reasoning
7. Re-vote. Repeat until consensus.
```

Simulate a round of Planning Poker on your backlog stories:

```bash
cat >> sprint-plan.md << 'EOF'

## Estimation Session Notes

### Story 1: [paste your story title]
Initial estimates: [e.g. 3, 3, 5, 8]
Discussion points: [what drove the disagreement?]
Final estimate: [agreed points]

### Story 2: [paste your story title]
Initial estimates:
Discussion points:
Final estimate:
EOF
```

> **The "?" card:** Use it when you genuinely cannot estimate because you do not understand the story. This is not a cop-out — it is information. If many people play "?", the story is not ready for Sprint and needs more refinement.

---

## Step 4 — Velocity: The Team's Delivery Rhythm

**Velocity** is the average number of story points a team completes per Sprint, measured over recent history (typically the last 3–5 Sprints).

```
Sprint 1: 22 points completed
Sprint 2: 18 points completed
Sprint 3: 25 points completed
Sprint 4: 20 points completed
Sprint 5: 23 points completed

Average velocity: (22 + 18 + 25 + 20 + 23) / 5 = 21.6 ≈ 22 points/Sprint
```

Velocity is used for **capacity planning** — deciding how much work to pull into the next Sprint. If your velocity is 22, you should not commit to 35 points of work.

**What velocity is NOT:**

| Misuse | Why it is harmful |
|--------|-----------------|
| A measure of individual productivity | Points are a team metric — never used to rank individuals |
| A target to be maximised | "Velocity inflation" — inflating estimates to look productive |
| Comparable across teams | A team doing "40 points" is not twice as productive as one doing "20" |
| Predictive without context | Illness, onboarding, and technical debt all reduce velocity |

> **As a new joiner:** Your first few Sprints will likely reduce team velocity slightly — this is expected and completely normal. Experienced teams factor in onboarding time when planning.

```bash
cat >> sprint-plan.md << 'EOF'

## Velocity Reference
Last 3 Sprint velocities: [e.g. 20, 22, 19]
Average velocity: [calculate]
Planned capacity this Sprint: [typically 80-90% of average velocity]
EOF
```

---

## Step 5 — Running Sprint Planning: What Actually Happens

With story points and velocity understood, here is what Sprint Planning looks like from the inside:

```
Before the meeting:
  - Product Owner has groomed the top of the backlog
  - Stories at the top are "ready": written, estimated, and small enough for one Sprint

During the meeting:
  1. Product Owner proposes the Sprint Goal (one sentence)
  2. Team reviews the top backlog items, asks questions
  3. Team pulls stories from the top of the backlog until they hit their velocity
  4. Each story is assigned (or self-selected) to a developer
  5. Team confirms: "Can we achieve the Sprint Goal with this work?"

After the meeting:
  - Sprint Backlog is visible on the team board (Jira, Linear, etc.)
  - Everyone knows what they are working on for the next two weeks
```

**Your job in Sprint Planning:**

```bash
cat >> sprint-plan.md << 'EOF'

## My committed stories this Sprint
| Story | Points | Notes |
|-------|--------|-------|
| [story title] | [points] | [any dependencies or risks I see] |
EOF
```

Fill in at least one story you will own this Sprint. Choose something from your `backlog.md`.

---

## Step 6 — Complete Your Sprint Plan

Pull together everything from this lab into a finished sprint plan. A real Sprint Plan is a lightweight document — a few decisions, clearly recorded.

```bash
cat sprint-plan.md
```

Your completed `sprint-plan.md` should contain:
- A Sprint Goal (one sentence)
- A committed stories table with point estimates
- Estimation session notes for at least two stories
- A velocity reference section
- At least one story assigned to you

Here is an example Sprint Goal to calibrate your style:

```markdown
Sprint goal: By the end of this Sprint, users can register, log in,
and reset their password via email — the complete basic authentication flow.

We will know this Sprint was successful when:
- All three authentication stories pass acceptance criteria
- The flows are live in the staging environment
- The Product Owner has signed off on the demo
```

In the final lab you will close the Scrum loop — learning how to run and contribute to a retrospective, give and receive feedback professionally, and set yourself up for a strong first 90 days on your new team.
