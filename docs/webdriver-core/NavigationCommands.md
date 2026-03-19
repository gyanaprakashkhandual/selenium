# Navigation Commands

## Overview

Navigation commands control how the browser moves between pages, manages browser history, and handles URL transitions. While seemingly simple, navigation has important nuances around page load behavior, URL verification, redirect handling, and synchronization. This document covers every navigation method with real-world patterns for building reliable test flows.

---

## Core Navigation Methods

### driver.get() — Primary Navigation

```java
// Navigate to a URL and wait for full page load
driver.get("https://example.com");
driver.get("https://example.com/login");
driver.get("https://staging.app.com/dashboard");

// With query parameters
driver.get("https://example.com/search?q=selenium&page=1");

// With URL built programmatically
String baseUrl = ConfigReader.get("base.url");
driver.get(baseUrl + "/login");

// Full URL construction
String url = String.format("%s/products/%s/details", baseUrl, productId);
driver.get(url);
```

**Behavior:** `driver.get()` blocks until the browser reports `document.readyState == "complete"`. It respects the **Page Load Timeout** set via `driver.manage().timeouts().pageLoadTimeout()`.

### driver.navigate() Methods

```java
// Navigate to URL (identical to driver.get() in Selenium 4)
driver.navigate().to("https://example.com");

// Using a URL object (type-safe URL construction)
import java.net.URL;
driver.navigate().to(new URL("https://example.com/login"));

// Go back to previous page (browser history)
driver.navigate().back();

// Go forward in browser history
driver.navigate().forward();

// Refresh / reload the current page
driver.navigate().refresh();
```

---

## Navigation Patterns with Waits

Never assume a page is ready immediately after navigation. Always wait for a meaningful condition:

```java
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;
import java.time.Duration;

WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(15));

// ===== Pattern 1: Wait for URL to match =====
driver.findElement(By.id("login-btn")).click();
wait.until(ExpectedConditions.urlContains("/dashboard"));
System.out.println("Successfully navigated to dashboard");

// Wait for exact URL
wait.until(ExpectedConditions.urlToBe("https://app.example.com/dashboard"));

// Wait for URL matching a regex
wait.until(ExpectedConditions.urlMatches("https://app\\.example\\.com/orders/\\d+"));

// ===== Pattern 2: Wait for page title =====
driver.get("https://example.com/about");
wait.until(ExpectedConditions.titleIs("About Us | Example Corp"));
wait.until(ExpectedConditions.titleContains("About Us"));

// ===== Pattern 3: Wait for a landmark element on the new page =====
driver.findElement(By.id("go-to-profile")).click();
wait.until(ExpectedConditions.visibilityOfElementLocated(
    By.id("profile-page-header")
));

// ===== Pattern 4: Wait for page load via JavaScript readyState =====
wait.until(d -> ((JavascriptExecutor) d)
    .executeScript("return document.readyState")
    .equals("complete")
);

// ===== Pattern 5: Wait for Angular/React to finish rendering =====
// For Angular apps
wait.until(d -> {
    try {
        return (Boolean) ((JavascriptExecutor) d).executeScript(
            "return window.getAllAngularTestabilities()" +
            ".every(t => t.isStable())"
        );
    } catch (Exception e) {
        return false;  // Angular not loaded yet
    }
});
```

---

## URL Verification and Assertions

URL verification is fundamental in navigation testing:

```java
// Get the full current URL
String currentUrl = driver.getCurrentUrl();
System.out.println("Current URL: " + currentUrl);

// TestNG assertions on URL
import org.testng.Assert;

Assert.assertEquals(driver.getCurrentUrl(), "https://app.example.com/dashboard");
Assert.assertTrue(driver.getCurrentUrl().contains("/dashboard"));

// AssertJ — more readable
import static org.assertj.core.api.Assertions.assertThat;
assertThat(driver.getCurrentUrl()).contains("/dashboard");
assertThat(driver.getCurrentUrl()).startsWith("https://");
assertThat(driver.getCurrentUrl()).doesNotContain("/login");

// URL utility helper
public class UrlHelper {

    public static String getCurrentPath(WebDriver driver) {
        String url = driver.getCurrentUrl();
        try {
            return new java.net.URL(url).getPath();
        } catch (Exception e) {
            return url;
        }
    }

    public static Map<String, String> getQueryParams(WebDriver driver) {
        String url = driver.getCurrentUrl();
        Map<String, String> params = new LinkedHashMap<>();
        try {
            String query = new java.net.URL(url).getQuery();
            if (query != null) {
                for (String param : query.split("&")) {
                    String[] pair = param.split("=", 2);
                    if (pair.length == 2) {
                        params.put(pair[0], java.net.URLDecoder.decode(pair[1], "UTF-8"));
                    }
                }
            }
        } catch (Exception e) {
            // handle
        }
        return params;
    }
}

// Usage
String path = UrlHelper.getCurrentPath(driver);   // "/dashboard"
Map<String, String> params = UrlHelper.getQueryParams(driver); // {q=selenium, page=1}
```

