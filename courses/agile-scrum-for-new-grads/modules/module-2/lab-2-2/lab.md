# The Product Backlog — Writing User Stories That Work

In this lab you will learn what a Product Backlog is, how to write user stories using the industry-standard format, and how to define acceptance criteria that make a story verifiable. These are skills you will use every Sprint — and writing good stories is something many experienced developers still struggle with.

**Prerequisites:** Lab 2-1 completed; `agile-journal/sprint-ceremonies.md` exists.

---

## Step 1 — What Is the Product Backlog?

The **Product Backlog** is an ordered list of everything that might be done to improve the product. It is owned by the **Product Owner** (PO) — a role responsible for maximising the value of the product and ensuring the team builds the right things.

```
Product Backlog (ordered by priority — most important at the top)
├── [1] User can reset their password via email
├── [2] Dashboard loads in under 2 seconds on mobile
├── [3] Admin can export user data as CSV
├── [4] Add dark mode toggle
└── ... (dozens or hundreds more items)
```

Key properties of a healthy backlog:

| Property | What it means |
|---------|--------------|
| **Ordered** | The most valuable items are at the top — always |
| **Estimated** | Top items have effort estimates; lower items may not yet |
  | **Emergent** | The backlog changes constantly as the team learns |
| **Transparent** | Anyone on the team can read it |

> **You are not expected to manage the backlog.** That is the Product Owner's job. But you will read it, pull items from it, ask questions about it, and sometimes write or refine items in it — so understanding how it works is essential.

Create your backlog practice file:

```bash
cd agile-journal
cat > backlog.md << 'EOF'
# Practice Backlog

## Epics
<!-- Large themes of work go here -->

## User Stories
<!-- Individual stories go here -->

## Definition of Done
<!-- Conditions that must be true before ANY story is marked complete -->
- [ ] Code reviewed by at least one teammate
- [ ] Automated tests written and passing
- [ ] Feature tested in staging environment
- [ ] Product Owner has accepted the story
EOF
```

---

## Step 2 — The User Story Format

A **User Story** is a short description of a feature from the perspective of the person who will use it. The standard format is:

```
As a <type of user>,
I want <some goal or action>,
So that <some benefit or reason>.
```

This format forces three important things:
1. **Who** is affected — not "the system" but a real person with a goal
2. **What** they want to do — the capability
3. **Why** — the business value, which helps the team make good decisions

**Good examples:**

```
As a registered customer,
I want to receive a password reset email,
So that I can regain access to my account if I forget my password.
```

```
As a team manager,
I want to export my team's timesheets as a CSV,
So that I can import them into our payroll system without manual data entry.
```

**Bad examples and why:**

| Bad story | Problem |
|-----------|---------|
| "As a user, I want to click a button" | No "so that" — no value, no decisions can be made |
| "Add dark mode to the settings page" | Not a story — no who or why, reads like a task |
| "The system should validate email addresses" | Written from the system's perspective, not the user's |
| "As a user, I want the app to be fast" | Too vague — what counts as "fast"? Not testable |

```bash
# Add two user stories to your backlog
cat >> backlog.md << 'EOF'

### Story 1
As a [user type],
I want [goal],
So that [benefit].

### Story 2
As a [user type],
I want [goal],
So that [benefit].
EOF
```

---

## Step 3 — Epics: The Container for Related Stories

An **Epic** is a large body of work that cannot fit in a single Sprint. It groups related user stories together under a common theme.

```
Epic: User Authentication
├── Story: As a new visitor, I want to register with email and password
├── Story: As a registered user, I want to log in with my credentials
├── Story: As a logged-in user, I want to reset my password via email
├── Story: As a logged-in user, I want to enable two-factor authentication
└── Story: As an admin, I want to deactivate a user account
```

Think of an epic like a university module: it takes a whole semester (many Sprints), but each lecture (story) delivers specific, learnable content.

Epics are useful for:
- Planning at a high level before you know all the details
- Communicating scope to stakeholders ("we are working on the Authentication epic")
- Grouping work in your issue tracker (Jira, Linear, GitHub Projects)

