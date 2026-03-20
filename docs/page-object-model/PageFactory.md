# Page Factory

## Overview

Page Factory is a built-in Selenium support class that provides an annotation-based approach to initializing WebElements in page objects. Instead of calling `driver.findElement()` every time you need an element, you declare WebElement fields with `@FindBy` annotations and let Page Factory initialize them for you.

Page Factory is part of the `org.openqa.selenium.support` package and is designed specifically to complement the Page Object Model.

---

## How Page Factory Works

When you call `PageFactory.initElements(driver, this)` in a page class constructor, Selenium creates a proxy for each annotated WebElement field. The actual `findElement()` call is deferred until the moment the element is used in an action or assertion. This is called lazy initialization.

This means:

- The element is not located when the page object is created.
- It is located fresh each time a method that uses it is called.
- This avoids stale element issues when the page re-renders between actions.

---

## Maven Dependency

Page Factory is included in the `selenium-support` artifact:

```xml
<dependency>
    <groupId>org.seleniumhq.selenium</groupId>
    <artifactId>selenium-support</artifactId>
    <version>4.18.1</version>
</dependency>
```

---

## @FindBy Annotation

`@FindBy` declares how Selenium should locate the element. It supports all standard locator strategies.

### Syntax

```java
@FindBy(how = How.ID, using = "username")
private WebElement usernameField;

// Shorthand (more common in practice)
@FindBy(id = "username")
private WebElement usernameField;
```

### All Supported Strategies

```java
@FindBy(id = "submitBtn")
private WebElement submitButton;

@FindBy(name = "email")
private WebElement emailField;

@FindBy(className = "error-message")
private WebElement errorLabel;

@FindBy(tagName = "h1")
private WebElement pageHeading;

@FindBy(linkText = "Forgot Password?")
private WebElement forgotPasswordLink;

@FindBy(partialLinkText = "Forgot")
private WebElement forgotPasswordLink;

@FindBy(css = "input[type='text']")
private WebElement textInput;

@FindBy(xpath = "//button[@data-testid='login-btn']")
private WebElement loginButton;
```

---

## Complete Page Factory Example

### LoginPage.java

```java
package pages;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;
import org.openqa.selenium.support.PageFactory;

public class LoginPage {

    private WebDriver driver;

    @FindBy(id = "username")
    private WebElement usernameField;

    @FindBy(id = "password")
    private WebElement passwordField;

    @FindBy(id = "loginBtn")
    private WebElement loginButton;

    @FindBy(css = ".error-message")
    private WebElement errorMessage;

    public LoginPage(WebDriver driver) {
        this.driver = driver;
        PageFactory.initElements(driver, this);
    }

    public void enterUsername(String username) {
        usernameField.clear();
        usernameField.sendKeys(username);
    }

    public void enterPassword(String password) {
        passwordField.clear();
        passwordField.sendKeys(password);
    }

    public DashboardPage clickLogin() {
        loginButton.click();
        return new DashboardPage(driver);
    }

    public DashboardPage login(String username, String password) {
        enterUsername(username);
        enterPassword(password);
        return clickLogin();
    }

    public String getErrorMessage() {
        return errorMessage.getText();
    }

    public boolean isErrorMessageDisplayed() {
        return errorMessage.isDisplayed();
    }
}
```

### DashboardPage.java

```java
package pages;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;
import org.openqa.selenium.support.PageFactory;

public class DashboardPage {

    private WebDriver driver;

    @FindBy(id = "welcomeMsg")
    private WebElement welcomeMessage;

    @FindBy(linkText = "Profile")
    private WebElement profileLink;

    @FindBy(linkText = "Logout")
    private WebElement logoutLink;

    public DashboardPage(WebDriver driver) {
        this.driver = driver;
        PageFactory.initElements(driver, this);
    }

    public String getWelcomeMessage() {
        return welcomeMessage.getText();
    }

    public ProfilePage navigateToProfile() {
        profileLink.click();
        return new ProfilePage(driver);
    }

    public LoginPage logout() {
        logoutLink.click();
        return new LoginPage(driver);
    }
}
```

---

## @FindBys and @FindAll

### @FindBys — Chain Locators (AND logic)

`@FindBys` finds elements that match ALL of the given criteria. The element must satisfy every condition.

```java
// Finds a button that is inside a form AND has class "submit"
@FindBys({
    @FindBy(tagName = "form"),
    @FindBy(className = "submit")
})
private WebElement submitInsideForm;
```

This is equivalent to a CSS descendant selector: `form .submit`.

### @FindAll — Multiple Locators (OR logic)

`@FindAll` finds all elements that match ANY of the given locators and combines the results into a single list.

```java
// Finds all elements that are either buttons or links
@FindAll({
    @FindBy(tagName = "button"),
    @FindBy(tagName = "a")
})
private List<WebElement> clickableElements;
```

---

## Working with Lists of Elements

Use `List<WebElement>` for groups of similar elements.

