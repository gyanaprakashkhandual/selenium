# Page Components

## Overview

Page Components are reusable page object classes that represent portions of a web page shared across multiple screens. A navigation bar, a sidebar, a modal dialog, a data table, a toast notification, or a date picker — these UI elements appear on many pages. Rather than duplicating their locators and interaction methods in every page class, you model each one as a standalone component class.

Page Components are sometimes called Page Fragments, Page Sections, or Widget Objects. The concept is the same regardless of the name: one class, one UI element, used everywhere it appears.

---

## Why Page Components Matter

Consider an e-commerce application where a shopping cart icon with an item count appears in the navigation bar on every page. Without components, every page class that needs to interact with the cart must declare its own locators and methods:

```java
// Product page — cart methods duplicated
public class ProductPage extends BasePage {
    @FindBy(css = ".cart-icon")
    private WebElement cartIcon;

    @FindBy(css = ".cart-count")
    private WebElement cartCount;

    public void clickCartIcon() { cartIcon.click(); }
    public int getCartItemCount() { return Integer.parseInt(cartCount.getText()); }
}

// Checkout page — same cart methods duplicated again
public class CheckoutPage extends BasePage {
    @FindBy(css = ".cart-icon")
    private WebElement cartIcon;

    @FindBy(css = ".cart-count")
    private WebElement cartCount;

    public void clickCartIcon() { cartIcon.click(); }
    public int getCartItemCount() { return Integer.parseInt(cartCount.getText()); }
}
```

Every page has two extra fields and two extra methods that belong to the navigation bar, not the page. When the cart icon CSS changes, you update every page class.

With a component:

```java
// One component class for the cart
public class CartComponent extends BasePage {
    @FindBy(css = ".cart-icon")
    private WebElement cartIcon;

    @FindBy(css = ".cart-count")
    private WebElement cartCount;

    public CartComponent(WebDriver driver) { super(driver); }

    public void open() { click(cartIcon); }
    public int getItemCount() { return Integer.parseInt(getText(cartCount)); }
}
```

Every page that has a cart simply creates a `CartComponent` instance.

---

## Implementing a Navigation Bar Component

The navigation bar appears on every authenticated page.

```java
package pages.components;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;
import pages.BasePage;
import pages.LoginPage;
import pages.ProfilePage;
import pages.SearchResultsPage;

public class NavigationBar extends BasePage {

    @FindBy(css = "nav .brand-logo")
    private WebElement logo;

    @FindBy(css = "nav input.search-box")
    private WebElement searchInput;

    @FindBy(css = "nav button.search-btn")
    private WebElement searchButton;

    @FindBy(css = "nav .user-menu")
    private WebElement userMenuToggle;

    @FindBy(css = "nav .menu-profile")
    private WebElement profileMenuItem;

    @FindBy(css = "nav .menu-logout")
    private WebElement logoutMenuItem;

    @FindBy(css = "nav .cart-count")
    private WebElement cartCount;

    public NavigationBar(WebDriver driver) {
        super(driver);
    }

    public SearchResultsPage search(String keyword) {
        type(searchInput, keyword);
        click(searchButton);
        return new SearchResultsPage(driver);
    }

    public ProfilePage goToProfile() {
        click(userMenuToggle);
        click(profileMenuItem);
        return new ProfilePage(driver);
    }

    public LoginPage logout() {
        click(userMenuToggle);
        click(logoutMenuItem);
        return new LoginPage(driver);
    }

    public int getCartCount() {
        return Integer.parseInt(getText(cartCount));
    }

    public boolean isLogoVisible() {
        return isDisplayed(logo);
    }
}
```

---

## Using a Component Inside a Page Class

Page classes that contain the navigation bar create a component instance and expose it to the test.

