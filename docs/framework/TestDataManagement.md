# Test Data Management

## Overview

Test data management is the practice of creating, organizing, supplying, and cleaning up the data that automated tests need to execute. In a professional Selenium Cucumber framework, test data is never hard-coded in step definitions or page objects. It is externalized into files, factory classes, or a dedicated data layer — making it easy to maintain, reuse, and update without touching test logic.

This document covers all major approaches to test data management: external files (JSON, CSV, Excel), Java model classes, data factories, Cucumber data tables, environment-specific data, and cleanup strategies.

---

## Why Test Data Management Matters

Hard-coded test data creates the following problems:

- A username change requires editing every step definition that uses it.
- Two scenarios that share a test user conflict when they run in parallel.
- Adding a new test scenario for an edge case requires duplicating existing values.
- Test data specific to the development environment breaks when running against staging.

Externalized, well-organized test data eliminates all of these problems.

---

## Data File Organization

Test data files live in `src/test/resources/testdata/`, organized by entity type:

```
src/test/resources/testdata/
  users.json
  products.json
  orders.json
  addresses.json
  payment-cards.json
  coupon-codes.json
```

---

## Java Model Classes

Model classes are simple POJOs that represent the entities in the test data files. They are placed in `src/main/java/models/` because they are reusable across tests.

### User.java

```java
package com.company.framework.models;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

@JsonIgnoreProperties(ignoreUnknown = true)
public class User {

    private String username;
    private String email;
    private String password;
    private String role;
    private boolean active;
    private String firstName;
    private String lastName;

    // Default constructor required by Jackson
    public User() {}

    public User(String email, String password) {
        this.email = email;
        this.password = password;
    }

    // Getters and setters
    public String getUsername()            { return username; }
    public void setUsername(String v)      { this.username = v; }

    public String getEmail()               { return email; }
    public void setEmail(String v)         { this.email = v; }

    public String getPassword()            { return password; }
    public void setPassword(String v)      { this.password = v; }

    public String getRole()                { return role; }
    public void setRole(String v)          { this.role = v; }

    public boolean isActive()              { return active; }
    public void setActive(boolean v)       { this.active = v; }

    public String getFirstName()           { return firstName; }
    public void setFirstName(String v)     { this.firstName = v; }

    public String getLastName()            { return lastName; }
    public void setLastName(String v)      { this.lastName = v; }

    @Override
    public String toString() {
        return "User{email='" + email + "', role='" + role + "'}";
    }
}
```

### Product.java

```java
package com.company.framework.models;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import java.math.BigDecimal;

@JsonIgnoreProperties(ignoreUnknown = true)
public class Product {

    private String id;
    private String name;
    private String category;
    private BigDecimal price;
    private int stock;
    private String sku;
    private boolean available;

    public Product() {}

    // Getters and setters
    public String getId()                { return id; }
    public void setId(String v)          { this.id = v; }

    public String getName()              { return name; }
    public void setName(String v)        { this.name = v; }

    public String getCategory()          { return category; }
    public void setCategory(String v)    { this.category = v; }

    public BigDecimal getPrice()         { return price; }
    public void setPrice(BigDecimal v)   { this.price = v; }

    public int getStock()                { return stock; }
    public void setStock(int v)          { this.stock = v; }

    public String getSku()               { return sku; }
    public void setSku(String v)         { this.sku = v; }

    public boolean isAvailable()         { return available; }
    public void setAvailable(boolean v)  { this.available = v; }
}
```

---

## Test Data Files

### users.json

```json
{
  "users": [
    {
      "username": "admin_user",
      "email": "admin@example.com",
      "password": "AdminPass123",
      "role": "ADMIN",
      "active": true,
      "firstName": "Admin",
      "lastName": "User"
    },
    {
      "username": "manager_user",
      "email": "manager@example.com",
      "password": "ManagerPass456",
      "role": "MANAGER",
      "active": true,
      "firstName": "Jane",
      "lastName": "Manager"
    },
    {
      "username": "buyer_user",
      "email": "buyer@example.com",
      "password": "BuyerPass789",
      "role": "BUYER",
      "active": true,
      "firstName": "John",
      "lastName": "Buyer"
    },
    {
      "username": "inactive_user",
      "email": "inactive@example.com",
      "password": "InactivePass000",
      "role": "BUYER",
      "active": false,
      "firstName": "Old",
      "lastName": "Account"
    }
  ]
}
```

