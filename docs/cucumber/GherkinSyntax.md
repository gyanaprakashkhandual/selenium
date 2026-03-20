# Gherkin Syntax

## Overview

Gherkin is the plain-text language used to write Cucumber feature files. It is designed to be readable by anyone — developers, testers, product managers, and business stakeholders — without requiring any technical knowledge. Gherkin uses indentation and specific keywords to give structure to specifications.

Gherkin files have the `.feature` extension and are stored in the `src/test/resources/features` directory.

---

## File Structure

A Gherkin file follows a strict hierarchy:

```
Feature
  Background (optional, once per Feature)
  Rule (optional grouping, Gherkin 6+)
    Scenario / Scenario Outline
      Given / When / Then / And / But steps
        Examples (for Scenario Outline only)
```

---

## Keywords

### Feature

Every feature file begins with the `Feature` keyword followed by a title. An optional description can follow on subsequent lines. The description is not parsed — it is purely informational.

```gherkin
Feature: User Authentication

  This feature covers all scenarios related to user login,
  logout, and session management for the web application.
```

The `Feature` keyword gives the file a name and groups all scenarios within it. A single feature file should cover one functional area.

### Scenario

`Scenario` marks the beginning of a single test case. It has a title and contains a sequence of steps. Each scenario is independent — it should not depend on the state left by another scenario.

```gherkin
Scenario: Successful login with valid credentials
  Given the user is on the login page
  When the user enters username "admin" and password "secret"
  Then the dashboard page should be displayed
```

`Scenario` can also be written as `Example` — both are identical keywords.

### Given

`Given` describes the precondition or initial state of the system before the user action. It sets up context for the test.

```gherkin
Given the user is on the login page
Given the user has an active account with email "user@example.com"
Given the shopping cart contains 3 items
```

From an automation perspective, `Given` steps typically navigate to a URL, set up test data, or put the system into a known state.

### When

`When` describes the action or event that the user performs. It is the trigger — the thing being tested.

```gherkin
When the user clicks the login button
When the user submits the registration form
When the user adds the product to the cart
```

A scenario should ideally have one `When` step. Multiple `When` steps suggest the scenario is testing too many things at once.

### Then

`Then` describes the expected outcome. It is the assertion — what should have happened as a result of the `When` action.

```gherkin
Then the dashboard page should be displayed
Then an error message "Invalid credentials" should appear
Then the cart item count should be 4
```

`Then` steps should be specific and verifiable. They must describe observable outcomes, not internal system state.

### And

`And` is a continuation keyword. It replaces `Given`, `When`, or `Then` when you need multiple steps of the same type, keeping the text natural.

```gherkin
Given the user is on the login page
And the user has a valid account

When the user enters valid credentials
And the user clicks the login button

Then the dashboard should be displayed
And the welcome message should say "Welcome, admin"
```

Without `And`, you would repeat `Given`, `When`, or `Then` repeatedly, which reads awkwardly.

### But

`But` is functionally identical to `And`. It is used when a negative condition reads more naturally with "but."

```gherkin
Then the user should be redirected to the dashboard
But the session should not have an admin token
```

### The Asterisk Step

The `*` keyword is a neutral step marker. It can be used in any position and reads as a bullet point. Some teams use it in Background or setup-heavy scenarios where Given/When/Then labeling feels forced.

```gherkin
Background:
  * the database is seeded with test data
  * the application is running on port 8080
```

---

## Scenario Outline

`Scenario Outline` (also written as `Scenario Template`) is used when the same scenario should run multiple times with different data values. The variable placeholders are written inside angle brackets `<variable>` and the data is supplied in an `Examples` table.

```gherkin
Scenario Outline: Login with multiple user roles
  Given the user is on the login page
  When the user logs in with username "<username>" and password "<password>"
  Then the page title should be "<expected_title>"

  Examples:
    | username   | password | expected_title    |
    | admin      | admin123 | Admin Dashboard   |
    | manager    | mgr456   | Manager Dashboard |
    | viewer     | view789  | Reports Page      |
```

Each row in the `Examples` table generates one scenario. The test runner treats each row as a separate test case.

---

## Background

`Background` defines steps that run before every scenario in the feature file. It avoids repeating the same `Given` steps in every scenario.

```gherkin
Feature: Product Search

  Background:
    Given the user is logged in
    And the user is on the product search page

  Scenario: Search by product name
    When the user searches for "laptop"
    Then results containing "laptop" should be displayed

  Scenario: Search with no results
    When the user searches for "xyznotexist"
    Then the message "No products found" should be displayed
```

The `Background` steps run before each scenario, including each row of a `Scenario Outline`.

---

## Rule

`Rule` is a Gherkin 6 keyword that groups related scenarios under a business rule. It adds a layer of organization between `Feature` and `Scenario`.

```gherkin
Feature: User Account Management

  Rule: A user cannot log in with incorrect credentials

    Scenario: Wrong password
      Given the user has an account with username "user1"
      When the user attempts to log in with password "wrongpass"
      Then the login should be rejected

    Scenario: Non-existent username
      When the user attempts to log in with username "nobody"
      Then the login should be rejected

  Rule: A locked account cannot log in

    Scenario: Locked account login attempt
      Given the user account "user2" is locked
      When the user attempts to log in as "user2"
      Then the login should be rejected with message "Account locked"
```

