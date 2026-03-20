# Allure Reports

## Overview

Allure is an open-source test reporting framework developed by Qameta Software that produces highly visual, interactive test reports suitable for both technical teams and non-technical stakeholders. Unlike Extent Reports, which generates a single self-contained HTML file, Allure generates a directory of raw result files during test execution and then builds the final HTML report from those files using a separate command or CI plugin. This separation makes Allure the preferred reporting solution for CI/CD pipelines where the report is published as a build artifact or served through the Allure TestOps dashboard.

Allure integrates natively with Cucumber, JUnit, TestNG, and Maven through official adapters and plugins.

---

## Architecture: Two-Phase Reporting

Allure works in two distinct phases:

**Phase 1 — Result Collection (during test execution):**
Each test writes a small JSON result file to `target/allure-results/`. This happens automatically through the Allure Cucumber adapter. No HTML is generated yet.

**Phase 2 — Report Generation (after test execution):**
The Allure command-line tool or the Maven plugin reads the JSON result files and generates the full HTML report in `target/allure-report/`.

```
mvn test                         # generates target/allure-results/*.json
mvn allure:report                # builds target/allure-report/index.html
allure serve target/allure-results  # builds and opens report in browser (local dev)
```

---

## Maven Dependencies and Plugin

```xml
<properties>
    <allure.version>2.25.0</allure.version>
    <aspectj.version>1.9.21</aspectj.version>
</properties>

<dependencies>
    <!-- Allure + Cucumber 7 integration -->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-cucumber7-jvm</artifactId>
        <version>${allure.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Allure + TestNG integration (if using TestNG runner) -->
    <dependency>
        <groupId>io.qameta.allure</groupId>
        <artifactId>allure-testng</artifactId>
        <version>${allure.version}</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <!-- Maven Surefire — AspectJ weaver required for Allure annotations -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.5</version>
            <configuration>
                <argLine>
                    -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/${aspectj.version}/aspectjweaver-${aspectj.version}.jar"
                </argLine>
                <suiteXmlFiles>
                    <suiteXmlFile>testng.xml</suiteXmlFile>
                </suiteXmlFiles>
                <systemPropertyVariables>
                    <allure.results.directory>
                        ${project.build.directory}/allure-results
                    </allure.results.directory>
                </systemPropertyVariables>
            </configuration>
        </plugin>

        <!-- Allure Maven Plugin — generates report and serves it -->
        <plugin>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-maven</artifactId>
            <version>2.12.0</version>
            <configuration>
                <reportVersion>${allure.version}</reportVersion>
                <resultsDirectory>${project.build.directory}/allure-results</resultsDirectory>
                <reportDirectory>${project.build.directory}/allure-report</reportDirectory>
            </configuration>
        </plugin>

        <!-- AspectJ Weaver — required for @Step and @Attachment to work -->
        <dependency>
            <groupId>org.aspectj</groupId>
            <artifactId>aspectjweaver</artifactId>
            <version>${aspectj.version}</version>
        </dependency>
    </plugins>
</build>
```

---

## Registering the Allure Adapter with Cucumber

Add the Allure Cucumber adapter as a `plugin` in every runner class. It automatically registers for every Feature, Scenario, and Step — zero additional Java code required.

```java
package com.company.framework.runners;

import io.cucumber.testng.AbstractTestNGCucumberTests;
import io.cucumber.testng.CucumberOptions;

@CucumberOptions(
    features  = "src/test/resources/features",
    glue      = {"stepdefinitions", "hooks"},
    tags      = "@regression",
    plugin    = {
        "pretty",
        "io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm",
        "html:target/cucumber-reports/cucumber.html",
        "json:target/cucumber-reports/cucumber.json"
    },
    monochrome = true
)
public class RegressionRunner extends AbstractTestNGCucumberTests {
}
```

After adding this plugin, every test run writes Allure result files automatically.

---

## Generating and Viewing the Report

```bash
# Run tests (writes allure-results)
mvn test -Dcucumber.filter.tags="@regression"

# Generate the HTML report
mvn allure:report

# Open the report in the browser
mvn allure:serve

# Or serve existing results directly
allure serve target/allure-results
```

