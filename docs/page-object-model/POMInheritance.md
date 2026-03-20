# POM with Inheritance

## Overview

Inheritance in the Page Object Model is the practice of using Java class hierarchies to share structure and behavior across page classes. The most foundational example is the `BasePage` class, which every page class extends. But inheritance can go deeper — application sections often share layout, navigation, and state that makes a mid-level parent class valuable.

This document covers how to design a class hierarchy for POM, when to use it, and how to combine it with composition (components) effectively.

---

## The Basic Inheritance Hierarchy

The minimal POM hierarchy has two levels:

```
BasePage (abstract)
  |-- LoginPage
  |-- DashboardPage
  |-- ProfilePage
  |-- OrdersPage
```

Every page class extends `BasePage`, inheriting WebDriver utilities, wait methods, and Page Factory initialization.

This level of inheritance is mandatory in any serious POM framework. It is covered in detail in BasePage.md.

---

## Section-Level Inheritance

Many applications have authenticated sections with a persistent navigation bar, header, or sidebar. All pages within that section share these elements. A section-level base class models this structure.

```
BasePage (abstract)
  |-- AuthenticatedPage (abstract) — pages requiring login
  |     |-- DashboardPage
  |     |-- ProfilePage
  |     |-- OrdersPage
  |     |-- ProductListPage
  |-- PublicPage (abstract) — pages without login
        |-- LoginPage
        |-- RegisterPage
        |-- ForgotPasswordPage
```

### AuthenticatedPage Implementation

```java
package pages;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;
import pages.components.NavigationBar;

public abstract class AuthenticatedPage extends BasePage {

    // Every authenticated page has these elements
    @FindBy(css = "header .user-display-name")
    private WebElement loggedInUsername;

    @FindBy(css = "header .notification-badge")
    private WebElement notificationBadge;

    // Every authenticated page has the navigation bar
    private NavigationBar navigationBar;

    public AuthenticatedPage(WebDriver driver) {
        super(driver);
        this.navigationBar = new NavigationBar(driver);
    }

    // Common methods available on all authenticated pages
    public NavigationBar getNavigationBar() {
        return navigationBar;
    }

    public String getLoggedInUsername() {
        return getText(loggedInUsername);
    }

    public int getNotificationCount() {
        String text = getText(notificationBadge);
        return text.isEmpty() ? 0 : Integer.parseInt(text);
    }

    public boolean isUserLoggedIn() {
        return isDisplayed(loggedInUsername);
    }

    public LoginPage logout() {
        return navigationBar.logout();
    }
}
```

### PublicPage Implementation

```java
package pages;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;

public abstract class PublicPage extends BasePage {

    @FindBy(css = "header .site-logo")
    private WebElement siteLogo;

    @FindBy(linkText = "Sign In")
    private WebElement signInLink;

    @FindBy(linkText = "Register")
    private WebElement registerLink;

    public PublicPage(WebDriver driver) {
        super(driver);
    }

    public LoginPage goToLogin() {
        click(signInLink);
        return new LoginPage(driver);
    }

    public RegisterPage goToRegister() {
        click(registerLink);
        return new RegisterPage(driver);
    }

    public boolean isLogoVisible() {
        return isDisplayed(siteLogo);
    }
}
```

### Concrete Page Classes

The concrete page classes extend the appropriate mid-level base class and focus only on page-specific content.

```java
package pages;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;

import java.util.List;

public class DashboardPage extends AuthenticatedPage {

    @FindBy(id = "welcomeMsg")
    private WebElement welcomeMessage;

    @FindBy(css = ".recent-orders tbody tr")
    private List<WebElement> recentOrderRows;

    @FindBy(css = ".quick-stats .revenue")
    private WebElement revenueWidget;

    public DashboardPage(WebDriver driver) {
        super(driver); // AuthenticatedPage -> BasePage
    }

    public String getWelcomeMessage() {
        return getText(welcomeMessage);
    }

    public int getRecentOrderCount() {
        return recentOrderRows.size();
    }

    public String getRevenue() {
        return getText(revenueWidget);
    }
}
```

```java
package pages;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;

public class LoginPage extends PublicPage {

    @FindBy(id = "username")
    private WebElement usernameField;

    @FindBy(id = "password")
    private WebElement passwordField;

    @FindBy(id = "loginBtn")
    private WebElement loginButton;

    @FindBy(css = ".alert-danger")
    private WebElement errorAlert;

    public LoginPage(WebDriver driver) {
        super(driver); // PublicPage -> BasePage
    }

    public DashboardPage login(String username, String password) {
        type(usernameField, username);
        type(passwordField, password);
        click(loginButton);
        return new DashboardPage(driver);
    }

    public String getErrorMessage() {
        return getText(errorAlert);
    }

    public boolean isErrorVisible() {
        return isDisplayed(errorAlert);
    }
}
```

---

## Test Class Hierarchy

The test classes follow a parallel hierarchy that mirrors the page hierarchy.

```
BaseTest (abstract)
  |-- AuthenticatedTest (abstract) — tests that start logged in
  |     |-- DashboardTest
  |     |-- ProfileTest
  |     |-- OrdersTest
  |-- PublicTest (abstract) — tests that run on public pages
        |-- LoginTest
        |-- RegisterTest
```

### BaseTest

```java
package tests;

import io.github.bonigarcia.wdm.WebDriverManager;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.testng.annotations.AfterMethod;
import org.testng.annotations.BeforeMethod;

public abstract class BaseTest {

    protected WebDriver driver;

    @BeforeMethod
    public void setUp() {
        WebDriverManager.chromedriver().setup();
        driver = new ChromeDriver();
        driver.manage().window().maximize();
        driver.get(System.getProperty("app.url", "https://example.com"));
    }

    @AfterMethod
    public void tearDown() {
        if (driver != null) {
            driver.quit();
        }
    }
}
```

