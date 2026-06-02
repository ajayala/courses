# Updating the Build and Fixing Encapsulation

In this lab you will update a Java 8 Spring Boot project's Maven build to compile and run on Java 17. You will update the compiler plugin, bump Spring Boot and key library versions, and work through the most common runtime failure mode of the Java 9+ module system: `InaccessibleObjectException` and `IllegalAccessError`.

**Prerequisites:** Lab 1-1 complete. SDKMAN installed with Java 8 and Java 17. Maven 3.8+.

---

## Step 1 — Switch to Java 17

Use SDKMAN to switch to Java 17 for this lab:

```bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk use java 17.0.10-tem
java -version
```

Expected:

```
openjdk version "17.0.10" 2024-01-16
OpenJDK Runtime Environment Temurin-17.0.10+7 (build 17.0.10+7)
```

Confirm Maven picks up the new JDK:

```bash
mvn -version
```

Both `Java version` and `Java home` in the output should reference 17.

---

## Step 2 — Update the Maven Compiler Plugin and Java Version Properties

Open `pom.xml`. Java version is usually set via properties. Replace the Java 8 values:

```xml
<!-- Before -->
<properties>
    <java.version>1.8</java.version>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
</properties>

<!-- After -->
<properties>
    <java.version>17</java.version>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <maven.compiler.release>17</maven.compiler.release>
</properties>
```

> **Why `--release` instead of `--source`/`--target`?** The `--release` flag sets source, target, *and* the bootclasspath in a single flag, preventing you from accidentally compiling against a newer standard library than you declared. It is the correct flag for Java 9+.

Update `maven-compiler-plugin` to a version that supports `--release`:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.13.0</version>
    <configuration>
        <release>17</release>
    </configuration>
</plugin>
```

---

## Step 3 — Bump Spring Boot to 2.7.x

Spring Boot 2.7.x is the last 2.x release and is the recommended waypoint before jumping to Spring Boot 3. It is fully tested on Java 17.

In `pom.xml`, update the Spring Boot parent:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.18</version>
    <relativePath/>
</parent>
```

Then update libraries that Spring Boot's BOM does not manage:

```xml
<properties>
    <!-- ... existing properties ... -->
    <lombok.version>1.18.30</lombok.version>
    <mapstruct.version>1.5.5.Final</mapstruct.version>
    <mockito.version>4.11.0</mockito.version>
</properties>
```

Run a clean compile to see the first wave of errors:

```bash
mvn clean compile 2>&1 | tee compile-java17.txt
```

> **Expect errors here.** A clean compile after a major Java bump almost always fails. The goal of this step is to see *all* errors at once, not to fix them yet.

---

## Step 4 — Fix Strong Encapsulation Errors

The most common Java 9+ runtime failure is `InaccessibleObjectException`. Java's module system now enforces that code in one module cannot reflectively access private fields of another module unless explicitly opened.

Libraries like Hibernate, Jackson, and Spring use reflection extensively. They need `--add-opens` JVM arguments until they release versions that use the official APIs.

Add these to `pom.xml` inside the Surefire plugin configuration (for tests) and in a `.mvn/jvm.config` file (for the running application):

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>
            --add-opens java.base/java.lang=ALL-UNNAMED
            --add-opens java.base/java.util=ALL-UNNAMED
            --add-opens java.base/java.lang.reflect=ALL-UNNAMED
            --add-opens java.base/java.text=ALL-UNNAMED
            --add-opens java.desktop/java.awt.font=ALL-UNNAMED
        </argLine>
    </configuration>
</plugin>
```

Create `.mvn/jvm.config` for running the app locally:

```bash
mkdir -p .mvn
cat > .mvn/jvm.config << 'EOF'
--add-opens java.base/java.lang=ALL-UNNAMED
--add-opens java.base/java.util=ALL-UNNAMED
--add-opens java.base/java.lang.reflect=ALL-UNNAMED
--add-opens java.base/java.text=ALL-UNNAMED
--add-opens java.desktop/java.awt.font=ALL-UNNAMED
EOF
```

> **These are temporary.** Each `--add-opens` is a sign that a library is using internal APIs. As you upgrade libraries to Java-17-native versions, remove the corresponding flag. Leaving them all in permanently hides future problems.

---

## Step 5 — Replace Removed sun.* API Usages

If `jdeps-report.txt` from Lab 1-1 listed any `sun.*` usages in your own code, fix them now. The most common cases:

**Base64 encoding:**

```java
// Java 8 (sun.misc — removed)
String encoded = new sun.misc.BASE64Encoder().encode(bytes);

// Java 17+ (standard library — available since Java 8u6!)
String encoded = java.util.Base64.getEncoder().encodeToString(bytes);
```

**XML parsing with internal factory:**

```java
// Java 8 (internal)
DocumentBuilderFactory dbf = com.sun.org.apache.xerces.internal.jaxp.DocumentBuilderFactoryImpl.newInstance();

// Java 17+
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
```

**Unsafe operations (rare, but present in some older codebases):**

```java
// Before — sun.misc.Unsafe
Field f = Unsafe.class.getDeclaredField("theUnsafe");
f.setAccessible(true);
Unsafe unsafe = (Unsafe) f.get(null);

// After — VarHandle (Java 9+) or use a library abstraction
```

Run `grep -r "sun\." src/main/java/` to find any remaining usages in your own source tree.

```bash
grep -rn "sun\." src/main/java/ src/test/java/ 2>/dev/null | grep -v "//.*sun\." | tee sun-api-usages.txt
```

---

## Step 6 — Compile and Run Tests Clean

With the build updated and `sun.*` usages replaced, attempt a full build:

```bash
mvn clean verify 2>&1 | tee build-java17-final.txt
```

Interpreting common remaining failures:

| Error | Cause | Fix |
|---|---|---|
| `InaccessibleObjectException` | Missing `--add-opens` | Add the specific package to `.mvn/jvm.config` |
| `NoSuchMethodError` | Binary incompatible library version | Check `mvn dependency:tree` for old transitive version |
| `ClassNotFoundException` | Class moved between modules | Add the new module as a dependency |
| `NullPointerException` in boot | Spring Boot version too old for Java 17 | Ensure Spring Boot ≥ 2.7.x |

Once `mvn clean verify` exits with `BUILD SUCCESS`, move on.

```bash
echo "Exit code: $?"
```

---

## Step 7 — Checkpoint and What's Next

You have:

- Switched the build toolchain to Java 17 using `--release`
- Upgraded Spring Boot to the 2.7.x LTS waypoint
- Added `--add-opens` arguments to unblock reflection-based libraries
- Replaced any `sun.*` API usages with standard equivalents
- Achieved a passing test suite on Java 17

In the next lab you will shift from *fixing breaks* to *improving code* — using the new language features introduced between Java 9 and 17 to make your codebase more expressive and maintainable: records, sealed classes, text blocks, pattern matching, and switch expressions.
