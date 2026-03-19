# Handling Dropdowns

## Overview

Dropdowns are among the most commonly tested UI components. They come in two fundamentally different forms: the native HTML `<select>` element, and custom UI dropdowns built with `<div>`, `<ul>`, or other elements. Each requires a different approach. This document covers both completely, along with multi-select, autocomplete, and date picker dropdowns.

---

## Native HTML Select Dropdown

The HTML `<select>` element is the simplest type of dropdown to automate. Selenium provides the `Select` wrapper class specifically for it.

### HTML Structure

```html
<select id="country" name="country">
  <option value="">-- Select Country --</option>
  <option value="IN">India</option>
  <option value="US">United States</option>
  <option value="UK">United Kingdom</option>
  <option value="AU">Australia</option>
</select>
```

### Select Class Setup

```java
import org.openqa.selenium.support.ui.Select;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.By;

WebElement dropdownElement = driver.findElement(By.id("country"));
Select select = new Select(dropdownElement);
```

### Selecting Options

```java
// By visible text — what the user sees in the dropdown
select.selectByVisibleText("India");
select.selectByVisibleText("United States");

// By value attribute — the HTML value attribute on <option>
select.selectByValue("IN");
select.selectByValue("US");

// By index — 0-based position in the list
select.selectByIndex(0); // "-- Select Country --" (the placeholder)
select.selectByIndex(1); // "India"
select.selectByIndex(2); // "United States"
```

### Reading Selected Value

```java
// Get the currently selected option
WebElement selected = select.getFirstSelectedOption();
System.out.println("Selected text:  " + selected.getText());
System.out.println("Selected value: " + selected.getAttribute("value"));

// Verify selected value in test assertion
String selectedCountry = select.getFirstSelectedOption().getText();
Assert.assertEquals(selectedCountry, "India");
```

### Getting All Options

```java
List<WebElement> allOptions = select.getOptions();
System.out.println("Total options: " + allOptions.size());

// Print all option texts
allOptions.forEach(option -> System.out.println(option.getText()));

// Collect all option texts to a list
List<String> optionTexts = allOptions.stream()
    .map(WebElement::getText)
    .collect(Collectors.toList());

// Verify a specific option exists
Assert.assertTrue(optionTexts.contains("India"), "India option not found in dropdown");
```

---

## Multi-Select Dropdown

```html
<select id="skills" multiple="multiple">
  <option value="java">Java</option>
  <option value="python">Python</option>
  <option value="selenium">Selenium</option>
  <option value="cucumber">Cucumber</option>
</select>
```

```java
WebElement multiElement = driver.findElement(By.id("skills"));
Select multiSelect = new Select(multiElement);

// Verify it supports multiple selection
boolean isMultiple = multiSelect.isMultiple();
Assert.assertTrue(isMultiple, "Skills dropdown should support multiple selection");

// Select multiple options
multiSelect.selectByVisibleText("Java");
multiSelect.selectByVisibleText("Selenium");
multiSelect.selectByVisibleText("Cucumber");

// Get all selected options
List<WebElement> selected = multiSelect.getAllSelectedOptions();
System.out.println("Selected count: " + selected.size());
selected.forEach(opt -> System.out.println("- " + opt.getText()));

// Deselect a specific option
multiSelect.deselectByVisibleText("Java");

// Deselect by value or index
multiSelect.deselectByValue("python");
multiSelect.deselectByIndex(0);

// Deselect ALL selections
multiSelect.deselectAll();
```

---

## Custom Dropdown (Div/Ul Based)

Most modern web applications use custom dropdowns built with `<div>`, `<ul>`, and `<li>` elements. These do not work with the `Select` class and require standard WebElement interaction.

### Typical HTML Structure

