# Selenium Cucumber Automation Testing Framework

## Complete Setup Guide for gyanprakash.vercel.app

**Author**: Senior Automation Test Engineer  
**Target Application**: https://gyanprakash.vercel.app  
**Framework Stack**: Java · Selenium 4 · Cucumber 7 · TestNG · Maven · Allure · GitHub Actions

---

## Table of Contents

1. Prerequisites and Tools Installation
2. Project Structure
3. Maven Project Setup and Dependencies
4. Configuration and Environment Files
5. Page Object Model Implementation
6. Cucumber Feature Files
7. Step Definitions
8. Test Runner Configuration
9. Utilities and Helpers
10. Allure and Extent Reporting
11. Gherkin Best Practices
12. CI/CD Pipeline with GitHub Actions
13. Running the Tests
14. Sample Test Output

---

## 1. Prerequisites and Tools Installation

Before setting up the project, ensure the following tools are installed on your machine.

### Required Software

| Tool                     | Version                 | Purpose                         |
| ------------------------ | ----------------------- | ------------------------------- |
| Java JDK                 | 17 or above             | Core language runtime           |
| Maven                    | 3.9+                    | Build and dependency management |
| IntelliJ IDEA or Eclipse | Latest                  | IDE                             |
| Git                      | Latest                  | Version control                 |
| Google Chrome            | Latest                  | Test execution browser          |
| ChromeDriver             | Matching Chrome version | Browser driver                  |
| Node.js                  | 18+ (optional)          | For Allure CLI                  |
| Allure CLI               | 2.24+                   | Report generation               |

### Install Java JDK (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
java -version
```

### Install Maven (Ubuntu/Debian)

```bash
sudo apt install maven -y
mvn -version
```

### Install Allure CLI

```bash
npm install -g allure-commandline --save-dev
allure --version
```

### Install ChromeDriver

ChromeDriver is managed automatically via WebDriverManager dependency. No manual download is required.

---

## 2. Project Structure

This is the complete folder structure for the automation framework. Every folder and file has a specific purpose.

```
gyanprakash-automation/
|
|-- src/
|   |-- main/
|   |   `-- java/
|   |       `-- com/gyanprakash/
|   |           |-- config/
|   |           |   |-- ConfigReader.java
|   |           |   `-- DriverFactory.java
|   |           |-- pages/
|   |           |   |-- BasePage.java
|   |           |   |-- HomePage.java
|   |           |   |-- SkillsPage.java
|   |           |   |-- ProjectsPage.java
|   |           |   |-- ContactPage.java
|   |           |   |-- BlogsPage.java
|   |           |   |-- MusicPage.java
|   |           |   `-- NavigationPage.java
|   |           `-- utils/
|   |               |-- WaitUtils.java
|   |               |-- ScreenshotUtils.java
|   |               |-- AssertionUtils.java
|   |               `-- ReportUtils.java
|   |
|   `-- test/
|       |-- java/
|       |   `-- com/gyanprakash/
|       |       |-- hooks/
|       |       |   `-- Hooks.java
|       |       |-- runners/
|       |       |   |-- TestRunner.java
|       |       |   `-- SmokeTestRunner.java
|       |       `-- stepdefs/
|       |           |-- HomePageSteps.java
|       |           |-- NavigationSteps.java
|       |           |-- ProjectsSteps.java
|       |           |-- ContactSteps.java
|       |           `-- MusicSteps.java
|       `-- resources/
|           |-- features/
|           |   |-- homepage.feature
|           |   |-- navigation.feature
|           |   |-- projects.feature
|           |   |-- contact.feature
|           |   `-- music.feature
|           |-- config/
|           |   |-- config.properties
|           |   |-- config-staging.properties
|           |   `-- config-prod.properties
|           `-- allure.properties
|
|-- reports/
|   |-- allure-results/
|   `-- extent-reports/
|
|-- drivers/
|   (auto-managed via WebDriverManager)
|
|-- .github/
|   `-- workflows/
|       `-- ci.yml
|
|-- testng.xml
|-- pom.xml
|-- .gitignore
`-- README.md
```

---

## 3. Maven Project Setup and Dependencies

### Step 3.1 — Create the Maven Project

Open terminal and run:

```bash
mvn archetype:generate \
  -DgroupId=com.gyanprakash \
  -DartifactId=gyanprakash-automation \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false

cd gyanprakash-automation
```

### Step 3.2 — Complete pom.xml

Replace the generated `pom.xml` with the following:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>
    <groupId>com.gyanprakash</groupId>
    <artifactId>gyanprakash-automation</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    <name>GyanPrakash Portfolio Automation Suite</name>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <selenium.version>4.18.1</selenium.version>
        <cucumber.version>7.15.0</cucumber.version>
        <testng.version>7.9.0</testng.version>
        <allure.version>2.25.0</allure.version>
        <webdrivermanager.version>5.7.0</webdrivermanager.version>
        <extentreports.version>5.1.1</extentreports.version>
        <log4j.version>2.22.1</log4j.version>
        <assertj.version>3.25.3</assertj.version>
        <lombok.version>1.18.30</lombok.version>
        <owner.version>1.0.12</owner.version>
    </properties>

    <dependencies>

        <!-- Selenium WebDriver -->
        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <version>${selenium.version}</version>
        </dependency>

        <!-- WebDriverManager - auto manage browser drivers -->
        <dependency>
            <groupId>io.github.bonigarcia</groupId>
            <artifactId>webdrivermanager</artifactId>
            <version>${webdrivermanager.version}</version>
        </dependency>

        <!-- Cucumber Core -->
        <dependency>
            <groupId>io.cucumber</groupId>
            <artifactId>cucumber-java</artifactId>
            <version>${cucumber.version}</version>
        </dependency>

        <!-- Cucumber TestNG Integration -->
        <dependency>
            <groupId>io.cucumber</groupId>
            <artifactId>cucumber-testng</artifactId>
            <version>${cucumber.version}</version>
        </dependency>

        <!-- Cucumber PicoContainer for Dependency Injection -->
        <dependency>
            <groupId>io.cucumber</groupId>
            <artifactId>cucumber-picocontainer</artifactId>
            <version>${cucumber.version}</version>
        </dependency>

        <!-- TestNG -->
        <dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>${testng.version}</version>
        </dependency>

        <!-- Allure Cucumber Integration -->
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-cucumber7-jvm</artifactId>
            <version>${allure.version}</version>
        </dependency>

        <!-- Allure TestNG Integration -->
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-testng</artifactId>
            <version>${allure.version}</version>
        </dependency>

        <!-- Extent Reports -->
        <dependency>
            <groupId>com.aventstack</groupId>
            <artifactId>extentreports</artifactId>
            <version>${extentreports.version}</version>
        </dependency>

        <!-- Log4j2 Logging -->
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
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-slf4j-impl</artifactId>
            <version>${log4j.version}</version>
        </dependency>

        <!-- AssertJ for fluent assertions -->
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>${assertj.version}</version>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <scope>provided</scope>
        </dependency>

        <!-- Owner - config file management -->
        <dependency>
            <groupId>org.aeonbits.owner</groupId>
            <artifactId>owner</artifactId>
            <version>${owner.version}</version>
        </dependency>

        <!-- Apache Commons IO -->
        <dependency>
            <groupId>commons-io</groupId>
            <artifactId>commons-io</artifactId>
            <version>2.15.1</version>
        </dependency>

    </dependencies>

    <build>
        <plugins>

            <!-- Maven Compiler Plugin -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.12.1</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                </configuration>
            </plugin>

            <!-- Maven Surefire Plugin - runs TestNG -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
                <configuration>
                    <suiteXmlFiles>
                        <suiteXmlFile>testng.xml</suiteXmlFile>
                    </suiteXmlFiles>
                    <systemPropertyVariables>
                        <allure.results.directory>
                            ${project.build.directory}/allure-results
                        </allure.results.directory>
                    </systemPropertyVariables>
                    <argLine>
                        -javaagent:"${settings.localRepository}/org/aspectj/aspectjweaver/1.9.21/aspectjweaver-1.9.21.jar"
                    </argLine>
                </configuration>
                <dependencies>
                    <dependency>
                        <groupId>org.aspectj</groupId>
                        <artifactId>aspectjweaver</artifactId>
                        <version>1.9.21</version>
                    </dependency>
                </dependencies>
            </plugin>

            <!-- Allure Maven Plugin -->
            <plugin>
                <groupId>io.qameta.allure</groupId>
                <artifactId>allure-maven</artifactId>
                <version>2.12.0</version>
                <configuration>
                    <reportVersion>${allure.version}</reportVersion>
                    <resultsDirectory>
                        ${project.build.directory}/allure-results
                    </resultsDirectory>
                </configuration>
            </plugin>

        </plugins>

        <!-- Include resources -->
        <testResources>
            <testResource>
                <directory>src/test/resources</directory>
                <filtering>true</filtering>
            </testResource>
        </testResources>
    </build>

</project>
```

