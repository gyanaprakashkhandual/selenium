# Browser Drivers

## Overview

A browser driver is the critical bridge between Selenium WebDriver and the actual browser. Without a compatible browser driver, Selenium cannot communicate with the browser. This document explains what browser drivers are, how they work, and how to manage them — both manually and automatically using WebDriverManager.

---

## What is a Browser Driver?

A browser driver is a standalone executable that implements the **W3C WebDriver protocol**. It acts as an HTTP server that listens for WebDriver commands from your Selenium code and translates them into native browser instructions.

```
Selenium Test Code
       │
       │  HTTP + JSON (W3C WebDriver Protocol)
       ▼
Browser Driver (e.g., chromedriver.exe)
       │
       │  Internal browser automation API
       ▼
Browser (Chrome / Firefox / Edge / Safari)
```

Each browser has its own driver, maintained either by the browser vendor or the open-source community:

| Browser           | Driver             | Maintained By                 |
| ----------------- | ------------------ | ----------------------------- |
| Google Chrome     | **ChromeDriver**   | Google                        |
| Mozilla Firefox   | **GeckoDriver**    | Mozilla                       |
| Microsoft Edge    | **EdgeDriver**     | Microsoft                     |
| Apple Safari      | **SafariDriver**   | Apple (built into macOS)      |
| Internet Explorer | **IEDriverServer** | Selenium Project (deprecated) |

---

## ChromeDriver — The Most Used Driver

ChromeDriver is the most widely used driver in automation frameworks. It must match your installed Chrome version exactly.

### Chrome Version Compatibility

| Chrome Version       | ChromeDriver Version                           |
| -------------------- | ---------------------------------------------- |
| Chrome 115+          | Managed via Chrome for Testing (CfT)           |
| Chrome 114 and below | Matched manually via chromedriver.chromium.org |

> **Important:** Since Chrome 115, Google moved ChromeDriver distribution to the **Chrome for Testing** infrastructure. WebDriverManager handles this automatically.

### Manual ChromeDriver Setup (Legacy — Not Recommended)

```bash
# 1. Check your Chrome version
# Chrome menu → Help → About Google Chrome
# e.g., Version 121.0.6167.140

# 2. Download matching ChromeDriver from:
# https://chromedriver.chromium.org/downloads

# 3. Place it in a known location
# Windows: C:\drivers\chromedriver.exe
# macOS/Linux: /usr/local/bin/chromedriver

# 4. Set in System PATH or specify in code:
System.setProperty("webdriver.chrome.driver", "/usr/local/bin/chromedriver");
WebDriver driver = new ChromeDriver();
```

This manual approach is fragile — every Chrome auto-update can break your tests if the driver version falls out of sync. This is why **WebDriverManager** was created.

---

## WebDriverManager — The Automated Solution