The report is available at `target/allure-report/index.html`. It is a multi-page HTML application that requires a web server to open — it cannot be opened as a local file due to browser security restrictions. Use `allure serve` or the Jenkins/GitHub Actions Allure plugin.

---

## Allure Annotations

Allure provides a rich annotation set that enriches report metadata without modifying Gherkin. These annotations are applied to step definition classes and methods.

### @Epic, @Feature, @Story

Map to the Allure report's sidebar hierarchy. They allow BDD-level grouping independent of Cucumber tags.

```java
import io.qameta.allure.Epic;
import io.qameta.allure.Feature;
import io.qameta.allure.Story;

@Epic("E-Commerce Platform")
@Feature("User Authentication")
public class LoginSteps {

    @Story("Login with valid credentials")
    @When("the user logs in with username {string} and password {string}")
    public void theUserLogsIn(String username, String password) {
        // ...
    }
}
```

### @Step

Marks a Java method as a named step in the report. When used in helper or utility methods called from step definitions, `@Step` makes those lower-level calls visible in the report's step tree.

```java
import io.qameta.allure.Step;

public class LoginPage extends BasePage {

    @Step("Enter username: {username}")
    public void enterUsername(String username) {
        type(usernameField, username);
    }

    @Step("Enter password")
    public void enterPassword(String password) {
        // password value is intentionally not shown in step name
        type(passwordField, password);
    }

    @Step("Click the login button")
    public DashboardPage clickLogin() {
        click(loginButton);
        return new DashboardPage(driver);
    }
}
```

The `{username}` in the step name is a parameter reference — Allure replaces it with the actual value at runtime.

### @Description

Adds a description block to a test node in the report.

```java
@Description("Verifies that a user with valid credentials is redirected to the dashboard and receives a welcome message.")
@When("the user logs in with username {string} and password {string}")
public void theUserLogsIn(String username, String password) {
    // ...
}
```

### @Severity

Labels the test with a severity level visible in the report's priority breakdown.

```java
import io.qameta.allure.Severity;
import io.qameta.allure.SeverityLevel;

@Severity(SeverityLevel.CRITICAL)
@When("the user completes checkout")
public void theUserCompletesCheckout() { ... }

@Severity(SeverityLevel.MINOR)
@When("the user updates their display name")
public void theUserUpdatesDisplayName() { ... }
```

Available severity levels: `BLOCKER`, `CRITICAL`, `NORMAL`, `MINOR`, `TRIVIAL`.

### @Owner

Assigns ownership of the test, visible in the report's owner breakdown.

```java
@Owner("jane.smith@company.com")
@When("the payment gateway processes the transaction")
public void thePaymentGatewayProcessesTransaction() { ... }
```

### @Link, @Issue, @TmsLink

Attach hyperlinks to issues or test management system items.

```java
import io.qameta.allure.Issue;
import io.qameta.allure.Link;
import io.qameta.allure.TmsLink;

@Issue("AUTH-4421")
@TmsLink("TC-1001")
@Story("Login with locked account shows appropriate error")
@When("a locked user attempts to log in")
public void lockedUserAttemptsLogin() { ... }
```

Configure the base URL for issue and TMS links in `allure.properties`:

```properties
allure.link.issue.pattern=https://jira.company.com/browse/{}
allure.link.tms.pattern=https://testrail.company.com/index.php?/cases/view/{}
```

---

## allure.properties Configuration

Create `src/test/resources/allure.properties`:

```properties
# Results output directory
allure.results.directory=target/allure-results

# Issue tracker base URL (used by @Issue annotation)
allure.link.issue.pattern=https://jira.company.com/browse/{}

# TMS base URL (used by @TmsLink annotation)
allure.link.tms.pattern=https://testrail.company.com/index.php?/cases/view/{}

# Custom report title
allure.report.name=Selenium Automation Report
```

---

## Attaching Screenshots to Allure

### Automatic Attachment in Hooks

