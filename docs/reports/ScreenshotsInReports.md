# Screenshots in Reports

## Overview

Embedding screenshots in test reports is one of the highest-value practices in Selenium automation. A screenshot captured at the moment of a test failure tells you in one glance what the browser was showing when the assertion failed — something no stack trace or log message can communicate as quickly. Professional frameworks go further: capturing screenshots at each test step, on demand during debugging, and after key page transitions, embedding them in Extent Reports, Allure, and Cucumber HTML reports where they are immediately visible in context.

This document covers every screenshot pattern: on-failure capture, step-level capture, full-page capture, element-only capture, and integration with all three reporting systems.

---

## Screenshot Types

| Type                 | When Taken                               | Use Case                  |
| -------------------- | ---------------------------------------- | ------------------------- |
| Failure screenshot   | When a scenario fails in `@After`        | Diagnosing failures       |
| Step screenshot      | After every step (or configurable steps) | Complete visual trace     |
| On-demand screenshot | Manually called from step definitions    | Asserting visual state    |
| Full-page screenshot | Captures content below the fold          | Long-page validation      |
| Element screenshot   | Captures a single WebElement             | Precise visual comparison |

---

## ScreenshotUtils — The Central Capture Class

All screenshot operations are centralized in `ScreenshotUtils`. This class is called from Hooks, step definitions, and page objects.

```java
package com.company.framework.utils;

import com.company.framework.config.ConfigReader;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.*;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class ScreenshotUtils {

    private static final Logger log = LogManager.getLogger(ScreenshotUtils.class);
    private static final DateTimeFormatter TS = DateTimeFormatter.ofPattern("yyyyMMdd_HHmmss_SSS");

    private static final String SCREENSHOT_DIR =
        ConfigReader.get("screenshots.dir", "target/screenshots");

    private ScreenshotUtils() {}

    // -----------------------------------------------------------------------
    // Core capture methods
    // -----------------------------------------------------------------------

    /**
     * Captures a full-browser screenshot and saves it to disk.
     * Returns the absolute path of the saved file, or null on failure.
     */
    public static String saveToDisk(WebDriver driver, String name) {
        ensureDirectoryExists();
        String fileName  = sanitize(name) + "_" + LocalDateTime.now().format(TS) + ".png";
        String filePath  = SCREENSHOT_DIR + "/" + fileName;

        try {
            File source = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
            Files.copy(source.toPath(), Paths.get(filePath));
            log.info("Screenshot saved: {}", filePath);
            return filePath;
        } catch (IOException e) {
            log.error("Failed to save screenshot '{}': {}", fileName, e.getMessage());
            return null;
        }
    }

    /**
     * Returns a screenshot as a byte array.
     * Used for embedding in Cucumber scenario, Extent, and Allure reports.
     */
    public static byte[] asBytes(WebDriver driver) {
        try {
            return ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
        } catch (Exception e) {
            log.error("Failed to capture screenshot as bytes: {}", e.getMessage());
            return new byte[0];
        }
    }

    /**
     * Returns a screenshot as a Base64-encoded string.
     * Used by Extent Reports MediaEntityBuilder for inline HTML embedding.
     */
    public static String asBase64(WebDriver driver) {
        try {
            return ((TakesScreenshot) driver).getScreenshotAs(OutputType.BASE64);
        } catch (Exception e) {
            log.error("Failed to capture screenshot as base64: {}", e.getMessage());
            return "";
        }
    }

    /**
     * Captures a screenshot of a specific WebElement only.
     */
    public static byte[] elementAsBytes(WebElement element) {
        try {
            return element.getScreenshotAs(OutputType.BYTES);
        } catch (Exception e) {
            log.error("Failed to capture element screenshot: {}", e.getMessage());
            return new byte[0];
        }
    }

    // -----------------------------------------------------------------------
    // Full-page screenshot
    // -----------------------------------------------------------------------

    /**
     * Expands browser window to full page height before capturing.
     * Restores original size after capture.
     * Use for pages with content below the initial viewport.
     */
    public static byte[] fullPageAsBytes(WebDriver driver) {
        Dimension originalSize = driver.manage().window().getSize();
        try {
            JavascriptExecutor js = (JavascriptExecutor) driver;
            Long scrollHeight = (Long) js.executeScript("return document.body.scrollHeight;");
            driver.manage().window().setSize(new Dimension(1920, scrollHeight.intValue()));
            return asBytes(driver);
        } finally {
            driver.manage().window().setSize(originalSize);
        }
    }

    // -----------------------------------------------------------------------
    // Utilities
    // -----------------------------------------------------------------------

    public static void ensureDirectoryExists() {
        File dir = new File(SCREENSHOT_DIR);
        if (!dir.exists()) {
            dir.mkdirs();
            log.debug("Created screenshots directory: {}", SCREENSHOT_DIR);
        }
    }

    private static String sanitize(String name) {
        return name.replaceAll("[^a-zA-Z0-9_\\-]", "_").replaceAll("_+", "_");
    }
}
```

