# Browser Commands

## Overview

Browser commands are the WebDriver methods that control the browser itself — as opposed to interacting with page elements. They cover everything from window sizing and tab management to executing JavaScript, managing cookies, and capturing screenshots. Mastering these commands is essential for building a framework that handles complex real-world scenarios.

---

## Window and Size Management

### Maximize, Resize, and Fullscreen

```java
import org.openqa.selenium.Dimension;
import org.openqa.selenium.Point;
import org.openqa.selenium.WebDriver;

// Maximize — fills the screen (most common setup step)
driver.manage().window().maximize();

// Set exact dimensions (for responsive/breakpoint testing)
driver.manage().window().setSize(new Dimension(1920, 1080));
driver.manage().window().setSize(new Dimension(1366, 768));
driver.manage().window().setSize(new Dimension(768, 1024));   // Tablet portrait
driver.manage().window().setSize(new Dimension(375, 812));    // iPhone X

// Get current window size
Dimension size = driver.manage().window().getSize();
System.out.println("Width: " + size.getWidth() + ", Height: " + size.getHeight());

// Move window position on screen
driver.manage().window().setPosition(new Point(0, 0));
driver.manage().window().setPosition(new Point(100, 100));

// Get current position
Point position = driver.manage().window().getPosition();
System.out.println("X: " + position.getX() + ", Y: " + position.getY());

// True fullscreen (hides browser chrome — like pressing F11)
driver.manage().window().fullscreen();

// Minimize (Selenium 4)
driver.manage().window().minimize();
```

### Responsive Testing Helper

```java
public enum Viewport {
    DESKTOP_FHD(1920, 1080),
    DESKTOP_HD(1366, 768),
    LAPTOP(1280, 800),
    TABLET_LANDSCAPE(1024, 768),
    TABLET_PORTRAIT(768, 1024),
    MOBILE_LARGE(414, 896),    // iPhone XR
    MOBILE_STANDARD(375, 667), // iPhone 8
    MOBILE_SMALL(320, 568);    // iPhone SE

    public final int width;
    public final int height;

    Viewport(int width, int height) {
        this.width = width;
        this.height = height;
    }
}

// Usage in tests
public void setViewport(Viewport viewport) {
    driver.manage().window().setSize(new Dimension(viewport.width, viewport.height));
}

setViewport(Viewport.MOBILE_STANDARD);
// Run mobile-specific assertions...

setViewport(Viewport.DESKTOP_FHD);
// Run desktop assertions...
```

---

## Multi-Window and Multi-Tab Management

### Understanding Window Handles

Every browser window and tab has a unique string identifier called a **window handle**. Selenium tracks the currently focused window, and you must explicitly switch focus to interact with other windows.

```java
// Get the current (focused) window handle
String mainWindow = driver.getWindowHandle();
System.out.println("Main window handle: " + mainWindow);

// Get ALL open window handles as a Set
Set<String> allHandles = driver.getWindowHandles();
System.out.println("Total open windows/tabs: " + allHandles.size());
```

### Switching to a New Window/Tab

```java
// Scenario: clicking a link opens a new tab
String mainWindow = driver.getWindowHandle();

driver.findElement(By.id("open-new-tab-link")).click();

// Wait for the new tab to open
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));
wait.until(d -> d.getWindowHandles().size() > 1);

// Switch to the newly opened tab
for (String handle : driver.getWindowHandles()) {
    if (!handle.equals(mainWindow)) {
        driver.switchTo().window(handle);
        break;
    }
}

System.out.println("New tab title: " + driver.getTitle());
System.out.println("New tab URL: " + driver.getCurrentUrl());

// Do work in new tab...

// Close the new tab and switch back to main
driver.close();
driver.switchTo().window(mainWindow);
System.out.println("Back to main: " + driver.getTitle());
```

### Opening New Tabs/Windows Programmatically (Selenium 4)