```java
@FindBy(css = "table#results tbody tr")
private List<WebElement> tableRows;

@FindBy(css = "ul.menu li a")
private List<WebElement> menuItems;

public int getRowCount() {
    return tableRows.size();
}

public String getMenuItemText(int index) {
    return menuItems.get(index).getText();
}

public List<String> getAllMenuItems() {
    List<String> items = new ArrayList<>();
    for (WebElement item : menuItems) {
        items.add(item.getText());
    }
    return items;
}
```

---

## CacheLookup Annotation

By default, Page Factory re-locates elements on every use. If an element is static (it never changes, no re-renders, no dynamic DOM updates), you can annotate it with `@CacheLookup` to locate it once and cache the reference.

```java
@FindBy(id = "staticHeader")
@CacheLookup
private WebElement pageHeader;
```

Use `@CacheLookup` with caution. If the element is replaced by a new DOM node after any interaction (for example, after an AJAX call or page reload), the cached reference will be stale and throw a `StaleElementReferenceException`.

**When it is safe to use:**

- Navigation bars that persist across the page lifecycle
- Static page titles or headings
- Elements that are never replaced, only shown or hidden

**When to avoid it:**

- Elements inside dynamic tables
- Elements that appear after form submission
- Any element inside a React, Angular, or Vue component that re-renders

---

## initElements Variants

Page Factory provides multiple ways to initialize elements:

```java
// Standard initialization — called in constructor
PageFactory.initElements(driver, this);

// Initialize a specific page object from outside the class
LoginPage loginPage = PageFactory.initElements(driver, LoginPage.class);

// Use a custom ElementLocatorFactory
PageFactory.initElements(new AjaxElementLocatorFactory(driver, 10), this);
```

---

## AjaxElementLocatorFactory

For pages with heavy AJAX content, use `AjaxElementLocatorFactory`. It wraps each element lookup with an implicit wait, retrying until the timeout is reached before throwing an exception.

```java
import org.openqa.selenium.support.pagefactory.AjaxElementLocatorFactory;

public class DashboardPage {

    private static final int TIMEOUT_SECONDS = 15;

    private WebDriver driver;

    @FindBy(id = "dynamicContent")
    private WebElement dynamicContent;

    public DashboardPage(WebDriver driver) {
        this.driver = driver;
        PageFactory.initElements(new AjaxElementLocatorFactory(driver, TIMEOUT_SECONDS), this);
    }
}
```

This is particularly useful in single-page applications where elements load asynchronously after the initial page render.

---

## Page Factory vs. Manual findElement

| Aspect              | Page Factory (@FindBy)         | Manual (By locator)          |
| ------------------- | ------------------------------ | ---------------------------- |
| Element declaration | Annotation on field            | By variable or inline        |
| Initialization      | PageFactory.initElements()     | None required                |
| Re-lookup on use    | Yes (lazy, by default)         | On every findElement() call  |
| Caching             | @CacheLookup                   | Store WebElement in variable |
| List support        | List<WebElement> field         | findElements() call          |
| AJAX support        | AjaxElementLocatorFactory      | Explicit WebDriverWait       |
| Readability         | High — annotations are compact | Medium — more boilerplate    |

Both approaches are valid. Page Factory reduces boilerplate. Manual `By` fields give more explicit control and work better with dynamic locators that are built at runtime.

---

## Handling Dynamic Locators

Page Factory `@FindBy` annotations are static — you cannot change them at runtime. For dynamic locators that require a runtime value, use `By` directly alongside the Page Factory pattern.

```java
public class ProductPage {

    private WebDriver driver;

    // Static locators via Page Factory
    @FindBy(id = "searchBox")
    private WebElement searchBox;

    @FindBy(id = "searchBtn")
    private WebElement searchButton;

    public ProductPage(WebDriver driver) {
        this.driver = driver;
        PageFactory.initElements(driver, this);
    }

    // Dynamic locator — built at runtime with a parameter
    public WebElement getProductByName(String productName) {
        String xpath = "//div[@class='product']//h3[text()='" + productName + "']";
        return driver.findElement(By.xpath(xpath));
    }

    public void search(String keyword) {
        searchBox.sendKeys(keyword);
        searchButton.click();
    }
}
```

---

## Common Errors and Solutions

### NoSuchElementException on initElements

Page Factory does not throw an exception when you call `initElements()`. The element is looked up only when you use it. A `NoSuchElementException` during a method call means the element was not found at the time of use — check the locator and ensure the page has loaded.

### StaleElementReferenceException

This occurs when a cached or previously located element is no longer attached to the DOM. Page Factory's default lazy lookup reduces this risk. If it still occurs, remove `@CacheLookup` from the affected element.

### NullPointerException on WebElement

You forgot to call `PageFactory.initElements(driver, this)` in the constructor. All annotated fields will be null without this call.

---

## Summary

Page Factory simplifies WebElement declaration through `@FindBy` annotations and automatic initialization via `PageFactory.initElements()`. It reduces boilerplate, improves readability, and integrates cleanly with the Page Object Model. For most page classes, Page Factory should be your default approach to declaring elements.

---

## Related Topics

- IntroPOM.md — Foundation concepts of Page Object Model
- BasePage.md — Centralizing PageFactory initialization in a base class
- PageComponents.md — Applying Page Factory to reusable components
