# Step Definitions — Parameters, Hooks, and the World Object

In this lab you will replace stub step definitions with real Playwright automation, share browser state between steps using the **World object**, and manage test lifecycle with **Before/After hooks**. You will also learn how to use Cucumber's parameter types and expressions to keep step definitions DRY and reusable. By the end, the login scenarios from the previous lab will run against a real browser.

**Prerequisites:** Lab 2-1 completed. `bdd-lab/features/login.feature` exists with Background, Scenario Outline, and Data Table scenarios.

---

## Step 1 — The World Object

In Lab 1-1 each step definition created its own `browser`, `context`, and `page` variables. This breaks down immediately when a scenario has multiple steps — there's no way to share state between them.

The **World** is Cucumber's solution: a fresh object created for each scenario that all step definitions in that scenario share. Every step function's `this` refers to the World when the step file uses `function` (not arrow functions).

Create the World class:

```typescript
// features/support/world.ts

import { World, IWorldOptions, setWorldConstructor } from "@cucumber/cucumber";
import { Browser, BrowserContext, Page, chromium } from "playwright";

export class PlaywrightWorld extends World {
  browser!: Browser;
  context!: BrowserContext;
  page!: Page;

  constructor(options: IWorldOptions) {
    super(options);
  }

  async openBrowser() {
    this.browser = await chromium.launch({ headless: true });
    this.context = await this.browser.newContext();
    this.page = await this.context.newPage();
  }

  async closeBrowser() {
    await this.browser?.close();
  }
}

setWorldConstructor(PlaywrightWorld);
```

> **`setWorldConstructor`** registers your class as the World factory. Cucumber creates a new instance before each scenario and discards it afterwards — so there is no state leakage between scenarios, even when running in parallel.

---

## Step 2 — Before and After Hooks

Hooks run before or after each scenario (or each step). They receive the World as `this`, which makes them the right place to start and stop the browser.

```typescript
// features/support/hooks.ts

import { Before, After, BeforeAll, AfterAll, Status } from "@cucumber/cucumber";
import { PlaywrightWorld } from "./world";

Before(async function (this: PlaywrightWorld) {
  await this.openBrowser();
});

After(async function (this: PlaywrightWorld, scenario) {
  if (scenario.result?.status === Status.FAILED) {
    // Take a screenshot on failure so you can debug
    const screenshot = await this.page.screenshot();
    this.attach(screenshot, "image/png");
  }
  await this.closeBrowser();
});
```

Notice `function` (not arrow `=>`). Arrow functions capture `this` from the enclosing scope — which would be `undefined` inside a Cucumber hook. Always use `function` when you need access to `this` (the World).

| Hook | Runs | `this` |
|------|------|--------|
| `Before` | Before each scenario | World instance |
| `After` | After each scenario | World instance |
| `BeforeAll` | Once before the entire suite | Not a World |
| `AfterAll` | Once after the entire suite | Not a World |
| `BeforeStep` | Before each step | World instance |
| `AfterStep` | After each step | World instance |

---

## Step 3 — Rewrite Login Steps with the World

Now that the World manages the browser, step definitions become clean and focused:

```typescript
// features/step-definitions/login.steps.ts  (rewrite completely)

import { Given, When, Then } from "@cucumber/cucumber";
import { expect } from "@playwright/test";
import { PlaywrightWorld } from "../support/world";

// Use a demo login page that works without a real backend
const LOGIN_URL = "https://practicetestautomation.com/practice-test-login/";

Given("I navigate to the login page", async function (this: PlaywrightWorld) {
  await this.page.goto(LOGIN_URL);
});

When(
  "I enter username {string} and password {string}",
  async function (this: PlaywrightWorld, username: string, password: string) {
    await this.page.fill("#username", username);
    await this.page.fill("#password", password);
  }
);

When("I click the login button", async function (this: PlaywrightWorld) {
  await this.page.click("#submit");
});

Then("I should see the dashboard", async function (this: PlaywrightWorld) {
  await expect(this.page.locator(".post-title")).toContainText("Logged In Successfully");
});

Then(
  "I should see the error message {string}",
  async function (this: PlaywrightWorld, expectedMessage: string) {
    const error = this.page.locator("#error");
    await expect(error).toBeVisible();
    await expect(error).toContainText(expectedMessage);
  }
);
```

```bash
cd bdd-lab
npx cucumber-js features/login.feature --name "Successful login"
```

> **`expect` from `@playwright/test`** gives you Playwright's assertion library with automatic retrying — it polls until the condition is true or a timeout is reached. This is much better than a manual `if (!condition) throw new Error(...)` because it handles timing without flaky sleeps.

