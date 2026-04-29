# Architecture Decision Records and Your Role as a Junior Engineer

In this final lab you will learn to write **Architecture Decision Records** — the documents that preserve not just what was decided but *why*, capturing the context and trade-offs that everyone will have forgotten in six months. You will also learn how to spot common architectural anti-patterns, how to grow your architectural influence as a junior engineer, and leave with a personal learning roadmap.

**Prerequisites:** Lab 3-1 completed; `arch-journal/c4-diagram.md` exists.

---

## Step 1 — The Problem That ADRs Solve

Every codebase contains decisions that look strange to newcomers. Why is the database MySQL and not PostgreSQL? Why are emails sent synchronously instead of via a queue? Why does this service have its own authentication instead of using the central one?

Without documentation, there are only three possible answers:
1. Nobody knows — the person who decided has left
2. Someone half-remembers — "I think it was a performance thing, or maybe a licensing thing?"
3. Someone confidently gives the wrong reason — the most dangerous case

An **Architecture Decision Record (ADR)** is a short document that records a significant architectural decision: the context, the options considered, the choice made, and the consequences. It is written at the time of the decision and stored in the repository.

The result: six months later, a new engineer reads the ADR and immediately understands why the seemingly strange decision was actually reasonable given the constraints at the time.

```bash
cd arch-journal
mkdir decisions
cat > decisions/adr-001.md << 'EOF'
# ADR-001: [Short title of the decision]

**Date:** 2026-04-29
**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-XXX
**Deciders:** [Names or roles of people involved]

## Context
[What situation or problem prompted this decision?
Include constraints, team size, quality attributes at stake.]

## Decision
[The decision that was made. One clear sentence if possible.]

## Options Considered
### Option 1: [Name]
- Pros:
- Cons:

### Option 2: [Name]
- Pros:
- Cons:

### Option 3: [Name] (chosen)
- Pros:
- Cons:

## Consequences
### Positive
-

### Negative
-

### Risks
-

## Revisit When
[What change in circumstances would make us reconsider this decision?]
EOF
```

---

## Step 2 — What Makes a Good ADR

The goal of an ADR is to be useful to a reader who was not in the room when the decision was made. Here is the difference between a bad and a good ADR.

**Bad ADR:**

```markdown
# ADR-001: Use PostgreSQL

We decided to use PostgreSQL as our database.
It is a good database with many features.
```

This tells the reader nothing useful. Why not MySQL? Why not MongoDB? What were the constraints?

**Good ADR:**

```markdown
# ADR-001: Use PostgreSQL as the primary database

**Date:** 2026-04-29
**Status:** Accepted

## Context
We need a relational database for user, order, and product data.
The team of four engineers all have PostgreSQL experience.
We need JSONB support for flexible product attributes without a schema migration
for every new product category. Budget limits us to a single managed database
instance for the first year (AWS RDS).

## Decision
Use PostgreSQL 16 on AWS RDS.

## Options Considered
### MySQL 8
- Pros: Widely known, good managed options, slightly faster for simple reads
- Cons: Weaker JSONB support, no native array types, team less familiar

### MongoDB
- Pros: Flexible schema, good for document data
- Cons: No ACID transactions across documents (needed for order + inventory),
        team has no experience, adds operational complexity

### PostgreSQL 16 (chosen)
- Pros: Full ACID, excellent JSONB, array types, team expertise, AWS RDS support
- Cons: Schema migrations required for structural changes

## Consequences
### Positive
- Team is immediately productive — no learning curve
- JSONB handles flexible product attributes without over-engineering a schema
- Full transactions cover the order + inventory update atomicity requirement

### Negative
- Vertical scaling only for the first year (single RDS instance)
- Schema migrations add friction as the product evolves

## Revisit When
If write throughput exceeds 5,000 inserts/second sustained (RDS bottleneck),
or if product attribute flexibility requires a true document store.
```

Fill in your `adr-001.md` with a real decision from the Online Bookshop — for example, the choice of message queue (RabbitMQ vs others), the choice of caching strategy, or the monolith-vs-microservices decision from Lab 2-2.

---

## Step 3 — Architectural Anti-Patterns to Watch For

As a new engineer reading an existing codebase, these patterns are warning signs worth flagging to your team.

**The Big Ball of Mud**

No discernible structure. Everything calls everything. Changes in one part unpredictably break other parts. Usually the result of years of growth with no architectural attention.

```
Symptom: You cannot name the architecture. Every module imports every other module.
Question to ask: "Is there an architecture diagram? Why do these modules depend on each other?"
```

**The Distributed Monolith**

Looks like microservices (many services, many repos) but services share a database, deploy together, or call each other synchronously in long chains. All the complexity of microservices with none of the independence.

```
Symptom: Services cannot be deployed independently. "We have to deploy A, B, and C together."
Question to ask: "Can we deploy this service without touching any other?"
```

**The Leaky Abstraction**

A layer's implementation details are visible to layers that should not know about them. Database entity classes used directly in API responses. SQL in controllers. HTTP client code in business logic services.

