# Modernising Code with Java 9–17 Language Features

In this lab you will refactor Java 8 idioms into their modern equivalents using language features introduced across Java 9 through 17. You will replace verbose boilerplate with records, simplify conditional logic with pattern matching and switch expressions, and use text blocks to make multi-line string literals readable.

**Prerequisites:** Lab 2-1 complete. The project must compile clean on Java 17.

---

## Step 1 — Replace DTOs and Value Objects with Records

Java 16 introduced records — immutable data carriers that eliminate the boilerplate of constructors, getters, `equals`, `hashCode`, and `toString`. They are ideal for DTOs, API responses, and domain value objects.

**Before (Java 8):**

```java
public class UserResponse {
    private final Long id;
    private final String name;
    private final String email;

    public UserResponse(Long id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }

    public Long getId()     { return id; }
    public String getName() { return name; }
    public String getEmail(){ return email; }

    @Override public boolean equals(Object o) { /* 10 lines */ }
    @Override public int hashCode()           { /* 5 lines */  }
    @Override public String toString()        { /* 5 lines */  }
}
```

**After (Java 16+):**

```java
public record UserResponse(Long id, String name, String email) {}
```

Records also work as Spring controller response types with Jackson — Jackson 2.12+ supports records natively (no `@JsonCreator` needed).

> **Limitation:** Records are implicitly final and immutable. Do not use them for JPA entities — Hibernate requires mutable classes with a no-arg constructor. Use records for the service/presentation layer only.

---

## Step 2 — Use Pattern Matching instanceof

Java 8 requires a redundant cast after `instanceof`:

```java
// Java 8
if (shape instanceof Circle) {
    Circle c = (Circle) shape;
    return Math.PI * c.radius() * c.radius();
}
```

Java 16 pattern matching eliminates the cast:

```java
// Java 16+
if (shape instanceof Circle c) {
    return Math.PI * c.radius() * c.radius();
}
```

The binding variable `c` is scoped to the `if` block and is guaranteed non-null. This removes an entire category of `ClassCastException` bugs.

Find all `instanceof` + cast patterns in the codebase:

```bash
grep -rn "instanceof" src/main/java/ | grep -v "//.*instanceof" | tee instanceof-candidates.txt
wc -l instanceof-candidates.txt
```

Refactor each match. A method that previously looked like this:

```java
public double area(Shape shape) {
    if (shape instanceof Circle) {
        return Math.PI * ((Circle) shape).radius() * ((Circle) shape).radius();
    } else if (shape instanceof Rectangle) {
        Rectangle r = (Rectangle) shape;
        return r.width() * r.height();
    }
    throw new IllegalArgumentException("Unknown shape: " + shape);
}
```

Becomes:

```java
public double area(Shape shape) {
    if (shape instanceof Circle c)       return Math.PI * c.radius() * c.radius();
    if (shape instanceof Rectangle r)    return r.width() * r.height();
    throw new IllegalArgumentException("Unknown shape: " + shape);
}
```

---

## Step 3 — Replace if-instanceof Chains with Switch Expressions

Java 14 introduced switch expressions (finalised in 14) and Java 21 adds pattern matching to switch. On Java 17, you can use switch expressions with sealed types as a preview, but the most portable improvement is switching from verbose if-else to switch expressions for enum-based dispatch.

**Before (Java 8 — statement form, fall-through risk):**

```java
String label;
switch (status) {
    case PENDING:
        label = "Awaiting review";
        break;
    case APPROVED:
        label = "Approved";
        break;
    case REJECTED:
        label = "Rejected";
        break;
    default:
        throw new IllegalStateException("Unknown: " + status);
}
```

**After (Java 14+ switch expression):**

```java
String label = switch (status) {
    case PENDING   -> "Awaiting review";
    case APPROVED  -> "Approved";
    case REJECTED  -> "Rejected";
};
```

The arrow form has no fall-through and each arm is an expression. When all enum cases are covered, the `default` branch is unnecessary and the compiler enforces exhaustiveness.

Find switch statements to upgrade:

```bash
grep -rn "switch\s*(" src/main/java/ | tee switch-candidates.txt
```

---

## Step 4 — Use Text Blocks for Multi-line Strings

