# Page Object Model with Playwright

In this lab you will refactor your step definitions to use the **Page Object Model (POM)** — the most important design pattern in browser automation. A Page Object wraps the selectors and interactions for a single page or component into a class, so that when the UI changes you update one file instead of hunting through dozens of step definitions. You will build Page Objects for the login page and a dashboard, then wire them into Cucumber via the World object.

**Prerequisites:** Lab 2-2 completed. `bdd-lab/features/support/world.ts` exists with `PlaywrightWorld`.

---

## Step 1 — Why Page Objects?

Without Page Objects, selectors are scattered across step definitions:

```typescript
// Without POM — selector duplicated everywhere
When("I enter username", async function (this: PlaywrightWorld, username: string) {
  await this.page.fill("#username", username);  // selector here
});

When("I clear the username field", async function (this: PlaywrightWorld) {
  await this.page.fill("#username", "");         // same selector again
});

Then("username field is highlighted", async function (this: PlaywrightWorld) {
  await expect(this.page.locator("#username")).toHaveClass(/error/); // and again
});
```

When the developer renames `#username` to `#login-username`, all three steps break. With a Page Object, the selector lives in exactly one place.

Create the pages directory:

```bash
mkdir -p bdd-lab/pages
```

> **The rule of Page Objects:** A Page Object knows *how* to interact with a page; step definitions know *what* to do in a scenario. No Playwright calls should appear in step definitions — only calls to Page Object methods.

---

## Step 2 — The LoginPage Object

```typescript
// pages/login-page.ts

import { Page, Locator, expect } from "@playwright/test";

export class LoginPage {
  readonly page: Page;

  // Locators — defined once, used everywhere
  readonly usernameInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.usernameInput = page.locator("#username");
    this.passwordInput = page.locator("#password");
    this.submitButton = page.locator("#submit");
    this.errorMessage = page.locator("#error");
  }

  async navigate() {
    await this.page.goto("https://practicetestautomation.com/practice-test-login/");
  }

  async login(username: string, password: string) {
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async expectError(message: string) {
    await expect(this.errorMessage).toBeVisible();
    await expect(this.errorMessage).toContainText(message);
  }

  async expectErrorVisible() {
    await expect(this.errorMessage).toBeVisible();
  }
}
```

---

## Step 3 — The DashboardPage Object

```typescript
// pages/dashboard-page.ts

import { Page, Locator, expect } from "@playwright/test";

export class DashboardPage {
  readonly page: Page;
  readonly pageTitle: Locator;
  readonly logoutButton: Locator;

  constructor(page: Page) {
    this.page = page;
    this.pageTitle = page.locator(".post-title");
    this.logoutButton = page.locator("a[href*='logged-out']");
  }

  async expectLoginSuccess() {
    await expect(this.pageTitle).toContainText("Logged In Successfully");
  }

  async logout() {
    await this.logoutButton.click();
  }
}
```

---

## Step 4 — Attach Page Objects to the World

The World is the right place to instantiate Page Objects — it is created fresh per scenario and gives all steps in that scenario access to the same object instances.

```typescript
// features/support/world.ts  (update)

import { World, IWorldOptions, setWorldConstructor } from "@cucumber/cucumber";
import { Browser, BrowserContext, Page, chromium } from "playwright";
import { LoginPage } from "../../pages/login-page";
import { DashboardPage } from "../../pages/dashboard-page";

export class PlaywrightWorld extends World {
  browser!: Browser;
  context!: BrowserContext;
  page!: Page;

  // Page Objects — created after openBrowser() sets this.page
  loginPage!: LoginPage;
  dashboardPage!: DashboardPage;

  constructor(options: IWorldOptions) {
    super(options);
  }

  async openBrowser() {
    this.browser = await chromium.launch({ headless: true });
    this.context = await this.browser.newContext();
    this.page = await this.context.newPage();

    // Instantiate all Page Objects here
    this.loginPage = new LoginPage(this.page);
    this.dashboardPage = new DashboardPage(this.page);
  }

  async closeBrowser() {
    await this.browser?.close();
  }
}

setWorldConstructor(PlaywrightWorld);
```