```java
package com.company.framework.hooks;

import io.cucumber.java.After;
import io.cucumber.java.Scenario;
import io.qameta.allure.Allure;
import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;

import java.io.ByteArrayInputStream;

public class AllureHooks {

    @After
    public void attachScreenshotOnFailure(Scenario scenario) {
        if (scenario.isFailed()) {
            try {
                byte[] screenshot = ((TakesScreenshot) DriverManager.getDriver())
                    .getScreenshotAs(OutputType.BYTES);

                Allure.addAttachment(
                    "Failure Screenshot — " + scenario.getName(),
                    "image/png",
                    new ByteArrayInputStream(screenshot),
                    ".png"
                );

                Allure.addAttachment(
                    "URL at Failure",
                    "text/plain",
                    DriverManager.getDriver().getCurrentUrl()
                );

            } catch (Exception e) {
                log.error("Failed to attach screenshot to Allure: {}", e.getMessage());
            }
        }
    }
}
```

### Attaching Screenshots Programmatically from Steps

```java
import io.qameta.allure.Allure;

// Attach screenshot at any point
public static void attachScreenshot(WebDriver driver, String name) {
    byte[] screenshot = ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
    Allure.addAttachment(name, "image/png",
        new ByteArrayInputStream(screenshot), ".png");
}

// Attach page source for debugging
public static void attachPageSource(WebDriver driver) {
    Allure.addAttachment("Page Source",
        "text/html", driver.getPageSource(), ".html");
}

// Attach a text log
public static void attachTextLog(String name, String content) {
    Allure.addAttachment(name, "text/plain", content, ".txt");
}
```

### Using @Attachment Annotation

```java
import io.qameta.allure.Attachment;

@Attachment(value = "Failure Screenshot", type = "image/png", fileExtension = ".png")
public byte[] takeScreenshot() {
    return ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);
}

@Attachment(value = "Console Logs", type = "text/plain")
public String getConsoleLogs() {
    return driver.manage().logs().get("browser").toString();
}
```

---

## Environment Information

Allure displays environment details in the report's Environment widget. Create `target/allure-results/environment.properties` before or after the test run:

```java
package com.company.framework.reports;

import com.company.framework.config.ConfigReader;
import java.io.*;
import java.util.Properties;

public class AllureEnvironmentWriter {

    public static void writeEnvironmentProperties() {
        Properties props = new Properties();
        props.setProperty("Environment", System.getProperty("env", "dev").toUpperCase());
        props.setProperty("Browser",     ConfigReader.get("browser", "chrome").toUpperCase());
        props.setProperty("Base URL",    ConfigReader.get("app.base.url", "N/A"));
        props.setProperty("Java Version", System.getProperty("java.version"));
        props.setProperty("OS",           System.getProperty("os.name"));
        props.setProperty("Branch",       System.getenv().getOrDefault("GIT_BRANCH", "local"));

        File resultsDir = new File("target/allure-results");
        resultsDir.mkdirs();

        try (FileOutputStream fos = new FileOutputStream(
                new File(resultsDir, "environment.properties"))) {
            props.store(fos, "Allure Environment Info");
        } catch (IOException e) {
            // log warning
        }
    }
}
```

Call this from `@BeforeAll` or `@AfterAll`:

```java
@AfterAll
public static void writeAllureEnvironment() {
    AllureEnvironmentWriter.writeEnvironmentProperties();
}
```

---

## Allure Report Sections

| Section    | Contents                                                       |
| ---------- | -------------------------------------------------------------- |
| Overview   | Summary: total, passed, failed, broken, skipped, unknown       |
| Suites     | Test tree grouped by suite/class                               |
| Behaviors  | Tests grouped by Epic → Feature → Story                        |
| Categories | Custom failure categorization (custom rules or defaults)       |
| Timeline   | Execution timeline showing parallel test threads               |
| Packages   | Java package hierarchy of step definitions                     |
| Graphs     | Severity distribution, status distribution, duration histogram |

---

## Failure Categorization

Allure categorizes failures into `Product Defects` (assertion failures), `Test Defects` (infrastructure failures), and custom categories. Configure in `categories.json` placed in `src/test/resources/`:

```json
[
  {
    "name": "Product Defects",
    "matchedStatuses": ["failed"],
    "messageRegex": ".*AssertionError.*"
  },
  {
    "name": "Test Infrastructure Issues",
    "matchedStatuses": ["broken"],
    "messageRegex": ".*NoSuchElementException.*|.*TimeoutException.*|.*StaleElementReferenceException.*"
  },
  {
    "name": "Known Login Issues",
    "matchedStatuses": ["failed"],
    "traceRegex": ".*LoginPage.*",
    "messageRegex": ".*Expected.*but was.*"
  }
]
```

Copy this file to `target/allure-results/` before generating the report:

```java
@BeforeAll
public static void copyCategories() throws IOException {
    Files.copy(
        Paths.get("src/test/resources/categories.json"),
        Paths.get("target/allure-results/categories.json"),
        StandardCopyOption.REPLACE_EXISTING
    );
}
```

---

## CI/CD Integration

### GitHub Actions

```yaml
- name: Run Tests
  run: mvn test -Dheadless=true

- name: Generate Allure Report
  run: mvn allure:report
  if: always()

- name: Upload Allure Results
  uses: actions/upload-artifact@v4
  if: always()
  with:
    name: allure-results-${{ github.run_number }}
    path: target/allure-results/
    retention-days: 30

- name: Upload Allure HTML Report
  uses: actions/upload-artifact@v4
  if: always()
  with:
    name: allure-report-${{ github.run_number }}
    path: target/allure-report/
```

### GitHub Pages Allure Deployment

```yaml
- name: Deploy to GitHub Pages
  uses: peaceiris/actions-gh-pages@v3
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: target/allure-report
```

### Jenkins

```groovy
pipeline {
    agent any
    stages {
        stage('Test') {
            steps {
                sh 'mvn test -Dheadless=true'
            }
        }
    }
    post {
        always {
            allure([
                includeProperties: false,
                jdk: '',
                properties: [],
                reportBuildPolicy: 'ALWAYS',
                results: [[path: 'target/allure-results']]
            ])
        }
    }
}
```

Jenkins requires the Allure Jenkins Plugin installed. The plugin automatically generates and publishes the report as a build action.

---

## Allure vs Extent Reports

| Aspect             | Allure                            | Extent Reports     |
| ------------------ | --------------------------------- | ------------------ |
| Report format      | Multi-page HTML app               | Single HTML file   |
| CI/CD native       | Yes (dedicated Jenkins/GH plugin) | Via HTML publisher |
| Self-contained     | No (needs web server)             | Yes                |
| Trend tracking     | Yes (history across runs)         | No (per-run only)  |
| Step annotations   | @Step — method-level              | Manual log() calls |
| BDD hierarchy      | @Epic/@Feature/@Story             | Category tags      |
| Failure categories | Built-in + configurable           | Exception view     |
| Timeline view      | Yes                               | No                 |
| Setup effort       | Higher                            | Lower              |

Use Allure when you need historical trend tracking, a centralized dashboard (Allure TestOps), or native Jenkins/GitHub integration. Use Extent Reports when you need a single portable HTML file shareable via email or Slack.

---

## Summary

Allure Reports provides the most comprehensive test reporting experience in the Java ecosystem. The two-phase architecture separates result collection from report generation, making it CI-native. The Cucumber adapter auto-generates results for every feature, scenario, and step. Allure's annotation set (`@Epic`, `@Feature`, `@Story`, `@Step`, `@Severity`, `@Issue`, `@TmsLink`) enriches reports with business context. The report itself provides behavior-driven views, severity graphs, execution timelines, and failure categorization that make diagnosing and communicating test results fast and clear.

---

## Related Topics

- ExtentReports.md — Single-file HTML reporting alternative
- CucumberHTMLReports.md — Cucumber's native reporting plugin
- ScreenshotsInReports.md — Screenshot attachment for Allure and Extent
- Hooks.md — @After hook for screenshot attachment on failure
- Jenkins.md — Allure Jenkins Plugin configuration
