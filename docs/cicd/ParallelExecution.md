# Parallel Execution

## Overview

Parallel execution runs multiple test scenarios simultaneously across multiple browser instances, dramatically reducing total test suite execution time. A regression suite that takes 60 minutes sequentially can complete in 10–15 minutes with 4–6 parallel threads. Parallel execution is essential for any CI/CD pipeline where fast feedback is required.

Implementing parallel execution correctly in a Selenium Cucumber framework requires careful management of three concerns: thread-safe WebDriver storage, independent scenario state, and resource-aware thread count configuration. Getting any one of these wrong produces intermittent failures that are difficult to diagnose.

---

## The Core Requirement — Thread Safety

Every scenario running in parallel needs its own independent browser session. If two parallel scenarios share a `WebDriver` instance, their navigation calls, element lookups, and assertions will interfere with each other, producing random, non-reproducible failures.

The solution is `ThreadLocal<WebDriver>` — Java's mechanism for storing a separate value per thread. This is covered in full in `DriverManager.md`. The contract is:

```
Thread 1 → DriverManager.setDriver(chromeDriver1)
Thread 2 → DriverManager.setDriver(chromeDriver2)

Thread 1 → DriverManager.getDriver() → chromeDriver1  (its own)
Thread 2 → DriverManager.getDriver() → chromeDriver2  (its own)
```

**No shared static state. No singleton driver. No driver passed between classes through static fields.**

---

## TestNG Parallel Execution

TestNG controls parallelism through the `testng.xml` file. It supports four parallel modes.

### Parallel Modes

| Mode        | Behavior                                                 |
| ----------- | -------------------------------------------------------- |
| `methods`   | Each `@Test` method runs in its own thread               |
| `tests`     | Each `<test>` block in testng.xml runs in its own thread |
| `classes`   | Each `<class>` element runs in its own thread            |
| `instances` | Each instance of a test class runs in its own thread     |

For Cucumber, the most effective mode is `tests` — each runner class (which maps to a tag subset) runs in a separate thread. Combined with multiple runner classes, this gives clean parallel execution with full isolation.

### testng.xml — Parallel Tests

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">

<suite name="Parallel Execution Suite"
       verbose="1"
       parallel="tests"
       thread-count="4"
       data-provider-thread-count="4">

    <listeners>
        <listener class-name="org.testng.reporters.SuiteHTMLReporter"/>
    </listeners>

    <test name="Smoke Tests">
        <classes>
            <class name="com.company.framework.runners.SmokeTestRunner"/>
        </classes>
    </test>

    <test name="Authentication Tests">
        <classes>
            <class name="com.company.framework.runners.AuthenticationRunner"/>
        </classes>
    </test>

    <test name="Product Tests">
        <classes>
            <class name="com.company.framework.runners.ProductRunner"/>
        </classes>
    </test>

    <test name="Checkout Tests">
        <classes>
            <class name="com.company.framework.runners.CheckoutRunner"/>
        </classes>
    </test>

</suite>
```

With `parallel="tests"` and `thread-count="4"`, all four runners execute simultaneously, each in its own thread.

### Scenario-Level Parallelism with AbstractTestNGCucumberTests

For finer-grained parallelism, override the data provider in the runner to run each scenario in its own thread:

```java
package com.company.framework.runners;

import io.cucumber.testng.AbstractTestNGCucumberTests;
import io.cucumber.testng.CucumberOptions;
import org.testng.annotations.DataProvider;

@CucumberOptions(
    features  = "src/test/resources/features",
    glue      = {"stepdefinitions", "hooks"},
    tags      = "@regression",
    plugin    = {
        "pretty",
        "json:target/cucumber-reports/regression.json",
        "junit:target/cucumber-reports/regression.xml",
        "rerun:target/cucumber-reports/rerun.txt",
        "io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm",
        "com.aventstack.extentreports.cucumber.adapter.ExtentCucumberAdapter:"
    },
    monochrome = true
)
public class RegressionRunner extends AbstractTestNGCucumberTests {