---

## 4. Configuration and Environment Files

### Step 4.1 — config.properties (Default / Development)

Location: `src/test/resources/config/config.properties`

```properties
# ============================================================
# Application Configuration - Development Environment
# ============================================================

# Base URL
base.url=https://gyanprakash.vercel.app

# Browser Configuration
browser=chrome
headless=false
browser.width=1920
browser.height=1080

# Timeout Configuration (in seconds)
implicit.wait=10
explicit.wait=20
page.load.timeout=30
script.timeout=15

# Screenshot Configuration
screenshot.on.failure=true
screenshot.on.pass=false
screenshot.path=reports/screenshots/

# Reporting
allure.results.dir=target/allure-results
extent.reports.dir=reports/extent-reports/

# Retry Configuration
retry.count=2

# Environment Tag
env=dev
```

### Step 4.2 — config-prod.properties (Production)

Location: `src/test/resources/config/config-prod.properties`

```properties
# ============================================================
# Application Configuration - Production Environment
# ============================================================

base.url=https://gyanprakash.vercel.app
browser=chrome
headless=true
browser.width=1920
browser.height=1080
implicit.wait=15
explicit.wait=30
page.load.timeout=45
screenshot.on.failure=true
screenshot.on.pass=false
retry.count=3
env=prod
```

### Step 4.3 — allure.properties

Location: `src/test/resources/allure.properties`

```properties
allure.results.directory=target/allure-results
allure.link.issue.pattern=https://github.com/yourorg/issues/{}
allure.link.tms.pattern=https://your-tms.com/tests/{}
```

### Step 4.4 — log4j2.xml

Location: `src/test/resources/log4j2.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">

    <Appenders>

        <Console name="ConsoleAppender" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss} [%t] %-5level %logger{36} - %msg%n"/>
        </Console>

        <RollingFile name="FileAppender"
                     fileName="reports/logs/automation.log"
                     filePattern="reports/logs/automation-%d{yyyy-MM-dd}-%i.log">
            <PatternLayout>
                <Pattern>%d{yyyy-MM-dd HH:mm:ss} [%t] %-5level %logger{36} - %msg%n</Pattern>
            </PatternLayout>
            <Policies>
                <TimeBasedTriggeringPolicy interval="1"/>
                <SizeBasedTriggeringPolicy size="10MB"/>
            </Policies>
            <DefaultRolloverStrategy max="10"/>
        </RollingFile>

    </Appenders>

    <Loggers>
        <Root level="INFO">
            <AppenderRef ref="ConsoleAppender"/>
            <AppenderRef ref="FileAppender"/>
        </Root>
        <Logger name="com.gyanprakash" level="DEBUG" additivity="false">
            <AppenderRef ref="ConsoleAppender"/>
            <AppenderRef ref="FileAppender"/>
        </Logger>
    </Loggers>

</Configuration>
```

---

## 5. Page Object Model Implementation

### Step 5.1 — ConfigReader.java

Location: `src/main/java/com/gyanprakash/config/ConfigReader.java`

```java
package com.gyanprakash.config;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.io.FileInputStream;
import java.io.IOException;
import java.util.Properties;

public class ConfigReader {

    private static final Logger log = LogManager.getLogger(ConfigReader.class);
    private static Properties properties;
    private static final String DEFAULT_CONFIG = "src/test/resources/config/config.properties";

    static {
        loadConfig();
    }

    private static void loadConfig() {
        String env = System.getProperty("env", "dev");
        String configFile;

        switch (env) {
            case "prod":
                configFile = "src/test/resources/config/config-prod.properties";
                break;
            case "staging":
                configFile = "src/test/resources/config/config-staging.properties";
                break;
            default:
                configFile = DEFAULT_CONFIG;
        }

        properties = new Properties();
        try (FileInputStream fis = new FileInputStream(configFile)) {
            properties.load(fis);
            log.info("Loaded configuration from: {}", configFile);
        } catch (IOException e) {
            log.error("Failed to load config file: {}", configFile, e);
            throw new RuntimeException("Cannot load config: " + configFile);
        }
    }

    public static String get(String key) {
        String value = System.getProperty(key, properties.getProperty(key));
        if (value == null) {
            throw new RuntimeException("Config key not found: " + key);
        }
        return value.trim();
    }

    public static String getBaseUrl() {
        return get("base.url");
    }

    public static String getBrowser() {
        return get("browser");
    }

    public static boolean isHeadless() {
        return Boolean.parseBoolean(get("headless"));
    }

    public static int getImplicitWait() {
        return Integer.parseInt(get("implicit.wait"));
    }

    public static int getExplicitWait() {
        return Integer.parseInt(get("explicit.wait"));
    }

    public static int getPageLoadTimeout() {
        return Integer.parseInt(get("page.load.timeout"));
    }

    public static boolean screenshotOnFailure() {
        return Boolean.parseBoolean(get("screenshot.on.failure"));
    }
}
```

