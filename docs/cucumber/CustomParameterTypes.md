# Custom Parameter Types

## Overview

Custom Parameter Types extend Cucumber's built-in parameter matching system with application-specific types. Instead of passing raw strings and converting them inside step definitions, you define a named parameter type once and let Cucumber automatically convert matching text into a typed Java object. This eliminates repetitive conversion code from step definitions and makes the feature file text more natural.

Cucumber's built-in types (`{string}`, `{int}`, `{double}`, `{word}`) handle primitive types. Custom parameter types handle your domain objects — `UserRole`, `OrderStatus`, `Currency`, `DateRange`, and any other type that appears repeatedly in your test scenarios.

---

## How Custom Parameter Types Work

A custom parameter type has three components:

1. **A name** — the identifier used in step expressions, for example `{role}` or `{status}`
2. **A regex pattern** — the text pattern that Cucumber matches in the step text
3. **A transformer** — a method that converts the matched text to a Java object

When Cucumber reads a step, it matches the text against all registered parameter type patterns and calls the transformer to produce the typed value before passing it to the step definition method.

---

## Defining a Custom Parameter Type

Use the `@ParameterType` annotation on a method in a class within the glue path. The method name becomes the parameter type name used in step expressions.

```java
package stepdefinitions;

import io.cucumber.java.ParameterType;
import model.UserRole;

public class ParameterTypes {

    @ParameterType("admin|manager|viewer|auditor")
    public UserRole role(String roleName) {
        return UserRole.fromString(roleName);
    }
}
```

With this definition, the `{role}` parameter type is available in any step expression.

**Feature file:**

```gherkin
When the user logs in as an admin
Then the admin dashboard should be displayed

When the user logs in as a viewer
Then the read-only view should be displayed
```

**Step definition:**

```java
@When("the user logs in as {a} {role}")
public void theUserLogsInAs(UserRole role) {
    loginPage.loginWithRole(role);
}
```

The `UserRole` enum receives the converted value directly.

---

## Domain Model Classes

The parameter type system is most useful when you have enums or value objects that represent domain concepts.

### UserRole Enum

```java
package model;

public enum UserRole {
    ADMIN, MANAGER, VIEWER, AUDITOR;

    public static UserRole fromString(String value) {
        return UserRole.valueOf(value.toUpperCase());
    }
}
```

### OrderStatus Enum

```java
package model;

public enum OrderStatus {
    PENDING, PROCESSING, SHIPPED, DELIVERED, CANCELLED, REFUNDED;

    public static OrderStatus fromString(String value) {
        return OrderStatus.valueOf(value.toUpperCase().replace(" ", "_"));
    }
}
```

### Currency Value Object

```java
package model;

import java.math.BigDecimal;

public class Currency {

    private final BigDecimal amount;
    private final String symbol;

    public Currency(String raw) {
        // Accepts formats: "$29.99", "29.99", "£49.99"
        this.symbol = raw.replaceAll("[0-9.,]", "").trim();
        this.amount = new BigDecimal(raw.replaceAll("[^0-9.]", ""));
    }

    public BigDecimal getAmount() { return amount; }
    public String getSymbol() { return symbol; }

    @Override
    public String toString() {
        return symbol + amount.toPlainString();
    }
}
```

---

## Complete ParameterTypes Registration Class

All `@ParameterType` methods should be grouped in a dedicated class to keep the codebase organized.

```java
package stepdefinitions;

import io.cucumber.java.ParameterType;
import model.Currency;
import model.OrderStatus;
import model.UserRole;

import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

public class ParameterTypes {

    // Matches role names in feature files
    @ParameterType("admin|manager|viewer|auditor|guest")
    public UserRole role(String value) {
        return UserRole.fromString(value);
    }

    // Matches order status descriptions
    @ParameterType("Pending|Processing|Shipped|Delivered|Cancelled|Refunded")
    public OrderStatus orderStatus(String value) {
        return OrderStatus.fromString(value);
    }

    // Matches currency amounts like $29.99 or £49.99
    @ParameterType("\\$[0-9]+\\.[0-9]{2}|£[0-9]+\\.[0-9]{2}|[0-9]+\\.[0-9]{2}")
    public Currency price(String value) {
        return new Currency(value);
    }

    // Matches dates in dd/MM/yyyy format
    @ParameterType("[0-9]{2}/[0-9]{2}/[0-9]{4}")
    public LocalDate date(String value) {
        return LocalDate.parse(value, DateTimeFormatter.ofPattern("dd/MM/yyyy"));
    }

    // Matches boolean-like natural language
    @ParameterType("enabled|disabled")
    public boolean featureState(String value) {
        return value.equalsIgnoreCase("enabled");
    }

    // Matches browser names
    @ParameterType("Chrome|Firefox|Edge|Safari")
    public String browser(String value) {
        return value.toLowerCase();
    }
}
```

---

## Using Custom Parameter Types in Feature Files and Step Definitions

### Example: User Role