```java
import org.openqa.selenium.WindowType;

// Open a new blank TAB (stays in same browser window)
driver.switchTo().newWindow(WindowType.TAB);
driver.get("https://example.com/page2");
System.out.println("New tab opened: " + driver.getTitle());

// Open a new browser WINDOW
driver.switchTo().newWindow(WindowType.WINDOW);
driver.get("https://example.com/page3");
System.out.println("New window opened: " + driver.getTitle());
```

### Managing Multiple Windows Utility

```java
import java.util.*;

public class WindowManager {

    private final WebDriver driver;
    private final WebDriverWait wait;

    public WindowManager(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    /**
     * Switch to the most recently opened window/tab.
     */
    public void switchToNewestWindow() {
        String currentHandle = driver.getWindowHandle();
        wait.until(d -> d.getWindowHandles().size() > 1);

        List<String> handles = new ArrayList<>(driver.getWindowHandles());
        String newest = handles.get(handles.size() - 1);
        driver.switchTo().window(newest);
    }

    /**
     * Switch to a window by its title (partial match).
     */
    public boolean switchToWindowByTitle(String titleFragment) {
        for (String handle : driver.getWindowHandles()) {
            driver.switchTo().window(handle);
            if (driver.getTitle().contains(titleFragment)) {
                return true;
            }
        }
        return false;
    }

    /**
     * Switch to a window by its URL (partial match).
     */
    public boolean switchToWindowByUrl(String urlFragment) {
        for (String handle : driver.getWindowHandles()) {
            driver.switchTo().window(handle);
            if (driver.getCurrentUrl().contains(urlFragment)) {
                return true;
            }
        }
        return false;
    }

    /**
     * Close all windows except the one with the given handle.
     */
    public void closeAllExcept(String keepHandle) {
        for (String handle : driver.getWindowHandles()) {
            if (!handle.equals(keepHandle)) {
                driver.switchTo().window(handle);
                driver.close();
            }
        }
        driver.switchTo().window(keepHandle);
    }

    /**
     * Get total number of open windows/tabs.
     */
    public int getWindowCount() {
        return driver.getWindowHandles().size();
    }
}
```

---

## JavaScript Execution

Selenium can execute JavaScript directly in the browser context. This is a powerful escape hatch for scenarios where the WebDriver API alone cannot accomplish a task.

### Basic JavaScript Execution

```java
import org.openqa.selenium.JavascriptExecutor;

JavascriptExecutor js = (JavascriptExecutor) driver;

// Execute JS and get a return value
Object result = js.executeScript("return document.title;");
System.out.println("Title via JS: " + result);

// Execute JS with no return
js.executeScript("window.scrollTo(0, document.body.scrollHeight);");

// Pass WebElement as argument to JS
WebElement button = driver.findElement(By.id("submit-btn"));
js.executeScript("arguments[0].click();", button);

// Pass multiple arguments
WebElement input = driver.findElement(By.id("email"));
js.executeScript("arguments[0].value = arguments[1];", input, "test@example.com");
```

### Common JavaScript Commands