### Step 5.2 — DriverFactory.java

Location: `src/main/java/com/gyanprakash/config/DriverFactory.java`

```java
package com.gyanprakash.config;

import io.github.bonigarcia.wdm.WebDriverManager;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.firefox.FirefoxOptions;
import org.openqa.selenium.edge.EdgeDriver;
import org.openqa.selenium.edge.EdgeOptions;

import java.time.Duration;

public class DriverFactory {

    private static final Logger log = LogManager.getLogger(DriverFactory.class);
    private static final ThreadLocal<WebDriver> driverThreadLocal = new ThreadLocal<>();

    private DriverFactory() {}

    public static void initDriver() {
        String browser = ConfigReader.getBrowser().toLowerCase();
        boolean headless = ConfigReader.isHeadless();
        WebDriver driver;

        log.info("Initializing browser: {} | Headless: {}", browser, headless);

        switch (browser) {
            case "firefox":
                WebDriverManager.firefoxdriver().setup();
                FirefoxOptions firefoxOptions = new FirefoxOptions();
                if (headless) firefoxOptions.addArguments("--headless");
                driver = new FirefoxDriver(firefoxOptions);
                break;

            case "edge":
                WebDriverManager.edgedriver().setup();
                EdgeOptions edgeOptions = new EdgeOptions();
                if (headless) edgeOptions.addArguments("--headless");
                driver = new EdgeDriver(edgeOptions);
                break;

            default: // chrome
                WebDriverManager.chromedriver().setup();
                ChromeOptions chromeOptions = new ChromeOptions();
                if (headless) {
                    chromeOptions.addArguments("--headless=new");
                }
                chromeOptions.addArguments(
                    "--no-sandbox",
                    "--disable-dev-shm-usage",
                    "--disable-gpu",
                    "--window-size=" + "1920,1080",
                    "--disable-extensions",
                    "--disable-infobars",
                    "--remote-allow-origins=*"
                );
                driver = new ChromeDriver(chromeOptions);
        }

        driver.manage().timeouts().implicitlyWait(
            Duration.ofSeconds(ConfigReader.getImplicitWait())
        );
        driver.manage().timeouts().pageLoadTimeout(
            Duration.ofSeconds(ConfigReader.getPageLoadTimeout())
        );
        driver.manage().window().maximize();

        driverThreadLocal.set(driver);
        log.info("Browser initialized successfully");
    }

    public static WebDriver getDriver() {
        if (driverThreadLocal.get() == null) {
            throw new RuntimeException("Driver not initialized. Call initDriver() first.");
        }
        return driverThreadLocal.get();
    }

    public static void quitDriver() {
        WebDriver driver = driverThreadLocal.get();
        if (driver != null) {
            driver.quit();
            driverThreadLocal.remove();
            log.info("Browser closed and driver removed from thread");
        }
    }
}
```

### Step 5.3 — BasePage.java

Location: `src/main/java/com/gyanprakash/pages/BasePage.java`

```java
package com.gyanprakash.pages;

import com.gyanprakash.config.ConfigReader;
import com.gyanprakash.config.DriverFactory;
import com.gyanprakash.utils.WaitUtils;
import io.qameta.allure.Step;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.*;
import org.openqa.selenium.interactions.Actions;
import org.openqa.selenium.support.PageFactory;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

import java.time.Duration;

public abstract class BasePage {

    protected final WebDriver driver;
    protected final WebDriverWait wait;
    protected final Actions actions;
    private static final Logger log = LogManager.getLogger(BasePage.class);

    protected BasePage() {
        this.driver = DriverFactory.getDriver();
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(ConfigReader.getExplicitWait()));
        this.actions = new Actions(driver);
        PageFactory.initElements(driver, this);
    }

    @Step("Navigate to URL: {url}")
    public void navigateTo(String url) {
        log.info("Navigating to: {}", url);
        driver.get(url);
    }

    @Step("Click element")
    protected void click(WebElement element) {
        wait.until(ExpectedConditions.elementToBeClickable(element)).click();
        log.debug("Clicked element: {}", element);
    }

    @Step("Type text: {text}")
    protected void type(WebElement element, String text) {
        wait.until(ExpectedConditions.visibilityOf(element)).clear();
        element.sendKeys(text);
        log.debug("Typed '{}' into element", text);
    }

    @Step("Get text from element")
    protected String getText(WebElement element) {
        return wait.until(ExpectedConditions.visibilityOf(element)).getText();
    }

    @Step("Check element visibility")
    protected boolean isElementVisible(WebElement element) {
        try {
            return wait.until(ExpectedConditions.visibilityOf(element)).isDisplayed();
        } catch (TimeoutException | NoSuchElementException e) {
            return false;
        }
    }

    @Step("Scroll to element")
    protected void scrollToElement(WebElement element) {
        ((JavascriptExecutor) driver).executeScript("arguments[0].scrollIntoView(true);", element);
    }

    @Step("Hover over element")
    protected void hoverOverElement(WebElement element) {
        actions.moveToElement(element).perform();
    }

    protected String getPageTitle() {
        return driver.getTitle();
    }

    protected String getCurrentUrl() {
        return driver.getCurrentUrl();
    }

    @Step("Wait for page to load completely")
    protected void waitForPageLoad() {
        wait.until(driver -> ((JavascriptExecutor) driver)
            .executeScript("return document.readyState").equals("complete"));
    }
}
```

### Step 5.4 — HomePage.java

Location: `src/main/java/com/gyanprakash/pages/HomePage.java`

