# Retry Mechanisms

## Overview

Retry mechanisms are controlled strategies for re-executing a test or a specific operation when it fails due to transient, non-deterministic causes. In Selenium automation, some failures are genuine application defects that should fail the test permanently. Others are environmental noise — network latency, AJAX loading delays, animation timing, WebDriver communication hiccups — that cause the test to fail once and pass reliably if attempted again immediately.

A well-designed retry strategy distinguishes between these two failure types: retrying transient failures automatically while surfacing genuine defects immediately and clearly. This document covers retry at every level: element interactions, explicit waits, step-level retries, scenario-level retries, and the TestNG retry analyzer.

---

## Root Causes of Transient Failures

Understanding why failures occur is the prerequisite for choosing the right retry strategy.

| Failure Type                       | Root Cause                      | Right Solution                          |
| ---------------------------------- | ------------------------------- | --------------------------------------- |
| `NoSuchElementException`           | Element not yet in DOM          | `WebDriverWait` for visibility          |
| `StaleElementReferenceException`   | DOM re-rendered after location  | Re-locate element, retry interaction    |
| `ElementNotInteractableException`  | Element present but not ready   | Wait for clickability                   |
| `ElementClickInterceptedException` | Overlay covers the element      | Wait for overlay to disappear, JS click |
| `TimeoutException`                 | Page/AJAX load exceeded wait    | Increase wait, check network            |
| Network error on RemoteWebDriver   | Grid communication glitch       | Retry scenario once                     |
| AJAX race condition                | Element appears then disappears | Poll with FluentWait                    |
| Animation not complete             | CSS transition in progress      | Pause or wait for CSS class change      |

---

## Level 1 — Explicit Waits (The Primary Tool)

The first line of defence against transient failures is correct use of `WebDriverWait` with `ExpectedConditions`. Most "flaky" tests are not genuinely flaky — they use `Thread.sleep()` or no wait at all, and they fail when the environment is slightly slower than usual.

### Standard Explicit Wait Patterns

```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// Wait for element to be visible before reading text
WebElement element = wait.until(
    ExpectedConditions.visibilityOfElementLocated(By.id("welcomeMsg")));

// Wait for element to be clickable before clicking
WebElement button = wait.until(
    ExpectedConditions.elementToBeClickable(By.cssSelector("[data-testid='submit-btn']")));

// Wait for text to appear in an element
wait.until(ExpectedConditions.textToBePresentInElementLocated(
    By.id("status"), "Order Confirmed"));

// Wait for URL to contain expected path
wait.until(ExpectedConditions.urlContains("/dashboard"));

// Wait for overlay/loader to disappear
wait.until(ExpectedConditions.invisibilityOfElementLocated(
    By.cssSelector(".loading-spinner")));

// Wait for element count to reach expected number
wait.until(ExpectedConditions.numberOfElementsToBe(
    By.cssSelector(".product-card"), 10));
```

### FluentWait — Custom Polling with Exception Tolerance

`FluentWait` is the most configurable wait. Use it when the standard conditions do not cover the exact situation, or when you need to ignore specific exceptions during polling.

```java
public WebElement fluentWait(By locator, int timeoutSeconds) {
    return new FluentWait<>(driver)
        .withTimeout(Duration.ofSeconds(timeoutSeconds))
        .pollingEvery(Duration.ofMillis(500))
        .ignoring(NoSuchElementException.class)
        .ignoring(StaleElementReferenceException.class)
        .withMessage("Element not found after " + timeoutSeconds + "s: " + locator)
        .until(d -> d.findElement(locator));
}
```

---

## Level 2 — Stale Element Recovery

`StaleElementReferenceException` is among the most common transient failures in modern web apps. It occurs when a `WebElement` reference is held and the DOM node it points to is replaced by a re-render. React, Angular, and Vue applications re-render components frequently.

### Pattern A — Re-Locate and Retry