    /**
     * Overriding the default data provider to enable parallel scenario execution.
     * Each scenario row gets its own thread from the TestNG thread pool.
     */
    @DataProvider(parallel = true)
    @Override
    public Object[][] scenarios() {
        return super.scenarios();
    }
}
```

This is the most granular level — each individual scenario runs concurrently. The `thread-count` in `testng.xml` controls the maximum concurrent scenarios.

---

## Thread-Safe Hooks

With parallel execution, `@Before` and `@After` hooks fire on multiple threads simultaneously. Hooks must never access shared mutable state.

```java
package com.company.framework.hooks;

import com.company.framework.config.ConfigReader;
import com.company.framework.driver.DriverFactory;
import com.company.framework.driver.DriverManager;
import com.company.framework.reports.ExtentReportManager;
import io.cucumber.java.After;
import io.cucumber.java.AfterAll;
import io.cucumber.java.Before;
import io.cucumber.java.BeforeAll;
import io.cucumber.java.Scenario;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;

public class Hooks {

    private static final Logger log = LogManager.getLogger(Hooks.class);

    @BeforeAll
    public static void globalSetup() {
        // SAFE: runs once before all threads start
        ExtentReportManager.getInstance();
        log.info("Global setup complete. Thread count: {}",
            Runtime.getRuntime().availableProcessors());
    }

    @Before(order = 1)
    public void initDriver(Scenario scenario) {
        // SAFE: ThreadLocal isolates each thread's driver
        log.info("[Thread-{}] Starting: {}",
            Thread.currentThread().getId(), scenario.getName());

        DriverManager.setDriver(DriverFactory.createDriver());
        DriverManager.getDriver().get(ConfigReader.get("app.base.url"));
    }

    @After(order = 2)
    public void captureFailure(Scenario scenario) {
        // SAFE: getDriver() returns this thread's driver only
        if (scenario.isFailed() && DriverManager.isDriverInitialized()) {
            try {
                byte[] screenshot = ((TakesScreenshot) DriverManager.getDriver())
                    .getScreenshotAs(OutputType.BYTES);
                scenario.attach(screenshot, "image/png", scenario.getName());
                log.error("[Thread-{}] FAILED: {}",
                    Thread.currentThread().getId(), scenario.getName());
            } catch (Exception e) {
                log.error("Screenshot failed: {}", e.getMessage());
            }
        }
    }

    @After(order = 1)
    public void tearDown(Scenario scenario) {
        // SAFE: quitDriver() removes ThreadLocal entry after quitting
        log.info("[Thread-{}] Finished [{}]: {}",
            Thread.currentThread().getId(),
            scenario.getStatus(), scenario.getName());
        DriverManager.quitDriver();
    }

    @AfterAll
    public static void globalTearDown() {
        // SAFE: runs once after all threads complete
        ExtentReportManager.flushReport();
        log.info("All tests complete. Report flushed.");
    }
}
```

---

## Thread-Safe ScenarioContext (World Object)

When using PicoContainer for dependency injection, the `ScenarioContext` (World Object) must be completely isolated per scenario — no static or shared fields.

```java
package com.company.framework.context;

import org.openqa.selenium.WebDriver;
import pages.*;

/**
 * ScenarioContext holds all state for a single scenario execution.
 * PicoContainer creates a new instance per scenario — naturally thread-safe.
 * NEVER use static fields here. Each scenario gets its own instance.
 */
public class ScenarioContext {

    // WebDriver — set from the thread's ThreadLocal via Hooks
    private WebDriver driver;

    // Page objects — created fresh per scenario
    private LoginPage loginPage;
    private DashboardPage dashboardPage;
    private ProductsPage productsPage;
    private CartPage cartPage;
    private CheckoutPage checkoutPage;

    // Scenario-scoped captured data
    private String capturedOrderId;
    private String capturedEmail;
    private double capturedTotal;

    // All fields are instance-level — no static, no shared state

