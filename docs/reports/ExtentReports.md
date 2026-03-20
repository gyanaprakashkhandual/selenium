# Extent Reports

## Overview

Extent Reports is the most widely used HTML reporting library in the Java Selenium ecosystem. It generates rich, interactive, self-contained HTML reports that display scenario results, step-level details, embedded screenshots, log messages, system information, and execution timelines. Unlike Cucumber's built-in HTML reports, Extent Reports is fully customizable, works independently of the test framework, and produces professional-grade output suitable for sharing with stakeholders and archiving in CI/CD artifact storage.

Extent Reports 5 (the current major version) supports Spark Reporter for HTML, PDF Reporter for document output, and Klov Reporter for a centralized dashboard. This document focuses on the Spark Reporter, which is the standard choice for Selenium Cucumber frameworks.

---

## Maven Dependencies

```xml
<!-- Extent Reports 5 -->
<dependency>
    <groupId>com.aventstack</groupId>
    <artifactId>extentreports</artifactId>
    <version>5.1.1</version>
</dependency>

<!-- Extent Reports Cucumber 7 Adapter — auto-generates report from Cucumber events -->
<dependency>
    <groupId>tech.grasshopper</groupId>
    <artifactId>extentreports-cucumber7-adapter</artifactId>
    <version>1.14.0</version>
    <scope>test</scope>
</dependency>
```

---

## Approach 1 — Cucumber Adapter (Recommended)

The Cucumber adapter integrates Extent Reports directly into Cucumber's event pipeline. It creates test nodes automatically for every Feature, Scenario, and Step — no manual code required in Hooks or step definitions. You only need two configuration files.

### Step 1 — extent.properties

Create `src/test/resources/extent.properties`:

```properties
# Report output path
extent.reporter.spark.start=true
extent.reporter.spark.out=target/extent-reports/SparkReport.html

# Optional PDF report
extent.reporter.pdf.start=false
extent.reporter.pdf.out=target/extent-reports/PdfReport.pdf

# Screenshot path (relative to report file location)
extent.reporter.spark.vieworder=dashboard,test,category,exception,author,device,log

# System info
systeminfo.Project=Selenium Automation Framework
systeminfo.Environment=${env}
systeminfo.Browser=${browser}
systeminfo.Tester=QA Team

# Screenshot embedding
screenshot.dir=target/extent-reports/screenshots/
screenshot.rel.path=screenshots/
```

### Step 2 — @CucumberOptions plugin Registration

Add the adapter to the `plugin` list in every runner class:

```java
@CucumberOptions(
    features  = "src/test/resources/features",
    glue      = {"stepdefinitions", "hooks"},
    tags      = "@regression",
    plugin    = {
        "pretty",
        "html:target/cucumber-reports/cucumber.html",
        "json:target/cucumber-reports/cucumber.json",
        "com.aventstack.extentreports.cucumber.adapter.ExtentCucumberAdapter:"
    },
    monochrome = true
)
public class RegressionRunner extends AbstractTestNGCucumberTests {
}
```

The trailing `:` after the adapter class name is required.

That is all. Run the tests. The Spark HTML report is generated automatically at `target/extent-reports/SparkReport.html`.

---

## Approach 2 — Manual Integration via Hooks

Manual integration gives full control over how tests are logged, what metadata is attached, and when the report is written. Use this when you need custom log entries at each step, conditional screenshot embedding, or additional metadata per scenario.

### ExtentReportManager.java

