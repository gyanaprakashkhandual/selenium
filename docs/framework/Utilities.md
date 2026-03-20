# Utilities and Helpers

## Overview

Utility classes in a Selenium framework are stateless, reusable collections of static methods that perform common operations not specific to any single page or test. They live in `src/main/java/utils/` and are available to page objects, step definitions, hooks, and data managers alike.

A well-organized utils package reduces duplication across the framework, ensures that common operations are implemented correctly in one place, and makes the codebase more readable by giving common patterns a clean API.

This document covers the six core utility categories in a professional framework: Waits, Screenshots, JavaScript, File, Date, and String utilities.

---

## WaitUtils

`WaitUtils` provides custom wait conditions that extend or wrap the standard `ExpectedConditions` class. It is designed for situations where built-in conditions are insufficient — AJAX loaders, React rendering delays, attribute changes, and staleness recovery.

```java
package com.company.framework.utils;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.ExpectedCondition;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.FluentWait;
import org.openqa.selenium.support.ui.WebDriverWait;

import java.time.Duration;
import java.util.List;
import java.util.function.Function;

public class WaitUtils {

    private static final Logger log = LogManager.getLogger(WaitUtils.class);
    private static final int DEFAULT_TIMEOUT = 10;
    private static final int DEFAULT_POLLING = 500;

    private WaitUtils() {}

    // -----------------------------------------------------------------------
    // FluentWait — polls at intervals, ignores specific exceptions
    // -----------------------------------------------------------------------

    public static WebElement fluentWaitForElement(WebDriver driver, By locator) {
        return fluentWaitForElement(driver, locator, DEFAULT_TIMEOUT, DEFAULT_POLLING);
    }

    public static WebElement fluentWaitForElement(
            WebDriver driver, By locator, int timeoutSec, int pollingMs) {

        FluentWait<WebDriver> wait = new FluentWait<>(driver)
            .withTimeout(Duration.ofSeconds(timeoutSec))
            .pollingEvery(Duration.ofMillis(pollingMs))
            .ignoring(NoSuchElementException.class)
            .ignoring(StaleElementReferenceException.class);

        return wait.until(d -> d.findElement(locator));
    }

    // -----------------------------------------------------------------------
    // Loader / spinner wait — waits for a loading indicator to disappear
    // -----------------------------------------------------------------------

    public static void waitForLoaderToDisappear(WebDriver driver, By loaderLocator) {
        waitForLoaderToDisappear(driver, loaderLocator, DEFAULT_TIMEOUT);
    }

    public static void waitForLoaderToDisappear(
            WebDriver driver, By loaderLocator, int timeoutSec) {

        WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(timeoutSec));
        try {
            // First wait for loader to appear (it may not have started yet)
            wait.until(ExpectedConditions.visibilityOfElementLocated(loaderLocator));
        } catch (TimeoutException e) {
            // Loader didn't appear — possibly already gone; that's fine
        }

        // Now wait for it to disappear
        wait.until(ExpectedConditions.invisibilityOfElementLocated(loaderLocator));
        log.debug("Loader disappeared: {}", loaderLocator);
    }

    // -----------------------------------------------------------------------
    // Attribute-based waits
    // -----------------------------------------------------------------------

    public static boolean waitForAttributeToContain(
            WebDriver driver, WebElement element, String attribute, String value) {

        WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(DEFAULT_TIMEOUT));
        return wait.until(
            ExpectedConditions.attributeContains(element, attribute, value)
        );
    }

    public static boolean waitForAttributeToEqual(
            WebDriver driver, By locator, String attribute, String expectedValue) {

        WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(DEFAULT_TIMEOUT));
        return wait.until(d -> {
            try {
                String actual = d.findElement(locator).getAttribute(attribute);
                return expectedValue.equals(actual);
            } catch (StaleElementReferenceException e) {
                return false;
            }
        });
    }

    // -----------------------------------------------------------------------
    // URL and title waits
    // -----------------------------------------------------------------------

    public static void waitForUrlToContain(WebDriver driver, String partialUrl) {
        new WebDriverWait(driver, Duration.ofSeconds(DEFAULT_TIMEOUT))
            .until(ExpectedConditions.urlContains(partialUrl));
    }

    public static void waitForUrlToMatch(WebDriver driver, String regex) {
        new WebDriverWait(driver, Duration.ofSeconds(DEFAULT_TIMEOUT))
            .until(ExpectedConditions.urlMatches(regex));
    }

    public static void waitForTitleContains(WebDriver driver, String titlePart) {
        new WebDriverWait(driver, Duration.ofSeconds(DEFAULT_TIMEOUT))
            .until(ExpectedConditions.titleContains(titlePart));
    }

    // -----------------------------------------------------------------------
    // Element count waits
    // -----------------------------------------------------------------------

    public static List<WebElement> waitForElementCount(
            WebDriver driver, By locator, int expectedCount) {

        WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(DEFAULT_TIMEOUT));
        return wait.until(d -> {
            List<WebElement> elements = d.findElements(locator);
            return elements.size() >= expectedCount ? elements : null;
        });
    }

    // -----------------------------------------------------------------------
    // Staleness recovery — retry finding an element if stale
    // -----------------------------------------------------------------------

    public static WebElement getElementWithStalenessRecovery(
            WebDriver driver, By locator, int maxRetries) {

        for (int attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                return driver.findElement(locator);
            } catch (StaleElementReferenceException e) {
                log.warn("Stale element on attempt {}/{} for locator: {}",
                    attempt, maxRetries, locator);
                if (attempt == maxRetries) throw e;
                try { Thread.sleep(500); } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                }
            }
        }
        throw new RuntimeException("Could not find fresh element after retries: " + locator);
    }

    // -----------------------------------------------------------------------
    // Page load complete check
    // -----------------------------------------------------------------------

    public static void waitForPageToLoad(WebDriver driver) {
        WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(30));
        wait.until(d -> ((JavascriptExecutor) d)
            .executeScript("return document.readyState")
            .equals("complete"));
    }
}
```