### products.json

```json
{
  "products": [
    {
      "id": "PROD-001",
      "name": "Wireless Mouse",
      "category": "Electronics",
      "price": 29.99,
      "stock": 150,
      "sku": "WM-X200",
      "available": true
    },
    {
      "id": "PROD-002",
      "name": "USB-C Hub",
      "category": "Electronics",
      "price": 49.99,
      "stock": 75,
      "sku": "UCH-7P",
      "available": true
    },
    {
      "id": "PROD-003",
      "name": "Office Chair",
      "category": "Furniture",
      "price": 299.0,
      "stock": 0,
      "sku": "OC-ERGO",
      "available": false
    }
  ]
}
```

---

## TestDataManager — The Central Data Loader

`TestDataManager` reads test data files, deserializes them into model objects, and provides query methods to retrieve specific records.

```java
package com.company.framework.testdata;

import com.company.framework.config.ConfigReader;
import com.company.framework.models.Product;
import com.company.framework.models.User;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.io.InputStream;
import java.util.Arrays;
import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

public class TestDataManager {

    private static final Logger log = LogManager.getLogger(TestDataManager.class);
    private static final ObjectMapper objectMapper = new ObjectMapper();

    private static List<User> users;
    private static List<Product> products;

    static {
        loadAll();
    }

    private TestDataManager() {
        throw new IllegalStateException("TestDataManager is a static utility class");
    }

    // -----------------------------------------------------------------------
    // Loading
    // -----------------------------------------------------------------------

    private static void loadAll() {
        users    = loadFromFile("testdata/users.json",    "users",    User[].class);
        products = loadFromFile("testdata/products.json", "products", Product[].class);
        log.info("Test data loaded — {} users, {} products", users.size(), products.size());
    }

    private static <T> List<T> loadFromFile(String path, String arrayKey, Class<T[]> type) {
        try (InputStream input = TestDataManager.class
                .getClassLoader()
                .getResourceAsStream(path)) {

            if (input == null) {
                log.warn("Test data file not found: {}", path);
                return List.of();
            }

            JsonNode root = objectMapper.readTree(input);
            T[] array = objectMapper.treeToValue(root.get(arrayKey), type);
            return Arrays.asList(array);

        } catch (Exception e) {
            log.error("Failed to load test data from {}: {}", path, e.getMessage());
            throw new RuntimeException("Cannot load test data: " + path, e);
        }
    }

    // -----------------------------------------------------------------------
    // User queries
    // -----------------------------------------------------------------------

    public static User getUserByRole(String role) {
        return users.stream()
            .filter(u -> u.getRole().equalsIgnoreCase(role) && u.isActive())
            .findFirst()
            .orElseThrow(() -> new RuntimeException(
                "No active user found with role: " + role));
    }

    public static User getUserByEmail(String email) {
        return users.stream()
            .filter(u -> u.getEmail().equalsIgnoreCase(email))
            .findFirst()
            .orElseThrow(() -> new RuntimeException(
                "No user found with email: " + email));
    }

    public static User getAdminUser() {
        return getUserByRole("ADMIN");
    }

    public static User getBuyerUser() {
        return getUserByRole("BUYER");
    }

    public static List<User> getInactiveUsers() {
        return users.stream()
            .filter(u -> !u.isActive())
            .collect(Collectors.toList());
    }

    // -----------------------------------------------------------------------
    // Product queries
    // -----------------------------------------------------------------------

    public static Product getProductByName(String name) {
        return products.stream()
            .filter(p -> p.getName().equalsIgnoreCase(name))
            .findFirst()
            .orElseThrow(() -> new RuntimeException(
                "No product found with name: " + name));
    }

    public static Product getProductById(String id) {
        return products.stream()
            .filter(p -> p.getId().equalsIgnoreCase(id))
            .findFirst()
            .orElseThrow(() -> new RuntimeException(
                "No product found with ID: " + id));
    }

    public static List<Product> getAvailableProducts() {
        return products.stream()
            .filter(Product::isAvailable)
            .collect(Collectors.toList());
    }

    public static List<Product> getProductsByCategory(String category) {
        return products.stream()
            .filter(p -> p.getCategory().equalsIgnoreCase(category))
            .collect(Collectors.toList());
    }

    public static Product getFirstAvailableProduct() {
        return getAvailableProducts().stream()
            .findFirst()
            .orElseThrow(() -> new RuntimeException("No available products in test data"));
    }
}
```

