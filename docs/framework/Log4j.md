# Logging with Log4j2

## Overview

Log4j2 is the logging framework used in a professional Selenium Cucumber automation suite. It provides structured, configurable, and performant logging that replaces raw `System.out.println` calls. Log4j2 supports multiple output destinations (console, files, rolling files), multiple severity levels, pattern-based formatting, and asynchronous logging — all configurable through an XML file without changing any Java code.

Effective logging is one of the most important debugging tools in a test framework. When a scenario fails in a CI pipeline, the execution log is often the first — and sometimes the only — artifact available to diagnose the failure.

---

## Maven Dependencies

Log4j2 requires two artifacts: the API (which your code imports) and the Core (the implementation).

```xml
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-api</artifactId>
    <version>2.22.1</version>
</dependency>

<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.22.1</version>
</dependency>
```

If the project uses SLF4J (which some Selenium or Cucumber dependencies use internally), add the SLF4J-to-Log4j2 bridge to route all logging through Log4j2:

```xml
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-slf4j2-impl</artifactId>
    <version>2.22.1</version>
</dependency>
```

---

## Log4j2 Configuration File

The configuration file is `log4j2.xml`. It lives in `src/main/resources/` so it is on the classpath for both `main` and `test` code without duplication.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN" monitorInterval="30">

    <Properties>
        <Property name="LOG_DIR">logs</Property>
        <Property name="LOG_FILE">${LOG_DIR}/test-execution.log</Property>
        <Property name="ARCHIVE_DIR">${LOG_DIR}/archive</Property>

        <!-- Console pattern: [LEVEL] HH:mm:ss [ClassName] message -->
        <Property name="CONSOLE_PATTERN">
            [%-5level] %d{HH:mm:ss.SSS} [%t] %logger{36} - %msg%n
        </Property>

        <!-- File pattern: includes date and full thread info -->
        <Property name="FILE_PATTERN">
            %d{yyyy-MM-dd HH:mm:ss.SSS} [%-5level] [%t] %logger{50} - %msg%n
        </Property>
    </Properties>

    <Appenders>

        <!-- Console Appender — outputs to stdout -->
        <Console name="ConsoleAppender" target="SYSTEM_OUT">
            <PatternLayout pattern="${CONSOLE_PATTERN}"/>
            <!-- Show INFO and above on console; suppress DEBUG noise -->
            <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
        </Console>

        <!-- Rolling File Appender — creates a new file each day, archives old ones -->
        <RollingFile name="FileAppender"
                     fileName="${LOG_FILE}"
                     filePattern="${ARCHIVE_DIR}/test-execution-%d{yyyy-MM-dd}-%i.log.gz">

            <PatternLayout pattern="${FILE_PATTERN}"/>

            <Policies>
                <!-- Roll over at midnight -->
                <TimeBasedTriggeringPolicy interval="1" modulate="true"/>
                <!-- Also roll over when file exceeds 10 MB -->
                <SizeBasedTriggeringPolicy size="10MB"/>
            </Policies>

            <!-- Keep the last 30 archived log files -->
            <DefaultRolloverStrategy max="30"/>
        </RollingFile>

        <!-- Separate appender for ERROR and FATAL — makes critical failures easy to find -->
        <RollingFile name="ErrorFileAppender"
                     fileName="${LOG_DIR}/errors.log"
                     filePattern="${ARCHIVE_DIR}/errors-%d{yyyy-MM-dd}-%i.log.gz">

            <PatternLayout pattern="${FILE_PATTERN}"/>
            <ThresholdFilter level="ERROR" onMatch="ACCEPT" onMismatch="DENY"/>

            <Policies>
                <TimeBasedTriggeringPolicy interval="1" modulate="true"/>
            </Policies>
        </RollingFile>

    </Appenders>

    <Loggers>

        <!-- Framework logger — DEBUG level for detailed internal tracing -->
        <Logger name="com.company.framework" level="DEBUG" additivity="false">
            <AppenderRef ref="ConsoleAppender"/>
            <AppenderRef ref="FileAppender"/>
            <AppenderRef ref="ErrorFileAppender"/>
        </Logger>

        <!-- Selenium WebDriver logger — suppress verbose driver noise -->
        <Logger name="org.openqa.selenium" level="WARN" additivity="false">
            <AppenderRef ref="FileAppender"/>
        </Logger>

        <!-- Cucumber logger -->
        <Logger name="io.cucumber" level="INFO" additivity="false">
            <AppenderRef ref="ConsoleAppender"/>
            <AppenderRef ref="FileAppender"/>
        </Logger>

        <!-- WebDriverManager — suppress download messages in normal runs -->
        <Logger name="io.github.bonigarcia" level="WARN" additivity="false">
            <AppenderRef ref="FileAppender"/>
        </Logger>

        <!-- Root logger — catches everything not matched by a named logger -->
        <Root level="INFO">
            <AppenderRef ref="ConsoleAppender"/>
            <AppenderRef ref="FileAppender"/>
        </Root>

    </Loggers>