```bash
# Add an epic to your backlog
sed -i 's/## Epics/## Epics\n\n### Epic: [Name your epic]\nGoal: [One sentence on what this epic achieves]\nStories: [List the stories that belong here]/' backlog.md
```

---

## Step 4 — Acceptance Criteria: Making Stories Testable

A user story without acceptance criteria is like an exam question without a mark scheme — nobody knows what "done" looks like.

**Acceptance criteria** are the specific conditions that must be true for the story to be considered complete. They are written by the Product Owner (often with developer input) before the story enters a Sprint.

The most readable format is **Given / When / Then** (Gherkin syntax):

```
Story: As a registered customer, I want to receive a password reset email,
       so that I can regain access if I forget my password.

Acceptance Criteria:

Given the user is on the login page and has entered their email address,
When they click "Forgot password",
Then they receive an email within 2 minutes containing a one-time reset link.

Given the user clicks the reset link,
When the link is less than 24 hours old,
Then they are taken to a page where they can set a new password.

Given the user clicks the reset link,
When the link is more than 24 hours old,
Then they see an error message: "This link has expired. Please request a new one."
```

> **Why this format matters for you as a developer:** Acceptance criteria are your contract with the Product Owner. When all the criteria pass, the story is done — no ambiguity. If the PO asks for more, it becomes a new story (scope change), not a silent extension of the current one.

```bash
# Add acceptance criteria to Story 1 in your backlog
cat >> backlog.md << 'EOF'

#### Acceptance Criteria for Story 1
Given [starting condition],
When [action taken],
Then [expected outcome].
EOF
```

---

## Step 5 — The Definition of Done vs Acceptance Criteria

New graduates often confuse these two. They are different and both matter.

| | Definition of Done (DoD) | Acceptance Criteria |
|--|--------------------------|---------------------|
| **Scope** | Applies to *every* story | Applies to *one specific* story |
| **Who writes it** | The whole team (agreed in retro) | The Product Owner |
| **What it covers** | Quality standards: tests, review, deployment | Functional behaviour of this feature |
| **Example** | "All code must be reviewed" | "User receives email within 2 minutes" |

The DoD is like your university's submission rules: every assignment must be submitted by the deadline, formatted correctly, with a cover sheet. Acceptance criteria are the specific marking rubric for *this* assignment.

Update your backlog's Definition of Done to make it more realistic:

```bash
# Replace the placeholder DoD with a real one
cat >> backlog.md << 'EOF'

## Updated Definition of Done
- [ ] Code reviewed and approved by at least one teammate
- [ ] Unit tests written for new logic (>80% coverage on changed files)
- [ ] Integration/end-to-end tests updated or added where relevant
- [ ] Feature deployed to staging and smoke-tested
- [ ] Acceptance criteria verified with the Product Owner
- [ ] No new warnings introduced in the CI pipeline
- [ ] Documentation updated if the change affects public APIs or user-facing behaviour
EOF
```

---

## Step 6 — Write a Complete Backlog Entry

Put it all together. A complete, professional backlog entry looks like this:

```markdown
## Story: Password Reset via Email
**Epic:** User Authentication
**Priority:** High
**Estimate:** 3 points

**As a** registered customer,
**I want** to receive a password reset email,
**So that** I can regain access to my account if I forget my password.

**Acceptance Criteria:**

Given the user is on the login page,
When they click "Forgot password" and enter a valid email,
Then they receive a reset link email within 2 minutes.

Given the user clicks a valid reset link (under 24 hours old),
When they submit a new password,
Then their password is updated and they are logged in automatically.

Given the user clicks an expired reset link (over 24 hours old),
When the page loads,
Then they see: "This link has expired. Please request a new one."
```

Write one complete entry in your `backlog.md` file using this format — choose any feature from a product you use daily (a shopping site, a messaging app, a music player).

```bash
cat backlog.md
```

Your backlog should now contain: the Definition of Done, at least one Epic, two User Stories, and at least one story with Given/When/Then acceptance criteria.

In the next lab you will take these stories into Sprint Planning — learning how to estimate effort, understand velocity, and decide what the team can realistically commit to in a Sprint.