---

## Embedding Screenshots in Cucumber Scenario Reports

Cucumber's `Scenario` object has an `attach()` method that embeds any binary content into the native Cucumber HTML/JSON report. Screenshots embedded this way appear inline in both the standard Cucumber HTML report and the `cucumber-reporting` enhanced report.

### On Failure (Most Common Pattern)

```java
package com.company.framework.hooks;

import com.company.framework.driver.DriverManager;
import com.company.framework.utils.ScreenshotUtils;
import io.cucumber.java.After;
import io.cucumber.java.Scenario;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class Hooks {

    private static final Logger log = LogManager.getLogger(Hooks.class);

    @After(order = 2)
    public void embedScreenshotOnFailure(Scenario scenario) {
        if (scenario.isFailed() && DriverManager.isDriverInitialized()) {
            try {
                byte[] screenshot = ScreenshotUtils.asBytes(DriverManager.getDriver());

                if (screenshot.length > 0) {
                    // Embeds screenshot inline in Cucumber HTML/JSON report
                    scenario.attach(screenshot, "image/png", scenario.getName());

                    // Also save to disk for direct file access
                    ScreenshotUtils.saveToDisk(DriverManager.getDriver(), scenario.getName());

                    // Log current URL for context
                    scenario.log("URL at failure: " + DriverManager.getDriver().getCurrentUrl());
                    scenario.log("Page title: " + DriverManager.getDriver().getTitle());

                    log.warn("Screenshot captured for failed scenario: {}", scenario.getName());
                }
            } catch (Exception e) {
                log.error("Screenshot capture failed for scenario [{}]: {}",
                    scenario.getName(), e.getMessage());
            }
        }
    }

    @After(order = 1)
    public void tearDownDriver(Scenario scenario) {
        DriverManager.quitDriver();
    }
}
```

### On Every Scenario (Pass and Fail)

```java
@After(order = 2)
public void embedScreenshotAlways(Scenario scenario) {
    if (!DriverManager.isDriverInitialized()) return;
    try {
        byte[] screenshot = ScreenshotUtils.asBytes(DriverManager.getDriver());
        String label = scenario.isFailed()
            ? "FAILURE — " + scenario.getName()
            : "Final State — " + scenario.getName();
        scenario.attach(screenshot, "image/png", label);
    } catch (Exception e) {
        log.error("Screenshot failed: {}", e.getMessage());
    }
}
```

### On Every Step (Step-Level Screenshots)

```java
package com.company.framework.hooks;

import io.cucumber.java.AfterStep;
import io.cucumber.java.Scenario;

public class StepHooks {

    @AfterStep
    public void embedStepScreenshot(Scenario scenario) {
        if (!DriverManager.isDriverInitialized()) return;
        try {
            byte[] screenshot = ScreenshotUtils.asBytes(DriverManager.getDriver());
            scenario.attach(screenshot, "image/png", "Step screenshot");
        } catch (Exception e) {
            // Non-fatal — step screenshots are best-effort
        }
    }
}
```

Step-level screenshots produce the most detailed visual trace but significantly increase report file size and test execution time. Use them only when diagnosing specific complex failures or for demo/showcase runs.

