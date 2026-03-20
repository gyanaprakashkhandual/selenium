# Background Keyword

## Overview

The `Background` keyword in Gherkin defines a set of steps that execute before every scenario in a feature file. It is the Cucumber equivalent of a setup method — a way to avoid repeating the same precondition steps at the start of every scenario.

`Background` runs automatically. It does not need to be called and cannot be skipped. Every scenario in the file, including every row of a Scenario Outline, is preceded by the Background steps.

---

## Syntax

```gherkin
Feature: [Feature title]

  Background:
    Given [shared precondition step 1]
    And   [shared precondition step 2]
    And   [shared precondition step 3]

  Scenario: First scenario
    When  [action]
    Then  [expected outcome]

  Scenario: Second scenario
    When  [action]
    Then  [expected outcome]
```

The `Background` block has no title (though Cucumber allows one for documentation purposes). It uses the same step keywords as scenarios: `Given`, `When`, `Then`, `And`, `But`, and `*`.

---

## Why Background Exists

Without `Background`, every scenario must repeat its setup steps:

```gherkin
# Without Background — repeated setup in every scenario
Feature: Product Search

  Scenario: Search by name
    Given the user is logged in as "buyer@example.com"
    And the user navigates to the products page
    When the user searches for "laptop"
    Then laptop results should be displayed

  Scenario: Filter by category
    Given the user is logged in as "buyer@example.com"
    And the user navigates to the products page
    When the user selects category "Electronics"
    Then only electronics should be displayed

  Scenario: Sort by price
    Given the user is logged in as "buyer@example.com"
    And the user navigates to the products page
    When the user sorts results by "Price: Low to High"
    Then the cheapest product should appear first
```

The first two `Given` steps are identical in every scenario. With `Background`:

```gherkin
# With Background — clean, no repetition
Feature: Product Search

  Background:
    Given the user is logged in as "buyer@example.com"
    And the user navigates to the products page

  Scenario: Search by name
    When the user searches for "laptop"
    Then laptop results should be displayed

  Scenario: Filter by category
    When the user selects category "Electronics"
    Then only electronics should be displayed

  Scenario: Sort by price
    When the user sorts results by "Price: Low to High"
    Then the cheapest product should appear first
```

Each scenario now starts with its `When` step. The reader immediately knows what the scenario is testing.

---

## Background with Scenario Outline

Background steps run before each generated row from a Scenario Outline, exactly as they run before regular scenarios.

```gherkin
Feature: Order Management

  Background:
    Given the user is logged in as "admin@example.com"
    And the user is on the orders management page

  Scenario Outline: Filter orders by status
    When the user filters orders by status "<status>"
    Then only orders with status "<status>" should be displayed
    And the result count should be greater than 0

    Examples:
      | status    |
      | Pending   |
      | Shipped   |
      | Delivered |
      | Cancelled |
```

The Background runs 4 times — once before each row. The admin is logged in and on the orders page before each filter test.

---

## Step Definition Implementation

Background steps use the same step definitions as scenario steps. There is no special annotation or class for Background — Cucumber simply executes the Background steps using the existing `@Given`, `@When`, `@Then`, `@And` annotations.

```java
package stepdefinitions;

import hooks.Hooks;
import io.cucumber.java.en.And;
import io.cucumber.java.en.Given;
import pages.LoginPage;
import pages.ProductsPage;

public class CommonSteps {

    private LoginPage loginPage;
    private ProductsPage productsPage;

    // Used in Background — shared across all product search scenarios
    @Given("the user is logged in as {string}")
    public void theUserIsLoggedInAs(String email) {
        loginPage = new LoginPage(Hooks.getDriver());
        Hooks.getDriver().get(System.getProperty("app.base.url") + "/login");
        loginPage.loginWithEmail(email, getPasswordForUser(email));
    }

    @And("the user navigates to the products page")
    public void theUserNavigatesToProductsPage() {
        Hooks.getDriver().get(System.getProperty("app.base.url") + "/products");
        productsPage = new ProductsPage(Hooks.getDriver());
    }

    private String getPasswordForUser(String email) {
        // Retrieve from config or test data
        return System.getProperty("test.default.password", "testpass123");
    }
}
```

---

## Background vs Hooks: When to Use Each

Both Background steps and `@Before` hooks run before each scenario. They serve different purposes and should not be treated as interchangeable.

| Aspect                | Background                            | @Before Hook                            |
| --------------------- | ------------------------------------- | --------------------------------------- |
| Language              | Gherkin (plain text)                  | Java code                               |
| Location              | Feature file                          | Hooks class                             |
| Visible in report     | Yes — steps appear in scenario report | No — invisible to non-technical readers |
| Business-readable     | Yes                                   | No                                      |
| Conditional execution | No — runs for every scenario in file  | Yes — can be scoped by tag              |
| Technical setup       | Not appropriate                       | Appropriate                             |
| Test data via DB/API  | Not appropriate                       | Appropriate                             |

**Use Background for** steps that have business meaning and should appear in the test report — logging in as a specific user, navigating to a specific page, selecting a specific product.

**Use @Before hooks for** technical infrastructure — starting the browser, setting cookies, resetting a database, calling an API to seed test data.