```html
<!-- Dropdown trigger button -->
<div class="dropdown">
  <button
    class="dropdown-toggle"
    id="country-btn"
    data-testid="country-dropdown"
  >
    Select Country
  </button>

  <!-- Options list (visible only when open) -->
  <ul class="dropdown-menu" id="country-options">
    <li data-value="IN">India</li>
    <li data-value="US">United States</li>
    <li data-value="UK">United Kingdom</li>
  </ul>
</div>
```

### Basic Custom Dropdown Interaction

```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// Step 1: Click to open the dropdown
driver.findElement(By.id("country-btn")).click();

// Step 2: Wait for options to be visible
wait.until(ExpectedConditions.visibilityOfElementLocated(
    By.id("country-options")
));

// Step 3: Click the desired option by text
driver.findElement(By.xpath(
    "//ul[@id='country-options']//li[normalize-space(text())='India']"
)).click();

// Step 4: Verify the selection
String selectedText = driver.findElement(By.id("country-btn")).getText();
Assert.assertTrue(selectedText.contains("India"));
```

---

## Custom Dropdown Helper Class

A reusable, framework-level helper for custom dropdowns:

```java
package com.yourcompany.automation.utils;

import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.*;
import java.time.Duration;
import java.util.List;
import java.util.stream.Collectors;

public class DropdownHelper {

    private final WebDriver driver;
    private final WebDriverWait wait;

    public DropdownHelper(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    // ========================
    // NATIVE SELECT METHODS
    // ========================

    public void selectByText(By locator, String visibleText) {
        Select select = new Select(wait.until(
            ExpectedConditions.visibilityOfElementLocated(locator)
        ));
        select.selectByVisibleText(visibleText);
    }

    public void selectByValue(By locator, String value) {
        Select select = new Select(wait.until(
            ExpectedConditions.visibilityOfElementLocated(locator)
        ));
        select.selectByValue(value);
    }

    public String getSelectedText(By locator) {
        Select select = new Select(driver.findElement(locator));
        return select.getFirstSelectedOption().getText();
    }

    public List<String> getAllOptionTexts(By locator) {
        Select select = new Select(driver.findElement(locator));
        return select.getOptions().stream()
            .map(WebElement::getText)
            .collect(Collectors.toList());
    }

    // ========================
    // CUSTOM DROPDOWN METHODS
    // ========================

    /**
     * Click a custom dropdown trigger, wait for options, and select by text.
     */
    public void selectFromCustomDropdown(By trigger, By optionsContainer, String optionText) {
        // Open dropdown
        wait.until(ExpectedConditions.elementToBeClickable(trigger)).click();

        // Wait for options to appear
        wait.until(ExpectedConditions.visibilityOfElementLocated(optionsContainer));

        // Find and click the option
        By optionLocator = By.xpath(
            "//*[normalize-space(text())='" + optionText + "']"
        );

        // Scope to options container for precision
        WebElement container = driver.findElement(optionsContainer);
        WebElement option = wait.until(
            d -> container.findElements(By.tagName("li")).stream()
                .filter(li -> li.getText().trim().equals(optionText))
                .findFirst()
                .orElse(null)
        );

        if (option == null) {
            throw new NoSuchElementException(
                "Option '" + optionText + "' not found in dropdown"
            );
        }

        option.click();
    }

    /**
     * Select from custom dropdown by partial text match.
     */
    public void selectFromCustomDropdownByPartialText(
            By trigger, By optionItems, String partialText) {

        wait.until(ExpectedConditions.elementToBeClickable(trigger)).click();
        wait.until(ExpectedConditions.visibilityOfElementLocated(optionItems));

        List<WebElement> options = driver.findElements(optionItems);
        for (WebElement option : options) {
            if (option.getText().contains(partialText)) {
                option.click();
                return;
            }
        }
        throw new NoSuchElementException(
            "No option containing '" + partialText + "' found"
        );
    }

    /**
     * Get all visible option texts from an open custom dropdown.
     */
    public List<String> getCustomDropdownOptions(By trigger, By optionItems) {
        wait.until(ExpectedConditions.elementToBeClickable(trigger)).click();
        wait.until(ExpectedConditions.visibilityOfElementLocated(optionItems));

        return driver.findElements(optionItems).stream()
            .map(el -> el.getText().trim())
            .filter(text -> !text.isEmpty())
            .collect(Collectors.toList());
    }

    /**
     * Close custom dropdown by pressing Escape.
     */
    public void closeDropdown() {
        driver.findElement(By.tagName("body")).sendKeys(Keys.ESCAPE);
    }
}
```

