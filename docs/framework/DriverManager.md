# Driver Manager

## Overview

The Driver Manager is the central component responsible for creating, providing, and destroying WebDriver instances in the framework. It ensures that each test scenario gets its own isolated browser session, that parallel test execution does not cause thread interference, and that driver creation details are not scattered across hooks, base classes, or step definitions.

A properly implemented Driver Manager uses `ThreadLocal<WebDriver>` to store one driver per thread. This is the only correct approach for frameworks that run scenarios in parallel.

---

## Why a Dedicated Driver Manager

Without a Driver Manager, driver management code ends up in multiple places: the Hooks class, the BasePage constructor, step definitions, and sometimes static fields on utility classes. This leads to:

- Driver instances leaking between scenarios if teardown fails
- Static WebDriver fields causing cross-thread contamination in parallel runs
- Tight coupling between Selenium code and test infrastructure
- No single place to control browser configuration

The Driver Manager consolidates all of this into two classes: `DriverManager` (the ThreadLocal container) and `DriverFactory` (the browser creation logic).

---

## ThreadLocal WebDriver — The Core Principle

`ThreadLocal<T>` in Java creates a separate storage slot for each thread. When two threads call `ThreadLocal.get()`, they each retrieve their own value independently. This is exactly what parallel test execution requires: Thread 1 has its Chrome instance, Thread 2 has its Firefox instance, and they never interfere with each other.

```
Thread 1 (Scenario: Login Test)         Thread 2 (Scenario: Search Test)
---------------------------------       ---------------------------------
ThreadLocal.set(chromeDriver)           ThreadLocal.set(firefoxDriver)
ThreadLocal.get() → chromeDriver        ThreadLocal.get() → firefoxDriver
```

Without `ThreadLocal`, a static `WebDriver` field would be shared:

```
Static WebDriver                         ← shared between ALL threads
Thread 1 calls driver.get("/login")      ← Thread 2 also reads this
Thread 2 calls driver.get("/search")     ← overwrites Thread 1's navigation
Test results become unpredictable
```

---

## DriverManager Implementation

`DriverManager` is a utility class that wraps the `ThreadLocal`. It has no instance — all methods are static.

```java
package com.company.framework.driver;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.WebDriver;

public class DriverManager {

    private static final Logger log = LogManager.getLogger(DriverManager.class);

    // One WebDriver per thread
    private static final ThreadLocal<WebDriver> driverThreadLocal = new ThreadLocal<>();

    // Private constructor — this class must never be instantiated
    private DriverManager() {
        throw new IllegalStateException("DriverManager is a static utility class");
    }

    /**
     * Returns the WebDriver instance for the current thread.
     * Returns null if no driver has been initialized for this thread.
     */
    public static WebDriver getDriver() {
        return driverThreadLocal.get();
    }

    /**
     * Stores a WebDriver instance for the current thread.
     * Called once per scenario from the @Before hook.
     */
    public static void setDriver(WebDriver driver) {
        log.info("Setting WebDriver for thread: {}", Thread.currentThread().getName());
        driverThreadLocal.set(driver);
    }

    /**
     * Quits the driver and removes the ThreadLocal binding.
     * Removing is critical — ThreadLocal values in thread pools persist
     * indefinitely if not removed, causing memory leaks.
     */
    public static void quitDriver() {
        WebDriver driver = driverThreadLocal.get();
        if (driver != null) {
            log.info("Quitting WebDriver for thread: {}", Thread.currentThread().getName());
            driver.quit();
            driverThreadLocal.remove();
        }
    }

    /**
     * Returns true if a driver is initialized for the current thread.
     */
    public static boolean isDriverInitialized() {
        return driverThreadLocal.get() != null;
    }
}
```

---

## DriverFactory Implementation

`DriverFactory` is responsible for creating browser-specific `WebDriver` instances. It reads the desired browser from configuration and returns a fully configured driver.

