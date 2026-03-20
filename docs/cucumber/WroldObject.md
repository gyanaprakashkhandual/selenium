# World Object

## Overview

The World Object is a Cucumber concept for maintaining shared state across step definition classes within a single scenario. In Cucumber-JVM (the Java implementation), the World Object is implemented through dependency injection — step definition classes declare shared state objects in their constructors, and the DI container creates and injects a single instance of each shared object for each scenario.

The most practical implementation in Java uses PicoContainer, a lightweight DI container included in the `cucumber-picocontainer` library. The "World" itself is typically a custom context class (often called `ScenarioContext` or `World`) that holds the state — WebDriver references, page objects, intermediate values, and any other data that must flow between steps.

---

## Why the World Object Pattern Is Needed

Cucumber creates a new instance of each step definition class for every scenario. Fields declared in one step definition class are not accessible from another step definition class.

Consider a login step that sets `dashboardPage`, and a dashboard step that needs to use it:

```java
// LoginSteps.java
public class LoginSteps {
    private DashboardPage dashboardPage; // set here
}

// DashboardSteps.java
public class DashboardSteps {
    // dashboardPage is not available here — different class instance
}
```

Without a sharing mechanism, each class is isolated. The World Object solves this by providing a single container that both classes reference.

---

## Setting Up PicoContainer

### Maven Dependency

```xml
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-picocontainer</artifactId>
    <version>7.15.0</version>
    <scope>test</scope>
</dependency>
```

PicoContainer is automatically detected when it is on the classpath. No additional configuration is required.

---

## Implementing the World Class

The World class is a plain Java object. It holds the shared state for a scenario. Create it in a package within the glue path.

```java
package context;

import org.openqa.selenium.WebDriver;
import pages.*;

public class World {

    // WebDriver — shared across all step definition classes
    private WebDriver driver;

    // Page objects — populated as the scenario progresses
    private LoginPage loginPage;
    private DashboardPage dashboardPage;
    private ProductsPage productsPage;
    private CartPage cartPage;
    private CheckoutPage checkoutPage;

    // Scenario data — values captured during steps for use in later steps
    private String capturedOrderId;
    private String capturedConfirmationNumber;
    private String capturedErrorMessage;
    private double capturedOrderTotal;

    // -------------------------------------------------------------------
    // Driver
    // -------------------------------------------------------------------

    public WebDriver getDriver() { return driver; }
    public void setDriver(WebDriver driver) { this.driver = driver; }

    // -------------------------------------------------------------------
    // Pages
    // -------------------------------------------------------------------

    public LoginPage getLoginPage() { return loginPage; }
    public void setLoginPage(LoginPage loginPage) { this.loginPage = loginPage; }

    public DashboardPage getDashboardPage() { return dashboardPage; }
    public void setDashboardPage(DashboardPage dashboardPage) {
        this.dashboardPage = dashboardPage;
    }

    public ProductsPage getProductsPage() { return productsPage; }
    public void setProductsPage(ProductsPage productsPage) {
        this.productsPage = productsPage;
    }

    public CartPage getCartPage() { return cartPage; }
    public void setCartPage(CartPage cartPage) { this.cartPage = cartPage; }

    public CheckoutPage getCheckoutPage() { return checkoutPage; }
    public void setCheckoutPage(CheckoutPage checkoutPage) {
        this.checkoutPage = checkoutPage;
    }

    // -------------------------------------------------------------------
    // Scenario data
    // -------------------------------------------------------------------

    public String getCapturedOrderId() { return capturedOrderId; }
    public void setCapturedOrderId(String orderId) { this.capturedOrderId = orderId; }

    public String getCapturedConfirmationNumber() { return capturedConfirmationNumber; }
    public void setCapturedConfirmationNumber(String number) {
        this.capturedConfirmationNumber = number;
    }

    public String getCapturedErrorMessage() { return capturedErrorMessage; }
    public void setCapturedErrorMessage(String message) {
        this.capturedErrorMessage = message;
    }

    public double getCapturedOrderTotal() { return capturedOrderTotal; }
    public void setCapturedOrderTotal(double total) { this.capturedOrderTotal = total; }
}
```

---

## Hooks Using the World