---

## ScreenshotUtils

`ScreenshotUtils` handles all screenshot operations: capturing, naming, saving to disk, and returning byte arrays for embedding in reports.

```java
package com.company.framework.utils;

import com.company.framework.config.ConfigReader;
import org.apache.commons.io.FileUtils;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;

import java.io.File;
import java.io.IOException;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class ScreenshotUtils {

    private static final Logger log = LogManager.getLogger(ScreenshotUtils.class);
    private static final String SCREENSHOT_DIR =
        ConfigReader.get("screenshots.dir", "target/screenshots");
    private static final DateTimeFormatter TIMESTAMP_FORMAT =
        DateTimeFormatter.ofPattern("yyyy-MM-dd_HH-mm-ss");

    private ScreenshotUtils() {}

    /**
     * Captures a full-page screenshot and saves it to disk.
     * Returns the absolute path of the saved file.
     */
    public static String takeScreenshot(WebDriver driver, String screenshotName) {
        String timestamp = LocalDateTime.now().format(TIMESTAMP_FORMAT);
        String sanitizedName = screenshotName.replaceAll("[^a-zA-Z0-9_-]", "_");
        String fileName = sanitizedName + "_" + timestamp + ".png";
        String filePath = SCREENSHOT_DIR + "/" + fileName;

        try {
            File source = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
            File destination = new File(filePath);
            FileUtils.copyFile(source, destination);
            log.info("Screenshot saved: {}", filePath);
            return filePath;
        } catch (IOException e) {
            log.error("Failed to save screenshot '{}': {}", fileName, e.getMessage());
            return null;
        }
    }

    /**
     * Captures a screenshot and returns it as a byte array.
     * Used for embedding into Cucumber/Extent/Allure reports.
     */
    public static byte[] takeScreenshotAsBytes(WebDriver driver) {
        try {
            return ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
        } catch (Exception e) {
            log.error("Failed to capture screenshot as bytes: {}", e.getMessage());
            return new byte[0];
        }
    }

    /**
     * Captures a screenshot of a specific WebElement only.
     */
    public static String takeElementScreenshot(WebElement element, String name) {
        String timestamp = LocalDateTime.now().format(TIMESTAMP_FORMAT);
        String fileName = name + "_element_" + timestamp + ".png";
        String filePath = SCREENSHOT_DIR + "/" + fileName;

        try {
            File source = element.getScreenshotAs(OutputType.FILE);
            File destination = new File(filePath);
            FileUtils.copyFile(source, destination);
            log.info("Element screenshot saved: {}", filePath);
            return filePath;
        } catch (IOException e) {
            log.error("Failed to save element screenshot: {}", e.getMessage());
            return null;
        }
    }

    /**
     * Ensures the screenshots directory exists. Call once at suite startup.
     */
    public static void ensureDirectoryExists() {
        File dir = new File(SCREENSHOT_DIR);
        if (!dir.exists()) {
            boolean created = dir.mkdirs();
            if (created) {
                log.info("Screenshots directory created: {}", SCREENSHOT_DIR);
            }
        }
    }
}
```