---

## Step 4 — Cucumber Expressions and Parameter Types

Cucumber expressions are the patterns in `Given/When/Then` strings. You have already used `{string}` and `{int}`. Cucumber provides several built-in types and lets you define custom ones.

```typescript
// features/support/parameter-types.ts

import { defineParameterType } from "@cucumber/cucumber";

// Built-in types
// {string}  — matches a double-quoted or single-quoted string
// {int}     — matches an integer
// {float}   — matches a floating-point number
// {word}    — matches a single word (no spaces)
// {}        — matches anything (anonymous)

// Custom parameter type — matches a browser name
defineParameterType({
  name: "browser",
  regexp: /chrome|firefox|webkit/,
  transformer: (s: string) => s,
});
```

Use the custom type in a step:

```typescript
// features/step-definitions/browser.steps.ts

import { Given } from "@cucumber/cucumber";
import { PlaywrightWorld } from "../support/world";
import { chromium, firefox, webkit } from "playwright";

Given(
  "I open the app in {browser}",
  async function (this: PlaywrightWorld, browserName: string) {
    const launcher = { chrome: chromium, firefox, webkit }[browserName] ?? chromium;
    this.browser = await launcher.launch({ headless: true });
    this.context = await this.browser.newContext();
    this.page = await this.context.newPage();
    await this.page.goto("https://practicetestautomation.com/practice-test-login/");
  }
);
```

Add a scenario to `features/login.feature` that uses it:

```gherkin
  Scenario: Login works in Firefox
    Given I open the app in firefox
    When I enter username "student" and password "Password123"
    And I click the login button
    Then I should see the dashboard
```

```bash
npx playwright install firefox
npx cucumber-js features/login.feature --name "Login works in Firefox"
```

---

## Step 5 — Attaching Evidence and Logging

Cucumber lets you attach arbitrary data to a step — screenshots, HTML, JSON logs — which are embedded in the HTML report.

```typescript
// Update the After hook in features/support/hooks.ts

import { Before, After, Status } from "@cucumber/cucumber";
import { PlaywrightWorld } from "./world";

Before(async function (this: PlaywrightWorld) {
  await this.openBrowser();
  this.attach(`Starting scenario: ${new Date().toISOString()}`, "text/plain");
});

After(async function (this: PlaywrightWorld, scenario) {
  // Always attach final page URL for debugging
  try {
    const url = this.page.url();
    this.attach(`Final URL: ${url}`, "text/plain");
  } catch {
    // page may be closed already
  }

  if (scenario.result?.status === Status.FAILED) {
    const screenshot = await this.page.screenshot({ fullPage: true });
    this.attach(screenshot, "image/png");

    // Attach page HTML for debugging invisible elements
    const html = await this.page.content();
    this.attach(html, "text/html");
  }

  await this.closeBrowser();
});
```

Run the full login suite and inspect the HTML report:

```bash
npx cucumber-js features/login.feature
open reports/cucumber-report.html   # macOS
# xdg-open reports/cucumber-report.html  # Linux
```

---

## Step 6 — Conditional Hooks with Tags

Hooks can be filtered by tags so they only run for scenarios that need special setup — for example, scenarios that require an authenticated session.

```typescript
// Add to features/support/hooks.ts

import { Before, After } from "@cucumber/cucumber";
import { PlaywrightWorld } from "./world";

// Only runs for scenarios tagged @authenticated
Before({ tags: "@authenticated" }, async function (this: PlaywrightWorld) {
  await this.page.goto("https://practicetestautomation.com/practice-test-login/");
  await this.page.fill("#username", "student");
  await this.page.fill("#password", "Password123");
  await this.page.click("#submit");
  // Wait for the authenticated page to load
  await this.page.waitForSelector(".post-title");
});
```

Tag a scenario to use it:

```gherkin
# Add to features/login.feature

  @authenticated
  Scenario: Authenticated user can see their profile
    Given I am on the profile page
    Then I should see "Logged In Successfully"
```

```typescript
// Add to features/step-definitions/login.steps.ts

Given("I am on the profile page", async function (this: PlaywrightWorld) {
  await this.page.goto("https://practicetestautomation.com/logged-in-successfully/");
});
```

```bash
npx cucumber-js --tags "@authenticated"
```

You now have a complete, professional step definition layer: shared state via the World, lifecycle management via hooks, rich failure evidence via attachments, and targeted setup via tagged hooks. The next module introduces the Page Object Model — the pattern that keeps your step definitions maintainable as the application grows.
