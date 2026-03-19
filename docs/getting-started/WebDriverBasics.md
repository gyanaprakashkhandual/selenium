# WebDriver Basics

## Overview

The WebDriver API is the heart of every Selenium test. It provides the methods you use to control the browser, navigate between pages, find elements, read page information, and manage the browser state. This document is a comprehensive reference of every essential WebDriver method — with practical examples, real-world usage patterns, and the gotchas that trap beginners.

---

## The WebDriver Interface

`WebDriver` is a Java interface. The concrete classes (`ChromeDriver`, `FirefoxDriver`, `EdgeDriver`) all implement this interface, which means your test code should always declare the type as `WebDriver`, not `ChromeDriver`:

```java
// CORRECT — interface type; easy to swap browsers
WebDriver driver = new ChromeDriver();

// AVOID — concrete type; locks you to Chrome
ChromeDriver driver = new ChromeDriver();
```

This is the **Liskov Substitution Principle** in action — it lets you change `new ChromeDriver()` to `new FirefoxDriver()` in one place (your DriverFactory) without touching any test code.

---

## Browser and Navigation Commands

### Opening a URL

```java
// Navigate to a URL — waits for page load (document.readyState == 'complete')
driver.get("https://example.com");

// Alternative — identical behavior in Selenium 4
driver.navigate().to("https://example.com");
// driver.navigate().to(new URL("https://example.com")); // URL object version
```

> **`driver.get()` vs `driver.navigate().to()`**
> In Selenium 4, both do the same thing. Use `driver.get()` for initial page loads; use `driver.navigate()` when you need browser history methods (back, forward).

### Navigation History

```java
// Go back in browser history
driver.navigate().back();

// Go forward in browser history
driver.navigate().forward();

// Reload the current page (equivalent to pressing F5)
driver.navigate().refresh();
```

### Getting Page Information

```java
// Get the current page URL
String currentUrl = driver.getCurrentUrl();
System.out.println("Current URL: " + currentUrl);

// Get the page title (content of <title> tag)
String pageTitle = driver.getTitle();
System.out.println("Page Title: " + pageTitle);

// Get the entire page source HTML
String pageSource = driver.getPageSource();
System.out.println("Source length: " + pageSource.length());
```

---

## Browser Window Management

### Maximize and Resize

```java
// Maximize browser window
driver.manage().window().maximize();

// Set a specific window size (useful for responsive testing)
driver.manage().window().setSize(new Dimension(1920, 1080));

// Get current window size
Dimension size = driver.manage().window().getSize();
System.out.println("Width: " + size.width + ", Height: " + size.height);

// Get window position (top-left corner)
Point position = driver.manage().window().getPosition();
System.out.println("X: " + position.x + ", Y: " + position.y);

// Move window to a specific position
driver.manage().window().setPosition(new Point(100, 100));

// Fullscreen (like pressing F11)
driver.manage().window().fullscreen();

// Minimize (Selenium 4)
driver.manage().window().minimize();
```

### Working with Multiple Windows and Tabs

In Selenium, both browser windows and tabs are referred to by a **window handle** — a unique string identifier for each.

```java
// Get the current window handle
String originalWindow = driver.getWindowHandle();
System.out.println("Original window handle: " + originalWindow);

// Get all open window handles
Set<String> allWindows = driver.getWindowHandles();
System.out.println("Total open windows: " + allWindows.size());

// Click a link that opens a new tab/window
driver.findElement(By.linkText("Open New Tab")).click();

// Switch to the new window/tab
for (String handle : driver.getWindowHandles()) {
    if (!handle.equals(originalWindow)) {
        driver.switchTo().window(handle);
        break;
    }
}
System.out.println("Switched to: " + driver.getTitle());

// Do work in the new window...
String newWindowTitle = driver.getTitle();

// Switch back to original window
driver.switchTo().window(originalWindow);

// Open a new blank tab programmatically (Selenium 4)
driver.switchTo().newWindow(WindowType.TAB);
driver.get("https://example.com/new-tab-url");

// Open a new browser window programmatically (Selenium 4)
driver.switchTo().newWindow(WindowType.WINDOW);
driver.get("https://example.com/new-window-url");

// Close the current window/tab (does NOT quit the session)
driver.close();

// Switch back after closing
driver.switchTo().window(originalWindow);
```

---

## Finding Elements

The `findElement()` and `findElements()` methods are the core of element interaction.

### findElement() — Single Element

```java
// Returns the first matching element; throws NoSuchElementException if not found
WebElement element = driver.findElement(By.id("username"));
```

### findElements() — Multiple Elements

```java
// Returns a List of all matching elements; returns EMPTY LIST (not exception) if none found
List<WebElement> items = driver.findElements(By.cssSelector(".menu-item"));
System.out.println("Found " + items.size() + " menu items");

// Use size check to verify presence without exception
boolean isPresent = !driver.findElements(By.id("error-message")).isEmpty();
```

> **Best Practice:** Use `findElements()` when checking if an element **might not** exist. It returns an empty list instead of throwing an exception, making null-safe checks cleaner.

