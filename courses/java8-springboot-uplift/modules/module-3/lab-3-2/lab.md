# Java 21 Features and Virtual Threads

In this final lab you will upgrade the application from Java 17 to Java 21 and enable virtual threads — Java 21's headline feature for server-side applications. You will also apply the remaining Java 18–21 language features that improve expressiveness and safety, and update your CI/CD pipeline to validate against multiple JDK versions.

**Prerequisites:** Lab 3-1 complete. Application runs on Spring Boot 3.2.x and Java 17.

---

## Step 1 — Switch to Java 21 and Update the Build

Install and activate Java 21:

```bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk use java 21.0.2-tem
java -version
```

Expected:

```
openjdk version "21.0.2" 2024-01-16
OpenJDK Runtime Environment Temurin-21.0.2+13 (build 21.0.2+13)
```

Update `pom.xml` to compile for Java 21:

```xml
<properties>
    <java.version>21</java.version>
    <maven.compiler.source>21</maven.compiler.source>
    <maven.compiler.target>21</maven.compiler.target>
    <maven.compiler.release>21</maven.compiler.release>
</properties>
```

Bump to Spring Boot 3.3.x — which officially targets Java 21 and enables virtual threads by default in Spring MVC and Spring WebFlux:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.0</version>
    <relativePath/>
</parent>
```

Verify the build still passes:

```bash
mvn clean verify -q
echo "Build result: $?"
```

---

## Step 2 — Enable Virtual Threads

Virtual threads (Project Loom, finalised in Java 21) are lightweight threads managed by the JVM rather than the OS. They allow each request to block freely on I/O without tying up an OS thread, dramatically increasing throughput on I/O-bound workloads without changing application code.

In Spring Boot 3.2+, enabling virtual threads for the embedded Tomcat request handler is a single property:

```properties
# application.properties
spring.threads.virtual.enabled=true
```

That is all. Spring Boot configures the Tomcat executor to use `Thread.ofVirtual().executor()` and the async task executor to use virtual threads.

Verify the setting took effect by checking the thread names at startup:

```bash
mvn spring-boot:run &
sleep 8
curl -s http://localhost:8080/actuator/metrics/jvm.threads.live | python3 -m json.tool
kill %1
```

> **What changes under the hood:** With virtual threads, each request to your `@RestController` runs on a new virtual thread. The carrier thread pool (real OS threads) is typically sized to the number of CPU cores. The JVM multiplexes thousands of concurrent virtual threads over this small pool, parking them whenever they block on I/O and resuming them when I/O completes.

---

## Step 3 — Benchmark: Platform Threads vs Virtual Threads

Measure the difference virtual threads make for I/O-bound endpoints using Apache Bench:

```bash
# Ensure ab is installed
sudo apt-get install -y apache2-utils   # Ubuntu/Debian
# or: brew install httpd                # macOS
```

Start the app with platform threads first:

```bash
# Temporarily disable virtual threads
mvn spring-boot:run -Dspring-boot.run.jvmArguments="-Dspring.threads.virtual.enabled=false" &
sleep 8

