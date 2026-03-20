# Headless Browser Testing

## Overview

Headless browser testing runs Selenium tests in a browser that has no visible graphical interface. The browser engine, JavaScript engine, DOM, CSS rendering, and network stack all operate normally — but no window appears on screen. Headless mode is the standard configuration for running automated tests in CI/CD pipelines where no display server is available, and it is significantly faster than running a visible browser because it skips the overhead of rendering pixels to a physical screen.

---

## Why Headless Testing Matters

**CI/CD environments have no display.** Linux servers, Docker containers, and cloud build agents typically run without a graphical desktop environment. Headless mode is required to run Selenium tests in these environments without setting up a virtual display (like Xvfb).

**Headless runs are faster.** Eliminating GUI rendering reduces CPU and memory usage. Test suites often run 20–40% faster in headless mode.

**Parallel execution scales better.** Running 10 parallel headless browser instances consumes significantly fewer resources than 10 visible browser windows.

**Screenshot and report capabilities are fully preserved.** Headless browsers still support `TakesScreenshot`, enabling failure screenshots in reports.

---

## Chrome Headless

Chrome has supported headless mode since version 59. Selenium 4 uses the `--headless=new` flag, which replaced the older `--headless` flag in Chrome 112+. The `new` headless mode uses the same renderer as the visible Chrome, making it more accurate than the legacy headless implementation.

### Basic Headless Chrome Setup

```java
public static WebDriver createHeadlessChrome() {
    WebDriverManager.chromedriver().setup();

    ChromeOptions options = new ChromeOptions();
    options.addArguments("--headless=new");
    options.addArguments("--window-size=1920,1080");   // required — no default size in headless
    options.addArguments("--no-sandbox");               // required in Docker/Linux containers
    options.addArguments("--disable-dev-shm-usage");    // prevents crashes in containers
    options.addArguments("--disable-gpu");              // stability on older Linux systems
    options.addArguments("--disable-extensions");
    options.addArguments("--disable-notifications");
    options.addArguments("--disable-infobars");
    options.addArguments("--remote-debugging-port=9222"); // useful for local debugging

    return new ChromeDriver(options);
}
```

### Why Each Flag Matters

| Flag                      | Reason                                                                 |
| ------------------------- | ---------------------------------------------------------------------- |
| `--headless=new`          | Enable headless mode using the modern renderer                         |
| `--window-size=1920,1080` | Set a fixed viewport; headless has no default window size              |
| `--no-sandbox`            | Required in Docker where Chrome cannot use OS sandbox                  |
| `--disable-dev-shm-usage` | Prevents Chrome from using `/dev/shm` which is too small in containers |
| `--disable-gpu`           | Avoids GPU-related crashes on Linux without GPU acceleration           |
| `--remote-debugging-port` | Enables CDP connection for debugging while headless                    |

---

## Firefox Headless

```java
public static WebDriver createHeadlessFirefox() {
    WebDriverManager.firefoxdriver().setup();

    FirefoxOptions options = new FirefoxOptions();
    options.addArguments("-headless");
    options.addArguments("--width=1920");
    options.addArguments("--height=1080");

    return new FirefoxDriver(options);
}
```

Firefox headless is slightly less commonly used than Chrome headless but works equivalently. Firefox `-headless` does not require `--no-sandbox` or shared memory adjustments.

---

## Integrating Headless Mode with DriverFactory

In a professional framework, headless is controlled by a configuration property, not by hardcoding. Any test run can be switched to headless without changing code.

```java
public static WebDriver createDriver() {
    String browser = System.getProperty("browser",
        ConfigReader.get("browser", "chrome")).toLowerCase();
    boolean headless = ConfigReader.getBoolean("headless", false);

    // Headless flag overrides the browser name
    if (headless) {
        browser = browser.contains("firefox") ? "firefox-headless" : "chrome-headless";
    }

    switch (browser) {
        case "chrome":           return createChromeDriver(false);
        case "chrome-headless":  return createChromeDriver(true);
        case "firefox":          return createFirefoxDriver(false);
        case "firefox-headless": return createFirefoxDriver(true);
        default:                 return createChromeDriver(false);
    }
}

private static WebDriver createChromeDriver(boolean headless) {
    WebDriverManager.chromedriver().setup();
    ChromeOptions options = new ChromeOptions();

    options.addArguments("--window-size=1920,1080");
    options.addArguments("--no-sandbox");
    options.addArguments("--disable-dev-shm-usage");
    options.addArguments("--disable-gpu");
    options.addArguments("--disable-notifications");
    options.addArguments("--disable-extensions");

    if (headless) {
        options.addArguments("--headless=new");
        log.info("Chrome running in HEADLESS mode");
    }

    return new ChromeDriver(options);
}
```

