# Feature Files

## Overview

Feature files are the heart of a Cucumber test suite. They are plain-text files with a `.feature` extension that contain Gherkin specifications describing the behavior of the application. Feature files serve simultaneously as test cases, acceptance criteria, and living documentation.

This document covers how to write, organize, and maintain feature files in a professional Selenium Cucumber framework.

---

## File Naming and Location

Feature files are stored in `src/test/resources/features`. This path is passed to the `features` attribute of `@CucumberOptions` in the test runner.

### Naming Conventions

- Use lowercase with hyphens: `user-login.feature`, `product-search.feature`
- Name the file after the feature it describes, not the test: `checkout.feature` not `checkout-tests.feature`
- One feature per file
- Group related features in sub-directories

### Directory Structure

For small projects, a flat structure works:

```
src/test/resources/features/
  login.feature
  registration.feature
  product-search.feature
  checkout.feature
```

For larger projects, group features by application module:

```
src/test/resources/features/
  authentication/
    login.feature
    logout.feature
    password-reset.feature
    session-management.feature
  products/
    product-search.feature
    product-detail.feature
    product-filters.feature
  orders/
    place-order.feature
    order-history.feature
    order-cancellation.feature
  account/
    profile-update.feature
    address-management.feature
    payment-methods.feature
```

---

## Feature File Structure

A complete feature file has the following structure:

```gherkin
@module-tag @additional-tag
Feature: [Short, descriptive title of the feature]

  [Optional multi-line description explaining the feature context,
   the user story, or business rules. This is ignored by Cucumber
   but read by humans.]

  Background:
    [Steps that run before every scenario in this file]

  @scenario-tag
  Scenario: [Scenario title]
    Given [precondition]
    When  [action]
    Then  [expected outcome]

  @scenario-tag
  Scenario Outline: [Parameterized scenario title]
    Given [precondition with <variable>]
    When  [action with <variable>]
    Then  [outcome with <expected>]

    Examples:
      | variable | expected |
      | value1   | result1  |
      | value2   | result2  |
```

---

## Writing the Feature Header

The feature header gives context. Use the User Story format when it helps stakeholders understand why the feature exists.

```gherkin
Feature: User Login

  As a registered user
  I want to be able to log in with my credentials
  So that I can access my personal account and preferences
```

The "As a / I want / So that" format is optional but encouraged for features that represent end-user value.

---

## Writing Effective Scenarios

### Positive Scenarios (Happy Path)

Always write the happy path first. This is the scenario that exercises the main, expected flow of the feature.

```gherkin
@smoke @positive
Scenario: Successful login with valid credentials
  Given the user is on the login page
  When the user enters a valid username and password
  Then the user should be redirected to the dashboard
  And the session cookie should be set
```

### Negative Scenarios (Sad Path)

Negative scenarios verify that the application handles errors, invalid inputs, and edge cases gracefully.

```gherkin
@negative
Scenario: Login rejected with incorrect password
  Given the user is on the login page
  When the user enters a valid username with an incorrect password
  Then the login form should display the error "Invalid credentials"
  And the user should remain on the login page
  And the password field should be cleared

@negative
Scenario: Login rejected when username does not exist
  Given the user is on the login page
  When the user enters an unregistered username
  Then the login form should display the error "Invalid credentials"

@negative
Scenario: Login rejected when fields are empty
  Given the user is on the login page
  When the user submits the login form without entering any credentials
  Then the validation message "Username is required" should appear
```

### Boundary and Edge Case Scenarios

```gherkin
@edge-case
Scenario: Login with username at maximum length
  Given the user is on the login page
  When the user enters a username of exactly 50 characters
  And the user enters a valid password
  Then the login should succeed

@security
Scenario: Login attempt with SQL injection in username field
  Given the user is on the login page
  When the user enters "admin' OR '1'='1" in the username field
  And the user enters any password
  Then the login should be rejected
  And no SQL error should be displayed
```

---

## Using Background Effectively

`Background` should contain only the steps that every scenario in the file genuinely needs. Do not put optional or scenario-specific setup in Background.

**Correct use of Background:**

```gherkin
Feature: Product Search

  Background:
    Given the user is logged in as "standard_user"
    And the user is on the products page

  Scenario: Search by exact product name
    When the user searches for "Wireless Mouse"
    Then the result "Wireless Mouse - Model X200" should appear in the list

  Scenario: Search returns no results
    When the user searches for "zzznoproduct"
    Then the message "No products found" should be displayed

  Scenario: Search by partial name
    When the user searches for "mouse"
    Then at least one result containing "mouse" should be displayed
```

**Incorrect use of Background (too specific):**

```gherkin
Background:
  Given the user is logged in
  And the user has 3 items in the cart          # <-- only relevant to one scenario
  And the first item is a laptop                 # <-- only relevant to one scenario
  And the coupon code "SAVE10" is applied        # <-- only relevant to one scenario
```

If setup steps are specific to one or two scenarios, keep them in those scenarios instead.

---

## Scenario Granularity

### Scenarios Should Be Atomic

Each scenario tests one behavior. It should have one `When` step describing a single user action.

```gherkin
# Wrong — tests two separate behaviors in one scenario
Scenario: Login and logout
  Given the user is on the login page
  When the user logs in with valid credentials
  Then the dashboard is displayed
  When the user clicks logout
  Then the user is on the login page

# Correct — two separate scenarios
Scenario: Successful login displays dashboard
  Given the user is on the login page
  When the user logs in with valid credentials
  Then the dashboard is displayed

Scenario: Logout redirects to login page
  Given the user is logged in
  When the user clicks logout
  Then the user is on the login page
```

