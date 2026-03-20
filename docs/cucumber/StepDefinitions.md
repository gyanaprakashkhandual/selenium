# Step Definitions

## Overview

Step definitions are Java methods that Cucumber executes when it matches a Gherkin step. Each step in a feature file — every `Given`, `When`, `Then`, `And`, and `But` line — must have exactly one matching step definition. Cucumber uses the text of the step (after the keyword) to find the match, either through Cucumber Expressions or regular expressions.

Step definitions are the bridge between human-readable specifications and Selenium automation code.

---

## How Step Matching Works

When Cucumber processes a feature file, it reads each step and searches all step definition classes in the configured glue packages for a method whose annotation pattern matches the step text.

```gherkin
When the user enters username "admin" and password "secret"
```

Cucumber finds the step definition annotated with a matching expression:

```java
@When("the user enters username {string} and password {string}")
public void theUserEntersCredentials(String username, String password) {
    loginPage.enterUsername(username);
    loginPage.enterPassword(password);
}
```

The `{string}` placeholder captures the quoted text `"admin"` and `"secret"` and passes them as method parameters.

---

## Step Definition Annotations

All five step keywords have corresponding Java annotations. They are functionally identical — `@Given`, `@When`, `@Then`, `@And`, and `@But` all do the same thing. Using the correct annotation improves readability.

```java
import io.cucumber.java.en.Given;
import io.cucumber.java.en.When;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.And;
import io.cucumber.java.en.But;

@Given("the user is on the login page")
public void theUserIsOnTheLoginPage() { }

@When("the user clicks the login button")
public void theUserClicksLoginButton() { }

@Then("the dashboard should be displayed")
public void theDashboardShouldBeDisplayed() { }

@And("the welcome message should say {string}")
public void theWelcomeMessageShouldSay(String message) { }

@But("the admin panel should not be accessible")
public void theAdminPanelShouldNotBeAccessible() { }
```

---

## Cucumber Expressions

Cucumber Expressions are the preferred way to match steps and capture parameters. They are simpler and more readable than regular expressions.

### Built-in Parameter Types

| Expression       | Java Type       | Matches Example                |
| ---------------- | --------------- | ------------------------------ |
| `{string}`       | String          | `"admin"`, `"any quoted text"` |
| `{int}`          | int / Integer   | `42`, `100`                    |
| `{long}`         | long / Long     | `123456789`                    |
| `{float}`        | float / Float   | `3.14`                         |
| `{double}`       | double / Double | `9.99`                         |
| `{word}`         | String          | `admin` (unquoted single word) |
| `{bigdecimal}`   | BigDecimal      | `19.99`                        |
| `{}` (anonymous) | String          | matches anything               |

### Examples

```java
// Captures a quoted string
@When("the user searches for {string}")
public void theUserSearchesFor(String keyword) {
    searchPage.enterSearchTerm(keyword);
}

// Captures an integer
@Then("the cart should contain {int} items")
public void theCartShouldContainItems(int expectedCount) {
    Assert.assertEquals(cartPage.getItemCount(), expectedCount);
}

// Captures two strings
@When("the user logs in with username {string} and password {string}")
public void theUserLogsIn(String username, String password) {
    loginPage.login(username, password);
}

// Captures a double
@Then("the order total should be {double}")
public void theOrderTotalShouldBe(double expectedTotal) {
    Assert.assertEquals(checkoutPage.getOrderTotal(), expectedTotal, 0.01);
}

// Captures an unquoted word
@Given("the {word} user is on the dashboard")
public void theUserIsOnDashboard(String role) {
    Assert.assertEquals(dashboardPage.getUserRole(), role);
}
```

---

## Regular Expressions in Step Definitions

For complex patterns, use regular expressions. The regex is written inside `^` and `$` anchors.

```java
// Match optional text
@Given("^the user is on the (login|registration) page$")
public void theUserIsOnPage(String page) {
    if (page.equals("login")) {
        driver.get(config.getLoginUrl());
    } else {
        driver.get(config.getRegistrationUrl());
    }
}

// Match any text (captured as group)
@Then("^the error message should (contain|equal) \"(.+)\"$")
public void theErrorMessageShouldMatch(String matchType, String expectedMessage) {
    String actual = loginPage.getErrorMessage();
    if (matchType.equals("contain")) {
        Assert.assertTrue(actual.contains(expectedMessage));
    } else {
        Assert.assertEquals(actual, expectedMessage);
    }
}

// Optional word
@When("^the user (slowly )?clicks the submit button$")
public void theUserClicksSubmit(String modifier) {
    submitButton.click();
}
```

---

## Step Definition Class Structure

Each step definition class should cover one feature or one logical group of scenarios. Keep classes focused and small.