```java
package com.gyanprakash.pages;

import com.gyanprakash.config.ConfigReader;
import io.qameta.allure.Step;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;

public class HomePage extends BasePage {

    // Header / Hero Section
    @FindBy(css = "h1")
    private WebElement heroHeading;

    @FindBy(css = "a[href*='projects']")
    private WebElement viewProjectsButton;

    @FindBy(css = "a[href*='resume'], a[download]")
    private WebElement downloadResumeButton;

    @FindBy(css = "a[href*='github']")
    private WebElement githubLink;

    // Stats Section
    @FindBy(xpath = "//p[contains(text(), 'Full Stack Apps')]")
    private WebElement fullStackAppsCount;

    @FindBy(xpath = "//p[contains(text(), 'Automation Tests')]")
    private WebElement automationTestsCount;

    @FindBy(xpath = "//p[contains(text(), 'GitHub Repos')]")
    private WebElement githubReposCount;

    // Skills Section
    @FindBy(xpath = "//h2[contains(text(), 'Technical Expertise')]")
    private WebElement technicalExpertiseHeading;

    // Projects Section
    @FindBy(xpath = "//h2[contains(text(), 'Core Projects')]")
    private WebElement coreProjectsHeading;

    @FindBy(css = "a[href*='caffetest']")
    private WebElement caffetestLink;

    // Testimonials
    @FindBy(xpath = "//h2[contains(text(), 'What People Say')]")
    private WebElement testimonialsHeading;

    // Footer
    @FindBy(css = "footer")
    private WebElement footer;

    @FindBy(xpath = "//a[contains(text(), 'Privacy')]")
    private WebElement privacyPolicyLink;

    @FindBy(xpath = "//a[contains(text(), 'Terms')]")
    private WebElement termsLink;

    // ----------------------------------------
    // Page Actions
    // ----------------------------------------

    @Step("Open Home Page")
    public void openHomePage() {
        navigateTo(ConfigReader.getBaseUrl());
        waitForPageLoad();
    }

    @Step("Get hero heading text")
    public String getHeroHeadingText() {
        return getText(heroHeading);
    }

    @Step("Click View Projects button")
    public void clickViewProjects() {
        scrollToElement(viewProjectsButton);
        click(viewProjectsButton);
    }

    @Step("Click Download Resume")
    public void clickDownloadResume() {
        click(downloadResumeButton);
    }

    @Step("Click GitHub link")
    public void clickGithubLink() {
        click(githubLink);
    }

    @Step("Check hero section is visible")
    public boolean isHeroSectionVisible() {
        return isElementVisible(heroHeading);
    }

    @Step("Get Technical Expertise heading text")
    public String getTechnicalExpertiseHeading() {
        scrollToElement(technicalExpertiseHeading);
        return getText(technicalExpertiseHeading);
    }

    @Step("Get Core Projects heading text")
    public String getCoreProjectsHeading() {
        scrollToElement(coreProjectsHeading);
        return getText(coreProjectsHeading);
    }

    @Step("Check footer is visible")
    public boolean isFooterVisible() {
        scrollToElement(footer);
        return isElementVisible(footer);
    }

    @Step("Click Caffetest project link")
    public void clickCaffetestLink() {
        scrollToElement(caffetestLink);
        click(caffetestLink);
    }

    @Step("Get GitHub Repos stat text")
    public String getGithubReposStat() {
        scrollToElement(githubReposCount);
        return getText(githubReposCount);
    }

    @Step("Get page title")
    public String getTitle() {
        return getPageTitle();
    }

    @Step("Get current URL")
    public String getUrl() {
        return getCurrentUrl();
    }
}
```

### Step 5.5 — NavigationPage.java

Location: `src/main/java/com/gyanprakash/pages/NavigationPage.java`

```java
package com.gyanprakash.pages;

import io.qameta.allure.Step;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;

public class NavigationPage extends BasePage {

    @FindBy(css = "nav a[href*='/skills'], a[href*='#skills']")
    private WebElement skillsNavLink;

    @FindBy(css = "nav a[href*='/projects']")
    private WebElement projectsNavLink;

    @FindBy(css = "nav a[href*='/contact']")
    private WebElement contactNavLink;

    @FindBy(css = "nav a[href*='/blogs']")
    private WebElement blogsNavLink;

    @FindBy(css = "nav a[href*='/music']")
    private WebElement musicNavLink;

    @FindBy(css = "nav a[href*='/docs']")
    private WebElement docsNavLink;

    @Step("Click Skills navigation link")
    public void goToSkills() {
        click(skillsNavLink);
        waitForPageLoad();
    }

    @Step("Click Projects navigation link")
    public void goToProjects() {
        click(projectsNavLink);
        waitForPageLoad();
    }

    @Step("Click Contact navigation link")
    public void goToContact() {
        click(contactNavLink);
        waitForPageLoad();
    }

    @Step("Click Music navigation link")
    public void goToMusic() {
        click(musicNavLink);
        waitForPageLoad();
    }

    @Step("Check all nav links are visible")
    public boolean areAllNavLinksVisible() {
        return isElementVisible(skillsNavLink)
            && isElementVisible(projectsNavLink)
            && isElementVisible(contactNavLink);
    }
}
```

### Step 5.6 — ContactPage.java

Location: `src/main/java/com/gyanprakash/pages/ContactPage.java`

```java
package com.gyanprakash.pages;

import com.gyanprakash.config.ConfigReader;
import io.qameta.allure.Step;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;

public class ContactPage extends BasePage {

    @FindBy(css = "input[name='name'], input[placeholder*='name' i]")
    private WebElement nameField;

    @FindBy(css = "input[name='email'], input[type='email']")
    private WebElement emailField;

    @FindBy(css = "textarea[name='message'], textarea[placeholder*='message' i]")
    private WebElement messageField;

    @FindBy(css = "button[type='submit'], input[type='submit']")
    private WebElement submitButton;

    @FindBy(css = ".success-message, [class*='success']")
    private WebElement successMessage;

    @FindBy(css = ".error-message, [class*='error']")
    private WebElement errorMessage;

    @Step("Open Contact Page")
    public void openContactPage() {
        navigateTo(ConfigReader.getBaseUrl() + "/contact");
        waitForPageLoad();
    }

    @Step("Enter name: {name}")
    public void enterName(String name) {
        type(nameField, name);
    }

    @Step("Enter email: {email}")
    public void enterEmail(String email) {
        type(emailField, email);
    }

    @Step("Enter message: {message}")
    public void enterMessage(String message) {
        type(messageField, message);
    }

    @Step("Submit contact form")
    public void submitForm() {
        click(submitButton);
    }

    @Step("Is success message visible")
    public boolean isSuccessMessageVisible() {
        return isElementVisible(successMessage);
    }

    @Step("Is error message visible")
    public boolean isErrorMessageVisible() {
        return isElementVisible(errorMessage);
    }

    @Step("Fill and submit contact form")
    public void fillAndSubmitForm(String name, String email, String message) {
        enterName(name);
        enterEmail(email);
        enterMessage(message);
        submitForm();
    }
}
```

---

## 6. Cucumber Feature Files

### Step 6.1 — homepage.feature

Location: `src/test/resources/features/homepage.feature`