### Scenarios Should Be Independent

Never write a scenario that depends on the state left by a previous scenario. Every scenario must be able to run in isolation and in any order.

```gherkin
# Wrong — scenario 2 depends on scenario 1 having run
Scenario: Add item to cart
  Given the user is on the product page for "Laptop"
  When the user clicks "Add to Cart"
  Then the cart count should be 1

Scenario: Checkout with item in cart   # <-- will fail if run alone
  When the user proceeds to checkout
  Then the checkout page should show 1 item
```

```gherkin
# Correct — each scenario sets its own state
Scenario: Add item to cart
  Given the user is on the product page for "Laptop"
  When the user clicks "Add to Cart"
  Then the cart count should be 1

Scenario: Checkout with item in cart
  Given the user has added "Laptop" to the cart    # <-- self-contained setup
  When the user proceeds to checkout
  Then the checkout page should show 1 item
```

---

## Scenario Tagging Strategy

Tags serve as the selection mechanism for test execution. A professional tagging strategy typically includes:

| Tag            | Purpose                                 |
| -------------- | --------------------------------------- |
| `@smoke`       | Critical path tests, run on every build |
| `@regression`  | Full regression suite                   |
| `@positive`    | Happy path scenarios                    |
| `@negative`    | Error and failure scenarios             |
| `@security`    | Security-related tests                  |
| `@ui`          | Pure front-end UI tests                 |
| `@api`         | Scenarios involving API calls           |
| `@wip`         | Work in progress, excluded from CI      |
| `@skip`        | Temporarily disabled scenarios          |
| `@module-name` | Tests for a specific application module |

Apply tags at both the `Feature` and `Scenario` level:

```gherkin
@authentication @regression
Feature: Login

  @smoke @positive
  Scenario: Successful login
    ...

  @negative
  Scenario: Invalid credentials
    ...

  @wip
  Scenario: SSO login
    ...
```

---

## Writing Steps at the Right Abstraction Level

Steps should describe user intent in business language, not implementation detail.

```gherkin
# Wrong — too technical
When the user clicks the element with id "btn-submit"
And the HTTP POST request is sent to "/api/v1/auth/login"

# Correct — business language
When the user submits the login form
```

```gherkin
# Wrong — too vague
When the user does the login thing
Then it works

# Correct — specific and verifiable
When the user logs in with username "admin" and password "admin123"
Then the admin dashboard page should be displayed
```

---

## Complete Real-World Feature File Example

```gherkin
@checkout @regression
Feature: Shopping Cart Checkout

  As a registered shopper
  I want to complete a purchase from my cart
  So that my order is placed and I receive a confirmation

  Background:
    Given the user is logged in as "buyer@example.com"
    And the user has the following items in the cart:
      | product         | quantity | price  |
      | Wireless Mouse  | 1        | 29.99  |
      | USB-C Hub       | 2        | 49.99  |

  @smoke @positive
  Scenario: Complete checkout with credit card
    Given the user is on the cart page
    When the user proceeds to checkout
    And the user enters shipping address for "123 Main St, New York, NY 10001"
    And the user pays with credit card "4111111111111111" expiry "12/26" CVV "123"
    And the user places the order
    Then the order confirmation page should be displayed
    And the order confirmation number should be generated
    And the cart should be empty

  @positive
  Scenario: Apply a valid discount code
    Given the user is on the cart page
    When the user applies the coupon code "SAVE15"
    Then a 15% discount should be applied to the order total
    And the new total should be displayed

  @negative
  Scenario: Apply an expired discount code
    Given the user is on the cart page
    When the user applies the coupon code "EXPIRED20"
    Then the message "This coupon code has expired" should be displayed
    And the order total should remain unchanged

  @negative
  Scenario: Checkout fails with declined credit card
    Given the user is on the checkout payment page
    When the user pays with declined card "4000000000000002"
    Then the error "Your card was declined. Please try a different payment method." should appear
    And the order should not be placed

  @regression
  Scenario Outline: Checkout with various payment methods
    Given the user is on the checkout payment page
    When the user selects "<payment_method>" as the payment option
    And the user completes the payment details
    And the user places the order
    Then the order should be confirmed with payment type "<payment_method>"

    Examples:
      | payment_method  |
      | Credit Card     |
      | Debit Card      |
      | PayPal          |
      | Bank Transfer   |
```

---

## Maintaining Feature Files Over Time

**Review feature files as part of code review.** Feature files describe requirements. When a feature changes, the feature file changes. Treat it with the same review discipline as production code.

**Delete scenarios that no longer apply.** Outdated scenarios that always pass but test removed functionality give false confidence.

**Keep the feature file in sync with the application.** A passing scenario that no longer tests what it claims to test is worse than a failing scenario, because it silently misrepresents coverage.

**Use step definition libraries for common steps.** Steps like "the user is logged in" should map to a single, shared step definition. Avoid creating slight variations of the same step in multiple feature files.

---

## Summary

Feature files are the specification layer of a Cucumber BDD framework. Effective feature files use meaningful titles, appropriate Background steps, atomic independent scenarios, consistent tagging, and business-level language. They serve as the authoritative record of what the application is supposed to do, readable by every member of the team.

---

## Related Topics

- GherkinSyntax.md — Full keyword reference for Gherkin
- StepDefinitions.md — Implementing the Java code behind each step
- Background.md — Background keyword in depth
- ScenarioOutline.md — Parameterized scenarios with Examples tables
- TagsFiltering.md — Filtering test runs by tag