```java
package pages;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;
import pages.components.NavigationBar;

public class DashboardPage extends BasePage {

    // The navigation bar is a component — not individual fields
    private NavigationBar navigationBar;

    @FindBy(id = "welcomeMsg")
    private WebElement welcomeMessage;

    @FindBy(css = ".recent-orders tbody tr")
    private java.util.List<WebElement> recentOrderRows;

    public DashboardPage(WebDriver driver) {
        super(driver);
        this.navigationBar = new NavigationBar(driver);
    }

    // Expose the component to the test
    public NavigationBar getNavigationBar() {
        return navigationBar;
    }

    public String getWelcomeMessage() {
        return getText(welcomeMessage);
    }

    public int getRecentOrderCount() {
        return recentOrderRows.size();
    }
}
```

**In the test:**

```java
@Test
public void testSearchFromDashboard() {
    DashboardPage dashboard = loginPage.login("user@example.com", "password");
    SearchResultsPage results = dashboard.getNavigationBar().search("laptop");
    Assert.assertTrue(results.getResultCount() > 0);
}

@Test
public void testLogoutFromDashboard() {
    DashboardPage dashboard = loginPage.login("user@example.com", "password");
    LoginPage login = dashboard.getNavigationBar().logout();
    Assert.assertTrue(login.isDisplayed());
}
```

---

## Implementing a Modal Dialog Component

Modals are a perfect use case for components. They appear on different pages for different purposes — confirmation dialogs, edit forms, image viewers — but they share the same structure.

```java
package pages.components;

import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;
import pages.BasePage;

public class ConfirmationModal extends BasePage {

    private By modalContainer = By.cssSelector(".modal.show");

    @FindBy(css = ".modal.show .modal-title")
    private WebElement modalTitle;

    @FindBy(css = ".modal.show .modal-body p")
    private WebElement modalMessage;

    @FindBy(css = ".modal.show .btn-confirm")
    private WebElement confirmButton;

    @FindBy(css = ".modal.show .btn-cancel")
    private WebElement cancelButton;

    @FindBy(css = ".modal.show .btn-close")
    private WebElement closeButton;

    public ConfirmationModal(WebDriver driver) {
        super(driver);
    }

    public void waitForModal() {
        waitForVisibility(modalContainer);
    }

    public String getTitle() {
        return getText(modalTitle);
    }

    public String getMessage() {
        return getText(modalMessage);
    }

    public void confirm() {
        click(confirmButton);
        waitForInvisibility(modalContainer);
    }

    public void cancel() {
        click(cancelButton);
        waitForInvisibility(modalContainer);
    }

    public void close() {
        click(closeButton);
        waitForInvisibility(modalContainer);
    }

    public boolean isVisible() {
        return isDisplayed(modalContainer);
    }
}
```

---

## Implementing a Data Table Component

Data tables are complex components that warrant their own class, especially when they appear in multiple sections of the application.

```java
package pages.components;

import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;
import pages.BasePage;

import java.util.ArrayList;
import java.util.List;

public class DataTable extends BasePage {

    private By tableRoot;

    public DataTable(WebDriver driver, By tableRoot) {
        super(driver);
        this.tableRoot = tableRoot;
    }

    private WebElement getTable() {
        return waitForVisibility(tableRoot);
    }

    public int getRowCount() {
        List<WebElement> rows = getTable().findElements(By.cssSelector("tbody tr"));
        return rows.size();
    }

    public int getColumnCount() {
        List<WebElement> headers = getTable().findElements(By.cssSelector("thead th"));
        return headers.size();
    }

    public String getCellValue(int row, int column) {
        // row and column are 1-indexed for natural readability
        String css = "tbody tr:nth-child(" + row + ") td:nth-child(" + column + ")";
        return getTable().findElement(By.cssSelector(css)).getText();
    }

    public List<String> getColumnValues(int column) {
        List<WebElement> cells = getTable()
            .findElements(By.cssSelector("tbody tr td:nth-child(" + column + ")"));
        List<String> values = new ArrayList<>();
        for (WebElement cell : cells) {
            values.add(cell.getText());
        }
        return values;
    }

    public void clickRowAction(int row, String actionText) {
        String xpath = ".//tbody/tr[" + row + "]//button[normalize-space()='" + actionText + "']";
        getTable().findElement(By.xpath(xpath)).click();
    }

    public boolean containsText(String text) {
        String xpath = ".//tbody//td[normalize-space()='" + text + "']";
        List<WebElement> matches = getTable().findElements(By.xpath(xpath));
        return !matches.isEmpty();
    }

    public List<String> getHeaderLabels() {
        List<WebElement> headers = getTable().findElements(By.cssSelector("thead th"));
        List<String> labels = new ArrayList<>();
        for (WebElement header : headers) {
            labels.add(header.getText());
        }
        return labels;
    }
}
```