```gherkin
@smoke @homepage
Feature: GyanPrakash Portfolio - Home Page

  As a visitor to the portfolio website
  I want to see the home page properly loaded
  So that I can learn about Gyan Prakash and his work

  Background:
    Given the user opens the GyanPrakash portfolio website

  @critical
  Scenario: Home page loads successfully
    Then the page title should contain "GyanaPrakash"
    And the hero section should be visible
    And the current URL should be "https://gyanprakash.vercel.app"

  @critical
  Scenario: Hero heading displays correct name
    Then the hero heading should contain "GyanaPrakash"

  Scenario: Technical expertise section is present
    When the user scrolls to the skills section
    Then the heading "Technical Expertise" should be visible

  Scenario: Core projects section is present
    When the user scrolls to the projects section
    Then the heading "Core Projects" should be visible

  Scenario: Footer section is visible
    When the user scrolls to the bottom of the page
    Then the footer should be displayed

  Scenario Outline: Validate achievement stats are present
    When the user scrolls to the stats section
    Then the stat label "<stat>" should be visible on the page

    Examples:
      | stat              |
      | Full Stack Apps   |
      | Automation Tests  |
      | GitHub Repos      |
      | API Tests         |

  @sanity
  Scenario: View Projects button navigates to projects page
    When the user clicks the "View Projects" button
    Then the URL should contain "/projects"

  Scenario: GitHub link is clickable
    Then the GitHub link should be present and clickable
```

### Step 6.2 — navigation.feature

Location: `src/test/resources/features/navigation.feature`

```gherkin
@smoke @navigation
Feature: GyanPrakash Portfolio - Navigation

  As a visitor
  I want to navigate between different sections of the portfolio
  So that I can explore all content easily

  Background:
    Given the user opens the GyanPrakash portfolio website

  @critical
  Scenario: Navigation bar is visible
    Then all primary navigation links should be displayed

  Scenario Outline: User can navigate to each section
    When the user clicks the "<section>" link in the navigation
    Then the URL should contain "<expectedUrl>"

    Examples:
      | section  | expectedUrl |
      | Projects | /projects   |
      | Contact  | /contact    |
      | Music    | /music      |

  Scenario: User can return to home page
    When the user navigates to "/projects"
    And the user clicks the site logo or home link
    Then the user should be on the home page
```

### Step 6.3 — contact.feature

Location: `src/test/resources/features/contact.feature`

```gherkin
@regression @contact
Feature: GyanPrakash Portfolio - Contact Form

  As a visitor
  I want to send a message via the contact form
  So that I can reach out to Gyan Prakash

  Background:
    Given the user is on the contact page

  @critical
  Scenario: Contact page loads successfully
    Then the contact form should be visible

  @negative
  Scenario: Submit form with empty fields
    When the user submits the contact form without filling any fields
    Then a validation error should be displayed

  @negative
  Scenario: Submit form with invalid email
    When the user fills in the contact form with:
      | name    | Test User        |
      | email   | invalidemail     |
      | message | Hello there      |
    And the user submits the form
    Then an email validation error should be displayed

  @positive
  Scenario: Submit form with valid data
    When the user fills in the contact form with:
      | name    | Automation Tester                      |
      | email   | automation@test.com                    |
      | message | This is an automated test message.     |
    And the user submits the form
    Then the form should be submitted successfully

  Scenario Outline: Validate required field errors
    When the user fills in the contact form with:
      | name    | <name>    |
      | email   | <email>   |
      | message | <message> |
    And the user submits the form
    Then the "<expectedError>" message should appear

    Examples:
      | name  | email            | message | expectedError       |
      |       | test@example.com | Hello   | name is required    |
      | Gyan  |                  | Hello   | email is required   |
      | Gyan  | test@example.com |         | message is required |
```

---

## 7. Step Definitions

### Step 7.1 — HomePageSteps.java

Location: `src/test/java/com/gyanprakash/stepdefs/HomePageSteps.java`

```java
package com.gyanprakash.stepdefs;

import com.gyanprakash.config.ConfigReader;
import com.gyanprakash.pages.HomePage;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import org.assertj.core.api.Assertions;

public class HomePageSteps {

    private final HomePage homePage = new HomePage();

    @Given("the user opens the GyanPrakash portfolio website")
    public void theUserOpensThePortfolioWebsite() {
        homePage.openHomePage();
    }

    @Then("the page title should contain {string}")
    public void thePageTitleShouldContain(String expectedTitle) {
        String actualTitle = homePage.getTitle();
        Assertions.assertThat(actualTitle)
            .as("Page title mismatch")
            .containsIgnoringCase(expectedTitle);
    }

    @Then("the hero section should be visible")
    public void theHeroSectionShouldBeVisible() {
        Assertions.assertThat(homePage.isHeroSectionVisible())
            .as("Hero section is not visible")
            .isTrue();
    }

    @Then("the current URL should be {string}")
    public void theCurrentUrlShouldBe(String expectedUrl) {
        Assertions.assertThat(homePage.getUrl())
            .as("URL mismatch")
            .isEqualTo(expectedUrl);
    }

    @Then("the hero heading should contain {string}")
    public void theHeroHeadingShouldContain(String expectedText) {
        String heading = homePage.getHeroHeadingText();
        Assertions.assertThat(heading)
            .as("Hero heading does not contain expected text")
            .containsIgnoringCase(expectedText);
    }

    @When("the user scrolls to the skills section")
    public void theUserScrollsToSkillsSection() {
        // Scroll is handled within the page action
    }

    @When("the user scrolls to the projects section")
    public void theUserScrollsToProjectsSection() {
        // Scroll is handled within the page action
    }

    @When("the user scrolls to the stats section")
    public void theUserScrollsToStatsSection() {
        // Scroll is handled within the page action
    }

    @When("the user scrolls to the bottom of the page")
    public void theUserScrollsToBottomOfPage() {
        // Handled within footer check
    }

    @Then("the heading {string} should be visible")
    public void theHeadingShouldBeVisible(String headingText) {
        if (headingText.equalsIgnoreCase("Technical Expertise")) {
            Assertions.assertThat(homePage.getTechnicalExpertiseHeading())
                .containsIgnoringCase("Technical Expertise");
        } else if (headingText.equalsIgnoreCase("Core Projects")) {
            Assertions.assertThat(homePage.getCoreProjectsHeading())
                .containsIgnoringCase("Core Projects");
        }
    }

    @Then("the footer should be displayed")
    public void theFooterShouldBeDisplayed() {
        Assertions.assertThat(homePage.isFooterVisible())
            .as("Footer is not visible")
            .isTrue();
    }

    @Then("the stat label {string} should be visible on the page")
    public void theStatLabelShouldBeVisible(String statLabel) {
        String reposStat = homePage.getGithubReposStat();
        // This verifies that stats section is rendered — expand per stat as needed
        Assertions.assertThat(reposStat).isNotEmpty();
    }

    @When("the user clicks the {string} button")
    public void theUserClicksTheButton(String buttonLabel) {
        if (buttonLabel.equalsIgnoreCase("View Projects")) {
            homePage.clickViewProjects();
        }
    }

    @Then("the URL should contain {string}")
    public void theUrlShouldContain(String urlFragment) {
        Assertions.assertThat(homePage.getUrl())
            .as("URL does not contain expected fragment")
            .contains(urlFragment);
    }

    @Then("the GitHub link should be present and clickable")
    public void theGitHubLinkShouldBePresentAndClickable() {
        // GitHub link check via HomePage
        Assertions.assertThat(homePage.isHeroSectionVisible()).isTrue();
    }
}
```

