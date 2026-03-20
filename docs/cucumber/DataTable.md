# Data Tables

## Overview

Data Tables are a Gherkin construct for passing structured tabular data into a step. They are defined inline within a step using pipe `|` characters and can represent key-value pairs, lists, or full two-dimensional grids of data. Data Tables allow a single step to receive rich, multi-row, multi-column data without requiring multiple separate steps or a Scenario Outline.

Data Tables are not the same as Scenario Outline Examples. Examples drive multiple scenario executions. A Data Table passes structured data into a single step of a single scenario execution.

---

## Basic Syntax

```gherkin
When the user fills in the registration form with:
  | Field        | Value              |
  | First Name   | Jane               |
  | Last Name    | Smith              |
  | Email        | jane@example.com   |
  | Password     | SecureP@ss1        |
```

The table is associated with the step it immediately follows. Indentation is conventional, not required. Trailing whitespace in cells is ignored; leading whitespace is significant.

---

## Data Table Types in Java

Cucumber maps Data Tables to different Java types depending on the table structure and the method signature you choose.

### 1. List of Lists — `DataTable`

The raw `DataTable` type gives access to the table as a list of lists of strings. Every row is a `List<String>`.

```java
import io.cucumber.datatable.DataTable;

@When("the user fills in the registration form with:")
public void theUserFillsInRegistrationForm(DataTable dataTable) {
    List<List<String>> rows = dataTable.asLists(String.class);
    for (List<String> row : rows) {
        String fieldName = row.get(0);
        String value = row.get(1);
        registrationPage.fillField(fieldName, value);
    }
}
```

### 2. List of Maps — `asMaps()`

When the table has a header row, map each row to a `Map<String, String>` where keys are column headers.

**Feature file:**

```gherkin
Then the search results should contain the following products:
  | name            | category    | price  |
  | Wireless Mouse  | Electronics | 29.99  |
  | USB-C Hub       | Electronics | 49.99  |
```

**Step definition:**

```java
@Then("the search results should contain the following products:")
public void theSearchResultsShouldContain(DataTable dataTable) {
    List<Map<String, String>> products = dataTable.asMaps(String.class, String.class);

    for (Map<String, String> product : products) {
        String name     = product.get("name");
        String category = product.get("category");
        String price    = product.get("price");

        Assert.assertTrue(
            resultsPage.isProductPresent(name),
            "Product not found: " + name
        );
        Assert.assertEquals(
            resultsPage.getProductCategory(name), category
        );
        Assert.assertEquals(
            resultsPage.getProductPrice(name), price
        );
    }
}
```

### 3. Map of Maps — `asMap()`

For a two-column key-value table (no header), use `asMap()` to get a single `Map<String, String>`.

**Feature file:**

```gherkin
Given the user has the following profile settings:
  | display_name    | John Doe           |
  | email           | john@example.com   |
  | timezone        | America/New_York   |
  | language        | English            |
```

**Step definition:**

```java
@Given("the user has the following profile settings:")
public void theUserHasProfileSettings(DataTable dataTable) {
    Map<String, String> settings = dataTable.asMap(String.class, String.class);

    String displayName = settings.get("display_name");
    String email       = settings.get("email");
    String timezone    = settings.get("timezone");
    String language    = settings.get("language");

    profilePage.setDisplayName(displayName);
    profilePage.setEmail(email);
    profilePage.setTimezone(timezone);
    profilePage.setLanguage(language);
}
```

### 4. List of POJOs

Cucumber can automatically map rows to Java objects if column headers match field names. This is the cleanest approach for strongly typed data.

**POJO class:**

```java
package model;

public class Product {
    public String name;
    public String category;
    public double price;
    public int stock;
}
```

**Feature file:**

```gherkin
Given the following products exist in the system:
  | name            | category    | price  | stock |
  | Wireless Mouse  | Electronics | 29.99  | 100   |
  | USB-C Hub       | Electronics | 49.99  | 50    |
  | Office Chair    | Furniture   | 299.00 | 20    |
```

**Step definition:**

```java
import io.cucumber.java.en.Given;
import io.cucumber.datatable.DataTable;
import model.Product;
import java.util.List;

@Given("the following products exist in the system:")
public void theFollowingProductsExist(DataTable dataTable) {
    List<Product> products = dataTable.asList(Product.class);

    for (Product product : products) {
        System.out.println("Name: " + product.name);
        System.out.println("Price: " + product.price);
        // Use product to seed database or verify UI
        adminPage.addProduct(product);
    }
}
```

Cucumber uses the column headers as field names and performs type conversion automatically for `int`, `double`, `boolean`, and `String`.

---

## Practical Examples

### Form Filling

```gherkin
When the user completes the checkout form with:
  | field             | value                    |
  | Full Name         | Jane Smith               |
  | Street Address    | 123 Main St              |
  | City              | New York                 |
  | State             | NY                       |
  | ZIP Code          | 10001                    |
  | Phone Number      | 555-0123                 |
```

```java
@When("the user completes the checkout form with:")
public void theUserCompletesCheckoutForm(DataTable dataTable) {
    Map<String, String> formData = dataTable.asMap(String.class, String.class);
    checkoutPage.fillShippingForm(formData);
}
```

```java
// In CheckoutPage
public void fillShippingForm(Map<String, String> formData) {
    formData.forEach((field, value) -> fillField(field, value));
}
```

### Verifying Table Data in the UI