</Configuration>
```

---

## Log Levels

Log4j2 defines six levels in ascending severity order:

| Level   | Numeric Value | Purpose                                                                  |
| ------- | ------------- | ------------------------------------------------------------------------ |
| `TRACE` | 600           | Extremely granular — method entry/exit, loop iterations                  |
| `DEBUG` | 500           | Diagnostic — element locations, wait details, values read                |
| `INFO`  | 400           | Normal operational messages — scenario start, page loaded, action taken  |
| `WARN`  | 300           | Unexpected but recoverable — retry attempt, element not found then found |
| `ERROR` | 200           | Operation failed — screenshot not saved, config key missing              |
| `FATAL` | 100           | Framework cannot continue — driver failed to create                      |

**Guideline for which level to use:**

- Use `INFO` for messages that describe what the test is doing: scenario starting, navigating to a page, clicking a button, asserting a value.
- Use `DEBUG` for messages that show how it is doing it: locator used, element found, value read from config.
- Use `WARN` for non-fatal problems: a retry was needed, a default value was used because a config key was missing.
- Use `ERROR` for failures that should be investigated: a screenshot could not be saved, a file could not be read.

---

## Using Log4j2 in Framework Classes

Every class that logs declares a static `Logger` field. The logger is named after the class using `LogManager.getLogger(ClassName.class)`. This ensures log output shows the exact class source.

### Declaration Pattern

```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class LoginPage extends BasePage {

    private static final Logger log = LogManager.getLogger(LoginPage.class);

    // rest of class
}
```

### Logging in Page Classes

```java
public class LoginPage extends BasePage {

    private static final Logger log = LogManager.getLogger(LoginPage.class);

    @FindBy(id = "username")
    private WebElement usernameField;

    @FindBy(id = "password")
    private WebElement passwordField;

    @FindBy(id = "loginBtn")
    private WebElement loginButton;

    public LoginPage(WebDriver driver) {
        super(driver);
        log.debug("LoginPage initialized");
    }

    public DashboardPage login(String username, String password) {
        log.info("Logging in as user: {}", username);
        type(usernameField, username);
        type(passwordField, password);
        click(loginButton);
        log.info("Login submitted for: {}", username);
        return new DashboardPage(driver);
    }

    public String getErrorMessage() {
        String message = getText(errorMessage);
        log.debug("Login error message: {}", message);
        return message;
    }
}
```

### Logging in Hooks

```java
public class Hooks {

    private static final Logger log = LogManager.getLogger(Hooks.class);

    @Before(order = 1)
    public void initializeDriver(Scenario scenario) {
        log.info("=== Starting scenario: [{}] ===", scenario.getName());
        log.debug("Scenario tags: {}", scenario.getSourceTagNames());

        WebDriver driver = DriverFactory.createDriver();
        DriverManager.setDriver(driver);

        log.info("Browser started. URL: {}", ConfigReader.get("app.base.url"));
    }

    @After(order = 2)
    public void captureFailure(Scenario scenario) {
        if (scenario.isFailed()) {
            log.error("SCENARIO FAILED: [{}]", scenario.getName());
            log.error("Current URL at failure: {}", DriverManager.getDriver().getCurrentUrl());
            // capture screenshot
        }
    }

    @After(order = 1)
    public void tearDown(Scenario scenario) {
        log.info("=== Scenario finished: [{}] — Status: {} ===",
            scenario.getName(), scenario.getStatus());
        DriverManager.quitDriver();
    }
}
```

### Logging in DriverFactory

```java
public class DriverFactory {

    private static final Logger log = LogManager.getLogger(DriverFactory.class);

    public static WebDriver createDriver() {
        String browser = ConfigReader.get("browser", "chrome");
        log.info("Creating {} WebDriver", browser);

        WebDriver driver;
        switch (browser) {
            case "firefox":
                log.debug("Configuring Firefox options");
                driver = createFirefoxDriver();
                break;
            default:
                log.debug("Configuring Chrome options. Headless: {}",
                    ConfigReader.getBoolean("headless", false));
                driver = createChromeDriver();
        }

        log.info("WebDriver created successfully for thread: {}",
            Thread.currentThread().getName());
        return driver;
    }
}
```

### Logging in Step Definitions

Step definitions should log at INFO for high-level actions and DEBUG for parameter values:

```java
public class LoginSteps {

    private static final Logger log = LogManager.getLogger(LoginSteps.class);