**Using the DataTable component in a page:**

```java
package pages;

import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import pages.components.DataTable;

public class OrdersPage extends BasePage {

    private DataTable ordersTable;

    public OrdersPage(WebDriver driver) {
        super(driver);
        this.ordersTable = new DataTable(driver, By.id("orders-table"));
    }

    public int getOrderCount() {
        return ordersTable.getRowCount();
    }

    public String getOrderId(int row) {
        return ordersTable.getCellValue(row, 1);
    }

    public String getOrderStatus(int row) {
        return ordersTable.getCellValue(row, 4);
    }

    public boolean isOrderPresent(String orderId) {
        return ordersTable.containsText(orderId);
    }

    public void clickViewForRow(int row) {
        ordersTable.clickRowAction(row, "View");
    }
}
```

---

## Component Composition Pattern

A page may contain multiple components. Compose them naturally:

```java
public class ProductListPage extends BasePage {

    private NavigationBar navigationBar;
    private FilterSidebar filterSidebar;
    private PaginationBar pagination;

    @FindBy(css = ".product-card")
    private List<WebElement> productCards;

    public ProductListPage(WebDriver driver) {
        super(driver);
        this.navigationBar = new NavigationBar(driver);
        this.filterSidebar = new FilterSidebar(driver);
        this.pagination = new PaginationBar(driver);
    }

    public NavigationBar getNavigationBar() { return navigationBar; }
    public FilterSidebar getFilters() { return filterSidebar; }
    public PaginationBar getPagination() { return pagination; }

    public int getDisplayedProductCount() {
        return productCards.size();
    }
}
```

**In the test:**

```java
@Test
public void testFilterByCategory() {
    ProductListPage products = dashboardPage.getNavigationBar().search("shoes");
    products.getFilters().selectCategory("Running");
    products.getFilters().applyFilters();

    Assert.assertTrue(products.getDisplayedProductCount() > 0);
    Assert.assertEquals(products.getPagination().getCurrentPage(), 1);
}
```

---

## Component Best Practices

**A component should only know about itself.** It should not navigate to another page, call methods on a parent page, or reference other components. It represents one discrete piece of the UI.

**Pass the WebDriver in through the constructor.** Never let a component create its own WebDriver or retrieve it from a static source.

**Use BasePage as the parent of component classes.** Components need the same wait utilities and interaction methods that page classes use.

**Name components descriptively.** Prefer `NavigationBar`, `FilterSidebar`, `DatePickerWidget`, and `ConfirmationModal` over `Nav`, `Filter`, `DatePicker`, and `Modal`.

**Organize components in a `components` sub-package.** This keeps the package structure clear.

```
pages/
  BasePage.java
  LoginPage.java
  DashboardPage.java
  components/
    NavigationBar.java
    ConfirmationModal.java
    DataTable.java
    PaginationBar.java
    FilterSidebar.java
```

---

## Summary

Page Components extend the POM pattern to sub-page structures, making your framework DRY (Don't Repeat Yourself) at a granular level. Modeling navigation bars, modals, tables, and other shared elements as component classes eliminates duplication, reduces maintenance effort, and makes tests more expressive. Whenever you find yourself copying WebElement locators from one page class to another, that is the signal to extract a component.

---

## Related Topics

- IntroPOM.md — Foundation concepts of Page Object Model
- BasePage.md — The parent class that components extend
- POMInheritance.md — Using inheritance alongside composition
- PageFactory.md — Declaring elements in component classes with @FindBy
