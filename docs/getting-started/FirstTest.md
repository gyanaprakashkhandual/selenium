# First Selenium Test

## Overview

Writing your first Selenium test is the milestone that confirms your environment is correctly set up and your understanding of the core WebDriver API is solid. This guide walks you through writing a complete, real test — not just `driver.get("https://google.com")`, but a proper structured test that opens a browser, interacts with elements, makes assertions, and cleans up after itself.

---

## What We Will Build

We will write a test that:

1. Opens a browser
2. Navigates to a login page (using [https://the-internet.herokuapp.com/login](https://the-internet.herokuapp.com/login) — a public test site)
3. Enters a valid username and password
4. Clicks the login button
5. Asserts that the login was successful
6. Logs out
7. Quits the browser

This is the simplest complete test that exercises the real automation flow.

---

## Test Site Reference

We use `https://the-internet.herokuapp.com` throughout this guide. It is a free, public test application built specifically for Selenium practice.

| Credential                   | Value                            |
| ---------------------------- | -------------------------------- |
| Username                     | `tomsmith`                       |
| Password                     | `SuperSecretPassword!`           |
| Expected message after login | `You logged into a secure area!` |

---

## The Simplest Possible Test (Raw WebDriver)

Before adding any framework patterns, here is the most basic version — everything in one method:

```java
package com.yourcompany.automation;

import io.github.bonigarcia.wdm.WebDriverManager;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;
import org.testng.Assert;
import org.testng.annotations.AfterMethod;
import org.testng.annotations.BeforeMethod;
import org.testng.annotations.Test;

import java.time.Duration;

public class FirstLoginTest {

    private WebDriver driver;
    private WebDriverWait wait;

    @BeforeMethod
    public void setUp() {
        // 1. Set up WebDriver using WebDriverManager
        WebDriverManager.chromedriver().setup();

        ChromeOptions options = new ChromeOptions();
        options.addArguments("--start-maximized");
        options.addArguments("--disable-notifications");

        // 2. Launch Chrome
        driver = new ChromeDriver(options);

        // 3. Set up explicit wait (10 seconds)
        wait = new WebDriverWait(driver, Duration.ofSeconds(10));

        // 4. Navigate to the application
        driver.get("https://the-internet.herokuapp.com/login");
    }

    @Test
    public void testSuccessfulLogin() {
        // ---- ARRANGE ----
        // Elements are found fresh at the time of interaction

        // ---- ACT ----
        // Enter username
        WebElement usernameField = wait.until(
            ExpectedConditions.visibilityOfElementLocated(By.id("username"))
        );
        usernameField.clear();
        usernameField.sendKeys("tomsmith");

        // Enter password
        WebElement passwordField = driver.findElement(By.id("password"));
        passwordField.clear();
        passwordField.sendKeys("SuperSecretPassword!");

        // Click the Login button
        driver.findElement(By.cssSelector("button[type='submit']")).click();

        // ---- ASSERT ----
        // Wait for the success message to appear
        WebElement successMessage = wait.until(
            ExpectedConditions.visibilityOfElementLocated(
                By.cssSelector(".flash.success")
            )
        );

        String actualMessage = successMessage.getText();
        Assert.assertTrue(
            actualMessage.contains("You logged into a secure area!"),
            "Login success message not found. Actual: " + actualMessage
        );

        // Assert URL changed to secure area
        Assert.assertTrue(
            driver.getCurrentUrl().contains("/secure"),
            "Expected URL to contain /secure, but got: " + driver.getCurrentUrl()
        );
    }

    @Test
    public void testFailedLogin() {
        // Enter wrong credentials
        driver.findElement(By.id("username")).sendKeys("invaliduser");
        driver.findElement(By.id("password")).sendKeys("wrongpassword");
        driver.findElement(By.cssSelector("button[type='submit']")).click();

        // Assert error message appears
        WebElement errorMessage = wait.until(
            ExpectedConditions.visibilityOfElementLocated(
                By.cssSelector(".flash.error")
            )
        );

        Assert.assertTrue(
            errorMessage.getText().contains("Your username is invalid!"),
            "Expected error message not shown"
        );
    }

    @AfterMethod
    public void tearDown() {
        // Always quit the driver to close the browser and end the session
        if (driver != null) {
            driver.quit();
        }
    }
}
```

---

## Running the Test

### From IntelliJ IDEA

1. Right-click the test class → **Run 'FirstLoginTest'**
2. Watch Chrome launch, navigate, interact, and close automatically.
3. View results in the **Run** panel at the bottom.

### From Maven (Command Line)

```bash
# Run all tests
mvn clean test

# Run a specific test class
mvn clean test -Dtest=FirstLoginTest

# Run with a specific browser
mvn clean test -Dbrowser=firefox

# Run in headless mode
mvn clean test -Dheadless=true
```

### Expected Output

```
-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running com.yourcompany.automation.FirstLoginTest
Tests run: 2, Failures: 0, Errors: 0, Skipped: 0

BUILD SUCCESS
```

---

## Understanding What Just Happened

Let's break down every important line:

### WebDriverManager Setup

```java
WebDriverManager.chromedriver().setup();
```

This single line detects your Chrome version, downloads the matching ChromeDriver, and sets the system property. Without this, you would need to manually download and configure ChromeDriver.

### Launching Chrome

```java
driver = new ChromeDriver(options);
```

This starts a new Chrome browser session. The `ChromeOptions` object configures browser behavior — maximizing the window and disabling notification popups.

### Navigating to a URL

```java
driver.get("https://the-internet.herokuapp.com/login");
```

`driver.get()` waits for the page to fully load (until the `document.readyState` is `complete`) before returning.

### Finding Elements

```java
WebElement usernameField = driver.findElement(By.id("username"));
```

`findElement()` searches the DOM for the first element matching the given locator. If no match is found, it throws `NoSuchElementException`.

### Explicit Wait

```java
WebElement usernameField = wait.until(
    ExpectedConditions.visibilityOfElementLocated(By.id("username"))
);
```

Instead of blindly calling `findElement()`, we wait for the element to be visible. This is the correct approach for dynamic pages. `WebDriverWait` + `ExpectedConditions` is covered in depth in the Waits and Synchronization topic.

### Typing Text

```java
usernameField.sendKeys("tomsmith");
```

`sendKeys()` simulates keyboard input. Always call `clear()` before `sendKeys()` to avoid appending to any pre-filled value.

### Clicking

```java
driver.findElement(By.cssSelector("button[type='submit']")).click();
```

Simulates a mouse click on the element.

### Assertions

```java
Assert.assertTrue(
    actualMessage.contains("You logged into a secure area!"),
    "Login success message not found. Actual: " + actualMessage
);
```

TestNG's `Assert` class verifies expected behavior. The second parameter is the failure message — always provide a descriptive one so you know what went wrong without reading the code.

### Quitting the Driver

```java
driver.quit();
```

`quit()` closes all browser windows and ends the WebDriver session. **Always call `quit()` in `@AfterMethod`** — even if the test fails — to prevent orphaned browser processes.

> **`driver.close()` vs `driver.quit()`**
>
> - `driver.close()` — closes the **current** browser window/tab only. The WebDriver session remains open.
> - `driver.quit()` — closes **all** windows and **kills** the WebDriver session and the driver process. Always use `quit()` in teardown.

---

## The AAA Pattern — Writing Readable Tests

Every test should follow the **Arrange-Act-Assert** pattern:

```java
@Test
public void testSuccessfulLogin() {

    // ARRANGE — set up preconditions
    String username = "tomsmith";
    String password = "SuperSecretPassword!";
    String expectedMessage = "You logged into a secure area!";

    // ACT — perform the action being tested
    driver.findElement(By.id("username")).sendKeys(username);
    driver.findElement(By.id("password")).sendKeys(password);
    driver.findElement(By.cssSelector("button[type='submit']")).click();

    // ASSERT — verify the expected outcome
    String actualMessage = wait.until(
        ExpectedConditions.visibilityOfElementLocated(By.cssSelector(".flash"))
    ).getText();

    Assert.assertTrue(actualMessage.contains(expectedMessage));
}
```

This structure makes tests:

- **Readable** — anyone can understand what the test does in 30 seconds
- **Maintainable** — each section has one responsibility
- **Debuggable** — when a test fails, you immediately know if it was setup, action, or assertion

---

## Common First-Test Problems and Fixes

| Problem                                    | Symptom                       | Fix                                                                                      |
| ------------------------------------------ | ----------------------------- | ---------------------------------------------------------------------------------------- |
| `NoSuchElementException`                   | Element not found             | Use explicit wait; verify the locator using DevTools                                     |
| `ElementNotInteractableException`          | Element found but can't click | Element may be hidden or behind another element; scroll into view or wait for visibility |
| `StaleElementReferenceException`           | Element reference expired     | Re-find the element after page change; use `@FindBy` with Page Factory                   |
| Test passes locally, fails in CI           | Timing issue                  | Remove all `Thread.sleep`; use proper explicit waits                                     |
| `WebDriverException: Chrome not reachable` | Browser crashed               | Check for conflicting ChromeDriver processes; update to latest Chrome and driver         |
| Test runs but browser doesn't close        | Missing `driver.quit()`       | Always put `driver.quit()` in `@AfterMethod` wrapped in null check                       |

---

## Improving the Test: Thread Safety

The test above is fine for a single-threaded run. For parallel execution, each thread must have its own WebDriver instance. This is done with `ThreadLocal`:

```java
public class DriverManager {

    private static final ThreadLocal<WebDriver> driverThreadLocal = new ThreadLocal<>();

    public static void setDriver(WebDriver driver) {
        driverThreadLocal.set(driver);
    }

    public static WebDriver getDriver() {
        return driverThreadLocal.get();
    }

    public static void quitDriver() {
        WebDriver driver = driverThreadLocal.get();
        if (driver != null) {
            driver.quit();
            driverThreadLocal.remove();
        }
    }
}
```

This is covered in depth in the **Driver Manager** section of the Framework Architecture chapter.

---

## What Comes Next

Now that your first test works, the natural progression is:

1. **Stop duplicating locators** — move them into Page Object classes
2. **Stop putting everything in one class** — separate concerns using POM
3. **Stop using `Thread.sleep`** — replace with explicit waits everywhere
4. **Stop hardcoding test data** — read from config files or data tables
5. **Add BDD** — write tests in Gherkin so they are readable by non-developers

Each of these improvements is covered in subsequent chapters.

---

## Summary

You have written and run a complete Selenium test that opens a browser, interacts with elements, makes meaningful assertions, and cleans up after itself. You understand the `@BeforeMethod`, `@Test`, `@AfterMethod` lifecycle; the core WebDriver methods; the importance of explicit waits; and the AAA test structure. This is the foundation everything else builds on.
