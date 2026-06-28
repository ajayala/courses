# Case Study — Implementing CAS Authentication in a Java Web Application

In this capstone lab you will combine everything from the course on one substantial, realistic task: adding **CAS single sign-on** to a Java web application. You will research the protocol with Claude, turn that understanding into an implementation plan, prompt for the actual Spring Security configuration, and then *review* the generated code against your plan rather than trusting it blindly. By the end you will have a research note, an implementation plan, a working Spring Security CAS configuration, and a review checklist — the full professional workflow.

**Prerequisites:** You completed Labs 1–3 and have the `claude-prompting` workspace. Access to Claude. A text editor — you do **not** need a running CAS server or a JDK for this lab; the focus is the prompt-driven research-to-implementation workflow and reviewing the result. Basic familiarity with Java helps.

---

## Step 1 — Set Up and Research the Protocol

You have been asked to protect a Spring Boot web app with your organisation's CAS server. Before writing any code, understand the protocol. Set up your folder and research it with Claude.

```bash
cd claude-prompting
mkdir cas
touch cas/research.md
```

Ask Claude to explain the CAS flow, then record it. Use a research prompt like this:

```markdown
# Prompt 09 — Research the CAS protocol (paste into Claude)

Explain CAS (Central Authentication Service) single sign-on for someone who
will implement it in a Spring Boot app. Cover:
1. The roles: client (browser), the service (my app), the CAS server.
2. The login flow step by step, including the service ticket.
3. How the app validates a ticket and what endpoint it calls on the CAS server.
Keep it concrete and tell me which parts I will need to configure.
```

Capture the essentials in `cas/research.md`. Make sure your notes describe the **service ticket** exchange:

```markdown
# CAS Research Notes

## Roles
- Client: the user's browser.
- Service: my Spring Boot application.
- CAS server: the central authentication server.

## Login flow
1. Unauthenticated user requests a protected page on the service.
2. The app redirects the browser to the CAS server's /login, passing a
   `service` parameter (the callback URL on my app).
3. The user authenticates at the CAS server.
4. CAS redirects back to my app's callback with a **service ticket** (ST-...).
5. My app validates the service ticket by calling the CAS server's
   /serviceValidate (CAS 2.0) or /p3/serviceValidate (CAS 3.0) endpoint.
6. On success, the app establishes an authenticated session.

## What I will need to configure
- The CAS server base URL and my app's service (callback) URL.
- A ticket validator, an entry point, an authentication filter, and a provider.
```

> **Understand the protocol before the framework.** It is tempting to jump straight to "give me the Spring config". But if you do not understand the service-ticket round-trip, you cannot review the generated code or debug it when the redirect loops. Research first; the framework wiring will then make sense.

---

## Step 2 — Turn Understanding into an Implementation Plan

A plan is the bridge between research and code. It lists the concrete pieces you expect — and becomes the checklist you will review the generated code against.

```bash
touch cas/implementation-plan.md
```

Write `cas/implementation-plan.md`. These are the standard Spring Security CAS building blocks:

```markdown
# CAS Implementation Plan (Spring Boot 3 / Spring Security 6)

## Dependencies
- spring-boot-starter-web
- spring-boot-starter-security
- spring-security-cas (pulls in the Apereo CAS client)

## Beans to configure
| Bean | Responsibility |
|------|----------------|
| ServiceProperties | my app's callback URL + sendRenew flag |
| CasAuthenticationEntryPoint | redirects unauthenticated users to CAS /login |
| TicketValidator | validates the service ticket against the CAS server |
| CasAuthenticationProvider | turns a validated ticket into an authenticated user |
| CasAuthenticationFilter | intercepts the /login/cas callback with the ticket |
| SecurityFilterChain | wires it all together and protects routes |

## Configuration values
- CAS server base URL, e.g. https://cas.example.org/cas
- App service URL, e.g. https://app.example.org/login/cas
```

> **The plan is your review checklist.** Notice the table of beans. After you generate code, you will literally tick off each bean: is `CasAuthenticationProvider` present? Is the `TicketValidator` pointed at the CAS server? A plan written before generation keeps you from accepting code that merely *looks* plausible.

