# C4 Architecture Diagrams with Mermaid

In this lab you will create C4 model diagrams using Mermaid — a four-level framework for communicating software architecture to different audiences. You will document a real-world SaaS platform from the 30,000-foot context view down to the component level, producing diagrams that are useful to executives, architects, and developers alike.

**Prerequisites:** Labs 3-1 complete. Familiarity with web application architecture (frontend, backend, database, external services).

---

## Step 1 — The C4 Model: Four Levels of Zoom

The C4 model (Context, Containers, Components, Code) was created by Simon Brown to address a common problem: architecture diagrams that are either too abstract to be useful or too detailed to be understood by non-engineers.

| Level | Audience | Answers |
|---|---|---|
| **L1 Context** | Everyone, including business stakeholders | What does the system do and who uses it? |
| **L2 Container** | Technical stakeholders | What deployable units make up the system? |
| **L3 Component** | Developers | What major building blocks are inside each container? |
| **L4 Code** | Developers | How is a specific component implemented? (Often a class diagram) |

> **Mermaid note:** Mermaid implements C4 diagrams using the `C4Context`, `C4Container`, `C4Component`, and `C4Dynamic` keywords. These are newer features — verify you are on Mermaid 10.3+ with `mmdc --version`.

---

## Step 2 — Level 1: System Context Diagram

The context diagram shows your system as a single box and everything it interacts with. It is the best starting point because it requires no technical knowledge to read.

Create `c4-context.mmd`:

```
C4Context
    title System Context — TaskFlow SaaS

    Person(user, "End User", "A team member who creates and manages tasks.")
    Person(admin, "Admin", "Manages team settings, billing, and integrations.")

    System(taskflow, "TaskFlow", "Cloud-based task and project management platform.")

    System_Ext(email, "Email Service", "Transactional email via SendGrid.")
    System_Ext(github, "GitHub", "Source code and issue tracking integration.")
    System_Ext(slack, "Slack", "Notification delivery via webhook.")
    System_Ext(stripe, "Stripe", "Subscription billing and invoicing.")

    Rel(user,      taskflow, "Uses", "HTTPS")
    Rel(admin,     taskflow, "Administers", "HTTPS")
    Rel(taskflow,  email,    "Sends emails via", "SMTP/API")
    Rel(taskflow,  github,   "Syncs issues with", "REST API")
    Rel(taskflow,  slack,    "Sends notifications to", "Webhook HTTPS")
    Rel(taskflow,  stripe,   "Processes payments via", "REST API")

    UpdateLayoutConfig($c4ShapeInRow="3", $c4BoundaryInRow="1")
```

```bash
mmdc -i c4-context.mmd -o c4-context.svg
```

> **Tip:** `System_Ext` (external system) renders with a grey background by default, distinguishing things your team owns from things you depend on. Use this consistently.

---

## Step 3 — Level 2: Container Diagram

Zoom into TaskFlow itself. A container is any separately deployable or runnable unit: a web app, mobile app, API, database, message queue, or serverless function.

Create `c4-containers.mmd`:

```
C4Container
    title Container Diagram — TaskFlow SaaS

    Person(user, "End User", "")
    Person(admin, "Admin", "")

    System_Boundary(taskflow, "TaskFlow Platform") {
        Container(spa,      "Web App",         "React / TypeScript", "Single-page application served via CDN.")
        Container(api,      "API Server",       "Go / Gin",           "REST and WebSocket API. Handles business logic.")
        Container(worker,   "Background Worker","Go",                 "Processes async jobs: emails, webhooks, imports.")
        ContainerDb(db,     "Primary Database", "PostgreSQL 16",      "Stores users, projects, tasks, and audit logs.")
        ContainerDb(cache,  "Cache",            "Redis",              "Session tokens, rate limit counters, task queues.")
        Container(storage,  "Object Storage",   "S3-compatible",      "File attachments and export archives.")
    }

    System_Ext(email,  "SendGrid", "")
    System_Ext(stripe, "Stripe",   "")
    System_Ext(slack,  "Slack",    "")

    Rel(user,    spa,     "Opens",              "HTTPS")
    Rel(admin,   spa,     "Opens",              "HTTPS")
    Rel(spa,     api,     "Calls",              "HTTPS / WSS")
    Rel(api,     db,      "Reads/writes",       "TCP 5432")
    Rel(api,     cache,   "Reads/writes",       "TCP 6379")
    Rel(api,     storage, "Uploads/downloads",  "HTTPS")
    Rel(api,     worker,  "Enqueues jobs",      "Redis queue")
    Rel(worker,  db,      "Reads/writes",       "TCP 5432")
    Rel(worker,  email,   "Sends via",          "HTTPS API")
    Rel(worker,  stripe,  "Webhooks to",        "HTTPS")
    Rel(worker,  slack,   "Posts to",           "HTTPS webhook")
```

```bash
mmdc -i c4-containers.mmd -o c4-containers.svg
```

---

## Step 4 — Level 3: Component Diagram

Zoom into the API Server container to show its internal structure. Components are the major logical groupings within a container — not every class or function, just the key building blocks.

Create `c4-components.mmd`:

```
C4Component
    title Component Diagram — API Server

    Container_Boundary(api, "API Server (Go / Gin)") {
        Component(router,    "HTTP Router",        "Gin",          "Route definitions and middleware chain.")
        Component(auth,      "Auth Middleware",    "JWT / Go",     "Validates bearer tokens, sets request context.")
        Component(taskCtrl,  "Task Controller",    "Go",           "CRUD handlers for tasks and subtasks.")
        Component(projCtrl,  "Project Controller", "Go",           "Handlers for projects, members, settings.")
        Component(wsHub,     "WebSocket Hub",      "Go goroutines","Broadcasts real-time updates to connected clients.")
        Component(taskSvc,   "Task Service",       "Go",           "Business logic: assignment rules, due date logic.")
        Component(notifySvc, "Notification Svc",   "Go",           "Decides which events trigger emails/Slack/webhooks.")
        Component(repo,      "Repository Layer",   "pgx / Go",     "Database access. One struct per aggregate root.")
    }

    ContainerDb(db,    "PostgreSQL", "")
    ContainerDb(cache, "Redis",      "")
    Container(worker,  "Worker",     "Go", "")

    Rel(router,    auth,      "Authenticates via", "")
    Rel(router,    taskCtrl,  "Routes to",         "")
    Rel(router,    projCtrl,  "Routes to",         "")
    Rel(router,    wsHub,     "Upgrades to WS",    "")
    Rel(taskCtrl,  taskSvc,   "Calls",             "")
    Rel(taskCtrl,  notifySvc, "Triggers",          "")
    Rel(taskSvc,   repo,      "Uses",              "")
    Rel(projCtrl,  repo,      "Uses",              "")
    Rel(notifySvc, worker,    "Enqueues job",      "Redis")
    Rel(repo,      db,        "Queries",           "TCP")
    Rel(auth,      cache,     "Checks token",      "TCP")
    Rel(wsHub,     cache,     "Pub/sub channel",   "TCP")
```

```bash
mmdc -i c4-components.mmd -o c4-components.svg
```

> **How detailed to go:** A component diagram should have 5–15 components. If you find yourself adding more, ask whether those belong in a separate component diagram for a different container, or whether you are going down to code level prematurely.

---

## Step 5 — C4 Dynamic Diagram: A Single Scenario

C4 Dynamic diagrams show how containers or components collaborate for a specific user story — think of them as a cross between C4 and a sequence diagram.

Create `c4-dynamic.mmd`:

```
C4Dynamic
    title Dynamic: User creates a task and assignee is notified

    Person(user,       "End User",           "")
    Container(spa,     "Web App",            "React", "")
    Container(api,     "API Server",         "Go",    "")
    Container(worker,  "Background Worker",  "Go",    "")
    ContainerDb(db,    "PostgreSQL",         "",      "")
    ContainerDb(cache, "Redis",              "",      "")
    System_Ext(slack,  "Slack",              "")

    RelIndex(1,  user,    spa,    "POST /tasks (form submit)")
    RelIndex(2,  spa,     api,    "POST /api/v1/tasks")
    RelIndex(3,  api,     db,     "INSERT INTO tasks")
    RelIndex(4,  db,      api,    "task_id = 9821")
    RelIndex(5,  api,     cache,  "LPUSH notify_queue job:9821")
    RelIndex(6,  api,     spa,    "201 { id: 9821 }")
    RelIndex(7,  spa,     user,   "Task created, optimistic UI update")
    RelIndex(8,  worker,  cache,  "BRPOP notify_queue")
    RelIndex(9,  worker,  db,     "SELECT assignee, project WHERE task_id=9821")
    RelIndex(10, worker,  slack,  "POST webhook — 'Assigned: Fix login bug'")
```

```bash
mmdc -i c4-dynamic.mmd -o c4-dynamic.svg
```

---

## Step 6 — Theming and Export for Presentations

For executive presentations or design-review documents, Mermaid's built-in themes and the `--cssFile` flag let you produce polished output.

Apply a dark theme:

```bash
mmdc -i c4-context.mmd -o c4-context-dark.svg -t dark
```

Export at higher resolution for print or slides:

```bash
mmdc -i c4-context.mmd -o c4-context-print.png -t neutral --width 2400
```

For a complete architecture document, generate all four C4 levels and combine them in a markdown file:

````markdown
# TaskFlow Architecture

## Context
```mermaid
<!-- paste c4-context.mmd content here -->
```

## Containers
```mermaid
<!-- paste c4-containers.mmd content here -->
```
````

```bash
mmdc -i architecture.md -o architecture-context.svg --input-format md
```

> **Decision log:** Keep a `decisions/` folder next to your diagrams with short ADR (Architecture Decision Record) files. Each diagram answers *what* the architecture is; ADRs explain *why* it was designed that way.

---

## Step 7 — Course Recap and Next Steps

Congratulations — you have completed the Mermaid Diagramming course. You can now:

| Diagram type | Use it for |
|---|---|
| Flowchart | Processes, business logic, decision trees |
| Sequence | API calls, event flows, multi-party protocols |
| ER | Database schemas, entity relationships |
| Class | OOP domain models, interfaces, generics |
| C4 Context | System boundaries and external dependencies |
| C4 Container | Deployable architecture overview |
| C4 Component | Internal structure of a single service |
| C4 Dynamic | One scenario through the architecture |

**Suggested next steps:**

- Add Mermaid diagrams to your team's existing README files — start with one ER or sequence diagram per service
- Set up a CI job using `mmdc` to regenerate diagrams on push and fail if the source changed but the output SVG was not committed
- Explore Mermaid's Gantt, Timeline, and Mindmap diagram types for planning and roadmap documentation
- Consider [Structurizr](https://structurizr.com) if you need a more formal C4 tooling workflow with workspace versioning

All diagrams in this course are plain text files. Keep them in your repository, review them in PRs, and they will always reflect reality.