```java
package com.company.framework.driver;

import com.company.framework.config.ConfigReader;
import io.github.bonigarcia.wdm.WebDriverManager;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.edge.EdgeDriver;
import org.openqa.selenium.edge.EdgeOptions;
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.firefox.FirefoxOptions;

import java.time.Duration;

public class DriverFactory {

    private static final Logger log = LogManager.getLogger(DriverFactory.class);

    private DriverFactory() {
        throw new IllegalStateException("DriverFactory is a static utility class");
    }

    /**
     * Creates a WebDriver instance for the browser specified in configuration.
     * The browser can be overridden at runtime with -Dbrowser=firefox.
     */
    public static WebDriver createDriver() {
        // System property overrides config file
        String browser = System.getProperty("browser",
            ConfigReader.get("browser", "chrome")).toLowerCase();

        log.info("Creating WebDriver for browser: {}", browser);

        WebDriver driver;

        switch (browser) {
            case "firefox":
                driver = createFirefoxDriver();
                break;
            case "edge":
                driver = createEdgeDriver();
                break;
            case "chrome-headless":
                driver = createChromeHeadlessDriver();
                break;
            case "firefox-headless":
                driver = createFirefoxHeadlessDriver();
                break;
            case "chrome":
            default:
                driver = createChromeDriver();
                break;
        }

        configureTimeouts(driver);
        return driver;
    }

    // -----------------------------------------------------------------------
    // Browser-specific factory methods
    // -----------------------------------------------------------------------

    private static WebDriver createChromeDriver() {
        WebDriverManager.chromedriver().setup();
        ChromeOptions options = buildChromeOptions(false);
        return new ChromeDriver(options);
    }

    private static WebDriver createChromeHeadlessDriver() {
        WebDriverManager.chromedriver().setup();
        ChromeOptions options = buildChromeOptions(true);
        return new ChromeDriver(options);
    }

    private static WebDriver createFirefoxDriver() {
        WebDriverManager.firefoxdriver().setup();
        FirefoxOptions options = buildFirefoxOptions(false);
        return new FirefoxDriver(options);
    }

    private static WebDriver createFirefoxHeadlessDriver() {
        WebDriverManager.firefoxdriver().setup();
        FirefoxOptions options = buildFirefoxOptions(true);
        return new FirefoxDriver(options);
    }

    private static WebDriver createEdgeDriver() {
        WebDriverManager.edgedriver().setup();
        EdgeOptions options = new EdgeOptions();
        options.addArguments("--start-maximized");
        options.addArguments("--disable-notifications");
        return new EdgeDriver(options);
    }

    // -----------------------------------------------------------------------
    // Options builders
    // -----------------------------------------------------------------------

    private static ChromeOptions buildChromeOptions(boolean headless) {
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--start-maximized");
        options.addArguments("--disable-notifications");
        options.addArguments("--disable-infobars");
        options.addArguments("--disable-extensions");
        options.addArguments("--no-sandbox");
        options.addArguments("--disable-dev-shm-usage");

        if (headless) {
            options.addArguments("--headless=new");
            options.addArguments("--window-size=1920,1080");
        }

        // Allow file downloads in headless mode
        if (headless) {
            options.addArguments("--allow-running-insecure-content");
        }

        return options;
    }

    private static FirefoxOptions buildFirefoxOptions(boolean headless) {
        FirefoxOptions options = new FirefoxOptions();

        if (headless) {
            options.addArguments("-headless");
        }

        return options;
    }

    // -----------------------------------------------------------------------
    // Timeout configuration
    // -----------------------------------------------------------------------

    private static void configureTimeouts(WebDriver driver) {
        int implicitWait = Integer.parseInt(
            ConfigReader.get("timeout.implicit", "0")
        );
        int pageLoad = Integer.parseInt(
            ConfigReader.get("timeout.pageload", "30")
        );
        int scriptTimeout = Integer.parseInt(
            ConfigReader.get("timeout.script", "30")
        );

        driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(implicitWait));
        driver.manage().timeouts().pageLoadTimeout(Duration.ofSeconds(pageLoad));
        driver.manage().timeouts().scriptTimeout(Duration.ofSeconds(scriptTimeout));

        log.debug("Timeouts configured — implicit: {}s, pageLoad: {}s, script: {}s",
            implicitWait, pageLoad, scriptTimeout);
    }
}
```

