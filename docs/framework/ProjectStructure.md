# Project Structure

## Overview

A well-defined project structure is the foundation of a maintainable, scalable Selenium Cucumber framework. It determines how responsibilities are separated, how easily new team members can navigate the codebase, and how confidently changes can be made without breaking unrelated parts of the suite.

This document describes the professional project layout used in a production-grade Java + Maven + Selenium + Cucumber framework with Page Object Model, and explains the purpose and responsibility of every directory and file in that structure.

---

## Maven Standard Directory Layout

A Cucumber Selenium framework follows Maven's standard directory convention. Maven expects source code in `src/main/java`, test code in `src/test/java`, and resource files in the corresponding `resources` directories. Deviating from this convention requires extra Maven configuration and causes confusion for new team members.

```
project-root/
├── pom.xml
├── testng.xml
├── README.md
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/company/framework/
│   │   │       ├── pages/
│   │   │       │   ├── BasePage.java
│   │   │       │   ├── LoginPage.java
│   │   │       │   ├── DashboardPage.java
│   │   │       │   └── components/
│   │   │       │       ├── NavigationBar.java
│   │   │       │       └── DataTable.java
│   │   │       ├── driver/
│   │   │       │   ├── DriverManager.java
│   │   │       │   └── DriverFactory.java
│   │   │       ├── config/
│   │   │       │   └── ConfigReader.java
│   │   │       ├── utils/
│   │   │       │   ├── WaitUtils.java
│   │   │       │   ├── ScreenshotUtils.java
│   │   │       │   ├── JavaScriptUtils.java
│   │   │       │   ├── DateUtils.java
│   │   │       │   └── StringUtils.java
│   │   │       └── models/
│   │   │           ├── User.java
│   │   │           ├── Product.java
│   │   │           └── Order.java
│   │   │
│   │   └── resources/
│   │       └── log4j2.xml
│   │
│   └── test/
│       ├── java/
│       │   └── com/company/framework/
│       │       ├── runners/
│       │       │   ├── SmokeTestRunner.java
│       │       │   ├── RegressionRunner.java
│       │       │   └── SanityRunner.java
│       │       ├── stepdefinitions/
│       │       │   ├── LoginSteps.java
│       │       │   ├── DashboardSteps.java
│       │       │   ├── CartSteps.java
│       │       │   ├── CheckoutSteps.java
│       │       │   └── CommonSteps.java
│       │       ├── hooks/
│       │       │   └── Hooks.java
│       │       ├── context/
│       │       │   └── ScenarioContext.java
│       │       └── paramtypes/
│       │           └── ParameterTypes.java
│       │
│       └── resources/
│           ├── features/
│           │   ├── authentication/
│           │   │   ├── login.feature
│           │   │   └── logout.feature
│           │   ├── products/
│           │   │   └── product-search.feature
│           │   ├── cart/
│           │   │   └── cart-management.feature
│           │   └── checkout/
│           │       └── checkout-flow.feature
│           ├── testdata/
│           │   ├── users.json
│           │   ├── products.json
│           │   └── orders.json
│           └── config/
│               ├── config.properties
│               ├── config-staging.properties
│               └── config-prod.properties
│
├── target/
│   └── cucumber-reports/
│       ├── report.html
│       ├── report.json
│       └── screenshots/
│
└── logs/
    └── test-execution.log
```

---

## Directory Responsibilities

### src/main/java — Production Framework Code

This directory contains all reusable, non-test code. The key principle is that everything here could theoretically be packaged and shared across multiple test projects. It has no dependency on Cucumber, JUnit, or TestNG — it only depends on Selenium.

#### pages/

Contains all Page Object Model classes. Every page of the application has one corresponding class here.

- `BasePage.java` — Abstract parent for all page classes. Contains shared WebDriver utilities, wait methods, and JavaScript helpers.
- Individual page classes — `LoginPage`, `DashboardPage`, `ProductsPage`, etc. Each extends `BasePage`.
- `components/` — Reusable UI components used across multiple pages: `NavigationBar`, `Modal`, `DataTable`, `PaginationBar`.

**Rule:** No Cucumber imports belong in this package. Page classes are pure Selenium + Java.

#### driver/

Contains all WebDriver lifecycle management code.

- `DriverManager.java` — ThreadLocal WebDriver container. Provides `getDriver()` and `setDriver()` methods used across the framework.
- `DriverFactory.java` — Creates browser-specific WebDriver instances based on configuration. Handles Chrome, Firefox, Edge, and headless variants.

