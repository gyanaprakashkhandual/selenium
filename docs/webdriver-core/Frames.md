# Handling Frames and iFrames

## Overview

An `<iframe>` (inline frame) is an HTML document embedded inside another HTML document. Many applications use iFrames for embedded maps, payment forms (like Stripe), rich text editors (like TinyMCE), video players, and third-party widgets. Selenium cannot interact with elements inside an iFrame until you explicitly switch context into it. This document covers every aspect of iFrame handling with professional patterns.

---

## Understanding iFrame Context

The browser maintains a concept of the **active frame context**. When a page loads, the context is the **main document** (the top-level page). When you call `driver.findElement()`, Selenium searches only within the current context.

If an element is inside an iFrame, `driver.findElement()` will throw `NoSuchElementException` — even if the element is clearly visible on screen — because you are searching the wrong context.

```
Main Document (top-level)
├── Regular elements
├── <iframe id="payment-frame">
│   └── Payment form elements  ← Can't find these until you switch in
└── More regular elements
```

---

## Switching into an iFrame

Selenium provides three ways to identify an iFrame to switch into:

### Method 1: By ID or Name (Most Reliable)

```java
// HTML: <iframe id="payment-iframe" src="..."></iframe>
driver.switchTo().frame("payment-iframe"); // By ID

// HTML: <iframe name="login-frame" src="..."></iframe>
driver.switchTo().frame("login-frame"); // By name attribute
```

### Method 2: By Index (0-Based Position)

```java
// First iframe on the page (index 0)
driver.switchTo().frame(0);

// Third iframe (index 2)
driver.switchTo().frame(2);
```

> **Caution:** Index-based switching is fragile. If a new iFrame is added to the page, the index of all subsequent iFrames shifts. Use ID/name or WebElement whenever possible.

### Method 3: By WebElement (Most Flexible)

```java
// Find the iFrame element first, then switch into it
WebElement iframeElement = driver.findElement(By.cssSelector("iframe.payment-form"));
driver.switchTo().frame(iframeElement);

// With explicit wait — recommended
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));
WebElement iframeElement = wait.until(
    ExpectedConditions.presenceOfElementLocated(By.cssSelector("iframe#payment-frame"))
);
driver.switchTo().frame(iframeElement);
```

### Wait for Frame and Switch (Built-in ExpectedCondition)

```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// Wait for the frame to be available, then switch into it — single line
wait.until(ExpectedConditions.frameToBeAvailableAndSwitchToIt(By.id("payment-frame")));
wait.until(ExpectedConditions.frameToBeAvailableAndSwitchToIt(By.name("login-frame")));
wait.until(ExpectedConditions.frameToBeAvailableAndSwitchToIt(
    By.cssSelector("iframe[data-testid='payment-iframe']")
));
```

---

## Switching Back to Main Document

After interacting with elements inside a frame, you must switch back to the main document before interacting with elements outside the frame:

```java
// Switch back to the outermost document (main page)
driver.switchTo().defaultContent();

// Switch back one level (to parent frame — useful for nested frames)
driver.switchTo().parentFrame();
```

---

## Complete iFrame Interaction Pattern

```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// Step 1: Navigate to the page
driver.get("https://app.example.com/checkout");

// Step 2: Interact with main page elements
wait.until(ExpectedConditions.visibilityOfElementLocated(By.id("order-summary")));
System.out.println("Order: " + driver.findElement(By.id("order-total")).getText());

// Step 3: Switch into the payment iFrame
wait.until(ExpectedConditions.frameToBeAvailableAndSwitchToIt(
    By.id("payment-iframe")
));

// Step 4: Interact with elements INSIDE the iFrame
wait.until(ExpectedConditions.visibilityOfElementLocated(By.id("card-number")));
driver.findElement(By.id("card-number")).sendKeys("4111111111111111");
driver.findElement(By.id("expiry")).sendKeys("12/28");
driver.findElement(By.id("cvv")).sendKeys("123");
driver.findElement(By.id("card-name")).sendKeys("John Doe");

// Step 5: Switch BACK to main document
driver.switchTo().defaultContent();

// Step 6: Continue on main page
driver.findElement(By.id("place-order-btn")).click();
wait.until(ExpectedConditions.urlContains("/order-confirmation"));
```