### Locator Strategies Overview

Full locator documentation is in the **Locator Strategies** chapter. Quick reference:

```java
// By ID — fastest, most reliable
driver.findElement(By.id("submitBtn"));

// By Name
driver.findElement(By.name("username"));

// By Class Name — single class only
driver.findElement(By.className("btn-primary"));

// By Tag Name
driver.findElement(By.tagName("h1"));

// By Link Text — exact match
driver.findElement(By.linkText("Sign In"));

// By Partial Link Text
driver.findElement(By.partialLinkText("Sign"));

// By CSS Selector — flexible and fast
driver.findElement(By.cssSelector("#loginForm input[type='email']"));

// By XPath — most powerful, use when CSS can't reach the element
driver.findElement(By.xpath("//button[@data-testid='submit']"));

// Relative Locators (Selenium 4)
WebElement passwordField = driver.findElement(
    RelativeLocator.with(By.tagName("input")).below(By.id("username"))
);
```

---

## WebElement Interactions

Once you have a `WebElement` reference, these are the actions you can perform:

### Input and Interaction

```java
WebElement inputField = driver.findElement(By.id("email"));

// Clear any existing content first
inputField.clear();

// Type text
inputField.sendKeys("user@example.com");

// Simulate keyboard keys using Keys enum
inputField.sendKeys(Keys.TAB);          // Press Tab
inputField.sendKeys(Keys.ENTER);        // Press Enter
inputField.sendKeys(Keys.CONTROL + "a"); // Ctrl+A (select all)
inputField.sendKeys(Keys.BACK_SPACE);   // Backspace

// Submit a form (calls the form's submit action)
inputField.submit();

// Click an element
driver.findElement(By.id("submitBtn")).click();
```

### Reading Element Properties

```java
WebElement element = driver.findElement(By.id("status-badge"));

// Get visible text content
String text = element.getText();

// Get an HTML attribute value
String href = driver.findElement(By.id("home-link")).getAttribute("href");
String inputValue = driver.findElement(By.id("email")).getAttribute("value");
String placeholder = driver.findElement(By.id("search")).getAttribute("placeholder");
String isDisabled = driver.findElement(By.id("btn")).getAttribute("disabled"); // null if not disabled

// Get a CSS property value
String color = element.getCssValue("color");
String fontSize = element.getCssValue("font-size");
String display = element.getCssValue("display");

// Get element dimensions and position
Dimension size = element.getSize();
System.out.println("Width: " + size.width + ", Height: " + size.height);

Point location = element.getLocation();
System.out.println("X: " + location.x + ", Y: " + location.y);

Rectangle rect = element.getRect();
System.out.println("Full rect: " + rect);
```

### State Checks

```java
WebElement checkbox = driver.findElement(By.id("agreeCheckbox"));
WebElement submitBtn = driver.findElement(By.id("submit"));
WebElement hiddenDiv = driver.findElement(By.id("hidden-section"));

// Is the element displayed in the DOM and visible to the user?
boolean isVisible = checkbox.isDisplayed();

// Is the element enabled (not disabled)?
boolean isEnabled = submitBtn.isEnabled();

// Is the element selected (for checkboxes, radio buttons, <option>)?
boolean isChecked = checkbox.isSelected();
```

---

## Timeout Management

### Types of Timeouts

```java
// Page Load Timeout — max time to wait for page load after driver.get()
driver.manage().timeouts().pageLoadTimeout(Duration.ofSeconds(30));

// Script Timeout — max time to wait for async JavaScript execution
driver.manage().timeouts().scriptTimeout(Duration.ofSeconds(30));

// Implicit Wait — global wait before throwing NoSuchElementException
// WARNING: Use with extreme caution — see note below
driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10));
```

> **CRITICAL: Never use Implicit Wait in a professional framework.**
>
> Implicit wait applies globally to every `findElement()` call. It interacts badly with explicit waits (causing double waits), slows down tests on expected-absent elements, and makes flakiness harder to diagnose. The industry consensus is: **set implicit wait to 0 and use only explicit waits.**
>
> ```java
> // Set this once in your setup and never use implicitlyWait elsewhere
> driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(0));
> ```

---

## Cookie Management

```java
// Get all cookies for the current domain
Set<Cookie> cookies = driver.manage().getCookies();
for (Cookie cookie : cookies) {
    System.out.println(cookie.getName() + " = " + cookie.getValue());
}

// Get a specific cookie by name
Cookie sessionCookie = driver.manage().getCookieNamed("JSESSIONID");
if (sessionCookie != null) {
    System.out.println("Session ID: " + sessionCookie.getValue());
}

// Add a cookie
Cookie newCookie = new Cookie("theme", "dark");
driver.manage().addCookie(newCookie);

// Add a cookie with full details
Cookie authCookie = new Cookie.Builder("auth_token", "eyJhbGciOiJSUzI1NiJ9...")
    .domain(".example.com")
    .path("/")
    .isHttpOnly(true)
    .isSecure(true)
    .build();
driver.manage().addCookie(authCookie);

// Delete a specific cookie
driver.manage().deleteCookieNamed("session_id");
driver.manage().deleteCookie(sessionCookie);

// Delete all cookies for the current domain
driver.manage().deleteAllCookies();
```

