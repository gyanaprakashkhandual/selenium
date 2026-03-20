# Cross-Browser Testing

## Overview

Cross-browser testing verifies that a web application behaves consistently across different browsers, browser versions, and operating systems. Despite modern browsers converging on web standards, meaningful differences still exist in CSS rendering, JavaScript engine behaviour, WebDriver event handling, file input processing, and font metrics. A professional Selenium framework makes cross-browser testing a configuration choice — switching from Chrome to Firefox requires changing one property, not modifying any test code.

---

## Supported Browsers and WebDriver Implementations

| Browser         | WebDriver Class          | WebDriverManager Artifact          | Notes                                   |
| --------------- | ------------------------ | ---------------------------------- | --------------------------------------- |
| Google Chrome   | `ChromeDriver`           | `WebDriverManager.chromedriver()`  | Most widely used in CI                  |
| Mozilla Firefox | `FirefoxDriver`          | `WebDriverManager.firefoxdriver()` | Strong Linux support                    |
| Microsoft Edge  | `EdgeDriver`             | `WebDriverManager.edgedriver()`    | Chromium-based since 2020               |
| Apple Safari    | `SafariDriver`           | Built-in (macOS only)              | Requires enabling in Safari preferences |
| Opera           | `ChromeDriver` + options | Via ChromeDriver                   | Chromium-based                          |

---

## DriverFactory — Full Cross-Browser Implementation

The `DriverFactory` is the single place where browser selection logic lives. The browser is driven entirely by configuration and runtime system properties. No test code, page class, or step definition knows which browser is running.

