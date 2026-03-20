# Chrome DevTools Protocol (CDP)

## Overview

The Chrome DevTools Protocol (CDP) is a low-level communication interface built into Chromium-based browsers (Chrome, Edge, Brave) that allows external tools to inspect, instrument, and control the browser at a depth that far exceeds what the standard WebDriver API provides. Selenium 4 exposes CDP access directly through the `ChromeDriver` and `EdgeDriver` classes, making advanced browser capabilities available from Java test code without any additional tooling.

CDP enables network interception, request mocking, performance profiling, JavaScript coverage analysis, geolocation spoofing, console log capture, dialog handling, cookie manipulation at the network level, and much more.

---

## How Selenium 4 Exposes CDP

Selenium 4 provides two layers of CDP access:

### Layer 1 — DevTools API (High-Level, Typed)

`ChromeDriver` implements `HasDevTools`, which gives access to a `DevTools` session object. The `DevTools` API provides typed Java classes for specific CDP domains: `Network`, `Page`, `Performance`, `Log`, `Security`, and others. This is the preferred approach — it is type-safe and IDE-friendly.

```java
ChromeDriver chromeDriver = (ChromeDriver) driver;
DevTools devTools = chromeDriver.getDevTools();
devTools.createSession();
```

### Layer 2 — executeCdpCommand (Low-Level, Raw)

`ChromeDriver` also exposes `executeCdpCommand()`, which accepts any CDP domain command as a string and a parameter map. This covers any CDP method not yet wrapped in the typed API.

```java
ChromeDriver chromeDriver = (ChromeDriver) driver;
Map<String, Object> params = new HashMap<>();
params.put("behavior", "allow");
params.put("downloadPath", "/tmp/downloads");
chromeDriver.executeCdpCommand("Page.setDownloadBehavior", params);
```

---

## Maven Dependency

The CDP API requires the `selenium-devtools-v*` artifact matching the Chrome version:

```xml
<!-- For Chrome 120 -->
<dependency>
    <groupId>org.seleniumhq.selenium</groupId>
    <artifactId>selenium-devtools-v120</artifactId>
    <version>4.18.1</version>
</dependency>
```

Check the Selenium changelog for the correct version artifact for your Chrome version. WebDriverManager can also automatically resolve the correct CDP version.

---

## Network Interception

Network interception is the most powerful CDP capability. It allows tests to monitor, block, mock, or modify every HTTP request and response made by the browser.

### Blocking Requests (Analytics, Ads, Trackers)

Block third-party scripts that slow tests or cause noise:

```java
public void blockExternalRequests(ChromeDriver driver, String... urlPatterns) {
    DevTools devTools = driver.getDevTools();
    devTools.createSession();
    devTools.send(Network.enable(Optional.empty(), Optional.empty(), Optional.empty()));

    devTools.send(Network.setBlockedURLs(Arrays.asList(urlPatterns)));
    log.info("Blocking URL patterns: {}", Arrays.toString(urlPatterns));
}

// Usage — block analytics and ads
blockExternalRequests(driver,
    "*google-analytics.com*",
    "*googletagmanager.com*",
    "*hotjar.com*",
    "*doubleclick.net*"
);
```

### Monitoring Network Requests and Responses

```java
public void captureNetworkTraffic(ChromeDriver driver) {
    DevTools devTools = driver.getDevTools();
    devTools.createSession();
    devTools.send(Network.enable(Optional.empty(), Optional.empty(), Optional.empty()));

    // Listen for every request
    devTools.addListener(Network.requestWillBeSent(), request -> {
        log.debug("REQUEST: {} {}", request.getRequest().getMethod(),
            request.getRequest().getUrl());
    });

    // Listen for every response
    devTools.addListener(Network.responseReceived(), response -> {
        log.debug("RESPONSE: {} {} {}",
            response.getResponse().getStatus(),
            response.getResponse().getUrl(),
            response.getResponse().getMimeType());
    });
}
```

### Mocking API Responses (Request Interception)

Intercept a specific API call and return a controlled mock response. This is invaluable for testing how the UI handles specific server responses (empty data, error codes, edge-case payloads) without needing the backend to produce those states.

