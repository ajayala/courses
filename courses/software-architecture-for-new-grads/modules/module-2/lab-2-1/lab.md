# Layered Architecture — The Pattern Hiding in Every Codebase

In this lab you will learn the layered architecture pattern — the most prevalent pattern in the industry — understand why it exists, where it breaks down, and design a layered system yourself. You will also encounter MVC and the Repository pattern: the two concrete expressions of layered thinking you will see in almost every framework and codebase you touch in your first years.

**Prerequisites:** Lab 1-1 completed; `arch-journal/` directory exists. Ability to read code in at least one language (examples use Python-like pseudocode).

---

## Step 1 — Why Layers Exist: Separation of Concerns

Imagine a codebase where database queries, business logic, and UI rendering are all mixed together in the same functions. Every time the UI changes, you risk breaking database logic. Every time the database schema changes, you must hunt through the entire codebase for affected code.

**Separation of concerns** is the principle that each part of a system should have one reason to change. Layers enforce this by grouping code that changes for the same reason and keeping code that changes for different reasons apart.

```
Without layers (spaghetti):               With layers (lasagne):
──────────────────────────                ─────────────────────────────────
function handleRequest(req) {             Presentation Layer
  user = db.query(                          └── handles HTTP, renders views
    "SELECT * FROM users                         changes when: UI changes
     WHERE id = " + req.id)
  if user.role == "admin":                Business Logic Layer
    tax = calculateTax(user)                └── applies rules and decisions
    db.execute(                                  changes when: business changes
      "INSERT INTO orders ...")
  return renderHtml(user, tax)            Data Access Layer
}                                           └── queries the database
                                                 changes when: DB changes
```

The layered model is not just about organisation — it is about **controlling the direction of change** so that a UI redesign does not force you to rewrite your database queries.

Create your design document:

```bash
cd arch-journal
cat > layered-design.md << 'EOF'
# Layered Architecture Design

## System: Online Bookshop

### Layers
| Layer | Responsibility | Changes when |
|-------|---------------|-------------|
| Presentation | | |
| Business Logic (Domain) | | |
| Data Access | | |
| Database | | |

### Dependency Rule
(Which layers may call which other layers?)

EOF
```

---

## Step 2 — The Classic Three-Tier Model

The most common form of layered architecture in web applications is the **three-tier** (or N-tier) model:

```
┌──────────────────────────────────────────┐
│           Presentation Tier              │  ← HTTP handlers, REST controllers,
│         (Web / API Layer)                │    HTML templates, JSON serialisation
└────────────────────┬─────────────────────┘
                     │ calls
┌────────────────────▼─────────────────────┐
│           Business Logic Tier            │  ← Domain rules, calculations,
│           (Application Layer)            │    workflows, validation
└────────────────────┬─────────────────────┘
                     │ calls
┌────────────────────▼─────────────────────┐
│            Data Access Tier              │  ← SQL queries, ORM models,
│          (Persistence Layer)             │    file I/O, external APIs
└────────────────────┬─────────────────────┘
                     │ reads/writes
┌────────────────────▼─────────────────────┐
│               Database                   │  ← PostgreSQL, MySQL, MongoDB, etc.
└──────────────────────────────────────────┘
```

**The dependency rule:** Dependencies point downward only. The Presentation layer knows about the Business Logic layer, but the Business Logic layer knows *nothing* about the Presentation layer. This means you can swap a REST API for a GraphQL API without touching a single line of business logic.

```
Allowed:      Presentation  →  Business Logic  →  Data Access
Forbidden:    Data Access   →  Business Logic
              Business Logic →  Presentation
```

> **Why this matters:** When you join a team and read code, look for violations of the dependency rule — business logic that calls HTTP libraries, or data access code that knows about JSON formatting. These "layer leaks" are a reliable indicator of maintenance problems ahead.

Fill in the layers table in `layered-design.md` for the Online Bookshop system.

---

## Step 3 — MVC: Layered Architecture in a Framework

**Model-View-Controller (MVC)** is the layered pattern baked into nearly every web framework — Rails, Django, Spring MVC, Laravel, ASP.NET MVC. If you understand MVC, you understand the structure of most web applications you will ever work on.

```
┌──────────┐   HTTP Request    ┌────────────┐
│          │ ────────────────► │            │
│  Client  │                  │ Controller │  ← Presentation layer
│          │ ◄──────────────── │            │    Receives request, calls services,
└──────────┘   HTTP Response   └─────┬──────┘    returns response
                                     │ calls
                               ┌─────▼──────┐
                               │            │
                               │   Model    │  ← Business logic + Data layer
                               │            │    Domain entities, validation rules,
                               └─────┬──────┘    database queries (via ORM)
                                     │ data
                               ┌─────▼──────┐
                               │            │
                               │    View    │  ← Presentation layer (output)
                               │            │    HTML template, JSON serialiser
                               └────────────┘
```

