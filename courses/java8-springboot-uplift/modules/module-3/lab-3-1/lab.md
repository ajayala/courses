# Migrating to Spring Boot 3 and Jakarta EE

In this lab you will upgrade a Spring Boot 2.7 application to Spring Boot 3.x. The two major breaking changes are the Java baseline moving to 17 (already satisfied) and the renaming of all `javax.*` packages to `jakarta.*` as part of Jakarta EE 10. You will use the OpenRewrite migration tool to automate the bulk of the work, then manually address Spring Security's redesigned configuration API.

**Prerequisites:** Labs 2-1 and 2-2 complete. Project must be on Spring Boot 2.7.x and Java 17. Maven 3.8.5+.

---

## Step 1 — Understand What Changes in Spring Boot 3

Spring Boot 3.0 (November 2022) raised the baseline to Java 17 and migrated from Java EE (javax) to Jakarta EE 10 (jakarta). This affects every class that touches the web layer, persistence layer, validation, or CDI.

Key changes at a glance:

| Area | Before (Spring Boot 2.x) | After (Spring Boot 3.x) |
|---|---|---|
| Servlet API | `javax.servlet.*` | `jakarta.servlet.*` |
| JPA | `javax.persistence.*` | `jakarta.persistence.*` |
| Bean Validation | `javax.validation.*` | `jakarta.validation.*` |
| Mail | `javax.mail.*` | `jakarta.mail.*` |
| Security config | `WebSecurityConfigurerAdapter` | `SecurityFilterChain` bean |
| Actuator paths | `/actuator/health` (same) | Same, but some props renamed |
| Properties | Many deprecated keys | `spring.config.migrate` to detect |

The rename affects import statements, annotations, and any configuration files that reference class names (`persistence.xml`, `beans.xml`).

---

## Step 2 — Run the OpenRewrite javax→jakarta Migration

OpenRewrite is a refactoring engine that applies large-scale migrations as code transformations. The Spring Boot team maintains an official recipe for the Boot 2.x → 3.x upgrade.

Add the OpenRewrite Maven plugin to `pom.xml`:

```xml
<plugin>
    <groupId>org.openrewrite.maven</groupId>
    <artifactId>rewrite-maven-plugin</artifactId>
    <version>5.34.1</version>
    <configuration>
        <activeRecipes>
            <recipe>org.openrewrite.java.spring.boot3.UpgradeSpringBoot_3_2</recipe>
        </activeRecipes>
    </configuration>
    <dependencies>
        <dependency>
            <groupId>org.openrewrite.recipe</groupId>
            <artifactId>rewrite-spring</artifactId>
            <version>5.19.0</version>
        </dependency>
    </dependencies>
</plugin>
```

Run a dry-run first to see what would change without modifying files:

```bash
mvn rewrite:dryRun 2>&1 | tee rewrite-dryrun.txt
grep "would change" rewrite-dryrun.txt | head -20
```

Once satisfied, apply the recipe:

```bash
mvn rewrite:run
```

Commit the automated changes before making manual fixes — this creates a clean audit trail:

```bash
git add -A && git commit -m "chore: apply OpenRewrite Spring Boot 3 migration recipe"
```

---

## Step 3 — Manually Fix Spring Security Configuration

OpenRewrite migrates many Security patterns automatically, but complex configurations often need manual attention. Spring Boot 3 / Spring Security 6 removed `WebSecurityConfigurerAdapter` entirely.

**Before (Spring Security 5 — deprecated in 5.7, removed in 6.0):**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            .and()
            .formLogin().loginPage("/login").permitAll()
            .and()
            .logout().permitAll();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(passwordEncoder());
    }
}
```

**After (Spring Security 6 — component-based):**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form.loginPage("/login").permitAll())
            .logout(LogoutConfigurer::permitAll);
        return http.build();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

Key API changes:
- `authorizeRequests()` → `authorizeHttpRequests()` with `requestMatchers()` (not `antMatchers()`)
- Configuration DSL is now lambda-based instead of method-chaining
- `WebSecurityCustomizer` replaces `configure(WebSecurity)` for ignoring paths

---

## Step 4 — Verify and Fix Renamed Application Properties

Spring Boot 3 renamed and removed many `application.properties` keys. Run the Boot migration assistant:

```bash
mvn spring-boot:run -Dspring-boot.run.arguments="--spring.config.migrate=true" 2>&1 | grep "WARN\|ERROR\|Replacement" | tee properties-migration.txt
```

Common renames to apply manually:

```properties
# Before (Spring Boot 2.x)
spring.redis.host=localhost
spring.redis.port=6379
server.max-http-header-size=8KB
spring.datasource.initialization-mode=always
logging.file=/var/log/app.log

# After (Spring Boot 3.x)
spring.data.redis.host=localhost
spring.data.redis.port=6379
server.max-http-request-header-size=8KB
spring.sql.init.mode=always
logging.file.name=/var/log/app.log
```

Also check `application.yml` files — grep for the old property prefixes:

```bash
grep -rn "spring\.redis\|logging\.file=" src/main/resources/ | tee old-properties.txt
```

---

## Step 5 — Update Spring Boot Version and Compile

Now bump the Spring Boot parent in `pom.xml` to 3.2.x (the current 3.x LTS-aligned release):

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.4</version>
    <relativePath/>
</parent>
```

Spring Boot 3 manages Java 17 by default — you can remove the explicit `<java.version>` override if desired, or keep it to be explicit.

Remove the `--add-opens` entries that are no longer needed — Spring Framework 6 uses method handles instead of reflection for most internals:

```bash
mvn clean verify 2>&1 | tee build-boot3.txt
```

> **If you see `NoSuchMethodError` or `ClassNotFoundException`:** Check `mvn dependency:tree` for any remaining `javax.*` dependencies being pulled in transitively. Add an exclusion and pull in the `jakarta.*` equivalent.

---

## Step 6 — Run Integration Tests and Validate Endpoints

Spring Boot 3 changes the default request mapping for some Actuator endpoints. Validate your test suite and spot-check key endpoints:

```bash
mvn clean verify -Pintegration-tests 2>&1 | tail -30
```

Check the Actuator health endpoint still responds:

```bash
mvn spring-boot:run &
sleep 10
curl -s http://localhost:8080/actuator/health | python3 -m json.tool
kill %1
```

Expected response:

```json
{
    "status": "UP",
    "components": {
        "db": { "status": "UP" },
        "diskSpace": { "status": "UP" }
    }
}
```

---

## Step 7 — Checkpoint and What's Next

You have completed the most disruptive part of the uplift:

- Run OpenRewrite to automatically rename `javax.*` → `jakarta.*` across the entire codebase
- Replaced the removed `WebSecurityConfigurerAdapter` with the `SecurityFilterChain` bean model
- Updated renamed `application.properties` keys to Spring Boot 3 equivalents
- Bumped to Spring Boot 3.2.x and verified a clean build and passing tests

The application now runs on Spring Boot 3.2 and Java 17. In the final lab you will take the last step to Java 21 and unlock its headline feature for server-side applications: **virtual threads** — which can dramatically improve throughput under I/O-bound workloads with a one-line configuration change.
