# Test Independence

## Overview

Test independence means every scenario in a Selenium Cucumber suite can execute correctly in complete isolation — alone, in any order, in parallel, or as part of a subset — without depending on state set by a previous test or leaving state that affects a subsequent one. It is the single most important architectural quality for a reliable, maintainable, and scalable test suite.

Non-independent tests produce the most insidious failures in automation: tests that pass when run in sequence but fail when run individually, tests that pass in one order but fail in another, and parallel execution failures that have nothing to do with the application under test. Eliminating these dependencies is not optional — it is the prerequisite for everything else in this document set.

---

## What Test Independence Means

A test is independent when all three of these conditions are true:

**Self-contained setup.** The scenario creates or navigates to every precondition it needs. It does not rely on another scenario having run first.

**Self-contained teardown.** The scenario cleans up every resource it creates. It does not leave users, orders, records, or state that another scenario will encounter.

**No shared mutable state.** The scenario does not read from or write to any variable, file, database record, or browser session that another concurrently running scenario also accesses.

---

## The Cost of Non-Independence

### Order Dependency

```gherkin
# BAD — Scenario 2 assumes Scenario 1 already ran
Scenario: Add item to cart                   # Scenario 1
  Given the user is logged in
  When the user adds "Wireless Mouse" to cart
  Then the cart count should be 1

Scenario: Proceed to checkout                # Scenario 2
  # No setup — assumes cart has 1 item from Scenario 1
  When the user navigates to checkout
  Then the checkout page should show 1 item  # FAILS when run alone or out of order
```

If Scenario 2 runs before Scenario 1, or if only Scenario 2 runs (as happens with tag filtering), it fails immediately. If they are separated into different feature files and run in parallel, the cart will be empty.

### Shared Account Conflicts

Two parallel scenarios that both log in as `admin@example.com` and change the admin's display name will overwrite each other's changes. One scenario's `Then` assertion will see the other scenario's data, producing a random-looking failure.

### Database Record Pollution

A scenario that creates an order record but never deletes it leaves that record for every future test run. Over hundreds of CI runs, the test environment accumulates thousands of orphaned records, causing "too many results" failures and performance degradation.

---

## Rule 1 — Each Scenario Sets Its Own State

Every scenario must set up everything it needs in its own `Given` steps or `Background`. It must never assume that another scenario has already performed some action.

### Correct Pattern

```gherkin
# CORRECT — every scenario is self-contained
Feature: Order Management

  Scenario: View order details
    Given the user is logged in as "buyer@example.com"
    And the user has an existing order "ORD-1001"
    When the user views the order details for "ORD-1001"
    Then the order status should be "Pending"

  Scenario: Cancel an order
    Given the user is logged in as "buyer@example.com"
    And the user has a cancellable order "ORD-2001"
    When the user cancels order "ORD-2001"
    Then the order status should be "Cancelled"
```

Each scenario creates its own data setup. Neither depends on the other.

### Implementation with Test Data Setup

```java
@Given("the user has an existing order {string}")
public void theUserHasExistingOrder(String orderId) {
    // Create the order via API — fast and isolated
    OrderHelper.createOrderForUser(
        context.getLoggedInEmail(),
        orderId,
        "Pending"
    );
    // Store for cleanup in @After
    context.addCreatedOrderId(orderId);
}

@Given("the user has a cancellable order {string}")
public void theUserHasCancellableOrder(String orderId) {
    OrderHelper.createOrderForUser(
        context.getLoggedInEmail(),
        orderId,
        "Pending"
    );
    context.addCreatedOrderId(orderId);
}
```

---

## Rule 2 — Each Scenario Cleans Up After Itself

Any data created during a scenario must be deleted when the scenario finishes, regardless of whether it passed or failed.

### Cleanup via @After Hooks

```java
package com.company.framework.hooks;

import com.company.framework.context.ScenarioContext;
import com.company.framework.helpers.OrderHelper;
import com.company.framework.helpers.UserHelper;
import io.cucumber.java.After;
import io.cucumber.java.Scenario;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class DataCleanupHooks {

    private static final Logger log = LogManager.getLogger(DataCleanupHooks.class);
    private final ScenarioContext context;

    public DataCleanupHooks(ScenarioContext context) {
        this.context = context;
    }

    @After(order = 3)  // Runs before screenshot and driver teardown
    public void cleanUpCreatedOrders() {
        if (context.getCreatedOrderIds().isEmpty()) return;

        for (String orderId : context.getCreatedOrderIds()) {
            try {
                OrderHelper.deleteOrder(orderId);
                log.info("Cleaned up order: {}", orderId);
            } catch (Exception e) {
                log.warn("Could not clean up order {}: {}", orderId, e.getMessage());
                // Non-fatal — test result is already recorded
            }
        }
    }

    @After(order = 3)
    public void cleanUpCreatedUsers() {
        if (context.getCreatedUserEmails().isEmpty()) return;

        for (String email : context.getCreatedUserEmails()) {
            try {
                UserHelper.deleteUserByEmail(email);
                log.info("Cleaned up user: {}", email);
            } catch (Exception e) {
                log.warn("Could not clean up user {}: {}", email, e.getMessage());
            }
        }
    }
}
```