| Component | Responsibility | Framework examples |
|-----------|---------------|-------------------|
| **Controller** | Receive request, validate input, call services, return response | Django views, Spring @Controller, Rails controllers |
| **Model** | Domain entities and their rules, database mapping | Django models, JPA entities, ActiveRecord |
| **View** | Render output (HTML, JSON, XML) | Django templates, Thymeleaf, Jinja2 |

> **Common confusion:** In modern REST APIs, there is often no "View" in the traditional sense — the controller serialises the model directly to JSON. Many frameworks call this "Model-View-Controller" anyway. The important thing is the separation between "handle the request" and "apply the business rules."

```bash
cat >> layered-design.md << 'EOF'

## MVC Mapping for the Bookshop

### A user searches for books by author:

Controller (handles request):

Model (applies logic):

View / Serialiser (formats response):

EOF
```

---

## Step 4 — The Repository Pattern: Hiding the Database

The **Repository pattern** is the standard way to implement the Data Access layer. It hides all database details behind an interface, so the Business Logic layer never knows (or cares) whether you are using PostgreSQL, MongoDB, or a flat file.

```python
# Without Repository — business logic leaks into SQL
class OrderService:
    def place_order(self, user_id, book_id):
        # Business logic mixed with raw SQL — tightly coupled to PostgreSQL
        db.execute("INSERT INTO orders (user_id, book_id, status) VALUES (?, ?, 'pending')", user_id, book_id)
        db.execute("UPDATE inventory SET quantity = quantity - 1 WHERE book_id = ?", book_id)


# With Repository — business logic is clean and testable
class OrderService:
    def __init__(self, order_repo: OrderRepository, inventory_repo: InventoryRepository):
        self.order_repo = order_repo
        self.inventory_repo = inventory_repo

    def place_order(self, user_id: int, book_id: int) -> Order:
        inventory = self.inventory_repo.find_by_book(book_id)
        if inventory.quantity < 1:
            raise OutOfStockError(book_id)
        order = Order(user_id=user_id, book_id=book_id, status="pending")
        self.order_repo.save(order)
        self.inventory_repo.decrement(book_id)
        return order
```

Why this matters:
- `OrderService` can be tested without a real database — just pass a fake repository
- Switching from PostgreSQL to MongoDB requires changing only the repository implementation
- The business rule ("cannot order out-of-stock books") is visible in the service, not buried in SQL

```bash
cat >> layered-design.md << 'EOF'

## Repository Pattern for the Bookshop

### Repositories needed:
- BookRepository: methods needed:
- OrderRepository: methods needed:
- UserRepository: methods needed:

### What the service layer looks like without knowing SQL:

EOF
```

---

## Step 5 — When Layered Architecture Breaks Down

Layered architecture is not a silver bullet. It has well-known failure modes that you will encounter.

**The Anemic Domain Model**

When business logic leaks out of the Model into Service classes, leaving the Model as nothing but getters and setters, you have an *anemic domain model* — the most common layered architecture smell.

```python
# Anemic: Order knows nothing about its own rules
class Order:
    def __init__(self):
        self.status = "pending"
        self.items = []

class OrderService:  # All logic ends up here — this class bloats indefinitely
    def ship_order(self, order):
        if order.status != "paid":
            raise Exception("Cannot ship unpaid order")
        order.status = "shipped"

# Rich model: Order enforces its own invariants
class Order:
    def ship(self):
        if self.status != "paid":
            raise InvalidStateError("Cannot ship unpaid order")
        self.status = "shipped"
```

**Strict layering can mean unnecessary indirection**

In a large codebase, calling a simple "get user by ID" requires a Controller → Service → Repository chain even when there is no business logic involved. Some teams relax strict layering for read-only queries and enforce it only for writes.

**Layers do not map cleanly to teams**

When fifty engineers all work in the same "business logic layer," that layer becomes a bottleneck. This is one of the pressures that eventually pushes large organisations toward microservices — not because the pattern is broken, but because the team structure has outgrown it.

> **Conway's Law:** "Any organisation that designs a system will produce a design whose structure is a copy of the organisation's communication structure." Architecture and team structure are inseparable — keep this in mind when evaluating any architectural recommendation.

---

## Step 6 — Design a Complete Layered System

Complete the `layered-design.md` document for the Online Bookshop system. It should cover:

- The layers table with all four rows filled in
- The dependency rule written in your own words
- The MVC mapping for a search request
- The repository list with methods
- One example of anemic vs rich model for the `Book` entity

```bash
cat >> layered-design.md << 'EOF'

## Anemic vs Rich Model Example: Book

### Anemic Book (anti-pattern):

### Rich Book (preferred):

## My Verdict
The layered pattern is the right choice for this bookshop because:

It would stop being the right choice if:

EOF
```

```bash
cat layered-design.md
```

In the next lab you will zoom out from a single-application architecture to the larger question that dominates team conversations in growing companies: should this system be a monolith, microservices, or something in between?
