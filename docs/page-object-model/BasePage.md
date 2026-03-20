# Base Page Class

## Overview

A Base Page class is an abstract parent class that all page objects extend. It centralizes common WebDriver interactions, wait utilities, and shared methods that every page in the application needs. Instead of repeating the same helper methods across dozens of page classes, you write them once in the base class and inherit them everywhere.

The Base Page class is one of the most important structural decisions in a Selenium framework. It defines the building blocks that every page object uses.

---

## Why a Base Page Class

Consider a typical page object that needs to:

- Wait for elements to be visible before interacting
- Click elements after waiting for them to be clickable
- Send keys after clearing a field
- Scroll to elements before clicking
- Take screenshots on failure

Without a base class, every page object must either duplicate these utilities or import separate helper classes. The base class makes these capabilities available to every page object through simple inheritance with no additional imports.

---

## Basic Base Page Implementation

```java
package pages;

import org.openqa.selenium.By;
import org.openqa.selenium.JavascriptExecutor;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.PageFactory;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

import java.time.Duration;
import java.util.List;

public abstract class BasePage {

    protected WebDriver driver;
    protected WebDriverWait wait;
    private static final int DEFAULT_TIMEOUT = 10;

    public BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(DEFAULT_TIMEOUT));
        PageFactory.initElements(driver, this);
    }

    // -----------------------------------------------------------------------
    // Navigation
    // -----------------------------------------------------------------------

    protected void navigateTo(String url) {
        driver.get(url);
    }

    protected String getCurrentUrl() {
        return driver.getCurrentUrl();
    }

    protected String getPageTitle() {
        return driver.getTitle();
    }

    // -----------------------------------------------------------------------
    // Element interaction
    // -----------------------------------------------------------------------

    protected void click(By locator) {
        waitForElementToBeClickable(locator).click();
    }

    protected void click(WebElement element) {
        waitForElementToBeClickable(element).click();
    }

    protected void type(By locator, String text) {
        WebElement element = waitForVisibility(locator);
        element.clear();
        element.sendKeys(text);
    }

    protected void type(WebElement element, String text) {
        waitForVisibility(element);
        element.clear();
        element.sendKeys(text);
    }

    protected String getText(By locator) {
        return waitForVisibility(locator).getText();
    }

    protected String getText(WebElement element) {
        waitForVisibility(element);
        return element.getText();
    }

    protected String getAttributeValue(By locator, String attribute) {
        return waitForVisibility(locator).getAttribute(attribute);
    }

    protected boolean isDisplayed(By locator) {
        try {
            return driver.findElement(locator).isDisplayed();
        } catch (Exception e) {
            return false;
        }
    }

    protected boolean isEnabled(WebElement element) {
        return element.isEnabled();
    }

    protected boolean isSelected(WebElement element) {
        return element.isSelected();
    }

    // -----------------------------------------------------------------------
    // Wait utilities
    // -----------------------------------------------------------------------

    protected WebElement waitForVisibility(By locator) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
    }

    protected WebElement waitForVisibility(WebElement element) {
        return wait.until(ExpectedConditions.visibilityOf(element));
    }

    protected WebElement waitForClickability(By locator) {
        return wait.until(ExpectedConditions.elementToBeClickable(locator));
    }

    protected WebElement waitForElementToBeClickable(By locator) {
        return wait.until(ExpectedConditions.elementToBeClickable(locator));
    }

    protected WebElement waitForElementToBeClickable(WebElement element) {
        return wait.until(ExpectedConditions.elementToBeClickable(element));
    }

    protected void waitForInvisibility(By locator) {
        wait.until(ExpectedConditions.invisibilityOfElementLocated(locator));
    }

    protected List<WebElement> waitForListToBeVisible(By locator) {
        return wait.until(ExpectedConditions.visibilityOfAllElementsLocatedBy(locator));
    }

    protected boolean waitForTextInElement(By locator, String text) {
        return wait.until(ExpectedConditions.textToBePresentInElementLocated(locator, text));
    }

    protected void waitForPageTitleContains(String title) {
        wait.until(ExpectedConditions.titleContains(title));
    }

    // -----------------------------------------------------------------------
    // JavaScript utilities
    // -----------------------------------------------------------------------

    protected void scrollToElement(WebElement element) {
        JavascriptExecutor js = (JavascriptExecutor) driver;
        js.executeScript("arguments[0].scrollIntoView(true);", element);
    }

    protected void clickWithJS(WebElement element) {
        JavascriptExecutor js = (JavascriptExecutor) driver;
        js.executeScript("arguments[0].click();", element);
    }

    protected void highlightElement(WebElement element) {
        JavascriptExecutor js = (JavascriptExecutor) driver;
        js.executeScript("arguments[0].style.border='3px solid red'", element);
    }

    // -----------------------------------------------------------------------
    // Custom wait with configurable timeout
    // -----------------------------------------------------------------------

    protected WebElement waitForVisibility(By locator, int timeoutSeconds) {
        WebDriverWait customWait = new WebDriverWait(driver, Duration.ofSeconds(timeoutSeconds));
        return customWait.until(ExpectedConditions.visibilityOfElementLocated(locator));
    }
}
```

---

## Page Classes Extending BasePage

Once BasePage is in place, all page classes extend it and immediately gain access to every utility method.