---

## Step 3 — Craft a Tightly-Constrained Generation Prompt

This is where the constraints you have practised all course pay off. CAS support has changed across Spring Security versions, so you must pin the versions and package names, or you risk getting outdated `WebSecurityConfigurerAdapter` code.

```bash
touch prompts/10-cas-impl.md
```

Write your generation prompt into `prompts/10-cas-impl.md`:

```markdown
# Prompt 10 — Generate the Spring Security CAS configuration

Context: a Spring Boot 3.x application (Spring Security 6.x, Java 17+). I am
adding CAS single sign-on. I have spring-security-cas on the classpath.

Task: produce a single @Configuration class named SecurityConfig that wires
up CAS authentication using the modern SecurityFilterChain bean style.

Constraints:
- Spring Security 6 APIs only. Use a SecurityFilterChain bean and the
  lambda DSL. Do NOT use the deprecated WebSecurityConfigurerAdapter.
- Use the Apereo CAS client packages (org.apereo.cas.client.*), not jasig.
- Define beans for: ServiceProperties, CasAuthenticationEntryPoint,
  TicketValidator (Cas30ServiceTicketValidator), CasAuthenticationProvider,
  CasAuthenticationFilter, AuthenticationManager, and SecurityFilterChain.
- Use placeholder constants for the CAS server URL and the app service URL.
- Permit "/" and require authentication for everything else.

Format: the complete SecurityConfig.java, then a 3-line note on what I must
change for my environment.
```

> **Pin versions and packages explicitly.** "Spring Security 6 only", "SecurityFilterChain not WebSecurityConfigurerAdapter", and "org.apereo.cas.client not jasig" are the constraints that keep the output current. Without them, a model may reach for older, widely-documented patterns that no longer compile against your dependencies.

---

## Step 4 — Generate and Save the Configuration

Send Prompt 10 to Claude. You should receive a configuration close to the one below. Save it as `cas/SecurityConfig.java` and read it against your plan as you do.

```java
package com.example.casdemo;

import org.apereo.cas.client.validation.Cas30ServiceTicketValidator;
import org.apereo.cas.client.validation.TicketValidator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.ProviderManager;
import org.springframework.security.cas.ServiceProperties;
import org.springframework.security.cas.authentication.CasAssertionAuthenticationToken;
import org.springframework.security.cas.authentication.CasAuthenticationProvider;
import org.springframework.security.cas.web.CasAuthenticationEntryPoint;
import org.springframework.security.cas.web.CasAuthenticationFilter;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.authority.AuthorityUtils;
import org.springframework.security.core.userdetails.AuthenticationUserDetailsService;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    // TODO: change these for your environment
    private static final String CAS_SERVER = "https://cas.example.org/cas";
    private static final String APP_SERVICE = "https://app.example.org/login/cas";

    @Bean
    public ServiceProperties serviceProperties() {
        ServiceProperties sp = new ServiceProperties();
        sp.setService(APP_SERVICE);
        sp.setSendRenew(false);
        return sp;
    }

    @Bean
    public CasAuthenticationEntryPoint casAuthenticationEntryPoint(ServiceProperties sp) {
        CasAuthenticationEntryPoint entryPoint = new CasAuthenticationEntryPoint();
        entryPoint.setLoginUrl(CAS_SERVER + "/login");
        entryPoint.setServiceProperties(sp);
        return entryPoint;
    }

    @Bean
    public TicketValidator ticketValidator() {
        return new Cas30ServiceTicketValidator(CAS_SERVER);
    }

    @Bean
    public AuthenticationUserDetailsService<CasAssertionAuthenticationToken> userDetailsService() {
        // Map the authenticated CAS principal to a user with a default role.
        return token -> new User(
                token.getName(), "",
                AuthorityUtils.createAuthorityList("ROLE_USER"));
    }

    @Bean
    public CasAuthenticationProvider casAuthenticationProvider(
            ServiceProperties sp,
            TicketValidator ticketValidator,
            AuthenticationUserDetailsService<CasAssertionAuthenticationToken> uds) {
        CasAuthenticationProvider provider = new CasAuthenticationProvider();
        provider.setServiceProperties(sp);
        provider.setTicketValidator(ticketValidator);
        provider.setAuthenticationUserDetailsService(uds);
        provider.setKey("cas-demo-key");
        return provider;
    }

    @Bean
    public AuthenticationManager authenticationManager(CasAuthenticationProvider provider) {
        return new ProviderManager(provider);
    }

    @Bean
    public CasAuthenticationFilter casAuthenticationFilter(
            AuthenticationManager authenticationManager, ServiceProperties sp) {
        CasAuthenticationFilter filter = new CasAuthenticationFilter();
        filter.setAuthenticationManager(authenticationManager);
        filter.setServiceProperties(sp);
        return filter;
    }

    @Bean
    public SecurityFilterChain filterChain(
            HttpSecurity http,
            CasAuthenticationEntryPoint entryPoint,
            CasAuthenticationFilter casFilter,
            CasAuthenticationProvider provider) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/").permitAll()
                .anyRequest().authenticated())
            .exceptionHandling(ex -> ex.authenticationEntryPoint(entryPoint))
            .addFilter(casFilter)
            .authenticationProvider(provider);
        return http.build();
    }
}
```