```java
public void clickWithStaleRecovery(By locator, int maxRetries) {
    int attempt = 0;
    while (attempt < maxRetries) {
        try {
            WebElement element = driver.findElement(locator);
            element.click();
            return; // success — exit
        } catch (StaleElementReferenceException e) {
            attempt++;
            log.warn("StaleElement on click attempt {}/{} — locator: {}",
                attempt, maxRetries, locator);
            if (attempt >= maxRetries) {
                throw new RuntimeException("Element remained stale after "
                    + maxRetries + " retries: " + locator, e);
            }
            try { Thread.sleep(300); } catch (InterruptedException ie) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

### Pattern B — Page Factory Default Behaviour

Page Factory's default lazy lookup re-finds the element on every use. This means `@FindBy` elements are naturally stale-resistant as long as `@CacheLookup` is not used.

```java
// @FindBy re-locates the element on each method call — no manual staleness handling needed
@FindBy(css = "[data-testid='submit-btn']")
private WebElement submitButton;

public void submit() {
    submitButton.click(); // finds fresh element each time
}
```

### Pattern C — Functional Retry Wrapper

A general-purpose retry wrapper for any lambda expression:

```java
package com.company.framework.utils;

import java.util.function.Supplier;

public class RetryUtils {

    private static final org.apache.logging.log4j.Logger log =
        org.apache.logging.log4j.LogManager.getLogger(RetryUtils.class);

    private RetryUtils() {}

    /**
     * Retries a supplier until it succeeds or maxAttempts is reached.
     * Retries only on the specified exception types.
     */
    @SafeVarargs
    public static <T> T retry(
            Supplier<T> action,
            int maxAttempts,
            long delayMs,
            Class<? extends Throwable>... retryOn) {

        Throwable lastException = null;

        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                return action.get();
            } catch (Throwable e) {
                boolean shouldRetry = false;
                for (Class<? extends Throwable> retryType : retryOn) {
                    if (retryType.isAssignableFrom(e.getClass())) {
                        shouldRetry = true;
                        break;
                    }
                }

                if (!shouldRetry || attempt == maxAttempts) {
                    throw new RuntimeException(
                        "Action failed after " + attempt + " attempt(s)", e);
                }

                lastException = e;
                log.warn("Retry {}/{} after {}: {}",
                    attempt, maxAttempts, e.getClass().getSimpleName(), e.getMessage());

                try { Thread.sleep(delayMs); }
                catch (InterruptedException ie) { Thread.currentThread().interrupt(); }
            }
        }

        throw new RuntimeException("Retry exhausted", lastException);
    }

    /**
     * Retries a void action.
     */
    @SafeVarargs
    public static void retryVoid(
            Runnable action,
            int maxAttempts,
            long delayMs,
            Class<? extends Throwable>... retryOn) {

        retry(() -> { action.run(); return null; }, maxAttempts, delayMs, retryOn);
    }
}
```

**Usage in page classes:**

```java
public void clickButton(By locator) {
    RetryUtils.retryVoid(
        () -> driver.findElement(locator).click(),
        3, 500,
        StaleElementReferenceException.class,
        ElementClickInterceptedException.class
    );
}

public String getTextWithRetry(By locator) {
    return RetryUtils.retry(
        () -> driver.findElement(locator).getText(),
        3, 300,
        StaleElementReferenceException.class
    );
}
```

---

## Level 3 — TestNG Retry Analyzer

The TestNG Retry Analyzer re-runs a failed `@Test` method automatically. When used with the Cucumber runner, it re-runs a failed scenario.

### IRetryAnalyzer Implementation

```java
package com.company.framework.retry;

import com.company.framework.config.ConfigReader;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.testng.IRetryAnalyzer;
import org.testng.ITestResult;

public class RetryAnalyzer implements IRetryAnalyzer {

    private static final Logger log = LogManager.getLogger(RetryAnalyzer.class);

    // Maximum number of retry attempts (in addition to the original run)
    private static final int MAX_RETRIES = ConfigReader.getInt("retry.max.attempts", 1);

    // Counter is per-test-instance — each test has its own RetryAnalyzer
    private int retryCount = 0;

    @Override
    public boolean retry(ITestResult result) {
        if (retryCount < MAX_RETRIES) {
            retryCount++;
            log.warn("Retrying test [{}] — attempt {}/{}",
                result.getName(), retryCount, MAX_RETRIES);
            return true;  // retry the test
        }
        log.error("Test [{}] FAILED after {} attempt(s)",
            result.getName(), retryCount + 1);
        return false;  // do not retry — mark as failed
    }
}
```

### RetryListener — Applies RetryAnalyzer Automatically

Rather than annotating every `@Test` method with `retryAnalyzer = RetryAnalyzer.class`, use a listener that attaches the analyzer programmatically to every test:

```java
package com.company.framework.retry;