```java
package com.company.framework.reports;

import com.aventstack.extentreports.ExtentReports;
import com.aventstack.extentreports.ExtentTest;
import com.aventstack.extentreports.reporter.ExtentSparkReporter;
import com.aventstack.extentreports.reporter.configuration.Theme;
import com.company.framework.config.ConfigReader;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class ExtentReportManager {

    private static final Logger log = LogManager.getLogger(ExtentReportManager.class);

    private static ExtentReports extent;
    private static ThreadLocal<ExtentTest> testThread = new ThreadLocal<>();

    private static final String REPORT_DIR  = "target/extent-reports/";
    private static final String REPORT_FILE = REPORT_DIR + "SparkReport.html";

    private ExtentReportManager() {}

    // -----------------------------------------------------------------------
    // Initialization — called once before all tests
    // -----------------------------------------------------------------------

    public static synchronized ExtentReports getInstance() {
        if (extent == null) {
            ExtentSparkReporter spark = new ExtentSparkReporter(REPORT_FILE);
            configureSparkReporter(spark);

            extent = new ExtentReports();
            extent.attachReporter(spark);
            addSystemInfo(extent);

            log.info("ExtentReports initialized. Output: {}", REPORT_FILE);
        }
        return extent;
    }

    private static void configureSparkReporter(ExtentSparkReporter spark) {
        spark.config().setTheme(Theme.DARK);
        spark.config().setDocumentTitle("Selenium Automation Report");
        spark.config().setReportName("Test Execution Results — "
            + LocalDateTime.now().format(DateTimeFormatter.ofPattern("dd-MM-yyyy HH:mm")));
        spark.config().setEncoding("UTF-8");
        spark.config().setTimeStampFormat("dd MMM yyyy HH:mm:ss");
        spark.config().setJs(
            "document.getElementById('nav-logo').style.display='none';"
        );
    }

    private static void addSystemInfo(ExtentReports extent) {
        extent.setSystemInfo("Project",     "Selenium Framework");
        extent.setSystemInfo("Environment", System.getProperty("env", "dev").toUpperCase());
        extent.setSystemInfo("Browser",     ConfigReader.get("browser", "chrome").toUpperCase());
        extent.setSystemInfo("OS",          System.getProperty("os.name"));
        extent.setSystemInfo("Java",        System.getProperty("java.version"));
        extent.setSystemInfo("Base URL",    ConfigReader.get("app.base.url", "N/A"));
        extent.setSystemInfo("Tester",      System.getProperty("user.name"));
    }

    // -----------------------------------------------------------------------
    // Per-test — called from @Before / @After hooks
    // -----------------------------------------------------------------------

    public static ExtentTest createTest(String scenarioName, String... tags) {
        ExtentTest test = getInstance().createTest(scenarioName);
        for (String tag : tags) {
            test.assignCategory(tag.replace("@", ""));
        }
        testThread.set(test);
        return test;
    }

    public static ExtentTest getTest() {
        return testThread.get();
    }

    public static void removeTest() {
        testThread.remove();
    }

    // -----------------------------------------------------------------------
    // Flush — called once after all tests
    // -----------------------------------------------------------------------

    public static synchronized void flushReport() {
        if (extent != null) {
            extent.flush();
            log.info("ExtentReports flushed. Report saved at: {}", REPORT_FILE);
        }
    }
}
```

### Hooks Integration

```java
package com.company.framework.hooks;

import com.aventstack.extentreports.ExtentTest;
import com.aventstack.extentreports.Status;
import com.aventstack.extentreports.MediaEntityBuilder;
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

import java.util.Base64;
import java.util.Collection;

public class Hooks {

    private static final Logger log = LogManager.getLogger(Hooks.class);

    @BeforeAll
    public static void suiteSetUp() {
        ExtentReportManager.getInstance(); // initialize report
        log.info("Test suite started");
    }

    @Before(order = 1)
    public void initDriver(Scenario scenario) {
        // ... driver initialization

        // Create Extent test node
        Collection<String> tags = scenario.getSourceTagNames();
        ExtentTest test = ExtentReportManager.createTest(
            scenario.getName(),
            tags.toArray(new String[0])
        );
        test.info("Scenario started: " + scenario.getName());
        test.info("Tags: " + tags);
    }

    @After(order = 2)
    public void reportResult(Scenario scenario) {
        ExtentTest test = ExtentReportManager.getTest();
        if (test == null) return;

        if (scenario.isFailed()) {
            try {
                // Capture screenshot as Base64 for inline embedding
                String base64Screenshot = ((TakesScreenshot) DriverManager.getDriver())
                    .getScreenshotAs(OutputType.BASE64);

                test.fail("Scenario FAILED: " + scenario.getName(),
                    MediaEntityBuilder.createScreenCaptureFromBase64String(
                        base64Screenshot, "Failure Screenshot").build());

                test.fail("URL at failure: " + DriverManager.getDriver().getCurrentUrl());

            } catch (Exception e) {
                test.fail("Scenario failed — screenshot capture also failed: " + e.getMessage());
                log.error("Screenshot capture failed: {}", e.getMessage());
            }

        } else {
            test.pass("Scenario PASSED: " + scenario.getName());
        }

        ExtentReportManager.removeTest();
    }

    @After(order = 1)
    public void tearDownDriver(Scenario scenario) {
        log.info("Scenario finished: {} — {}", scenario.getName(), scenario.getStatus());
        DriverManager.quitDriver();
    }

    @AfterAll
    public static void suiteTearDown() {
        ExtentReportManager.flushReport();
        log.info("Test suite completed");
    }
}
```

---

## Logging Steps in Extent Reports

For detailed step-level logging in the report, call `ExtentReportManager.getTest()` from step definitions and log each action.

