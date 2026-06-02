# Migration Readiness Assessment

In this lab you will audit an existing Java 8 Spring Boot application to understand exactly what needs to change before touching a single line of code. You will use `jdeps`, `jdeprscan`, and Maven tooling to produce a dependency inventory, surface deprecated and removed API usage, and document a phased migration plan.

**Prerequisites:** Java 8 JDK installed. Maven 3.8+. A Java 8 Spring Boot project to assess (a sample project is provided at `sample-app/`).

---

## Step 1 — Understand the Java LTS Roadmap

Oracle moved Java to a 6-month release cadence in 2017. Long-Term Support (LTS) releases get extended security patches and are the only versions most enterprises run in production.

| Version | Type | Released | End of support |
|---|---|---|---|
| Java 8 | LTS | 2014 | 2030 (extended) |
| Java 11 | LTS | 2018 | 2026 |
| Java 17 | LTS | 2021 | 2029 |
| Java 21 | LTS | 2023 | 2031 |
| Java 25 | LTS (upcoming) | Sep 2025 | TBC |

The recommended migration path for a Java 8 app is **8 → 17 → 21**, stepping through LTS versions rather than jumping directly. Each step is smaller and its failure modes are well-documented.

```bash
java -version
```

Expected output (your baseline):

```
openjdk version "1.8.0_392"
OpenJDK Runtime Environment (build 1.8.0_392-b08)
```

> **Why not jump straight to 21?** The biggest breaking changes happened at the module system boundary (Java 9) and the javax→jakarta rename (Spring Boot 3 / Java 17 baseline). Mixing both in one step makes it very hard to isolate failures.

---

## Step 2 — Install SDKMAN and Multiple JDKs

SDKMAN lets you install and switch between Java versions without system-level changes, which is essential when you need to compile under Java 8 but test under Java 17 and 21.

```bash
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
```

Install the three JDKs you will use in this course:

```bash
sdk install java 8.0.392-amzn
sdk install java 17.0.10-tem
sdk install java 21.0.2-tem
```

Verify all three are available:

```bash
sdk list java | grep -E "8\.|17\.|21\."
```

> **Tip:** `sdk use java <version>` changes the JDK for the current shell session. `sdk default java <version>` changes it globally. Use `sdk use` during the uplift so your other projects are unaffected.

---

## Step 3 — Scan Dependencies with jdeps

`jdeps` is a JDK tool that analyses `.class` files and reports which JDK internal APIs your code (and libraries) depend on. Internal APIs (`sun.*`, `com.sun.*`) were accessible in Java 8 but are strongly encapsulated or removed in Java 9+.

Switch to Java 17 temporarily for the scan (it can analyse Java 8 bytecode):

```bash
sdk use java 17.0.10-tem
```

Build the project fat JAR first:

```bash
cd sample-app
mvn package -DskipTests -q
```

Run `jdeps` on the assembled JAR:

```bash
jdeps --multi-release 17 \
      --ignore-missing-deps \
      --print-module-deps \
      target/sample-app-1.0.0.jar
```

Then check specifically for internal API usage:

```bash
jdeps --jdk-internals \
      --multi-release 17 \
      target/sample-app-1.0.0.jar 2>&1 | tee jdeps-report.txt
```

Any line mentioning `JDK internal API` is a risk item. Save `jdeps-report.txt` — you will use it in the next step.

---

## Step 4 — Find Deprecated and Removed APIs with jdeprscan

`jdeprscan` lists usages of APIs marked `@Deprecated` in the target JDK. Since Java 9, many APIs deprecated in Java 8 have been removed entirely.

```bash
jdeprscan --release 17 target/sample-app-1.0.0.jar 2>&1 | tee jdeprscan-17.txt
jdeprscan --release 21 target/sample-app-1.0.0.jar 2>&1 | tee jdeprscan-21.txt
```

Common removals to watch for:

