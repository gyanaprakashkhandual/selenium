# Environment Setup

## Overview

A correctly configured environment is the foundation of a reliable automation framework. This guide walks through setting up a complete Selenium + Cucumber + Page Object Model project from scratch — covering Java, Maven, IDE setup, and project scaffolding. Every step reflects real-world practices used in enterprise automation teams.

---

## Prerequisites

Before you begin, ensure the following are installed on your machine:

| Requirement  | Minimum Version                        | Purpose                         |
| ------------ | -------------------------------------- | ------------------------------- |
| **Java JDK** | 11 or higher (17 LTS recommended)      | Runtime and compilation         |
| **Maven**    | 3.8+                                   | Dependency management and build |
| **IDE**      | IntelliJ IDEA (recommended) or Eclipse | Code editing                    |
| **Git**      | Any recent version                     | Version control                 |
| **Browser**  | Chrome (latest)                        | Primary test browser            |

---

## Step 1 — Install Java JDK

### Windows

1. Download the JDK 17 installer from [Adoptium](https://adoptium.net/) (Eclipse Temurin — fully free, production-grade).
2. Run the installer and follow the prompts.
3. Set the `JAVA_HOME` environment variable:
   - Open **System Properties → Advanced → Environment Variables**
   - Add a new System Variable: `JAVA_HOME` = `C:\Program Files\Eclipse Adoptium\jdk-17.x.x`
   - Append `;%JAVA_HOME%\bin` to the `Path` variable.

### macOS

```bash
# Using Homebrew (recommended)
brew install --cask temurin@17

# Verify installation
java -version
```

### Linux (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y

# Verify
java -version
javac -version
```

### Verify Java Installation

```bash
java -version
# Expected output:
# openjdk version "17.x.x" ...

javac -version
# Expected output:
# javac 17.x.x
```

> **Why Java 17?** It is the current LTS (Long-Term Support) release. It has better performance, records, switch expressions, and text blocks — all useful in test code. Java 11 is the absolute minimum for modern Selenium 4.

---

## Step 2 — Install Maven

### Windows

1. Download the binary zip from [maven.apache.org](https://maven.apache.org/download.cgi)
2. Extract to `C:\Program Files\Apache\maven`
3. Set environment variables:
   - `MAVEN_HOME` = `C:\Program Files\Apache\maven`
   - Append `;%MAVEN_HOME%\bin` to `Path`

### macOS

```bash
brew install maven
```

### Linux

```bash
sudo apt install maven -y
```

### Verify Maven

```bash
mvn -version
# Expected output:
# Apache Maven 3.9.x
# Java version: 17.x.x
```

---

## Step 3 — Install IntelliJ IDEA

IntelliJ IDEA Community Edition is free and sufficient for Selenium projects.

1. Download from [jetbrains.com/idea](https://www.jetbrains.com/idea/download/)
2. Install and launch.
3. Install these plugins (File → Settings → Plugins → Marketplace):
   - **Cucumber for Java** — syntax highlighting, step navigation
   - **Gherkin** — feature file support (usually bundled)
   - **Maven Helper** — dependency tree visualization

---

## Step 4 — Create the Maven Project

### Option A: Using IntelliJ IDEA

1. File → New → Project
2. Select **Maven Archetype**
3. Choose `maven-archetype-quickstart`
4. Set GroupId: `com.yourcompany.automation`
5. Set ArtifactId: `selenium-cucumber-framework`
6. Click Finish

### Option B: Using Command Line

```bash
mvn archetype:generate \
  -DgroupId=com.yourcompany.automation \
  -DartifactId=selenium-cucumber-framework \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.4 \
  -DinteractiveMode=false

cd selenium-cucumber-framework
```

---

## Step 5 — Configure pom.xml

Replace the contents of `pom.xml` with this complete, production-ready configuration:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.yourcompany.automation</groupId>
    <artifactId>selenium-cucumber-framework</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>Selenium Cucumber Framework</name>
    <description>Enterprise Selenium + Cucumber + POM Automation Framework</description>

    <properties>
        <java.version>17</java.version>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

        <!-- Dependency Versions -->
        <selenium.version>4.18.1</selenium.version>
        <cucumber.version>7.15.0</cucumber.version>
        <testng.version>7.9.0</testng.version>
        <webdrivermanager.version>5.7.0</webdrivermanager.version>
        <extentreports.version>5.1.1</extentreports.version>
        <log4j.version>2.22.1</log4j.version>
        <jackson.version>2.16.1</jackson.version>
        <assertj.version>3.25.1</assertj.version>
    </properties>

    <dependencies>

        <!-- ==================== SELENIUM ==================== -->
        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <version>${selenium.version}</version>
        </dependency>

        <!-- ==================== WEBDRIVER MANAGER ==================== -->
        <!-- Automatically manages browser driver binaries -->
        <dependency>
            <groupId>io.github.bonigarcia</groupId>
            <artifactId>webdrivermanager</artifactId>
            <version>${webdrivermanager.version}</version>
        </dependency>

        <!-- ==================== CUCUMBER ==================== -->
        <dependency>
            <groupId>io.cucumber</groupId>
            <artifactId>cucumber-java</artifactId>
            <version>${cucumber.version}</version>
        </dependency>
        <dependency>
            <groupId>io.cucumber</groupId>
            <artifactId>cucumber-testng</artifactId>
            <version>${cucumber.version}</version>
        </dependency>

        <!-- ==================== TESTNG ==================== -->
        <dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>${testng.version}</version>
        </dependency>

        <!-- ==================== REPORTING ==================== -->
        <dependency>
            <groupId>com.aventstack</groupId>
            <artifactId>extentreports</artifactId>
            <version>${extentreports.version}</version>
        </dependency>

        <!-- ==================== LOGGING ==================== -->
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
            <artifactId>log4j-slf4j2-impl</artifactId>
            <version>${log4j.version}</version>
        </dependency>

        <!-- ==================== JSON / DATA ==================== -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version>
        </dependency>

        <!-- ==================== ASSERTIONS ==================== -->
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>${assertj.version}</version>
        </dependency>

        <!-- ==================== APACHE COMMONS ==================== -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.14.0</version>
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
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                </configuration>
            </plugin>

            <!-- Maven Surefire Plugin — runs TestNG tests -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
                <configuration>
                    <suiteXmlFiles>
                        <suiteXmlFile>testng.xml</suiteXmlFile>
                    </suiteXmlFiles>
                    <!-- Pass system properties for environment control -->
                    <systemPropertyVariables>
                        <browser>${browser}</browser>
                        <environment>${environment}</environment>
                        <headless>${headless}</headless>
                    </systemPropertyVariables>
                </configuration>
            </plugin>

        </plugins>

        <!-- Ensure resources are included -->
        <testResources>
            <testResource>
                <directory>src/test/resources</directory>
            </testResource>
        </testResources>
    </build>

</project>
```

---

## Step 6 — Create Project Directory Structure

Create the following folder structure. This is the standard layout for a Cucumber + POM framework:

```
selenium-cucumber-framework/
├── src/
│   ├── main/
│   │   └── java/
│   │       └── com/yourcompany/automation/
│   │           ├── pages/           ← Page Object classes
│   │           ├── components/      ← Reusable page components
│   │           └── utils/           ← Utility/helper classes
│   └── test/
│       ├── java/
│       │   └── com/yourcompany/automation/
│       │       ├── runner/          ← Cucumber TestNG runner
│       │       ├── steps/           ← Step Definition classes
│       │       └── hooks/           ← Before/After hooks
│       └── resources/
│           ├── features/            ← Gherkin .feature files
│           │   ├── login/
│           │   └── checkout/
│           ├── config/
│           │   └── config.properties
│           ├── testdata/
│           │   └── users.json
│           └── log4j2.xml
├── drivers/                         ← (optional) local driver binaries
├── reports/                         ← Generated test reports
├── screenshots/                     ← Captured failure screenshots
├── testng.xml                       ← TestNG suite configuration
└── pom.xml
```

### Create the directory structure via terminal:

```bash
mkdir -p src/main/java/com/yourcompany/automation/{pages,components,utils}
mkdir -p src/test/java/com/yourcompany/automation/{runner,steps,hooks}
mkdir -p src/test/resources/{features/login,features/checkout,config,testdata}
mkdir -p {reports,screenshots}
```

---

## Step 7 — Create Configuration File

Create `src/test/resources/config/config.properties`:

```properties
# ===================================
# Application Configuration
# ===================================
base.url=https://yourapp.com
app.title=Your Application

# ===================================
# Browser Configuration
# ===================================
browser=chrome
headless=false

# ===================================
# Timeout Configuration (seconds)
# ===================================
implicit.wait=0
explicit.wait=10
page.load.timeout=30
script.timeout=30

# ===================================
# Screenshot Configuration
# ===================================
screenshot.on.failure=true
screenshot.path=screenshots/

# ===================================
# Report Configuration
# ===================================
report.path=reports/
report.name=AutomationReport

# ===================================
# Test Data
# ===================================
testdata.path=src/test/resources/testdata/
```

---

## Step 8 — Create testng.xml

Create `testng.xml` in the project root:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE suite SYSTEM "http://testng.org/testng-1.0.dtd">
<suite name="Automation Test Suite" verbose="1" parallel="false">

    <test name="Regression Tests">
        <classes>
            <class name="com.yourcompany.automation.runner.TestRunner"/>
        </classes>
    </test>

</suite>
```

---

## Step 9 — Verify the Setup

Run a quick Maven dependency resolution to confirm everything is configured correctly:

```bash
# Download all dependencies
mvn dependency:resolve

# Clean and compile the project (no tests yet)
mvn clean compile

# Expected output at the end:
# [INFO] BUILD SUCCESS
```

If you see `BUILD SUCCESS`, your environment is correctly configured.

---

## Common Setup Issues and Fixes

| Issue                                  | Cause                        | Fix                                                      |
| -------------------------------------- | ---------------------------- | -------------------------------------------------------- |
| `JAVA_HOME not set`                    | Environment variable missing | Set `JAVA_HOME` and restart terminal                     |
| `mvn: command not found`               | Maven not on PATH            | Add Maven `bin` to `PATH`                                |
| `BUILD FAILURE - dependency not found` | Network issue or typo        | Check internet connection, verify dependency coordinates |
| `Unsupported class file version`       | Java version mismatch        | Ensure `JAVA_HOME` points to the right JDK version       |
| IntelliJ not recognizing Maven         | Maven not auto-imported      | Right-click `pom.xml` → Add as Maven Project             |

---

## Summary

You now have a fully configured Maven project with:

- Java 17 runtime
- Selenium 4 and WebDriverManager
- Cucumber 7 with TestNG integration
- Log4j2 for logging
- ExtentReports for HTML reporting
- A professional project directory structure
- A configuration properties file for environment control

The next section covers **Browser Drivers** — understanding how ChromeDriver, GeckoDriver, and WebDriverManager work together to connect Selenium to browsers.
