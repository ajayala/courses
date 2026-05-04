# Reporting, Tagging, and Running in CI

In this lab you will add professional HTML reporting with `multiple-cucumber-html-reporter`, organise your test suite with tags so you can run smoke tests separately from regression tests, configure retry logic for flaky tests, and set up a GitHub Actions workflow that runs your suite on every push and publishes the HTML report as a build artifact. These are the practices that turn a collection of passing tests into a maintainable, visible quality gate.

**Prerequisites:** Lab 3-1 completed. The full Page Object layer is in place and `npx cucumber-js` passes.

---

## Step 1 — Tagging Scenarios

Tags are `@annotations` placed on features, scenarios, or examples rows. They let you select subsets of your suite to run — essential for keeping fast feedback loops in CI while still running the full regression nightly.

Update your feature files to add meaningful tags:

```gherkin
# features/login.feature  (add tags)

@login @smoke
Feature: User login

  Background:
    Given I navigate to the login page

  @happy-path @smoke
  Scenario: Successful login with valid credentials
    When I enter username "student" and password "Password123"
    And I click the login button
    Then I should see the dashboard

  @negative @regression
  Scenario: Failed login with wrong password
    When I enter username "student" and password "wrongpass"
    And I click the login button
    Then I should see the error message "Your password is invalid!"

  @negative @regression
  Scenario Outline: Login attempt with various credentials
    When I enter username "<username>" and password "<password>"
    And I click the login button
    Then I should see the error message "<error>"

    Examples:
      | username | password  | error                       |
      | baduser  | Password123 | Your username is invalid! |
      | student  | badpass     | Your password is invalid! |
```

Run by tag:

```bash
cd bdd-lab

# Only smoke tests (fast — run on every commit)
npx cucumber-js --tags "@smoke"

# Only regression tests (slower — run nightly or on release branches)
npx cucumber-js --tags "@regression"

# Exclude a tag
npx cucumber-js --tags "not @wip"

# Boolean tag expressions
npx cucumber-js --tags "@smoke and @login"
npx cucumber-js --tags "@happy-path or @negative"
```

> **Standard tag conventions:** `@smoke` — fast critical-path tests; `@regression` — full suite; `@wip` — work in progress, excluded from CI; `@slow` — tests with long waits; `@flaky` — tests known to be intermittent (should be fixed, but tag them while pending). Keep the number of tag categories small — five or fewer is usually enough.

---

## Step 2 — JSON Reporter and Multiple HTML Reports

Cucumber can output multiple formats simultaneously. The `json` formatter produces a machine-readable file that third-party reporters can transform into rich HTML.

Install the HTML reporter:

```bash
npm install --save-dev multiple-cucumber-html-reporter
```

Update the `cucumber` config in `package.json`:

```json
{
  "cucumber": {
    "require": ["features/support/**/*.ts", "features/step-definitions/**/*.ts"],
    "requireModule": ["ts-node/register"],
    "format": [
      "progress-bar",
      "json:reports/cucumber-report.json",
      "html:reports/cucumber-report.html"
    ],
    "publishQuiet": true
  }
}
```

Create a reporter script that runs after Cucumber:

```javascript
// scripts/generate-report.js

const report = require("multiple-cucumber-html-reporter");

report.generate({
  jsonDir: "reports",
  reportPath: "reports/html",
  metadata: {
    browser: { name: "chromium", version: "latest" },
    device: "Local test machine",
    platform: { name: "Linux" },
  },
  customData: {
    title: "Test Execution Info",
    data: [
      { label: "Project", value: "BDD Lab" },
      { label: "Release", value: "1.0.0" },
      { label: "Execution Start Time", value: new Date().toISOString() },
    ],
  },
});
```

Add npm scripts to `package.json`:

```json
{
  "scripts": {
    "test": "npx cucumber-js",
    "test:smoke": "npx cucumber-js --tags '@smoke'",
    "test:regression": "npx cucumber-js --tags '@regression'",
    "report": "node scripts/generate-report.js",
    "test:report": "npx cucumber-js && node scripts/generate-report.js"
  }
}
```

```bash
mkdir -p scripts
node scripts/generate-report.js   # will fail until cucumber has run first
npm run test:report
```

---

## Step 3 — Retry Logic for Flaky Tests