```java
package com.company.framework.driver;

import com.company.framework.config.ConfigReader;
import io.github.bonigarcia.wdm.WebDriverManager;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.MutableCapabilities;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.edge.EdgeDriver;
import org.openqa.selenium.edge.EdgeOptions;
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.firefox.FirefoxOptions;
import org.openqa.selenium.firefox.FirefoxProfile;
import org.openqa.selenium.remote.RemoteWebDriver;
import org.openqa.selenium.safari.SafariDriver;
import org.openqa.selenium.safari.SafariOptions;

import java.net.MalformedURLException;
import java.net.URL;
import java.time.Duration;
import java.util.HashMap;
import java.util.Map;

public class DriverFactory {

    private static final Logger log = LogManager.getLogger(DriverFactory.class);

    private DriverFactory() {}

    public static WebDriver createDriver() {
        String browser   = resolveBrowser();
        boolean headless = ConfigReader.getBoolean("headless", false);
        boolean useGrid  = ConfigReader.getBoolean("grid.enabled", false);

        log.info("Creating driver: browser={} | headless={} | grid={}",
            browser, headless, useGrid);

        WebDriver driver = useGrid
            ? createRemoteDriver(browser, headless)
            : createLocalDriver(browser, headless);

        configureTimeouts(driver);
        return driver;
    }

    // -----------------------------------------------------------------------
    // Local Driver Creation
    // -----------------------------------------------------------------------

    private static WebDriver createLocalDriver(String browser, boolean headless) {
        switch (browser) {
            case "firefox":          return createFirefox(headless);
            case "edge":             return createEdge(headless);
            case "safari":           return createSafari();
            case "chrome":
            default:                 return createChrome(headless);
        }
    }

    private static WebDriver createChrome(boolean headless) {
        WebDriverManager.chromedriver().setup();
        ChromeOptions options = buildChromeOptions(headless);
        log.info("ChromeDriver initialized{}", headless ? " (headless)" : "");
        return new ChromeDriver(options);
    }

    private static WebDriver createFirefox(boolean headless) {
        WebDriverManager.firefoxdriver().setup();
        FirefoxOptions options = buildFirefoxOptions(headless);
        log.info("FirefoxDriver initialized{}", headless ? " (headless)" : "");
        return new FirefoxDriver(options);
    }

    private static WebDriver createEdge(boolean headless) {
        WebDriverManager.edgedriver().setup();
        EdgeOptions options = buildEdgeOptions(headless);
        log.info("EdgeDriver initialized{}", headless ? " (headless)" : "");
        return new EdgeDriver(options);
    }

    private static WebDriver createSafari() {
        // Safari driver requires "Allow Remote Automation" enabled in Safari → Develop menu
        SafariOptions options = new SafariOptions();
        options.setUseTechnologyPreview(false);
        log.info("SafariDriver initialized (macOS only)");
        return new SafariDriver(options);
    }

    // -----------------------------------------------------------------------
    // Remote Driver (Selenium Grid / BrowserStack / Sauce Labs)
    // -----------------------------------------------------------------------

    private static WebDriver createRemoteDriver(String browser, boolean headless) {
        String gridUrl = ConfigReader.get("grid.url", "http://localhost:4444");

        MutableCapabilities options;
        switch (browser) {
            case "firefox": options = buildFirefoxOptions(headless); break;
            case "edge":    options = buildEdgeOptions(headless);    break;
            default:        options = buildChromeOptions(headless);  break;
        }

        try {
            log.info("Connecting RemoteWebDriver: {} → {}", browser, gridUrl);
            return new RemoteWebDriver(new URL(gridUrl), options);
        } catch (MalformedURLException e) {
            throw new RuntimeException("Invalid grid URL: " + gridUrl, e);
        }
    }

    // -----------------------------------------------------------------------
    // Options Builders
    // -----------------------------------------------------------------------

    private static ChromeOptions buildChromeOptions(boolean headless) {
        ChromeOptions options = new ChromeOptions();

        options.addArguments("--no-sandbox");
        options.addArguments("--disable-dev-shm-usage");
        options.addArguments("--disable-gpu");
        options.addArguments("--disable-extensions");
        options.addArguments("--disable-notifications");
        options.addArguments("--disable-infobars");
        options.addArguments("--window-size=1920,1080");

        if (headless) {
            options.addArguments("--headless=new");
        } else {
            options.addArguments("--start-maximized");
        }

        // Download preferences
        String downloadDir = ConfigReader.get("download.dir", "target/downloads");
        Map<String, Object> prefs = new HashMap<>();
        prefs.put("download.default_directory",  new java.io.File(downloadDir).getAbsolutePath());
        prefs.put("download.prompt_for_download", false);
        prefs.put("safebrowsing.enabled",         true);
        prefs.put("plugins.always_open_pdf_externally", true);
        options.setExperimentalOption("prefs", prefs);

        return options;
    }

    private static FirefoxOptions buildFirefoxOptions(boolean headless) {
        FirefoxOptions options = new FirefoxOptions();

        if (headless) {
            options.addArguments("-headless");
        }

        // Firefox-specific preferences
        FirefoxProfile profile = new FirefoxProfile();
        profile.setPreference("browser.download.folderList", 2);
        profile.setPreference("browser.download.dir",
            new java.io.File(ConfigReader.get("download.dir", "target/downloads"))
                .getAbsolutePath());
        profile.setPreference("browser.helperApps.neverAsk.saveToDisk",
            "application/pdf,application/octet-stream,text/csv,application/zip");
        profile.setPreference("pdfjs.disabled", true);
        profile.setPreference("browser.download.manager.showWhenStarting", false);
        options.setProfile(profile);

        return options;
    }

    private static EdgeOptions buildEdgeOptions(boolean headless) {
        EdgeOptions options = new EdgeOptions();

        options.addArguments("--no-sandbox");
        options.addArguments("--disable-dev-shm-usage");
        options.addArguments("--disable-notifications");
        options.addArguments("--window-size=1920,1080");

        if (headless) {
            options.addArguments("--headless=new");
        } else {
            options.addArguments("--start-maximized");
        }

        return options;
    }

    // -----------------------------------------------------------------------
    // Timeout Configuration
    // -----------------------------------------------------------------------

    private static void configureTimeouts(WebDriver driver) {
        int implicit = ConfigReader.getInt("timeout.implicit", 0);
        int pageLoad = ConfigReader.getInt("timeout.pageload", 30);
        int script   = ConfigReader.getInt("timeout.script", 30);

        driver.manage().timeouts()
            .implicitlyWait(Duration.ofSeconds(implicit))
            .pageLoadTimeout(Duration.ofSeconds(pageLoad))
            .scriptTimeout(Duration.ofSeconds(script));
    }

    private static String resolveBrowser() {
        return System.getProperty("browser",
            ConfigReader.get("browser", "chrome")).toLowerCase().trim();
    }
}
```

---

## Running Against Different Browsers

```bash
# Chrome (default)
mvn test

# Firefox
mvn test -Dbrowser=firefox

# Edge
mvn test -Dbrowser=edge

# Chrome headless
mvn test -Dbrowser=chrome -Dheadless=true

# Firefox headless
mvn test -Dbrowser=firefox -Dheadless=true

# Remote (Grid)
mvn test -Dbrowser=chrome -Dgrid.enabled=true -Dgrid.url=http://localhost:4444
```

---

## Cross-Browser TestNG Matrix

Run the full regression suite against multiple browsers in parallel:

```xml
<!-- testng-cross-browser.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">

<suite name="Cross-Browser Suite" parallel="tests" thread-count="3">

    <test name="Chrome Regression">
        <parameter name="browser" value="chrome"/>
        <classes>
            <class name="com.company.framework.runners.RegressionRunner"/>
        </classes>
    </test>

    <test name="Firefox Regression">
        <parameter name="browser" value="firefox"/>
        <classes>
            <class name="com.company.framework.runners.RegressionRunner"/>
        </classes>
    </test>

    <test name="Edge Smoke">
        <parameter name="browser" value="edge"/>
        <classes>
            <class name="com.company.framework.runners.SmokeTestRunner"/>
        </classes>
    </test>

</suite>
```

Read the browser parameter in `@Before`:

```java
@Before(order = 1)
public void initDriver(Scenario scenario) {
    // @Parameter from testng.xml overrides -Dbrowser system property
    String browserParam = System.getProperty("browser");
    if (browserParam != null) {
        System.setProperty("browser", browserParam);
    }
    DriverManager.setDriver(DriverFactory.createDriver());
}
```

---

## Browser-Specific Issues and Fixes

### Issue 1 — sendKeys Behaviour

```java
// Chrome and Edge: sendKeys appends to existing content
// Fix: always clear first
public void type(WebElement element, String text) {
    element.clear();
    element.sendKeys(text);
}

// Firefox: clear() can sometimes fail to clear all content
// Fix: use Ctrl+A then type for Firefox reliability
public void typeFirefoxSafe(WebElement element, String text) {
    element.click();
    element.sendKeys(Keys.chord(Keys.CONTROL, "a"), text);
}
```

### Issue 2 — Click Interception

```java
// Firefox is stricter about click interception than Chrome
// Fix: scroll into view before clicking
public void scrollAndClick(WebElement element) {
    ((JavascriptExecutor) driver).executeScript(
        "arguments[0].scrollIntoView({block:'center'});", element);
    new WebDriverWait(driver, Duration.ofSeconds(5))
        .until(ExpectedConditions.elementToBeClickable(element))
        .click();
}
```

### Issue 3 — Hover/Hover Menus

```java
// Safari does not support Actions.moveToElement reliably
// Fix: use JavaScript to trigger hover events on Safari
public void hoverElement(WebElement element) {
    String browser = ConfigReader.get("browser", "chrome");
    if (browser.equals("safari")) {
        ((JavascriptExecutor) driver).executeScript(
            "arguments[0].dispatchEvent(new MouseEvent('mouseover', {bubbles: true}));",
            element);
    } else {
        new Actions(driver).moveToElement(element).perform();
    }
}
```

### Issue 4 — File Input

```java
// File upload with sendKeys works on all browsers except Safari
// Safari requires the element to be visible
public void uploadFile(WebElement fileInput, String path) {
    String browser = ConfigReader.get("browser", "chrome");
    if (browser.equals("safari")) {
        // Safari needs the input to be displayed first
        ((JavascriptExecutor) driver).executeScript(
            "arguments[0].style.display='block';", fileInput);
    }
    fileInput.sendKeys(path);
}
```

### Issue 5 — Scrolling Behaviour

```java
// window.scrollIntoView() behaviour differs across browsers
// Use a consistent scroll approach
public void scrollToElement(WebElement element) {
    ((JavascriptExecutor) driver).executeScript(
        "arguments[0].scrollIntoView({behavior: 'instant', block: 'center'});",
        element);
    // Give the browser a moment to reflow after scroll
    try { Thread.sleep(100); } catch (InterruptedException ignored) {}
}
```

### Issue 6 — Date Input Fields

```java
// Chrome accepts input type="date" via sendKeys with format yyyy-MM-dd
// Firefox requires different handling
public void enterDate(WebElement dateField, String date) {
    String browser = ConfigReader.get("browser", "chrome");
    dateField.clear();
    if (browser.equals("firefox")) {
        // Firefox: send each part separately
        String[] parts = date.split("-");  // yyyy-MM-dd
        dateField.sendKeys(parts[1]);  // month
        dateField.sendKeys(parts[2]);  // day
        dateField.sendKeys(parts[0]);  // year
    } else {
        // Chrome, Edge: direct send
        dateField.sendKeys(date);
    }
}
```

---

## Browser Detection Utility

When browser-specific branches are unavoidable, encapsulate the detection:

```java
package com.company.framework.utils;

import com.company.framework.config.ConfigReader;
import org.openqa.selenium.Capabilities;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.remote.RemoteWebDriver;

public class BrowserUtils {

    private BrowserUtils() {}

    public static String getBrowserName() {
        return ConfigReader.get("browser", "chrome").toLowerCase();
    }

    public static String getBrowserName(WebDriver driver) {
        if (driver instanceof RemoteWebDriver) {
            Capabilities caps = ((RemoteWebDriver) driver).getCapabilities();
            return caps.getBrowserName().toLowerCase();
        }
        return getBrowserName();
    }

    public static String getBrowserVersion(WebDriver driver) {
        if (driver instanceof RemoteWebDriver) {
            Capabilities caps = ((RemoteWebDriver) driver).getCapabilities();
            return caps.getBrowserVersion();
        }
        return "unknown";
    }

    public static boolean isChrome(WebDriver driver)   { return getBrowserName(driver).contains("chrome"); }
    public static boolean isFirefox(WebDriver driver)  { return getBrowserName(driver).contains("firefox"); }
    public static boolean isEdge(WebDriver driver)     { return getBrowserName(driver).contains("msedge"); }
    public static boolean isSafari(WebDriver driver)   { return getBrowserName(driver).contains("safari"); }
    public static boolean isHeadless()                 { return ConfigReader.getBoolean("headless", false); }
}
```

---

## Cloud-Based Cross-Browser Testing

For extensive browser and OS matrix coverage, use a cloud provider:

### BrowserStack Configuration

```java
private static WebDriver createBrowserStackDriver(String browser, String os) {
    MutableCapabilities caps = new MutableCapabilities();

    caps.setCapability("browserName", browser);
    caps.setCapability("browserVersion", "latest");

    Map<String, Object> bsOptions = new HashMap<>();
    bsOptions.put("os",          os);
    bsOptions.put("osVersion",   "11");
    bsOptions.put("projectName", "Selenium Framework");
    bsOptions.put("buildName",   System.getenv().getOrDefault("BUILD_NUMBER", "local"));
    bsOptions.put("sessionName", "Regression Suite");
    bsOptions.put("debug",        true);
    bsOptions.put("networkLogs",  true);

    caps.setCapability("bstack:options", bsOptions);

    String username  = ConfigReader.get("browserstack.username");
    String accessKey = ConfigReader.get("browserstack.accesskey");
    String bsUrl     = "https://" + username + ":" + accessKey + "@hub.browserstack.com/wd/hub";

    try {
        return new RemoteWebDriver(new URL(bsUrl), caps);
    } catch (MalformedURLException e) {
        throw new RuntimeException("Invalid BrowserStack URL", e);
    }
}
```

```properties
# config-browserstack.properties
browserstack.username=your_username
browserstack.accesskey=your_access_key
browser=chrome
grid.enabled=true
grid.url=https://hub.browserstack.com/wd/hub
```

---

## Cross-Browser Test Strategy

Not every test needs to run on every browser. A tiered cross-browser strategy maximises coverage efficiently:

| Tier      | Browsers       | Tests           | When              |
| --------- | -------------- | --------------- | ----------------- |
| Primary   | Chrome         | Full regression | Every CI run      |
| Secondary | Firefox        | Full regression | Nightly           |
| Tertiary  | Edge           | Smoke only      | Weekly            |
| Extended  | Safari, mobile | Smoke only      | Release candidate |
| Cloud     | All + versions | Critical paths  | Pre-release       |

Tag your scenarios to target the right tier:

```gherkin
@smoke @cross-browser
Scenario: Core login flow works on all browsers
  ...

@regression @chrome-only
Scenario: CDP network interception test
  # CDP is Chrome/Edge specific — don't run on Firefox/Safari
  ...
```

---

## Summary

Cross-browser testing is an architectural decision, not a test-by-test effort. The `DriverFactory` abstracts every browser-specific option behind a single `browser` configuration property. Switching browsers requires changing one property value or passing `-Dbrowser=firefox` at the command line. Browser-specific workarounds for sendKeys, hover, scroll, file upload, and date input are encapsulated in `BasePage` or utility methods — never in step definitions or feature files. A tiered strategy runs Chrome on every build, Firefox nightly, Edge weekly, and cloud providers before release, maximising coverage without multiplying build times unnecessarily.

---

## Related Topics

- DriverManager.md — ThreadLocal WebDriver management per thread
- ParallelExecution.md — Running multiple browsers in parallel via testng.xml
- SeleniumGrid.md — Grid nodes for multi-browser execution
- GitHubActions.md — Matrix strategy for parallel cross-browser CI builds
- HeadlessTesting.md — Headless configuration per browser