Java 15 introduced text blocks, eliminating the `\n` + concatenation pattern common in Java 8 code for SQL queries, JSON templates, and HTML snippets.

**Before (Java 8):**

```java
String query = "SELECT u.id, u.name, u.email\n" +
               "FROM users u\n" +
               "JOIN accounts a ON a.user_id = u.id\n" +
               "WHERE a.status = 'ACTIVE'\n" +
               "  AND u.created_at > :since";
```

**After (Java 15+ text block):**

```java
String query = """
        SELECT u.id, u.name, u.email
        FROM users u
        JOIN accounts a ON a.user_id = u.id
        WHERE a.status = 'ACTIVE'
          AND u.created_at > :since
        """;
```

The closing `"""` controls indentation stripping — the common leading whitespace up to that column is removed. The result has no leading or trailing spaces beyond your intent.

Text blocks are especially useful for:
- JPQL / native SQL in `@Query` annotations
- JSON test fixtures in `@SpringBootTest` tests
- Error message templates

```bash
grep -rn '"\s*\\n\s*"' src/main/java/ src/test/java/ | tee multiline-string-candidates.txt
```

---

## Step 5 — Use `var` for Local Type Inference

Java 10 introduced `var` for local variable type inference. It reduces noise in variable declarations where the type is already obvious from the right-hand side.

```java
// Before
Map<String, List<UserResponse>> groupedByRole = userService.getGroupedByRole();
Iterator<Map.Entry<String, List<UserResponse>>> iterator = groupedByRole.entrySet().iterator();

// After
var groupedByRole = userService.getGroupedByRole();
var iterator = groupedByRole.entrySet().iterator();
```

> **When NOT to use `var`:**
> - When the right-hand side doesn't make the type obvious (e.g., `var x = process()` — what does `process()` return?)
> - For method parameters or fields — `var` is only valid for local variables
> - When the inferred type would be an anonymous class or intersection type

A good heuristic: use `var` when the type name appears within 5 tokens of the `=` sign.

---

## Step 6 — Introduce Sealed Classes for Closed Hierarchies

Java 17 introduced sealed classes — class hierarchies where the compiler knows every permitted subtype. This is a powerful modelling tool when you have a fixed set of variants, such as payment methods or notification types.

**Before (Java 8 — open hierarchy, checked at runtime):**

```java
public abstract class Notification {}
public class EmailNotification extends Notification { ... }
public class SlackNotification extends Notification { ... }
public class SmsNotification extends Notification { ... }
// Nothing stops someone adding WebhookNotification without updating the handler
```

**After (Java 17 sealed classes):**

```java
public sealed interface Notification
    permits EmailNotification, SlackNotification, SmsNotification {}

public record EmailNotification(String to, String subject, String body) implements Notification {}
public record SlackNotification(String channel, String text)             implements Notification {}
public record SmsNotification(String phoneNumber, String message)        implements Notification {}
```

Now any switch over `Notification` that doesn't cover all three permits will produce a compile-time warning (and in Java 21, a compile error without a default).

Update your `NotificationDispatcher` to use a switch expression:

```java
public void dispatch(Notification n) {
    switch (n) {
        case EmailNotification e -> emailSender.send(e.to(), e.subject(), e.body());
        case SlackNotification s -> slackClient.post(s.channel(), s.text());
        case SmsNotification  sms -> smsGateway.send(sms.phoneNumber(), sms.message());
    }
}
```

---

## Step 7 — Checkpoint and What's Next

You have modernised the codebase by applying:

| Java version | Feature | Used for |
|---|---|---|
| Java 10 | `var` | Reducing declaration noise |
| Java 14 | Switch expressions | Exhaustive, fall-through-free dispatch |
| Java 15 | Text blocks | Multi-line SQL, JSON, error messages |
| Java 16 | Records | Immutable DTOs and value objects |
| Java 16 | Pattern matching `instanceof` | Eliminating redundant casts |
| Java 17 | Sealed classes | Closed domain hierarchies |

In Module 3 you will take the application one step further — upgrading from Spring Boot 2.7 to Spring Boot 3, migrating all `javax.*` imports to `jakarta.*`, and then stepping up to Java 21 to enable virtual threads for dramatically improved I/O concurrency.