```java
JavascriptExecutor js = (JavascriptExecutor) driver;

// ===== SCROLLING =====

// Scroll to bottom of page
js.executeScript("window.scrollTo(0, document.body.scrollHeight)");

// Scroll to top of page
js.executeScript("window.scrollTo(0, 0)");

// Scroll by a specific amount (pixels)
js.executeScript("window.scrollBy(0, 500)");

// Scroll a specific element into view
WebElement element = driver.findElement(By.id("footer-section"));
js.executeScript("arguments[0].scrollIntoView(true);", element);

// Scroll into view with options (smooth + center)
js.executeScript(
    "arguments[0].scrollIntoView({behavior: 'smooth', block: 'center'});",
    element
);

// ===== ELEMENT INTERACTION =====

// Click via JS (use when regular click fails due to overlay/intercept)
WebElement btn = driver.findElement(By.id("submit"));
js.executeScript("arguments[0].click();", btn);

// Set input value via JS (use when sendKeys is blocked)
WebElement dateInput = driver.findElement(By.id("date-picker"));
js.executeScript("arguments[0].value = '2024-12-25';", dateInput);

// Remove readonly attribute from an input
WebElement readonlyInput = driver.findElement(By.id("readonly-field"));
js.executeScript("arguments[0].removeAttribute('readonly');", readonlyInput);
readonlyInput.clear();
readonlyInput.sendKeys("new value");

// Remove disabled attribute from a button
WebElement disabledBtn = driver.findElement(By.id("disabled-btn"));
js.executeScript("arguments[0].removeAttribute('disabled');", disabledBtn);

// ===== DOM QUERIES =====

// Get element text
String text = (String) js.executeScript(
    "return arguments[0].textContent;",
    driver.findElement(By.id("message"))
);

// Get computed style
String color = (String) js.executeScript(
    "return window.getComputedStyle(arguments[0]).getPropertyValue('color');",
    driver.findElement(By.id("alert-box"))
);

// Check if element is visible (using JS instead of WebElement.isDisplayed())
Boolean visible = (Boolean) js.executeScript(
    "var elem = arguments[0], " +
    "box = elem.getBoundingClientRect(); " +
    "return box.top < window.innerHeight && box.bottom >= 0 && " +
    "box.left < window.innerWidth && box.right >= 0;",
    element
);

// ===== BROWSER STATE =====

// Get page load state
String readyState = (String) js.executeScript("return document.readyState;");
// "loading" | "interactive" | "complete"

// Get viewport dimensions
Long viewportWidth  = (Long) js.executeScript("return window.innerWidth;");
Long viewportHeight = (Long) js.executeScript("return window.innerHeight;");

// Get scroll position
Long scrollY = (Long) js.executeScript("return window.pageYOffset;");
Long scrollX = (Long) js.executeScript("return window.pageXOffset;");

// Highlight element (useful for debugging)
js.executeScript(
    "arguments[0].style.border = '3px solid red';",
    driver.findElement(By.id("target-element"))
);
```

### Asynchronous JavaScript

```java
// executeAsyncScript — waits for callback to be called
// Timeout controlled by driver.manage().timeouts().scriptTimeout()
driver.manage().timeouts().scriptTimeout(Duration.ofSeconds(10));

Object result = ((JavascriptExecutor) driver).executeAsyncScript(
    "var callback = arguments[arguments.length - 1];" +
    "setTimeout(function() { callback('done'); }, 2000);"
);
System.out.println("Async result: " + result); // "done"

// Wait for Angular to finish rendering
((JavascriptExecutor) driver).executeAsyncScript(
    "var callback = arguments[arguments.length - 1];" +
    "angular.element(document.body).injector()" +
    ".get('$browser').notifyWhenNoOutstandingRequests(callback);"
);
```

---

## Page Load Strategies

Page load strategy controls how long Selenium waits after `driver.get()` before returning control to your test.

```java
import org.openqa.selenium.PageLoadStrategy;
import org.openqa.selenium.chrome.ChromeOptions;

// NORMAL (default) — waits for document.readyState == "complete"
// Waits for all resources (HTML, CSS, JS, images) to finish loading
ChromeOptions normalOptions = new ChromeOptions();
normalOptions.setPageLoadStrategy(PageLoadStrategy.NORMAL);

// EAGER — waits for document.readyState == "interactive"
// HTML is parsed and DOM is ready, but images/CSS may still be loading
// Faster than NORMAL for pages with many large assets
ChromeOptions eagerOptions = new ChromeOptions();
eagerOptions.setPageLoadStrategy(PageLoadStrategy.EAGER);

// NONE — returns immediately after the navigate command is sent
// You must implement your own wait strategy
// Use for SPAs where document.readyState never fully reaches 'complete'
ChromeOptions noneOptions = new ChromeOptions();
noneOptions.setPageLoadStrategy(PageLoadStrategy.NONE);
WebDriver driver = new ChromeDriver(noneOptions);
driver.get("https://app.example.com");
// You must now manually wait for the page to be in a usable state
new WebDriverWait(driver, Duration.ofSeconds(15))
    .until(d -> ((JavascriptExecutor) d)
        .executeScript("return document.readyState").equals("complete"));
```