```java
public void mockApiResponse(ChromeDriver driver, String urlPattern,
                              int statusCode, String responseBody) {
    DevTools devTools = driver.getDevTools();
    devTools.createSession();

    devTools.send(Fetch.enable(
        Optional.of(List.of(new RequestPattern(
            Optional.of(urlPattern),
            Optional.empty(),
            Optional.of(RequestStage.RESPONSE)
        ))),
        Optional.empty()
    ));

    devTools.addListener(Fetch.requestPaused(), request -> {
        devTools.send(Fetch.fulfillRequest(
            request.getRequestId(),
            statusCode,
            Optional.of(List.of(
                new HeaderEntry("Content-Type", "application/json"),
                new HeaderEntry("Access-Control-Allow-Origin", "*")
            )),
            Optional.empty(),
            Optional.of(responseBody),
            Optional.empty()
        ));
        log.info("Mocked response for: {} → {} ", urlPattern, statusCode);
    });
}
```

**Usage — test empty state UI:**

```java
// Mock the products API to return an empty list
mockApiResponse(chromeDriver,
    "**/api/v1/products*",
    200,
    "{\"products\": [], \"total\": 0}"
);
driver.get(baseUrl + "/products");
// Assert empty state message is displayed
```

**Usage — test server error handling:**

```java
// Mock the orders API to return 500
mockApiResponse(chromeDriver,
    "**/api/v1/orders*",
    500,
    "{\"error\": \"Internal server error\"}"
);
driver.get(baseUrl + "/orders");
// Assert error message is displayed to user
```

### Simulating Slow Network (Throttling)

```java
public void simulateSlowNetwork(ChromeDriver driver, int latencyMs,
                                  int downloadThroughputBps, int uploadThroughputBps) {
    DevTools devTools = driver.getDevTools();
    devTools.createSession();
    devTools.send(Network.enable(Optional.empty(), Optional.empty(), Optional.empty()));

    devTools.send(Network.emulateNetworkConditions(
        false,              // offline
        latencyMs,          // added latency in ms
        downloadThroughputBps,  // download bytes/sec (-1 to disable)
        uploadThroughputBps,    // upload bytes/sec (-1 to disable)
        Optional.empty()
    ));

    log.info("Network throttled: {}ms latency, {}bps download", latencyMs, downloadThroughputBps);
}

// Simulate 3G
simulateSlowNetwork(driver, 200, 750_000, 250_000);   // 200ms latency, ~750 KB/s down

// Simulate offline
devTools.send(Network.emulateNetworkConditions(
    true, 0, 0, 0, Optional.empty()  // offline=true
));
```

### Setting Custom Request Headers

Inject custom headers into every outgoing request — useful for bypassing authentication middleware in test environments, or adding test trace IDs.

```java
public void setExtraRequestHeaders(ChromeDriver driver, Map<String, String> headers) {
    DevTools devTools = driver.getDevTools();
    devTools.createSession();
    devTools.send(Network.enable(Optional.empty(), Optional.empty(), Optional.empty()));
    devTools.send(Network.setExtraHTTPHeaders(new Headers(headers)));
    log.info("Extra headers set: {}", headers);
}

// Usage
Map<String, String> headers = new HashMap<>();
headers.put("X-Test-Environment", "automation");
headers.put("X-Bypass-Rate-Limit", "true");
setExtraRequestHeaders(chromeDriver, headers);
```

---

## Console Log Capture

Capture all browser console output during test execution. Useful for detecting JavaScript errors that would otherwise be invisible to Selenium.

```java
public void enableConsoleLogCapture(ChromeDriver driver) {
    DevTools devTools = driver.getDevTools();
    devTools.createSession();
    devTools.send(Log.enable());

    devTools.addListener(Log.entryAdded(), entry -> {
        String level  = entry.getLevel().toString();
        String source = entry.getSource().toString();
        String text   = entry.getText();

        if (level.equals("error")) {
            log.error("BROWSER CONSOLE ERROR [{}]: {}", source, text);
        } else if (level.equals("warning")) {
            log.warn("BROWSER CONSOLE WARN [{}]: {}", source, text);
        } else {
            log.debug("BROWSER CONSOLE [{}]: {}", source, text);
        }
    });
}
```

**Collecting and asserting on console errors:**