| Removed API | Replacement |
|---|---|
| `sun.misc.BASE64Encoder/Decoder` | `java.util.Base64` |
| `com.sun.net.ssl.*` | `javax.net.ssl.*` |
| `Thread.stop()` / `Thread.suspend()` | Cooperative cancellation via `interrupted()` |
| `SecurityManager` | Removed in Java 17, replaced by OS-level sandboxing |
| `RMISecurityManager` | Removed alongside SecurityManager |
| `Applet` / `AppletContext` | Removed in Java 17 |

> **Warning:** `jdeprscan` only scans your own code, not transitive dependencies. A library you depend on may use a removed API internally — `jdeps --jdk-internals` on the full fat JAR catches those.

---

## Step 5 — Audit Spring Boot and Third-Party Dependency Versions

Java 17 and 21 have minimum version requirements for popular libraries. Versions below these floors will not work or will produce `InaccessibleObjectException` at runtime.

Check your current versions:

```bash
mvn dependency:tree -Dincludes="org.springframework.boot,org.springframework,org.hibernate,org.springframework.security" | tee deps-tree.txt
```

Minimum safe versions for Java 17:

| Dependency | Minimum version for Java 17 |
|---|---|
| Spring Boot | 2.7.x (for Java 17 on Boot 2) or 3.0+ |
| Spring Framework | 5.3.x |
| Hibernate ORM | 5.6.x |
| Mockito | 4.x |
| Byte Buddy | 1.12.x |
| ASM (via Spring) | 9.x |
| Lombok | 1.18.22+ |
| MapStruct | 1.5.x |

For Java 21, you need Spring Boot 3.2+ and Spring Framework 6.1+.

```bash
mvn versions:display-dependency-updates 2>/dev/null | grep -E "^\[INFO\].*->.*" | head -30
```

---

## Step 6 — Produce a Migration Plan Document

Consolidate your findings into a structured migration plan. Create `migration-plan.md` in the project root:

```bash
cat > migration-plan.md << 'EOF'
# Java 8 → 21 Migration Plan

## Phase 1: Java 8 → Java 17 (Target: Sprint X)
- [ ] Upgrade Spring Boot to 2.7.x
- [ ] Update Mockito, Byte Buddy, ASM, Lombok to Java-17-safe versions
- [ ] Replace all internal API usages found in jdeps-report.txt
- [ ] Add --add-opens JVM args for remaining reflection-based libraries
- [ ] Fix compiler warnings surfaced by -source 17

## Phase 2: Java 17 → Spring Boot 3 / Jakarta EE 10 (Target: Sprint Y)
- [ ] Run OpenRewrite javax→jakarta migration recipe
- [ ] Replace deprecated Spring Security WebSecurityConfigurerAdapter
- [ ] Replace deprecated Spring MVC/Data APIs
- [ ] Validate renamed application.properties keys

## Phase 3: Spring Boot 3 → Java 21 (Target: Sprint Z)
- [ ] Enable virtual threads (spring.threads.virtual.enabled=true)
- [ ] Replace anonymous classes with records where applicable
- [ ] Replace if-instanceof chains with pattern matching switch
- [ ] Run JMH benchmarks before/after virtual threads

## Risk Register
| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Hidden sun.* usage in transitive deps | Medium | High | Fat JAR jdeps scan |
| Reflection broken by strong encapsulation | High | Medium | --add-opens + upgrade lib |
| javax→jakarta in config files/SQL | Low | Medium | Full-text grep |
EOF
```

```bash
cat migration-plan.md
```

---

## Step 7 — Checkpoint and What's Next

You now have:

- A clear picture of the Java LTS roadmap and why a phased approach is safest
- `jdeps-report.txt` listing internal API dependencies
- `jdeprscan-17.txt` and `jdeprscan-21.txt` listing deprecated and removed API usage
- A `deps-tree.txt` showing libraries that need version bumps
- A `migration-plan.md` to track progress across sprints

In the next module you will make the first real code changes — updating the Maven build to target Java 17, fixing the `InaccessibleObjectException` errors caused by the module system, and running the test suite green on the new JDK.