### ScenarioContext Tracks Created Resources

```java
public class ScenarioContext {

    private final List<String> createdOrderIds   = new ArrayList<>();
    private final List<String> createdUserEmails = new ArrayList<>();
    private final List<String> createdProductIds = new ArrayList<>();

    public void addCreatedOrderId(String id)     { createdOrderIds.add(id); }
    public List<String> getCreatedOrderIds()     { return createdOrderIds; }

    public void addCreatedUserEmail(String email) { createdUserEmails.add(email); }
    public List<String> getCreatedUserEmails()   { return createdUserEmails; }

    public void addCreatedProductId(String id)   { createdProductIds.add(id); }
    public List<String> getCreatedProductIds()   { return createdProductIds; }
}
```

### Tag-Based Conditional Cleanup

For expensive cleanup operations, scope them to scenarios that need them:

```java
@After("@creates-user")
public void deleteCreatedUser() {
    if (context.getRegisteredEmail() != null) {
        UserHelper.deleteUserByEmail(context.getRegisteredEmail());
    }
}

@After("@modifies-settings")
public void restoreDefaultSettings() {
    SettingsHelper.resetToDefault(context.getLoggedInEmail());
}

@After("@uploads-file")
public void deleteUploadedFiles() {
    context.getUploadedFileIds().forEach(FileHelper::deleteFile);
}
```

---

## Rule 3 — No Shared Mutable State Between Scenarios

### No Static Fields in Step Definitions

```java
// WRONG — static fields are shared across all threads and scenarios
public class LoginSteps {
    private static LoginPage loginPage;       // shared between parallel scenarios
    private static DashboardPage dashboard;  // scenario 2 overwrites scenario 1's reference
}

// CORRECT — instance fields via ScenarioContext (new instance per scenario)
public class LoginSteps {
    private final ScenarioContext context;   // PicoContainer injects per-scenario instance

    public LoginSteps(ScenarioContext context) {
        this.context = context;
    }
}
```

### No Shared Test User Accounts for Conflicting Scenarios

```java
// WRONG — two parallel scenarios both modify the same admin account
@Given("the admin updates notification settings")
public void adminUpdatesSettings() {
    User admin = TestDataManager.getAdminUser(); // always returns same account
    // Both threads log in as the same user and overwrite each other
}

// CORRECT — each scenario gets its own dedicated user
@Given("an admin user is available for this scenario")
public void adminUserIsAvailable() {
    // Option A: Use role-specific pools
    User admin = TestDataManager.getAvailableAdminFromPool();
    context.setCurrentUser(admin);

    // Option B: Create unique user per scenario
    String email = TestDataFactory.generateUniqueEmail();
    UserHelper.createAdminUser(email);
    context.setCurrentUser(new User(email, "Admin"));
    context.addCreatedUserEmail(email);
}
```

### No Direct Database Record Sharing

```java
// WRONG — both parallel scenarios read and modify ORDER-TEST-001
@Given("order ORDER-TEST-001 exists")
public void orderExists() {
    // Scenario 1 sets status to Pending
    // Scenario 2 sets status to Shipped
    // Both then assert on the same record — unpredictable result
}

// CORRECT — each scenario creates its own unique record
@Given("a Pending order exists for this scenario")
public void pendingOrderExists() {
    String uniqueOrderId = "ORD-" + System.currentTimeMillis();
    OrderHelper.createOrder(uniqueOrderId, "Pending");
    context.setCurrentOrderId(uniqueOrderId);
    context.addCreatedOrderId(uniqueOrderId);
}
```

---

## Rule 4 — Use API Setup Instead of UI Setup

Setting up preconditions through the UI is slow and fragile. A login flow alone takes 2–5 seconds. When 50 scenarios each start by logging in through the UI, that is 100–250 seconds of overhead doing nothing related to what is being tested.

Use direct API calls for all setup that is not itself under test.

### Cookie Injection Login (Fastest)

```java
@Before("@authenticated")
public void loginViaCookieInjection() throws Exception {
    String baseUrl = ConfigReader.get("app.base.url");
    User user      = TestDataManager.getAdminUser();

    // API call — milliseconds, not seconds
    String token = ApiAuthHelper.getSessionToken(
        baseUrl, user.getEmail(), user.getPassword()
    );

    // Navigate to base URL to set domain context
    context.getDriver().get(baseUrl);

    // Inject the session cookie — instant authenticated state
    Cookie sessionCookie = new Cookie.Builder("auth_token", token)
        .domain(baseUrl.replaceAll("https?://", "").split("/")[0])
        .path("/")
        .isSecure(true)
        .build();

    context.getDriver().manage().addCookie(sessionCookie);
    log.info("Session cookie injected for: {}", user.getEmail());
}
```