```gherkin
# Background — business-readable, shows in report
Background:
  Given the test database contains 10 active products
  And the user is logged in as "admin@example.com"
  And the user is on the product management page
```

```java
// @Before hook — technical setup, not visible in report
@Before
public void setUp() {
    WebDriverManager.chromedriver().setup();
    driver = new ChromeDriver();
    driver.manage().window().maximize();
}

@Before("@clean-db")
public void resetDatabase() {
    DatabaseHelper.truncateProductTable();
    DatabaseHelper.seedProducts(10);
}
```

---

## Background Scope

`Background` is scoped to one feature file. It does not span multiple files. If the same setup is needed across multiple features, use a shared step definition called from each Background, or use a conditional `@Before` hook.

```gherkin
# login.feature
Feature: Login Tests
  Background:
    Given the user is on the login page
  ...

# dashboard.feature
Feature: Dashboard Tests
  Background:
    Given the user is logged in as "admin"
    And the user is on the dashboard
  ...
```

Each file has its own Background with steps appropriate for that feature.

---

## Background with Data Tables

Background steps support all Gherkin constructs including Data Tables.

```gherkin
Feature: Product Catalogue Management

  Background:
    Given the following products exist in the catalogue:
      | name            | category    | price  | stock |
      | Wireless Mouse  | Electronics | 29.99  | 100   |
      | USB-C Hub       | Electronics | 49.99  | 50    |
      | Office Chair    | Furniture   | 299.00 | 20    |
    And the user is logged in as "catalogue_manager@example.com"

  Scenario: Display all products in Electronics category
    When the user filters by category "Electronics"
    Then 2 products should be displayed

  Scenario: Search for a specific product
    When the user searches for "USB-C Hub"
    Then the product "USB-C Hub" should be displayed
    And the price should be "49.99"
```

The Data Table in the Background step is passed to the step definition exactly as it would be in a scenario step. See DataTables.md for full implementation.

---

## Documenting the Background Step

A Background block can have an optional title on the `Background:` line. This title does not affect execution but improves readability in large feature files.

```gherkin
Feature: Checkout Process

  Background: Authenticated user with items in cart
    Given the user is logged in as "shopper@example.com"
    And the user has the following items in the cart:
      | product        | quantity |
      | Wireless Mouse | 2        |
      | Laptop Stand   | 1        |

  Scenario: Proceed to checkout
    When the user clicks "Proceed to Checkout"
    Then the checkout page should display 3 items total

  Scenario: Apply a discount code
    When the user applies coupon "SAVE10"
    Then a 10% discount should be applied
```

---

## Common Mistakes

**Putting scenario-specific steps in Background.** If a setup step is only needed by one or two scenarios, it belongs in those scenarios, not in Background.

```gherkin
# Wrong — "with admin privileges" is not needed by all scenarios
Background:
  Given the user is logged in with admin privileges
  And the user is on the dashboard

# Better — put the admin-specific step only in the scenarios that need it
Background:
  Given the user is on the dashboard

Scenario: Admin deletes a user
  Given the current user has admin privileges
  When the admin deletes user "john@example.com"
  Then the user should be removed
```

**Making Background steps too long.** A Background with 8 steps is a warning sign that the feature file covers too many concerns, or that scenarios are not independent enough.

**Confusing Background with @Before.** Background is for business-readable preconditions. Technical browser setup belongs in `@Before` hooks in Java.

**Expecting Background to run only once.** Background runs before every scenario. If you need a step to run only once for the entire file, use `@BeforeAll` in a hook class.

---

## Complete Feature File Using Background

```gherkin
@account @regression
Feature: User Account Profile Management

  Background:
    Given the user is logged in as "user@example.com"
    And the user is on the profile settings page

  @smoke @positive
  Scenario: Update display name
    When the user changes the display name to "Jane Smith"
    And the user saves the changes
    Then the success message "Profile updated successfully" should appear
    And the display name should show "Jane Smith"

  @positive
  Scenario: Update email address
    When the user changes the email to "new.email@example.com"
    And the user saves the changes
    Then a verification email should be sent to "new.email@example.com"

  @negative
  Scenario: Save profile with empty display name
    When the user clears the display name field
    And the user saves the changes
    Then the validation error "Display name is required" should appear

  @negative
  Scenario: Save profile with invalid email format
    When the user enters "notanemail" in the email field
    And the user saves the changes
    Then the validation error "Please enter a valid email address" should appear

  @positive
  Scenario: Upload a profile picture
    When the user uploads the file "profile-photo.jpg" as their avatar
    And the user saves the changes
    Then the profile picture should be updated
    And the thumbnail should reflect the new image
```

---

## Summary

The `Background` keyword eliminates repetition of shared precondition steps across all scenarios in a feature file. It improves feature file readability by moving the "given" state into a clearly separated block, letting each scenario start directly with its `When` action. Background is strictly scoped to its feature file, runs before every scenario including Scenario Outline rows, and is best used for business-readable preconditions rather than technical setup (which belongs in `@Before` hooks).

---

## Related Topics

- GherkinSyntax.md — Full keyword reference
- Hooks.md — @Before hooks for technical setup alongside Background
- FeatureFiles.md — Feature file organization and structure
- ScenarioOutline.md — Background with parameterized scenarios