```java
package stepdefinitions;

import hooks.Hooks;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import org.openqa.selenium.WebDriver;
import org.testng.Assert;
import pages.DashboardPage;
import pages.LoginPage;

public class LoginSteps {

    private WebDriver driver;
    private LoginPage loginPage;
    private DashboardPage dashboardPage;

    // Retrieve the driver from the Hooks class (thread-safe)
    private WebDriver getDriver() {
        return Hooks.getDriver();
    }

    @Given("the user is on the login page")
    public void theUserIsOnTheLoginPage() {
        driver = getDriver();
        driver.get(System.getProperty("app.base.url") + "/login");
        loginPage = new LoginPage(driver);
    }

    @When("the user enters username {string} and password {string}")
    public void theUserEntersCredentials(String username, String password) {
        loginPage.enterUsername(username);
        loginPage.enterPassword(password);
    }

    @When("the user clicks the login button")
    public void theUserClicksLoginButton() {
        dashboardPage = loginPage.clickLogin();
    }

    @When("the user logs in with username {string} and password {string}")
    public void theUserLogsIn(String username, String password) {
        dashboardPage = loginPage.login(username, password);
    }

    @Then("the dashboard page should be displayed")
    public void theDashboardPageShouldBeDisplayed() {
        Assert.assertTrue(dashboardPage.isLoaded(), "Dashboard page did not load");
    }

    @Then("the welcome message should contain {string}")
    public void theWelcomeMessageShouldContain(String expectedText) {
        String actual = dashboardPage.getWelcomeMessage();
        Assert.assertTrue(actual.contains(expectedText),
            "Expected welcome message to contain '" + expectedText + "' but was: " + actual);
    }

    @Then("an error message should be displayed")
    public void anErrorMessageShouldBeDisplayed() {
        Assert.assertTrue(loginPage.isErrorDisplayed(), "Error message was not displayed");
    }

    @Then("the error message should be {string}")
    public void theErrorMessageShouldBe(String expectedMessage) {
        Assert.assertEquals(loginPage.getErrorMessage(), expectedMessage);
    }

    @Then("the user should remain on the login page")
    public void theUserShouldRemainOnLoginPage() {
        Assert.assertTrue(driver.getCurrentUrl().contains("/login"),
            "User navigated away from login page");
    }
}
```

---

## Sharing State Between Step Definitions

In Cucumber, each step definition class is instantiated fresh for each scenario by default. To share state between multiple step definition classes within the same scenario, use Cucumber's dependency injection (PicoContainer is the simplest option).

### Maven Dependency for PicoContainer

```xml
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-picocontainer</artifactId>
    <version>7.15.0</version>
    <scope>test</scope>
</dependency>
```

### Shared Context Class

```java
package context;

import org.openqa.selenium.WebDriver;
import pages.DashboardPage;
import pages.LoginPage;

public class ScenarioContext {

    private WebDriver driver;
    private LoginPage loginPage;
    private DashboardPage dashboardPage;

    // Getters and setters
    public WebDriver getDriver() { return driver; }
    public void setDriver(WebDriver driver) { this.driver = driver; }

    public LoginPage getLoginPage() { return loginPage; }
    public void setLoginPage(LoginPage loginPage) { this.loginPage = loginPage; }

    public DashboardPage getDashboardPage() { return dashboardPage; }
    public void setDashboardPage(DashboardPage dashboardPage) { this.dashboardPage = dashboardPage; }
}
```

### Step Definitions Using Shared Context

```java
package stepdefinitions;

import context.ScenarioContext;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import pages.LoginPage;

public class LoginSteps {

    private final ScenarioContext context;

    // PicoContainer injects ScenarioContext automatically
    public LoginSteps(ScenarioContext context) {
        this.context = context;
    }

    @Given("the user is on the login page")
    public void theUserIsOnTheLoginPage() {
        context.getDriver().get("https://example.com/login");
        context.setLoginPage(new LoginPage(context.getDriver()));
    }

    @When("the user logs in with username {string} and password {string}")
    public void theUserLogsIn(String username, String password) {
        context.setDashboardPage(
            context.getLoginPage().login(username, password)
        );
    }
}
```

```java
package stepdefinitions;

import context.ScenarioContext;
import io.cucumber.java.en.Then;
import org.testng.Assert;

public class DashboardSteps {

    private final ScenarioContext context;

    public DashboardSteps(ScenarioContext context) {
        this.context = context;
    }

    @Then("the dashboard should be displayed")
    public void theDashboardShouldBeDisplayed() {
        Assert.assertTrue(context.getDashboardPage().isLoaded());
    }

    @Then("the welcome message should say {string}")
    public void theWelcomeMessageShouldSay(String expected) {
        Assert.assertEquals(context.getDashboardPage().getWelcomeMessage(), expected);
    }
}
```

Both classes share the same `ScenarioContext` instance because PicoContainer creates one instance per scenario and injects it into every class that declares it as a constructor parameter.

---

## Step Definition Best Practices

### Keep Step Definitions Thin

Step definitions should be a thin delegation layer. Business logic belongs in page objects. Selenium interactions belong in page objects. Assertions belong in step definitions, not page objects.

