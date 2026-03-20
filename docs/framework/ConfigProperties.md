# Config and Properties

## Overview

Configuration management is the practice of externalizing all environment-dependent, deployment-specific, and tunable values out of the Java source code and into properties files. Values like base URLs, browser type, timeout durations, credentials, and API endpoints change between environments and teams. Hard-coding them in test classes or page objects means editing source files every time an environment changes.

A professional Selenium framework uses a `ConfigReader` utility to load `.properties` files at startup, expose typed getters for every value, and support environment-specific overrides through a naming convention and a runtime system property.

---

## Properties File Format

Java `.properties` files use simple `key=value` syntax. Lines beginning with `#` are comments.

```properties
# Browser configuration
browser=chrome
headless=false

# Application URLs
app.base.url=https://dev.example.com
app.login.url=https://dev.example.com/login
app.api.url=https://api.dev.example.com

# Timeouts (in seconds)
timeout.implicit=0
timeout.explicit=10
timeout.pageload=30
timeout.script=30

# Test credentials
test.admin.username=admin@example.com
test.admin.password=AdminPass123
test.buyer.username=buyer@example.com
test.buyer.password=BuyerPass123

# Reporting
report.title=Selenium Test Report
report.name=Automation Suite
screenshots.on.failure=true
screenshots.dir=target/screenshots

# Selenium Grid
grid.enabled=false
grid.url=http://localhost:4444
```

---

## Environment-Specific Properties Files

Each environment has its own properties file. The active environment is selected at runtime via the `-Denv` system property.

```
src/test/resources/config/
  config.properties           ← default (development)
  config-staging.properties   ← staging environment
  config-uat.properties       ← UAT environment
  config-prod.properties      ← production smoke tests
```

`config.properties` contains all keys with their development defaults. Environment-specific files only need to override the keys that differ.

**config-staging.properties:**

```properties
app.base.url=https://staging.example.com
app.login.url=https://staging.example.com/login
app.api.url=https://api.staging.example.com
test.admin.password=StagingAdmin456
```

**config-prod.properties:**

```properties
app.base.url=https://www.example.com
app.login.url=https://www.example.com/login
headless=true
timeout.explicit=15
timeout.pageload=60
```

---

## ConfigReader Implementation

`ConfigReader` is a static utility class that loads the appropriate properties file at startup and provides typed access to every value. It uses a two-tier lookup: system properties take precedence over file properties, so any value can be overridden from the command line without changing the file.

