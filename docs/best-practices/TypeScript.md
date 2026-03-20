# TypeScript with Selenium

## Overview

TypeScript is a statically typed superset of JavaScript that compiles to plain JavaScript. When combined with WebdriverIO (the leading TypeScript-native browser automation framework) or with `selenium-webdriver` (the official Selenium JavaScript binding), TypeScript brings the same structural discipline to web automation that Java provides — but for teams working in a JavaScript/Node.js ecosystem. This document covers setting up a professional TypeScript automation framework, implementing Page Object Model, integrating Cucumber BDD, configuring parallel execution, and producing reports.

---

## Why TypeScript for Web Automation

**Compile-time error detection.** Type errors in locators, method signatures, and return types are caught before the test runs, not during CI execution.

**IDE intelligence.** Full autocomplete, method signatures, and inline documentation for WebDriver, page objects, and step definitions.

**Refactoring confidence.** Renaming a page method propagates across all step definition files automatically in a typed project.

**Self-documenting page objects.** Method signatures show exactly what parameters are required and what type is returned.

**Team velocity.** Large teams benefit from TypeScript's contracts between page classes and step definitions.

---

## Technology Stack

| Component          | Library                    | Version |
| ------------------ | -------------------------- | ------- |
| Browser automation | WebdriverIO                | v8      |
| Language           | TypeScript                 | v5      |
| BDD framework      | `@wdio/cucumber-framework` | v8      |
| Test runner        | `@wdio/cli`                | v8      |
| Assertions         | `expect-webdriverio`       | bundled |
| Reporting          | `@wdio/allure-reporter`    | v8      |
| Reports            | `@wdio/spec-reporter`      | v8      |
| Parallel           | Built-in WDIO runner       | —       |

---

## Project Initialisation

```bash
# Create project directory
mkdir selenium-typescript-framework
cd selenium-typescript-framework

# Initialise npm
npm init -y

# Install WebdriverIO CLI and run the setup wizard
npm install -D @wdio/cli

# Run the interactive setup — choose TypeScript, Cucumber, Chrome
npx wdio config
```

The wizard generates `wdio.conf.ts` and installs all required packages. Choose:

- Framework: Cucumber
- Language: TypeScript
- Browser: Chrome
- Reporter: Spec + Allure
- Services: chromedriver

---

## Project Structure

```
selenium-typescript-framework/
├── wdio.conf.ts                     ← WebdriverIO configuration
├── tsconfig.json                    ← TypeScript configuration
├── package.json
│
├── src/
│   ├── pages/
│   │   ├── BasePage.ts              ← Abstract base with shared utilities
│   │   ├── LoginPage.ts
│   │   ├── DashboardPage.ts
│   │   └── components/
│   │       └── NavigationBar.ts
│   │
│   ├── utils/
│   │   ├── ConfigReader.ts
│   │   ├── WaitUtils.ts
│   │   └── ScreenshotUtils.ts
│   │
│   ├── data/
│   │   ├── TestDataManager.ts
│   │   └── TestDataFactory.ts
│   │
│   └── types/
│       └── index.d.ts               ← Shared type declarations
│
├── test/
│   ├── features/
│   │   ├── login.feature
│   │   └── dashboard.feature
│   │
│   ├── step-definitions/
│   │   ├── login.steps.ts
│   │   ├── dashboard.steps.ts
│   │   └── common.steps.ts
│   │
│   └── hooks/
│       └── hooks.ts
│
├── config/
│   ├── config.ts                    ← Default configuration
│   └── config.staging.ts            ← Staging overrides
│
└── reports/
    └── allure-results/
```

---

## tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "strict": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "baseUrl": ".",
    "paths": {
      "@pages/*": ["src/pages/*"],
      "@utils/*": ["src/utils/*"],
      "@data/*": ["src/data/*"],
      "@config": ["config/config.ts"]
    },
    "outDir": "./dist",
    "rootDir": ".",
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*", "test/**/*", "config/**/*", "wdio.conf.ts"],
  "exclude": ["node_modules", "dist", "reports"]
}
```

---

## wdio.conf.ts — Configuration

```typescript
import type { Options } from "@wdio/types";
import { config as baseConfig } from "./config/config";

export const config: Options.Testrunner = {
  runner: "local",
  autoCompileOpts: {
    autoCompile: true,
    tsNodeOpts: {
      transpileOnly: true,
      project: "./tsconfig.json",
    },
  },

  specs: ["./test/features/**/*.feature"],
  exclude: [],

  maxInstances: 4, // parallel instances

  capabilities: [
    {
      browserName: process.env.BROWSER || "chrome",
      acceptInsecureCerts: true,
      "goog:chromeOptions": {
        args: [
          "--headless=new",
          "--no-sandbox",
          "--disable-dev-shm-usage",
          "--window-size=1920,1080",
          "--disable-notifications",
        ],
      },
    },
  ],

  logLevel: "warn",
  bail: 0,
  waitforTimeout: 10000,
  connectionRetryTimeout: 120000,
  connectionRetryCount: 3,

  services: ["chromedriver"],

  framework: "cucumber",

  reporters: [
    "spec",
    [
      "allure",
      {
        outputDir: "reports/allure-results",
        disableWebdriverStepsReporting: false,
        useCucumberStepReporter: true,
      },
    ],
  ],

  cucumberOpts: {
    require: ["./test/step-definitions/**/*.ts", "./test/hooks/hooks.ts"],
    backtrace: false,
    requireModule: [],
    dryRun: false,
    failFast: false,
    snippets: true,
    source: true,
    strict: false,
    tagExpression: process.env.TAGS || "@smoke",
    timeout: 60000,
    ignoreUndefinedDefinitions: false,
  },
};
```

---

## Configuration Module

```typescript
// config/config.ts
export interface AppConfig {
  baseUrl: string;
  browser: string;
  headless: boolean;
  timeouts: {
    implicit: number;
    explicit: number;
    pageLoad: number;
  };
  credentials: {
    admin: { email: string; password: string };
    buyer: { email: string; password: string };
  };
  grid: {
    enabled: boolean;
    url: string;
  };
}

export const config: AppConfig = {
  baseUrl: process.env.APP_BASE_URL || "https://dev.example.com",
  browser: process.env.BROWSER || "chrome",
  headless: process.env.HEADLESS === "true",

  timeouts: {
    implicit: 0,
    explicit: 10000,
    pageLoad: 30000,
  },

  credentials: {
    admin: {
      email: process.env.ADMIN_EMAIL || "admin@example.com",
      password: process.env.ADMIN_PASSWORD || "AdminPass123",
    },
    buyer: {
      email: process.env.BUYER_EMAIL || "buyer@example.com",
      password: process.env.BUYER_PASSWORD || "BuyerPass456",
    },
  },

  grid: {
    enabled: process.env.GRID_ENABLED === "true",
    url: process.env.GRID_URL || "http://localhost:4444",
  },
};
```

---

## BasePage — Abstract Base Class

```typescript
// src/pages/BasePage.ts
import { config } from "@config";

export abstract class BasePage {
  protected baseUrl: string = config.baseUrl;

  // -----------------------------------------------------------------------
  // Navigation
  // -----------------------------------------------------------------------

  async navigate(path: string = ""): Promise<void> {
    await browser.url(`${this.baseUrl}${path}`);
  }

  async getTitle(): Promise<string> {
    return browser.getTitle();
  }

  async getCurrentUrl(): Promise<string> {
    return browser.getUrl();
  }

  // -----------------------------------------------------------------------
  // Element interaction
  // -----------------------------------------------------------------------

  async click(selector: string): Promise<void> {
    const element = await $(selector);
    await element.waitForClickable({ timeout: config.timeouts.explicit });
    await element.click();
  }