---

## JavaScriptUtils

`JavaScriptUtils` wraps the `JavascriptExecutor` with named, purposeful methods that are safer and more readable than raw `executeScript` calls scattered through page classes.

```java
package com.company.framework.utils;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.JavascriptExecutor;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;

public class JavaScriptUtils {

    private static final Logger log = LogManager.getLogger(JavaScriptUtils.class);

    private JavaScriptUtils() {}

    private static JavascriptExecutor js(WebDriver driver) {
        return (JavascriptExecutor) driver;
    }

    // -----------------------------------------------------------------------
    // Click utilities
    // -----------------------------------------------------------------------

    /** Clicks an element via JavaScript. Use when normal click is intercepted. */
    public static void click(WebDriver driver, WebElement element) {
        js(driver).executeScript("arguments[0].click();", element);
    }

    // -----------------------------------------------------------------------
    // Scroll utilities
    // -----------------------------------------------------------------------

    /** Scrolls the element into the visible viewport. */
    public static void scrollIntoView(WebDriver driver, WebElement element) {
        js(driver).executeScript("arguments[0].scrollIntoView({block: 'center'});", element);
    }

    /** Scrolls to the very top of the page. */
    public static void scrollToTop(WebDriver driver) {
        js(driver).executeScript("window.scrollTo(0, 0);");
    }

    /** Scrolls to the very bottom of the page. */
    public static void scrollToBottom(WebDriver driver) {
        js(driver).executeScript("window.scrollTo(0, document.body.scrollHeight);");
    }

    /** Scrolls by a specific pixel offset from current position. */
    public static void scrollBy(WebDriver driver, int x, int y) {
        js(driver).executeScript("window.scrollBy(arguments[0], arguments[1]);", x, y);
    }

    // -----------------------------------------------------------------------
    // Element value and attribute manipulation
    // -----------------------------------------------------------------------

    /** Sets the value of an input field directly, bypassing sendKeys. */
    public static void setInputValue(WebDriver driver, WebElement element, String value) {
        js(driver).executeScript("arguments[0].value = arguments[1];", element, value);
    }

    /** Removes the 'readonly' attribute from an input field. */
    public static void removeReadOnly(WebDriver driver, WebElement element) {
        js(driver).executeScript("arguments[0].removeAttribute('readonly');", element);
    }

    /** Removes the 'disabled' attribute from a form element. */
    public static void removeDisabled(WebDriver driver, WebElement element) {
        js(driver).executeScript("arguments[0].removeAttribute('disabled');", element);
    }

    /** Retrieves the innerHTML of an element. */
    public static String getInnerHTML(WebDriver driver, WebElement element) {
        return (String) js(driver).executeScript("return arguments[0].innerHTML;", element);
    }

    /** Retrieves the innerText of an element, including hidden text. */
    public static String getInnerText(WebDriver driver, WebElement element) {
        return (String) js(driver).executeScript("return arguments[0].innerText;", element);
    }

    // -----------------------------------------------------------------------
    // Page state
    // -----------------------------------------------------------------------

    /** Returns true when the page's document.readyState is 'complete'. */
    public static boolean isPageLoaded(WebDriver driver) {
        return js(driver).executeScript("return document.readyState")
            .equals("complete");
    }

    /** Returns the current page scroll Y position in pixels. */
    public static long getScrollPosition(WebDriver driver) {
        return (Long) js(driver).executeScript("return window.pageYOffset;");
    }

    // -----------------------------------------------------------------------
    // Highlight — useful for debugging
    // -----------------------------------------------------------------------

    /** Draws a colored border around an element. */
    public static void highlight(WebDriver driver, WebElement element) {
        js(driver).executeScript(
            "arguments[0].style.border='3px solid red'; " +
            "arguments[0].style.backgroundColor='yellow';",
            element
        );
    }

    /** Removes highlight styling from an element. */
    public static void removeHighlight(WebDriver driver, WebElement element) {
        js(driver).executeScript(
            "arguments[0].style.border=''; " +
            "arguments[0].style.backgroundColor='';",
            element
        );
    }
}
```

---

## FileUtils (Test Framework Specific)

The framework's own `FileUtils` handles reading, writing, and managing test-related files beyond what Apache Commons IO provides out of the box.