[WebDriverManager](https://github.com/bonigarcia/webdrivermanager) by Boni García is the industry-standard solution for automatic driver management. It detects the browser version installed on the system, downloads the appropriate driver binary, caches it locally, and configures the system property — all in one line of code.

### Maven Dependency

```xml
<dependency>
    <groupId>io.github.bonigarcia</groupId>
    <artifactId>webdrivermanager</artifactId>
    <version>5.7.0</version>
</dependency>
```

### Basic Usage

```java
import io.github.bonigarcia.wdm.WebDriverManager;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.edge.EdgeDriver;

public class DriverSetupExample {

    public static void main(String[] args) {

        // Chrome
        WebDriverManager.chromedriver().setup();
        WebDriver chromeDriver = new ChromeDriver();

        // Firefox
        WebDriverManager.firefoxdriver().setup();
        WebDriver firefoxDriver = new FirefoxDriver();

        // Edge
        WebDriverManager.edgedriver().setup();
        WebDriver edgeDriver = new EdgeDriver();
    }
}
```

### How WebDriverManager Works Internally

1. **Detects** the installed browser version (reads from registry on Windows, binary on macOS/Linux).
2. **Looks up** the correct driver version for that browser version.
3. **Checks** the local cache (`~/.cache/selenium` by default).
4. **Downloads** the driver if not cached.
5. **Sets** the system property automatically (e.g., `webdriver.chrome.driver`).

### WebDriverManager Cache Location

```
# Default cache directory
Windows:  C:\Users\<username>\.cache\selenium\
macOS:    /Users/<username>/.cache/selenium/
Linux:    /home/<username>/.cache/selenium/

# Cache structure example
.cache/selenium/
├── chromedriver/
│   └── win64/
│       └── 121.0.6167.140/
│           └── chromedriver.exe
├── geckodriver/
│   └── linux64/
│       └── 0.34.0/
│           └── geckodriver
```

### Advanced WebDriverManager Configuration

```java
// Force a specific driver version
WebDriverManager.chromedriver().driverVersion("121.0.6167.140").setup();

// Use a proxy (corporate environments)
WebDriverManager.chromedriver()
    .proxy("proxy.company.com:8080")
    .proxyUser("username")
    .proxyPass("password")
    .setup();

// Use a mirror (for CI environments without internet)
WebDriverManager.chromedriver()
    .mirrorUrl("http://internal-nexus/drivers/")
    .setup();

// Set a custom cache path
WebDriverManager.chromedriver()
    .cachePath("/opt/drivers")
    .setup();

// Force download even if cached
WebDriverManager.chromedriver().forceDownload().setup();
```

---

## Setting Up Each Browser Driver

### Chrome with Options

```java
import io.github.bonigarcia.wdm.WebDriverManager;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;

public class ChromeDriverSetup {

    public static WebDriver createChromeDriver(boolean headless) {
        WebDriverManager.chromedriver().setup();

        ChromeOptions options = new ChromeOptions();

        // Common options for stable test execution
        options.addArguments("--start-maximized");
        options.addArguments("--disable-notifications");
        options.addArguments("--disable-popup-blocking");
        options.addArguments("--disable-infobars");
        options.addArguments("--no-sandbox");           // Required in Docker/CI
        options.addArguments("--disable-dev-shm-usage"); // Required in Docker
        options.addArguments("--remote-allow-origins=*"); // Required in Selenium 4

        // Headless mode (no browser UI — faster, for CI pipelines)
        if (headless) {
            options.addArguments("--headless=new"); // Use new headless mode (Chrome 112+)
            options.addArguments("--window-size=1920,1080");
        }

        // Disable password save dialog
        options.setExperimentalOption("prefs", Map.of(
            "credentials_enable_service", false,
            "profile.password_manager_enabled", false
        ));

        return new ChromeDriver(options);
    }
}
```

### Firefox with Options

```java
import io.github.bonigarcia.wdm.WebDriverManager;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.firefox.FirefoxOptions;
import org.openqa.selenium.firefox.FirefoxProfile;

public class FirefoxDriverSetup {

    public static WebDriver createFirefoxDriver(boolean headless) {
        WebDriverManager.firefoxdriver().setup();

        FirefoxOptions options = new FirefoxOptions();

        // Custom Firefox profile settings
        FirefoxProfile profile = new FirefoxProfile();
        profile.setPreference("browser.download.folderList", 2);
        profile.setPreference("browser.helperApps.neverAsk.saveToDisk",
            "application/pdf,application/zip");

        options.setProfile(profile);

        if (headless) {
            options.addArguments("--headless");
            options.addArguments("--width=1920");
            options.addArguments("--height=1080");
        }

        return new FirefoxDriver(options);
    }
}
```

### Edge with Options

```java
import io.github.bonigarcia.wdm.WebDriverManager;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.edge.EdgeDriver;
import org.openqa.selenium.edge.EdgeOptions;

public class EdgeDriverSetup {

    public static WebDriver createEdgeDriver(boolean headless) {
        WebDriverManager.edgedriver().setup();

        EdgeOptions options = new EdgeOptions();
        options.addArguments("--start-maximized");
        options.addArguments("--disable-notifications");

        if (headless) {
            options.addArguments("--headless=new");
            options.addArguments("--window-size=1920,1080");
        }

        return new EdgeDriver(options);
    }
}
```

### Safari Driver

Safari's driver (`safaridriver`) comes pre-installed on macOS. No download is needed, but you must enable it once:

```bash
# Enable SafariDriver on macOS (run once)
safaridriver --enable

# For Safari Technology Preview (used for testing bleeding-edge web features)
/Applications/Safari\ Technology\ Preview.app/Contents/MacOS/safaridriver --enable
```

```java
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.safari.SafariDriver;
import org.openqa.selenium.safari.SafariOptions;

public class SafariDriverSetup {

    public static WebDriver createSafariDriver() {
        // No WebDriverManager needed for Safari
        SafariOptions options = new SafariOptions();
        // options.setUseTechnologyPreview(true); // For Safari Technology Preview
        return new SafariDriver(options);
    }
}
```

---

## Driver Factory Pattern

In a real framework, you never scatter driver creation across your codebase. Instead, create a **DriverFactory** class:

```java
package com.yourcompany.automation.utils;

import io.github.bonigarcia.wdm.WebDriverManager;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.edge.EdgeDriver;
import org.openqa.selenium.edge.EdgeOptions;
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.firefox.FirefoxOptions;

import java.util.Map;

public class DriverFactory {

    /**
     * Creates a WebDriver instance based on the browser name.
     *
     * @param browser  Browser name: "chrome", "firefox", "edge", "safari"
     * @param headless Whether to run in headless mode
     * @return Configured WebDriver instance
     */
    public static WebDriver createDriver(String browser, boolean headless) {
        return switch (browser.toLowerCase().trim()) {
            case "chrome" -> createChromeDriver(headless);
            case "firefox" -> createFirefoxDriver(headless);
            case "edge" -> createEdgeDriver(headless);
            default -> throw new IllegalArgumentException(
                "Unsupported browser: " + browser +
                ". Supported values: chrome, firefox, edge"
            );
        };
    }

    private static WebDriver createChromeDriver(boolean headless) {
        WebDriverManager.chromedriver().setup();
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--start-maximized", "--disable-notifications",
                             "--no-sandbox", "--disable-dev-shm-usage",
                             "--remote-allow-origins=*");
        if (headless) {
            options.addArguments("--headless=new", "--window-size=1920,1080");
        }
        return new ChromeDriver(options);
    }

    private static WebDriver createFirefoxDriver(boolean headless) {
        WebDriverManager.firefoxdriver().setup();
        FirefoxOptions options = new FirefoxOptions();
        if (headless) {
            options.addArguments("--headless", "--width=1920", "--height=1080");
        }
        return new FirefoxDriver(options);
    }

    private static WebDriver createEdgeDriver(boolean headless) {
        WebDriverManager.edgedriver().setup();
        EdgeOptions options = new EdgeOptions();
        options.addArguments("--start-maximized", "--disable-notifications");
        if (headless) {
            options.addArguments("--headless=new", "--window-size=1920,1080");
        }
        return new EdgeDriver(options);
    }
}
```

Usage:

```java
// Read from config
String browser = ConfigReader.get("browser");    // e.g., "chrome"
boolean headless = Boolean.parseBoolean(ConfigReader.get("headless")); // e.g., false

WebDriver driver = DriverFactory.createDriver(browser, headless);
```

---

## Common Driver Issues and Solutions

| Issue                                                 | Root Cause                              | Solution                                                           |
| ----------------------------------------------------- | --------------------------------------- | ------------------------------------------------------------------ |
| `SessionNotCreatedException: Chrome version mismatch` | Driver and browser version incompatible | Use WebDriverManager; ensure Chrome is up to date                  |
| `WebDriverException: cannot find Chrome binary`       | Chrome not installed at expected path   | Specify path via `options.setBinary(...)`                          |
| `Permission denied: ./chromedriver`                   | Driver not executable on Linux/macOS    | Run `chmod +x chromedriver`                                        |
| `Address already in use: 9515`                        | Previous driver process still running   | Kill orphaned chromedriver processes                               |
| `DevToolsActivePort file doesn't exist`               | Race condition on startup               | Add `--remote-allow-origins=*` to ChromeOptions                    |
| Corporate proxy blocks driver download                | Firewall restriction                    | Configure WebDriverManager proxy settings; or pre-download drivers |

---

## CI/CD Considerations

In CI/CD environments (Jenkins, GitHub Actions), browser and driver setup requires special care:

```bash
# GitHub Actions — use the pre-installed Chrome
# The runner already has Chrome and ChromeDriver at matching versions
# WebDriverManager will auto-detect and configure correctly

# Docker — always use headless mode
# options.addArguments("--headless=new");
# options.addArguments("--no-sandbox");
# options.addArguments("--disable-dev-shm-usage");
```

For Docker-based execution, the recommended approach is to use the **Selenium Grid Docker images** which bundle browsers and drivers together. This is covered in the CI/CD section.

---

## Summary

Browser drivers are the communication layer between Selenium and browsers. Each browser has its own driver, and version compatibility is critical. **WebDriverManager** eliminates manual driver management by automatically downloading, caching, and configuring the correct driver version. Use the **DriverFactory pattern** to centralize driver creation in your framework, making it easy to switch browsers via configuration without changing test code.