# Benchmark: 1000 requests, 100 concurrent
ab -n 1000 -c 100 http://localhost:8080/api/users > bench-platform.txt 2>&1
kill %1
```

Then benchmark with virtual threads:

```bash
mvn spring-boot:run &
sleep 8
ab -n 1000 -c 100 http://localhost:8080/api/users > bench-virtual.txt 2>&1
kill %1
```

Compare the "Requests per second" and "Time per request" figures:

```bash
grep -E "Requests per second|Time per request" bench-platform.txt bench-virtual.txt
```

For I/O-heavy endpoints (database calls, external HTTP), virtual threads typically show 2–5× higher throughput at the same concurrency level.

---

## Step 4 — Apply Java 21 Pattern Matching for Switch

Java 21 finalises pattern matching for switch, which extends the Java 17 sealed-class switch to work with any type hierarchy and adds guarded patterns (`when` clauses).

Update the `NotificationDispatcher` from Lab 2-2:

```java
// Java 21 — pattern matching switch with guard
public void dispatch(Notification n) {
    switch (n) {
        case EmailNotification e when e.to().endsWith("@vip.example.com") ->
            priorityEmailSender.send(e);
        case EmailNotification e ->
            emailSender.send(e.to(), e.subject(), e.body());
        case SlackNotification s ->
            slackClient.post(s.channel(), s.text());
        case SmsNotification sms when sms.phoneNumber().startsWith("+1") ->
            usSmsSender.send(sms.phoneNumber(), sms.message());
        case SmsNotification sms ->
            internationalSmsSender.send(sms.phoneNumber(), sms.message());
    }
}
```

The `when` clause is a guard condition evaluated after the pattern matches. This replaces nested `if` statements inside switch arms.

> **Null handling in switch:** Java 21 also allows `case null ->` in switch expressions. Previously, switching on a null reference always threw `NullPointerException`. Now you can handle it explicitly.

---

## Step 5 — Use Record Patterns for Structural Decomposition

Java 21 finalises record patterns, which allow you to destructure a record in a `switch` or `instanceof` check in a single step.

```java
// Before — extract record fields manually
if (event instanceof OrderPlaced op) {
    Long orderId   = op.orderId();
    String userId  = op.userId();
    process(orderId, userId);
}

// After — record pattern destructures in place
if (event instanceof OrderPlaced(Long orderId, String userId)) {
    process(orderId, userId);
}
```

Nested records can be destructured in one step:

```java
// Given: record Address(String city, String country) {}
//        record Customer(String name, Address address) {}

switch (customer) {
    case Customer(var name, Address(var city, "US")) ->
        usHandler.handle(name, city);
    case Customer(var name, Address(var city, var country)) ->
        internationalHandler.handle(name, city, country);
}
```

Update your event-handling code to use record patterns where the pattern is clearer than manual field access.

---

## Step 6 — Update CI/CD for Multi-JDK Validation

Update your GitHub Actions (or Jenkins/GitLab CI) pipeline to build and test against both Java 17 and Java 21 to prevent regressions as you maintain the codebase going forward.

Create or update `.github/workflows/ci.yml`:

```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    strategy:
      matrix:
        java: ['17', '21']

    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK ${{ matrix.java }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java }}
          distribution: temurin
          cache: maven

      - name: Build and test
        run: mvn clean verify -B

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-java${{ matrix.java }}
          path: target/surefire-reports/
```

This ensures every PR is validated on both the current LTS (21) and the previous LTS (17), making it safe to downgrade if a production incident requires it.

```bash
# Validate the workflow file is valid YAML
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))" && echo "YAML valid"
```

---

## Step 7 — Course Recap: The Full Uplift Journey

Congratulations — you have completed the full Java 8 → Java 21 upgrade path. Here is the complete picture of what you did and why each step mattered:

| Phase | Action | Why |
|---|---|---|
| **Assessment** | `jdeps` + `jdeprscan` + dep audit | Discover risk before touching code |
| **Java 17 build** | `--release 17`, Spring Boot 2.7.x | First LTS waypoint; module system |
| **Encapsulation** | `--add-opens`, `sun.*` replacements | Module system broke reflection |
| **Code modernisation** | Records, text blocks, switch expr, sealed | Reduce boilerplate, improve safety |
| **Spring Boot 3** | OpenRewrite `javax→jakarta`, Security rewrite | Jakarta EE 10 rename; Security 6 |
| **Java 21** | `--release 21`, Boot 3.3.x | Final LTS target |
| **Virtual threads** | `spring.threads.virtual.enabled=true` | I/O concurrency without reactive |
| **CI matrix** | Build on Java 17 + 21 | Prevent silent regressions |

**Recommended next steps:**

- Remove any remaining `--add-opens` flags — if they are still needed, file issues with the upstream library
- Explore GraalVM Native Image via `spring-boot-starter-parent` AOT compilation for faster startup times
- Monitor the Java 25 LTS release (September 2025) and begin the 21→25 assessment using the same methodology from Module 1
