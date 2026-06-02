# Flowcharts: Modelling Processes and Decisions

In this lab you will master Mermaid flowcharts — the most versatile diagram type. You will model a real-world user authentication flow, use subgraphs to group related steps, apply custom styling, and learn the full range of node and edge shapes available to you.

**Prerequisites:** Lab 1-1 complete. `mmdc` installed. A text editor and terminal.

---

## Step 1 — Flowchart Direction and Node Shapes

Mermaid flowcharts start with `graph` (or `flowchart`) followed by a direction keyword:

| Keyword | Meaning |
|---|---|
| `TD` / `TB` | Top to bottom |
| `BT` | Bottom to top |
| `LR` | Left to right |
| `RL` | Right to left |

Node shapes communicate meaning at a glance:

```
flowchart LR
    A[Rectangle — process step]
    B(Rounded — start / end)
    C{Diamond — decision}
    D[(Cylinder — database)]
    E((Circle — connector))
    F>Asymmetric — document]
    G[/Parallelogram — input\/output/]
```

Create `shapes.mmd` with the content above and render it:

```bash
mmdc -i shapes.mmd -o shapes.svg
```

> **Convention:** Use rounded rectangles `()` for start/end nodes, diamonds `{}` for decisions, and plain rectangles `[]` for process steps. Consistent shapes make diagrams immediately readable.

---

## Step 2 — Edge Types and Labels

Edges connect nodes. Mermaid supports several visual styles:

```
flowchart TD
    A -->  B
    C --- D
    E --> |label on arrow| F
    G -.-> H
    I ==> J
    K --o L
    M --x N
```

| Syntax | Renders as |
|---|---|
| `-->` | Arrow |
| `---` | Line, no arrow |
| `-->|text|` | Arrow with label |
| `-.->` | Dashed arrow |
| `==>` | Thick arrow |
| `--o` | Circle at end |
| `--x` | Cross at end |

Save this as `edges.mmd` and render it to confirm you understand each style before moving on.

```bash
mmdc -i edges.mmd -o edges.svg
```

---

## Step 3 — Model a User Authentication Flow

Now build something real. Create `auth-flow.mmd`:

```
flowchart TD
    Start([User visits login page]) --> EnterCreds[Enter email & password]
    EnterCreds --> Validate{Credentials valid?}

    Validate -- No --> IncCount[Increment failure counter]
    IncCount --> Locked{Account locked?}
    Locked -- Yes --> ShowLock[Show lockout message]
    Locked -- No  --> ShowError[Show error, prompt retry]
    ShowError --> EnterCreds

    Validate -- Yes --> MFA{MFA enabled?}
    MFA -- No  --> CreateSession[Create session token]
    MFA -- Yes --> SendCode[Send OTP to device]
    SendCode --> EnterOTP[User enters OTP]
    EnterOTP --> OTPValid{OTP valid?}
    OTPValid -- No  --> ShowOTPErr[Show OTP error]
    ShowOTPErr --> EnterOTP
    OTPValid -- Yes --> CreateSession

    CreateSession --> Dashboard([Redirect to dashboard])
    ShowLock --> SupportLink([Contact support])
```

Render it:

```bash
mmdc -i auth-flow.mmd -o auth-flow.svg
```

> **Why this matters:** Mapping out a flow like this *before* writing code surfaces edge cases early — lockout logic, OTP retry behaviour — that are easy to miss when you jump straight into implementation.

---

## Step 4 — Group Nodes with Subgraphs

Large flowcharts become hard to read when all nodes float at the same level. Subgraphs let you group related steps into labelled boxes.

Extend `auth-flow.mmd` by wrapping sections in subgraph blocks. Create `auth-flow-sub.mmd`:

```
flowchart TD
    subgraph Input ["User Input"]
        Start([Visit login page]) --> EnterCreds[Enter credentials]
    end

    subgraph Validation ["Server Validation"]
        EnterCreds --> Validate{Credentials valid?}
        Validate -- Yes --> MFA{MFA enabled?}
        Validate -- No  --> IncCount[Increment failure counter]
    end

    subgraph MFAFlow ["Multi-Factor Auth"]
        MFA -- Yes --> SendCode[Send OTP]
        SendCode --> EnterOTP[Enter OTP]
        EnterOTP --> OTPValid{OTP valid?}
    end

    subgraph Session ["Session"]
        MFA -- No --> CreateSession[Create session]
        OTPValid -- Yes --> CreateSession
        CreateSession --> Dashboard([Dashboard])
    end

    IncCount --> Locked{Account locked?}
    Locked -- Yes --> ShowLock([Lockout page])
    Locked -- No  --> EnterCreds
    OTPValid -- No --> EnterOTP
```

```bash
mmdc -i auth-flow-sub.mmd -o auth-flow-sub.svg
```

---

## Step 5 — Apply Styles and Classes

You can style individual nodes or define reusable CSS classes.

Create `auth-styled.mmd`:

```
flowchart TD
    A([Start]) --> B{Valid?}
    B -- Yes --> C[Process]
    B -- No  --> D[Error]
    C --> E([End])

    style A fill:#4ade80,stroke:#16a34a,color:#000
    style D fill:#f87171,stroke:#dc2626,color:#fff
    style E fill:#60a5fa,stroke:#2563eb,color:#fff

    classDef decision fill:#fbbf24,stroke:#d97706,color:#000
    class B decision
```

```bash
mmdc -i auth-styled.mmd -o auth-styled.svg
```

| Directive | Purpose |
|---|---|
| `style <id> ...` | Inline style for one node |
| `classDef <name> ...` | Define a reusable style class |
| `class <id> <name>` | Apply a class to a node |

> **Tip:** Limit colour use to signal meaning — green for success paths, red for errors, yellow for decisions. Decorative colour creates visual noise.

---

## Step 6 — Click Interactions and Tooltips

Mermaid flowcharts can include clickable nodes when rendered in an HTML context (not PNG/SVG file export):

```
flowchart LR
    A[Documentation] --> B[API Reference]
    A --> C[Getting Started]

    click A "https://mermaid.js.org" "Open Mermaid docs" _blank
    click B callback "Show API tooltip"
```

For exported files, use tooltips instead — they appear on hover in SVG output rendered in a browser:

```
flowchart LR
    DB[(Users DB)]
    API[Auth Service]
    API --> DB

    click DB href "https://example.com/schema" "View schema"
```

This technique is useful for architecture diagrams where each box links to its service's own documentation.

---

## Step 7 — Checkpoint and What's Next

You have now built production-quality flowcharts using:

- All standard node shapes and edge types
- Subgraphs for logical grouping
- CSS-based node styling and reusable class definitions
- A realistic multi-step authentication model

In the next lab you will learn **sequence diagrams** — a completely different diagram type that shows *who* communicates with *whom* and *in what order*, which is ideal for documenting API calls, event-driven flows, and microservice interactions.