    public WebDriver getDriver()             { return driver; }
    public void setDriver(WebDriver v)       { this.driver = v; }

    public LoginPage getLoginPage()          { return loginPage; }
    public void setLoginPage(LoginPage v)    { this.loginPage = v; }

    public DashboardPage getDashboardPage()  { return dashboardPage; }
    public void setDashboardPage(DashboardPage v) { this.dashboardPage = v; }

    public ProductsPage getProductsPage()    { return productsPage; }
    public void setProductsPage(ProductsPage v) { this.productsPage = v; }

    public CartPage getCartPage()            { return cartPage; }
    public void setCartPage(CartPage v)      { this.cartPage = v; }

    public CheckoutPage getCheckoutPage()    { return checkoutPage; }
    public void setCheckoutPage(CheckoutPage v) { this.checkoutPage = v; }

    public String getCapturedOrderId()       { return capturedOrderId; }
    public void setCapturedOrderId(String v) { this.capturedOrderId = v; }

    public String getCapturedEmail()         { return capturedEmail; }
    public void setCapturedEmail(String v)   { this.capturedEmail = v; }

    public double getCapturedTotal()         { return capturedTotal; }
    public void setCapturedTotal(double v)   { this.capturedTotal = v; }
}
```

---

## Thread-Safe Extent Reports

The `ExtentReports` singleton must be thread-safe. The `ThreadLocal<ExtentTest>` ensures each thread's test node is independent.

```java
public class ExtentReportManager {

    // Singleton — initialized once, accessed by all threads safely (read-only after init)
    private static ExtentReports extent;

    // Each thread holds its own ExtentTest node
    private static final ThreadLocal<ExtentTest> testThread = new ThreadLocal<>();

    // synchronized to prevent double-initialization from multiple threads
    public static synchronized ExtentReports getInstance() {
        if (extent == null) {
            ExtentSparkReporter spark = new ExtentSparkReporter(REPORT_FILE);
            extent = new ExtentReports();
            extent.attachReporter(spark);
        }
        return extent;
    }

    // synchronized — ExtentReports.createTest is not thread-safe
    public static synchronized ExtentTest createTest(String name, String... tags) {
        ExtentTest test = getInstance().createTest(name);
        for (String tag : tags) test.assignCategory(tag.replace("@", ""));
        testThread.set(test);
        return test;
    }

    public static ExtentTest getTest() { return testThread.get(); }
    public static void removeTest()    { testThread.remove(); }

    // synchronized — flush is not thread-safe
    public static synchronized void flushReport() {
        if (extent != null) extent.flush();
    }
}
```

---

## Optimal Thread Count

The right thread count depends on available hardware and whether tests run locally or on Grid.

### Rule of Thumb

```
Local machine:     thread-count = number of CPU cores (e.g., 4 or 8)
CI server:         thread-count = available CPUs on agent (typically 4–8)
Selenium Grid:     thread-count = number of available Grid slots
Cloud Grid:        thread-count = subscription concurrent session limit
```

### Dynamic Thread Count from Configuration

```java
// In testng.xml, reference a system property
// <suite ... thread-count="${thread.count}">

// Run with:
// mvn test -Dthread.count=6
```

Or compute based on environment:

```java
// In a BeforeAll hook or test listener
int cores = Runtime.getRuntime().availableProcessors();
int threads = Math.max(1, cores - 1); // leave one core for the OS
System.setProperty("thread.count", String.valueOf(threads));
```

---

## Parallel Execution with Selenium Grid

Parallel local execution uses machine resources. Parallel Grid execution distributes load across nodes. Combine both for maximum throughput.

```xml
<!-- testng.xml for Grid parallel execution -->
<suite name="Grid Parallel Suite" parallel="tests" thread-count="8">

    <parameter name="grid.enabled" value="true"/>
    <parameter name="grid.url"     value="http://selenium-hub:4444"/>

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