    @When("the user logs in with username {string} and password {string}")
    public void theUserLogsIn(String username, String password) {
        log.info("Step: User logs in as '{}'", username);
        log.debug("Password length: {} characters", password.length());
        // Never log the actual password value
        context.setDashboardPage(
            context.getLoginPage().login(username, password)
        );
    }
}
```

---

## Parameterized Log Messages

Always use the parameterized form `log.info("Message {}", value)` instead of string concatenation `log.info("Message " + value)`. With parameterized messages, if the log level is below the configured threshold, the string is never constructed — improving performance.

```java
// Wrong — string concatenation always runs even if DEBUG is disabled
log.debug("Element found: " + element.toString() + " with text: " + element.getText());

// Correct — arguments are only evaluated if DEBUG is enabled
log.debug("Element found: {} with text: {}", element, element.getText());
```

---

## Masking Sensitive Data in Logs

Never log passwords, tokens, or credit card numbers. Use the `StringUtils.maskSensitive()` method or the `{}` placeholder pattern with a masked representation.

```java
// Wrong
log.info("Logging in with password: {}", password);

// Correct
log.info("Logging in as: {} (password provided, {} characters)", username, password.length());

// Or use the masking utility
log.debug("Credentials: {} / {}", username, StringUtils.maskSensitive(password));
// Output: Credentials: admin@example.com / Ad*********23
```

---

## Changing Log Level at Runtime

Log level can be overridden at runtime using a system property, without changing the XML file. Add this to the configuration:

```xml
<Configuration status="WARN">
    <Properties>
        <Property name="LOG_LEVEL">${sys:log.level:-INFO}</Property>
    </Properties>

    <Loggers>
        <Logger name="com.company.framework" level="${LOG_LEVEL}" additivity="false">
            <AppenderRef ref="ConsoleAppender"/>
            <AppenderRef ref="FileAppender"/>
        </Logger>
    </Loggers>
</Configuration>
```

Runtime override:

```bash
# Run with DEBUG logging
mvn test -Dlog.level=DEBUG

# Run with WARN only (minimal output)
mvn test -Dlog.level=WARN
```

---

## Log Output Examples

**Console output during a passing scenario:**

```
[INFO ] 14:22:03.441 [main] com.company.Hooks - === Starting scenario: [Successful login] ===
[INFO ] 14:22:03.892 [main] com.company.DriverFactory - Creating chrome WebDriver
[INFO ] 14:22:05.117 [main] com.company.DriverFactory - WebDriver created for thread: main
[INFO ] 14:22:05.832 [main] com.company.stepdefs.LoginSteps - Step: User is on the login page
[INFO ] 14:22:06.201 [main] com.company.pages.LoginPage - Logging in as user: admin@example.com
[INFO ] 14:22:06.890 [main] com.company.pages.LoginPage - Login submitted for: admin@example.com
[INFO ] 14:22:07.341 [main] com.company.Hooks - === Scenario finished: [Successful login] — Status: PASSED ===
```

**Console output on failure:**

```
[INFO ] 14:23:11.002 [main] com.company.Hooks - === Starting scenario: [Login with wrong password] ===
[INFO ] 14:23:13.441 [main] com.company.pages.LoginPage - Logging in as user: admin@example.com
[WARN ] 14:23:14.002 [main] com.company.pages.BasePage - Element not immediately visible, retrying: #errorMessage
[ERROR] 14:23:16.331 [main] com.company.Hooks - SCENARIO FAILED: [Login with wrong password]
[ERROR] 14:23:16.332 [main] com.company.Hooks - Current URL at failure: https://dev.example.com/login
[INFO ] 14:23:16.405 [main] com.company.Hooks - Screenshot captured for failure report
[INFO ] 14:23:16.901 [main] com.company.Hooks - === Scenario finished: [Login with wrong password] — Status: FAILED ===
```

---

## Log File Management

Logs accumulate quickly in a CI environment. The rolling file configuration handles this automatically:

- Daily rollover creates `test-execution-2025-03-15-1.log.gz` each midnight
- Size rollover ensures no single file exceeds 10 MB
- `DefaultRolloverStrategy max="30"` deletes archives older than 30 days

For CI pipelines, publish the `logs/` directory as a build artifact so logs from any run are accessible for post-failure debugging.

---

## Summary

Log4j2 provides the observability layer that makes a Selenium Cucumber framework debuggable in production. The `log4j2.xml` configuration controls log levels, output destinations, and rolling file management without touching Java code. Every class declares a static `Logger` named after itself, uses parameterized message format for performance, uses appropriate severity levels, and never logs sensitive values. The result is structured, searchable log output that makes diagnosing any failure — locally or in CI — efficient and reliable.

---

## Related Topics

- ProjectStructure.md — log4j2.xml file location in the project
- DriverManager.md — Logger usage in driver lifecycle events
- Hooks.md — Scenario start, failure, and finish log points
- Utilities.md — Logger declarations in all utility classes
- ConfigProperties.md — Log level controlled via -Dlog.level property