---

## Embedding Screenshots in Extent Reports

Extent Reports embeds screenshots using `MediaEntityBuilder.createScreenCaptureFromBase64String()` for inline Base64 embedding, or a file path for disk-based images.

### Inline Base64 Embedding (Recommended — Portable Report)

```java
package com.company.framework.hooks;

import com.aventstack.extentreports.ExtentTest;
import com.aventstack.extentreports.MediaEntityBuilder;
import com.aventstack.extentreports.Status;
import com.company.framework.driver.DriverManager;
import com.company.framework.reports.ExtentReportManager;
import com.company.framework.utils.ScreenshotUtils;
import io.cucumber.java.After;
import io.cucumber.java.Scenario;

public class ExtentHooks {

    @After(order = 2)
    public void addScreenshotToExtentReport(Scenario scenario) {
        ExtentTest test = ExtentReportManager.getTest();
        if (test == null || !DriverManager.isDriverInitialized()) return;

        try {
            String base64 = ScreenshotUtils.asBase64(DriverManager.getDriver());

            if (scenario.isFailed()) {
                // Embed screenshot inline in the FAILED log entry
                test.fail(
                    "Scenario FAILED: " + scenario.getName(),
                    MediaEntityBuilder
                        .createScreenCaptureFromBase64String(base64, "Failure Screenshot")
                        .build()
                );
                test.fail("URL: " + DriverManager.getDriver().getCurrentUrl());

            } else {
                // Optionally embed final-state screenshot for passing scenarios
                test.pass(
                    "Scenario PASSED",
                    MediaEntityBuilder
                        .createScreenCaptureFromBase64String(base64, "Final State")
                        .build()
                );
            }

        } catch (Exception e) {
            test.log(Status.WARNING, "Screenshot attachment failed: " + e.getMessage());
        }
    }
}
```

### File Path Embedding

For very large test suites where Base64 inflates the HTML file size, save screenshots to disk and reference them by relative path:

```java
String filePath = ScreenshotUtils.saveToDisk(
    DriverManager.getDriver(), scenario.getName());

if (filePath != null) {
    // Path must be relative to the report HTML file location
    String relativePath = Paths.get("target/extent-reports")
        .relativize(Paths.get(filePath)).toString();

    test.fail("FAILED",
        MediaEntityBuilder.createScreenCaptureFromPath(relativePath, "Failure").build()
    );
}
```

### Adding Screenshot at Specific Step in Extent

```java
// In a step definition, take an on-demand screenshot
@Then("the order confirmation page should display the order total")
public void orderTotalShouldBeDisplayed() {
    ExtentTest test = ExtentReportManager.getTest();

    // Take screenshot before assertion
    String base64 = ScreenshotUtils.asBase64(context.getDriver());
    test.info("Order confirmation page state",
        MediaEntityBuilder.createScreenCaptureFromBase64String(base64, "Order Confirmation").build()
    );

    // Assert
    BigDecimal displayed = context.getConfirmationPage().getOrderTotal();
    BigDecimal expected  = context.getCapturedOrderTotal();
    Assert.assertEquals(displayed, expected, "Order total mismatch");
    test.pass("Order total verified: " + displayed);
}
```

---

## Embedding Screenshots in Allure Reports

Allure uses `Allure.addAttachment()` or the `@Attachment` annotation.

### On Failure in @After Hook

```java
package com.company.framework.hooks;

import io.cucumber.java.After;
import io.cucumber.java.Scenario;
import io.qameta.allure.Allure;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.io.ByteArrayInputStream;

public class AllureScreenshotHook {

    private static final Logger log = LogManager.getLogger(AllureScreenshotHook.class);

    @After
    public void attachScreenshotToAllure(Scenario scenario) {
        if (scenario.isFailed() && DriverManager.isDriverInitialized()) {
            try {
                byte[] screenshot = ScreenshotUtils.asBytes(DriverManager.getDriver());

                Allure.addAttachment(
                    "Failure Screenshot",
                    "image/png",
                    new ByteArrayInputStream(screenshot),
                    ".png"
                );

                Allure.addAttachment(
                    "URL at Failure",
                    "text/plain",
                    DriverManager.getDriver().getCurrentUrl()
                );

                Allure.addAttachment(
                    "Page Title",
                    "text/plain",
                    DriverManager.getDriver().getTitle()
                );

                log.warn("Allure screenshot attached for: {}", scenario.getName());

            } catch (Exception e) {
                log.error("Allure screenshot attachment failed: {}", e.getMessage());
            }
        }
    }
}
```

