# Your First Feature File — Gherkin, Cucumber, and Playwright

In this lab you will set up a Cucumber + Playwright project from scratch, write your first Gherkin feature file, implement step definitions in TypeScript, and run a real browser test against a live website. By the end you will understand why BDD (Behaviour-Driven Development) exists, how the three layers of the stack fit together, and how to make a failing scenario go green.

**Prerequisites:** Node.js 18 or later (`node --version`). A terminal and VS Code.

---

## Step 1 — What Is BDD and Why Does It Matter?

BDD (Behaviour-Driven Development) bridges the gap between business requirements and automated tests. Instead of writing tests that describe implementation details, BDD tests describe *user-observable behaviour* in plain language that non-technical stakeholders can read and verify.

The three layers you will use in this course:

| Layer | Tool | Role |
|-------|------|------|
| Specification language | **Gherkin** | Plain-English scenarios written by the whole team |
| Test runner | **Cucumber.js** | Parses `.feature` files and maps steps to code |
| Browser automation | **Playwright** | Drives a real browser to execute each step |

```bash
mkdir bdd-lab
cd bdd-lab
npm init -y
```

The `npm init -y` creates a `package.json` with defaults. This is the project root for the entire course.

> **Why not just use Playwright's built-in test runner?** Playwright Test is excellent for developer-focused tests. Cucumber adds a collaboration layer: product managers, QA leads, and developers can all contribute to `.feature` files without knowing TypeScript. Many teams use both — Cucumber for acceptance tests driven by requirements, Playwright Test for component and integration tests.

---

## Step 2 — Install Dependencies

```bash
npm install --save-dev @cucumber/cucumber playwright @playwright/test
npx playwright install chromium
```

Create the project directory structure:

```bash
mkdir -p features/step-definitions
mkdir -p features/support
```

Your project should look like this:

```
bdd-lab/
├── features/
│   ├── step-definitions/   ← TypeScript step implementations
│   ├── support/            ← hooks, world object, shared setup
│   └── *.feature           ← Gherkin scenario files
├── package.json
└── node_modules/
```

Configure Cucumber in `package.json` so it knows where to find everything:

```json
{
  "cucumber": {
    "require": ["features/support/**/*.ts", "features/step-definitions/**/*.ts"],
    "requireModule": ["ts-node/register"],
    "format": ["progress-bar", "html:reports/cucumber-report.html"],
    "publishQuiet": true
  }
}
```

```bash
npm install --save-dev ts-node typescript @types/node
npx tsc --init --target ES2020 --module commonjs --strict false --esModuleInterop true
```

> **`requireModule: ["ts-node/register"]`** lets Cucumber load `.ts` files directly without a separate compile step. This is the standard setup for TypeScript Cucumber projects. For production CI you would pre-compile, but for development the direct approach is faster.

---

## Step 3 — Your First Feature File

A feature file describes a single feature of your application. Each scenario is an independent, concrete example of how that feature should behave. The keywords `Given`, `When`, and `Then` form the three-act structure of every scenario.

```gherkin
# features/search.feature

Feature: Web search
  As a user
  I want to search for information on the web
  So that I can find answers quickly

  Scenario: Searching for a term returns relevant results
    Given I am on the search engine homepage
    When I search for "Playwright browser automation"
    Then the page title contains "Playwright"
```

The **Given** sets up the starting state.
The **When** describes the action the user takes.
The **Then** asserts the observable outcome.

> **One scenario = one behaviour.** Resist the urge to chain multiple `When/Then` pairs into a single scenario. Each scenario should be independently runnable and describe exactly one thing. Long scenarios are a sign that the feature file is being used as a test script rather than a specification.

---

## Step 4 — Implement Step Definitions

Step definitions connect each Gherkin step to runnable TypeScript code. Cucumber matches the text in the feature file to a `Given/When/Then` function using either a string or a regular expression.

```typescript
// features/step-definitions/search.steps.ts

import { Given, When, Then } from "@cucumber/cucumber";
import { Browser, BrowserContext, Page, chromium } from "playwright";

let browser: Browser;
let context: BrowserContext;
let page: Page;

Given("I am on the search engine homepage", async () => {
  browser = await chromium.launch({ headless: true });
  context = await browser.newContext();
  page = await context.newPage();
  await page.goto("https://www.bing.com");
});

When("I search for {string}", async (searchTerm: string) => {
  await page.fill('input[name="q"]', searchTerm);
  await page.press('input[name="q"]', "Enter");
  await page.waitForLoadState("networkidle");
});

Then("the page title contains {string}", async (expectedText: string) => {
  const title = await page.title();
  if (!title.includes(expectedText)) {
    throw new Error(`Expected title to contain "${expectedText}" but got: "${title}"`);
  }
  await browser.close();
});
```

The `{string}` in step patterns is a **Cucumber expression** — it captures the quoted value in the step text and passes it as a typed argument to the function.

---

## Step 5 — Run the Test

```bash
npx cucumber-js
```

You should see output like:

```
...

1 scenario (1 passed)
3 steps (3 passed)
0m03.241s
```

If the test fails, Cucumber prints exactly which step failed and why:

```
✗ Then the page title contains "Playwright"
    Error: Expected title to contain "Playwright" but got: "Playwright browser automation - Bing"
```

> **`waitForLoadState("networkidle")`** tells Playwright to wait until there are no more than 2 network connections for 500ms. This is safer than a fixed `sleep()` but can be slow on flaky networks. In later labs you will learn more targeted waits using `waitForSelector` and `expect(locator).toBeVisible()`.

Run with `--format @cucumber/pretty-formatter` for coloured output during development:

```bash
npx cucumber-js --format @cucumber/pretty-formatter
```

---

## Step 6 — Add a Failing Scenario and Make It Pass

The BDD workflow is: write a failing scenario → run it → write just enough code to make it pass. This keeps your tests honest and ensures every scenario is actually testing something.

Add a second scenario to `features/search.feature`:

```gherkin
  Scenario: Searching for nothing shows the homepage
    Given I am on the search engine homepage
    When I search for ""
    Then the page title contains "Bing"
```

Run the tests:

```bash
npx cucumber-js
```

Both scenarios share the `Given I am on the search engine homepage` step. Right now each step definition manages its own browser — which is wasteful. The next lab introduces the **World object** and **hooks** to share setup and teardown properly.

```bash
mkdir -p reports
```

Open `reports/cucumber-report.html` in a browser to see the HTML report Cucumber generated.

You have wired together Gherkin, Cucumber, and Playwright into a working BDD stack. The three-layer separation — specification, runner, automation — is what makes this approach scale: feature files stay readable by the whole team while step definitions evolve independently. The next lab deepens your Gherkin skills with scenario outlines, data tables, and backgrounds.