```java
private List<String> consoleErrors = new ArrayList<>();

public void startCapturingConsoleErrors(ChromeDriver driver) {
    DevTools devTools = driver.getDevTools();
    devTools.createSession();
    devTools.send(Log.enable());

    devTools.addListener(Log.entryAdded(), entry -> {
        if (entry.getLevel().toString().equals("error")) {
            consoleErrors.add(entry.getText());
        }
    });
}

@Then("no JavaScript errors should have occurred")
public void noJavaScriptErrorsShouldHaveOccurred() {
    Assert.assertTrue(consoleErrors.isEmpty(),
        "JavaScript console errors detected:\n" + String.join("\n", consoleErrors));
}
```

---

## Geolocation Spoofing

Override the browser's geolocation for testing location-based features.

```java
public void setGeolocation(ChromeDriver driver, double latitude, double longitude, double accuracy) {
    DevTools devTools = driver.getDevTools();
    devTools.createSession();

    devTools.send(Emulation.setGeolocationOverride(
        Optional.of(latitude),
        Optional.of(longitude),
        Optional.of(accuracy)
    ));

    log.info("Geolocation set: lat={}, lon={}", latitude, longitude);
}

// Simulate being in New York
setGeolocation(chromeDriver, 40.7128, -74.0060, 1.0);

// Simulate being in London
setGeolocation(chromeDriver, 51.5074, -0.1278, 1.0);

// Reset to real location
devTools.send(Emulation.clearGeolocationOverride());
```

---

## CPU and Network Performance Throttling

```java
public void throttleCPU(ChromeDriver driver, int slowdownFactor) {
    DevTools devTools = driver.getDevTools();
    devTools.createSession();
    devTools.send(Emulation.setCPUThrottlingRate(slowdownFactor));
    log.info("CPU throttled {}x", slowdownFactor);
}

// Simulate a low-end mobile device (6x CPU slowdown)
throttleCPU(chromeDriver, 6);
```

---

## JavaScript Dialog Handling

Auto-dismiss or auto-accept JavaScript dialogs (alert, confirm, prompt) — useful in headless mode or to prevent dialogs from blocking tests.

```java
public void autoAcceptDialogs(ChromeDriver driver) {
    DevTools devTools = driver.getDevTools();
    devTools.createSession();

    devTools.addListener(Page.javascriptDialogOpening(), dialog -> {
        log.info("Auto-accepting JS dialog: [{}] '{}'",
            dialog.getType(), dialog.getMessage());
        devTools.send(Page.handleJavaScriptDialog(true, Optional.empty()));
    });
}

public void autoDismissDialogs(ChromeDriver driver) {
    DevTools devTools = driver.getDevTools();
    devTools.createSession();

    devTools.addListener(Page.javascriptDialogOpening(), dialog -> {
        log.info("Auto-dismissing JS dialog: [{}]", dialog.getType());
        devTools.send(Page.handleJavaScriptDialog(false, Optional.empty()));
    });
}
```

---

## Performance Metrics

Capture Core Web Vitals and performance metrics directly from the browser:

```java
public Map<String, Number> getPerformanceMetrics(ChromeDriver driver) {
    DevTools devTools = driver.getDevTools();
    devTools.createSession();
    devTools.send(Performance.enable(Optional.empty()));

    List<Metric> metrics = devTools.send(Performance.getMetrics());

    Map<String, Number> result = new HashMap<>();
    metrics.forEach(m -> result.put(m.getName(), m.getValue()));

    log.info("Performance metrics captured: {}", result.keySet());
    return result;
}

// Usage
Map<String, Number> metrics = getPerformanceMetrics(chromeDriver);
Number domContentLoaded = metrics.get("DomContentLoaded");
Number firstMeaningfulPaint = metrics.get("FirstMeaningfulPaint");

Assert.assertTrue(firstMeaningfulPaint.doubleValue() < 2.0,
    "First Meaningful Paint exceeded 2 seconds: " + firstMeaningfulPaint);
```

---

## Complete CDPUtils Class