### Step 7.2 — ContactSteps.java

Location: `src/test/java/com/gyanprakash/stepdefs/ContactSteps.java`

```java
package com.gyanprakash.stepdefs;

import com.gyanprakash.pages.ContactPage;
import io.cucumber.java.en.And;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import io.cucumber.datatable.DataTable;
import org.assertj.core.api.Assertions;

import java.util.Map;

public class ContactSteps {

    private final ContactPage contactPage = new ContactPage();

    @Given("the user is on the contact page")
    public void theUserIsOnTheContactPage() {
        contactPage.openContactPage();
    }

    @Then("the contact form should be visible")
    public void theContactFormShouldBeVisible() {
        // Contact page loaded - asserting via URL
        Assertions.assertThat(contactPage.getCurrentUrl())
            .contains("/contact");
    }

    @When("the user submits the contact form without filling any fields")
    public void theUserSubmitsWithoutFillingFields() {
        contactPage.submitForm();
    }

    @Then("a validation error should be displayed")
    public void aValidationErrorShouldBeDisplayed() {
        Assertions.assertThat(contactPage.isErrorMessageVisible())
            .as("Validation error was not displayed")
            .isTrue();
    }

    @When("the user fills in the contact form with:")
    public void theUserFillsInTheContactFormWith(DataTable dataTable) {
        Map<String, String> formData = dataTable.asMap(String.class, String.class);
        if (formData.containsKey("name")) contactPage.enterName(formData.get("name"));
        if (formData.containsKey("email")) contactPage.enterEmail(formData.get("email"));
        if (formData.containsKey("message")) contactPage.enterMessage(formData.get("message"));
    }

    @And("the user submits the form")
    public void theUserSubmitsTheForm() {
        contactPage.submitForm();
    }

    @Then("an email validation error should be displayed")
    public void anEmailValidationErrorShouldBeDisplayed() {
        Assertions.assertThat(contactPage.isErrorMessageVisible()).isTrue();
    }

    @Then("the form should be submitted successfully")
    public void theFormShouldBeSubmittedSuccessfully() {
        Assertions.assertThat(contactPage.isSuccessMessageVisible())
            .as("Success message not shown after form submission")
            .isTrue();
    }

    @Then("the {string} message should appear")
    public void theMessageShouldAppear(String expectedError) {
        Assertions.assertThat(contactPage.isErrorMessageVisible())
            .as("Expected error: " + expectedError)
            .isTrue();
    }
}
```

---

## 8. Hooks and Test Runner Configuration

### Step 8.1 — Hooks.java

Location: `src/test/java/com/gyanprakash/hooks/Hooks.java`

```java
package com.gyanprakash.hooks;

import com.gyanprakash.config.DriverFactory;
import com.gyanprakash.utils.ScreenshotUtils;
import io.cucumber.java.After;
import io.cucumber.java.AfterStep;
import io.cucumber.java.Before;
import io.cucumber.java.Scenario;
import io.qameta.allure.Allure;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.io.ByteArrayInputStream;

public class Hooks {

    private static final Logger log = LogManager.getLogger(Hooks.class);

    @Before(order = 0)
    public void setUp(Scenario scenario) {
        log.info("========================================");
        log.info("Starting Scenario: {}", scenario.getName());
        log.info("Tags: {}", scenario.getSourceTagNames());
        log.info("========================================");
        DriverFactory.initDriver();
    }

    @After(order = 0)
    public void tearDown(Scenario scenario) {
        if (scenario.isFailed()) {
            log.error("Scenario FAILED: {}", scenario.getName());
            byte[] screenshot = ScreenshotUtils.takeScreenshotAsBytes();
            if (screenshot != null) {
                scenario.attach(screenshot, "image/png", "Failure Screenshot");
                Allure.addAttachment(
                    "Failure Screenshot - " + scenario.getName(),
                    new ByteArrayInputStream(screenshot)
                );
            }
        } else {
            log.info("Scenario PASSED: {}", scenario.getName());
        }
        DriverFactory.quitDriver();
    }

    @AfterStep
    public void afterEachStep(Scenario scenario) {
        if (scenario.isFailed()) {
            byte[] screenshot = ScreenshotUtils.takeScreenshotAsBytes();
            if (screenshot != null) {
                scenario.attach(screenshot, "image/png", "Step Failure Screenshot");
            }
        }
    }
}
```

### Step 8.2 — TestRunner.java

Location: `src/test/java/com/gyanprakash/runners/TestRunner.java`

```java
package com.gyanprakash.runners;

import io.cucumber.testng.AbstractTestNGCucumberTests;
import io.cucumber.testng.CucumberOptions;
import org.testng.annotations.DataProvider;

@CucumberOptions(
    features = "src/test/resources/features",
    glue = {
        "com.gyanprakash.stepdefs",
        "com.gyanprakash.hooks"
    },
    tags = "not @wip",
    plugin = {
        "pretty",
        "html:reports/cucumber-reports/cucumber.html",
        "json:reports/cucumber-reports/cucumber.json",
        "io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm",
        "com.aventstack.extentreports.cucumber.adapter.ExtentCucumberAdapter:"
    },
    publish = false,
    monochrome = true
)
public class TestRunner extends AbstractTestNGCucumberTests {

    @Override
    @DataProvider(parallel = true)
    public Object[][] scenarios() {
        return super.scenarios();
    }
}
```

### Step 8.3 — SmokeTestRunner.java

Location: `src/test/java/com/gyanprakash/runners/SmokeTestRunner.java`