```java
package pages;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;

public class LoginPage extends BasePage {

    @FindBy(id = "username")
    private WebElement usernameField;

    @FindBy(id = "password")
    private WebElement passwordField;

    @FindBy(id = "loginBtn")
    private WebElement loginButton;

    @FindBy(css = ".alert-danger")
    private WebElement errorAlert;

    public LoginPage(WebDriver driver) {
        super(driver); // BasePage constructor calls PageFactory.initElements
    }

    public DashboardPage login(String username, String password) {
        type(usernameField, username);     // BasePage method
        type(passwordField, password);     // BasePage method
        click(loginButton);                // BasePage method
        return new DashboardPage(driver);
    }

    public String getErrorMessage() {
        return getText(errorAlert);        // BasePage method
    }

    public boolean isErrorDisplayed() {
        return isDisplayed(errorAlert);    // BasePage method — but needs By locator version
    }
}
```

```java
package pages;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;

public class DashboardPage extends BasePage {

    @FindBy(id = "welcomeMsg")
    private WebElement welcomeMessage;

    @FindBy(css = "nav a.logout")
    private WebElement logoutLink;

    @FindBy(id = "pageLoader")
    private WebElement loader;

    public DashboardPage(WebDriver driver) {
        super(driver);
    }

    public String getWelcomeMessage() {
        return getText(welcomeMessage);
    }

    public void waitForDashboardToLoad() {
        waitForInvisibility(loader);       // Wait for spinner to disappear
        waitForVisibility(welcomeMessage); // Then wait for content
    }

    public LoginPage logout() {
        click(logoutLink);
        return new LoginPage(driver);
    }
}
```

---

## Adding Page Verification to BasePage

A useful pattern is to verify that the correct page is loaded when a page object is created. This catches navigation errors early.

```java
public abstract class BasePage {

    protected WebDriver driver;
    protected WebDriverWait wait;

    public BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
        PageFactory.initElements(driver, this);
    }

    // Each subclass declares which URL or title it expects
    protected abstract void verifyPage();
}
```

```java
public class LoginPage extends BasePage {

    public LoginPage(WebDriver driver) {
        super(driver);
        verifyPage();
    }

    @Override
    protected void verifyPage() {
        wait.until(ExpectedConditions.urlContains("/login"));
    }
}
```

If a test accidentally lands on the wrong page and constructs the wrong page object, `verifyPage()` will throw a `TimeoutException` immediately with a clear failure point rather than a confusing element-not-found error later.

---

## Thread-Safe BasePage for Parallel Execution

When tests run in parallel (with TestNG parallel mode or Selenium Grid), each thread must have its own WebDriver instance. Do not use a static WebDriver in BasePage.

```java
public abstract class BasePage {

    protected WebDriver driver;
    protected WebDriverWait wait;

    // Each BasePage instance holds its own driver reference
    public BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
        PageFactory.initElements(driver, this);
    }
}
```

The `driver` is passed in from a `BaseTest` class that manages `ThreadLocal<WebDriver>`. See DriverManager.md for details.

---

## Extended BasePage with Logging

In a professional framework, BasePage logs every interaction. This makes debugging easier when a test fails.

```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public abstract class BasePage {

    protected WebDriver driver;
    protected WebDriverWait wait;
    protected static final Logger log = LogManager.getLogger(BasePage.class);

    public BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
        PageFactory.initElements(driver, this);
        log.info("Initialized page: {}", this.getClass().getSimpleName());
    }

    protected void click(WebElement element) {
        log.info("Clicking element: {}", element);
        waitForElementToBeClickable(element).click();
    }

    protected void type(WebElement element, String text) {
        log.info("Typing '{}' into element: {}", text, element);
        waitForVisibility(element);
        element.clear();
        element.sendKeys(text);
    }
}
```

---

## BasePage with Screenshot on Failure

```java
import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;

import java.io.File;
import java.io.IOException;
import org.apache.commons.io.FileUtils;

public abstract class BasePage {

    // ... driver, wait, constructor

    public void takeScreenshot(String screenshotName) {
        TakesScreenshot ts = (TakesScreenshot) driver;
        File source = ts.getScreenshotAs(OutputType.FILE);
        File destination = new File("reports/screenshots/" + screenshotName + ".png");
        try {
            FileUtils.copyFile(source, destination);
            log.info("Screenshot saved: {}", destination.getAbsolutePath());
        } catch (IOException e) {
            log.error("Failed to save screenshot: {}", e.getMessage());
        }
    }
}
```

---

## BasePage Design Guidelines

**Keep BasePage focused on WebDriver utilities.** Do not add business logic, test data, or application-specific assertions. BasePage is an infrastructure class.

**Use protected access for utility methods.** Page subclasses call them, but test classes should not. Marking them `protected` enforces this.

**Do not make BasePage concrete.** Declare it `abstract` so it cannot be instantiated directly. It only exists as a parent for page classes.

**Pass WebDriver in through the constructor.** Never use a singleton or static driver field in BasePage. Singleton drivers cause failures in parallel test execution.

**Initialize Page Factory in BasePage, not in subclasses.** Call `PageFactory.initElements(driver, this)` once in the BasePage constructor. Subclasses should only call `super(driver)`.

---

## Complete File Structure

```
src/main/java/
  pages/
    BasePage.java          <-- abstract, all utilities
    LoginPage.java         <-- extends BasePage
    DashboardPage.java     <-- extends BasePage
    ProfilePage.java       <-- extends BasePage
    components/
      NavigationBar.java   <-- extends BasePage
      Modal.java           <-- extends BasePage
```

---

## Summary

The Base Page class is the single most effective way to eliminate duplication in a Selenium POM framework. By centralizing WebDriver interactions, wait strategies, JavaScript utilities, and logging in one abstract class, you ensure that every page object in your suite uses consistent, tested, and maintainable code. Every page class you write should extend BasePage.

---

## Related Topics

- IntroPOM.md — Core POM concepts
- PageFactory.md — @FindBy annotation and element initialization
- PageComponents.md — Applying BasePage to reusable UI components
- DriverManager.md — Thread-safe WebDriver management for parallel tests
