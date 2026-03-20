# Cucumber HTML Reports

## Overview

Cucumber provides several built-in reporting plugins that generate test output during execution without any additional libraries. These built-in reporters write feature results, scenario outcomes, step statuses, error messages, and execution metadata to HTML, JSON, and XML formats. They are the foundation of every Cucumber reporting strategy — even when Extent Reports or Allure are used, the Cucumber JSON output is often consumed as the data source for richer reports.

This document covers all native Cucumber reporters, the professional `cucumber-reporting` library that generates enhanced HTML from Cucumber JSON, report configuration, and multi-environment report publishing strategies.

---

## Built-in Cucumber Reporters

Cucumber reporters are registered in the `plugin` array of `@CucumberOptions`. Multiple reporters can be active simultaneously.

### Configuring Plugins

```java
@CucumberOptions(
    features  = "src/test/resources/features",
    glue      = {"stepdefinitions", "hooks"},
    tags      = "@regression",
    plugin    = {
        "pretty",                                              // console output
        "html:target/cucumber-reports/cucumber.html",         // basic HTML
        "json:target/cucumber-reports/cucumber.json",         // JSON for processing
        "junit:target/cucumber-reports/cucumber.xml",         // JUnit XML for CI
        "rerun:target/cucumber-reports/rerun.txt",            // failed scenario list
        "usage:target/cucumber-reports/usage.json"            // step usage analytics
    },
    monochrome = true
)
public class RegressionRunner extends AbstractTestNGCucumberTests {
}
```

---

## Plugin Reference

### pretty

Formats test output to the console with color-coded step results and full Gherkin text. This is the most readable console output for local development.

```
Feature: User Login

  Background:               # src/test/resources/features/login.feature:4
    Given the user is on the login page     # LoginSteps.theUserIsOnLoginPage()

  @smoke @positive
  Scenario: Successful login               # line 9
    When the user enters valid credentials  # LoginSteps.theUserEntersCredentials()
    Then the dashboard should be displayed  # LoginSteps.theDashboardShouldBeDisplayed()

  @negative
  Scenario: Login with wrong password      # line 15  *** FAILED ***
    When the user enters wrong credentials  # LoginSteps.theUserEntersCredentials()
    Then an error message should appear     # LoginSteps.theErrorShouldAppear()
      AssertionError: Expected error message but page showed dashboard
```

### html

Generates a basic single-page HTML report. It is functional and self-contained but minimal — no charts, no filtering, no step details beyond pass/fail. Useful as a lightweight archive.

```properties
plugin = "html:target/cucumber-reports/cucumber.html"
```

### json

Generates a detailed JSON file containing the complete execution data for every feature, scenario, and step. This output is consumed by:

- The `cucumber-reporting` library (see below)
- Allure Cucumber adapter
- Jenkins Cucumber Reports plugin
- Custom report processors
- Test result dashboards

Always include the JSON plugin — it is the most versatile output format.

```properties
plugin = "json:target/cucumber-reports/cucumber.json"
```

### junit

Generates JUnit-style XML output, compatible with every CI system that understands JUnit XML (Jenkins, GitHub Actions, GitLab CI, Azure DevOps, TeamCity). This is how CI systems display test counts and failure details in the build summary.

```properties
plugin = "junit:target/cucumber-reports/cucumber.xml"
```

In GitHub Actions, this enables the test results table in the build summary:

```yaml
- name: Publish JUnit Results
  uses: dorny/test-reporter@v1
  if: always()
  with:
    name: Cucumber Test Results
    path: target/cucumber-reports/cucumber.xml
    reporter: java-junit
```

### rerun

Writes the file paths and line numbers of every failed scenario to a text file. This file can be used as the `features` input to a re-run runner, executing only the previously failed scenarios.

```properties
plugin = "rerun:target/cucumber-reports/rerun.txt"
```

**Rerun Runner:**

```java
@CucumberOptions(
    features  = "@target/cucumber-reports/rerun.txt",  // @ prefix reads from file
    glue      = {"stepdefinitions", "hooks"},
    plugin    = {
        "pretty",
        "html:target/cucumber-reports/rerun-report.html",
        "json:target/cucumber-reports/rerun.json"
    }
)
public class RerunFailedRunner extends AbstractTestNGCucumberTests {
}
```

This is the standard pattern for flaky test management: run the full suite, then rerun only the failures once to confirm they are genuine defects rather than infrastructure flakiness.

### usage

Generates a JSON file containing statistics on step definition usage — how many times each step was called, average duration, and slowest invocations. Useful for identifying slow steps and unused step definitions.

```properties
plugin = "usage:target/cucumber-reports/usage.json"
```

### message (Cucumber Messages Protocol)

The `message` plugin writes a binary Cucumber Messages file. This is the protocol used by Cucumber's tooling ecosystem and is the basis for cucumber-html-formatter.

```properties
plugin = "message:target/cucumber-reports/cucumber.ndjson"
```

---

## cucumber-reporting — Professional HTML Reports

