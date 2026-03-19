# Waits and Synchronization

## Overview

Synchronization is the single most important skill for writing stable, non-flaky Selenium tests. The root cause of 90% of flaky tests is improper synchronization — tests that move faster than the application can respond. This document covers every wait mechanism in Selenium with deep explanation of when and how to use each one.

---

## Why Waits Are Necessary

Modern web applications are dynamic. After a button click or page navigation:

- API calls are made in the background
- The DOM is updated asynchronously
- Animations play
- Elements appear, disappear, or change state

Selenium can execute commands far faster than a browser can render. Without waits, you get:

```
NoSuchElementException         ← Element hasn't appeared yet
ElementNotInteractableException ← Element is present but not visible/enabled yet
StaleElementReferenceException ← Element existed but DOM was replaced
ElementClickInterceptedException ← Element is behind a loading spinner
```

The fix is always the same: **wait for the right condition before acting.**

---

## The Three Types of Waits

| Type              | How It Works                                             | When to Use                          |
| ----------------- | -------------------------------------------------------- | ------------------------------------ |
| **Implicit Wait** | Global wait applied to every `findElement` call          | ❌ Do NOT use — causes conflicts     |
| **Explicit Wait** | Wait for a specific condition on a specific element      | ✅ Always use this                   |
| **Fluent Wait**   | Explicit wait with custom polling and exception ignoring | ✅ Use for complex custom conditions |

---

## ❌ Implicit Wait — Understand to Avoid

```java
// Implicit wait — applied globally to ALL findElement calls
driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10));
```

Implicit wait tells WebDriver to wait up to N seconds when trying to find an element before throwing `NoSuchElementException`. Sounds useful — it is not.

### Why Implicit Wait is Harmful

**Problem 1: It combines badly with explicit waits**

If you set implicit wait to 10s and an explicit wait to 15s, and an element never appears, Selenium waits 10s × every polling attempt × 15s worth of attempts = potentially 2.5 minutes of waiting.

**Problem 2: It slows down negative tests**

When you intentionally check that an element does NOT exist, implicit wait forces Selenium to wait the full timeout every time before confirming the element is absent.

**Problem 3: It hides real synchronization problems**

Tests appear stable but are actually relying on a time buffer, not a real application state.

**The correct approach:**

```java
// Set implicit wait to ZERO and never use it
driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(0));
// Use only explicit waits everywhere
```

---

## ✅ Explicit Wait — The Right Way

`WebDriverWait` waits for a specific condition to become true, polling every 500ms by default, up to the timeout.

### Setup

```java
import org.openqa.selenium.support.ui.WebDriverWait;
import org.openqa.selenium.support.ui.ExpectedConditions;
import java.time.Duration;

// Create a wait instance
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// Use it before any element interaction
WebElement loginBtn = wait.until(
    ExpectedConditions.elementToBeClickable(By.id("login-btn"))
);
loginBtn.click();
```

---

## ExpectedConditions — Complete Reference

The `ExpectedConditions` class provides pre-built conditions for every common scenario:

### Presence and Visibility

```java
// Element is in the DOM (may be hidden, not necessarily visible)
WebElement el = wait.until(
    ExpectedConditions.presenceOfElementLocated(By.id("form"))
);

// Element is in DOM AND visible (has non-zero size, is not hidden)
WebElement el = wait.until(
    ExpectedConditions.visibilityOfElementLocated(By.id("success-msg"))
);

// Wait for an element you already have a reference to, to become visible
WebElement el = driver.findElement(By.id("panel"));
wait.until(ExpectedConditions.visibilityOf(el));

// Wait for ALL elements in a list to be visible
List<WebElement> items = wait.until(
    ExpectedConditions.visibilityOfAllElementsLocatedBy(By.cssSelector(".menu-item"))
);

// Wait for at least one element to be present
List<WebElement> results = wait.until(
    ExpectedConditions.presenceOfAllElementsLocatedBy(By.cssSelector(".search-result"))
);

// Wait for element to be INVISIBLE (disappear)
wait.until(ExpectedConditions.invisibilityOfElementLocated(By.id("loading-spinner")));
wait.until(ExpectedConditions.invisibilityOf(spinner)); // If you have the WebElement reference
```

### Clickability and Interactability

```java
// Element is visible AND enabled — safe to click
WebElement btn = wait.until(
    ExpectedConditions.elementToBeClickable(By.id("submit"))
);
btn.click();

// Works with a WebElement reference too
WebElement submitBtn = driver.findElement(By.id("submit"));
wait.until(ExpectedConditions.elementToBeClickable(submitBtn));
submitBtn.click();
```

