# Introduction to BDD and Cucumber

## Overview

Behavior-Driven Development (BDD) is a software development methodology that encourages collaboration between developers, testers, and non-technical business stakeholders. BDD describes application behavior using plain-language specifications that everyone on the team can read and contribute to. These specifications are written before code is implemented, serving both as documentation and as the basis for automated tests.

Cucumber is the most widely used BDD framework for Java. It reads plain-English specifications written in a language called Gherkin and executes them as automated acceptance tests.

---

## What is Behavior-Driven Development

BDD evolved from Test-Driven Development (TDD). While TDD focuses on unit-level testing driven by developers, BDD focuses on the behavior of the entire system from the user's perspective, driven by conversations between all team members.

The core premise of BDD is that automated tests should describe what the system does, not how it does it. A BDD test says "when a user submits valid login credentials, the system should display the dashboard." A traditional test says "find element by ID, send keys, click button, assert text equals."

The BDD cycle follows three phases:

- **Discovery** — Business stakeholders, developers, and testers collaborate to define expected behaviors using concrete examples.
- **Formulation** — The examples are written as structured specifications in Gherkin (feature files).
- **Automation** — Developers and testers write code (step definitions) that maps each Gherkin step to automated test actions.

---

## Why Use BDD with Selenium

Selenium provides the mechanism to automate browser interactions. BDD provides the structure to express what those interactions should verify. Together they produce:

**Living documentation.** Feature files describe what the application does in business language. They are always accurate because they are executable — if the test passes, the documentation is correct.

**Shared understanding.** When a product manager writes acceptance criteria in Gherkin and a developer implements the step definitions, both parties are working from the same definition of done. Misunderstandings are discovered earlier.

**Test coverage aligned with business value.** BDD tests start from user stories and business rules, not from technical implementation. Every test has a clear business reason for existing.

**Readable failure reports.** When a Cucumber test fails, the failure output shows exactly which step of which scenario failed, in language that non-developers can understand.

---

## Cucumber Architecture

Cucumber operates by connecting three components:

```
Feature Files (.feature)         Step Definitions (.java)        Application Under Test
----------------------           ----------------------          ----------------------
Given the user is on             @Given("the user is on          Selenium opens
the login page           ---->   the login page")          --->  browser, navigates
                                 {driver.get(loginUrl)}          to login URL

When the user enters             @When("the user enters          Selenium finds
valid credentials        ---->   valid credentials")       --->  fields, sends keys
                                 {loginPage.login(...)}

Then the dashboard               @Then("the dashboard            Selenium asserts
should be displayed      ---->   should be displayed")     --->  page title or
                                 {Assert.assertTrue(...)}        welcome element
```

**Feature files** contain the Gherkin specifications. They are plain text files with a `.feature` extension. They describe scenarios using the Given-When-Then structure.

**Step definitions** are Java methods annotated with `@Given`, `@When`, `@Then`, `@And`, and `@But`. Cucumber matches each Gherkin step to a step definition using text patterns or regular expressions.

**Glue code** connects Cucumber to the test framework (TestNG or JUnit), the WebDriver management layer, and any hooks for setup and teardown.

---

## Cucumber with Maven

### POM Dependencies

```xml
<properties>
    <cucumber.version>7.15.0</cucumber.version>
    <selenium.version>4.18.1</selenium.version>
    <testng.version>7.9.0</testng.version>
</properties>

<dependencies>

    <!-- Cucumber core -->
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-java</artifactId>
        <version>${cucumber.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Cucumber + TestNG integration -->
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-testng</artifactId>
        <version>${cucumber.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Cucumber + JUnit 4 integration (alternative to TestNG) -->
    <dependency>
        <groupId>io.cucumber</groupId>
        <artifactId>cucumber-junit</artifactId>
        <version>${cucumber.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Selenium WebDriver -->
    <dependency>
        <groupId>org.seleniumhq.selenium</groupId>
        <artifactId>selenium-java</artifactId>
        <version>${selenium.version}</version>
    </dependency>

    <!-- WebDriver Manager -->
    <dependency>
        <groupId>io.github.bonigarcia</groupId>
        <artifactId>webdrivermanager</artifactId>
        <version>5.7.0</version>
    </dependency>

    <!-- Assertions -->
    <dependency>
        <groupId>org.testng</groupId>
        <artifactId>testng</artifactId>
        <version>${testng.version}</version>
        <scope>test</scope>
    </dependency>

</dependencies>
```

---

## Project Structure

A well-organized Cucumber + Selenium project follows this layout:

```
src/
  main/
    java/
      pages/
        BasePage.java
        LoginPage.java
        DashboardPage.java
      utils/
        DriverManager.java
        ConfigReader.java

  test/
    java/
      runners/
        TestRunner.java
      stepdefinitions/
        LoginSteps.java
        DashboardSteps.java
      hooks/
        Hooks.java

    resources/
      features/
        login/
          Login.feature
        dashboard/
          Dashboard.feature
      config/
        config.properties
```

Page classes live in `src/main/java/pages` because they are reusable production-quality code. Step definitions, runners, and hooks live in `src/test/java` because they are test infrastructure.

Feature files live in `src/test/resources/features`. Placing them in `resources` rather than `java` ensures they are correctly treated as non-compiled resource files.

---

## Test Runner

Cucumber needs a runner class to discover and execute feature files.

### TestNG Runner