```java
// Wrong — Selenium interaction inside step definition
@When("the user enters username {string}")
public void theUserEntersUsername(String username) {
    driver.findElement(By.id("username")).clear();
    driver.findElement(By.id("username")).sendKeys(username);  // wrong place
}

// Correct — delegate to page object
@When("the user enters username {string}")
public void theUserEntersUsername(String username) {
    loginPage.enterUsername(username);  // page object handles Selenium
}
```

### Reuse Steps Across Scenarios

Step definitions should be reusable. A step like "the user is logged in" should be written once and used by any scenario that needs a logged-in state. If you find yourself writing near-identical steps with minor wording differences, consolidate them.

```java
// Consolidate these two into one
@Given("the user is on the login page")
@Given("the login page is open")  // aliases — both map to the same method

// Alternatively, use one canonical form and stick to it in all feature files
@Given("the user is on the login page")
public void theUserIsOnTheLoginPage() {
    driver.get(baseUrl + "/login");
    loginPage = new LoginPage(driver);
}
```

### Use Meaningful Assertion Messages

When an assertion fails, the message should tell you exactly what went wrong without reading the stack trace.

```java
// Poor assertion message
Assert.assertTrue(dashboardPage.isLoaded());

// Good assertion message
Assert.assertTrue(dashboardPage.isLoaded(),
    "Dashboard page did not load after login. Current URL: " + driver.getCurrentUrl());
```

### Group Step Definitions by Feature

Do not put all step definitions in one giant class. One step definition class per feature (or per logical group) keeps the codebase organized and each file focused.

```
stepdefinitions/
  LoginSteps.java
  RegistrationSteps.java
  SearchSteps.java
  CartSteps.java
  CheckoutSteps.java
  CommonSteps.java   <-- steps shared across multiple features
```

### Avoid Duplicate Step Definitions

Cucumber throws an `AmbiguousStepDefinitionsException` if two step definitions match the same step text. Always check existing step definitions before writing a new one.

---

## Common Step Patterns

### Navigation Steps

```java
@Given("the user navigates to {string}")
public void theUserNavigatesTo(String path) {
    driver.get(baseUrl + path);
}

@Given("the user is on the {string} page")
public void theUserIsOnPage(String pageName) {
    driver.get(baseUrl + "/" + pageName.toLowerCase().replace(" ", "-"));
}
```

### Form Interaction Steps

```java
@When("the user enters {string} in the {string} field")
public void theUserEntersInField(String value, String fieldName) {
    currentPage.fillField(fieldName, value);
}

@When("the user clears the {string} field")
public void theUserClearsField(String fieldName) {
    currentPage.clearField(fieldName);
}

@When("the user selects {string} from the {string} dropdown")
public void theUserSelectsFromDropdown(String option, String dropdownName) {
    currentPage.selectFromDropdown(dropdownName, option);
}
```

### Assertion Steps

```java
@Then("the page title should be {string}")
public void thePageTitleShouldBe(String expectedTitle) {
    Assert.assertEquals(driver.getTitle(), expectedTitle,
        "Page title mismatch");
}

@Then("the {string} field should show the error {string}")
public void theFieldShouldShowError(String fieldName, String expectedError) {
    Assert.assertEquals(currentPage.getFieldError(fieldName), expectedError);
}

@Then("the {string} button should be disabled")
public void theButtonShouldBeDisabled(String buttonName) {
    Assert.assertFalse(currentPage.isButtonEnabled(buttonName),
        "Expected button '" + buttonName + "' to be disabled");
}
```

---

## Handling Undefined Steps

When Cucumber encounters a step that has no matching step definition, it marks the scenario as `Undefined` and suggests a snippet. Always check the console output for suggested snippets when adding new scenarios.

```
You can implement missing steps with the snippets below:

@When("the user clicks the forgot password link")
public void theUserClicksTheForgotPasswordLink() {
    // Write code here that turns the phrase above into concrete actions
    throw new io.cucumber.java.PendingException();
}
```

Replace the `throw new PendingException()` with actual implementation code.

---

## Dry Run Mode

Use dry run to validate all steps are defined without executing any browser automation:

```java
@CucumberOptions(
    features = "src/test/resources/features",
    glue = {"stepdefinitions", "hooks"},
    dryRun = true
)
public class DryRunRunner extends AbstractTestNGCucumberTests {
}
```

Dry run is useful during development to confirm that all Gherkin steps map to defined step definition methods before running the full suite.

---

## Summary

Step definitions are the implementation layer of Cucumber BDD. They match Gherkin steps using Cucumber Expressions or regular expressions, delegate to page objects for Selenium interactions, and contain the assertions that verify expected outcomes. Well-written step definitions are thin, reusable, and organized by feature. State shared between step definition classes is managed through dependency injection using PicoContainer.

---

## Related Topics

- GherkinSyntax.md — Writing the Gherkin steps that step definitions match
- Hooks.md — Setup and teardown using @Before and @After
- WorldObject.md — Advanced state sharing across step definitions
- CustomParameterTypes.md — Defining custom {parameter} types
- IntroBDD.md — Project setup and test runner configuration