---

## Handling Redirects

Many pages redirect to another URL. Selenium handles them transparently — `driver.get()` follows redirects automatically. However, you may need to verify the final destination:

```java
// Navigate to a URL that redirects
driver.get("https://example.com/old-url");

// Wait for redirect to complete and verify final URL
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// Option 1: Wait for URL to change from the original
String originalUrl = "https://example.com/old-url";
wait.until(ExpectedConditions.not(ExpectedConditions.urlToBe(originalUrl)));

// Option 2: Wait for the expected destination URL
wait.until(ExpectedConditions.urlContains("/new-url"));

System.out.println("Final URL after redirect: " + driver.getCurrentUrl());

// Verify redirect count (not built into Selenium — use CDP or proxy for this)
```

---

## Navigation Helper Class

Centralizing navigation logic in a reusable class keeps your tests clean:

```java
package com.yourcompany.automation.utils;

import org.openqa.selenium.JavascriptExecutor;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

import java.time.Duration;

public class NavigationHelper {

    private final WebDriver driver;
    private final WebDriverWait wait;
    private final String baseUrl;

    public NavigationHelper(WebDriver driver, String baseUrl) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(15));
        this.baseUrl = baseUrl;
    }

    /**
     * Navigate to a full URL and wait for page load.
     */
    public void navigateTo(String url) {
        driver.get(url);
        waitForPageLoad();
    }

    /**
     * Navigate to a relative path (appended to base URL).
     */
    public void navigateToPath(String path) {
        String fullUrl = path.startsWith("http") ? path : baseUrl + path;
        driver.get(fullUrl);
        waitForPageLoad();
    }

    /**
     * Navigate back and wait for the previous page.
     */
    public void goBack() {
        String urlBeforeBack = driver.getCurrentUrl();
        driver.navigate().back();
        wait.until(ExpectedConditions.not(
            ExpectedConditions.urlToBe(urlBeforeBack)
        ));
        waitForPageLoad();
    }

    /**
     * Navigate forward.
     */
    public void goForward() {
        driver.navigate().forward();
        waitForPageLoad();
    }

    /**
     * Refresh the current page and wait for it to reload.
     */
    public void refresh() {
        driver.navigate().refresh();
        waitForPageLoad();
    }

    /**
     * Wait for full page load via document.readyState.
     */
    public void waitForPageLoad() {
        wait.until(d ->
            ((JavascriptExecutor) d)
                .executeScript("return document.readyState")
                .equals("complete")
        );
    }

    /**
     * Navigate and wait for a specific URL pattern.
     */
    public void navigateAndWaitForUrl(String url, String expectedUrlFragment) {
        driver.get(url);
        wait.until(ExpectedConditions.urlContains(expectedUrlFragment));
    }

    /**
     * Open URL in a new tab, switch to it, and return the previous window handle.
     */
    public String openInNewTab(String url) {
        String originalHandle = driver.getWindowHandle();
        driver.switchTo().newWindow(org.openqa.selenium.WindowType.TAB);
        driver.get(url);
        waitForPageLoad();
        return originalHandle;  // Caller can use this to switch back
    }

    /**
     * Get the current page path (without domain).
     */
    public String getCurrentPath() {
        String url = driver.getCurrentUrl();
        try {
            return new java.net.URL(url).getPath();
        } catch (Exception e) {
            return url;
        }
    }

    /**
     * Verify current URL contains the expected fragment.
     */
    public boolean isOnPage(String urlFragment) {
        return driver.getCurrentUrl().contains(urlFragment);
    }

    /**
     * Assert we are on the expected page (throws if not).
     */
    public void assertOnPage(String urlFragment) {
        if (!isOnPage(urlFragment)) {
            throw new AssertionError(
                "Expected URL to contain '" + urlFragment +
                "' but current URL is: " + driver.getCurrentUrl()
            );
        }
    }
}
```