**Rule:** Driver creation logic belongs here, not in Hooks. Hooks call DriverFactory; they do not construct drivers.

#### config/

- `ConfigReader.java` — Reads `config.properties` files. Provides a typed interface to configuration values. Handles environment-specific overrides.

#### utils/

Stateless utility classes. Every method should be `public static`. These classes have no instance state.

- `WaitUtils.java` — Custom wait conditions beyond what `ExpectedConditions` provides.
- `ScreenshotUtils.java` — Screenshot capture and file management.
- `JavaScriptUtils.java` — JavaScript execution wrappers.
- `DateUtils.java` — Date formatting and comparison helpers for UI date fields.
- `StringUtils.java` — String normalization, sanitization, and comparison helpers.

#### models/

Plain Java objects representing domain entities used in test data management and page interactions.

- `User.java`, `Product.java`, `Order.java` — POJOs with fields matching the application domain.

---

### src/main/resources — Production Resources

- `log4j2.xml` — Log4j2 logging configuration. Lives in `main/resources` so it applies to both main and test classpath without duplication.

---

### src/test/java — Test Infrastructure Code

This directory contains everything Cucumber-specific: runners, step definitions, hooks, and context classes.

#### runners/

Test runner classes decorated with `@CucumberOptions`. One runner per execution profile.

- `SmokeTestRunner.java` — Runs `@smoke` tagged scenarios.
- `RegressionRunner.java` — Runs `@regression` tagged scenarios.
- `SanityRunner.java` — Runs `@sanity` tagged scenarios.

Each runner specifies its own `tags`, `plugin` list, and output directory.

#### stepdefinitions/

Step definition classes. One class per application module.

- `LoginSteps.java` — Steps for authentication scenarios.
- `DashboardSteps.java` — Steps for dashboard and navigation scenarios.
- `CartSteps.java` — Steps for cart management scenarios.
- `CheckoutSteps.java` — Steps for checkout and payment scenarios.
- `CommonSteps.java` — Reusable steps used across multiple features (navigation, assertions on common elements).

**Rule:** Step definitions delegate to page objects. Selenium calls do not belong in step definitions.

#### hooks/

- `Hooks.java` — `@Before` and `@After` lifecycle methods. Handles driver initialization, screenshot on failure, and driver teardown. Receives `ScenarioContext` via PicoContainer injection.

#### context/

- `ScenarioContext.java` — The World Object. Holds the WebDriver, current page objects, and any captured scenario data that needs to flow between step definition classes.

#### paramtypes/

- `ParameterTypes.java` — Custom `@ParameterType` definitions that convert Gherkin step text into domain objects.

---

### src/test/resources — Test Resources

#### features/

All Gherkin `.feature` files organized into sub-directories by application module. The directory name corresponds to the module tag applied to the features within it.

#### testdata/

JSON, CSV, or Excel files containing test data sets loaded by `TestDataManager`. Organized by entity type.

#### config/

Environment-specific properties files.

- `config.properties` — Default (development) configuration.
- `config-staging.properties` — Staging environment overrides.
- `config-prod.properties` — Production smoke test configuration.

The active environment is selected at runtime via a system property: `-Denv=staging`.

---

## pom.xml Structure

The Maven POM declares dependencies, plugin configuration, and build properties. Key sections:

```xml
<project>
    <groupId>com.company</groupId>
    <artifactId>selenium-framework</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <selenium.version>4.18.1</selenium.version>
        <cucumber.version>7.15.0</cucumber.version>
        <testng.version>7.9.0</testng.version>
        <log4j.version>2.22.1</log4j.version>
        <webdrivermanager.version>5.7.0</webdrivermanager.version>
    </properties>

    <dependencies>
        <!-- Selenium -->
        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <version>${selenium.version}</version>
        </dependency>

        <!-- WebDriver Manager -->
        <dependency>
            <groupId>io.github.bonigarcia</groupId>
            <artifactId>webdrivermanager</artifactId>
            <version>${webdrivermanager.version}</version>
        </dependency>

        <!-- Cucumber -->
        <dependency>
            <groupId>io.cucumber</groupId>
            <artifactId>cucumber-java</artifactId>
            <version>${cucumber.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.cucumber</groupId>
            <artifactId>cucumber-testng</artifactId>
            <version>${cucumber.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.cucumber</groupId>
            <artifactId>cucumber-picocontainer</artifactId>
            <version>${cucumber.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- TestNG -->
        <dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>${testng.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- Logging -->
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>${log4j.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>${log4j.version}</version>
        </dependency>

        <!-- JSON parsing for test data -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.16.1</version>
        </dependency>

        <!-- Apache Commons for utilities -->
        <dependency>
            <groupId>commons-io</groupId>
            <artifactId>commons-io</artifactId>
            <version>2.15.1</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Maven Surefire Plugin — runs TestNG tests -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
                <configuration>
                    <suiteXmlFiles>
                        <suiteXmlFile>testng.xml</suiteXmlFile>
                    </suiteXmlFiles>
                    <systemPropertyVariables>
                        <env>${env}</env>
                        <browser>${browser}</browser>
                    </systemPropertyVariables>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## testng.xml

The TestNG suite file controls which runner classes execute, thread count for parallel runs, and test grouping.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">

<suite name="Selenium Framework Suite" verbose="1" parallel="tests" thread-count="3">

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

## Java Package Naming

All classes use a consistent base package that matches the company and project:

```
com.company.framework.pages
com.company.framework.driver
com.company.framework.config
com.company.framework.utils
com.company.framework.models
```

The test packages mirror the main packages under the same base:

```
com.company.framework.runners
com.company.framework.stepdefinitions
com.company.framework.hooks
com.company.framework.context
```

This consistency means any developer can predict the location of any class from its name alone.

---

## What Belongs Where: Decision Guide

| Code Type             | Location                         | Reasoning                              |
| --------------------- | -------------------------------- | -------------------------------------- |
| Page Object class     | `src/main/java/pages/`           | Reusable production code               |
| WebElement locator    | Inside page class                | Co-located with the page it belongs to |
| WebDriver creation    | `src/main/java/driver/`          | Isolated, testable, reusable           |
| Configuration reading | `src/main/java/config/`          | Used by both main and test code        |
| Utility methods       | `src/main/java/utils/`           | Stateless, reusable utilities          |
| Gherkin feature file  | `src/test/resources/features/`   | Resource file, not compiled            |
| Step definition       | `src/test/java/stepdefinitions/` | Test infrastructure                    |
| Cucumber Hook         | `src/test/java/hooks/`           | Test lifecycle, not reusable           |
| Test runner           | `src/test/java/runners/`         | Test execution entry point             |
| Test data file        | `src/test/resources/testdata/`   | Test-only resource                     |
| Config properties     | `src/test/resources/config/`     | Test-only configuration                |
| Log configuration     | `src/main/resources/`            | Applies to all classpath users         |

---

## Naming Conventions

Consistent naming makes navigation fast and prevents ambiguity.

| Element                | Convention                    | Example                            |
| ---------------------- | ----------------------------- | ---------------------------------- |
| Page class             | PascalCase + "Page"           | `LoginPage`, `CheckoutPage`        |
| Component class        | PascalCase + component type   | `NavigationBar`, `DataTable`       |
| Step definition class  | PascalCase + "Steps"          | `LoginSteps`, `CartSteps`          |
| Test runner class      | PascalCase + "Runner"         | `SmokeTestRunner`                  |
| Feature file           | lowercase-hyphenated          | `user-login.feature`               |
| Step definition method | camelCase, describes the step | `theUserLogsInWith()`              |
| Locator field          | camelCase, describes element  | `loginButton`, `usernameField`     |
| Config property key    | dot.separated.lowercase       | `app.base.url`, `browser.type`     |
| Test data file         | lowercase-hyphenated          | `users.json`, `test-products.json` |

---

## Growing the Framework

As the application under test grows, extend the structure rather than restructuring it:

- Add new page classes to `pages/` as new application screens are built.
- Add new step definition classes to `stepdefinitions/` as new feature files are written.
- Add new feature sub-directories to `features/` as new modules are added.
- Add new runner classes to `runners/` if a new execution profile is needed.
- Add new utility classes to `utils/` for any shared non-Selenium logic.

The structure is designed to absorb growth without requiring reorganization.

---

## Summary

A clean, consistent project structure is what separates a proof-of-concept automation script from a professional framework. By placing production code in `src/main`, test infrastructure in `src/test`, and resources in their respective `resources` directories — and by following strict naming and responsibility conventions — the framework remains navigable, maintainable, and scalable as the team and the test suite grow.

---

## Related Topics

- DriverManager.md — ThreadLocal WebDriver management
- ConfigProperties.md — Reading environment-specific configuration
- TestDataManagement.md — Loading and managing test data files
- Utilities.md — Shared utility class implementations
- Log4j.md — Logging configuration and usage