  async type(selector: string, value: string): Promise<void> {
    const element = await $(selector);
    await element.waitForDisplayed({ timeout: config.timeouts.explicit });
    await element.clearValue();
    await element.setValue(value);
  }

  async getText(selector: string): Promise<string> {
    const element = await $(selector);
    await element.waitForDisplayed({ timeout: config.timeouts.explicit });
    return element.getText();
  }

  async getAttribute(
    selector: string,
    attribute: string,
  ): Promise<string | null> {
    const element = await $(selector);
    return element.getAttribute(attribute);
  }

  async isDisplayed(selector: string): Promise<boolean> {
    try {
      const element = await $(selector);
      return element.isDisplayed();
    } catch {
      return false;
    }
  }

  async isEnabled(selector: string): Promise<boolean> {
    const element = await $(selector);
    return element.isEnabled();
  }

  // -----------------------------------------------------------------------
  // Wait utilities
  // -----------------------------------------------------------------------

  async waitForVisible(selector: string, timeout?: number): Promise<void> {
    const element = await $(selector);
    await element.waitForDisplayed({
      timeout: timeout ?? config.timeouts.explicit,
    });
  }

  async waitForNotVisible(selector: string, timeout?: number): Promise<void> {
    const element = await $(selector);
    await element.waitForDisplayed({
      timeout: timeout ?? config.timeouts.explicit,
      reverse: true,
    });
  }

  async waitForText(selector: string, expectedText: string): Promise<void> {
    await browser.waitUntil(
      async () => {
        const element = await $(selector);
        const text = await element.getText();
        return text.includes(expectedText);
      },
      {
        timeout: config.timeouts.explicit,
        timeoutMsg: `Expected text '${expectedText}' not found in '${selector}'`,
      },
    );
  }

  async waitForUrl(partialUrl: string): Promise<void> {
    await browser.waitUntil(
      async () => {
        const url = await browser.getUrl();
        return url.includes(partialUrl);
      },
      {
        timeout: config.timeouts.explicit,
        timeoutMsg: `URL did not contain '${partialUrl}'`,
      },
    );
  }

  // -----------------------------------------------------------------------
  // Scroll utilities
  // -----------------------------------------------------------------------

  async scrollToElement(selector: string): Promise<void> {
    const element = await $(selector);
    await element.scrollIntoView({ block: "center" });
  }

  async scrollToBottom(): Promise<void> {
    await browser.execute(() => window.scrollTo(0, document.body.scrollHeight));
  }

  // -----------------------------------------------------------------------
  // Screenshot
  // -----------------------------------------------------------------------

  async takeScreenshot(name: string): Promise<string> {
    return browser.saveScreenshot(`./reports/screenshots/${name}.png`);
  }
}
```

---

## LoginPage — Page Object

```typescript
// src/pages/LoginPage.ts
import { BasePage } from "./BasePage";
import { DashboardPage } from "./DashboardPage";

export class LoginPage extends BasePage {
  // Selectors — using data-testid for stability
  private readonly selectors = {
    usernameField: '[data-testid="username-input"]',
    passwordField: '[data-testid="password-input"]',
    loginButton: '[data-testid="login-submit-btn"]',
    errorMessage: '[data-testid="login-error-msg"]',
    forgotPassword: '[data-testid="forgot-password-link"]',
  } as const;

  async open(): Promise<void> {
    await this.navigate("/login");
    await this.waitForVisible(this.selectors.usernameField);
  }

  async enterUsername(username: string): Promise<LoginPage> {
    await this.type(this.selectors.usernameField, username);
    return this;
  }

  async enterPassword(password: string): Promise<LoginPage> {
    await this.type(this.selectors.passwordField, password);
    return this;
  }

  async clickLogin(): Promise<DashboardPage> {
    await this.click(this.selectors.loginButton);
    const dashboardPage = new DashboardPage();
    await dashboardPage.waitForPageLoad();
    return dashboardPage;
  }