```gherkin
Feature: Role-Based Access Control

  Scenario: Admin can access the user management panel
    Given the user is logged in as admin
    When the user navigates to the user management panel
    Then the panel should be accessible

  Scenario: Viewer cannot access the user management panel
    Given the user is logged in as viewer
    When the user navigates to the user management panel
    Then access should be denied with message "Insufficient permissions"
```

```java
@Given("the user is logged in as {role}")
public void theUserIsLoggedInAs(UserRole role) {
    String username = TestUsers.getUsernameForRole(role);
    String password = TestUsers.getPasswordForRole(role);
    loginPage.login(username, password);
}
```

### Example: Order Status

```gherkin
Scenario: Filter orders by Shipped status
  Given the user is on the orders page
  When the user filters by status Shipped
  Then all displayed orders should have status Shipped

Scenario: Cancel a Pending order
  Given the user has an order with status Pending
  When the user cancels the order
  Then the order status should change to Cancelled
```

```java
@When("the user filters by status {orderStatus}")
public void theUserFiltersByStatus(OrderStatus status) {
    ordersPage.selectStatusFilter(status.name());
}

@Then("all displayed orders should have status {orderStatus}")
public void allOrdersShouldHaveStatus(OrderStatus expectedStatus) {
    List<String> statuses = ordersPage.getAllOrderStatuses();
    for (String status : statuses) {
        Assert.assertEquals(
            OrderStatus.fromString(status), expectedStatus,
            "Found order with unexpected status: " + status
        );
    }
}

@Given("the user has an order with status {orderStatus}")
public void theUserHasOrderWithStatus(OrderStatus status) {
    String orderId = OrderHelper.createOrderWithStatus(status);
    context.setOrderId(orderId);
}
```

### Example: Currency Price

```gherkin
Scenario: Product price is displayed correctly
  Given the product "Wireless Mouse" has a price of $29.99
  When the user views the product detail page
  Then the displayed price should be $29.99

Scenario: Discount reduces the price
  Given the product total is $99.99
  When the user applies a 10% discount
  Then the discounted price should be $89.99
```

```java
@Given("the product {string} has a price of {price}")
public void theProductHasPrice(String productName, Currency price) {
    Assert.assertEquals(
        productsPage.getProductPrice(productName),
        price.getAmount()
    );
}

@Then("the displayed price should be {price}")
public void theDisplayedPriceShouldBe(Currency expectedPrice) {
    BigDecimal actualPrice = productDetailPage.getDisplayedPrice();
    Assert.assertEquals(actualPrice, expectedPrice.getAmount(),
        "Price mismatch: expected " + expectedPrice);
}
```

### Example: Date

```gherkin
Scenario: Order placed on a specific date is retrieved correctly
  Given an order was placed on 15/03/2025
  When the user searches orders from 01/03/2025 to 31/03/2025
  Then the order from 15/03/2025 should appear in the results
```

```java
@Given("an order was placed on {date}")
public void anOrderWasPlacedOn(LocalDate orderDate) {
    OrderHelper.createOrderOnDate(orderDate);
}

@When("the user searches orders from {date} to {date}")
public void theUserSearchesOrdersFromTo(LocalDate fromDate, LocalDate toDate) {
    ordersPage.setDateRange(fromDate, toDate);
    ordersPage.clickSearch();
}
```

---

## Named vs Unnamed Groups in Regex

The regex pattern in `@ParameterType` must not use capturing groups (parentheses) unless they are non-capturing. Cucumber uses the entire pattern as one capture group. Using inner capturing groups causes a parameter count mismatch.

```java
// Wrong — inner capturing group creates extra capture
@ParameterType("(active|inactive) user")
public UserState state(String value) { ... }

// Correct — non-capturing group
@ParameterType("(?:active|inactive) user")
public UserState state(String value) { ... }

// Correct — no grouping needed for simple alternation
@ParameterType("active|inactive")
public UserState state(String value) { ... }
```

---

## Usefulness in Large Test Suites

In a framework with 200+ scenarios, manually converting strings to enums in every step definition becomes a significant source of boilerplate and potential bugs. A misspelled status value like "shiped" instead of "shipped" causes a step to pass a wrong value silently if conversion is done manually. With a `@ParameterType`, the regex pattern either matches or does not — a non-matching step fails during scenario discovery, not during execution.

Custom parameter types also improve the feature file's expressiveness. Compare:

```gherkin
# Without custom parameter type — string passed through
When the order status is changed to "SHIPPED"
Then the order status should be "SHIPPED"
```

```gherkin
# With custom parameter type — natural language
When the order status is changed to Shipped
Then the order status should be Shipped
```

---

## Summary

Custom Parameter Types eliminate repetitive type-conversion boilerplate from step definitions and make feature files read more naturally. By registering named patterns with `@ParameterType`, you give Cucumber the ability to recognize domain concepts — roles, statuses, prices, dates — directly in step text and deliver typed Java objects to step definition methods. Every type that appears in three or more step expressions is a candidate for a custom parameter type.

---

## Related Topics

- StepDefinitions.md — Using custom parameter types in step methods
- DataTables.md — DataTableType for custom table-to-object mapping
- DocStrings.md — DocStringType for custom doc string transformation
- GherkinSyntax.md — Writing step text for custom parameter expressions