---

## Tags

Tags are labels attached to features or scenarios. They are used to filter which scenarios run in a given test execution. Tags begin with the `@` symbol.

```gherkin
@regression @authentication
Feature: User Login

  @smoke @positive
  Scenario: Successful login
    Given the user is on the login page
    When the user logs in with valid credentials
    Then the dashboard is displayed

  @negative
  Scenario: Login with wrong password
    Given the user is on the login page
    When the user logs in with an incorrect password
    Then an error message is displayed
```

Tags can be applied at the `Feature`, `Scenario`, or `Scenario Outline` level. A tag on a `Feature` applies to every scenario within it.

---

## Comments

Lines beginning with `#` are comments. Cucumber ignores them.

```gherkin
# This scenario verifies the core login path
# Jira ticket: AUTH-101
Scenario: Admin user login
  Given the admin user is on the login page
  When the admin logs in with valid credentials
  Then the admin dashboard is displayed
```

---

## Data Tables

Data tables pass structured data into a step. They are defined using pipe `|` characters.

```gherkin
Scenario: Register a new user
  Given the user fills in the registration form with:
    | field        | value              |
    | First Name   | Jane               |
    | Last Name    | Smith              |
    | Email        | jane@example.com   |
    | Password     | SecureP@ss1        |
  When the user submits the form
  Then the account should be created successfully
```

See DataTables.md for full implementation details.

---

## Doc Strings

Doc strings pass multi-line text content into a step. They are enclosed in triple quotes `"""`.

```gherkin
Scenario: Submit a support ticket with a long message
  Given the user is on the support page
  When the user submits a ticket with the following message:
    """
    I am unable to access my account after the recent password reset.
    I have tried the reset link three times but the email is not arriving.
    Please escalate this issue.
    """
  Then the ticket should be created with status "Open"
```

See DocStrings.md for full implementation details.

---

## Internationalization

Gherkin supports over 70 languages. The `# language:` directive at the top of a feature file sets the language.

```gherkin
# language: fr
Fonctionnalité: Connexion utilisateur

  Scénario: Connexion réussie
    Étant donné que l'utilisateur est sur la page de connexion
    Quand l'utilisateur saisit ses identifiants valides
    Alors le tableau de bord doit s'afficher
```

For international teams, this allows business stakeholders to write specifications in their native language.

---

## Complete Feature File Example

```gherkin
@authentication
Feature: User Login and Session Management

  As a registered user
  I want to be able to log in to the application
  So that I can access my account and perform actions

  Background:
    Given the application is running
    And the user is on the login page

  @smoke @positive
  Scenario: Successful login with valid credentials
    When the user enters username "admin" and password "admin123"
    And the user clicks the login button
    Then the user should be redirected to the dashboard
    And the welcome message should display "Welcome, admin"

  @negative
  Scenario: Login fails with incorrect password
    When the user enters username "admin" and password "wrongpass"
    And the user clicks the login button
    Then the error message "Invalid username or password." should be displayed
    And the user should remain on the login page

  @negative
  Scenario: Login fails with empty credentials
    When the user clicks the login button without entering credentials
    Then the validation message "Username is required." should appear

  @security
  Scenario: Session expires after inactivity
    Given the user is logged in as "admin"
    When the session is idle for 30 minutes
    Then the user should be redirected to the login page
    And the message "Your session has expired." should be shown

  @regression
  Scenario Outline: Login with different roles
    When the user enters username "<username>" and password "<password>"
    And the user clicks the login button
    Then the page title should be "<page_title>"

    Examples:
      | username  | password  | page_title          |
      | admin     | admin123  | Admin Dashboard     |
      | manager   | mgr456    | Manager Panel       |
      | auditor   | aud789    | Audit Reports       |
```

---

## Gherkin Best Practices

**Write in the third person, present tense.** Use "the user clicks" not "I click" or "clicking." This reads more naturally as a specification than as a test instruction.

**Keep scenarios short.** A scenario should have 3 to 7 steps. More than that usually means the scenario is doing too much or needs a Background.

**Make each scenario independent.** Never share state between scenarios. Each scenario must be able to run in isolation in any order.

**Use specific, concrete values in Then steps.** "The price should be $29.99" is testable. "The price should be correct" is not.

**Avoid conjunctions in step text.** "When the user enters a username and clicks the submit button" is two actions in one step. Split them.

**Use Background for repeated Given steps.** If the first three Given steps are identical in every scenario, move them to Background.

**Do not expose technical detail.** No CSS selectors, element IDs, database queries, or API calls should appear in Gherkin steps.

---

## Summary

Gherkin provides the vocabulary and structure for writing human-readable test specifications. Its keywords — `Feature`, `Scenario`, `Given`, `When`, `Then`, `And`, `But`, `Background`, `Scenario Outline`, `Examples`, `Rule`, and `Tags` — give precise meaning to each part of a specification. Mastering Gherkin syntax is the prerequisite for writing effective BDD tests with Cucumber.

---

## Related Topics

- IntroBDD.md — What BDD is and why it matters
- FeatureFiles.md — Organizing and writing feature files at scale
- ScenarioOutline.md — Data-driven testing with Scenario Outline and Examples
- Background.md — The Background keyword in detail
- TagsFiltering.md — Filtering test execution with tags
