# Introduction to Page Object Model (POM)

## Overview

The Page Object Model is a design pattern widely used in test automation that creates an object-oriented layer between your test code and the web application under test. Each web page or significant component of the application is represented as a class, and the interactions with that page are encapsulated as methods within that class.

POM is not a framework by itself. It is a pattern that you apply on top of Selenium WebDriver to produce cleaner, more maintainable, and more scalable test automation code.

---

## Why Page Object Model

Without POM, test scripts directly embed locators and interaction logic. When the UI changes, every test that touches that element must be updated individually. This creates a maintenance burden that grows with the size of the test suite.

With POM, the locator and the interaction live in one place — the page class. When the UI changes, you update the page class, and all tests that use it automatically benefit from the fix.

The core benefits are:

- **Separation of concerns** — Test logic is kept separate from UI interaction logic.
- **Reusability** — Page methods can be called from multiple test classes.
- **Maintainability** — UI changes require updates in one place only.
- **Readability** — Tests read like plain English because method names describe actions, not Selenium calls.
- **Reduced duplication** — Common sequences like login or navigation are written once.

---

## Core Concepts

### Page Class

A page class represents one screen or component of the application. It holds:

- WebElement declarations (the locators)
- Methods that describe actions a user can perform on that page
- Methods that retrieve state or data from the page (for assertions)

### Test Class

A test class uses page classes to perform test steps. It knows nothing about locators or Selenium calls — it only calls methods on page objects.

### Separation of Responsibilities

```
Test Class          Page Class           WebDriver
-----------         ----------           ---------
calls login()  -->  finds elements  -->  interacts with browser
asserts title  -->  returns text    -->  reads DOM
```

---

## Basic Page Object Example

### Application Under Test: Login Page

The login page has a username field, a password field, and a submit button.

**Without POM (poor approach):**

```java
@Test
public void testSuccessfulLogin() {
    driver.findElement(By.id("username")).sendKeys("admin");
    driver.findElement(By.id("password")).sendKeys("secret");
    driver.findElement(By.id("loginBtn")).click();
    String title = driver.findElement(By.id("welcomeMsg")).getText();
    Assert.assertEquals("Welcome, admin", title);
}
```

This works for one test. But if the `username` field ID changes, you must hunt through every test file to fix it.

**With POM (correct approach):**

**LoginPage.java**

```java
package pages;

import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;

public class LoginPage {

    private WebDriver driver;

    // Locators
    private By usernameField = By.id("username");
    private By passwordField = By.id("password");
    private By loginButton   = By.id("loginBtn");

    // Constructor
    public LoginPage(WebDriver driver) {
        this.driver = driver;
    }

    // Actions
    public void enterUsername(String username) {
        driver.findElement(usernameField).sendKeys(username);
    }

    public void enterPassword(String password) {
        driver.findElement(passwordField).sendKeys(password);
    }

    public void clickLogin() {
        driver.findElement(loginButton).click();
    }

    public DashboardPage login(String username, String password) {
        enterUsername(username);
        enterPassword(password);
        clickLogin();
        return new DashboardPage(driver);
    }
}
```

**DashboardPage.java**

```java
package pages;

import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;

public class DashboardPage {

    private WebDriver driver;

    private By welcomeMessage = By.id("welcomeMsg");

    public DashboardPage(WebDriver driver) {
        this.driver = driver;
    }

    public String getWelcomeMessage() {
        return driver.findElement(welcomeMessage).getText();
    }
}
```

**LoginTest.java**

```java
package tests;

import org.testng.Assert;
import org.testng.annotations.Test;
import pages.LoginPage;
import pages.DashboardPage;

public class LoginTest extends BaseTest {

    @Test
    public void testSuccessfulLogin() {
        LoginPage loginPage = new LoginPage(driver);
        DashboardPage dashboard = loginPage.login("admin", "secret");
        Assert.assertEquals(dashboard.getWelcomeMessage(), "Welcome, admin");
    }
}
```

The test is now clean. It describes what is being tested, not how Selenium works.

---

## Page Object Return Types

A well-designed page method returns the page object that becomes active after the action. This enforces correct navigation flow in the test.

```java
// After clicking Login, the user lands on DashboardPage
public DashboardPage clickLoginButton() {
    driver.findElement(loginButton).click();
    return new DashboardPage(driver);
}

// After clicking Logout, the user lands back on LoginPage
public LoginPage clickLogout() {
    driver.findElement(logoutButton).click();
    return new LoginPage(driver);
}
```

If an action stays on the same page (such as an inline validation), return `this`:

```java
// Clicking an invalid form stays on LoginPage
public LoginPage clickLoginWithInvalidCredentials() {
    driver.findElement(loginButton).click();
    return this;
}

public String getErrorMessage() {
    return driver.findElement(errorMessage).getText();
}
```

---

## Organizing Page Objects

A standard project layout for POM:

```
src/
  main/
    java/
      pages/
        LoginPage.java
        DashboardPage.java
        ProfilePage.java
        components/
          NavigationBar.java
          Footer.java
  test/
    java/
      tests/
        LoginTest.java
        DashboardTest.java
      base/
        BaseTest.java
```

Keep page classes under `src/main/java/pages` and test classes under `src/test/java/tests`. This reflects the intent: page classes are reusable code, not tests.

---

## Common Mistakes to Avoid

### Putting assertions inside page objects

Page objects should not make assertions. They represent the page and expose its state. Tests make the assertions.

```java
// Wrong
public void verifyWelcomeMessage(String expected) {
    Assert.assertEquals(driver.findElement(welcomeMsg).getText(), expected); // do not do this
}

// Correct
public String getWelcomeMessage() {
    return driver.findElement(welcomeMsg).getText(); // return the value, let the test assert
}
```

### Duplicating locators across classes

Never define the same locator in two page classes. If two tests need the same element, they both go through the same page class method.

### Making page classes too large

If a page class has 50+ methods, the page is too complex or the class is doing too much. Consider splitting large pages into components (see PageComponents.md).

### Mixing test data into page classes

Page classes handle interactions with the UI. They do not hold usernames, URLs, or expected strings. Pass that data in from the test or a data provider.

---

## POM with Cucumber BDD

POM works naturally with Cucumber. Step definition classes call page object methods, keeping the step definitions clean.

```java
// Step definition class
public class LoginSteps {

    private LoginPage loginPage;
    private DashboardPage dashboardPage;

    @Given("the user is on the login page")
    public void userIsOnLoginPage() {
        loginPage = new LoginPage(Hooks.getDriver());
    }

    @When("the user logs in with username {string} and password {string}")
    public void userLogsIn(String username, String password) {
        dashboardPage = loginPage.login(username, password);
    }

    @Then("the welcome message should be {string}")
    public void welcomeMessageShouldBe(String expected) {
        Assert.assertEquals(dashboardPage.getWelcomeMessage(), expected);
    }
}
```

---

## Summary

The Page Object Model is the foundation of every professional Selenium framework. It separates test intent from implementation detail, reduces duplication, and makes the suite resilient to UI changes. Every topic that follows — PageFactory, BasePage, PageComponents, POM with Inheritance, and Fluent POM — builds on this foundation.

---

## Related Topics

- PageFactory.md — Using @FindBy annotations to declare elements
- BasePage.md — Sharing common logic across all page classes
- PageComponents.md — Modeling reusable UI components like headers and modals
- POMInheritance.md — Using inheritance to structure page hierarchies
- FluentPOM.md — Chaining method calls for more expressive tests