```gherkin
Then the order history table should display:
  | Order ID  | Product         | Status    | Amount  |
  | ORD-1001  | Wireless Mouse  | Delivered | 29.99   |
  | ORD-1002  | USB-C Hub       | Shipped   | 49.99   |
  | ORD-1003  | Office Chair    | Pending   | 299.00  |
```

```java
@Then("the order history table should display:")
public void theOrderHistoryTableShouldDisplay(DataTable dataTable) {
    List<Map<String, String>> expectedOrders = dataTable.asMaps(String.class, String.class);

    for (int i = 0; i < expectedOrders.size(); i++) {
        Map<String, String> expected = expectedOrders.get(i);
        int row = i + 1; // 1-indexed

        Assert.assertEquals(
            ordersPage.getOrderId(row), expected.get("Order ID")
        );
        Assert.assertEquals(
            ordersPage.getProductName(row), expected.get("Product")
        );
        Assert.assertEquals(
            ordersPage.getOrderStatus(row), expected.get("Status")
        );
        Assert.assertEquals(
            ordersPage.getOrderAmount(row), expected.get("Amount")
        );
    }
}
```

### Seeding Multiple Users

```gherkin
Given the following user accounts exist:
  | username  | email                   | role    | active |
  | admin     | admin@example.com       | ADMIN   | true   |
  | manager   | manager@example.com     | MANAGER | true   |
  | viewer    | viewer@example.com      | VIEWER  | true   |
  | suspended | suspended@example.com   | VIEWER  | false  |
```

```java
@Given("the following user accounts exist:")
public void theFollowingUserAccountsExist(DataTable dataTable) {
    List<Map<String, String>> users = dataTable.asMaps(String.class, String.class);

    for (Map<String, String> user : users) {
        DatabaseHelper.createUser(
            user.get("username"),
            user.get("email"),
            user.get("role"),
            Boolean.parseBoolean(user.get("active"))
        );
    }
}
```

---

## Tables Without Headers

If the table does not have a header row, use `asLists()` to access rows directly by index.

```gherkin
Given the following discount codes are active:
  | SAVE10  |
  | SAVE20  |
  | FREESHIP|
```

```java
@Given("the following discount codes are active:")
public void theFollowingDiscountCodesAreActive(DataTable dataTable) {
    List<List<String>> rows = dataTable.asLists(String.class);
    List<String> codes = rows.stream()
        .map(row -> row.get(0))
        .collect(Collectors.toList());

    codes.forEach(code -> DatabaseHelper.activateCoupon(code));
}
```

---

## Data Table Transformer

For fine-grained control over how the table is mapped to a Java type, implement a `TableEntryTransformer` or `TableRowTransformer`.

```java
import io.cucumber.java.DataTableType;
import model.Product;

public class DataTableTransformers {

    @DataTableType
    public Product productEntry(Map<String, String> entry) {
        Product product = new Product();
        product.setName(entry.get("name"));
        product.setCategory(entry.get("category"));
        product.setPrice(Double.parseDouble(entry.get("price")));
        product.setStock(Integer.parseInt(entry.get("stock")));
        return product;
    }
}
```

Once registered, Cucumber uses this transformer automatically when a step method declares `List<Product>`:

```java
@Given("the following products exist:")
public void theFollowingProductsExist(List<Product> products) {
    products.forEach(product -> adminPage.addProduct(product));
}
```

The `@DataTableType` method must be in a class in the glue path.

---

## Comparing Actual vs Expected Tables

The `DataTable` class provides a `diff()` method for comparing expected and actual tables. This is useful for asserting that a UI table matches expected data.

```java
@Then("the product list should match:")
public void theProductListShouldMatch(DataTable expectedTable) {
    List<List<String>> actualData = productsPage.getAllProductsAsTable();
    DataTable actualTable = DataTable.create(actualData);
    expectedTable.diff(actualTable);
    // diff() throws an exception with a detailed comparison if tables differ
}
```

---

## Best Practices

**Use headers for tables with more than two columns.** Headers make the data self-documenting and enable `asMaps()`.

**Keep tables to 5-7 columns maximum.** Wider tables are hard to read in plain text and in reports.

**Use Data Tables for multi-row data.** For a single value, use a Cucumber Expression parameter in the step text. Data Tables shine when you need 3 or more rows or structured fields.

**Align column widths with spaces.** Gherkin ignores whitespace within cells, so you can align columns for readability without affecting behavior.

```gherkin
# Hard to read
| name|category|price|
| Wireless Mouse|Electronics|29.99|

# Easy to read — same data, aligned
| name            | category    | price  |
| Wireless Mouse  | Electronics | 29.99  |
```

**Do not mix concerns in one table.** A table containing both input data and expected output in the same row is confusing. Split them into separate steps if needed.

---

## Summary

Data Tables give Cucumber steps the ability to receive rich, structured data inline in the feature file. Whether used for form input, database seeding, or UI verification, they map cleanly to Java collections and POJOs. The `DataTable` API supports lists, maps, typed objects, and custom transformers, giving full control over how tabular data is consumed in step definitions.

---

## Related Topics

- GherkinSyntax.md — Gherkin table syntax rules
- ScenarioOutline.md — Examples tables for data-driven scenario repetition
- DocStrings.md — Passing multi-line text content to steps
- StepDefinitions.md — Step definition parameter handling
- CustomParameterTypes.md — Custom type converters for step parameters