**Running headless from the command line:**

```bash
# Run in headless mode
mvn test -Dheadless=true

# Run in headless mode on staging
mvn test -Denv=staging -Dheadless=true

# CI/CD pipeline — always headless
mvn test -Dheadless=true -Dcucumber.filter.tags="@regression"
```

---

## Screenshots in Headless Mode

Screenshots work identically in headless and visible modes. `TakesScreenshot` is fully supported.

```java
@After
public void captureScreenshotOnFailure(Scenario scenario) {
    if (scenario.isFailed()) {
        byte[] screenshot = ((TakesScreenshot) driver)
            .getScreenshotAs(OutputType.BYTES);
        scenario.attach(screenshot, "image/png", "Failure screenshot");
        // Screenshot is embedded in the Cucumber/Extent/Allure report
    }
}
```

Full-page screenshots (capturing content below the fold) require a separate approach since the viewport size determines what is captured:

```java
// Expand the window to the full page height before screenshotting
public byte[] takeFullPageScreenshot(WebDriver driver) {
    JavascriptExecutor js = (JavascriptExecutor) driver;
    Long pageHeight = (Long) js.executeScript("return document.body.scrollHeight;");

    driver.manage().window().setSize(
        new Dimension(1920, pageHeight.intValue())
    );

    byte[] screenshot = ((TakesScreenshot) driver).getScreenshotAs(OutputType.BYTES);

    // Restore original size
    driver.manage().window().setSize(new Dimension(1920, 1080));

    return screenshot;
}
```

---

## File Downloads in Headless Chrome

Headless Chrome requires a CDP command to enable file downloads, in addition to the standard preferences:

```java
public static WebDriver createHeadlessChromeWithDownloads(String downloadDir) {
    new File(downloadDir).mkdirs();

    ChromeOptions options = new ChromeOptions();
    options.addArguments("--headless=new");
    options.addArguments("--window-size=1920,1080");
    options.addArguments("--no-sandbox");
    options.addArguments("--disable-dev-shm-usage");

    Map<String, Object> prefs = new HashMap<>();
    prefs.put("download.default_directory", new File(downloadDir).getAbsolutePath());
    prefs.put("download.prompt_for_download", false);
    prefs.put("safebrowsing.enabled", false);
    options.setExperimentalOption("prefs", prefs);

    WebDriverManager.chromedriver().setup();
    ChromeDriver driver = new ChromeDriver(options);

    // CDP command enables download in headless mode
    Map<String, Object> params = new HashMap<>();
    params.put("behavior", "allow");
    params.put("downloadPath", new File(downloadDir).getAbsolutePath());
    driver.executeCdpCommand("Page.setDownloadBehavior", params);

    return driver;
}
```

---

## Docker Configuration

Running Selenium tests in Docker is the most common CI/CD deployment. The container must have Chrome or Firefox installed, or use a pre-built Selenium image.

### Dockerfile for Chrome Headless

```dockerfile
FROM ubuntu:22.04

# Install dependencies
RUN apt-get update && apt-get install -y \
    wget curl unzip gnupg2 \
    libgconf-2-4 libglib2.0-0 libnss3 \
    libxss1 libatk1.0-0 libatk-bridge2.0-0 \
    libcups2 libdrm2 libxkbcommon0 libxcomposite1 \
    libxrandr2 libgbm1 libasound2 \
    openjdk-11-jdk maven \
    && rm -rf /var/lib/apt/lists/*

# Install Chrome
RUN wget -q https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb \
    && dpkg -i google-chrome-stable_current_amd64.deb; apt-get -fy install \
    && rm google-chrome-stable_current_amd64.deb

WORKDIR /app
COPY . .

CMD ["mvn", "test", "-Dheadless=true", "-Dcucumber.filter.tags=@regression"]
```

### Using Official Selenium Docker Images

The Selenium project provides ready-to-use Docker images:

```yaml
# docker-compose.yml
version: "3.8"
services:
  selenium-chrome:
    image: selenium/standalone-chrome:4.18.1
    ports:
      - "4444:4444"
    environment:
      - SE_NODE_MAX_SESSIONS=3
    volumes:
      - /dev/shm:/dev/shm # required for Chrome stability

  test-runner:
    image: maven:3.9-openjdk-11
    depends_on:
      - selenium-chrome
    volumes:
      - .:/app
    working_dir: /app
    command: mvn test -Dbrowser=remote -Dgrid.url=http://selenium-chrome:4444
```

---

## GitHub Actions CI Configuration

```yaml
name: Regression Tests

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK 11
        uses: actions/setup-java@v4
        with:
          java-version: "11"
          distribution: "temurin"

      - name: Install Chrome
        uses: browser-actions/setup-chrome@v1

      - name: Run Regression Tests
        run: mvn test -Dheadless=true -Dcucumber.filter.tags="@regression"
        env:
          APP_BASE_URL: ${{ secrets.STAGING_URL }}

      - name: Upload Test Reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: cucumber-reports
          path: target/cucumber-reports/

      - name: Upload Screenshots
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: failure-screenshots
          path: target/screenshots/
```

---

## Debugging Headless Test Failures

When a test fails in headless mode but passes visibly, the problem is usually one of these:

### Issue 1 — Element Not Found Due to Viewport Size

Headless mode with no `--window-size` flag defaults to 800x600. Elements that are only rendered above a CSS breakpoint will not exist.

**Fix:** Always set `--window-size=1920,1080` or match the viewport size expected by the application.

### Issue 2 — Font Rendering Differences

Some XPath expressions based on exact text fail in headless because fonts render slightly differently, causing line breaks in unexpected places.

**Fix:** Use `normalize-space()` in XPath: `//button[normalize-space()='Submit']`

### Issue 3 — Timing Differences

Headless runs faster. An element may appear faster than the wait timeout accommodates in visible mode, or DOM events may fire in a slightly different order.

**Fix:** Use explicit WebDriverWait for every interaction. Never use fixed `Thread.sleep()`.

### Issue 4 — JavaScript Dialogs

`window.alert()`, `window.confirm()`, and `window.prompt()` behave differently in headless mode. They may auto-dismiss.

**Fix:** Use `driver.switchTo().alert()` for alerts, or configure CDP to handle dialogs automatically.

### Comparing Headless vs Visible

If a test fails only in headless mode:

1. Take a screenshot on failure — the screenshot shows the exact state of the headless browser at the moment of failure.
2. Compare the failure screenshot with what the visible browser shows at the same step.
3. Check the window size: `driver.manage().window().getSize()`.
4. Add `log.debug` statements to the relevant page methods to trace what elements are found.

---

## Performance Comparison

| Metric             | Visible Chrome | Headless Chrome | Improvement             |
| ------------------ | -------------- | --------------- | ----------------------- |
| Test startup time  | ~2.5s          | ~1.5s           | ~40% faster             |
| Memory per session | ~150 MB        | ~90 MB          | ~40% less               |
| CPU during test    | Higher         | Lower           | Significant in parallel |
| Screenshot support | Yes            | Yes             | Same                    |
| JavaScript support | Full           | Full            | Same                    |
| Cookie management  | Full           | Full            | Same                    |
| File downloads     | Yes            | Needs CDP       | Minor config diff       |

---

## Summary

Headless browser testing is the standard mode for CI/CD pipeline execution. Chrome headless with `--headless=new` and a fixed `--window-size` provides a rendering environment equivalent to the visible browser. Firefox headless requires minimal additional configuration. Both integrate seamlessly with DriverFactory through a single `headless` configuration flag. Screenshots, cookies, JavaScript execution, and the full Selenium API work identically in headless mode. The primary operational consideration is container-specific flags (`--no-sandbox`, `--disable-dev-shm-usage`) and explicit window size configuration.

---

## Related Topics

- DriverManager.md — DriverFactory where headless configuration is applied
- ConfigProperties.md — `headless` property and `-Dheadless=true` override
- FileUploadDownload.md — CDP download configuration in headless mode
- CDP.md — Chrome DevTools Protocol for headless debugging and network control
- Docker.md — Container setup for headless Selenium execution