---

## Navigation in Page Object Model

In a POM-based framework, page navigation returns the next page object — this is the **Fluent Page Object** pattern:

```java
// LoginPage.java
public class LoginPage {

    private WebDriver driver;
    private WebDriverWait wait;

    @FindBy(id = "username")
    private WebElement usernameInput;

    @FindBy(id = "password")
    private WebElement passwordInput;

    @FindBy(id = "login-btn")
    private WebElement loginButton;

    public LoginPage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
        PageFactory.initElements(driver, this);
    }

    /**
     * Navigate to the login page.
     */
    public LoginPage navigate() {
        driver.get(ConfigReader.get("base.url") + "/login");
        wait.until(ExpectedConditions.visibilityOf(usernameInput));
        return this;
    }

    /**
     * Perform login and return the DashboardPage on success.
     */
    public DashboardPage loginAs(String username, String password) {
        usernameInput.clear();
        usernameInput.sendKeys(username);
        passwordInput.clear();
        passwordInput.sendKeys(password);
        loginButton.click();
        // Wait for navigation to dashboard
        wait.until(ExpectedConditions.urlContains("/dashboard"));
        return new DashboardPage(driver);
    }

    /**
     * Attempt login and return this LoginPage (for failure scenarios).
     */
    public LoginPage attemptLoginAs(String username, String password) {
        usernameInput.clear();
        usernameInput.sendKeys(username);
        passwordInput.clear();
        passwordInput.sendKeys(password);
        loginButton.click();
        return this;
    }
}

// Test code — readable fluent chain
LoginPage loginPage = new LoginPage(driver).navigate();
DashboardPage dashboard = loginPage.loginAs("admin", "password");
```

---

## Browser History Navigation

### Back and Forward with State Verification

```java
// Test that back navigation works correctly
driver.get("https://example.com/page1");
String page1Title = driver.getTitle();
String page1Url   = driver.getCurrentUrl();

driver.get("https://example.com/page2");
String page2Url = driver.getCurrentUrl();

// Go back
driver.navigate().back();
wait.until(ExpectedConditions.urlToBe(page1Url));
Assert.assertEquals(driver.getTitle(), page1Title);

// Go forward
driver.navigate().forward();
wait.until(ExpectedConditions.urlToBe(page2Url));

// Test browser back button via JS (alternative)
((JavascriptExecutor) driver).executeScript("window.history.back();");
wait.until(ExpectedConditions.urlContains("page1"));
```

### Multi-Step Navigation Flow Test

```java
@Test
public void testCheckoutNavigationFlow() {
    WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

    // Step 1: Start on home page
    driver.get(BASE_URL + "/");
    Assert.assertTrue(driver.getTitle().contains("Home"));

    // Step 2: Navigate to products
    driver.findElement(By.linkText("Products")).click();
    wait.until(ExpectedConditions.urlContains("/products"));

    // Step 3: Go to a product detail page
    driver.findElement(By.cssSelector("[data-testid='product-card']:first-child")).click();
    wait.until(ExpectedConditions.urlMatches(".*\\/products\\/\\d+"));
    String productUrl = driver.getCurrentUrl();

    // Step 4: Add to cart
    driver.findElement(By.id("add-to-cart")).click();
    wait.until(ExpectedConditions.urlContains("/cart"));

    // Step 5: Go back to product page
    driver.navigate().back();
    wait.until(ExpectedConditions.urlToBe(productUrl));
    Assert.assertTrue(driver.getTitle().contains("Product"));

    // Step 6: Proceed to checkout
    driver.findElement(By.id("buy-now")).click();
    wait.until(ExpectedConditions.urlContains("/checkout"));
    Assert.assertTrue(driver.getCurrentUrl().contains("/checkout"));
}
```

---

## Deep Linking (Direct Navigation to Protected Pages)