---

## Nested iFrames

Some pages embed iFrames within iFrames. You must switch into each level sequentially:

```html
<!-- Main document -->
<iframe id="outer-frame">
  <!-- Inside outer frame -->
  <iframe id="inner-frame">
    <!-- Inside inner frame — target elements here -->
    <input id="nested-input" />
  </iframe>
</iframe>
```

```java
// Step 1: Switch into outer frame
driver.switchTo().frame("outer-frame");

// Step 2: Switch into inner frame (from within outer)
driver.switchTo().frame("inner-frame");

// Step 3: Interact with elements in the innermost frame
driver.findElement(By.id("nested-input")).sendKeys("value");

// Step 4: Go back one level (to outer frame)
driver.switchTo().parentFrame();

// Step 5: Interact with outer frame elements
driver.findElement(By.id("outer-element")).click();

// Step 6: Go back to main document
driver.switchTo().defaultContent();
```

---

## Real-World Use Cases

### Stripe Payment iFrame

Stripe embeds payment form fields in separate iFrames for security. Each field (card number, expiry, CVC) is in its own iFrame:

```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(15));

// Card Number field is in its own iframe
wait.until(ExpectedConditions.frameToBeAvailableAndSwitchToIt(
    By.cssSelector("iframe[name*='__privateStripeFrame'][title*='card number']")
));
wait.until(ExpectedConditions.elementToBeClickable(
    By.cssSelector("input[name='cardnumber']")
)).sendKeys("4242424242424242");
driver.switchTo().defaultContent();

// Expiry date
wait.until(ExpectedConditions.frameToBeAvailableAndSwitchToIt(
    By.cssSelector("iframe[name*='__privateStripeFrame'][title*='expiration date']")
));
driver.findElement(By.cssSelector("input[name='exp-date']")).sendKeys("12/28");
driver.switchTo().defaultContent();

// CVC
wait.until(ExpectedConditions.frameToBeAvailableAndSwitchToIt(
    By.cssSelector("iframe[name*='__privateStripeFrame'][title*='security code']")
));
driver.findElement(By.cssSelector("input[name='cvc']")).sendKeys("123");
driver.switchTo().defaultContent();
```

### TinyMCE Rich Text Editor

TinyMCE loads the editable area inside an iFrame:

```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// Wait for TinyMCE to load and switch to its iframe
wait.until(ExpectedConditions.frameToBeAvailableAndSwitchToIt(
    By.id("tinymce-editor_ifr") // TinyMCE iframe ID pattern
));

// The editable area is the body inside the iframe
WebElement editorBody = wait.until(
    ExpectedConditions.visibilityOfElementLocated(By.id("tinymce"))
);

// Clear and type (or use JS for complex input)
editorBody.clear();
editorBody.sendKeys("This is the article content I am typing.");

// OR use JavaScript for complex content
((JavascriptExecutor) driver).executeScript(
    "arguments[0].innerHTML = '<p>Rich <strong>text</strong> content</p>';",
    editorBody
);

// Switch back
driver.switchTo().defaultContent();

// Submit the form
driver.findElement(By.id("save-article-btn")).click();
```

### Google reCAPTCHA (Detection Only)

Note: Automating reCAPTCHA is against Google's Terms of Service. The correct approach in automated tests is to use test credentials that bypass CAPTCHA. However, detecting its presence is legitimate:

```java
// Detect if reCAPTCHA iFrame is present (to validate it shows)
boolean captchaPresent = !driver.findElements(
    By.cssSelector("iframe[src*='recaptcha']")
).isEmpty();
Assert.assertTrue(captchaPresent, "reCAPTCHA should be displayed on the page");
```

---

## iFrame Utility Class