import org.testng.IAnnotationTransformer;
import org.testng.annotations.ITestAnnotation;

import java.lang.reflect.Constructor;
import java.lang.reflect.Method;

public class RetryListener implements IAnnotationTransformer {

    @Override
    public void transform(ITestAnnotation annotation,
                          Class testClass,
                          Constructor testConstructor,
                          Method testMethod) {
        // Attach RetryAnalyzer to every @Test method automatically
        annotation.setRetryAnalyzer(RetryAnalyzer.class);
    }
}
```

### Registering the Listener in testng.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">

<suite name="Test Suite" parallel="tests" thread-count="4">

    <listeners>
        <listener class-name="com.company.framework.retry.RetryListener"/>
    </listeners>

    <test name="Regression">
        <classes>
            <class name="com.company.framework.runners.RegressionRunner"/>
        </classes>
    </test>

</suite>
```

### Configuration

```properties
# config.properties
# Number of retry attempts for failed scenarios (0 = no retry)
retry.max.attempts=1
```

Set `retry.max.attempts=1` in production (retry once). Set to `0` during active development to see failures immediately.

---

## Level 4 — Cucumber Rerun Plugin (Post-Run Retry)

The cleanest approach for scenario-level retry is the Cucumber rerun plugin: run the full suite, then immediately rerun only the failures. This avoids bloating individual scenario execution time and keeps reports clean.

### Step 1 — Add Rerun Plugin to Runner

```java
@CucumberOptions(
    features = "src/test/resources/features",
    glue     = {"stepdefinitions", "hooks"},
    plugin   = {
        "pretty",
        "json:target/cucumber-reports/regression.json",
        "rerun:target/cucumber-reports/rerun.txt"    // writes failed scenario paths
    }
)
public class RegressionRunner extends AbstractTestNGCucumberTests {}
```

### Step 2 — Rerun Runner

```java
@CucumberOptions(
    features = "@target/cucumber-reports/rerun.txt",   // @ = read paths from file
    glue     = {"stepdefinitions", "hooks"},
    plugin   = {
        "pretty",
        "json:target/cucumber-reports/rerun-results.json",
        "html:target/cucumber-reports/rerun-report.html"
    }
)
public class RerunFailedScenariosRunner extends AbstractTestNGCucumberTests {}
```

### Step 3 — CI Pipeline Execution

```bash
# Step 1 — run full suite
mvn test -Dtest=RegressionRunner

# Step 2 — rerun only failures (if rerun.txt is non-empty)
if [ -s target/cucumber-reports/rerun.txt ]; then
    echo "Re-running failed scenarios..."
    mvn test -Dtest=RerunFailedScenariosRunner
fi
```

In Jenkins:

```groovy
stage('Run Tests') {
    steps {
        sh 'mvn test -Dtest=RegressionRunner || true'  // allow failure to continue
    }
}

stage('Rerun Failed Tests') {
    when {
        expression { fileExists('target/cucumber-reports/rerun.txt') &&
                     readFile('target/cucumber-reports/rerun.txt').trim() != '' }
    }
    steps {
        sh 'mvn test -Dtest=RerunFailedScenariosRunner'
    }
}
```

---

## Level 5 — Smart Wait Patterns in BasePage

Embed retry intelligence into `BasePage` methods so every page class inherits resilient interactions without extra code.