```java
package com.company.framework.stepdefinitions;

import com.aventstack.extentreports.ExtentTest;
import com.aventstack.extentreports.Status;
import com.company.framework.reports.ExtentReportManager;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import org.testng.Assert;

public class LoginSteps {

    @Given("the user is on the login page")
    public void theUserIsOnLoginPage() {
        ExtentTest test = ExtentReportManager.getTest();
        // ... page setup
        test.log(Status.INFO, "Navigated to login page");
    }

    @When("the user logs in with username {string} and password {string}")
    public void theUserLogsIn(String username, String password) {
        ExtentTest test = ExtentReportManager.getTest();
        test.log(Status.INFO, "Logging in as: " + username);
        // ... login action
        test.log(Status.INFO, "Login form submitted");
    }

    @Then("the dashboard should be displayed")
    public void theDashboardShouldBeDisplayed() {
        ExtentTest test = ExtentReportManager.getTest();
        boolean loaded = context.getDashboardPage().isLoaded();
        if (loaded) {
            test.pass("Dashboard loaded successfully");
        } else {
            test.fail("Dashboard did not load");
            Assert.fail("Dashboard page was not loaded");
        }
    }
}
```

---

## Report Configuration — extent-config.xml (Advanced)

For the Adapter approach, you can also use `extent-config.xml` in `src/test/resources/`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<extentreports>
    <configuration>
        <theme>dark</theme>
        <encoding>UTF-8</encoding>
        <protocol>https</protocol>
        <documentTitle>Automation Report</documentTitle>
        <reportName>Regression Execution Results</reportName>
        <thumbnailForBase64>true</thumbnailForBase64>
        <timeStampFormat>MMM dd, yyyy HH:mm:ss</timeStampFormat>
        <scripts>
            <![CDATA[
                $(document).ready(function() {
                    $(".brand-logo").hide();
                });
            ]]>
        </scripts>
    </configuration>
</extentreports>
```

---

## Report Structure and Features

A generated Spark Report contains these views:

**Dashboard View** — Executive summary with pass/fail/skip counts, a donut chart, time taken, and environment details.

**Test View** — Each scenario listed with expandable step-level detail, log entries, and embedded screenshots. Color-coded: green (pass), red (fail), orange (skip).

**Category View** — All scenarios grouped by their Cucumber tags. Clicking `@smoke` shows only smoke test results.

**Exception View** — All failures grouped by exception type. Immediately shows how many tests failed with `NoSuchElementException` vs `AssertionError`.

**Author View** — If you assign authors to tests, this view groups by author.

---

## Timestamped Reports

For CI pipelines where you want each run to produce a unique report file rather than overwriting the previous one:

```java
private static String buildReportPath() {
    String timestamp = LocalDateTime.now()
        .format(DateTimeFormatter.ofPattern("yyyy-MM-dd_HH-mm-ss"));
    return "target/extent-reports/SparkReport_" + timestamp + ".html";
}
```

Or configure in `extent.properties`:

```properties
extent.reporter.spark.out=target/extent-reports/SparkReport_%d{yyyy-MM-dd_HH-mm-ss}.html
```

---

## Integrating with CI/CD

### Publishing in GitHub Actions

```yaml
- name: Run Tests
  run: mvn test -Dheadless=true

- name: Upload Extent Report
  uses: actions/upload-artifact@v4
  if: always()
  with:
    name: extent-report-${{ github.run_number }}
    path: target/extent-reports/
    retention-days: 30
```

### Publishing in Jenkins

In your Jenkins pipeline:

```groovy
post {
    always {
        publishHTML([
            allowMissing: false,
            alwaysLinkToLastBuild: true,
            keepAll: true,
            reportDir: 'target/extent-reports',
            reportFiles: 'SparkReport.html',
            reportName: 'Extent Report'
        ])
    }
}
```

---

## Summary

Extent Reports produces the most visually detailed and stakeholder-friendly HTML reports in the Java Selenium ecosystem. The Cucumber adapter approach requires zero Java code changes — only a `plugin` entry and `extent.properties` file. The manual Hooks approach provides full control over log granularity and screenshot embedding. Either approach produces a self-contained HTML file with dashboard, scenario, category, and exception views that clearly communicate the health of every test run.

---

## Related Topics

- AllureReports.md — Alternative reporting with Allure for CI dashboards
- CucumberHTMLReports.md — Cucumber's built-in HTML report
- ScreenshotsInReports.md — Embedding screenshots in all report types
- Hooks.md — @Before/@After integration with report lifecycle
- Log4j.md — Log4j output that complements report entries
