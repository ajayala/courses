# Monolith vs Microservices — The Conversation Every Team Has

In this lab you will learn what a monolith actually is (hint: it is not automatically bad), what drove the industry toward microservices, what the real costs of microservices are that nobody mentions in the blog posts, and how the **modular monolith** often turns out to be the better answer for most teams. By the end you will be able to contribute intelligently to the conversation your team will inevitably have.

**Prerequisites:** Lab 2-1 completed; `arch-journal/layered-design.md` exists.

---

## Step 1 — What a Monolith Actually Is

A monolith is a system where all the functionality is deployed as a single unit. That is it. It is not a pejorative term, and it does not mean unstructured or poorly designed.

```
Monolith deployment:                 Microservices deployment:
────────────────────────             ─────────────────────────────────────────
┌──────────────────────┐             ┌─────────┐  ┌──────────┐  ┌──────────┐
│                      │             │  Users  │  │  Orders  │  │ Products │
│  Users  Orders  ...  │             │ Service │  │ Service  │  │ Service  │
│  Products  Payments  │             └────┬────┘  └────┬─────┘  └────┬─────┘
│                      │                  │             │              │
└──────────────────────┘             ─────┴─────────────┴──────────────┘
  One process, one deploy               Many processes, many deploys
```

Some of the most successful products in the world ran as monoliths for years: Stack Overflow still runs largely as a monolith and handles millions of requests daily. Shopify, Basecamp, and GitHub ran as monoliths far longer than the microservices discourse suggests they should have.

The monolith gets its bad reputation not because the pattern is flawed, but because **poorly structured** monoliths — ones with no internal boundaries, where everything calls everything — become unmanageable over time. This is a discipline failure, not a pattern failure.

Create your analysis document:

```bash
cd arch-journal
cat > monolith-vs-microservices.md << 'EOF'
# Monolith vs Microservices Analysis

## My Team's Hypothetical System
System: Online Bookshop (from Lab 2-1)
Current team size: 5 engineers
Expected team size in 2 years: [fill in a number and reason]
Top quality attributes: [pick 2-3 from your Lab 1-1 notes]

## Analysis
EOF
```

---

## Step 2 — The Problems That Motivated Microservices

Microservices did not emerge from a theoretical preference for distributed systems. They emerged as a solution to real, painful problems at large organisations — primarily around **team autonomy** and **independent deployability**.

The monolith pain points at scale:

```
Problem 1: Deployment coupling
  200 engineers all work in one codebase. To deploy Team A's feature,
  you must deploy Team B's untested changes too. One bad deploy takes
  everyone down at once.

Problem 2: Technology coupling  
  The entire company must use one language, one runtime, one database.
  The payments team cannot use a Rust library because the platform is Java.

Problem 3: Scaling coupling
  The product catalogue needs 10× more compute than the user profile service,
  but you can only scale the whole application — you cannot scale just the
  catalogue.

Problem 4: Organisational coupling
  Teams that should be independent block each other on shared code,
  shared databases, and shared deployment pipelines.
```

> **Notice the theme:** All four problems are most acute when the **team is large**. With five engineers, deployment coupling is a 15-minute coordination call. With 500 engineers across 40 teams, it is a 3-week release train with a dedicated programme manager.

```bash
cat >> monolith-vs-microservices.md << 'EOF'

## Monolith Pain Points at Scale
| Problem | At what team size does this become real? | Does our bookshop have this problem? |
|---------|----------------------------------------|--------------------------------------|
| Deployment coupling | | |
| Technology coupling | | |
| Scaling coupling | | |
| Organisational coupling | | |
EOF
```

---

## Step 3 — What Microservices Actually Are

A microservice is a small, independently deployable service that owns a single bounded capability and communicates with other services over a network (usually HTTP/REST or messaging).

```
Microservices: key properties
──────────────────────────────────────────────────────────────────
✓  Independently deployable (deploy Users without touching Orders)
✓  Independently scalable (run 10 Product replicas, 1 User replica)
✓  Owns its own data (each service has its own database)
✓  Communicates over network (HTTP, gRPC, message queue)
✓  Bounded by a business capability (not technical layer)
```

The phrase **"owns its own data"** is the most important and most violated. If Service A directly queries Service B's database, they are not microservices — they are a distributed monolith, which has all the costs of microservices with none of the benefits.

```
Correct:                              Incorrect (distributed monolith):
──────────────────────────────        ──────────────────────────────────
Orders Service                        Orders Service
  └── orders_db (private)               └── SELECT * FROM users_db.users
                                                          ↑
Users Service                         Users database (shared — no boundary)
  └── users_db (private)
  └── GET /users/{id} (API)
```

Microservices communicate via:

| Channel | When to use | Example |
|---------|------------|---------|
| **Synchronous HTTP/REST** | Request needs an immediate response | Checkout calls Payment service |
| **Asynchronous messaging** | Fire and forget, or eventual consistency is fine | Order placed → notify email service |
| **gRPC** | High-performance internal calls with typed contracts | Recommendation engine |

---

## Step 4 — The Real Costs Nobody Mentions in the Blog Posts

Every microservices success story is written by the companies that survived the transition. The failures are quiet.

**Operational complexity:**
```
Monolith:         1 service to deploy, monitor, log, restart, and debug
10 microservices: 10 services × all of the above
                  + service discovery
                  + load balancing
                  + distributed tracing (a request spans 4 services — which one is slow?)
                  + network failures that did not exist before
                  + eventual consistency problems that are very hard to debug
```

**Distributed transactions — the hardest problem:**

In a monolith, you wrap a database transaction around multiple operations — either all succeed or all roll back. In microservices, a single business operation often spans multiple services with separate databases. There is no global transaction. If Order service writes successfully but Payment service fails, you must implement **compensating transactions** (explicit undo logic) or **sagas** — complex patterns that require significant engineering investment.

**The cognitive overhead:**

A new engineer joining a monolith can `grep` the entire codebase. A new engineer joining a microservices system must understand which of 40 repositories to look in, how services discover each other, and how to run the system locally (usually involving Docker Compose with 12 services).

```bash
cat >> monolith-vs-microservices.md << 'EOF'

## Microservices Hidden Costs
| Cost | Impact on a 5-person team | Impact on a 50-person team |
|------|--------------------------|---------------------------|
| Operational complexity | | |
| Distributed transactions | | |
| Local development setup | | |
| Debugging across services | | |
| Network failure handling | | |
EOF
```

---

## Step 5 — The Modular Monolith: The Often-Better Answer

The modular monolith is a single deployable unit with strong internal module boundaries. It delivers the main benefit of a monolith (simplicity) while preserving the option to extract services later.

```
Modular Monolith:
┌───────────────────────────────────────────────────┐
│                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────┐ │
│  │   Orders    │  │   Users     │  │ Products  │ │
│  │   Module    │  │   Module    │  │  Module   │ │
│  │             │  │             │  │           │ │
│  │ (private DB │  │ (private DB │  │(private DB│ │
│  │  tables)    │  │  tables)    │  │  tables)  │ │
│  └──────┬──────┘  └──────┬──────┘  └─────┬─────┘ │
│         │  Public API    │  Public API    │       │
│         └───────────────►│◄──────────────┘       │
│                                                   │
└───────────────────────────────────────────────────┘
         One process, one deploy, clear boundaries
```

The key discipline: modules communicate only through **public interfaces** — never by directly accessing each other's internal classes or database tables. This is enforced by convention, code review, or tooling (package visibility rules in Java/Kotlin, `__all__` in Python).

When the business grows and a module needs independent scaling or deployment, it can be extracted into a microservice — and the extraction is clean because the boundary already exists.

| | Big Monolith | Modular Monolith | Microservices |
|--|-------------|-----------------|--------------|
| Team size | Any (small teams) | Small–medium | Medium–large |
| Deployment | Simple | Simple | Complex |
| Scaling | All-or-nothing | All-or-nothing | Per-service |
| Boundaries | Weak | Strong (enforced) | Strong (network) |
| Local dev | Easy | Easy | Complex |
| Extraction path | Hard | Easy | Already there |

> **Martin Fowler's advice:** "Don't start with microservices. Start with a monolith, keep it modular, and split out services when you have a proven need — scaling pain or team independence." The teams that succeed with microservices almost always started as a monolith first.

---

## Step 6 — Write Your Trade-off Analysis

Complete `monolith-vs-microservices.md` with a recommendation for the Online Bookshop. Use the tables you have been building throughout this lab and the quality attributes from Lab 1-1.

```bash
cat >> monolith-vs-microservices.md << 'EOF'

## My Recommendation for the Online Bookshop

### Recommended architecture:
(Monolith / Modular Monolith / Microservices — pick one)

### Reasoning:
The top quality attributes for this system are [X, Y, Z].
The team is [N] engineers, which means [coupling problem / no coupling problem yet].
The main risk of my choice is [risk].
We would revisit this decision when [trigger].

### What I would do differently if the team were 50 engineers:

### The one question I would ask before finalising this recommendation:

EOF
```

```bash
cat monolith-vs-microservices.md
```

In the next module you will learn how to communicate architectural decisions visually — drawing diagrams that actually help your audience understand the system, and writing the records that preserve *why* decisions were made long after the people who made them have moved on.