  async login(email: string, password: string): Promise<DashboardPage> {
    await this.enterUsername(email);
    await this.enterPassword(password);
    return this.clickLogin();
  }

  async getErrorMessage(): Promise<string> {
    await this.waitForVisible(this.selectors.errorMessage);
    return this.getText(this.selectors.errorMessage);
  }

  async isErrorDisplayed(): Promise<boolean> {
    return this.isDisplayed(this.selectors.errorMessage);
  }
}
```

---

## DashboardPage

```typescript
// src/pages/DashboardPage.ts
import { BasePage } from "./BasePage";
import { LoginPage } from "./LoginPage";

export class DashboardPage extends BasePage {
  private readonly selectors = {
    welcomeMessage: '[data-testid="welcome-message"]',
    userAvatar: '[data-testid="user-avatar"]',
    logoutButton: '[data-testid="logout-btn"]',
    pageLoader: '[data-testid="page-loader"]',
  } as const;

  async waitForPageLoad(): Promise<void> {
    await this.waitForNotVisible(this.selectors.pageLoader, 15000);
    await this.waitForVisible(this.selectors.welcomeMessage);
  }

  async getWelcomeMessage(): Promise<string> {
    return this.getText(this.selectors.welcomeMessage);
  }

  async isLoaded(): Promise<boolean> {
    return this.isDisplayed(this.selectors.welcomeMessage);
  }

  async logout(): Promise<LoginPage> {
    await this.click(this.selectors.logoutButton);
    const loginPage = new LoginPage();
    await loginPage.waitForVisible('[data-testid="username-input"]');
    return loginPage;
  }
}
```

---

## Type Declarations

```typescript
// src/types/index.d.ts

export interface User {
  email: string;
  password: string;
  role: UserRole;
  firstName: string;
  lastName: string;
  active: boolean;
}

export type UserRole = "ADMIN" | "MANAGER" | "BUYER" | "VIEWER";

export interface Product {
  id: string;
  name: string;
  category: string;
  price: number;
  stock: number;
  available: boolean;
}

export interface ScenarioContext {
  currentUser?: User;
  createdUserEmails: string[];
  capturedOrderId?: string;
}
```

---

## Cucumber Hooks

```typescript
// test/hooks/hooks.ts
import { Before, After, Status } from "@cucumber/cucumber";
import { LoginPage } from "@pages/LoginPage";

Before(async function (this: any, scenario) {
  console.log(`Starting: ${scenario.pickle.name}`);
  this.loginPage = new LoginPage();
  this.context = { createdUserEmails: [] };
});

After(async function (this: any, scenario) {
  if (scenario.result?.status === Status.FAILED) {
    // Attach screenshot to Allure report
    const screenshot = await browser.takeScreenshot();
    await this.attach(Buffer.from(screenshot, "base64"), "image/png");

    // Log current URL
    const url = await browser.getUrl();
    await this.attach(`URL at failure: ${url}`, "text/plain");
  }
});
```

---

## Step Definitions

```typescript
// test/step-definitions/login.steps.ts
import { Given, When, Then } from "@cucumber/cucumber";
import { expect } from "expect-webdriverio";
import { LoginPage } from "@pages/LoginPage";
import { DashboardPage } from "@pages/DashboardPage";
import { config } from "@config";

let loginPage: LoginPage;
let dashboardPage: DashboardPage;

Given("the user is on the login page", async function () {
  loginPage = new LoginPage();
  await loginPage.open();
});

When(
  "the user logs in with username {string} and password {string}",
  async function (username: string, password: string) {
    dashboardPage = await loginPage.login(username, password);
  },
);

When("the admin user logs in", async function () {
  const { email, password } = config.credentials.admin;
  dashboardPage = await loginPage.login(email, password);
});