### Using Cookies for Login Bypass

One of the most powerful cookie use cases in automation: bypassing the UI login form:

```java
// 1. First login via UI once to get the session cookie
driver.get("https://app.example.com/login");
driver.findElement(By.id("username")).sendKeys("admin");
driver.findElement(By.id("password")).sendKeys("password");
driver.findElement(By.id("submit")).click();

// 2. Store the session cookie
Cookie sessionCookie = driver.manage().getCookieNamed("session_token");

// 3. In subsequent tests — inject the cookie and navigate directly
driver.get("https://app.example.com"); // Must be on same domain first
driver.manage().addCookie(sessionCookie);
driver.navigate().refresh(); // Apply the cookie
// Now directly navigate to authenticated pages
driver.get("https://app.example.com/dashboard");
```

---

## Screenshots

```java
import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;
import java.io.File;
import org.apache.commons.io.FileUtils;

// Take a full page screenshot
File screenshot = ((TakesScreenshot) driver).getScreenshotAs(OutputType.FILE);
FileUtils.copyFile(screenshot, new File("screenshots/test_failure.png"));

// Get as byte array (for attaching to reports)
byte[] screenshotBytes = ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);

// Get as Base64 string (for embedding in HTML reports)
String screenshotBase64 = ((TakesScreenshot) driver).getScreenshotAs(OutputType.BASE64);

// Screenshot of a specific element (Selenium 4)
WebElement element = driver.findElement(By.id("error-message"));
File elementScreenshot = element.getScreenshotAs(OutputType.FILE);
FileUtils.copyFile(elementScreenshot, new File("screenshots/error_element.png"));
```

---

## Closing and Quitting

```java
// Close the current window/tab (keeps session alive if other windows exist)
driver.close();

// Quit — closes ALL windows and terminates the WebDriver session
// Always call this in teardown
driver.quit();
```

### Proper Teardown Pattern

```java
@AfterMethod(alwaysRun = true)
public void tearDown() {
    if (driver != null) {
        try {
            driver.quit();
        } catch (Exception e) {
            // Log but don't fail the test on teardown exception
            System.err.println("Error during driver quit: " + e.getMessage());
        }
    }
}
```

---

## Complete WebDriver Method Reference

| Category        | Method                                        | Description                    |
| --------------- | --------------------------------------------- | ------------------------------ |
| **Navigation**  | `driver.get(url)`                             | Navigate to URL, wait for load |
|                 | `driver.navigate().to(url)`                   | Same as get()                  |
|                 | `driver.navigate().back()`                    | Browser back                   |
|                 | `driver.navigate().forward()`                 | Browser forward                |
|                 | `driver.navigate().refresh()`                 | Reload page                    |
| **Page Info**   | `driver.getCurrentUrl()`                      | Returns current URL string     |
|                 | `driver.getTitle()`                           | Returns page `<title>`         |
|                 | `driver.getPageSource()`                      | Returns full HTML source       |
| **Finding**     | `driver.findElement(By)`                      | Returns first match or throws  |
|                 | `driver.findElements(By)`                     | Returns list (empty if none)   |
| **Windows**     | `driver.getWindowHandle()`                    | Current window identifier      |
|                 | `driver.getWindowHandles()`                   | All window identifiers         |
|                 | `driver.switchTo().window(handle)`            | Switch focus to window         |
|                 | `driver.switchTo().newWindow(type)`           | Open new tab or window         |
|                 | `driver.manage().window().maximize()`         | Maximize window                |
|                 | `driver.manage().window().setSize()`          | Set specific size              |
| **Timeouts**    | `driver.manage().timeouts()`                  | Configure wait timeouts        |
| **Cookies**     | `driver.manage().getCookies()`                | All current cookies            |
|                 | `driver.manage().addCookie()`                 | Add a cookie                   |
|                 | `driver.manage().deleteAllCookies()`          | Clear all cookies              |
| **Frames**      | `driver.switchTo().frame(id/index/element)`   | Switch into an iframe          |
|                 | `driver.switchTo().defaultContent()`          | Switch back to main page       |
| **Alerts**      | `driver.switchTo().alert()`                   | Get alert object               |
| **Screenshots** | `((TakesScreenshot)driver).getScreenshotAs()` | Capture screenshot             |
| **Cleanup**     | `driver.close()`                              | Close current window           |
|                 | `driver.quit()`                               | End session, close all windows |

---

## Summary

The WebDriver API gives you complete control over browser behavior — navigation, element interaction, window management, cookies, screenshots, and more. Master these fundamentals and everything else in Selenium becomes composing these building blocks in smarter and more organized ways. The Page Object Model, Cucumber steps, and utilities all ultimately call these same WebDriver methods underneath.
