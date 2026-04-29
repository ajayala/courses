# Big Decisions, Quality Attributes, and Trade-offs

In this lab you will learn what software architecture actually is — not the abstract, textbook definition, but the practical understanding of why certain decisions are treated as special, what drives them, and why every architectural choice is really a trade-off between competing concerns. By the end you will have a vocabulary and a mental framework that will help you read your team's codebase and conversations with new eyes.

**Prerequisites:** Some programming experience (any language). Familiarity with building a small application. No prior architecture knowledge required.

---

## Step 1 — What Architecture Is (and What It Is Not)

Software architecture is one of those terms that gets used loosely. Here is a precise definition that will serve you well:

> **Architecture is the set of significant design decisions that are costly or difficult to change later.**

The word *significant* does the work. Not every decision is architectural. Naming a variable is not architecture. Choosing which database engine your entire product will use — that is architecture. The test is: *how painful and expensive would it be to reverse this decision in six months?*

```
Architecture                          Design / Implementation
────────────────────────────────      ─────────────────────────────────────
"We will use a relational database"   "This table has these columns"
"Services communicate over HTTP"      "This endpoint returns JSON"
"We deploy as a monolith"             "This class has these methods"
"Authentication is centralised"       "This function validates a JWT"
```

> **University parallel:** In a large group project, deciding to use a shared Google Doc vs separate files vs a Git repository is an architectural decision — it affects how everyone works and is very hard to change halfway through. Deciding what font to use is not.

Create your architecture journal:

```bash
mkdir arch-journal && cd arch-journal
cat > quality-attributes.md << 'EOF'
# Architecture Journal

## What Architecture Is
My definition in my own words:

## What Makes a Decision "Architectural"
The test I will apply:

EOF
```

---

## Step 2 — The Real Drivers: Quality Attributes

Here is the insight most new graduates are missing: **architecture is not primarily driven by features — it is driven by quality attributes** (also called non-functional requirements or NFRs).

A quality attribute is a property of the *system as a whole* rather than a specific feature. The same feature list can be implemented with radically different architectures depending on which quality attributes matter most.

The most important quality attributes in industry:

| Attribute | Question it answers | Example constraint |
|-----------|--------------------|--------------------|
| **Scalability** | Can it handle more load? | Must support 10× current traffic within 12 months |
| **Availability** | Is it up when users need it? | 99.9% uptime (≤ 8.7 hours downtime/year) |
| **Performance** | How fast does it respond? | API responses under 200ms at p95 |
| **Security** | Can it be compromised? | PCI-DSS compliance required |
| **Maintainability** | How easy is it to change? | New features must not require changes to more than two modules |
| **Testability** | Can it be verified? | 80% automated test coverage required |
| **Deployability** | How fast can we ship? | Deploy to production in under 10 minutes |
| **Cost** | What does it cost to run? | Cloud bill must stay under $5k/month |

> **Key insight:** Two teams building a "chat application" might choose completely different architectures — one using a simple relational database, another using event streaming — because one team's top constraint is *development speed* and the other's is *message delivery guarantee*. Same features. Opposite trade-offs.

```bash
cat >> quality-attributes.md << 'EOF'

## Quality Attributes Reference

### The Eight Attributes I Need to Know
| Attribute | What it means | When it dominates architecture |
|-----------|--------------|-------------------------------|
| Scalability | | |
| Availability | | |
| Performance | | |
| Security | | |
| Maintainability | | |
| Testability | | |
| Deployability | | |
| Cost | | |

EOF
```

Fill in the last two columns in your own words.

---

## Step 3 — Architecture Is Always a Trade-off

No architecture is universally "best." Every choice that improves one quality attribute usually degrades another. This is the central truth of the discipline.

A few classic tensions:

```
Consistency  ←──────────────────────→  Availability
(every read returns the latest data)    (the system always responds)
         You can rarely have both fully

Performance  ←──────────────────────→  Security
(respond as fast as possible)           (verify, encrypt, audit everything)
         Encryption and auth take time

Simplicity   ←──────────────────────→  Scalability
(fewest moving parts)                   (distribute load across many parts)
         Distribution adds complexity

Developer    ←──────────────────────→  Operational
speed                                   reliability
(ship features fast)                    (everything tested, documented, monitored)
         Both require time and effort
```

The famous **CAP theorem** formalises one of these tensions. It states that a distributed system can guarantee at most two of three properties simultaneously:

```
         Consistency
            /\
           /  \
          /    \
         /      \
        /  Pick  \
       /   two    \
      ─────────────
Availability ── Partition Tolerance
```

| System type | C | A | P | Example |
|------------|---|---|---|---------|
| Traditional relational DB (single node) | ✓ | ✓ | ✗ | PostgreSQL (non-distributed) |
| Most NoSQL databases | ✗ | ✓ | ✓ | Cassandra, DynamoDB |
| Strongly consistent distributed DB | ✓ | ✗ | ✓ | HBase, Zookeeper |

> **You do not need to memorise the CAP theorem.** What you need to internalise is the *idea* behind it: in a distributed system, when the network fails (which it will), you must choose whether to return potentially stale data (sacrificing consistency) or return an error (sacrificing availability). There is no free lunch.

---

## Step 4 — The Spectrum of Architectural Styles

Architectures are not invented from scratch — they are chosen from a palette of well-understood styles, each suited to different combinations of quality attributes.

```
Simple ──────────────────────────────────────────────── Complex
                                                        
  Script/      Layered        Modular       Microservices
  Procedural   (N-tier)       Monolith      / Event-Driven
  
  ↑ Easy to    ↑ Industry     ↑ Best of     ↑ Best for
    start with   standard for   both worlds    large teams
               most apps                     and scale
```

| Style | Best when | Watch out for |
|-------|----------|--------------|
| **Layered / N-tier** | Standard business apps, small teams | Anemic domain model, layer leakage |
| **Modular monolith** | Growing teams, unclear service boundaries | Modules gradually coupling together |
| **Microservices** | Many teams, independent deployment needed | Network complexity, distributed transactions |
| **Event-driven** | Async workflows, audit trails, high throughput | Debugging is harder, eventual consistency |
| **Serverless** | Spiky traffic, pay-per-use cost model | Cold starts, vendor lock-in |

You will study the top two in depth in Module 2. For now, remember: **choose the simplest style that satisfies your quality attributes**. Over-engineering is one of the most common and costly architectural mistakes.

---

## Step 5 — The Architect's Role and Where You Fit

Architecture in real companies is rarely done by a single person with "Architect" in their title. It is a *responsibility* distributed across the team.

```
Staff / Principal Engineer  ──  System-wide decisions, cross-team standards
Senior Engineer             ──  Service-level decisions, pattern selection
Mid-level Engineer          ──  Module-level design, pattern application
Junior Engineer (you)       ──  Implementing agreed patterns, asking good questions
```

**What you should do as a new grad:**

- **Read** the architecture before writing code — ask "where does this fit?"
- **Ask why** the team made its current architectural choices — the answers reveal constraints you cannot see in the code
- **Never change** architectural patterns unilaterally — raise the conversation first
- **Notice smells** — code that seems to violate the architecture is worth flagging
- **Grow into it** — understanding the architecture deeply is one of the fastest paths to promotion

> **The question that impresses architects more than any other:** "What quality attributes was this design optimised for?" It shows you understand that architecture is intentional, not accidental.

```bash
cat >> quality-attributes.md << 'EOF'

## Questions to Ask on My First Week
- What are the top quality attributes this system is optimised for?
- What architectural style does the team use and why was it chosen?
- What is the biggest architectural decision the team regrets?
- Are there any architectural decision records (ADRs) I can read?

## Architectural Styles Summary
| Style | I would choose it when... |
|-------|--------------------------|
| Layered | |
| Modular monolith | |
| Microservices | |
| Event-driven | |
EOF
```

---

## Step 6 — Complete Your Quality Attributes Reference

Go back through your `quality-attributes.md` and complete every section. This is your reference document for the rest of the course.

```bash
cat quality-attributes.md
```

Your file should contain:
- Your own definition of software architecture
- The test you will apply to decide if something is "architectural"
- The quality attributes table with all columns filled in
- The architectural styles summary with your own "I would choose it when" notes
- The first-week questions list

Here is a sample entry for Scalability to guide your style:

```markdown
| Scalability | The system's ability to handle more users, data, or requests
               by adding resources — either vertically (bigger machine)
               or horizontally (more machines) | When the product expects
               rapid user growth, seasonal spikes, or data volume that
               will outgrow a single server |
```

In the next lab you will move from theory into the most common architectural pattern you will encounter in your first job: the layered architecture — and the reason almost every codebase you ever touch is organised the same way.
