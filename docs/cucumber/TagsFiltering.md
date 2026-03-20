# Tags and Filtering

## Overview

Tags are labels attached to features, scenarios, and examples blocks in Cucumber. They control which tests run in a given execution, enable reporting categories, drive conditional hook behavior, and serve as the primary mechanism for organizing a large test suite by priority, type, or module.

A tag begins with `@` and can be placed on any `Feature`, `Scenario`, `Scenario Outline`, or `Examples` block.

---

## Defining Tags in Feature Files

### Feature-Level Tags

A tag on a `Feature` applies to every scenario within that file.

```gherkin
@authentication @regression
Feature: User Login

  Scenario: Successful login
    ...

  Scenario: Failed login
    ...
```

Both scenarios inherit the `@authentication` and `@regression` tags.

### Scenario-Level Tags

Tags on individual scenarios override nothing — they add to the feature-level tags.

```gherkin
@authentication @regression
Feature: User Login

  @smoke @positive
  Scenario: Successful login with valid credentials
    Given the user is on the login page
    When the user logs in with valid credentials
    Then the dashboard should be displayed

  @negative
  Scenario: Login fails with wrong password
    Given the user is on the login page
    When the user enters an incorrect password
    Then an error message should be shown
```

In this example:

- "Successful login" has tags: `@authentication`, `@regression`, `@smoke`, `@positive`
- "Login fails" has tags: `@authentication`, `@regression`, `@negative`

### Examples-Level Tags

Tags can be applied to individual `Examples` blocks within a `Scenario Outline`.

```gherkin
Scenario Outline: Search with different inputs
  Given the user searches for "<term>"
  Then the result count should be "<count>"

  @positive @smoke
  Examples: Valid searches
    | term    | count |
    | laptop  | 25    |
    | tablet  | 10    |

  @negative
  Examples: Invalid searches
    | term           | count |
    | xyz_not_exist  | 0     |
```

---

## Filtering Tests in the Test Runner

The `tags` attribute in `@CucumberOptions` accepts tag expressions that filter which scenarios execute.

### TestNG Runner with Tag Filter

```java
@CucumberOptions(
    features = "src/test/resources/features",
    glue = {"stepdefinitions", "hooks"},
    tags = "@smoke",
    plugin = {
        "pretty",
        "html:target/cucumber-reports/report.html"
    }
)
public class SmokeTestRunner extends AbstractTestNGCucumberTests {
}
```

### Multiple Runners for Different Suites

Professional frameworks maintain separate runner classes for different execution profiles:

```java
// Smoke runner — critical path only
@CucumberOptions(
    features = "src/test/resources/features",
    glue = {"stepdefinitions", "hooks"},
    tags = "@smoke"
)
public class SmokeRunner extends AbstractTestNGCucumberTests {}

// Regression runner — full suite
@CucumberOptions(
    features = "src/test/resources/features",
    glue = {"stepdefinitions", "hooks"},
    tags = "@regression"
)
public class RegressionRunner extends AbstractTestNGCucumberTests {}

// Negative test runner
@CucumberOptions(
    features = "src/test/resources/features",
    glue = {"stepdefinitions", "hooks"},
    tags = "@negative"
)
public class NegativeTestRunner extends AbstractTestNGCucumberTests {}
```

---

## Tag Expressions

Tag expressions follow a logical syntax using `and`, `or`, and `not`. They are passed as a string to the `tags` attribute.

### Basic Expressions

| Expression                  | Meaning                               |
| --------------------------- | ------------------------------------- |
| `@smoke`                    | Run scenarios tagged `@smoke`         |
| `not @wip`                  | Run all scenarios NOT tagged `@wip`   |
| `@smoke and @positive`      | Run scenarios tagged with BOTH        |
| `@smoke or @regression`     | Run scenarios tagged with EITHER      |
| `@regression and not @skip` | Run `@regression` but exclude `@skip` |

### Examples in @CucumberOptions

```java
// Run smoke tests only
tags = "@smoke"

// Run regression but skip work-in-progress
tags = "@regression and not @wip"

// Run authentication OR checkout tests
tags = "@authentication or @checkout"

// Run smoke tests that are also positive
tags = "@smoke and @positive"

// Run everything except disabled and wip
tags = "not @skip and not @wip"

// Run a specific module's regression tests
tags = "@orders and @regression"
```

### Combining Multiple Conditions

```java
// Run smoke AND regression tests, excluding known failures
tags = "(@smoke or @regression) and not @known-failure"
```

---

## Running Tags from the Command Line

Tags can be passed via Maven command line at execution time, overriding what is defined in the runner class. This is the standard approach in CI/CD pipelines.

```bash
# Run smoke tests
mvn test -Dcucumber.filter.tags="@smoke"

# Run regression excluding wip
mvn test -Dcucumber.filter.tags="@regression and not @wip"

# Run a specific feature module
mvn test -Dcucumber.filter.tags="@checkout"

# Run negative tests for the login module
mvn test -Dcucumber.filter.tags="@authentication and @negative"
```

The `-Dcucumber.filter.tags` system property takes precedence over the `tags` attribute in `@CucumberOptions`.