The `Hooks` class also receives the World through constructor injection. It initializes the driver in `@Before` and stores it on the World so all step definitions can access it.

```java
package hooks;

import context.World;
import io.cucumber.java.After;
import io.cucumber.java.Before;
import io.cucumber.java.Scenario;
import io.github.bonigarcia.wdm.WebDriverManager;
import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;

public class Hooks {

    private final World world;

    // PicoContainer injects the shared World instance
    public Hooks(World world) {
        this.world = world;
    }

    @Before(order = 1)
    public void initDriver() {
        WebDriverManager.chromedriver().setup();
        ChromeOptions options = new ChromeOptions();
        options.addArguments("--start-maximized");
        WebDriver driver = new ChromeDriver(options);
        world.setDriver(driver);
    }

    @After(order = 2)
    public void captureScreenshotOnFailure(Scenario scenario) {
        if (scenario.isFailed() && world.getDriver() != null) {
            byte[] screenshot = ((TakesScreenshot) world.getDriver())
                .getScreenshotAs(OutputType.BYTES);
            scenario.attach(screenshot, "image/png", "Failure screenshot");
        }
    }

    @After(order = 1)
    public void quitDriver() {
        if (world.getDriver() != null) {
            world.getDriver().quit();
        }
    }
}
```

---

## Step Definition Classes Using the World

All step definition classes accept the World through constructor injection. PicoContainer creates one `World` instance per scenario and passes the same instance to every class that declares it.

```java
package stepdefinitions;

import context.World;
import io.cucumber.java.en.Given;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import org.testng.Assert;
import pages.DashboardPage;
import pages.LoginPage;

public class LoginSteps {

    private final World world;

    public LoginSteps(World world) {
        this.world = world;
    }

    @Given("the user is on the login page")
    public void theUserIsOnTheLoginPage() {
        world.getDriver().get(System.getProperty("app.base.url") + "/login");
        world.setLoginPage(new LoginPage(world.getDriver()));
    }

    @When("the user logs in with username {string} and password {string}")
    public void theUserLogsIn(String username, String password) {
        DashboardPage dashboardPage = world.getLoginPage().login(username, password);
        world.setDashboardPage(dashboardPage);
    }

    @Then("the login should fail with error {string}")
    public void theLoginShouldFailWithError(String expectedError) {
        String actualError = world.getLoginPage().getErrorMessage();
        world.setCapturedErrorMessage(actualError);
        Assert.assertEquals(actualError, expectedError);
    }
}
```

```java
package stepdefinitions;

import context.World;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import org.testng.Assert;
import pages.CartPage;

public class DashboardSteps {

    private final World world;

    public DashboardSteps(World world) {
        this.world = world;
    }

    @Then("the dashboard page should be displayed")
    public void theDashboardPageShouldBeDisplayed() {
        Assert.assertTrue(world.getDashboardPage().isLoaded(),
            "Dashboard did not load. URL: " + world.getDriver().getCurrentUrl());
    }

    @When("the user navigates to the cart from the dashboard")
    public void theUserNavigatesToCart() {
        CartPage cartPage = world.getDashboardPage()
            .getNavigationBar()
            .openCart();
        world.setCartPage(cartPage);
    }
}
```

```java
package stepdefinitions;

import context.World;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import org.testng.Assert;
import pages.CheckoutPage;

public class CartSteps {

    private final World world;

    public CartSteps(World world) {
        this.world = world;
    }

    @When("the user proceeds to checkout from the cart")
    public void theUserProceedsToCheckout() {
        CheckoutPage checkoutPage = world.getCartPage().proceedToCheckout();
        world.setCheckoutPage(checkoutPage);
        world.setCapturedOrderTotal(world.getCartPage().getOrderTotal());
    }

    @Then("the checkout page should show the correct total")
    public void checkoutPageShouldShowCorrectTotal() {
        double displayedTotal = world.getCheckoutPage().getDisplayedTotal();
        Assert.assertEquals(displayedTotal, world.getCapturedOrderTotal(), 0.01,
            "Checkout total does not match cart total");
    }
}
```

---

## Complete Multi-Step Scenario Flow

**Feature file:**

