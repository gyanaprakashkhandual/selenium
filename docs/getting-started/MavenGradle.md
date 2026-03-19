# Maven and Gradle Setup

## Overview

Build tools are the backbone of any professional Java automation project. They manage dependencies, compile code, run tests, and integrate with CI/CD pipelines. This document covers both **Maven** and **Gradle** in depth — explaining when to use each, how to configure them for a Selenium + Cucumber project, and the essential commands you will use every day.

In most enterprise automation teams, **Maven** is the dominant choice. Gradle is more common in product engineering teams and Android projects. Both are covered so you can work confidently in either environment.

---

## Maven

### What is Maven?

Apache Maven is a build automation tool based on the **Project Object Model (POM)**. Your project's entire build configuration — dependencies, plugins, build lifecycle, profiles — is declared in a single XML file: `pom.xml`.

Maven follows **Convention over Configuration**: if you place your files in the expected locations (`src/main/java`, `src/test/java`, `src/test/resources`), Maven builds correctly with minimal configuration.

### Maven Build Lifecycle

Maven has three built-in lifecycles. The most important for testing is the **default** lifecycle:

```
validate → compile → test-compile → test → package → verify → install → deploy
```

| Phase | What It Does |
|---|---|
| `validate` | Checks the project structure is correct |
| `compile` | Compiles `src/main/java` source files |
| `test-compile` | Compiles `src/test/java` test source files |
| `test` | Runs tests using Surefire plugin |
| `package` | Packages compiled code (JAR/WAR) |
| `verify` | Runs integration checks |
| `install` | Installs the artifact to local Maven repository |
| `deploy` | Deploys artifact to remote repository |

> Each phase includes all previous phases. Running `mvn test` also runs `validate`, `compile`, and `test-compile` first.

---

## Complete pom.xml for Selenium + Cucumber