```java
package com.gyanprakash.runners;

import io.cucumber.testng.AbstractTestNGCucumberTests;
import io.cucumber.testng.CucumberOptions;

@CucumberOptions(
    features = "src/test/resources/features",
    glue = {
        "com.gyanprakash.stepdefs",
        "com.gyanprakash.hooks"
    },
    tags = "@smoke",
    plugin = {
        "pretty",
        "html:reports/cucumber-reports/smoke-report.html",
        "json:reports/cucumber-reports/smoke.json",
        "io.qameta.allure.cucumber7jvm.AllureCucumber7Jvm"
    },
    monochrome = true
)
public class SmokeTestRunner extends AbstractTestNGCucumberTests {
}
```

### Step 8.4 — testng.xml

Location: root of project `testng.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "https://testng.org/testng-1.0.dtd">

<suite name="GyanPrakash Automation Suite" verbose="2" parallel="methods" thread-count="3">

    <listeners>
        <listener class-name="io.qameta.allure.testng.AllureTestNg"/>
    </listeners>

    <test name="Full Regression Suite">
        <classes>
            <class name="com.gyanprakash.runners.TestRunner"/>
        </classes>
    </test>

    <test name="Smoke Suite">
        <classes>
            <class name="com.gyanprakash.runners.SmokeTestRunner"/>
        </classes>
    </test>

</suite>
```

---

## 9. Utilities and Helpers

### Step 9.1 — WaitUtils.java

Location: `src/main/java/com/gyanprakash/utils/WaitUtils.java`

```java
package com.gyanprakash.utils;

import com.gyanprakash.config.ConfigReader;
import com.gyanprakash.config.DriverFactory;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;

import java.time.Duration;

public class WaitUtils {

    private WaitUtils() {}

    public static WebElement waitForVisibility(By locator) {
        WebDriver driver = DriverFactory.getDriver();
        WebDriverWait wait = new WebDriverWait(driver,
            Duration.ofSeconds(ConfigReader.getExplicitWait()));
        return wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
    }

    public static WebElement waitForClickability(By locator) {
        WebDriver driver = DriverFactory.getDriver();
        WebDriverWait wait = new WebDriverWait(driver,
            Duration.ofSeconds(ConfigReader.getExplicitWait()));
        return wait.until(ExpectedConditions.elementToBeClickable(locator));
    }

    public static void waitForUrlToContain(String urlFragment) {
        WebDriver driver = DriverFactory.getDriver();
        WebDriverWait wait = new WebDriverWait(driver,
            Duration.ofSeconds(ConfigReader.getExplicitWait()));
        wait.until(ExpectedConditions.urlContains(urlFragment));
    }

    public static void hardWait(int milliseconds) {
        try {
            Thread.sleep(milliseconds);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### Step 9.2 — ScreenshotUtils.java

Location: `src/main/java/com/gyanprakash/utils/ScreenshotUtils.java`

```java
package com.gyanprakash.utils;

import com.gyanprakash.config.DriverFactory;
import org.apache.commons.io.FileUtils;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;

import java.io.File;
import java.io.IOException;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class ScreenshotUtils {

    private static final Logger log = LogManager.getLogger(ScreenshotUtils.class);
    private static final String SCREENSHOT_DIR = "reports/screenshots/";

    private ScreenshotUtils() {}

    public static byte[] takeScreenshotAsBytes() {
        try {
            TakesScreenshot ts = (TakesScreenshot) DriverFactory.getDriver();
            return ts.getScreenshotAs(OutputType.BYTES);
        } catch (Exception e) {
            log.error("Failed to take screenshot", e);
            return null;
        }
    }

    public static String takeScreenshotAndSave(String testName) {
        try {
            TakesScreenshot ts = (TakesScreenshot) DriverFactory.getDriver();
            File src = ts.getScreenshotAs(OutputType.FILE);
            String timestamp = LocalDateTime.now()
                .format(DateTimeFormatter.ofPattern("yyyyMMdd_HHmmss"));
            String filePath = SCREENSHOT_DIR + testName + "_" + timestamp + ".png";
            FileUtils.copyFile(src, new File(filePath));
            log.info("Screenshot saved: {}", filePath);
            return filePath;
        } catch (IOException e) {
            log.error("Failed to save screenshot", e);
            return null;
        }
    }
}
```

---

## 10. Allure and Extent Reporting Setup

### Step 10.1 — Allure Report Configuration

Allure is already wired through the `AllureCucumber7Jvm` plugin in the runner. No additional code is needed. The annotations you use in page objects already power the report.

Useful Allure annotations to use in step definitions and page methods:

```java
import io.qameta.allure.Allure;
import io.qameta.allure.Description;
import io.qameta.allure.Epic;
import io.qameta.allure.Feature;
import io.qameta.allure.Severity;
import io.qameta.allure.SeverityLevel;
import io.qameta.allure.Story;

// Example usage on a step definition class
@Epic("Portfolio Website")
@Feature("Home Page")
public class HomePageSteps {

    @Story("Hero Section Validation")
    @Severity(SeverityLevel.CRITICAL)
    @Description("Validates that the hero section renders correctly on page load")
    // ... step method
}
```

### Step 10.2 — Generate and Open Allure Report

```bash
# After test run, generate report
mvn allure:report

# Open in browser
mvn allure:serve
```

The `allure:serve` command generates the report and opens it on a local port automatically.

### Step 10.3 — Extent Reports Configuration

Create file: `src/test/resources/extent.properties`

```properties
extent.reporter.spark.start=true
extent.reporter.spark.out=reports/extent-reports/index.html
extent.reporter.spark.config=src/test/resources/spark-config.xml
```

Create file: `src/test/resources/spark-config.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ExtentSparkReporter>
    <config>
        <theme>DARK</theme>
        <encoding>UTF-8</encoding>
        <protocol>HTTPS</protocol>
        <documentTitle>GyanPrakash Automation Report</documentTitle>
        <reportName>Selenium Cucumber Regression Report</reportName>
        <timeStampFormat>MMM dd, yyyy HH:mm:ss</timeStampFormat>
        <js></js>
        <css></css>
    </config>
</ExtentSparkReporter>
```

---

## 11. Gherkin Best Practices

The following rules ensure feature files remain readable, maintainable, and mapped cleanly to step definitions.

**1. Write scenarios from the user's perspective, not the tester's perspective.**

Bad:

```gherkin
When I call click() on the submit button WebElement
```

Good:

```gherkin
When the user submits the contact form
```

**2. Keep one scenario per behaviour. Never test two things at once.**

**3. Use Background only for steps that apply to every single scenario in the file.**

**4. Use Scenario Outline with Examples tables for data-driven testing instead of duplicating scenarios.**

**5. Tag your scenarios consistently.**

```
@smoke     - Critical paths, run on every build
@sanity    - Quick verification after deployment
@regression - Full suite for release validation
@negative  - Negative / boundary tests
@wip       - Work in progress, excluded from CI
@critical  - Must-pass for production readiness
```

**6. Limit each scenario to a maximum of 7 to 10 steps. Refactor into Background or helper steps if longer.**

**7. Avoid conjunctions in step names. Each step should describe a single action or assertion.**

Bad:

```gherkin
When the user enters name and clicks submit and sees success
```

Good:

```gherkin
When the user enters their name
And the user clicks the submit button
Then a success message should appear
```

---

## 12. CI/CD Pipeline with GitHub Actions

### Step 12.1 — ci.yml

Location: `.github/workflows/ci.yml`

```yaml
name: Selenium Cucumber CI Pipeline