```java
public abstract class BasePage {

    protected WebDriver driver;
    protected WebDriverWait wait;
    private static final int DEFAULT_TIMEOUT = 10;
    private static final int MAX_INTERACTION_RETRIES = 3;

    public BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait   = new WebDriverWait(driver, Duration.ofSeconds(DEFAULT_TIMEOUT));
        PageFactory.initElements(driver, this);
    }

    // -----------------------------------------------------------------------
    // Click with stale recovery and intercepted click fallback
    // -----------------------------------------------------------------------
    protected void click(By locator) {
        for (int attempt = 1; attempt <= MAX_INTERACTION_RETRIES; attempt++) {
            try {
                waitForClickable(locator).click();
                return;
            } catch (StaleElementReferenceException e) {
                log.warn("StaleElement on click attempt {}: {}", attempt, locator);
                if (attempt == MAX_INTERACTION_RETRIES) throw e;
            } catch (ElementClickInterceptedException e) {
                log.warn("Click intercepted attempt {}: {} — trying JS click", attempt, locator);
                JavascriptExecutor js = (JavascriptExecutor) driver;
                js.executeScript("arguments[0].click();", driver.findElement(locator));
                return;
            }
        }
    }

    protected void click(WebElement element) {
        for (int attempt = 1; attempt <= MAX_INTERACTION_RETRIES; attempt++) {
            try {
                waitForClickable(element).click();
                return;
            } catch (StaleElementReferenceException e) {
                log.warn("StaleElement on WebElement click attempt {}", attempt);
                if (attempt == MAX_INTERACTION_RETRIES) throw e;
            }
        }
    }

    // -----------------------------------------------------------------------
    // Type with stale recovery
    // -----------------------------------------------------------------------
    protected void type(By locator, String text) {
        for (int attempt = 1; attempt <= MAX_INTERACTION_RETRIES; attempt++) {
            try {
                WebElement element = waitForVisibility(locator);
                element.clear();
                element.sendKeys(text);
                return;
            } catch (StaleElementReferenceException e) {
                if (attempt == MAX_INTERACTION_RETRIES) throw e;
            }
        }
    }

    // -----------------------------------------------------------------------
    // getText with stale recovery
    // -----------------------------------------------------------------------
    protected String getText(By locator) {
        for (int attempt = 1; attempt <= MAX_INTERACTION_RETRIES; attempt++) {
            try {
                return waitForVisibility(locator).getText();
            } catch (StaleElementReferenceException e) {
                if (attempt == MAX_INTERACTION_RETRIES) throw e;
            }
        }
        return "";
    }

    // Private wait helpers
    private WebElement waitForVisibility(By locator) {
        return wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
    }

    private WebElement waitForClickable(By locator) {
        return wait.until(ExpectedConditions.elementToBeClickable(locator));
    }

    private WebElement waitForClickable(WebElement element) {
        return wait.until(ExpectedConditions.elementToBeClickable(element));
    }
}
```

---

## When NOT to Retry

Retry is not a solution for genuine failures. It is only appropriate for transient environmental issues.

**Do not retry assertion failures.** If `Assert.assertEquals("Welcome", dashboardPage.getTitle())` fails, the application is returning the wrong title. Retrying will not fix the application. It will just delay the failure.

**Do not retry failures caused by broken test data.** If the test expects order `ORD-9001` to exist but it was never created, retrying will fail the same way every time.

**Do not mask slow environments with retries.** If a page consistently takes 35 seconds to load against a 30-second timeout, increasing the retry count is not the answer. Increase the timeout or investigate the environment.

**Set a maximum of 1–2 retries.** More than two retries inflates test execution time significantly and suggests the test is genuinely unreliable rather than occasionally affected by environment noise.

---

## Retry Configuration Reference

```properties
# config.properties

# TestNG retry analyzer — max additional attempts after first failure
# 0 = no retry, 1 = retry once (recommended), 2 = retry twice (max recommended)
retry.max.attempts=1

# Delay between stale element recovery attempts (milliseconds)
retry.stale.delay.ms=300

# Max attempts for BasePage interaction retries
retry.interaction.max=3

# FluentWait polling interval (milliseconds)
retry.fluent.poll.ms=500
```

---

## Summary

Retry mechanisms are a precision tool, not a blunt instrument. The right solution for most "flaky" tests is correct explicit waits — `WebDriverWait` with `ExpectedConditions` — not retries. For genuine environmental transience, retry at three targeted levels: stale element recovery in `BasePage` interaction methods using functional retry wrappers, TestNG `IRetryAnalyzer` with a `RetryListener` for scenario-level retry via the runner, and the Cucumber rerun plugin for post-suite retry of the complete failure set. Never retry assertion failures. Never use retries to mask slow environments or broken test data. With these strategies correctly applied, the test suite becomes resilient to infrastructure noise while remaining honest about genuine application defects.

---

## Related Topics

- BasePage.md — Interaction methods that embed retry logic
- Waits.md — WebDriverWait and FluentWait detailed reference
- ParallelExecution.md — Retry in parallel execution context
- Hooks.md — @After cleanup that must succeed after retried scenarios
- TestIndependence.md — Independent scenarios are more retry-safe