This is a production-grade `pom.xml` with every section explained:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <!-- ============================================================
         PROJECT COORDINATES
         groupId  : Your organization's reverse domain name
         artifactId: The project name (hyphen-separated, lowercase)
         version  : Semantic versioning; SNAPSHOT = in development
    ============================================================ -->
    <groupId>com.yourcompany.automation</groupId>
    <artifactId>selenium-cucumber-framework</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>Selenium Cucumber Automation Framework</name>
    <description>
        Enterprise-grade automation framework using Selenium 4, Cucumber 7,
        and Page Object Model pattern.
    </description>

    <!-- ============================================================
         PROPERTIES — Centralize version numbers here
         Reference as ${property.name} anywhere in the POM
    ============================================================ -->
    <properties>
        <!-- Java version -->
        <java.version>17</java.version>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>

        <!-- Test framework versions -->
        <selenium.version>4.18.1</selenium.version>
        <cucumber.version>7.15.0</cucumber.version>
        <testng.version>7.9.0</testng.version>
        <webdrivermanager.version>5.7.0</webdrivermanager.version>

        <!-- Reporting -->
        <extentreports.version>5.1.1</extentreports.version>
        <allure.version>2.25.0</allure.version>

        <!-- Utilities -->
        <log4j.version>2.22.1</log4j.version>
        <jackson.version>2.16.1</jackson.version>
        <assertj.version>3.25.1</assertj.version>
        <commons.lang.version>3.14.0</commons.lang.version>
        <commons.io.version>2.15.1</commons.io.version>
        <faker.version>1.0.2</faker.version>

        <!-- Maven plugin versions -->
        <maven.compiler.plugin.version>3.12.1</maven.compiler.plugin.version>
        <maven.surefire.plugin.version>3.2.5</maven.surefire.plugin.version>
        <allure.maven.version>2.12.0</allure.maven.version>

        <!-- Runtime configuration (overridable from CLI) -->
        <browser>chrome</browser>
        <headless>false</headless>
        <environment>staging</environment>
        <threads>1</threads>
    </properties>

    <!-- ============================================================
         DEPENDENCIES
    ============================================================ -->
    <dependencies>

        <!-- ===== SELENIUM ===== -->
        <dependency>
            <groupId>org.seleniumhq.selenium</groupId>
            <artifactId>selenium-java</artifactId>
            <version>${selenium.version}</version>
        </dependency>

        <!-- ===== WEBDRIVER MANAGER ===== -->
        <dependency>
            <groupId>io.github.bonigarcia</groupId>
            <artifactId>webdrivermanager</artifactId>
            <version>${webdrivermanager.version}</version>
        </dependency>

        <!-- ===== CUCUMBER ===== -->
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
        <!-- For dependency injection between step classes -->
        <dependency>
            <groupId>io.cucumber</groupId>
            <artifactId>cucumber-picocontainer</artifactId>
            <version>${cucumber.version}</version>
        </dependency>

        <!-- ===== TESTNG ===== -->
        <dependency>
            <groupId>org.testng</groupId>
            <artifactId>testng</artifactId>
            <version>${testng.version}</version>
        </dependency>

        <!-- ===== REPORTING ===== -->
        <!-- Extent Reports -->
        <dependency>
            <groupId>com.aventstack</groupId>
            <artifactId>extentreports</artifactId>
            <version>${extentreports.version}</version>
        </dependency>
        <!-- Allure TestNG -->
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-testng</artifactId>
            <version>${allure.version}</version>
        </dependency>
        <!-- Allure Cucumber -->
        <dependency>
            <groupId>io.qameta.allure</groupId>
            <artifactId>allure-cucumber7-jvm</artifactId>
            <version>${allure.version}</version>
        </dependency>

        <!-- ===== LOGGING ===== -->
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

        <!-- ===== DATA HANDLING ===== -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version>
        </dependency>
        <!-- For reading Excel test data -->
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-ooxml</artifactId>
            <version>5.2.5</version>
        </dependency>

        <!-- ===== ASSERTIONS ===== -->
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>${assertj.version}</version>
        </dependency>

        <!-- ===== UTILITIES ===== -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>${commons.lang.version}</version>
        </dependency>
        <dependency>
            <groupId>commons-io</groupId>
            <artifactId>commons-io</artifactId>
            <version>${commons.io.version}</version>
        </dependency>
        <!-- Faker — generates realistic test data -->
        <dependency>
            <groupId>com.github.javafaker</groupId>
            <artifactId>javafaker</artifactId>
            <version>${faker.version}</version>
        </dependency>

    </dependencies>

    <!-- ============================================================
         BUILD CONFIGURATION
    ============================================================ -->
    <build>
        <plugins>

            <!-- Maven Compiler Plugin -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>${maven.compiler.plugin.version}</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>

            <!-- Maven Surefire Plugin — executes TestNG tests -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>${maven.surefire.plugin.version}</version>
                <configuration>
                    <!-- Point to your TestNG XML file -->
                    <suiteXmlFiles>
                        <suiteXmlFile>testng.xml</suiteXmlFile>
                    </suiteXmlFiles>
                    <!-- Thread count for parallel execution -->
                    <forkCount>1</forkCount>
                    <reuseForks>true</reuseForks>
                    <!-- Pass system properties to tests -->
                    <systemPropertyVariables>
                        <browser>${browser}</browser>
                        <headless>${headless}</headless>
                        <environment>${environment}</environment>
                        <threads>${threads}</threads>
                    </systemPropertyVariables>
                    <!-- Keep running after test failures (don't stop at first failure) -->
                    <testFailureIgnore>false</testFailureIgnore>
                </configuration>
            </plugin>

            <!-- Allure Maven Plugin — generate Allure reports -->
            <plugin>
                <groupId>io.qameta.allure</groupId>
                <artifactId>allure-maven</artifactId>
                <version>${allure.maven.version}</version>
                <configuration>
                    <reportVersion>${allure.version}</reportVersion>
                    <resultsDirectory>target/allure-results</resultsDirectory>
                </configuration>
            </plugin>

        </plugins>

        <!-- Include resource directories -->
        <testResources>
            <testResource>
                <directory>src/test/resources</directory>
                <filtering>true</filtering>
            </testResource>
        </testResources>
    </build>

    <!-- ============================================================
         PROFILES — Environment-specific configuration
         Usage: mvn test -Pstaging
    ============================================================ -->
    <profiles>

        <profile>
            <id>staging</id>
            <properties>
                <environment>staging</environment>
                <base.url>https://staging.yourapp.com</base.url>
            </properties>
        </profile>

        <profile>
            <id>production</id>
            <properties>
                <environment>production</environment>
                <base.url>https://www.yourapp.com</base.url>
            </properties>
        </profile>

        <profile>
            <id>ci</id>
            <properties>
                <headless>true</headless>
                <browser>chrome</browser>
                <threads>4</threads>
            </properties>
        </profile>

        <profile>
            <id>parallel</id>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-surefire-plugin</artifactId>
                        <configuration>
                            <suiteXmlFiles>
                                <suiteXmlFile>testng-parallel.xml</suiteXmlFile>
                            </suiteXmlFiles>
                        </configuration>
                    </plugin>
                </plugins>
            </build>
        </profile>

    </profiles>

</project>
```

---

## Essential Maven Commands

### Daily Use Commands

```bash
# Download all dependencies (run once after adding new ones)
mvn dependency:resolve

# Clean the target/ directory (removes all compiled files and reports)
mvn clean

# Compile the project without running tests
mvn clean compile

# Run all tests
mvn clean test

# Run tests in a specific class
mvn clean test -Dtest=LoginTest

# Run tests matching a pattern
mvn clean test -Dtest="Login*"

# Run a specific method
mvn clean test -Dtest=LoginTest#testSuccessfulLogin

# Skip tests (just compile and package)
mvn clean package -DskipTests

# Run tests with custom properties (override pom.xml values)
mvn clean test -Dbrowser=firefox
mvn clean test -Dheadless=true
mvn clean test -Denvironment=staging

# Run with a specific Maven profile
mvn clean test -Pci
mvn clean test -Pstaging
mvn clean test -Pparallel

# Combine profile and property overrides
mvn clean test -Pci -Dbrowser=firefox -Dheadless=true

# Run and generate Allure report
mvn clean test allure:report

# Serve Allure report locally on http://localhost:8080
mvn allure:serve
```

### Dependency Management Commands

```bash
# Show dependency tree (helps resolve version conflicts)
mvn dependency:tree

# Show only direct dependencies
mvn dependency:tree -Dincludes=org.seleniumhq.selenium

# Analyze unused/undeclared dependencies
mvn dependency:analyze

# Check for newer dependency versions
mvn versions:display-dependency-updates

# Check for newer plugin versions
mvn versions:display-plugin-updates

# Enforce no SNAPSHOT dependencies (useful in CI)
mvn enforcer:enforce
```

---

## Gradle Setup

### When to Use Gradle

Gradle is preferred when:
- Your project uses Kotlin DSL (`.gradle.kts`)
- You are in an Android or Spring Boot team context
- Build performance matters at large scale (Gradle's incremental build is faster)
- Your team already standardizes on Gradle

### Complete build.gradle for Selenium + Cucumber

```groovy
plugins {
    id 'java'
}

group = 'com.yourcompany.automation'
version = '1.0.0-SNAPSHOT'
sourceCompatibility = JavaVersion.VERSION_17
targetCompatibility = JavaVersion.VERSION_17

repositories {
    mavenCentral()
}

ext {
    seleniumVersion    = '4.18.1'
    cucumberVersion    = '7.15.0'
    testngVersion      = '7.9.0'
    wdmVersion         = '5.7.0'
    extentVersion      = '5.1.1'
    log4jVersion       = '2.22.1'
    jacksonVersion     = '2.16.1'
    assertjVersion     = '3.25.1'
}

dependencies {
    // Selenium
    implementation "org.seleniumhq.selenium:selenium-java:${seleniumVersion}"

    // WebDriverManager
    implementation "io.github.bonigarcia:webdrivermanager:${wdmVersion}"

    // Cucumber
    testImplementation "io.cucumber:cucumber-java:${cucumberVersion}"
    testImplementation "io.cucumber:cucumber-testng:${cucumberVersion}"
    testImplementation "io.cucumber:cucumber-picocontainer:${cucumberVersion}"

    // TestNG
    testImplementation "org.testng:testng:${testngVersion}"

    // Extent Reports
    testImplementation "com.aventstack:extentreports:${extentVersion}"

    // Logging
    implementation "org.apache.logging.log4j:log4j-api:${log4jVersion}"
    implementation "org.apache.logging.log4j:log4j-core:${log4jVersion}"
    implementation "org.apache.logging.log4j:log4j-slf4j2-impl:${log4jVersion}"

    // Jackson
    implementation "com.fasterxml.jackson.core:jackson-databind:${jacksonVersion}"

    // AssertJ
    testImplementation "org.assertj:assertj-core:${assertjVersion}"

    // Commons
    implementation 'org.apache.commons:commons-lang3:3.14.0'
    implementation 'commons-io:commons-io:2.15.1'
}

test {
    useTestNG {
        suites 'testng.xml'
    }

    // System properties — pass from command line with -Pbrowser=firefox
    systemProperty 'browser', System.getProperty('browser', 'chrome')
    systemProperty 'headless', System.getProperty('headless', 'false')
    systemProperty 'environment', System.getProperty('environment', 'staging')

    // Test output
    testLogging {
        events "passed", "skipped", "failed"
        exceptionFormat "full"
        showStandardStreams = false
    }
}

// Task to run specific tags
task runSmokeTests(type: Test) {
    useTestNG {
        suites 'testng-smoke.xml'
    }
    systemProperty 'browser', System.getProperty('browser', 'chrome')
    systemProperty 'headless', 'true'
}

// Task to run regression with parallel execution
task runRegressionParallel(type: Test) {
    useTestNG {
        suites 'testng-parallel.xml'
    }
    maxParallelForks = 4
    systemProperty 'headless', 'true'
}
```

### Kotlin DSL Version (build.gradle.kts)

```kotlin
plugins {
    java
}

group = "com.yourcompany.automation"
version = "1.0.0-SNAPSHOT"

java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

repositories {
    mavenCentral()
}

val seleniumVersion = "4.18.1"
val cucumberVersion = "7.15.0"
val testngVersion   = "7.9.0"

dependencies {
    implementation("org.seleniumhq.selenium:selenium-java:$seleniumVersion")
    implementation("io.github.bonigarcia:webdrivermanager:5.7.0")

    testImplementation("io.cucumber:cucumber-java:$cucumberVersion")
    testImplementation("io.cucumber:cucumber-testng:$cucumberVersion")
    testImplementation("org.testng:testng:$testngVersion")
    testImplementation("org.assertj:assertj-core:3.25.1")
}

tasks.test {
    useTestNG {
        suites("testng.xml")
    }
    systemProperty("browser", System.getProperty("browser", "chrome"))
    systemProperty("headless", System.getProperty("headless", "false"))
}
```

### Essential Gradle Commands

```bash
# Download dependencies
./gradlew dependencies

# Clean build directory
./gradlew clean

# Compile without running tests
./gradlew compileTestJava

# Run all tests
./gradlew clean test

# Run a specific test class
./gradlew test --tests "com.yourcompany.automation.LoginTest"

# Run with system properties
./gradlew test -Dbrowser=firefox -Dheadless=true

# Run specific custom task
./gradlew runSmokeTests
./gradlew runRegressionParallel

# Show dependency tree
./gradlew dependencies --configuration testRuntimeClasspath

# Continuous build (re-runs on file change)
./gradlew test --continuous
```

---

## Maven vs Gradle Comparison

| Aspect | Maven | Gradle |
|---|---|---|
| **Configuration** | XML (verbose but explicit) | Groovy/Kotlin DSL (concise) |
| **Build speed** | Slower (no incremental builds) | Faster (incremental, build cache) |
| **Learning curve** | Lower (more documentation) | Moderate |
| **Enterprise adoption** | Very high | Growing |
| **Plugin ecosystem** | Mature and vast | Good and growing |
| **IDE support** | Excellent (IntelliJ, Eclipse) | Excellent |
| **Flexibility** | Convention-based | Highly customizable |
| **Recommendation** | Default choice for test frameworks | Use if team already uses Gradle |

---

## Dependency Version Conflict Resolution

### Maven
```xml
<!-- Force a specific version when transitive dependencies conflict -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>33.0.0-jre</version>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- Exclude a transitive dependency -->
<dependency>
    <groupId>org.seleniumhq.selenium</groupId>
    <artifactId>selenium-java</artifactId>
    <version>${selenium.version}</version>
    <exclusions>
        <exclusion>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

### Gradle
```groovy
// Force a version
configurations.all {
    resolutionStrategy.force 'com.google.guava:guava:33.0.0-jre'
}

// Exclude a transitive dependency
implementation("org.seleniumhq.selenium:selenium-java:${seleniumVersion}") {
    exclude group: 'com.google.guava', module: 'guava'
}
```

---

## Summary

Maven is the go-to build tool for enterprise Selenium + Cucumber frameworks. Its `pom.xml` centralizes dependency management, plugin configuration, and build profiles. Use profiles to switch between environments and execution modes without changing code. Gradle offers faster builds and a more concise DSL — choose it when your team already uses it or when build performance at scale matters. Every Maven and Gradle command you need is available above for daily development, CI/CD integration, and maintenance.