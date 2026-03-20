# Selenium Grid

## Overview

Selenium Grid is the official distributed test execution infrastructure provided by the Selenium project. It allows you to run Selenium tests on multiple machines simultaneously, across multiple browsers, browser versions, and operating systems — all from a single test suite. Instead of every test running on the developer's local machine, the Grid acts as a central hub that receives test requests and routes them to available browser nodes.

Selenium Grid 4 (the current version) was rewritten from scratch with a modern architecture that supports both distributed and standalone deployment modes, Docker-native operation, observability through OpenTelemetry, and dynamic node registration.

---

## Grid Architecture

### Components

**Router** — The entry point for all incoming WebDriver requests. Receives session creation requests and routes them to the Distributor.

**Distributor** — Maintains an inventory of available nodes and their capabilities. Assigns new sessions to the most suitable available node slot.

**Session Map** — Tracks the mapping between session ID and the node hosting that session. Routes subsequent requests for an existing session to the correct node.

**Node** — A machine that runs browser instances. Each node registers with the Distributor and advertises its capabilities (browser name, version, platform). A node can host multiple browser slots.

**Event Bus** — The message queue that all components use to communicate asynchronously.

### Deployment Modes

| Mode              | Use Case                            | Components                            |
| ----------------- | ----------------------------------- | ------------------------------------- |
| Standalone        | Local development, single machine   | All components in one process         |
| Hub-Node          | Classic two-tier, small teams       | Hub (Router+Distributor+Map) + Nodes  |
| Fully Distributed | Large enterprises, cloud deployment | Each component deployed independently |
| Docker            | CI/CD pipelines                     | Official Selenium Docker images       |

---

## Standalone Mode (Single Machine)

Standalone mode runs all Grid components in one process. It is the quickest way to get a Grid running locally and is ideal for verifying Grid configuration before scaling up.

### Download and Start

```bash
# Download the Selenium Server JAR
wget https://github.com/SeleniumHQ/selenium/releases/download/selenium-4.18.1/selenium-server-4.18.1.jar

# Start standalone Grid on port 4444
java -jar selenium-server-4.18.1.jar standalone --port 4444
```

Grid UI is available at: `http://localhost:4444`

### Configure Concurrent Sessions

```bash
java -jar selenium-server-4.18.1.jar standalone \
  --port 4444 \
  --max-sessions 5 \
  --session-timeout 300
```

---

## Hub-Node Mode (Classic Distributed)

The Hub-Node architecture is the most common setup for small to medium teams. The Hub receives test requests; Nodes execute them.

### Start the Hub

```bash
java -jar selenium-server-4.18.1.jar hub --port 4444
```

### Start a Chrome Node

```bash
java -jar selenium-server-4.18.1.jar node \
  --port 5555 \
  --hub http://localhost:4444 \
  --max-sessions 3 \
  --override-max-sessions true
```

### Start a Firefox Node

```bash
java -jar selenium-server-4.18.1.jar node \
  --port 5556 \
  --hub http://localhost:4444 \
  --max-sessions 3
```

### Node Configuration File

For production nodes, use a JSON configuration file instead of command-line flags:

```json
{
  "sessionTimeout": 300,
  "enableManagedDownloads": true,
  "node": {
    "implementation": "org.openqa.selenium.grid.node.local.LocalNode",
    "driverImplementation": "org.openqa.selenium.chrome.ChromeDriver",
    "maxSessions": 5,
    "stereotypes": [
      {
        "config": {
          "slots": 3,
          "stereotype": {
            "browserName": "chrome",
            "browserVersion": "120",
            "platformName": "linux"
          }
        }
      },
      {
        "config": {
          "slots": 2,
          "stereotype": {
            "browserName": "firefox",
            "browserVersion": "121",
            "platformName": "linux"
          }
        }
      }
    ]
  }
}
```

```bash
java -jar selenium-server-4.18.1.jar node --config node-config.json
```

---

## Connecting Tests to Selenium Grid

### DriverFactory — Remote WebDriver

Replace local `ChromeDriver()` with `RemoteWebDriver` pointing to the Grid URL:

```java
package com.company.framework.driver;

import com.company.framework.config.ConfigReader;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.firefox.FirefoxOptions;
import org.openqa.selenium.remote.RemoteWebDriver;

import java.net.MalformedURLException;
import java.net.URL;
import java.time.Duration;

public class DriverFactory {

    private static final Logger log = LogManager.getLogger(DriverFactory.class);

    public static WebDriver createDriver() {
        boolean useGrid = ConfigReader.getBoolean("grid.enabled", false);
        String browser  = System.getProperty("browser",
            ConfigReader.get("browser", "chrome")).toLowerCase();

        if (useGrid) {
            return createRemoteDriver(browser);
        }

        return createLocalDriver(browser);
    }

    private static WebDriver createRemoteDriver(String browser) {
        String gridUrl = ConfigReader.get("grid.url", "http://localhost:4444");
        log.info("Connecting to Selenium Grid at: {} with browser: {}", gridUrl, browser);

        try {
            URL remoteUrl = new URL(gridUrl);

            switch (browser) {
                case "firefox":
                    FirefoxOptions ffOptions = new FirefoxOptions();
                    ffOptions.addArguments("-headless");
                    return configureTimeouts(new RemoteWebDriver(remoteUrl, ffOptions));

                case "edge":
                    org.openqa.selenium.edge.EdgeOptions edgeOptions =
                        new org.openqa.selenium.edge.EdgeOptions();
                    return configureTimeouts(new RemoteWebDriver(remoteUrl, edgeOptions));

                case "chrome":
                default:
                    ChromeOptions chromeOptions = new ChromeOptions();
                    chromeOptions.addArguments("--no-sandbox");
                    chromeOptions.addArguments("--disable-dev-shm-usage");
                    chromeOptions.addArguments("--headless=new");
                    chromeOptions.addArguments("--window-size=1920,1080");
                    return configureTimeouts(new RemoteWebDriver(remoteUrl, chromeOptions));
            }

        } catch (MalformedURLException e) {
            throw new RuntimeException("Invalid Selenium Grid URL: " + gridUrl, e);
        }
    }

    private static WebDriver configureTimeouts(WebDriver driver) {
        driver.manage().timeouts()
            .implicitlyWait(Duration.ofSeconds(0))
            .pageLoadTimeout(Duration.ofSeconds(30))
            .scriptTimeout(Duration.ofSeconds(30));
        return driver;
    }

    // ... createLocalDriver() remains unchanged
}
```

### config.properties Grid Settings

```properties
# Selenium Grid
grid.enabled=true
grid.url=http://selenium-hub:4444

# Browser for Grid runs
browser=chrome
```

---

## Grid With Docker Compose

The most production-ready Grid setup uses Docker Compose with official Selenium images. This eliminates manual JAR management, browser installation, and port configuration.

```yaml
# docker-compose-grid.yml
version: "3.8"

services:
  selenium-hub:
    image: selenium/hub:4.18.1
    container_name: selenium-hub
    ports:
      - "4444:4444"
      - "4442:4442"
      - "4443:4443"
    environment:
      - GRID_MAX_SESSION=10
      - GRID_BROWSER_TIMEOUT=300
      - GRID_TIMEOUT=300

  chrome-node-1:
    image: selenium/node-chrome:4.18.1
    container_name: chrome-node-1
    depends_on:
      - selenium-hub
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
      - SE_NODE_MAX_SESSIONS=3
      - SE_NODE_OVERRIDE_MAX_SESSIONS=true
      - SE_VNC_NO_PASSWORD=1
    volumes:
      - /dev/shm:/dev/shm
    shm_size: "2gb"

  chrome-node-2:
    image: selenium/node-chrome:4.18.1
    container_name: chrome-node-2
    depends_on:
      - selenium-hub
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
      - SE_NODE_MAX_SESSIONS=3
      - SE_VNC_NO_PASSWORD=1
    volumes:
      - /dev/shm:/dev/shm
    shm_size: "2gb"

  firefox-node:
    image: selenium/node-firefox:4.18.1
    container_name: firefox-node
    depends_on:
      - selenium-hub
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
      - SE_NODE_MAX_SESSIONS=2
    volumes:
      - /dev/shm:/dev/shm
    shm_size: "2gb"

  test-runner:
    image: maven:3.9-openjdk-11
    container_name: test-runner
    depends_on:
      - chrome-node-1
      - chrome-node-2
      - firefox-node
    environment:
      - GRID_URL=http://selenium-hub:4444
    volumes:
      - .:/app
      - ~/.m2:/root/.m2
    working_dir: /app
    command: >
      mvn test
      -Dgrid.enabled=true
      -Dgrid.url=http://selenium-hub:4444
      -Dheadless=true
      -Dcucumber.filter.tags="@regression"
```