```java
package com.yourcompany.automation.utils;

import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.*;
import java.time.Duration;
import java.util.List;

public class FrameHandler {

    private final WebDriver driver;
    private final WebDriverWait wait;

    public FrameHandler(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    /**
     * Switch into a frame by locator and wait for it to be ready.
     */
    public void switchToFrame(By frameLocator) {
        wait.until(ExpectedConditions.frameToBeAvailableAndSwitchToIt(frameLocator));
    }

    /**
     * Switch into a frame by ID or name.
     */
    public void switchToFrame(String idOrName) {
        wait.until(d -> {
            try {
                d.switchTo().frame(idOrName);
                return true;
            } catch (NoSuchFrameException e) {
                return false;
            }
        });
    }

    /**
     * Switch into a frame by index.
     */
    public void switchToFrame(int index) {
        wait.until(d -> {
            try {
                d.switchTo().frame(index);
                return true;
            } catch (NoSuchFrameException e) {
                return false;
            }
        });
    }

    /**
     * Return to the main document.
     */
    public void switchToMain() {
        driver.switchTo().defaultContent();
    }

    /**
     * Return one level up (to parent frame).
     */
    public void switchToParent() {
        driver.switchTo().parentFrame();
    }

    /**
     * Execute an action inside a frame, then switch back automatically.
     *
     * Usage: frameHandler.withinFrame(By.id("payment-iframe"), () -> {
     *     driver.findElement(By.id("card-number")).sendKeys("4111...");
     * });
     */
    public void withinFrame(By frameLocator, Runnable actions) {
        switchToFrame(frameLocator);
        try {
            actions.run();
        } finally {
            switchToMain();
        }
    }

    /**
     * Get the count of all iFrames on the current page.
     */
    public int getFrameCount() {
        return driver.findElements(By.tagName("iframe")).size();
    }

    /**
     * Get all iframe src attributes on the page.
     */
    public List<String> getFrameSources() {
        return driver.findElements(By.tagName("iframe")).stream()
            .map(iframe -> iframe.getAttribute("src"))
            .collect(java.util.stream.Collectors.toList());
    }

    /**
     * Check if an iframe with the given ID exists on the page.
     */
    public boolean frameExists(String frameId) {
        return !driver.findElements(By.id(frameId)).isEmpty();
    }
}
```

### Using the withinFrame Pattern

```java
FrameHandler frameHandler = new FrameHandler(driver);

// Elegant, clean frame interaction
frameHandler.withinFrame(By.id("payment-iframe"), () -> {
    driver.findElement(By.id("card-number")).sendKeys("4111111111111111");
    driver.findElement(By.id("expiry")).sendKeys("12/28");
    driver.findElement(By.id("cvv")).sendKeys("123");
});
// Automatically switched back to main document

// Continue on main page
driver.findElement(By.id("place-order-btn")).click();
```

---

## Common iFrame Errors and Fixes

| Error                                                                | Cause                                                        | Fix                                                          |
| -------------------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `NoSuchElementException` on a visible element                        | You are in the wrong context                                 | Switch into the containing iFrame first                      |
| `NoSuchFrameException`                                               | iFrame not yet loaded                                        | Use `ExpectedConditions.frameToBeAvailableAndSwitchToIt()`   |
| Element found in iFrame but actions don't work                       | Switched to wrong frame                                      | Verify frame ID/name; check for nested frames                |
| `StaleElementReferenceException` after `switchTo().defaultContent()` | Element reference from inside frame used after switching out | Re-find the element after switching contexts                 |
| Frame works locally, fails in CI                                     | Timing difference                                            | Increase wait timeout; use `frameToBeAvailableAndSwitchToIt` |

---

## Summary

iFrames create isolated document contexts that Selenium must explicitly enter. Always use `ExpectedConditions.frameToBeAvailableAndSwitchToIt()` for reliable frame switching. After working inside a frame, always call `driver.switchTo().defaultContent()` before interacting with the main page. Build a `FrameHandler` utility with the `withinFrame()` method to make iFrame interactions clean and automatically context-safe. For nested iFrames, switch in sequentially and return with `parentFrame()` level by level.