### Using @Attachment Annotation in Page Classes

```java
import io.qameta.allure.Attachment;

public class BasePage {

    @Attachment(value = "{name}", type = "image/png", fileExtension = ".png")
    public byte[] attachScreenshot(String name) {
        return ScreenshotUtils.asBytes(driver);
    }

    @Attachment(value = "Page Source", type = "text/html", fileExtension = ".html")
    public String attachPageSource() {
        return driver.getPageSource();
    }
}
```

### Attaching Element Screenshots to Allure

```java
@Then("the product image should be displayed correctly")
public void productImageShouldBeDisplayed() {
    WebElement productImage = context.getProductPage().getProductImage();

    // Attach element-only screenshot
    byte[] elementScreenshot = ScreenshotUtils.elementAsBytes(productImage);
    Allure.addAttachment("Product Image",
        "image/png",
        new ByteArrayInputStream(elementScreenshot),
        ".png"
    );

    Assert.assertTrue(productImage.isDisplayed(), "Product image not visible");
}
```

---

## Unified Screenshot Hook — All Three Systems

A single `@After` hook can serve all three reporting systems simultaneously:

```java
package com.company.framework.hooks;

import com.aventstack.extentreports.ExtentTest;
import com.aventstack.extentreports.MediaEntityBuilder;
import com.company.framework.driver.DriverManager;
import com.company.framework.reports.ExtentReportManager;
import com.company.framework.utils.ScreenshotUtils;
import io.cucumber.java.After;
import io.cucumber.java.Scenario;
import io.qameta.allure.Allure;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.io.ByteArrayInputStream;

public class UnifiedScreenshotHook {

    private static final Logger log = LogManager.getLogger(UnifiedScreenshotHook.class);

    @After(order = 2)
    public void captureAndAttachScreenshot(Scenario scenario) {

        if (!DriverManager.isDriverInitialized()) return;
        if (!scenario.isFailed()) return;

        try {
            // Capture once — reuse for all report systems
            byte[] screenshotBytes = ScreenshotUtils.asBytes(DriverManager.getDriver());
            String base64          = ScreenshotUtils.asBase64(DriverManager.getDriver());
            String currentUrl      = DriverManager.getDriver().getCurrentUrl();
            String pageTitle       = DriverManager.getDriver().getTitle();

            if (screenshotBytes.length == 0) {
                log.error("Screenshot capture returned empty bytes for: {}", scenario.getName());
                return;
            }

            // ---- 1. Cucumber native report ----
            scenario.attach(screenshotBytes, "image/png", "Failure Screenshot");
            scenario.log("URL: " + currentUrl);
            scenario.log("Page: " + pageTitle);

            // ---- 2. Extent Reports ----
            ExtentTest extentTest = ExtentReportManager.getTest();
            if (extentTest != null) {
                extentTest.fail("Scenario FAILED: " + scenario.getName(),
                    MediaEntityBuilder
                        .createScreenCaptureFromBase64String(base64, "Failure Screenshot")
                        .build()
                );
                extentTest.fail("URL: " + currentUrl);
            }

            // ---- 3. Allure ----
            Allure.addAttachment("Failure Screenshot", "image/png",
                new ByteArrayInputStream(screenshotBytes), ".png");
            Allure.addAttachment("URL at Failure", "text/plain", currentUrl);

            // ---- 4. Disk file (for CI artifact storage) ----
            ScreenshotUtils.saveToDisk(DriverManager.getDriver(), scenario.getName());

            log.warn("Screenshot attached to all reports for: {}", scenario.getName());

        } catch (Exception e) {
            log.error("Unified screenshot hook failed for [{}]: {}",
                scenario.getName(), e.getMessage());
        }
    }
}
```