### Selection State

```java
// Wait for checkbox to be selected
wait.until(ExpectedConditions.elementToBeSelected(By.id("agree-checkbox")));

// Wait for element to be in a specific selection state
wait.until(ExpectedConditions.elementSelectionStateToBe(
    By.id("agree-checkbox"), true  // true = selected, false = not selected
));
```

### Text Content

```java
// Wait for element text to be exactly this value
wait.until(ExpectedConditions.textToBe(By.id("status"), "Order Confirmed"));

// Wait for element text to contain a substring
wait.until(ExpectedConditions.textToBePresentInElementLocated(
    By.id("flash-message"), "successfully"
));

// Wait for element value attribute to contain text (for inputs)
wait.until(ExpectedConditions.textToBePresentInElementValue(
    By.id("auto-fill-input"), "John"
));
```

### URL and Title

```java
// Wait for URL to be exactly this
wait.until(ExpectedConditions.urlToBe("https://app.example.com/dashboard"));

// Wait for URL to contain a fragment
wait.until(ExpectedConditions.urlContains("/dashboard"));

// Wait for URL to match a regex
wait.until(ExpectedConditions.urlMatches(".*\\/orders\\/\\d+"));

// Wait for page title
wait.until(ExpectedConditions.titleIs("Dashboard | MyApp"));
wait.until(ExpectedConditions.titleContains("Dashboard"));
```

### Alert

```java
// Wait for a JS alert/confirm/prompt to appear
org.openqa.selenium.Alert alert = wait.until(
    ExpectedConditions.alertIsPresent()
);
System.out.println("Alert text: " + alert.getText());
alert.accept(); // Click OK
// alert.dismiss(); // Click Cancel
```

### Staleness

```java
// Wait for an old stale element to be detached from DOM
// Useful when navigating and re-finding elements
WebElement oldElement = driver.findElement(By.id("content"));
// ... trigger navigation...
wait.until(ExpectedConditions.stalenessOf(oldElement));
// Now re-find the element in the new DOM
WebElement newElement = driver.findElement(By.id("content"));
```

### Frame

```java
// Wait for a frame and switch into it
wait.until(ExpectedConditions.frameToBeAvailableAndSwitchToIt(By.id("myFrame")));
```

### Number of Windows

```java
// Wait for a new tab/window to open
wait.until(ExpectedConditions.numberOfWindowsToBe(2));
```

### NOT Condition

```java
// Negate any condition
wait.until(ExpectedConditions.not(
    ExpectedConditions.urlToBe("https://app.example.com/loading")
));

// Wait for spinner to NOT be visible
wait.until(ExpectedConditions.not(
    ExpectedConditions.visibilityOfElementLocated(By.id("loading-overlay"))
));
```

---

## ✅ Fluent Wait — Fine-Grained Control

`FluentWait` gives you full control over polling interval, timeout, and which exceptions to ignore:

```java
import org.openqa.selenium.support.ui.FluentWait;
import org.openqa.selenium.support.ui.Wait;
import org.openqa.selenium.NoSuchElementException;
import org.openqa.selenium.StaleElementReferenceException;

Wait<WebDriver> fluentWait = new FluentWait<>(driver)
    .withTimeout(Duration.ofSeconds(30))          // Max wait time
    .pollingEvery(Duration.ofMillis(500))          // Check every 500ms
    .ignoring(NoSuchElementException.class)        // Ignore while polling
    .ignoring(StaleElementReferenceException.class)// Ignore stale refs
    .withMessage("Element did not appear within 30 seconds");

// Use it like WebDriverWait
WebElement element = fluentWait.until(
    ExpectedConditions.visibilityOfElementLocated(By.id("result"))
);
```

### Custom Condition with FluentWait

```java
// Custom condition — wait until element text changes
WebElement statusBadge = driver.findElement(By.id("order-status"));

Wait<WebDriver> wait = new FluentWait<>(driver)
    .withTimeout(Duration.ofSeconds(60))
    .pollingEvery(Duration.ofSeconds(2))
    .ignoring(StaleElementReferenceException.class);

// Lambda-based custom condition
String finalStatus = wait.until(d -> {
    String status = d.findElement(By.id("order-status")).getText();
    if (status.equals("Processing") || status.equals("Pending")) {
        return null; // Return null to keep waiting
    }
    return status; // Return non-null value to stop waiting
});
System.out.println("Final order status: " + finalStatus);
```

---

## Custom Wait Conditions

Build a reusable `WaitConditions` class for application-specific conditions:

```java
package com.yourcompany.automation.utils;

import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.ExpectedCondition;

public class WaitConditions {

    /**
     * Wait for page to fully load (document.readyState == 'complete').
     */
    public static ExpectedCondition<Boolean> pageLoadComplete() {
        return driver -> {
            String state = (String) ((JavascriptExecutor) driver)
                .executeScript("return document.readyState;");
            return "complete".equals(state);
        };
    }

    /**
     * Wait for loading spinner to disappear.
     */
    public static ExpectedCondition<Boolean> spinnerToDisappear() {
        return driver -> {
            List<WebElement> spinners = driver.findElements(
                By.cssSelector(".loading-spinner, .overlay-loader, [data-testid='spinner']")
            );
            return spinners.stream().noneMatch(WebElement::isDisplayed);
        };
    }

    /**
     * Wait for element count to be greater than a minimum.
     */
    public static ExpectedCondition<List<WebElement>> elementCountGreaterThan(
            By locator, int minCount) {
        return driver -> {
            List<WebElement> elements = driver.findElements(locator);
            return elements.size() > minCount ? elements : null;
        };
    }

    /**
     * Wait for element attribute to match a value.
     */
    public static ExpectedCondition<Boolean> attributeToBe(
            By locator, String attribute, String value) {
        return driver -> {
            try {
                String actual = driver.findElement(locator).getAttribute(attribute);
                return value.equals(actual);
            } catch (NoSuchElementException | StaleElementReferenceException e) {
                return false;
            }
        };
    }

    /**
     * Wait for element CSS class to contain a specific class name.
     */
    public static ExpectedCondition<Boolean> elementHasClass(By locator, String cssClass) {
        return driver -> {
            try {
                String classes = driver.findElement(locator).getAttribute("class");
                return classes != null && classes.contains(cssClass);
            } catch (NoSuchElementException | StaleElementReferenceException e) {
                return false;
            }
        };
    }

    /**
     * Wait for AJAX calls to complete (jQuery).
     */
    public static ExpectedCondition<Boolean> jQueryAjaxComplete() {
        return driver -> {
            try {
                return (Boolean) ((JavascriptExecutor) driver)
                    .executeScript("return jQuery.active == 0");
            } catch (Exception e) {
                return true; // jQuery not present — assume complete
            }
        };
    }

    /**
     * Wait for Angular to be stable.
     */
    public static ExpectedCondition<Boolean> angularStable() {
        return driver -> {
            try {
                return (Boolean) ((JavascriptExecutor) driver).executeScript(
                    "return window.getAllAngularTestabilities()" +
                    ".every(t => t.isStable())"
                );
            } catch (Exception e) {
                return false;
            }
        };
    }
}

// Usage
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(15));

wait.until(WaitConditions.pageLoadComplete());
wait.until(WaitConditions.spinnerToDisappear());
wait.until(WaitConditions.elementHasClass(By.id("submit-btn"), "active"));
wait.until(WaitConditions.attributeToBe(By.id("status"), "data-state", "loaded"));
List<WebElement> results = wait.until(
    WaitConditions.elementCountGreaterThan(By.cssSelector(".result-item"), 0)
);
```

---

## The SmartWait Utility Class

A centralized wait utility used from Page Objects:

```java
package com.yourcompany.automation.utils;

import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.*;
import java.time.Duration;
import java.util.List;

public class SmartWait {

    private static final int DEFAULT_TIMEOUT = 10;
    private static final int LONG_TIMEOUT    = 30;
    private static final int SHORT_TIMEOUT   = 5;

    private final WebDriver driver;

    public SmartWait(WebDriver driver) {
        this.driver = driver;
    }

    private WebDriverWait wait(int seconds) {
        return new WebDriverWait(driver, Duration.ofSeconds(seconds));
    }

    public WebElement waitForVisible(By locator) {
        return wait(DEFAULT_TIMEOUT).until(
            ExpectedConditions.visibilityOfElementLocated(locator)
        );
    }

    public WebElement waitForClickable(By locator) {
        return wait(DEFAULT_TIMEOUT).until(
            ExpectedConditions.elementToBeClickable(locator)
        );
    }

    public void waitForInvisible(By locator) {
        wait(DEFAULT_TIMEOUT).until(
            ExpectedConditions.invisibilityOfElementLocated(locator)
        );
    }

    public List<WebElement> waitForAll(By locator) {
        return wait(DEFAULT_TIMEOUT).until(
            ExpectedConditions.visibilityOfAllElementsLocatedBy(locator)
        );
    }

    public void waitForUrl(String urlFragment) {
        wait(DEFAULT_TIMEOUT).until(
            ExpectedConditions.urlContains(urlFragment)
        );
    }

    public void waitForPageLoad() {
        wait(LONG_TIMEOUT).until(WaitConditions.pageLoadComplete());
    }

    public void waitForSpinner() {
        By spinnerLocator = By.cssSelector("[data-testid='loading-spinner']");
        // If spinner is present, wait for it to disappear
        if (!driver.findElements(spinnerLocator).isEmpty()) {
            waitForInvisible(spinnerLocator);
        }
    }

    public boolean isPresent(By locator) {
        try {
            new WebDriverWait(driver, Duration.ofSeconds(SHORT_TIMEOUT))
                .until(ExpectedConditions.presenceOfElementLocated(locator));
            return true;
        } catch (TimeoutException e) {
            return false;
        }
    }

    public String waitForText(By locator, String expectedText) {
        wait(DEFAULT_TIMEOUT).until(
            ExpectedConditions.textToBePresentInElementLocated(locator, expectedText)
        );
        return driver.findElement(locator).getText();
    }
}
```