---

## Step 5 — Refactor Step Definitions to Use Page Objects

Step definitions now read like a description of user behaviour, with zero selector knowledge:

```typescript
// features/step-definitions/login.steps.ts  (refactored)

import { Given, When, Then } from "@cucumber/cucumber";
import { PlaywrightWorld } from "../support/world";

Given("I navigate to the login page", async function (this: PlaywrightWorld) {
  await this.loginPage.navigate();
});

When(
  "I enter username {string} and password {string}",
  async function (this: PlaywrightWorld, username: string, password: string) {
    await this.loginPage.login(username, password);
  }
);

When("I click the login button", async function (this: PlaywrightWorld) {
  await this.loginPage.submitButton.click();
});

Then("I should see the dashboard", async function (this: PlaywrightWorld) {
  await this.dashboardPage.expectLoginSuccess();
});

Then(
  "I should see the error message {string}",
  async function (this: PlaywrightWorld, expectedMessage: string) {
    await this.loginPage.expectError(expectedMessage);
  }
);
```

Run the full login suite to verify nothing broke:

```bash
cd bdd-lab
npx cucumber-js features/login.feature
```

The scenarios should pass exactly as before — the refactor changed the internals, not the behaviour.

> **The two-class rule:** Each Page Object should map to a single page or reusable component. A `LoginPage` and a `NavigationBar` are correct granularity. A `TestHelper` that mixes methods from three different pages is a sign the abstraction has broken down.

---

## Step 6 — A Component Page Object for Reusable UI

Some UI elements appear on multiple pages — navigation bars, modals, toast notifications. Model these as component objects that any Page Object can compose.

```typescript
// pages/navigation.ts

import { Page, Locator } from "@playwright/test";

export class Navigation {
  readonly page: Page;
  readonly homeLink: Locator;
  readonly practiceLink: Locator;

  constructor(page: Page) {
    this.page = page;
    this.homeLink = page.locator("a[href='https://practicetestautomation.com/']").first();
    this.practiceLink = page.locator("a[href*='practice']").first();
  }

  async goHome() {
    await this.homeLink.click();
  }
}
```

```typescript
// pages/dashboard-page.ts  (updated to compose Navigation)

import { Page, Locator, expect } from "@playwright/test";
import { Navigation } from "./navigation";

export class DashboardPage {
  readonly page: Page;
  readonly pageTitle: Locator;
  readonly nav: Navigation;

  constructor(page: Page) {
    this.page = page;
    this.pageTitle = page.locator(".post-title");
    this.nav = new Navigation(page);
  }

  async expectLoginSuccess() {
    await expect(this.pageTitle).toContainText("Logged In Successfully");
  }
}
```

Add Navigation to the World and write a scenario that uses it:

```gherkin
# Add to features/login.feature

  Scenario: Logged-in user can navigate home
    Given I navigate to the login page
    When I enter username "student" and password "Password123"
    And I click the login button
    Then I should see the dashboard
    When I navigate to the home page
    Then the page URL contains "practicetestautomation.com"
```

```typescript
// Add to features/step-definitions/login.steps.ts

When("I navigate to the home page", async function (this: PlaywrightWorld) {
  await this.dashboardPage.nav.goHome();
});

Then("the page URL contains {string}", async function (this: PlaywrightWorld, urlPart: string) {
  const { expect } = await import("@playwright/test");
  await expect(this.page).toHaveURL(new RegExp(urlPart));
});
```

```bash
npx cucumber-js features/login.feature --name "Logged-in user"
```

You now have a clean, maintainable Page Object layer: selectors in Page Objects, behaviour in step definitions, shared instances in the World, and reusable components composed into pages. The final lab adds HTML reporting, tag-based test selection, and GitHub Actions CI so your suite can run automatically on every push.
