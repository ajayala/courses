# Gherkin Scenarios — Tables, Outlines, and Backgrounds

In this lab you will move beyond single-example scenarios and learn the Gherkin features that let you express complex requirements concisely: Scenario Outlines for data-driven testing, Data Tables for structured inputs, and Background for shared preconditions. You will apply these to a login feature — one of the most common and scenario-rich areas of any application.

**Prerequisites:** Lab 1-1 completed. `bdd-lab/` exists with Cucumber and Playwright installed.

---

## Step 1 — The Background Keyword

A `Background` block runs before every scenario in the same feature file. Use it for setup steps that every scenario shares — such as navigating to a page or establishing a logged-in state. It is the feature-file equivalent of a `beforeEach`.

Create a new feature file for a login flow:

```gherkin
# features/login.feature

Feature: User login
  As a registered user
  I want to log in to the application
  So that I can access my account

  Background:
    Given I navigate to the login page

  Scenario: Successful login with valid credentials
    When I enter username "admin" and password "secret123"
    And I click the login button
    Then I should see the dashboard

  Scenario: Failed login with wrong password
    When I enter username "admin" and password "wrongpass"
    And I click the login button
    Then I should see the error message "Invalid credentials"

  Scenario: Failed login with empty fields
    When I click the login button
    Then I should see the error message "Username is required"
```

The `Background` step `Given I navigate to the login page` runs automatically before each of the three scenarios. This eliminates duplication without hiding which page each scenario starts on.

> **Background vs hooks:** `Background` is visible in the feature file — anyone reading the spec can see the precondition. Hooks (covered in the next lab) run invisibly. Prefer `Background` for domain-level setup that stakeholders should know about; use hooks for technical plumbing like starting a browser.

---

## Step 2 — Stub Step Definitions for the Login Feature

Before writing a real implementation, create stubs so you can run the feature file and see `pending` rather than `undefined`:

```typescript
// features/step-definitions/login.steps.ts

import { Given, When, Then } from "@cucumber/cucumber";

Given("I navigate to the login page", async () => {
  // Will be implemented with Playwright
  return "pending";
});

When("I enter username {string} and password {string}", async (username: string, password: string) => {
  return "pending";
});

When("I click the login button", async () => {
  return "pending";
});

Then("I should see the dashboard", async () => {
  return "pending";
});

Then("I should see the error message {string}", async (message: string) => {
  return "pending";
});
```

```bash
cd bdd-lab
npx cucumber-js features/login.feature
```

You should see:
```
3 scenarios (3 pending)
9 steps (3 pending, 6 skipped)
```

> **Returning `"pending"`** from a step definition tells Cucumber the step is not yet implemented. It is different from a failure — Cucumber marks it as pending (yellow) and skips the remaining steps in the scenario. This is the correct way to track work in progress.

---

## Step 3 — Scenario Outline

A `Scenario Outline` replaces a scenario's literal values with placeholders (`<angle-brackets>`) and supplies multiple rows of data in an `Examples` table. Cucumber runs one scenario per row — giving you data-driven tests without repeating the scenario structure.

Add a Scenario Outline to `features/login.feature`:

```gherkin
  Scenario Outline: Login attempt with various credentials
    When I enter username "<username>" and password "<password>"
    And I click the login button
    Then I should see "<outcome>"

    Examples:
      | username | password  | outcome             |
      | admin    | secret123 | the dashboard       |
      | admin    | wrong     | Invalid credentials |
      | guest    | secret123 | Invalid credentials |
      |          | secret123 | Username is required|
```

```bash
npx cucumber-js features/login.feature
```

Cucumber now runs four scenarios — one for each row in the `Examples` table:

```
8 scenarios (8 pending)
24 steps (8 pending, 16 skipped)
```

> **Use Scenario Outlines for boundary conditions.** They are especially effective for form validation: you can cover valid inputs, blank fields, fields that are too long, and special characters all in one concise block. Keep the Example table focused — a table with 20 rows is a sign the feature needs to be split.

---

## Step 4 — Data Tables

A `Data Table` passes structured data directly into a single step — useful when a step needs multiple related values that don't fit naturally as a single parameter. Unlike `Scenario Outline`, a `Data Table` is part of one specific step, not repeated across scenarios.

Add a scenario that creates a new user to `features/login.feature`:

```gherkin
  Scenario: Registering a new user
    Given the following user details:
      | field     | value              |
      | username  | newuser            |
      | email     | new@example.com    |
      | password  | StrongPass1!       |
      | full_name | New User           |
    When I submit the registration form
    Then I should see the dashboard
```

Implement the step definition that reads the table:

```typescript
// Add to features/step-definitions/login.steps.ts

import { DataTable } from "@cucumber/cucumber";

Given("the following user details:", async (table: DataTable) => {
  // table.rowsHash() converts a two-column key/value table to an object
  const details = table.rowsHash();
  console.log("Registering:", details);
  // details = { username: 'newuser', email: 'new@example.com', ... }
  return "pending";
});

When("I submit the registration form", async () => {
  return "pending";
});
```

```bash
npx cucumber-js features/login.feature --name "Registering"
```

| DataTable method | Use case |
|-----------------|----------|
| `table.rows()` | Array of rows (each row is an array of strings), header excluded |
| `table.hashes()` | Array of objects using header row as keys — for lists of records |
| `table.rowsHash()` | Single object from a two-column key/value table |
| `table.raw()` | Raw 2D array including the header row |

---

## Step 5 — The `And` and `But` Keywords

`And` and `But` are aliases for the previous keyword (`Given`, `When`, or `Then`). They exist purely for readability — Cucumber treats them identically to their parent keyword.

```gherkin
# features/checkout.feature

Feature: Shopping cart checkout

  Background:
    Given I am logged in as "alice"
    And my cart contains 3 items

  Scenario: Successful checkout
    When I proceed to checkout
    And I enter valid payment details
    And I confirm the order
    Then I should see the order confirmation page
    And I should receive a confirmation email
    But my cart should be empty

  Scenario: Checkout fails with expired card
    When I proceed to checkout
    And I enter expired payment details
    Then I should see the error "Your card has expired"
    But my order should not be placed
```

Create stub step definitions for the checkout feature:

```typescript
// features/step-definitions/checkout.steps.ts

import { Given, When, Then } from "@cucumber/cucumber";

Given("I am logged in as {string}", async (username: string) => {
  return "pending";
});

Given("my cart contains {int} items", async (count: number) => {
  return "pending";
});

When("I proceed to checkout", async () => { return "pending"; });
When("I enter valid payment details", async () => { return "pending"; });
When("I enter expired payment details", async () => { return "pending"; });
When("I confirm the order", async () => { return "pending"; });

Then("I should see the order confirmation page", async () => { return "pending"; });
Then("I should receive a confirmation email", async () => { return "pending"; });
Then("my cart should be empty", async () => { return "pending"; });
Then("I should see the error {string}", async (msg: string) => { return "pending"; });
Then("my order should not be placed", async () => { return "pending"; });
```

```bash
npx cucumber-js features/checkout.feature
```

---

## Step 6 — Rule and Example (Gherkin 6+)

Modern Gherkin supports `Rule` to group related scenarios under a business rule, and `Example` as a synonym for `Scenario`. This gives feature files a cleaner hierarchy when a feature has multiple distinct rules.

```gherkin
# features/password-policy.feature

Feature: Password policy enforcement

  Rule: Passwords must meet minimum complexity requirements

    Example: Password is too short
      Given I am on the registration page
      When I enter a password "abc"
      Then I should see "Password must be at least 8 characters"

    Example: Password has no uppercase letter
      Given I am on the registration page
      When I enter a password "alllowercase1!"
      Then I should see "Password must contain at least one uppercase letter"

  Rule: Account is locked after repeated failures

    Example: Account locks after 5 failed attempts
      Given I am on the login page
      When I fail to log in 5 times with username "alice"
      Then I should see "Your account has been locked"
```

Create the stub file:

```typescript
// features/step-definitions/password.steps.ts

import { Given, When, Then } from "@cucumber/cucumber";

Given("I am on the registration page", async () => { return "pending"; });
Given("I am on the login page", async () => { return "pending"; });

When("I enter a password {string}", async (password: string) => { return "pending"; });
When("I fail to log in {int} times with username {string}", async (times: number, username: string) => {
  return "pending";
});

Then("I should see {string}", async (message: string) => { return "pending"; });
```

```bash
npx cucumber-js features/password-policy.feature
```

You now have a solid command of Gherkin: Background, Scenario Outline with Examples, Data Tables, And/But connectors, and Rule/Example grouping. The next lab moves into the TypeScript layer — parameters, hooks, and the World object — where you will wire these scenarios to real Playwright browser automation.