---

## Integrating DriverManager with Hooks

The Hooks class calls `DriverFactory.createDriver()` in `@Before` and stores the result in `DriverManager`. This is the only place driver creation occurs.

```java
package com.company.framework.hooks;

import com.company.framework.driver.DriverFactory;
import com.company.framework.driver.DriverManager;
import io.cucumber.java.After;
import io.cucumber.java.Before;
import io.cucumber.java.Scenario;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;

public class Hooks {

    private static final Logger log = LogManager.getLogger(Hooks.class);

    @Before(order = 1)
    public void initializeDriver(Scenario scenario) {
        log.info("Starting scenario: [{}]", scenario.getName());
        DriverManager.setDriver(DriverFactory.createDriver());
    }

    @After(order = 2)
    public void captureScreenshotOnFailure(Scenario scenario) {
        if (scenario.isFailed() && DriverManager.isDriverInitialized()) {
            try {
                byte[] screenshot = ((TakesScreenshot) DriverManager.getDriver())
                    .getScreenshotAs(OutputType.BYTES);
                scenario.attach(screenshot, "image/png", scenario.getName());
                log.warn("Screenshot captured for failed scenario: {}", scenario.getName());
            } catch (Exception e) {
                log.error("Failed to capture screenshot: {}", e.getMessage());
            }
        }
    }

    @After(order = 1)
    public void tearDownDriver(Scenario scenario) {
        log.info("Scenario [{}] finished with status: {}", scenario.getName(), scenario.getStatus());
        DriverManager.quitDriver();
    }
}
```

---

## Accessing the Driver in Page Classes

Page classes receive the driver through their constructor. They never call `DriverManager.getDriver()` directly. Dependency injection through the constructor is the correct approach.

```java
// BasePage receives the driver via constructor
public abstract class BasePage {

    protected WebDriver driver;
    protected WebDriverWait wait;

    public BasePage(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
        PageFactory.initElements(driver, this);
    }
}

// Step definition creates the page with the driver from DriverManager
@Given("the user is on the login page")
public void theUserIsOnLoginPage() {
    loginPage = new LoginPage(DriverManager.getDriver());
}
```

---

## Accessing the Driver via ScenarioContext (World Object Pattern)

When using the World Object / PicoContainer pattern, the driver is stored on the context instead of being fetched from DriverManager directly in each step.

```java
// ScenarioContext holds the driver
public class ScenarioContext {
    private WebDriver driver;
    public WebDriver getDriver() { return driver; }
    public void setDriver(WebDriver driver) { this.driver = driver; }
}

// Hooks sets the driver on the context
public class Hooks {

    private final ScenarioContext context;

    public Hooks(ScenarioContext context) {
        this.context = context;
    }

    @Before(order = 1)
    public void initializeDriver() {
        WebDriver driver = DriverFactory.createDriver();
        DriverManager.setDriver(driver);        // ThreadLocal for parallel safety
        context.setDriver(driver);              // Context for step definition access
    }

    @After(order = 1)
    public void tearDown() {
        DriverManager.quitDriver();
    }
}

// Step definition uses the context
public class LoginSteps {

    private final ScenarioContext context;

    public LoginSteps(ScenarioContext context) {
        this.context = context;
    }

    @Given("the user is on the login page")
    public void theUserIsOnLoginPage() {
        context.setLoginPage(new LoginPage(context.getDriver()));
    }
}
```

---