---

## Autocomplete / Typeahead Dropdown

Autocomplete inputs show suggestions as you type (e.g., Google search, city pickers):

```html
<input id="city-search" type="text" placeholder="Search for city..." />
<ul id="autocomplete-results" class="suggestions">
  <li>Mumbai</li>
  <li>Mumbai Suburban</li>
  <li>Mumbai City</li>
</ul>
```

```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// Step 1: Click and type in the autocomplete input
WebElement searchInput = driver.findElement(By.id("city-search"));
searchInput.click();
searchInput.clear();
searchInput.sendKeys("Mum");

// Step 2: Wait for suggestions to appear
wait.until(ExpectedConditions.visibilityOfElementLocated(
    By.id("autocomplete-results")
));

// Step 3: Wait for the specific option to appear and click it
List<WebElement> suggestions = wait.until(
    ExpectedConditions.visibilityOfAllElementsLocatedBy(
        By.cssSelector("#autocomplete-results li")
    )
);

// Click the first suggestion
suggestions.get(0).click();

// OR click a specific suggestion by text
suggestions.stream()
    .filter(s -> s.getText().equals("Mumbai"))
    .findFirst()
    .ifPresent(WebElement::click);

// Step 4: Verify the selection
String selectedCity = searchInput.getAttribute("value");
Assert.assertEquals(selectedCity, "Mumbai");
```

### Autocomplete with API Delay

```java
// For autocomplete that makes an API call before showing results
WebElement searchInput = driver.findElement(By.id("city-search"));
searchInput.sendKeys("Bang");

// Wait for results to load (spinner disappears and results appear)
wait.until(ExpectedConditions.invisibilityOfElementLocated(
    By.cssSelector(".autocomplete-loading")
));
wait.until(ExpectedConditions.visibilityOfElementLocated(
    By.cssSelector(".autocomplete-option")
));

// Click the matching option
driver.findElement(By.xpath(
    "//li[contains(@class,'autocomplete-option') and text()='Bangalore']"
)).click();
```

---

## React Select Dropdown

React Select is an extremely popular React library. Its HTML structure changes frequently between versions — here is how to handle it robustly:

```html
<!-- React Select typical structure -->
<div class="react-select__control">
  <div class="react-select__value-container">
    <div class="react-select__placeholder">Select...</div>
  </div>
  <div class="react-select__indicators">...</div>
</div>
```

```java
// Click the React Select control to open the dropdown
driver.findElement(By.cssSelector(".react-select__control")).click();

WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// Wait for the menu to open
wait.until(ExpectedConditions.visibilityOfElementLocated(
    By.cssSelector(".react-select__menu")
));

// Click the desired option
driver.findElement(By.xpath(
    "//div[contains(@class,'react-select__option') and normalize-space(text())='India']"
)).click();

// Verify selection (value appears in the control)
String selectedValue = driver.findElement(
    By.cssSelector(".react-select__single-value")
).getText();
Assert.assertEquals(selectedValue, "India");

// For React Select with search/filter
WebElement selectInput = driver.findElement(
    By.cssSelector(".react-select__input input")
);
selectInput.sendKeys("Ind");

// Wait for filtered results
wait.until(ExpectedConditions.visibilityOfElementLocated(
    By.cssSelector(".react-select__option")
));
driver.findElement(By.xpath(
    "//div[contains(@class,'react-select__option') and text()='India']"
)).click();
```

---

## Material UI Select (Angular/React Material)