---

## Browser Profiles and Preferences

### Chrome Preferences

```java
import org.openqa.selenium.chrome.ChromeOptions;
import java.util.HashMap;
import java.util.Map;

ChromeOptions options = new ChromeOptions();

// ===== COMMON PREFERENCES =====
Map<String, Object> prefs = new HashMap<>();

// Disable password save dialog
prefs.put("credentials_enable_service", false);
prefs.put("profile.password_manager_enabled", false);

// Set default download directory
prefs.put("download.default_directory", System.getProperty("user.dir") + "/downloads");
prefs.put("download.prompt_for_download", false);
prefs.put("download.directory_upgrade", true);
prefs.put("safebrowsing.enabled", true);

// Auto-allow file download without dialog
prefs.put("plugins.always_open_pdf_externally", true);

// Disable notifications
prefs.put("profile.default_content_setting_values.notifications", 2);

// Disable geolocation prompts
prefs.put("profile.default_content_setting_values.geolocation", 2);

// Disable images (faster page load)
prefs.put("profile.managed_default_content_settings.images", 2);

options.setExperimentalOption("prefs", prefs);
options.addArguments("--disable-notifications");
options.addArguments("--disable-popup-blocking");

WebDriver driver = new ChromeDriver(options);
```

### Firefox Profile

```java
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.firefox.FirefoxOptions;
import org.openqa.selenium.firefox.FirefoxProfile;

FirefoxProfile profile = new FirefoxProfile();

// Set download directory
profile.setPreference("browser.download.folderList", 2);
profile.setPreference("browser.download.dir", System.getProperty("user.dir") + "/downloads");
profile.setPreference("browser.download.manager.showWhenStarting", false);

// Auto-download these MIME types (don't ask for save dialog)
profile.setPreference("browser.helperApps.neverAsk.saveToDisk",
    "application/pdf," +
    "application/zip," +
    "application/x-zip-compressed," +
    "text/csv," +
    "application/vnd.ms-excel"
);

// Disable update checks
profile.setPreference("app.update.enabled", false);

// Disable native events (can improve stability)
profile.setPreference("webdriver_enable_native_events", false);

FirefoxOptions options = new FirefoxOptions();
options.setProfile(profile);
WebDriver driver = new FirefoxDriver(options);
```

---

## Browser Logs

```java
import org.openqa.selenium.logging.LogEntries;
import org.openqa.selenium.logging.LogEntry;
import org.openqa.selenium.logging.LogType;
import org.openqa.selenium.logging.LoggingPreferences;
import org.openqa.selenium.chrome.ChromeOptions;

import java.util.logging.Level;

// Enable browser console log capture
ChromeOptions options = new ChromeOptions();
LoggingPreferences logPrefs = new LoggingPreferences();
logPrefs.enable(LogType.BROWSER, Level.ALL);      // JS console logs
logPrefs.enable(LogType.PERFORMANCE, Level.INFO); // Network/performance logs
options.setCapability("goog:loggingPrefs", logPrefs);

WebDriver driver = new ChromeDriver(options);
driver.get("https://example.com");

// After page action, capture browser logs
LogEntries browserLogs = driver.manage().logs().get(LogType.BROWSER);
for (LogEntry entry : browserLogs) {
    System.out.println("[" + entry.getLevel() + "] " + entry.getMessage());
}

// Filter only SEVERE errors (good for checking JS errors in tests)
browserLogs.getAll().stream()
    .filter(entry -> entry.getLevel() == Level.SEVERE)
    .forEach(entry -> System.err.println("JS ERROR: " + entry.getMessage()));
```

---

## Browser Capabilities — Full Configuration