**Start the Grid:**

```bash
docker-compose -f docker-compose-grid.yml up -d --scale chrome-node-1=3
```

**Wait for nodes to register, then run tests:**

```bash
docker-compose -f docker-compose-grid.yml run test-runner
```

---

## Grid with Video Recording

Selenium Grid nodes support VNC and video recording out of the box using the `selenium/node-chrome-video` image:

```yaml
chrome-node-video:
  image: selenium/node-chrome:4.18.1
  container_name: chrome-video-node
  depends_on:
    - selenium-hub
  environment:
    - SE_EVENT_BUS_HOST=selenium-hub
    - SE_EVENT_BUS_PUBLISH_PORT=4442
    - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
    - SE_ENABLE_TRACING=false
  volumes:
    - /dev/shm:/dev/shm
    - ./recordings:/home/seluser/recordings
  shm_size: "2gb"

video-recorder:
  image: selenium/video:ffmpeg-4.3.1-20230920
  volumes:
    - ./recordings:/videos
  depends_on:
    - chrome-node-video
  environment:
    - DISPLAY_CONTAINER_NAME=chrome-video-node
    - FILE_NAME=test_recording.mp4
```

---

## Grid Observability and Monitoring

Selenium Grid 4 exposes a REST API and a built-in UI for monitoring:

```bash
# Grid status — JSON API
curl http://localhost:4444/status

# Available nodes and sessions
curl http://localhost:4444/grid/api/hub/nodes

# Grid UI
open http://localhost:4444
```

**Grid Status Response:**

```json
{
  "value": {
    "ready": true,
    "message": "Selenium Grid ready.",
    "nodes": [
      {
        "id": "node-uuid-123",
        "uri": "http://chrome-node-1:5555",
        "maxSessions": 3,
        "stereotypes": [{ "browserName": "chrome", "count": 3 }],
        "sessionCount": 1
      }
    ]
  }
}
```

---

## Waiting for Grid Readiness

In automated pipelines, nodes take a few seconds to register after startup. Always wait for the Grid to be ready before running tests:

```bash
#!/bin/bash
# wait-for-grid.sh
GRID_URL=${1:-"http://localhost:4444"}
MAX_ATTEMPTS=30
ATTEMPT=0

echo "Waiting for Selenium Grid at $GRID_URL..."

until curl -sSf "$GRID_URL/status" | grep -q '"ready":true'; do
    ATTEMPT=$((ATTEMPT + 1))
    if [ $ATTEMPT -ge $MAX_ATTEMPTS ]; then
        echo "Grid not ready after $MAX_ATTEMPTS attempts. Exiting."
        exit 1
    fi
    echo "Attempt $ATTEMPT/$MAX_ATTEMPTS — Grid not ready yet. Waiting 5s..."
    sleep 5
done

echo "Selenium Grid is ready."
```

```bash
chmod +x wait-for-grid.sh
./wait-for-grid.sh http://localhost:4444
mvn test -Dgrid.enabled=true
```

---

## Session Request Capabilities

When connecting to Grid, specify browser capabilities to target the exact node type needed:

```java
ChromeOptions options = new ChromeOptions();
options.setCapability("platformName", "linux");
options.setCapability("browserVersion", "120");

// Selenium 4 — use standard capabilities
options.setCapability("se:recordVideo", true);      // request video recording
options.setCapability("se:name", "Login Smoke Test"); // session name in Grid UI

RemoteWebDriver driver = new RemoteWebDriver(new URL(gridUrl), options);
```

---

## Summary

Selenium Grid provides the distributed infrastructure needed to run large test suites quickly across multiple browsers. Standalone mode serves local development. Hub-Node mode serves small to medium teams. Docker Compose with official Selenium images provides the simplest, most reproducible CI/CD Grid deployment. The Grid integrates with the framework through a single configuration flag — `grid.enabled=true` — which switches `DriverFactory` from creating local drivers to creating `RemoteWebDriver` instances pointed at the Grid URL.

---

## Related Topics

- ParallelExecution.md — Parallel test configuration that maximizes Grid utilization
- Docker.md — Full Docker setup for Grid and test runner containers
- Jenkins.md — Jenkins pipeline that starts and tears down Grid
- GitHubActions.md — GitHub Actions with Selenium Grid service containers
- DriverManager.md — ThreadLocal WebDriver management for parallel Grid sessions
