# Sequence Diagrams: Capturing Interactions Over Time

In this lab you will learn Mermaid sequence diagrams — the go-to format for documenting how systems, services, and actors exchange messages over time. You will model an OAuth2 authorisation code flow, use activation bars to show processing time, and add notes and loops to express conditional behaviour.

**Prerequisites:** Lab 2-1 complete. Understanding of basic HTTP request/response cycles.

---

## Step 1 — Sequence Diagram Syntax Fundamentals

A sequence diagram reads top to bottom. Participants are declared at the top, then messages flow between them on numbered lines.

```
sequenceDiagram
    participant Browser
    participant Server
    participant Database

    Browser->>Server: GET /dashboard
    Server->>Database: SELECT user WHERE id=42
    Database-->>Server: User record
    Server-->>Browser: 200 OK, HTML
```

Arrow types:

| Syntax | Meaning |
|---|---|
| `->>` | Solid arrow (request / call) |
| `-->>` | Dashed arrow (response / return) |
| `-)` | Async message (open arrowhead) |
| `--)`| Async response |
| `-x` | Lost message (cross at end) |

> **Convention:** Use solid arrows for requests and dashed for responses. This mirrors the UML convention and makes message direction immediately obvious.

Create `basic-seq.mmd` with the snippet above and render it:

```bash
mmdc -i basic-seq.mmd -o basic-seq.svg
```

---

## Step 2 — Participants, Aliases, and Actors

You can use `actor` instead of `participant` to render a stick-figure icon — useful to distinguish humans from systems:

```
sequenceDiagram
    actor User as End User
    participant FE as Frontend App
    participant BE as Backend API
    participant Auth as Auth Service

    User->>FE: Click "Log In"
    FE->>BE: POST /login
    BE->>Auth: Validate credentials
    Auth-->>BE: JWT token
    BE-->>FE: 200 { token }
    FE-->>User: Show dashboard
```

The `as` keyword gives a display label separate from the identifier used in the rest of the diagram. Use short identifiers (`FE`, `BE`) so lines stay readable, but give them meaningful labels.

```bash
mmdc -i aliases.mmd -o aliases.svg
```

---

## Step 3 — Model an OAuth2 Authorisation Code Flow

OAuth2 is a multi-party flow with redirects, code exchanges, and token issuance — exactly the kind of interaction that is hard to describe in prose but trivial to follow as a sequence diagram.

Create `oauth2.mmd`:

```
sequenceDiagram
    actor User
    participant Browser
    participant App as Client App
    participant AuthServer as Auth Server
    participant ResourceServer as Resource API

    User->>Browser: Click "Sign in with GitHub"
    Browser->>App: GET /auth/start
    App-->>Browser: Redirect to AuthServer + state + code_challenge

    Browser->>AuthServer: GET /authorize?client_id=...
    AuthServer-->>Browser: Show login page
    User->>Browser: Enter credentials
    Browser->>AuthServer: POST credentials
    AuthServer-->>Browser: Redirect to App /callback?code=AUTH_CODE

    Browser->>App: GET /callback?code=AUTH_CODE
    App->>AuthServer: POST /token (code + code_verifier)
    AuthServer-->>App: { access_token, refresh_token }

    App->>ResourceServer: GET /user (Bearer access_token)
    ResourceServer-->>App: User profile JSON
    App-->>Browser: Render profile page
    Browser-->>User: Logged-in dashboard
```

```bash
mmdc -i oauth2.mmd -o oauth2.svg
```

> **Why model this?** Reviewing this diagram during a security design review immediately reveals that the code verifier must be sent — if it's missing, you've implemented OAuth2 without PKCE and are vulnerable to authorization code interception.

---

## Step 4 — Activation Bars, Loops, and Alt Blocks

Activation bars show when a participant is actively processing. Conditional and loop blocks add control flow to the diagram.

```
sequenceDiagram
    participant Client
    participant API
    participant Cache
    participant DB

    Client->>+API: GET /products
    API->>+Cache: Lookup products
    alt Cache hit
        Cache-->>-API: Cached data
    else Cache miss
        Cache-->>API: nil
        API->>+DB: SELECT * FROM products
        DB-->>-API: Rows
        API->>Cache: SET products (TTL 60s)
    end
    API-->>-Client: 200 JSON

    loop Retry on 503
        Client->>API: GET /products
        API-->>Client: 503 Service Unavailable
    end
```

| Block | Syntax | Purpose |
|---|---|---|
| Activation | `+` / `-` on arrow | Show active processing span |
| Conditional | `alt … else … end` | Branch based on condition |
| Loop | `loop Description … end` | Repeated interaction |
| Optional | `opt Description … end` | Interaction that may not happen |
| Parallel | `par … and … end` | Concurrent interactions |

```bash
mmdc -i control-flow.mmd -o control-flow.svg
```

---

## Step 5 — Notes and Critical Sections

Notes add explanatory text alongside the diagram without adding new participants. Critical sections highlight where atomicity or careful ordering matters.

```
sequenceDiagram
    participant Producer
    participant Broker as Message Broker
    participant Consumer

    Note over Producer,Broker: Messages must be acknowledged<br/>before the offset advances

    Producer->>+Broker: Publish event (key=user.123)
    Broker-->>-Producer: ACK

    critical Consumer must process exactly once
        Broker->>+Consumer: Deliver event
        Consumer->>Consumer: Process & write to DB
        Consumer-->>-Broker: ACK offset
    option Duplicate delivery
        Broker->>Consumer: Redeliver event
        Consumer->>Consumer: Idempotency check — skip
    end

    Note right of Consumer: Idempotency key stored<br/>for 24 hours
```

```bash
mmdc -i notes.mmd -o notes.svg
```

> **Tip:** Use `Note over A,B` to span a note across multiple participants. Use `Note left of` / `Note right of` for a note attached to one side of a single participant.

---

## Step 6 — Auto-number Messages

For documentation shared in reviews or RFCs, numbering each message makes it easy to refer to a specific interaction in discussion.

Add `autonumber` as the first line of any sequence diagram:

```
sequenceDiagram
    autonumber
    participant Client
    participant Server

    Client->>Server: Connect
    Server-->>Client: 101 Switching Protocols
    Client->>Server: Subscribe to channel
    Server-->>Client: Subscription confirmed
    Server-)Client: Event pushed (async)
    Server-)Client: Event pushed (async)
    Client->>Server: Unsubscribe
    Server-->>Client: OK
```

```bash
mmdc -i numbered.mmd -o numbered.svg
```

Each arrow is now prefixed with an incrementing number. When a reviewer says "step 5 looks wrong", everyone is looking at the same message.

---

## Step 7 — Checkpoint and What's Next

You can now model any multi-party interaction using Mermaid sequence diagrams:

- Participant and actor declarations with display aliases
- Solid/dashed arrows, activation bars, and async messages
- `alt`, `loop`, `opt`, `par`, and `critical` control flow blocks
- Notes and auto-numbering for documentation-quality output

In Module 3 you will shift from runtime behaviour to **structural models** — entity-relationship diagrams for databases, class diagrams for object-oriented designs, and finally C4 architecture diagrams for high-level system overviews.