</suite>
```

With this setup, the same regression suite runs simultaneously on Chrome and Firefox via the Grid, doubling cross-browser coverage without increasing calendar time.

---

## Test Independence — The Prerequisite for Parallelism

Parallel execution exposes every violation of test independence. Scenarios that share state, depend on execution order, or modify shared test data will fail unpredictably when run in parallel.

### Violations to Eliminate Before Enabling Parallelism

**Shared static state in page objects or step definitions:**

```java
// WRONG — static page reference shared across threads
public class LoginSteps {
    private static LoginPage loginPage; // will be overwritten by parallel threads
}

// CORRECT — instance field, one per scenario
public class LoginSteps {
    private final ScenarioContext context; // PicoContainer injects per-scenario instance
    public LoginSteps(ScenarioContext context) { this.context = context; }
}
```

**Shared test user accounts:**

Two parallel scenarios that both log in as `admin@example.com` and modify account settings will conflict. Solutions:

```java
// Option 1: Use separate users per scenario type
@Given("the admin user logs in")
public void adminLogsIn() {
    User admin = TestDataManager.getAdminUser(); // dedicated test account
}

// Option 2: Generate unique users per scenario
@When("a new user registers")
public void newUserRegisters() {
    String email = TestDataFactory.generateUniqueEmail(); // unique per call
    context.setCapturedEmail(email);
}
```

**Shared test data records:**

Two scenarios that both modify `ORDER-1001` will overwrite each other's changes. Use data factories to create isolated records, or tag scenarios with `@sequential` and use a tag-scoped hook to serialize them.

---

## Running Tests in Parallel on CI

### Maven Surefire Parallel Configuration

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
    <configuration>
        <suiteXmlFiles>
            <suiteXmlFile>testng.xml</suiteXmlFile>
        </suiteXmlFiles>
        <systemPropertyVariables>
            <grid.enabled>${grid.enabled}</grid.enabled>
            <grid.url>${grid.url}</grid.url>
            <browser>${browser}</browser>
            <env>${env}</env>
            <headless>true</headless>
        </systemPropertyVariables>
        <!-- Fork count for Maven-level parallelism -->
        <forkCount>1</forkCount>
        <reuseForks>true</reuseForks>
    </configuration>
</plugin>
```

### Command-Line Execution

```bash
# Run parallel regression suite with 6 threads against Grid
mvn test \
  -Dgrid.enabled=true \
  -Dgrid.url=http://localhost:4444 \
  -Dthread.count=6 \
  -Dheadless=true \
  -Dcucumber.filter.tags="@regression and not @wip"
```

---

## Parallel Execution Checklist

Before enabling parallel execution, verify every item:

- `ThreadLocal<WebDriver>` used in `DriverManager` — no static driver field
- `ScenarioContext` has no static fields — all instance-level
- `ExtentReportManager.createTest()` is synchronized
- No step definition class has static page object fields
- Test data does not use shared mutable records between parallel scenarios
- Each scenario cleans up data it creates in `@After` hooks
- The `@DataProvider(parallel = true)` override is added to all runner classes
- `thread-count` in `testng.xml` is set appropriately for the target environment
- Screenshots are captured via `DriverManager.getDriver()` (thread-local), not a shared reference

---

## Summary

Parallel execution multiplies test throughput by running multiple scenarios simultaneously across independent browser sessions. The foundation is `ThreadLocal<WebDriver>` in `DriverManager`, which ensures each thread has its own isolated driver. `ScenarioContext` with PicoContainer provides per-scenario state isolation between step definition classes. `ExtentReportManager` uses synchronized initialization and `ThreadLocal<ExtentTest>` for thread-safe reporting. Test independence — no shared accounts, no shared data records, no static state in step definitions — is the non-negotiable prerequisite for reliable parallel execution.

---

## Related Topics

- SeleniumGrid.md — Distributed Grid infrastructure for parallel execution
- DriverManager.md — ThreadLocal WebDriver implementation
- WorldObject.md — PicoContainer ScenarioContext for per-scenario state isolation
- Hooks.md — Thread-safe @Before and @After implementation
- Jenkins.md — Jenkins pipeline with parallel stage execution