The `cucumber-reporting` library by Damian Szczepanik generates a significantly enhanced HTML report from the Cucumber JSON output. It includes charts, trend tracking, feature summaries, scenario breakdowns, and tag analysis.

### Maven Dependency

```xml
<dependency>
    <groupId>net.masterthought</groupId>
    <artifactId>cucumber-reporting</artifactId>
    <version>5.7.7</version>
</dependency>
```

### Report Generation in Code

Generate the report from the `@AfterAll` hook or a Maven post-test plugin:

```java
package com.company.framework.reports;

import net.masterthought.cucumber.Configuration;
import net.masterthought.cucumber.ReportBuilder;
import net.masterthought.cucumber.Reportable;
import net.masterthought.cucumber.presentation.PresentationMode;
import net.masterthought.cucumber.sorting.SortingMethod;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.io.File;
import java.util.ArrayList;
import java.util.List;

public class CucumberReportGenerator {

    private static final Logger log = LogManager.getLogger(CucumberReportGenerator.class);

    public static void generateReport() {
        File reportOutputDir = new File("target/cucumber-html-report");

        // Collect all JSON files from the reports directory
        List<String> jsonFiles = new ArrayList<>();
        File reportsDir = new File("target/cucumber-reports");
        if (reportsDir.exists()) {
            for (File file : reportsDir.listFiles()) {
                if (file.getName().endsWith(".json")) {
                    jsonFiles.add(file.getAbsolutePath());
                }
            }
        }

        if (jsonFiles.isEmpty()) {
            log.warn("No Cucumber JSON files found. Report not generated.");
            return;
        }

        // Configure the report
        Configuration config = new Configuration(reportOutputDir, "Selenium Framework");

        config.setBuildNumber(
            System.getenv().getOrDefault("BUILD_NUMBER", "local-" + System.currentTimeMillis())
        );
        config.addPresentationModes(PresentationMode.EXPAND_ALL_STEPS);
        config.setSortingMethod(SortingMethod.NATURAL);
        config.addClassifications("Browser",     System.getProperty("browser", "chrome"));
        config.addClassifications("Environment", System.getProperty("env", "dev").toUpperCase());
        config.addClassifications("Platform",    System.getProperty("os.name"));
        config.addClassifications("Branch",
            System.getenv().getOrDefault("GIT_BRANCH", "local"));
        config.addClassifications("Build",
            System.getenv().getOrDefault("BUILD_NUMBER", "manual"));

        // Build the report
        ReportBuilder reportBuilder = new ReportBuilder(jsonFiles, config);
        Reportable result = reportBuilder.generateReports();

        log.info("Cucumber HTML report generated at: {}/cucumber-html-reports/overview-features.html",
            reportOutputDir.getAbsolutePath());
        log.info("Total scenarios: {}, Passed: {}, Failed: {}",
            result.getScenarioCount(),
            result.getPassedScenarios(),
            result.getFailedScenarios()
        );
    }
}
```

**Call from @AfterAll:**

```java
@AfterAll
public static void generateCucumberReport() {
    CucumberReportGenerator.generateReport();
}
```

### Report Pages

The generated report contains these pages:

| Page              | Contents                                                          |
| ----------------- | ----------------------------------------------------------------- |
| Overview Features | Feature-level summary table with pass/fail counts and percentages |
| Overview Tags     | Results grouped by Cucumber tag                                   |
| Overview Steps    | Step-level statistics including duration                          |
| Overview Failures | List of all failed scenarios with error messages                  |
| Feature detail    | Full scenario breakdown for one feature                           |

---

## Multiple JSON File Merging

When running tests in parallel with multiple runners, each runner generates its own JSON file. The `cucumber-reporting` library automatically merges them:

```java
// Collect all JSON files from all parallel runners
List<String> jsonFiles = Arrays.asList(
    "target/cucumber-reports/smoke.json",
    "target/cucumber-reports/regression.json",
    "target/cucumber-reports/sanity.json"
);

// Pass all files to ReportBuilder — it merges them
ReportBuilder reportBuilder = new ReportBuilder(jsonFiles, config);
reportBuilder.generateReports();
```

Or scan the directory dynamically:

```java
List<String> jsonFiles = Files.walk(Paths.get("target/cucumber-reports"))
    .filter(p -> p.toString().endsWith(".json"))
    .map(Path::toString)
    .collect(Collectors.toList());
```

---

## Configuring monochrome Output

When running in CI environments, ANSI color codes in the console output produce garbled characters. The `monochrome` option removes color codes:

```java
@CucumberOptions(
    // ...
    monochrome = true   // removes color codes from console output
)
```

---

## dryRun Mode

`dryRun` validates all step definitions are mapped without executing tests. Use it when adding new feature files to verify no undefined steps exist.

```java
@CucumberOptions(
    features  = "src/test/resources/features",
    glue      = {"stepdefinitions", "hooks"},
    dryRun    = true,    // validates steps, no execution
    plugin    = { "pretty" }
)
public class DryRunValidator extends AbstractTestNGCucumberTests {
}
```