---

## Using TestDataManager in Step Definitions

```java
package com.company.framework.stepdefinitions;

import com.company.framework.context.ScenarioContext;
import com.company.framework.models.User;
import com.company.framework.testdata.TestDataManager;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.When;

public class LoginSteps {

    private final ScenarioContext context;

    public LoginSteps(ScenarioContext context) {
        this.context = context;
    }

    @Given("the admin user logs in")
    public void theAdminUserLogsIn() {
        User admin = TestDataManager.getAdminUser();
        context.setDashboardPage(
            context.getLoginPage().login(admin.getEmail(), admin.getPassword())
        );
    }

    @Given("the user logs in as a {string} role")
    public void theUserLogsInWithRole(String role) {
        User user = TestDataManager.getUserByRole(role);
        context.setDashboardPage(
            context.getLoginPage().login(user.getEmail(), user.getPassword())
        );
    }

    @When("the user views the product {string}")
    public void theUserViewsProduct(String productName) {
        // Load product data to set up assertion values
        context.setCurrentProduct(TestDataManager.getProductByName(productName));
        context.getProductsPage().openProduct(productName);
    }
}
```

---

## Test Data Factory Pattern

For scenarios that require unique data on every run — such as user registration — use a factory pattern that generates values programmatically. Static files cannot supply unique email addresses; factories can.

```java
package com.company.framework.testdata;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.UUID;

public class TestDataFactory {

    private TestDataFactory() {}

    /**
     * Generates a unique email address for each test run.
     * Format: test_<timestamp>_<uuid-prefix>@example.com
     */
    public static String generateUniqueEmail() {
        String timestamp = LocalDateTime.now()
            .format(DateTimeFormatter.ofPattern("yyyyMMddHHmmss"));
        String unique = UUID.randomUUID().toString().substring(0, 6);
        return "test_" + timestamp + "_" + unique + "@example.com";
    }

    /**
     * Generates a unique username for each test run.
     */
    public static String generateUniqueUsername() {
        return "user_" + System.currentTimeMillis();
    }

    /**
     * Generates a valid test password meeting standard complexity requirements.
     */
    public static String generatePassword() {
        return "TestPass_" + UUID.randomUUID().toString().substring(0, 8) + "1!";
    }

    /**
     * Generates a random phone number in US format.
     */
    public static String generatePhoneNumber() {
        int area = 200 + (int)(Math.random() * 800);
        int prefix = 200 + (int)(Math.random() * 800);
        int line = 1000 + (int)(Math.random() * 9000);
        return area + "-" + prefix + "-" + line;
    }

    /**
     * Generates a test credit card number (Visa test range).
     */
    public static String generateTestVisaCard() {
        return "4111111111111111"; // Stripe/test gateway test card
    }

    /**
     * Returns a future expiry date for payment forms.
     */
    public static String generateCardExpiry() {
        int year = LocalDateTime.now().getYear() + 2;
        return "12/" + String.valueOf(year).substring(2);
    }
}
```

**Usage in step definitions:**

```java
@When("the user registers a new account")
public void theUserRegistersNewAccount() {
    String email    = TestDataFactory.generateUniqueEmail();
    String password = TestDataFactory.generatePassword();

    context.setRegisteredEmail(email);
    context.setRegisteredPassword(password);

    context.getRegistrationPage()
        .enterEmail(email)
        .enterPassword(password)
        .confirmPassword(password)
        .acceptTerms()
        .submitRegistration();
}

@Then("the user should be able to log in with the new credentials")
public void theUserShouldLogInWithNewCredentials() {
    context.getLoginPage().login(
        context.getRegisteredEmail(),
        context.getRegisteredPassword()
    );
    Assert.assertTrue(context.getDashboardPage().isLoaded());
}
```