```java
package com.company.framework.utils;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.io.*;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;
import java.util.List;
import java.util.stream.Collectors;

public class FileUtils {

    private static final Logger log = LogManager.getLogger(FileUtils.class);

    private FileUtils() {}

    /** Reads the entire content of a classpath resource as a String. */
    public static String readResourceAsString(String resourcePath) {
        try (InputStream input = FileUtils.class
                .getClassLoader()
                .getResourceAsStream(resourcePath)) {

            if (input == null) {
                throw new RuntimeException("Resource not found: " + resourcePath);
            }

            return new String(input.readAllBytes(), StandardCharsets.UTF_8);

        } catch (IOException e) {
            throw new RuntimeException("Failed to read resource: " + resourcePath, e);
        }
    }

    /** Reads all lines from a classpath resource. */
    public static List<String> readResourceLines(String resourcePath) {
        return readResourceAsString(resourcePath)
            .lines()
            .collect(Collectors.toList());
    }

    /** Writes text content to a file, creating parent directories if needed. */
    public static void writeToFile(String filePath, String content) {
        try {
            Path path = Paths.get(filePath);
            Files.createDirectories(path.getParent());
            Files.writeString(path, content, StandardCharsets.UTF_8);
            log.info("File written: {}", filePath);
        } catch (IOException e) {
            log.error("Failed to write file {}: {}", filePath, e.getMessage());
            throw new RuntimeException("Cannot write file: " + filePath, e);
        }
    }

    /** Deletes a file if it exists. Returns true if deleted. */
    public static boolean deleteFile(String filePath) {
        try {
            return Files.deleteIfExists(Paths.get(filePath));
        } catch (IOException e) {
            log.error("Failed to delete file {}: {}", filePath, e.getMessage());
            return false;
        }
    }

    /** Creates a directory and all parent directories if they do not exist. */
    public static void createDirectory(String dirPath) {
        try {
            Files.createDirectories(Paths.get(dirPath));
        } catch (IOException e) {
            throw new RuntimeException("Cannot create directory: " + dirPath, e);
        }
    }

    /** Returns true if a file exists at the given path. */
    public static boolean fileExists(String filePath) {
        return Files.exists(Paths.get(filePath));
    }

    /** Returns the file size in bytes. */
    public static long getFileSize(String filePath) {
        try {
            return Files.size(Paths.get(filePath));
        } catch (IOException e) {
            log.error("Cannot get file size: {}", filePath);
            return -1L;
        }
    }
}
```

---

## DateUtils

`DateUtils` handles formatting, parsing, and comparing dates and times that appear on the UI.

```java
package com.company.framework.utils;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.time.format.DateTimeParseException;
import java.time.temporal.ChronoUnit;
import java.util.Arrays;
import java.util.List;

public class DateUtils {

    private static final Logger log = LogManager.getLogger(DateUtils.class);

    // Common date formats encountered in web applications
    private static final List<DateTimeFormatter> KNOWN_FORMATS = Arrays.asList(
        DateTimeFormatter.ofPattern("MM/dd/yyyy"),
        DateTimeFormatter.ofPattern("dd/MM/yyyy"),
        DateTimeFormatter.ofPattern("yyyy-MM-dd"),
        DateTimeFormatter.ofPattern("dd-MM-yyyy"),
        DateTimeFormatter.ofPattern("MMMM d, yyyy"),
        DateTimeFormatter.ofPattern("MMM d, yyyy"),
        DateTimeFormatter.ofPattern("d MMM yyyy")
    );

    private DateUtils() {}

    /** Formats a LocalDate to the given pattern. */
    public static String format(LocalDate date, String pattern) {
        return date.format(DateTimeFormatter.ofPattern(pattern));
    }

    /** Returns today's date formatted with the given pattern. */
    public static String today(String pattern) {
        return format(LocalDate.now(), pattern);
    }

    /** Returns a date N days in the future. */
    public static String futureDate(int daysFromNow, String pattern) {
        return format(LocalDate.now().plusDays(daysFromNow), pattern);
    }

    /** Returns a date N days in the past. */
    public static String pastDate(int daysAgo, String pattern) {
        return format(LocalDate.now().minusDays(daysAgo), pattern);
    }

    /**
     * Attempts to parse a date string using a list of known formats.
     * Useful for asserting date values from the UI without hardcoding the format.
     */
    public static LocalDate parseFlexible(String dateString) {
        for (DateTimeFormatter formatter : KNOWN_FORMATS) {
            try {
                return LocalDate.parse(dateString.trim(), formatter);
            } catch (DateTimeParseException e) {
                // Try the next format
            }
        }
        throw new RuntimeException("Cannot parse date: '" + dateString
            + "'. Known formats: " + KNOWN_FORMATS);
    }

    /** Returns the number of days between two dates (positive if end is after start). */
    public static long daysBetween(LocalDate start, LocalDate end) {
        return ChronoUnit.DAYS.between(start, end);
    }

    /** Returns true if the given date string represents today's date. */
    public static boolean isToday(String dateString) {
        try {
            LocalDate parsed = parseFlexible(dateString);
            return parsed.equals(LocalDate.now());
        } catch (RuntimeException e) {
            log.warn("Cannot determine if '{}' is today: {}", dateString, e.getMessage());
            return false;
        }
    }

    /** Returns a timestamp string for use in unique file names and identifiers. */
    public static String timestamp() {
        return LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMdd_HHmmss"));
    }
}
```