```gherkin
Feature: End-to-End Purchase Flow

  Scenario: Registered user completes a purchase
    Given the user is on the login page
    When the user logs in with username "buyer@example.com" and password "pass123"
    Then the dashboard page should be displayed
    When the user navigates to the cart from the dashboard
    And the user proceeds to checkout from the cart
    Then the checkout page should show the correct total
    When the user enters payment details for card "4111111111111111"
    And the user places the order
    Then the order confirmation number should be displayed
    And the confirmation number should be saved for reference
```

**CheckoutSteps.java:**

```java
public class CheckoutSteps {

    private final World world;

    public CheckoutSteps(World world) {
        this.world = world;
    }

    @When("the user enters payment details for card {string}")
    public void theUserEntersPaymentDetails(String cardNumber) {
        world.getCheckoutPage().enterCardNumber(cardNumber);
        world.getCheckoutPage().enterExpiry("12/27");
        world.getCheckoutPage().enterCvv("123");
    }

    @When("the user places the order")
    public void theUserPlacesTheOrder() {
        world.getCheckoutPage().clickPlaceOrder();
    }

    @Then("the order confirmation number should be displayed")
    public void confirmationNumberShouldBeDisplayed() {
        String confirmationNumber = world.getCheckoutPage()
            .getConfirmationNumber();
        Assert.assertNotNull(confirmationNumber, "Confirmation number was null");
        Assert.assertFalse(confirmationNumber.isEmpty(),
            "Confirmation number was empty");
    }

    @Then("the confirmation number should be saved for reference")
    public void confirmationNumberShouldBeSaved() {
        String confirmationNumber = world.getCheckoutPage()
            .getConfirmationNumber();
        world.setCapturedConfirmationNumber(confirmationNumber);
        System.out.println("Captured confirmation number: " + confirmationNumber);
    }
}
```

Each step definition class works with the same World instance. Page objects created in one class are available in all other classes through the World.

---

## World vs Static ThreadLocal Driver

In simpler frameworks, the WebDriver is managed through a static `ThreadLocal` in the `Hooks` class. The World pattern replaces that approach with a cleaner, dependency-injected solution.

| Aspect        | Static ThreadLocal                  | World with DI                              |
| ------------- | ----------------------------------- | ------------------------------------------ |
| Driver access | `Hooks.getDriver()`                 | `world.getDriver()`                        |
| State sharing | Only driver                         | Driver + pages + scenario data             |
| Thread safety | Via ThreadLocal                     | Via PicoContainer (one World per scenario) |
| Testability   | Harder to test hooks in isolation   | Easier — World can be mocked               |
| Coupling      | Step classes coupled to Hooks class | Step classes coupled only to World         |
| Setup         | Simpler — no DI library             | Requires cucumber-picocontainer            |

For small frameworks, the ThreadLocal approach is sufficient. For large frameworks with many step definition classes sharing complex state, the World pattern is cleaner and more maintainable.

---

## Alternative DI Containers

PicoContainer is the simplest option, but Cucumber-JVM also supports:

- **Spring** — via `cucumber-spring`. Suitable if the project already uses Spring and benefits from Spring's full DI capabilities and bean lifecycle management.
- **Guice** — via `cucumber-guice`. Suitable for projects using Google Guice.
- **CDI/Weld** — via `cucumber-cdi2`. Suitable for Jakarta EE environments.

For Selenium projects without an existing DI framework, PicoContainer is always the recommended choice — it has no configuration requirements and no learning curve beyond adding the dependency.

---

## Summary

The World Object pattern provides clean, dependency-injected state sharing across step definition classes in Cucumber-JVM. Using PicoContainer, each scenario receives its own World instance containing the WebDriver, page objects, and captured data. All step definition classes and the Hooks class declare the World as a constructor parameter and receive the same instance automatically. This eliminates static state, reduces coupling between classes, and makes multi-step, multi-class scenarios straightforward to implement and maintain.

---

## Related Topics

- StepDefinitions.md — Step definitions and state management
- Hooks.md — Integrating Hooks with the World using constructor injection
- IntroBDD.md — Project setup and glue package configuration
- CustomParameterTypes.md — Typed parameters that work with World-stored objects