Deep linking tests navigate directly to a URL that requires authentication, testing that:

1. Unauthenticated users are redirected to login
2. Authenticated users land on the correct page

```java
@Test
public void testDeepLinkRedirectsUnauthenticatedUser() {
    WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

    // Attempt to access protected page without login
    driver.get(BASE_URL + "/admin/users");

    // Should redirect to login
    wait.until(ExpectedConditions.urlContains("/login"));
    Assert.assertTrue(driver.getCurrentUrl().contains("/login"),
        "Expected redirect to login, but got: " + driver.getCurrentUrl());
}

@Test
public void testDeepLinkAfterLogin() {
    WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

    // Login first
    LoginPage loginPage = new LoginPage(driver).navigate();
    loginPage.loginAs("admin", "password");
    wait.until(ExpectedConditions.urlContains("/dashboard"));

    // Navigate directly to a deep page
    driver.get(BASE_URL + "/admin/users/settings");
    wait.until(ExpectedConditions.urlContains("/admin/users/settings"));

    // Verify we landed on the correct page
    Assert.assertTrue(
        driver.findElement(By.id("page-title")).getText().contains("User Settings")
    );
}
```

---

## Handling Page Load Timeout

```java
import org.openqa.selenium.TimeoutException;

// Set page load timeout
driver.manage().timeouts().pageLoadTimeout(Duration.ofSeconds(30));

// Handle pages that take too long to load
try {
    driver.get("https://slow-loading-page.com");
} catch (TimeoutException e) {
    System.err.println("Page load timed out after 30 seconds");
    // Option 1: Stop the page load and work with what's loaded
    ((JavascriptExecutor) driver).executeScript("window.stop();");
    // Option 2: Take a screenshot and fail the test gracefully
    TakesScreenshot ts = (TakesScreenshot) driver;
    File screenshot = ts.getScreenshotAs(OutputType.FILE);
    FileUtils.copyFile(screenshot, new File("screenshots/timeout_" + System.currentTimeMillis() + ".png"));
    throw e; // Re-throw to fail the test
}
```

---

## Navigation in Cucumber Step Definitions

```java
package com.yourcompany.automation.steps;

import io.cucumber.java.en.Given;
import io.cucumber.java.en.When;
import io.cucumber.java.en.Then;

public class NavigationSteps {

    private final WebDriver driver = DriverManager.getDriver();
    private final WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

    @Given("the user is on the login page")
    public void userIsOnLoginPage() {
        driver.get(ConfigReader.get("base.url") + "/login");
        wait.until(ExpectedConditions.titleContains("Login"));
    }

    @Given("the user is on the {string} page")
    public void userIsOnPage(String pageName) {
        String url = ConfigReader.get("base.url") + getPathForPage(pageName);
        driver.get(url);
        wait.until(d -> ((JavascriptExecutor) d)
            .executeScript("return document.readyState").equals("complete"));
    }

    @When("the user navigates back")
    public void userNavigatesBack() {
        String urlBeforeBack = driver.getCurrentUrl();
        driver.navigate().back();
        wait.until(ExpectedConditions.not(ExpectedConditions.urlToBe(urlBeforeBack)));
    }

    @Then("the user should be on the {string} page")
    public void userShouldBeOnPage(String urlFragment) {
        wait.until(ExpectedConditions.urlContains(urlFragment));
        Assert.assertTrue(
            driver.getCurrentUrl().contains(urlFragment),
            "Expected URL to contain '" + urlFragment + "' but was: " + driver.getCurrentUrl()
        );
    }

    private String getPathForPage(String pageName) {
        return switch (pageName.toLowerCase()) {
            case "login"     -> "/login";
            case "dashboard" -> "/dashboard";
            case "products"  -> "/products";
            case "cart"      -> "/cart";
            case "checkout"  -> "/checkout";
            default          -> "/" + pageName.toLowerCase();
        };
    }
}
```

---

## Summary

Navigation commands are the connective tissue of any end-to-end test. Always pair navigation with explicit waits — wait for URL changes, page titles, or landmark elements rather than relying on implicit timing. Centralize navigation logic in a `NavigationHelper` or in your Page Objects using the fluent pattern, where each navigation action returns the next page object. This keeps test code readable and the navigation logic reusable.