Output on success: all steps show as mapped. Output on failure: Cucumber prints the missing step definition snippet.

---

## Complete Multi-Runner Setup with Reports

A production framework maintains separate runners, each producing its own JSON, with a post-run report generator that merges them all:

### Runner Classes

```java
// SmokeTestRunner.java
@CucumberOptions(
    features = "src/test/resources/features",
    glue     = {"stepdefinitions", "hooks"},
    tags     = "@smoke",
    plugin   = {
        "pretty",
        "json:target/cucumber-reports/smoke.json",
        "junit:target/cucumber-reports/smoke.xml",
        "rerun:target/cucumber-reports/smoke-rerun.txt",
        "io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm",
        "com.aventstack.extentreports.cucumber.adapter.ExtentCucumberAdapter:"
    }
)
public class SmokeTestRunner extends AbstractTestNGCucumberTests { }

// RegressionRunner.java
@CucumberOptions(
    features = "src/test/resources/features",
    glue     = {"stepdefinitions", "hooks"},
    tags     = "@regression and not @wip",
    plugin   = {
        "pretty",
        "json:target/cucumber-reports/regression.json",
        "junit:target/cucumber-reports/regression.xml",
        "rerun:target/cucumber-reports/regression-rerun.txt",
        "io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm",
        "com.aventstack.extentreports.cucumber.adapter.ExtentCucumberAdapter:"
    }
)
public class RegressionRunner extends AbstractTestNGCucumberTests { }
```

### testng.xml

```xml
<suite name="Automation Suite" verbose="1">

    <test name="Smoke Tests">
        <classes>
            <class name="com.company.framework.runners.SmokeTestRunner"/>
        </classes>
    </test>

    <test name="Regression Tests">
        <classes>
            <class name="com.company.framework.runners.RegressionRunner"/>
        </classes>
    </test>

</suite>
```

---

## CI/CD Report Publishing

### GitHub Actions — All Three Reports

```yaml
- name: Run Tests
  run: mvn test -Dheadless=true -Denv=staging

- name: Upload Cucumber JSON
  uses: actions/upload-artifact@v4
  if: always()
  with:
    name: cucumber-json-${{ github.run_number }}
    path: target/cucumber-reports/*.json

- name: Upload Cucumber HTML Report
  uses: actions/upload-artifact@v4
  if: always()
  with:
    name: cucumber-html-report-${{ github.run_number }}
    path: target/cucumber-html-report/

- name: Upload Allure Results
  uses: actions/upload-artifact@v4
  if: always()
  with:
    name: allure-results-${{ github.run_number }}
    path: target/allure-results/

- name: Publish Test Results
  uses: dorny/test-reporter@v1
  if: always()
  with:
    name: Test Results
    path: target/cucumber-reports/*.xml
    reporter: java-junit
```

### Jenkins

```groovy
post {
    always {
        // JUnit XML — shows test counts in build summary
        junit 'target/cucumber-reports/*.xml'

        // Cucumber Reports Plugin
        cucumber([
            fileIncludePattern: '**/*.json',
            jsonReportDirectory: 'target/cucumber-reports',
            reportTitle: 'Cucumber Report'
        ])

        // Allure Report
        allure([results: [[path: 'target/allure-results']]])

        // HTML Publisher for Extent Report
        publishHTML([
            allowMissing: false,
            keepAll: true,
            reportDir: 'target/extent-reports',
            reportFiles: 'SparkReport.html',
            reportName: 'Extent Report'
        ])
    }
}
```

---

## Report Retention Strategy

In CI, retain reports strategically to balance disk usage and audit history:

```yaml
# GitHub Actions — different retention for different report types
- name: Upload Full Report
  uses: actions/upload-artifact@v4
  with:
    name: full-report-${{ github.run_number }}
    path: target/
    retention-days: 7 # keep 7 days for debugging

- name: Upload Allure Results
  uses: actions/upload-artifact@v4
  with:
    name: allure-results-${{ github.run_number }}
    path: target/allure-results/
    retention-days: 30 # keep 30 days for trend tracking
```

---

## Summary

Cucumber's built-in reporters (`pretty`, `html`, `json`, `junit`, `rerun`) cover every standard reporting need with zero additional dependencies. The `json` output is the universal data source for all enhanced reporting tools. The `junit` output integrates with every CI system. The `rerun` plugin enables efficient rerun-of-failures workflows. The `cucumber-reporting` library transforms the JSON output into a feature-rich HTML report with charts, tag analysis, and failure summaries. Combined with Allure and Extent, this stack provides reporting at every level: live console output, machine-readable XML for CI, stakeholder-facing HTML, and trend-tracking dashboards.

---

## Related Topics

- ExtentReports.md — Rich HTML reporting with Extent Spark Reporter
- AllureReports.md — Allure integration and CI dashboard reporting
- ScreenshotsInReports.md — Embedding screenshots in all report types
- IntroBDD.md — Runner class setup and @CucumberOptions
- TagsFiltering.md — Tags used to control which scenarios each runner executes