---

## Screenshot Naming Strategy

Consistent screenshot naming makes failure archives easy to navigate:

```java
// Pattern: FeatureName_ScenarioName_Timestamp
public static String buildScreenshotName(Scenario scenario) {
    String scenarioId = scenario.getId();  // e.g., "features/login.feature:15"

    // Extract feature name from ID
    String featurePart = scenarioId.contains("/")
        ? scenarioId.substring(scenarioId.lastIndexOf('/') + 1, scenarioId.indexOf(':'))
        : "unknown-feature";

    String scenarioName = scenario.getName()
        .replaceAll("[^a-zA-Z0-9\\s]", "")
        .replaceAll("\\s+", "_")
        .toLowerCase();

    return featurePart + "_" + scenarioName;
}
```

---

## Screenshot Configuration

Control screenshot behavior through `config.properties`:

```properties
# Whether to capture screenshots on failure
screenshots.on.failure=true

# Whether to capture screenshots on passing scenarios too
screenshots.always=false

# Storage directory
screenshots.dir=target/screenshots

# Whether to embed as Base64 (true) or file path (false) in Extent
screenshots.embed.base64=true

# Maximum screenshot age in days before cleanup
screenshots.retain.days=7
```

---

## Cleaning Up Old Screenshots

In CI environments, screenshots accumulate across runs. Add a cleanup step:

```java
@BeforeAll
public static void cleanupOldScreenshots() {
    String screenshotDir = ConfigReader.get("screenshots.dir", "target/screenshots");
    File dir = new File(screenshotDir);
    if (!dir.exists()) {
        dir.mkdirs();
        return;
    }

    long cutoff = System.currentTimeMillis() - (7 * 24 * 60 * 60 * 1000L); // 7 days

    File[] old = dir.listFiles(f -> f.lastModified() < cutoff);
    if (old != null) {
        for (File file : old) {
            file.delete();
        }
        log.info("Cleaned {} old screenshot(s)", old.length);
    }
}
```

---

## Best Practices

**Capture once, embed everywhere.** Take the screenshot once as bytes and reuse the same array for Cucumber, Extent, and Allure. Never call `driver.getScreenshotAs()` multiple times for the same failure.

**Capture before driver.quit().** The screenshot must be taken in a hook with a higher `order` number than the quit hook. The quit destroys the browser — you cannot screenshot an already-closed driver.

**Use Base64 for portable reports.** HTML reports containing Base64-embedded screenshots are self-contained and can be shared via email or Slack without attaching separate image files. Use file paths only when report size is a concern.

**Name screenshots meaningfully.** A screenshot named `test_20250315_142233.png` requires opening the file to understand what it shows. A screenshot named `login_feature_valid_credentials_20250315.png` is immediately identifiable.

**Do not take step screenshots in production CI runs.** Step-by-step screenshots multiply the report size by the number of steps per scenario. Reserve them for targeted debugging sessions.

**Always wrap screenshot code in try-catch.** A screenshot failure during teardown should never prevent the test result from being recorded or the driver from being quit.

---

## Summary

Screenshot capture is the bridge between an automated test failure and a human understanding what went wrong. `ScreenshotUtils` provides the central capture layer serving bytes, Base64, and disk files. The unified `@After` hook embeds the screenshot into Cucumber's native report, Extent Reports, and Allure simultaneously from a single capture call. Step-level screenshots via `@AfterStep` produce a complete visual trace for complex debugging. Element screenshots provide precise targeted captures. Together, a well-configured screenshot strategy transforms test reports from text summaries into visual evidence that any team member can interpret immediately.

---

## Related Topics

- ExtentReports.md — MediaEntityBuilder for Base64 screenshot embedding
- AllureReports.md — Allure.addAttachment and @Attachment annotation
- CucumberHTMLReports.md — scenario.attach() for native Cucumber embedding
- Hooks.md — @After and @AfterStep hook execution order
- HeadlessTesting.md — Screenshot behavior in headless Chrome and Firefox