### API-Based Test Data Creation

```java
@Given("the user's cart contains {string}")
public void userCartContains(String productName) {
    // Direct API call — no UI navigation required
    String productId = TestDataManager.getProductByName(productName).getId();
    String cartId = ApiCartHelper.addToCart(
        context.getSessionToken(), productId, 1
    );
    context.setCartId(cartId);
    log.info("Added {} to cart via API", productName);
}
```

This setup step executes in 200–500ms via API vs 5–10 seconds via UI navigation.

---

## Rule 5 — Database State Isolation

For test environments with a shared database, use one of these isolation strategies:

### Strategy A — Create and Delete Per Scenario

Each scenario creates its own data records, stores their IDs, and deletes them in `@After`. Covered in Rule 2 above.

### Strategy B — Database Transaction Rollback

Wrap each test in a database transaction and roll it back on teardown:

```java
@Before("@db-isolated")
public void beginTransaction() {
    DatabaseHelper.beginTransaction();
}

@After("@db-isolated")
public void rollbackTransaction() {
    DatabaseHelper.rollbackTransaction();
}
```

Every insert or update made during the scenario is reversed automatically. No cleanup code required per entity type.

### Strategy C — Test Data Namespacing

Prefix all test-created records with a unique test run identifier:

```java
public class TestDataFactory {

    private static final String RUN_PREFIX = "TEST_" + System.currentTimeMillis() + "_";

    public static String generateOrderId() {
        return RUN_PREFIX + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
    }

    public static String generateEmail() {
        return "test_" + System.currentTimeMillis() + "@automation.example.com";
    }
}
```

A scheduled cleanup job deletes all records with the `TEST_` prefix older than 24 hours. This decouples cleanup from individual test scenarios.

---

## Rule 6 — Idempotent Before Hooks

`@Before` hooks must produce the same starting state regardless of what any previous scenario left behind. They must not assume a clean environment.

```java
// FRAGILE — assumes the previous scenario left the browser on the base URL
@Before
public void navigateToApp() {
    driver.findElement(By.id("homeLink")).click();  // may fail if no homeLink
}

// CORRECT — always navigates explicitly, no assumption about current state
@Before
public void navigateToApp() {
    driver.get(ConfigReader.get("app.base.url"));   // explicit, always works
}
```

```java
// FRAGILE — assumes the cart is empty
@Before("@cart-test")
public void setupCart() {
    addItemToCart("Laptop");  // adds 1 item, but previous test may have left 3
}

// CORRECT — clear first, then set up
@Before("@cart-test")
public void setupCart() {
    ApiCartHelper.clearCart(context.getSessionToken());  // explicit clear
    ApiCartHelper.addItem(context.getSessionToken(), "Laptop");
}
```

---

## Rule 7 — Feature File Independence

Feature files themselves must not imply execution order. Each feature file covers one functional area and is self-sufficient.

```
# WRONG layout — Feature B depends on Feature A having been executed
features/
  01-login.feature       (sets up the logged-in state somehow)
  02-dashboard.feature   (assumes user is already logged in)
  03-products.feature    (assumes user is on products page)

# CORRECT layout — every feature is self-contained
features/
  authentication/
    login.feature        (each scenario sets up its own state)
  dashboard/
    dashboard.feature    (each scenario logs in independently)
  products/
    product-search.feature (each scenario navigates independently)
```

Numbered prefixes on feature files are a red flag that execution order is being relied upon.

---

## The Payoff of Independence

When every scenario is truly independent, the following become possible without any additional work:

**Tag filtering.** Run only `@smoke` scenarios without worrying that skipped scenarios were setting up state that smoke tests depend on.

**Parallel execution.** Run 8 scenarios simultaneously — none of them will interfere with the others.

**Individual debugging.** Run a single failing scenario in isolation to reproduce and debug without running the whole suite.

**Order randomization.** Randomize execution order to catch hidden dependencies. Every run should produce the same results regardless of order.

**Reliable CI pipelines.** Failures indicate genuine application defects, not execution order artifacts.

---

## Summary

Test independence is not a nice-to-have quality — it is the architectural requirement that makes every other best practice in this guide work. Independent scenarios set up their own state using explicit `Given` steps and API helpers. They clean up everything they create in `@After` hooks tracked through `ScenarioContext`. They use unique test data generated per scenario rather than shared accounts. They avoid static state in step definitions. They use idempotent hooks that produce a known starting state regardless of prior scenario results. The investment in independence pays compound interest: the test suite becomes faster, more reliable, and more maintainable with every scenario added to it.

---

## Related Topics

- TestDataManagement.md — TestDataFactory for unique per-scenario data
- Hooks.md — @Before and @After lifecycle for setup and cleanup
- WorldObject.md — ScenarioContext for per-scenario state isolation
- ParallelExecution.md — Why independence is the prerequisite for parallelism
- Cookies.md — Session cookie injection for fast authenticated setup