### AuthenticatedTest

```java
package tests;

import org.testng.annotations.BeforeMethod;
import pages.DashboardPage;
import pages.LoginPage;

public abstract class AuthenticatedTest extends BaseTest {

    protected DashboardPage dashboardPage;

    @BeforeMethod(dependsOnMethods = "setUp")
    public void login() {
        LoginPage loginPage = new LoginPage(driver);
        dashboardPage = loginPage.login(
            System.getProperty("test.username", "admin"),
            System.getProperty("test.password", "secret")
        );
    }
}
```

### Concrete Test Class

```java
package tests;

import org.testng.Assert;
import org.testng.annotations.Test;

public class DashboardTest extends AuthenticatedTest {

    @Test
    public void testWelcomeMessageContainsUsername() {
        Assert.assertTrue(dashboardPage.getWelcomeMessage().contains("admin"));
    }

    @Test
    public void testNotificationCountIsVisible() {
        Assert.assertTrue(dashboardPage.isUserLoggedIn()); // from AuthenticatedPage
        int count = dashboardPage.getNotificationCount();  // from AuthenticatedPage
        Assert.assertTrue(count >= 0);
    }

    @Test
    public void testLogoutFromDashboard() {
        LoginPage loginPage = dashboardPage.logout();     // from AuthenticatedPage
        Assert.assertTrue(loginPage.isLogoVisible());
    }
}
```

The test class is completely clean. It knows nothing about locators, waits, or driver management. It only calls methods.

---

## Inheritance vs. Composition: Choosing the Right Tool

Both inheritance and composition are valid POM techniques. Use them for different purposes.

**Use inheritance when:**

- There is a genuine "is-a" relationship. A `DashboardPage` is an `AuthenticatedPage`.
- Multiple classes share state (fields) that should not be duplicated.
- A group of pages shares a URL pattern, a title structure, or a common set of actions.

**Use composition (components) when:**

- There is a "has-a" relationship. A `DashboardPage` has a `NavigationBar`.
- The shared UI piece appears on pages that are not related in hierarchy.
- The shared element is a discrete widget (table, modal, picker) rather than a page section.

**Combining both:**

```java
// DashboardPage IS an AuthenticatedPage (inheritance)
// DashboardPage HAS a RecentActivityTable (composition)

public class DashboardPage extends AuthenticatedPage {

    private DataTable recentActivityTable;

    public DashboardPage(WebDriver driver) {
        super(driver);
        this.recentActivityTable = new DataTable(driver, By.id("recent-activity"));
    }

    public DataTable getRecentActivity() {
        return recentActivityTable;
    }
}
```

---

## Abstract Methods in Mid-Level Classes

Use abstract methods in intermediate base classes to enforce contracts. Every authenticated page must be able to verify it has loaded correctly.

```java
public abstract class AuthenticatedPage extends BasePage {

    public AuthenticatedPage(WebDriver driver) {
        super(driver);
        verifyPageLoaded(); // called after init, enforces that every subclass verifies itself
    }

    // Every subclass must implement this
    protected abstract void verifyPageLoaded();

    // ... shared methods
}
```

```java
public class DashboardPage extends AuthenticatedPage {

    @FindBy(id = "dashboardContent")
    private WebElement dashboardContent;

    public DashboardPage(WebDriver driver) {
        super(driver); // triggers verifyPageLoaded()
    }

    @Override
    protected void verifyPageLoaded() {
        waitForVisibility(dashboardContent);
        // Optionally also verify the URL
        wait.until(ExpectedConditions.urlContains("/dashboard"));
    }
}
```

If a navigation step fails and the wrong page is returned, `verifyPageLoaded()` throws an exception immediately at the point of failure rather than producing a confusing error when a test later tries to use a wrong-page element.

---

## Full Hierarchy Diagram

```
BasePage (abstract)
    - driver : WebDriver
    - wait : WebDriverWait
    + click(WebElement)
    + type(WebElement, String)
    + getText(WebElement)
    + waitForVisibility(By)
    + scrollToElement(WebElement)

AuthenticatedPage (abstract) extends BasePage
    - navigationBar : NavigationBar
    - loggedInUsername : WebElement
    + getNavigationBar() : NavigationBar
    + getLoggedInUsername() : String
    + logout() : LoginPage
    # verifyPageLoaded() [abstract]

PublicPage (abstract) extends BasePage
    - siteLogo : WebElement
    + goToLogin() : LoginPage
    + goToRegister() : RegisterPage

DashboardPage extends AuthenticatedPage
    - welcomeMessage : WebElement
    + getWelcomeMessage() : String
    # verifyPageLoaded()

LoginPage extends PublicPage
    - usernameField : WebElement
    - passwordField : WebElement
    + login(String, String) : DashboardPage
    + getErrorMessage() : String
```

---

## Summary

Inheritance in POM creates a structured hierarchy that eliminates duplication at every level. The `BasePage` removes low-level Selenium boilerplate. Mid-level classes like `AuthenticatedPage` and `PublicPage` remove section-level duplication. Concrete page classes are left with only what is unique to that screen. Combined with composition for reusable components, this produces a framework that is easy to read, easy to extend, and easy to maintain.

---

## Related Topics

- BasePage.md — The root of the page hierarchy
- PageComponents.md — Composition for reusable UI pieces
- IntroPOM.md — Core POM concepts
- FluentPOM.md — Method chaining pattern for expressive tests