```java
package com.company.framework.config;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.util.Properties;

public class ConfigReader {

    private static final Logger log = LogManager.getLogger(ConfigReader.class);
    private static final Properties properties = new Properties();
    private static boolean loaded = false;

    static {
        load();
    }

    private ConfigReader() {
        throw new IllegalStateException("ConfigReader is a static utility class");
    }

    // -----------------------------------------------------------------------
    // Initialization
    // -----------------------------------------------------------------------

    /**
     * Loads the appropriate config file based on the -Denv system property.
     * Falls back to config.properties if no environment is specified.
     */
    private static void load() {
        String env = System.getProperty("env", "");
        String fileName = env.isEmpty()
            ? "config/config.properties"
            : "config/config-" + env + ".properties";

        // First load the default config
        loadFile("config/config.properties");

        // Then overlay environment-specific values if an env was specified
        if (!env.isEmpty()) {
            loadFile(fileName);
        }

        loaded = true;
        log.info("Configuration loaded. Environment: [{}]", env.isEmpty() ? "default" : env);
    }

    private static void loadFile(String filePath) {
        try (InputStream input = ConfigReader.class
                .getClassLoader()
                .getResourceAsStream(filePath)) {

            if (input == null) {
                log.warn("Config file not found on classpath: {}", filePath);
                return;
            }

            properties.load(input);
            log.debug("Loaded properties from: {}", filePath);

        } catch (IOException e) {
            log.error("Failed to load config file: {} — {}", filePath, e.getMessage());
            throw new RuntimeException("Cannot load configuration file: " + filePath, e);
        }
    }

    // -----------------------------------------------------------------------
    // Value accessors — two-tier: system property overrides file property
    // -----------------------------------------------------------------------

    /**
     * Returns the value for the key. Throws if not found.
     */
    public static String get(String key) {
        // System property (-Dkey=value) always takes precedence
        String systemValue = System.getProperty(key);
        if (systemValue != null && !systemValue.isEmpty()) {
            return systemValue;
        }

        String value = properties.getProperty(key);
        if (value == null) {
            throw new RuntimeException("Configuration key not found: " + key
                + ". Check config.properties or -D" + key + " system property.");
        }

        return value.trim();
    }

    /**
     * Returns the value for the key, or defaultValue if not found.
     */
    public static String get(String key, String defaultValue) {
        try {
            return get(key);
        } catch (RuntimeException e) {
            return defaultValue;
        }
    }

    /**
     * Returns the value as an int. Throws if not found or not parseable.
     */
    public static int getInt(String key) {
        String value = get(key);
        try {
            return Integer.parseInt(value);
        } catch (NumberFormatException e) {
            throw new RuntimeException("Config key '" + key + "' is not a valid integer: " + value);
        }
    }

    public static int getInt(String key, int defaultValue) {
        try {
            return getInt(key);
        } catch (RuntimeException e) {
            return defaultValue;
        }
    }

    /**
     * Returns the value as a boolean.
     * Accepts: true/false, yes/no, 1/0 (case-insensitive).
     */
    public static boolean getBoolean(String key) {
        String value = get(key).toLowerCase().trim();
        return value.equals("true") || value.equals("yes") || value.equals("1");
    }

    public static boolean getBoolean(String key, boolean defaultValue) {
        try {
            return getBoolean(key);
        } catch (RuntimeException e) {
            return defaultValue;
        }
    }

    /**
     * Returns the value as a long.
     */
    public static long getLong(String key) {
        String value = get(key);
        try {
            return Long.parseLong(value);
        } catch (NumberFormatException e) {
            throw new RuntimeException("Config key '" + key + "' is not a valid long: " + value);
        }
    }

    public static long getLong(String key, long defaultValue) {
        try {
            return getLong(key);
        } catch (RuntimeException e) {
            return defaultValue;
        }
    }

    /**
     * Returns true if the key exists in the config (file or system property).
     */
    public static boolean hasKey(String key) {
        return System.getProperty(key) != null
            || properties.getProperty(key) != null;
    }

    /**
     * Reloads all properties. Useful for testing or dynamic environment switches.
     */
    public static void reload() {
        properties.clear();
        loaded = false;
        load();
    }
}
```

---

## Using ConfigReader in the Framework

### In DriverFactory

```java
public static WebDriver createDriver() {
    String browser = System.getProperty("browser",
        ConfigReader.get("browser", "chrome")).toLowerCase();

    boolean headless = ConfigReader.getBoolean("headless", false);
    // ...
}
```

### In BasePage

```java
public abstract class BasePage {

    protected WebDriver driver;
    protected WebDriverWait wait;

    public BasePage(WebDriver driver) {
        this.driver = driver;
        int timeout = ConfigReader.getInt("timeout.explicit", 10);
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(timeout));
        PageFactory.initElements(driver, this);
    }
}
```

### In Step Definitions

```java
@Given("the user is on the login page")
public void theUserIsOnLoginPage() {
    String loginUrl = ConfigReader.get("app.login.url");
    context.getDriver().get(loginUrl);
    context.setLoginPage(new LoginPage(context.getDriver()));
}
```

### In Hooks

```java
@Before
public void setUp(Scenario scenario) {
    log.info("Base URL: {}", ConfigReader.get("app.base.url"));
    WebDriver driver = DriverFactory.createDriver();
    DriverManager.setDriver(driver);
    driver.get(ConfigReader.get("app.base.url"));
}
```

---

## Typed Configuration Constants

For large frameworks, define a `ConfigKeys` constants class to avoid string literals scattered throughout the code.

```java
package com.company.framework.config;

public final class ConfigKeys {

    private ConfigKeys() {}

    // Browser
    public static final String BROWSER          = "browser";
    public static final String HEADLESS         = "headless";

    // URLs
    public static final String BASE_URL         = "app.base.url";
    public static final String LOGIN_URL        = "app.login.url";
    public static final String API_URL          = "app.api.url";

    // Timeouts
    public static final String TIMEOUT_IMPLICIT = "timeout.implicit";
    public static final String TIMEOUT_EXPLICIT = "timeout.explicit";
    public static final String TIMEOUT_PAGELOAD = "timeout.pageload";

    // Credentials
    public static final String ADMIN_USERNAME   = "test.admin.username";
    public static final String ADMIN_PASSWORD   = "test.admin.password";
    public static final String BUYER_USERNAME   = "test.buyer.username";
    public static final String BUYER_PASSWORD   = "test.buyer.password";

    // Grid
    public static final String GRID_ENABLED     = "grid.enabled";
    public static final String GRID_URL         = "grid.url";

    // Reports
    public static final String SCREENSHOTS_DIR  = "screenshots.dir";
    public static final String REPORT_TITLE     = "report.title";
}
```