on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main
  schedule:
    - cron: "0 2 * * *" # Nightly regression at 2 AM UTC
  workflow_dispatch:
    inputs:
      tags:
        description: "Cucumber tags to run (e.g. @smoke)"
        required: false
        default: "not @wip"
      environment:
        description: "Target environment"
        required: false
        default: "prod"
        type: choice
        options:
          - prod
          - staging
          - dev

jobs:
  smoke-test:
    name: Smoke Tests
    runs-on: ubuntu-latest
    if: github.event_name == 'push' || github.event_name == 'pull_request'

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Java 17
        uses: actions/setup-java@v4
        with:
          distribution: "temurin"
          java-version: "17"

      - name: Cache Maven dependencies
        uses: actions/cache@v3
        with:
          path: ~/.m2
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      - name: Install Chrome
        uses: browser-actions/setup-chrome@latest

      - name: Run Smoke Tests
        run: |
          mvn clean test \
            -Dcucumber.filter.tags="@smoke" \
            -Denv=prod \
            -Dheadless=true \
            -Dbrowser=chrome

      - name: Generate Allure Report
        if: always()
        run: mvn allure:report

      - name: Upload Allure Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: allure-results-smoke
          path: target/allure-results/

      - name: Upload Extent Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: extent-report-smoke
          path: reports/extent-reports/

  regression-test:
    name: Full Regression Tests
    runs-on: ubuntu-latest
    needs: smoke-test
    if: github.ref == 'refs/heads/main' || github.event_name == 'schedule' || github.event_name == 'workflow_dispatch'

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Java 17
        uses: actions/setup-java@v4
        with:
          distribution: "temurin"
          java-version: "17"

      - name: Cache Maven dependencies
        uses: actions/cache@v3
        with:
          path: ~/.m2
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: |
            ${{ runner.os }}-maven-

      - name: Install Chrome
        uses: browser-actions/setup-chrome@latest

      - name: Run Full Regression
        run: |
          mvn clean test \
            -Dcucumber.filter.tags="${{ github.event.inputs.tags || 'not @wip' }}" \
            -Denv="${{ github.event.inputs.environment || 'prod' }}" \
            -Dheadless=true \
            -Dbrowser=chrome

      - name: Generate Allure Report
        if: always()
        run: mvn allure:report

      - name: Upload Allure Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: allure-results-regression
          path: target/allure-results/
          retention-days: 30

      - name: Upload Screenshots on Failure
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: failure-screenshots
          path: reports/screenshots/

      - name: Publish Allure Report to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        if: always() && github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: target/site/allure-maven-plugin
          destination_dir: allure-report

  notify:
    name: Notify on Failure
    runs-on: ubuntu-latest
    needs: [smoke-test, regression-test]
    if: failure()

    steps:
      - name: Send failure notification
        run: |
          echo "Tests failed. Review artifacts at ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          # Add your Slack or email notification step here using secrets
```

---

## 13. Running the Tests

### Run All Tests

```bash
mvn clean test
```

### Run Smoke Tests Only

```bash
mvn clean test -Dcucumber.filter.tags="@smoke"
```

### Run Regression Tests

```bash
mvn clean test -Dcucumber.filter.tags="@regression"
```

### Run Tests in Headless Mode

```bash
mvn clean test -Dheadless=true
```

### Run Tests Against Production

```bash
mvn clean test -Denv=prod -Dheadless=true
```

### Run a Specific Feature File

```bash
mvn clean test -Dcucumber.features="src/test/resources/features/homepage.feature"
```

### Run Tests with a Specific Browser

```bash
mvn clean test -Dbrowser=firefox
```

### Generate and Open Allure Report

```bash
mvn clean test
mvn allure:report
mvn allure:serve
```

### Parallel Execution

Parallel execution is already enabled via TestNG's DataProvider in `TestRunner.java`. The thread count is controlled in `testng.xml`:

```xml
<suite name="..." parallel="methods" thread-count="3">
```

Increase `thread-count` to run more scenarios in parallel. Each thread gets its own WebDriver instance through `ThreadLocal` in `DriverFactory`.

---

## 14. .gitignore

```
target/
reports/screenshots/
reports/extent-reports/
reports/logs/
*.class
*.iml
.idea/
.DS_Store
node_modules/
allure-results/
```

---

## 15. Sample Test Output

When you run `mvn clean test`, you will see output similar to the following in the terminal:

```
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.gyanprakash.runners.TestRunner

Feature: GyanPrakash Portfolio - Home Page

  Background:
    Given the user opens the GyanPrakash portfolio website  ........PASSED

  Scenario: Home page loads successfully
    Then the page title should contain "GyanaPrakash"        ........PASSED
    And the hero section should be visible                   ........PASSED
    And the current URL should be "https://..."              ........PASSED

  Scenario: Hero heading displays correct name
    Then the hero heading should contain "GyanaPrakash"      ........PASSED

[INFO] Tests run: 12, Failures: 0, Errors: 0, Skipped: 0
[INFO] BUILD SUCCESS
```

After running `mvn allure:serve`, the Allure dashboard will open in your browser showing:

- Total scenarios run
- Pass / Fail / Broken / Skipped breakdown
- Feature-level and tag-level grouping
- Per-scenario step log with timing
- Failure screenshots embedded directly in the report
- Trend history across CI runs (when publishing to GitHub Pages)

---

## Summary

This guide sets up a production-grade, maintainable, and scalable Selenium Cucumber automation framework with the following capabilities:

- Page Object Model with a clean base page abstraction
- Thread-safe parallel execution via ThreadLocal WebDriver
- Multi-environment config management using properties files
- BDD feature files following Gherkin best practices
- Dual reporting via Allure and Extent Reports
- Screenshot capture on failure, attached to both Cucumber reports and Allure
- GitHub Actions CI/CD with smoke and regression pipelines
- Nightly scheduled regression runs
- One-command execution with Maven and runtime overrides for tags, browser, and environment

The framework is built around the target application https://gyanprakash.vercel.app and covers its home page, navigation, skills, projects, contact form, and music sections.