## Remote WebDriver for Selenium Grid

When running against Selenium Grid or cloud providers like BrowserStack or Sauce Labs, `DriverFactory` creates a `RemoteWebDriver` instead of a local driver.

```java
private static WebDriver createRemoteDriver(String browser) throws MalformedURLException {
    String gridUrl = ConfigReader.get("grid.url", "http://localhost:4444");

    DesiredCapabilities capabilities = new DesiredCapabilities();
    capabilities.setBrowserName(browser);

    if (ConfigReader.get("platform") != null) {
        capabilities.setPlatform(Platform.fromString(ConfigReader.get("platform")));
    }

    log.info("Connecting to Grid at: {}", gridUrl);
    return new RemoteWebDriver(new URL(gridUrl), capabilities);
}
```

Add `remote` as a browser option in `DriverFactory.createDriver()`:

```java
case "remote":
    try {
        driver = createRemoteDriver(ConfigReader.get("remote.browser", "chrome"));
    } catch (MalformedURLException e) {
        throw new RuntimeException("Invalid Selenium Grid URL", e);
    }
    break;
```

---

## Driver Manager in Parallel Execution

When TestNG runs scenarios in parallel (`parallel="tests"` or `parallel="methods"` in `testng.xml`), each thread executes a scenario independently. `ThreadLocal` ensures they do not share a driver.

Key requirements for parallel-safe Driver Manager:

1. **Never use `static WebDriver`** — static fields are shared across threads.
2. **Always call `driverThreadLocal.remove()`** after `driver.quit()` — prevents memory leaks in thread pools.
3. **Do not pass `WebDriver` through static setters called from multiple threads simultaneously** — each thread must call `DriverManager.setDriver()` independently.
4. **Page objects receive the driver from the calling thread** — they must not retrieve it from a shared location.

```java
// Thread-safe usage — each thread manages its own driver
@Before
public void setUp() {
    DriverManager.setDriver(DriverFactory.createDriver()); // per-thread
}

@After
public void tearDown() {
    DriverManager.quitDriver(); // removes per-thread entry
}
```

---

## Complete Driver Initialization Sequence

```
TestNG starts Thread 1 for Scenario A
  │
  ├── @Before fires in Thread 1
  │     └── DriverFactory.createDriver() → new ChromeDriver()
  │     └── DriverManager.setDriver(chromeDriver)   ← stored in Thread 1's ThreadLocal slot
  │
  ├── @Given fires → LoginSteps.theUserIsOnLoginPage()
  │     └── new LoginPage(DriverManager.getDriver())  ← Thread 1 gets its own chromeDriver
  │
  ├── @When fires → LoginSteps.theUserLogsIn()
  │     └── loginPage.login(...)  ← uses Thread 1's chromeDriver
  │
  ├── @Then fires → assertions
  │
  └── @After fires
        └── DriverManager.quitDriver()
              └── chromeDriver.quit()
              └── driverThreadLocal.remove()   ← Thread 1's slot is cleared
```

Meanwhile, Thread 2 follows the exact same sequence with its own FirefoxDriver instance, completely independently.

---

## Summary

The Driver Manager is the backbone of thread-safe WebDriver lifecycle management. `DriverManager` provides a `ThreadLocal` container that isolates each thread's driver instance. `DriverFactory` centralizes browser creation, options configuration, timeout setup, and remote driver support. Together they ensure that every scenario — whether running sequentially or in parallel — has a clean, properly configured browser that is reliably created before the scenario and reliably closed after it.

---

## Related Topics

- ProjectStructure.md — Where DriverManager and DriverFactory live in the framework
- Hooks.md — The @Before and @After lifecycle that calls DriverManager
- ConfigProperties.md — Browser and timeout configuration read by DriverFactory
- ParallelExecution.md — Parallel test setup that depends on ThreadLocal
- SeleniumGrid.md — Remote WebDriver configuration for Grid execution