---

## StringUtils (Framework-Specific)

The framework's `StringUtils` supplements Apache Commons Lang with test-specific string operations.

```java
package com.company.framework.utils;

public class StringUtils {

    private StringUtils() {}

    /** Removes all whitespace and normalizes internal spaces. */
    public static String normalize(String input) {
        if (input == null) return "";
        return input.trim().replaceAll("\\s+", " ");
    }

    /** Returns true if two strings are equal after normalization. */
    public static boolean equalsNormalized(String a, String b) {
        return normalize(a).equalsIgnoreCase(normalize(b));
    }

    /** Extracts the numeric portion from a string like "$29.99" or "29.99 USD". */
    public static double extractNumeric(String value) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("Cannot extract number from blank string");
        }
        String numeric = value.replaceAll("[^0-9.]", "");
        return Double.parseDouble(numeric);
    }

    /** Converts a display label like "First Name" to a field key like "first_name". */
    public static String toFieldKey(String label) {
        return label.trim().toLowerCase().replace(" ", "_");
    }

    /** Truncates a string to maxLength, appending "..." if truncated. */
    public static String truncate(String input, int maxLength) {
        if (input == null || input.length() <= maxLength) return input;
        return input.substring(0, maxLength - 3) + "...";
    }

    /** Returns true if the string contains only digits. */
    public static boolean isNumeric(String value) {
        return value != null && value.matches("\\d+");
    }

    /** Masks sensitive values for logging: "SecretPass" → "Se******ss". */
    public static String maskSensitive(String value) {
        if (value == null || value.length() <= 4) return "****";
        return value.substring(0, 2)
            + "*".repeat(value.length() - 4)
            + value.substring(value.length() - 2);
    }
}
```

---

## Utility Class Design Rules

**All methods must be `public static`.** Utility classes have no state. They are never instantiated.

**Constructors must be private and throw `IllegalStateException`.** This prevents accidental instantiation:

```java
private WaitUtils() {
    throw new IllegalStateException("WaitUtils is a static utility class");
}
```

**Each utility class covers one concern.** Do not create a single `Utils.java` with 50 methods of mixed responsibility. One class per domain: waits, screenshots, JavaScript, files, dates, strings.

**Return values, do not log failures silently.** Utilities should either succeed and return a value, or throw a runtime exception with a clear message. Never silently return null without logging.

**Do not depend on DriverManager or ConfigReader inside utility methods.** Accept WebDriver and configuration values as method parameters. This keeps utilities testable and decoupled.

---

## Summary

Utility classes are the shared toolbox of the framework — stateless, focused, and available everywhere. `WaitUtils` provides advanced custom waits beyond `ExpectedConditions`. `ScreenshotUtils` handles screenshot capture and storage. `JavaScriptUtils` wraps raw JS execution into named, purposeful methods. `FileUtils` manages classpath resources and file operations. `DateUtils` handles flexible date parsing and formatting. `StringUtils` performs test-specific string operations. Together they eliminate duplication, enforce correct patterns, and make the codebase consistently readable.

---

## Related Topics

- BasePage.md — Where WaitUtils and JavaScriptUtils are used in page interactions
- Hooks.md — Where ScreenshotUtils is called on scenario failure
- TestDataManagement.md — Where FileUtils is used to load data files
- Log4j.md — All utility classes use the logger from Log4j2
- ProjectStructure.md — utils/ package location in the framework