```java
import org.openqa.selenium.chrome.ChromeOptions;

ChromeOptions options = new ChromeOptions();

// ===== WINDOW =====
options.addArguments("--start-maximized");
options.addArguments("--window-size=1920,1080");
options.addArguments("--window-position=0,0");

// ===== SECURITY / AUTH =====
options.addArguments("--ignore-certificate-errors");
options.addArguments("--allow-insecure-localhost");

// ===== NETWORK =====
options.addArguments("--proxy-server=http://proxy.company.com:8080");
options.addArguments("--proxy-bypass-list=localhost,127.0.0.1");

// ===== PERFORMANCE =====
options.addArguments("--disable-gpu");                // Required for headless
options.addArguments("--disable-extensions");         // Faster startup
options.addArguments("--disable-background-networking");
options.addArguments("--disable-default-apps");
options.addArguments("--disable-sync");
options.addArguments("--no-first-run");

// ===== DOCKER / CI =====
options.addArguments("--no-sandbox");                 // Required in Docker
options.addArguments("--disable-dev-shm-usage");      // Overcome limited /dev/shm
options.addArguments("--remote-allow-origins=*");     // Required Selenium 4

// ===== HEADLESS =====
options.addArguments("--headless=new");               // Chrome 112+

// ===== CHROME BINARY PATH =====
// Specify a non-default Chrome installation
options.setBinary("/opt/google/chrome-unstable/chrome");

// ===== LOAD EXTENSION =====
options.addExtensions(new File("path/to/extension.crx"));

WebDriver driver = new ChromeDriver(options);
```

---

## Printing / PDF (Selenium 4)

Selenium 4 added native printing/PDF support:

```java
import org.openqa.selenium.print.PrintOptions;
import org.openqa.selenium.print.PrintOptions.Orientation;
import org.openqa.selenium.Pdf;

// Configure print options
PrintOptions printOptions = new PrintOptions();
printOptions.setOrientation(Orientation.LANDSCAPE);
printOptions.setScale(0.75);
printOptions.setPageRanges("1-3");
printOptions.setShrinkToFit(true);

// Print to PDF (returns Base64-encoded PDF)
Pdf pdf = ((PrintsPage) driver).print(printOptions);

// Decode and save as a file
byte[] pdfBytes = Base64.getDecoder().decode(pdf.getContent());
Files.write(Path.of("output/report.pdf"), pdfBytes);
```

---

## Emulating Network Conditions (Chrome DevTools Protocol)

Using Chrome DevTools Protocol (CDP) via Selenium 4's `DevTools` API:

```java
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.v121.network.Network;
import org.openqa.selenium.devtools.v121.network.model.ConnectionType;

import java.util.Optional;

ChromeDriver driver = new ChromeDriver();
DevTools devTools = driver.getDevTools();
devTools.createSession();

// Enable network domain
devTools.send(Network.enable(Optional.empty(), Optional.empty(), Optional.empty()));

// Emulate slow 3G network
devTools.send(Network.emulateNetworkConditions(
    false,          // offline
    20,             // latency (ms)
    500 * 1024 / 8, // downloadThroughput (bytes/s) — 500 kbps
    500 * 1024 / 8, // uploadThroughput
    Optional.of(ConnectionType.CELLULAR3G)
));

driver.get("https://example.com"); // Will load slowly like on 3G

// Simulate offline
devTools.send(Network.emulateNetworkConditions(
    true, 0, 0, 0, Optional.of(ConnectionType.NONE)
));

// Restore normal network
devTools.send(Network.emulateNetworkConditions(
    false, 0, -1, -1, Optional.of(ConnectionType.ALL)
));
```

---

## Summary

Browser commands give you full control over the browser as a whole — window sizing, tab management, JavaScript execution, preferences, logging, and network emulation. The most frequently used are `maximize()`, `switchTo().window()`, JavaScript execution for edge cases, and Chrome preferences for controlling download and notification behavior. Build utility classes (like `WindowManager`) to encapsulate complex browser interactions and keep your test code clean.