Then("the dashboard should be displayed", async function () {
  const loaded = await dashboardPage.isLoaded();
  expect(loaded).toBe(true);
});

Then(
  "the welcome message should contain {string}",
  async function (expectedText: string) {
    const message = await dashboardPage.getWelcomeMessage();
    expect(message).toContain(expectedText);
  },
);

Then("an error message should be displayed", async function () {
  const displayed = await loginPage.isErrorDisplayed();
  expect(displayed).toBe(true);
});

Then("the error should say {string}", async function (expected: string) {
  const actual = await loginPage.getErrorMessage();
  expect(actual).toBe(expected);
});
```

---

## Running Tests

```bash
# Run with default config (smoke tests, chrome)
npx wdio run wdio.conf.ts

# Run specific tag
TAGS="@regression" npx wdio run wdio.conf.ts

# Run on Firefox
BROWSER=firefox npx wdio run wdio.conf.ts

# Run against staging
APP_BASE_URL=https://staging.example.com npx wdio run wdio.conf.ts

# Run headless (already default in wdio.conf.ts)
HEADLESS=true npx wdio run wdio.conf.ts

# Generate and open Allure report
npx allure generate reports/allure-results -o reports/allure-report --clean
npx allure open reports/allure-report
```

---

## GitHub Actions for TypeScript WDIO

```yaml
name: TypeScript WDIO Tests

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install Dependencies
        run: npm ci

      - name: Install Chrome
        uses: browser-actions/setup-chrome@v1

      - name: Run Tests
        run: TAGS="@regression" npx wdio run wdio.conf.ts
        env:
          APP_BASE_URL: ${{ secrets.STAGING_BASE_URL }}
          ADMIN_EMAIL: ${{ secrets.ADMIN_EMAIL }}
          ADMIN_PASSWORD: ${{ secrets.ADMIN_PASSWORD }}

      - name: Generate Allure Report
        if: always()
        run: npx allure generate reports/allure-results -o reports/allure-report --clean

      - name: Upload Allure Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: allure-report-${{ github.run_number }}
          path: reports/allure-report/
          retention-days: 14
```

---

## TypeScript vs Java — Key Differences

| Aspect            | Java (Selenium)                 | TypeScript (WDIO)                    |
| ----------------- | ------------------------------- | ------------------------------------ |
| Driver management | Manual `WebDriver` lifecycle    | WDIO handles it automatically        |
| Async handling    | Synchronous API                 | `async/await` required everywhere    |
| Page Factory      | `@FindBy` annotations           | Not applicable — use `$()` selectors |
| Parallelism       | TestNG + ThreadLocal            | WDIO `maxInstances` config           |
| Type safety       | Compile-time (Java types)       | Compile-time (TypeScript types)      |
| Ecosystem         | Maven, extensive Java libs      | npm, Node.js ecosystem               |
| CI runner         | Maven Surefire + TestNG         | WDIO CLI                             |
| Reporting         | Extent, Allure, Cucumber native | Allure, Spec, Cucumber native        |

---

## Summary

TypeScript with WebdriverIO brings static typing, IDE intelligence, and modern async/await patterns to browser automation for JavaScript teams. The Page Object Model translates naturally — `BasePage` provides shared utilities, page classes expose typed fluent methods, and step definitions remain thin delegators. WebdriverIO's built-in runner handles browser management, parallel execution, and Allure reporting with minimal configuration. The `wdio.conf.ts` file is the single configuration entry point for browser selection, tag filtering, parallel instance count, and reporter setup — making environment switching and CI integration straightforward.

---

## Related Topics

- IntroPOM.md — Page Object Model concepts apply equally to TypeScript
- StableLocators.md — data-testid strategy works identically in TypeScript
- ParallelExecution.md — maxInstances in WDIO vs thread-count in TestNG
- GitHubActions.md — CI workflow patterns applicable to Node.js/WDIO projects
- AllureReports.md — Allure reporting concepts shared between Java and TypeScript