> **Generated code is a draft to be reviewed, not trusted.** Read every bean and match it to your plan from Step 2. The `setKey("cas-demo-key")` value, the placeholder URLs, and the role mapping are all things *you* must adapt — the model cannot know your environment. This is the same "read before you run" habit from Lab 1, applied to security-critical code where it matters most.

---

## Step 5 — Add the Dependencies

The configuration only compiles if the CAS classes are on the classpath. Record the Maven dependencies your plan called for.

```bash
touch cas/pom-snippet.xml
```

Save the dependency snippet to `cas/pom-snippet.xml`:

```xml
<!-- Add to <dependencies> in pom.xml -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
  <!-- Brings in the Apereo CAS client; version managed by the Spring Boot BOM -->
  <groupId>org.springframework.security</groupId>
  <artifactId>spring-security-cas</artifactId>
</dependency>
```

> **Trace every import to a dependency.** A common failure mode is generated code that imports `org.apereo.cas.client.validation.Cas30ServiceTicketValidator` with no matching dependency, then fails to compile. Whenever you accept generated code, confirm each unfamiliar import has a home in your build file.

---

## Step 6 — Review the Result Against Your Plan

Finish like a professional: review the generated code against the plan, and write down what a human still must do. Create a review checklist.

```bash
touch cas/review.md
```

Complete `cas/review.md`, ticking each bean off your Step 2 plan and listing the human follow-ups:

```markdown
# CAS Implementation Review

## Plan coverage (does the code contain each planned bean?)
- [ ] ServiceProperties
- [ ] CasAuthenticationEntryPoint
- [ ] TicketValidator (Cas30ServiceTicketValidator)
- [ ] CasAuthenticationProvider
- [ ] CasAuthenticationFilter
- [ ] AuthenticationManager
- [ ] SecurityFilterChain

## Must change before production
- Real CAS_SERVER and APP_SERVICE URLs.
- A real, secret provider key (not "cas-demo-key").
- Proper role mapping from CAS attributes, not a hard-coded ROLE_USER.

## Still to verify
- Confirm the Apereo client package names against the spring-security-cas
  version resolved by my Spring Boot BOM.
- Test the full login round-trip against a real (or dockerised) CAS server.
```

> **The last 10% is human judgement.** Claude can produce a correct skeleton in seconds, but deciding the key-management strategy, the role mapping, and the testing approach is engineering judgement. Naming the human follow-ups explicitly — as you did in your research brief in Lab 3 — is what makes AI-assisted delivery trustworthy.

---

You have now run the entire prompt-engineering workflow on a meaty, real-world task: research the protocol, plan the implementation, generate tightly-constrained code, and review it against your plan with the human follow-ups written down. Across this course you built an application, investigated a codebase, hunted a bug, ran disciplined research, and delivered a security feature — all through deliberate prompting, with a prompt library and journal to show for it. The tools will keep changing; the method — context, constraints, iteration, and verification — is what lasts.