---

## Handling StaleElementReferenceException

The most common timing issue after navigation or DOM updates:

```java
// Problem: element reference becomes stale after DOM update
WebElement row = driver.findElement(By.cssSelector("tr.order-row"));
driver.findElement(By.id("refresh-btn")).click();
row.getText(); // ← StaleElementReferenceException — DOM was updated

// Solution 1: Re-find element after DOM change
driver.findElement(By.id("refresh-btn")).click();
wait.until(WaitConditions.spinnerToDisappear());
WebElement freshRow = driver.findElement(By.cssSelector("tr.order-row")); // Re-find
freshRow.getText(); // ✅

// Solution 2: Retry wrapper
public String getTextWithRetry(By locator, int maxRetries) {
    for (int i = 0; i < maxRetries; i++) {
        try {
            return driver.findElement(locator).getText();
        } catch (StaleElementReferenceException e) {
            if (i == maxRetries - 1) throw e;
            try { Thread.sleep(500); } catch (InterruptedException ie) { Thread.currentThread().interrupt(); }
        }
    }
    return null;
}

// Solution 3: Use Fluent Wait with StaleElementReferenceException ignored
Wait<WebDriver> fluentWait = new FluentWait<>(driver)
    .withTimeout(Duration.ofSeconds(10))
    .pollingEvery(Duration.ofMillis(300))
    .ignoring(StaleElementReferenceException.class);

String text = fluentWait.until(d -> d.findElement(locator).getText());
```

---

## Thread.sleep() — The Anti-Pattern

```java
// ❌ NEVER USE THIS in production test code
Thread.sleep(3000); // Arbitrary 3-second sleep

// Why it's bad:
// 1. Wastes time when the app responds in 200ms
// 2. Still fails when the app takes 4 seconds
// 3. Accumulates — 10 tests × 3 sleeps × 3s = 90s added to your suite
// 4. Masks the real problem — you don't know WHY you need to wait

// ✅ ALWAYS replace with explicit wait
wait.until(ExpectedConditions.visibilityOfElementLocated(By.id("result")));
wait.until(WaitConditions.spinnerToDisappear());
```

**The only acceptable use of `Thread.sleep()`** is in debugging during development to slow down execution and observe what's happening. Never commit it.

---

## Wait Strategy Decision Chart

```
Before interacting with an element, ask:

1. Could the element be absent initially?
   → wait.until(presenceOfElementLocated(...))

2. Could the element be present but not visible?
   → wait.until(visibilityOfElementLocated(...))

3. Could the element be visible but not yet clickable (disabled/covered)?
   → wait.until(elementToBeClickable(...))

4. Am I waiting for a loading overlay/spinner to go away?
   → wait.until(invisibilityOfElementLocated(spinner))

5. Am I waiting for a page navigation to complete?
   → wait.until(urlContains("/expected-path"))
   → wait.until(titleContains("Expected Title"))

6. Am I waiting for text to update dynamically?
   → wait.until(textToBePresentInElementLocated(..., "expected text"))

7. Is the condition complex or app-specific?
   → write a custom ExpectedCondition lambda
```

---

## Summary

Synchronization is the discipline that separates reliable automation from flaky automation. The rule is simple: use only explicit waits with meaningful `ExpectedConditions`, set implicit wait to zero, and never use `Thread.sleep()`. Build a `SmartWait` utility class and custom `WaitConditions` for your application-specific loading behaviors. Every page action should be wrapped in an appropriate wait — not as a habit, but as a deliberate choice to wait for the right state.