```
Symptom: The Presentation layer imports ORM models. The business logic layer imports `requests`.
Question to ask: "What would have to change if we switched databases / HTTP clients?"
```

**Over-Engineering**

A system built for complexity it does not yet have — twelve microservices for an application with fifty users, event sourcing for a simple CRUD app, an abstraction layer for a use case that has never varied.

```
Symptom: The team spends more time maintaining infrastructure than building features.
Question to ask: "What specific problem does this complexity solve today?"
```

```bash
cat >> decisions/adr-001.md << 'EOF'

---

# Anti-Patterns I Will Watch For

## Big Ball of Mud
How to spot it:

## Distributed Monolith
How to spot it:

## Leaky Abstraction
How to spot it:

## Over-Engineering
How to spot it:
EOF
```

---

## Step 4 — How to Influence Architecture as a Junior Engineer

New graduates sometimes feel they have no voice in architectural discussions. This is a misconception — you have significant influence, just through different channels than senior engineers.

**What junior engineers can do:**

```
Ask "why" questions in design reviews
  → "Why did we choose this approach over X?"
  → Forces the team to articulate reasoning, which surfaces unstated assumptions

Flag anomalies you notice
  → "I noticed this module is calling the database directly from the controller —
     is that intentional or a pattern we want to fix?"
  → Fresh eyes see violations that veterans have normalised

Write the ADR for the decision you just observed
  → Volunteer to document the outcome of a design discussion
  → This forces you to understand the decision deeply and makes you visible

Build the prototype that tests the decision
  → "Can I spike this alternative approach over the next sprint?"
  → Junior engineers are often given spikes; use them to contribute to architectural evidence

Read widely and bring ideas back
  → Share an article about a pattern the team has not considered
  → "I read about X — would that apply to our problem with Y?"
```

**What junior engineers should NOT do:**

```
Do NOT make architectural changes unilaterally
  → Changing a pattern without discussion undermines trust and can break assumptions
  → Always raise it first: "I'm thinking about changing X to Y — is that consistent with our approach?"

Do NOT dismiss the existing architecture as "wrong"
  → It was usually right for the constraints that existed when it was built
  → The constraints may have changed — but you need to understand them before critiquing

Do NOT propose rewrites
  → "Let's rewrite this in [new technology]" is the most reliable way to lose credibility early
  → Start by understanding why the existing choices were made
```

---

## Step 5 — Write Your First Complete ADR

Complete `adr-001.md` with a full decision record for the Online Bookshop. Choose the most interesting architectural decision from the course — the database choice, the messaging approach, or the monolith-vs-modular decision from Lab 2-2.

```bash
cat decisions/adr-001.md
```

A complete ADR has:
- A meaningful title (not just "Use X")
- Context that explains the forces and constraints
- At least two options considered with honest pros and cons
- A clear decision statement
- Positive and negative consequences
- A "revisit when" trigger condition

---

## Step 6 — Your Architecture Learning Roadmap

Software architecture is a discipline you grow into over years. Here is an honest roadmap for your first three years.

```bash
cat > architecture-roadmap.md << 'EOF'
# My Architecture Learning Roadmap

## Year 1 — Understand What Exists
Goals:
- [ ] Read and understand the architecture of every system I contribute to
- [ ] Identify which architectural style each codebase uses
- [ ] Learn to read C4 diagrams and ask questions about ones I don't understand
- [ ] Notice and flag (not fix) architectural smells
- [ ] Write my first ADR

Books to read this year:
- [ ] "Clean Architecture" — Robert C. Martin (focus on Part 3: Design Principles)
- [ ] "Designing Data-Intensive Applications" — Martin Kleppmann (Ch. 1-3 to start)

## Year 2 — Contribute to Design Decisions
Goals:
- [ ] Participate actively in design reviews with prepared questions
- [ ] Lead the design of a small module or service
- [ ] Understand the quality attributes driving my team's key decisions
- [ ] Write multiple ADRs that the team adopts

Books to read this year:
- [ ] "Building Microservices" — Sam Newman
- [ ] "Domain-Driven Design Distilled" — Vaughn Vernon

## Year 3 — Lead Architectural Thinking for a Scope
Goals:
- [ ] Own the architecture for a product area or service group
- [ ] Run design reviews and facilitate trade-off discussions
- [ ] Mentor others in reading and writing architecture documents
- [ ] Propose and justify a significant architectural change

## The Question I Will Ask Myself Every Quarter
"Do I understand *why* the systems I work on are structured the way they are?"
If the answer is no — find out.

## One Architecture Anti-Pattern I Have Already Seen
(Fill this in from your current experience or the course):

EOF
```

```bash
cat architecture-roadmap.md
```

Congratulations — you have completed the Software Architecture for New Graduates course. You now understand what architecture is and why it matters, can recognise and apply the most common patterns, make and document trade-off decisions, draw C4 diagrams that actually communicate, and write ADRs that will be useful to your team for years. Architecture is a career-long discipline — every system you build and every design decision you observe is adding to your intuition. Start paying attention to it from day one.