---

## Tag-Based Conditional Hooks

Hooks can be scoped to specific tags so they only execute for matching scenarios.

```java
// Only runs before scenarios tagged @authenticated
@Before("@authenticated")
public void loginUser() {
    LoginPage loginPage = new LoginPage(Hooks.getDriver());
    loginPage.login("test@example.com", "password123");
}

// Only runs before scenarios tagged @clean-db
@Before("@clean-db")
public void resetDatabase() {
    DatabaseHelper.truncateTestTables();
    DatabaseHelper.seedBaseData();
}

// Only runs before scenarios tagged @slow
@Before("@slow")
public void setExtendedTimeout() {
    Hooks.getDriver().manage().timeouts()
        .pageLoadTimeout(Duration.ofSeconds(60));
}

// Only runs after scenarios tagged @report-screenshot
@After("@report-screenshot")
public void captureScreenshot(Scenario scenario) {
    byte[] screenshot = ((TakesScreenshot) Hooks.getDriver())
        .getScreenshotAs(OutputType.BYTES);
    scenario.attach(screenshot, "image/png", "Test screenshot");
}
```

Tag-based hooks avoid running expensive setup (like database resets or login flows) for scenarios that do not need them.

---

## Recommended Tagging Strategy

A professional Selenium Cucumber framework typically uses tags across three dimensions: execution tier, scenario type, and application module.

### Execution Tier Tags

```
@smoke       — 5-10 critical scenarios that verify the application is alive
               Run on every commit
@sanity      — 20-30 scenarios verifying primary functionality per module
               Run after every deployment
@regression  — Full suite, all stable scenarios
               Run nightly or on release branches
@e2e         — End-to-end business flow scenarios
               Run on release candidate builds
```

### Scenario Type Tags

```
@positive    — Happy path, expected successful flows
@negative    — Error handling, invalid input, rejection flows
@security    — Authentication, authorization, injection tests
@performance — Scenarios measuring page load or response time
@accessibility — Accessibility and WCAG compliance checks
@ui          — Visual layout and UI component tests
@edge-case   — Boundary conditions and unusual inputs
```

### Application Module Tags

```
@authentication   — Login, logout, password reset
@registration     — New user signup
@products         — Product catalog and search
@cart             — Shopping cart operations
@checkout         — Payment and order placement
@orders           — Order history and management
@account          — Profile and settings management
@admin            — Admin panel functionality
```

### Lifecycle Management Tags

```
@wip         — Work in progress, not ready to run
@skip        — Temporarily disabled (include reason in description)
@known-failure — Known bug, tracked in Jira
@flaky       — Intermittently failing, under investigation
```

### Complete Example

```gherkin
@checkout @regression
Feature: Order Checkout

  @smoke @positive
  Scenario: Complete purchase with credit card
    ...

  @positive @sanity
  Scenario: Apply valid discount code
    ...

  @negative
  Scenario: Checkout fails with expired card
    ...

  @security
  Scenario: Checkout rejects tampered price parameter
    ...

  @wip
  Scenario: Checkout with cryptocurrency payment
    ...

  @known-failure
  Scenario: Guest checkout saves shipping address
    # Bug SHOP-4421 — guest address persistence broken
    ...
```

---

## Reading Tags Programmatically in Step Definitions

Inside a step definition, you can read the tags of the current scenario using the `Scenario` object (available in hooks) or by injecting scenario context.

```java
@After
public void conditionalCleanup(Scenario scenario) {
    Collection<String> tags = scenario.getSourceTagNames();

    if (tags.contains("@clean-db")) {
        DatabaseHelper.cleanUp();
    }

    if (tags.contains("@delete-test-user")) {
        UserHelper.deleteTestUser();
    }
}
```

---

## Tag Naming Conventions

**Use lowercase with hyphens for multi-word tags.** `@known-failure` not `@KnownFailure` or `@known_failure`.

**Be specific and consistent.** `@checkout-payment` is better than `@payment` if the application has multiple payment contexts.

**Document tag meanings.** Maintain a `TAGS.md` or wiki page that defines every tag, its purpose, and when to use it. This prevents tag proliferation and ensures team-wide consistency.

**Avoid redundant tags.** If a feature is tagged `@regression`, its scenarios do not also need `@regression` unless you specifically want to enable running some scenarios from that feature without running others.

---

## Summary

Tags are the navigation and filtering system of a Cucumber test suite. Applied consistently across features, scenarios, and examples blocks, they allow any subset of the suite to be executed in isolation: smoke tests for quick build validation, regression tests for full coverage, module-specific tags for focused testing, and lifecycle tags for managing work-in-progress scenarios. Combined with tag-based conditional hooks, they make the framework both flexible and efficient.

---

## Related Topics

- GherkinSyntax.md — Where and how to place tags in feature files
- Hooks.md — Conditional hooks based on tags
- ScenarioOutline.md — Tagging individual Examples blocks
- IntroBDD.md — Test runner configuration with @CucumberOptions
- ParallelExecution.md — Tag-based parallel test distribution