---

## CSV Test Data

For large tabular datasets, CSV files are more maintainable than JSON.

**payment-cards.csv:**

```csv
card_type,card_number,expiry,cvv,expected_result
Visa Valid,4111111111111111,12/27,123,success
Visa Declined,4000000000000002,12/27,123,declined
Mastercard Valid,5500005555555559,12/27,456,success
Amex Valid,378282246310005,12/27,1234,success
Expired Card,4111111111111111,01/20,123,expired
```

**CsvDataLoader.java:**

```java
package com.company.framework.testdata;

import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVParser;
import org.apache.commons.csv.CSVRecord;

import java.io.InputStream;
import java.io.InputStreamReader;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class CsvDataLoader {

    public static List<Map<String, String>> load(String resourcePath) {
        List<Map<String, String>> records = new ArrayList<>();

        try (InputStream input = CsvDataLoader.class
                .getClassLoader()
                .getResourceAsStream(resourcePath);
             InputStreamReader reader = new InputStreamReader(input, StandardCharsets.UTF_8);
             CSVParser parser = CSVFormat.DEFAULT
                 .withFirstRecordAsHeader()
                 .withIgnoreHeaderCase()
                 .withTrim()
                 .parse(reader)) {

            for (CSVRecord record : parser) {
                records.add(new HashMap<>(record.toMap()));
            }

        } catch (Exception e) {
            throw new RuntimeException("Failed to load CSV: " + resourcePath, e);
        }

        return records;
    }
}
```

**Maven dependency for Apache Commons CSV:**

```xml
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-csv</artifactId>
    <version>1.10.0</version>
</dependency>
```

---

## Environment-Aware Test Data

Different environments may require different test data (different product IDs, different user credentials). Use a data resolver that selects the appropriate data based on the active environment.

```java
public class TestDataManager {

    private static final String ENV = System.getProperty("env", "dev");

    private static String resolveDataPath(String fileName) {
        // Try environment-specific file first
        String envPath = "testdata/" + ENV + "/" + fileName;
        if (TestDataManager.class.getClassLoader().getResource(envPath) != null) {
            return envPath;
        }
        // Fall back to default
        return "testdata/" + fileName;
    }
}
```

Directory structure:

```
src/test/resources/testdata/
  users.json              ← default / dev data
  products.json
  staging/
    users.json            ← staging-specific users
  prod/
    users.json            ← production smoke test users
```

---

## Test Data Cleanup

Tests that create data during execution (registrations, orders, submitted forms) must clean up after themselves to keep the environment stable for the next run.

### Cleanup in @After Hooks

```java
@After("@creates-user")
public void deleteTestUser() {
    if (context.getRegisteredEmail() != null) {
        DatabaseHelper.deleteUserByEmail(context.getRegisteredEmail());
        log.info("Cleaned up test user: {}", context.getRegisteredEmail());
    }
}

@After("@creates-order")
public void cancelTestOrder() {
    if (context.getCapturedOrderId() != null) {
        ApiHelper.cancelOrder(context.getCapturedOrderId());
        log.info("Cleaned up test order: {}", context.getCapturedOrderId());
    }
}
```

### Tagging Scenarios That Create Data

```gherkin
@registration @creates-user
Scenario: New user registration flow
  When the user registers a new account
  Then the user should receive a welcome email
```

The `@creates-user` tag triggers the cleanup hook, which deletes the test user created during the scenario.

---

## Summary

Professional test data management externalizes all data into files and factory classes, provides a typed query API through `TestDataManager`, generates unique values through `TestDataFactory` for scenarios that require fresh data, and cleans up any data created during test execution. The result is a framework where data changes require updates in one place, scenarios are independent of each other, and the test environment remains consistent across runs.

---

## Related Topics

- ProjectStructure.md — testdata directory location
- ConfigProperties.md — testdata file paths from configuration
- ScenarioOutline.md — Cucumber Examples tables as inline test data
- DataTables.md — Passing structured data directly in Gherkin steps
- Hooks.md — @After cleanup hooks for test data teardown