Usage:

```java
String url = ConfigReader.get(ConfigKeys.BASE_URL);
boolean headless = ConfigReader.getBoolean(ConfigKeys.HEADLESS, false);
int timeout = ConfigReader.getInt(ConfigKeys.TIMEOUT_EXPLICIT, 10);
```

---

## Environment Selection at Runtime

### Command Line (Maven)

```bash
# Run against staging
mvn test -Denv=staging

# Run against staging with Firefox
mvn test -Denv=staging -Dbrowser=firefox

# Run against staging in headless mode
mvn test -Denv=staging -Dheadless=true

# Run smoke tests against production
mvn test -Denv=prod -Dcucumber.filter.tags="@smoke"

# Override a single value without changing files
mvn test -Dapp.base.url=https://preview.example.com
```

### In CI/CD Pipelines (GitHub Actions example)

```yaml
- name: Run Regression Tests
  run: mvn test -Denv=staging -Dbrowser=chrome-headless -Dcucumber.filter.tags="@regression"
  env:
    SELENIUM_GRID_URL: ${{ secrets.GRID_URL }}
```

System properties passed via environment variables or pipeline parameters let you run the same framework against any environment without modifying any file.

---

## Securing Sensitive Values

Never commit passwords, API keys, or tokens to properties files in source control. Use one of these approaches:

### Environment Variables

```java
public static String getSecure(String key) {
    // First try environment variable (CI/CD injects these)
    String envValue = System.getenv(key.toUpperCase().replace(".", "_"));
    if (envValue != null && !envValue.isEmpty()) {
        return envValue;
    }
    // Fall back to properties file for local development
    return get(key);
}
```

Usage in `config.properties`:

```properties
# Passwords are loaded from environment variables in CI
# Locally, developers set these in their shell or .env file
test.admin.password=${ADMIN_PASSWORD}
```

### Jenkins / GitHub Actions Secrets

Store passwords as encrypted secrets in the CI system and inject them as environment variables:

```yaml
env:
  TEST_ADMIN_PASSWORD: ${{ secrets.TEST_ADMIN_PASSWORD }}
```

```bash
mvn test -Dtest.admin.password=$TEST_ADMIN_PASSWORD
```

---

## Complete config.properties Reference

```properties
# ============================================================
# Browser Configuration
# ============================================================
browser=chrome
# Options: chrome | firefox | edge | chrome-headless | firefox-headless | remote
headless=false

# ============================================================
# Application URLs
# ============================================================
app.base.url=https://dev.example.com
app.login.url=https://dev.example.com/login
app.api.url=https://api.dev.example.com
app.admin.url=https://dev.example.com/admin

# ============================================================
# Timeout Configuration (seconds)
# ============================================================
timeout.implicit=0
timeout.explicit=10
timeout.pageload=30
timeout.script=30
timeout.ajax=15

# ============================================================
# Test User Credentials
# ============================================================
test.admin.username=admin@example.com
test.admin.password=AdminDev123
test.buyer.username=buyer@example.com
test.buyer.password=BuyerDev123
test.viewer.username=viewer@example.com
test.viewer.password=ViewerDev123

# ============================================================
# Selenium Grid
# ============================================================
grid.enabled=false
grid.url=http://localhost:4444
remote.browser=chrome
platform=LINUX

# ============================================================
# Reporting
# ============================================================
report.title=Selenium Automation Report
report.name=Test Execution Results
screenshots.on.failure=true
screenshots.dir=target/screenshots
extent.report.dir=target/extent-reports
allure.results.dir=target/allure-results

# ============================================================
# Test Data
# ============================================================
testdata.dir=src/test/resources/testdata
testdata.users.file=users.json
testdata.products.file=products.json
```

---

## Summary

Effective configuration management externalizes all environment-dependent values, supports multi-environment test runs through a simple `-Denv` system property, allows any individual value to be overridden from the command line without code changes, and keeps sensitive credentials out of source control. The `ConfigReader` class is the single access point for all configuration values across the entire framework, with typed getters that prevent conversion errors and clear error messages when keys are missing.

---

## Related Topics

- ProjectStructure.md — Location of config files in the project layout
- DriverManager.md — Browser and timeout values consumed by DriverFactory
- TestDataManagement.md — Test data file paths loaded from configuration
- Log4j.md — Logging configuration file location
- Jenkins.md — Passing environment parameters in CI/CD pipelines