```java
package com.company.framework.utils;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.v120.emulation.Emulation;
import org.openqa.selenium.devtools.v120.log.Log;
import org.openqa.selenium.devtools.v120.network.Network;
import org.openqa.selenium.devtools.v120.network.model.Headers;
import org.openqa.selenium.devtools.v120.page.Page;
import org.openqa.selenium.devtools.v120.performance.Performance;
import org.openqa.selenium.devtools.v120.performance.model.Metric;

import java.util.*;

public class CDPUtils {

    private static final Logger log = LogManager.getLogger(CDPUtils.class);

    private CDPUtils() {}

    public static DevTools createSession(ChromeDriver driver) {
        DevTools devTools = driver.getDevTools();
        devTools.createSession();
        return devTools;
    }

    public static void enableNetwork(DevTools devTools) {
        devTools.send(Network.enable(Optional.empty(), Optional.empty(), Optional.empty()));
    }

    public static void blockURLs(DevTools devTools, String... patterns) {
        enableNetwork(devTools);
        devTools.send(Network.setBlockedURLs(Arrays.asList(patterns)));
        log.info("CDP: Blocked URLs: {}", Arrays.toString(patterns));
    }

    public static void setExtraHeaders(DevTools devTools, Map<String, String> headers) {
        enableNetwork(devTools);
        devTools.send(Network.setExtraHTTPHeaders(new Headers(headers)));
        log.info("CDP: Extra headers set: {}", headers.keySet());
    }

    public static void setGeolocation(DevTools devTools,
                                       double lat, double lon, double accuracy) {
        devTools.send(Emulation.setGeolocationOverride(
            Optional.of(lat), Optional.of(lon), Optional.of(accuracy)));
        log.info("CDP: Geolocation set to ({}, {})", lat, lon);
    }

    public static void clearGeolocation(DevTools devTools) {
        devTools.send(Emulation.clearGeolocationOverride());
    }

    public static void throttleCPU(DevTools devTools, int factor) {
        devTools.send(Emulation.setCPUThrottlingRate(factor));
        log.info("CDP: CPU throttled {}x", factor);
    }

    public static void autoAcceptDialogs(DevTools devTools) {
        devTools.addListener(Page.javascriptDialogOpening(), dialog ->
            devTools.send(Page.handleJavaScriptDialog(true, Optional.empty())));
    }

    public static void captureConsoleErrors(DevTools devTools, List<String> errorList) {
        devTools.send(Log.enable());
        devTools.addListener(Log.entryAdded(), entry -> {
            if (entry.getLevel().toString().equals("error")) {
                errorList.add(entry.getText());
                log.error("BROWSER CONSOLE ERROR: {}", entry.getText());
            }
        });
    }

    public static Map<String, Number> getPerformanceMetrics(DevTools devTools) {
        devTools.send(Performance.enable(Optional.empty()));
        List<Metric> metrics = devTools.send(Performance.getMetrics());
        Map<String, Number> result = new HashMap<>();
        metrics.forEach(m -> result.put(m.getName(), m.getValue()));
        return result;
    }

    public static void setDownloadBehavior(ChromeDriver driver, String downloadDir) {
        Map<String, Object> params = new HashMap<>();
        params.put("behavior", "allow");
        params.put("downloadPath", new java.io.File(downloadDir).getAbsolutePath());
        driver.executeCdpCommand("Page.setDownloadBehavior", params);
        log.info("CDP: Download path set to {}", downloadDir);
    }
}
```

---

## When to Use CDP vs Standard WebDriver

| Requirement                    | Standard WebDriver | CDP                |
| ------------------------------ | ------------------ | ------------------ |
| Click, type, navigate          | Yes                | Not needed         |
| Wait for elements              | Yes                | Not needed         |
| Screenshots                    | Yes                | Not needed         |
| Network monitoring             | No                 | Yes                |
| Request mocking                | No                 | Yes                |
| Block third-party scripts      | No                 | Yes                |
| Console error capture          | Partial (JS)       | Yes                |
| Geolocation spoofing           | No                 | Yes                |
| Network throttling             | No                 | Yes                |
| Performance metrics            | Partial            | Yes                |
| File download control headless | Limited            | Yes                |
| Dialog auto-handling           | driver.switchTo()  | Yes (more control) |

---

## Summary

The Chrome DevTools Protocol gives Selenium tests direct access to the browser's internal capabilities. Network interception enables request mocking for controlled UI state testing. Console log capture surfaces JavaScript errors that standard Selenium misses. Geolocation and CPU throttling enable device simulation. Performance metrics enable basic Web Vitals assertions. Used in combination with standard WebDriver, CDP makes Chrome and Edge test automation comprehensive enough to handle the most demanding modern web application testing scenarios.

---

## Related Topics

- JavaScriptExecutor.md — JS execution for DOM manipulation
- HeadlessTesting.md — CDP for download configuration in headless mode
- FileUploadDownload.md — CDP download behavior settings
- DriverManager.md — ChromeDriver cast required for CDP access
- ParallelExecution.md — CDP sessions are per-driver (thread-safe by design)