```java
// Open Material dropdown
driver.findElement(By.cssSelector("mat-select[formcontrolname='country']")).click();

WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// Wait for panel to open (rendered in an overlay, outside the form)
wait.until(ExpectedConditions.visibilityOfElementLocated(
    By.cssSelector(".mat-select-panel")
));

// Click the option in the panel
driver.findElement(By.xpath(
    "//mat-option[.//span[normalize-space(text())='India']]"
)).click();

// Verify
String selected = driver.findElement(
    By.cssSelector("mat-select[formcontrolname='country'] .mat-select-value-text span")
).getText();
Assert.assertEquals(selected, "India");
```

---

## Date Picker Dropdown

Date pickers are a specific type of custom dropdown:

```java
// Strategy 1: Type directly into date input (simplest)
WebElement dateInput = driver.findElement(By.id("booking-date"));
dateInput.clear();
dateInput.sendKeys("12/25/2024"); // Format depends on locale
dateInput.sendKeys(Keys.TAB);    // Confirm input

// Strategy 2: Use JavaScript to set value (for read-only date inputs)
WebElement dateInput = driver.findElement(By.id("booking-date"));
((JavascriptExecutor) driver).executeScript(
    "arguments[0].value = '2024-12-25';", dateInput
);
// Trigger change event so Angular/React picks up the new value
((JavascriptExecutor) driver).executeScript(
    "arguments[0].dispatchEvent(new Event('change'));", dateInput
);

// Strategy 3: Calendar widget navigation
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// Open the calendar
driver.findElement(By.id("date-picker-btn")).click();

// Navigate to the correct month/year
// (click next/previous month arrow until correct month is shown)
String targetMonth = "December 2024";
while (!driver.findElement(By.cssSelector(".calendar-header")).getText()
        .equals(targetMonth)) {
    driver.findElement(By.cssSelector(".next-month-arrow")).click();
    wait.until(ExpectedConditions.textToBePresentInElementLocated(
        By.cssSelector(".calendar-header"), "2024"
    ));
}

// Click the target date
driver.findElement(By.xpath(
    "//td[contains(@class,'calendar-day') and text()='25']"
)).click();

// Verify date was set
String selectedDate = driver.findElement(By.id("booking-date")).getAttribute("value");
Assert.assertTrue(selectedDate.contains("25"));
```

---

## Complete Select Helper for Page Objects

```java
package com.yourcompany.automation.pages.components;

import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.*;

public class SelectComponent {

    private final WebElement element;

    public SelectComponent(WebElement element) {
        this.element = element;
    }

    /**
     * Returns true if this is a native HTML select element.
     */
    public boolean isNativeSelect() {
        return element.getTagName().equalsIgnoreCase("select");
    }

    /**
     * Selects by visible text — works for both native and simple custom selects.
     */
    public SelectComponent select(String optionText) {
        if (isNativeSelect()) {
            new Select(element).selectByVisibleText(optionText);
        } else {
            element.click();
            // Assumes options are children — override in subclass for complex dropdowns
            element.findElements(By.tagName("li")).stream()
                .filter(li -> li.getText().trim().equals(optionText))
                .findFirst()
                .orElseThrow(() -> new NoSuchElementException(
                    "Option '" + optionText + "' not found"
                ))
                .click();
        }
        return this;
    }

    /**
     * Get the currently selected visible text.
     */
    public String getSelected() {
        if (isNativeSelect()) {
            return new Select(element).getFirstSelectedOption().getText();
        }
        return element.getText();
    }
}
```

---

## Summary

Native HTML `<select>` dropdowns are handled elegantly by Selenium's `Select` class — use `selectByVisibleText()` for readability. Custom dropdowns require opening the trigger, waiting for options to appear, and clicking the target option using CSS or XPath. Build a centralized `DropdownHelper` so all dropdown interactions in your Page Objects are consistent. For framework-specific dropdowns (React Select, Material UI), use their specific CSS class patterns and always wait for the menu panel to be visible before clicking options.