Network-dependent browser tests occasionally fail for transient reasons. Cucumber's `--retry` flag reruns failed scenarios automatically before marking them as failed.

```bash
# Retry failed scenarios up to 2 times before counting as a failure
npx cucumber-js --retry 2

# Only retry scenarios tagged @flaky
npx cucumber-js --retry 2 --retry-tag-filter "@flaky"
```

Add retry to the npm test script:

```json
{
  "scripts": {
    "test": "npx cucumber-js --retry 1",
    "test:ci": "npx cucumber-js --retry 2 --format json:reports/cucumber-report.json"
  }
}
```

You can also configure this directly in `package.json`:

```json
{
  "cucumber": {
    "retry": 1,
    "retryTagFilter": "not @no-retry"
  }
}
```

> **Retry is a band-aid, not a cure.** It reduces false failures from transient network issues, but a scenario that fails 30% of the time needs to be fixed. Track your retry rate — if a specific scenario retries frequently, investigate the root cause rather than raising the retry limit.

---

## Step 4 — Parallel Execution

Running scenarios in parallel is the single biggest speedup available to large test suites. Cucumber supports parallel execution natively with the `--parallel` flag.

```bash
# Run with 4 parallel workers
npx cucumber-js --parallel 4
```

Parallel execution requires that each scenario is fully isolated — no shared mutable state between scenarios. The World object already handles this (a fresh World per scenario), but you must ensure your test data does not collide.

Add a `cucumber.js` configuration file (an alternative to inline `package.json` config) to separate dev and CI settings:

```javascript
// cucumber.js  (project root)

const common = [
  "features/**/*.feature",
  "--require-module ts-node/register",
  "--require features/support/**/*.ts",
  "--require features/step-definitions/**/*.ts",
  "--publish-quiet",
].join(" ");

module.exports = {
  default: `${common} --format progress-bar`,
  ci: `${common} --parallel 4 --retry 2 --format json:reports/cucumber-report.json`,
};
```

```bash
# Run with the 'ci' profile
npx cucumber-js --profile ci
```

---

## Step 5 — GitHub Actions CI Workflow

Create a workflow that installs dependencies, runs the test suite, and uploads the HTML report as a downloadable artifact.

```bash
mkdir -p .github/workflows
```

```yaml
# .github/workflows/bdd-tests.yml

name: BDD Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Run BDD tests
        run: npx cucumber-js --profile ci

      - name: Generate HTML report
        if: always()   # run even if tests failed
        run: node scripts/generate-report.js

      - name: Upload test report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: cucumber-report
          path: reports/html/
          retention-days: 30
```

Add a `.gitignore` to exclude generated reports and node_modules:

```bash
cat > .gitignore << 'EOF'
node_modules/
reports/
.venv/
EOF
```

> **`if: always()`** ensures the report generation and upload steps run even when tests fail — which is exactly when you need the report most. Without it, a test failure would skip the upload and leave you with no evidence.

---

## Step 6 — Smoke Gate and Nightly Regression

A common CI strategy is to run only smoke tests on every pull request (fast feedback) and run the full regression suite on a nightly schedule.

```yaml
# .github/workflows/bdd-tests.yml  (extend with nightly job)

  smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - name: Run smoke tests
        run: npm run test:smoke
      - name: Upload smoke report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: smoke-report
          path: reports/html/
          retention-days: 7

---

name: Nightly Regression

on:
  schedule:
    - cron: "0 2 * * *"   # 02:00 UTC every night

jobs:
  regression:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - name: Run full regression
        run: npx cucumber-js --tags "@regression" --profile ci
      - name: Generate report
        if: always()
        run: node scripts/generate-report.js
      - name: Upload regression report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: regression-report-${{ github.run_number }}
          path: reports/html/
          retention-days: 60
```

Save the nightly workflow to a separate file:

```bash
# The smoke + PR workflow stays in bdd-tests.yml
# Add the nightly content to:
touch .github/workflows/nightly-regression.yml
```

Congratulations — you have built a complete professional BDD suite: Gherkin feature files that communicate requirements to the whole team, Playwright automation that drives a real browser, Page Objects that keep selectors maintainable, hooks and the World that manage test lifecycle correctly, tags that enable selective test execution, retry logic for resilience, HTML reports for visibility, and a CI pipeline that gates merges and runs regression nightly. These are the practices that QA engineers and SDETs use in production.