```java
package runners;

import io.cucumber.testng.AbstractTestNGCucumberTests;
import io.cucumber.testng.CucumberOptions;

@CucumberOptions(
    features = "src/test/resources/features",
    glue = {"stepdefinitions", "hooks"},
    tags = "@regression",
    plugin = {
        "pretty",
        "html:target/cucumber-reports/report.html",
        "json:target/cucumber-reports/report.json"
    },
    monochrome = true
)
public class TestRunner extends AbstractTestNGCucumberTests {
}
```

### JUnit 4 Runner

```java
package runners;

import io.cucumber.junit.Cucumber;
import io.cucumber.junit.CucumberOptions;
import org.junit.runner.RunWith;

@RunWith(Cucumber.class)
@CucumberOptions(
    features = "src/test/resources/features",
    glue = {"stepdefinitions", "hooks"},
    plugin = {
        "pretty",
        "html:target/cucumber-reports/report.html",
        "json:target/cucumber-reports/report.json"
    }
)
public class TestRunner {
}
```

### CucumberOptions Explained

| Option       | Purpose                                                     |
| ------------ | ----------------------------------------------------------- |
| `features`   | Path to the directory containing `.feature` files           |
| `glue`       | Package(s) containing step definitions and hooks            |
| `tags`       | Filter scenarios by tag expression                          |
| `plugin`     | Output formatters for reporting                             |
| `monochrome` | Removes special characters from console output              |
| `dryRun`     | Validates step definitions without executing tests          |
| `strict`     | Fails the run if any step is undefined (default true in v7) |

---

## A Complete Hello World Example

### Feature File: src/test/resources/features/login/Login.feature

```gherkin
Feature: User Login

  Background:
    Given the user is on the login page

  Scenario: Successful login with valid credentials
    When the user enters username "admin" and password "secret"
    Then the dashboard should be displayed
    And the welcome message should contain "Welcome"

  Scenario: Failed login with invalid credentials
    When the user enters username "wrong" and password "wrong"
    Then an error message should be displayed
    And the error should say "Invalid username or password."
```

### Step Definitions: stepdefinitions/LoginSteps.java

```java
package stepdefinitions;

import hooks.Hooks;
import io.cucumber.java.en.And;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import org.testng.Assert;
import pages.DashboardPage;
import pages.LoginPage;

public class LoginSteps {

    private LoginPage loginPage;
    private DashboardPage dashboardPage;

    @Given("the user is on the login page")
    public void theUserIsOnTheLoginPage() {
        loginPage = new LoginPage(Hooks.getDriver());
        Hooks.getDriver().get("https://example.com/login");
    }

    @When("the user enters username {string} and password {string}")
    public void theUserEntersCredentials(String username, String password) {
        dashboardPage = loginPage
            .enterUsername(username)
            .enterPassword(password)
            .clickLogin();
    }

    @Then("the dashboard should be displayed")
    public void theDashboardShouldBeDisplayed() {
        Assert.assertTrue(dashboardPage.isLoaded());
    }

    @And("the welcome message should contain {string}")
    public void theWelcomeMessageShouldContain(String expected) {
        Assert.assertTrue(dashboardPage.getWelcomeMessage().contains(expected));
    }

    @Then("an error message should be displayed")
    public void anErrorMessageShouldBeDisplayed() {
        Assert.assertTrue(loginPage.isErrorDisplayed());
    }

    @And("the error should say {string}")
    public void theErrorShouldSay(String expected) {
        Assert.assertEquals(loginPage.getErrorMessage(), expected);
    }
}
```

### Hooks: hooks/Hooks.java

```java
package hooks;

import io.cucumber.java.After;
import io.cucumber.java.Before;
import io.github.bonigarcia.wdm.WebDriverManager;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;

public class Hooks {

    private static ThreadLocal<WebDriver> driverThread = new ThreadLocal<>();

    @Before
    public void setUp() {
        WebDriverManager.chromedriver().setup();
        WebDriver driver = new ChromeDriver();
        driver.manage().window().maximize();
        driverThread.set(driver);
    }

    @After
    public void tearDown() {
        if (getDriver() != null) {
            getDriver().quit();
            driverThread.remove();
        }
    }

    public static WebDriver getDriver() {
        return driverThread.get();
    }
}
```

---

## BDD Anti-Patterns to Avoid

**Writing steps in technical language.** Steps describe user behavior, not Selenium calls. "I click the element with ID loginBtn" is wrong. "I click the login button" is correct.

**One step definition per scenario.** Step definitions should be reusable. If every scenario has its own unique step that cannot be used anywhere else, you are missing the reuse benefit of BDD.

**Putting assertions in When steps.** `Given` sets up state. `When` describes the action. `Then` describes the expected outcome. Assertions belong in `Then` steps.

**Scenarios that are too long.** A scenario that has 15+ steps is hard to read and hard to debug. Break it into smaller scenarios or extract repetitive setup into a `Background`.

**Exposing implementation details in feature files.** Feature files should read like plain English to a non-developer. Any Gherkin step that mentions a database table, an API endpoint, or a CSS selector is too technical.

---

## Summary

BDD with Cucumber bridges the communication gap between business and technical teams by expressing test scenarios in a shared language. Cucumber connects plain-English Gherkin specifications to Selenium WebDriver automation through step definitions and hooks. The result is a test suite that serves as living documentation, provides regression coverage, and is readable by everyone on the team.

---

## Related Topics

- GherkinSyntax.md — Keywords, rules, and structure of the Gherkin language
- FeatureFiles.md — Organizing and writing feature files effectively
- StepDefinitions.md — Implementing step definitions in Java
- Hooks.md — Before and After hooks for setup and teardown
