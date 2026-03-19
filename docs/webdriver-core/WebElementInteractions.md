# WebElement Interactions

## Overview

WebElement interactions are the core of what automation does — clicking buttons, typing text, reading values, selecting options, uploading files, and verifying element states. Every test step that touches the UI goes through `WebElement`. This document is a complete, deeply detailed reference of every WebElement interaction method, with professional patterns and real-world examples.

---

## Obtaining a WebElement

```java
import org.openqa.selenium.By;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;
import java.time.Duration;

WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// Direct find (no wait — use only if element is guaranteed present)
WebElement btn = driver.findElement(By.id("submit"));

// With explicit wait (recommended)
WebElement btn = wait.until(ExpectedConditions.elementToBeClickable(By.id("submit")));

// Scoped search — search within a parent element
WebElement form = driver.findElement(By.id("checkout-form"));
WebElement emailInput = form.findElement(By.name("email"));
```

---

## Text Input

### sendKeys() — Typing Text

```java
WebElement nameInput = driver.findElement(By.id("full-name"));

// Always clear before typing to avoid appending to existing content
nameInput.clear();
nameInput.sendKeys("John Doe");

// Chain multiple sendKeys
WebElement searchBox = driver.findElement(By.id("search"));
searchBox.clear();
searchBox.sendKeys("selenium automation");
searchBox.sendKeys(Keys.ENTER); // Press Enter to submit

// Type and then Tab to the next field
WebElement emailField = driver.findElement(By.id("email"));
emailField.clear();
emailField.sendKeys("user@example.com");
emailField.sendKeys(Keys.TAB); // Move focus to next field

// Simulate Ctrl+A (select all) then type (replaces all content)
WebElement textArea = driver.findElement(By.id("description"));
textArea.sendKeys(Keys.chord(Keys.CONTROL, "a"));
textArea.sendKeys("Completely new content");
```

### Keys Enum — Keyboard Simulation

```java
import org.openqa.selenium.Keys;

// Navigation keys
element.sendKeys(Keys.TAB);
element.sendKeys(Keys.ENTER);
element.sendKeys(Keys.RETURN);
element.sendKeys(Keys.SPACE);
element.sendKeys(Keys.BACK_SPACE);
element.sendKeys(Keys.DELETE);
element.sendKeys(Keys.ESCAPE);

// Arrow keys
element.sendKeys(Keys.ARROW_UP);
element.sendKeys(Keys.ARROW_DOWN);
element.sendKeys(Keys.ARROW_LEFT);
element.sendKeys(Keys.ARROW_RIGHT);

// Function keys
element.sendKeys(Keys.F1);
element.sendKeys(Keys.F5);
element.sendKeys(Keys.F12);

// Modifier key combinations
element.sendKeys(Keys.chord(Keys.CONTROL, "a")); // Ctrl+A (select all)
element.sendKeys(Keys.chord(Keys.CONTROL, "c")); // Ctrl+C (copy)
element.sendKeys(Keys.chord(Keys.CONTROL, "v")); // Ctrl+V (paste)
element.sendKeys(Keys.chord(Keys.CONTROL, "z")); // Ctrl+Z (undo)
element.sendKeys(Keys.chord(Keys.SHIFT, Keys.HOME)); // Shift+Home (select to beginning)

// Mac users: use Keys.COMMAND instead of Keys.CONTROL
element.sendKeys(Keys.chord(Keys.COMMAND, "a")); // Cmd+A on Mac
```

### Clearing Input Fields

```java
// Standard clear (works for most inputs)
element.clear();

// Clear via keyboard shortcut (for stubborn fields)
element.sendKeys(Keys.chord(Keys.CONTROL, "a"));
element.sendKeys(Keys.DELETE);

// Clear via JavaScript (for readonly or custom inputs)
((JavascriptExecutor) driver).executeScript("arguments[0].value = '';", element);
```

---

## Clicking

### Standard Click

```java
// Basic click
driver.findElement(By.id("submit-btn")).click();

// Always wait for clickability before clicking
WebElement button = wait.until(
    ExpectedConditions.elementToBeClickable(By.id("submit-btn"))
);
button.click();
```

### When Regular Click Fails

```java
import org.openqa.selenium.JavascriptExecutor;
import org.openqa.selenium.interactions.Actions;

// Option 1: JavaScript click (bypasses visibility/interactability checks)
// Use when element is covered by an overlay or click is intercepted
WebElement btn = driver.findElement(By.id("covered-button"));
((JavascriptExecutor) driver).executeScript("arguments[0].click();", btn);

// Option 2: Actions class click (more realistic — moves to element first)
new Actions(driver).moveToElement(btn).click().perform();

// Option 3: Scroll into view, then click
((JavascriptExecutor) driver).executeScript(
    "arguments[0].scrollIntoView({behavior:'smooth', block:'center'});", btn
);
wait.until(ExpectedConditions.elementToBeClickable(btn));
btn.click();
```

### Double Click and Right Click

```java
Actions actions = new Actions(driver);

// Double click
WebElement editableCell = driver.findElement(By.id("editable-cell"));
actions.doubleClick(editableCell).perform();

// Right click (context menu)
WebElement fileIcon = driver.findElement(By.id("file-icon"));
actions.contextClick(fileIcon).perform();

// Click and hold (for drag operations)
actions.clickAndHold(driver.findElement(By.id("drag-handle"))).perform();
```

---

## Checkbox and Radio Button Interactions

```java
WebElement checkbox = driver.findElement(By.id("agree-terms"));

// Check the checkbox (only if not already checked)
if (!checkbox.isSelected()) {
    checkbox.click();
}

// Uncheck (only if currently checked)
if (checkbox.isSelected()) {
    checkbox.click();
}

// Verify state
boolean isChecked = checkbox.isSelected();
Assert.assertTrue(isChecked, "Terms checkbox should be checked");

// Radio buttons — select by value
List<WebElement> radioButtons = driver.findElements(
    By.cssSelector("input[name='gender']")
);
for (WebElement radio : radioButtons) {
    if (radio.getAttribute("value").equals("male")) {
        radio.click();
        break;
    }
}

// Or directly by attribute
driver.findElement(By.cssSelector("input[name='gender'][value='female']")).click();
```

---

## Dropdown — Select Element

The `Select` class is the standard way to interact with `<select>` HTML elements:

```java
import org.openqa.selenium.support.ui.Select;

WebElement dropdownElement = driver.findElement(By.id("country-select"));
Select countryDropdown = new Select(dropdownElement);

// ===== SELECTING OPTIONS =====

// By visible text (the text the user sees)
countryDropdown.selectByVisibleText("India");
countryDropdown.selectByVisibleText("United States");

// By value attribute
countryDropdown.selectByValue("IN");
countryDropdown.selectByValue("US");

// By index (0-based)
countryDropdown.selectByIndex(0); // First option
countryDropdown.selectByIndex(2); // Third option

// ===== READING SELECTED OPTION =====

// Get the currently selected option (single-select)
WebElement selectedOption = countryDropdown.getFirstSelectedOption();
System.out.println("Selected: " + selectedOption.getText());
System.out.println("Value:    " + selectedOption.getAttribute("value"));

// Get all selected options (multi-select)
List<WebElement> selectedOptions = countryDropdown.getAllSelectedOptions();
selectedOptions.forEach(opt -> System.out.println(opt.getText()));

// ===== GETTING ALL OPTIONS =====

List<WebElement> allOptions = countryDropdown.getOptions();
allOptions.forEach(opt -> System.out.println(opt.getText() + " = " + opt.getAttribute("value")));
System.out.println("Total options: " + allOptions.size());

// ===== MULTI-SELECT DROPDOWNS =====

WebElement multiSelect = driver.findElement(By.id("skills"));
Select skillsSelect = new Select(multiSelect);

// Verify it supports multiple selection
if (skillsSelect.isMultiple()) {
    skillsSelect.selectByVisibleText("Java");
    skillsSelect.selectByVisibleText("Selenium");
    skillsSelect.selectByVisibleText("Cucumber");

    // Deselect options
    skillsSelect.deselectByVisibleText("Java");
    skillsSelect.deselectAll(); // Clear all selections
}
```

---

## Custom Dropdown (Non-Select Elements)

Many modern apps use `<div>`-based custom dropdowns instead of `<select>`:

```java
public class CustomDropdownHelper {

    private final WebDriver driver;
    private final WebDriverWait wait;

    public CustomDropdownHelper(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    /**
     * Select option from a custom dropdown (div/ul based).
     *
     * @param dropdownTrigger Locator for the dropdown trigger/button
     * @param optionText      Visible text of the option to select
     */
    public void selectOption(By dropdownTrigger, String optionText) {
        // Click to open the dropdown
        wait.until(ExpectedConditions.elementToBeClickable(dropdownTrigger)).click();

        // Wait for options to appear and select by text
        By optionLocator = By.xpath(
            "//ul[contains(@class,'dropdown-menu')]//li[normalize-space(text())='" + optionText + "']"
        );
        wait.until(ExpectedConditions.elementToBeClickable(optionLocator)).click();
    }

    /**
     * Get selected value from a custom dropdown.
     */
    public String getSelectedValue(By dropdownTrigger) {
        return driver.findElement(dropdownTrigger).getText().trim();
    }

    /**
     * Get all available options from an open dropdown.
     */
    public List<String> getAllOptions(By optionsContainer) {
        return driver.findElements(optionsContainer)
            .stream()
            .map(WebElement::getText)
            .collect(java.util.stream.Collectors.toList());
    }
}

// Usage
CustomDropdownHelper dropdown = new CustomDropdownHelper(driver);
dropdown.selectOption(By.id("country-trigger"), "India");
String selected = dropdown.getSelectedValue(By.id("country-trigger"));
Assert.assertEquals(selected, "India");
```

---

## File Upload

```java
// Standard HTML file input — sendKeys with absolute file path
WebElement fileInput = driver.findElement(By.cssSelector("input[type='file']"));
String filePath = System.getProperty("user.dir") + "/src/test/resources/testdata/sample.pdf";
fileInput.sendKeys(filePath);

// For hidden file inputs (style="display:none")
WebElement hiddenFileInput = driver.findElement(By.id("file-upload-input"));
// Remove hidden/display:none via JS to make it interactable
((JavascriptExecutor) driver).executeScript(
    "arguments[0].style.display = 'block'; arguments[0].style.visibility = 'visible';",
    hiddenFileInput
);
hiddenFileInput.sendKeys(filePath);

// Multiple file upload
WebElement multiFileInput = driver.findElement(By.id("multi-file-input"));
multiFileInput.sendKeys(
    System.getProperty("user.dir") + "/testdata/file1.pdf" + "\n" +
    System.getProperty("user.dir") + "/testdata/file2.pdf"
);

// Verify file was selected
String uploadedFileName = fileInput.getAttribute("value");
Assert.assertTrue(uploadedFileName.contains("sample.pdf"));
```

---

## Reading Element Information

### Text and Attributes

```java
WebElement element = driver.findElement(By.id("product-name"));

// Get visible text content (what user sees on screen)
String visibleText = element.getText();

// Get HTML attribute value
String id          = element.getAttribute("id");
String className   = element.getAttribute("class");
String href        = element.getAttribute("href");
String src         = element.getAttribute("src");
String type        = element.getAttribute("type");
String value       = element.getAttribute("value");    // For inputs
String placeholder = element.getAttribute("placeholder");
String dataId      = element.getAttribute("data-id");  // Custom data attributes
String disabled    = element.getAttribute("disabled"); // null if not disabled; "true" if disabled

// Get DOM property (slightly different from attribute in some cases)
String inputValue  = (String) ((JavascriptExecutor) driver)
    .executeScript("return arguments[0].value;", element);

// Get inner HTML
String innerHTML   = (String) ((JavascriptExecutor) driver)
    .executeScript("return arguments[0].innerHTML;", element);

// Get CSS property value
String color       = element.getCssValue("color");
String background  = element.getCssValue("background-color");
String fontSize    = element.getCssValue("font-size");
String fontWeight  = element.getCssValue("font-weight");
String display     = element.getCssValue("display");
String visibility  = element.getCssValue("visibility");
String opacity     = element.getCssValue("opacity");
```

### Element State

```java
WebElement input    = driver.findElement(By.id("username"));
WebElement button   = driver.findElement(By.id("submit"));
WebElement checkbox = driver.findElement(By.id("agree"));
WebElement hidden   = driver.findElement(By.id("hidden-panel"));

// Is the element rendered and visible on screen?
boolean isDisplayed = input.isDisplayed();

// Is the element enabled (interactive)?
boolean isEnabled   = button.isEnabled();

// Is the element checked/selected? (checkbox, radio, <option>)
boolean isSelected  = checkbox.isSelected();

// Check if element exists without throwing (returns empty list)
boolean exists = !driver.findElements(By.id("error-banner")).isEmpty();

// Combined readiness check
boolean isReady = input.isDisplayed() && input.isEnabled();
```

### Element Dimensions and Position

```java
WebElement card = driver.findElement(By.className("product-card"));

// Width and height in pixels
Dimension size = card.getSize();
System.out.println("Width: "  + size.getWidth());
System.out.println("Height: " + size.getHeight());

// Position (X, Y from top-left corner of viewport)
Point location = card.getLocation();
System.out.println("X: " + location.getX());
System.out.println("Y: " + location.getY());

// Combined — Rectangle includes position + size
Rectangle rect = card.getRect();
System.out.println("X: " + rect.getX() + ", Y: " + rect.getY());
System.out.println("Width: " + rect.getWidth() + ", Height: " + rect.getHeight());
```

---

## Element Screenshot (Selenium 4)

```java
import org.openqa.selenium.OutputType;
import java.io.File;
import org.apache.commons.io.FileUtils;

// Capture a screenshot of just this element (crops automatically)
WebElement errorBanner = driver.findElement(By.id("error-banner"));
File elementScreenshot = errorBanner.getScreenshotAs(OutputType.FILE);
FileUtils.copyFile(elementScreenshot, new File("screenshots/error_banner.png"));

// As bytes (for test report attachment)
byte[] elementBytes = errorBanner.getScreenshotAs(OutputType.BYTES);
```

---

## Submit vs Click

```java
// submit() — triggers form's submit event (works on any element inside a <form>)
driver.findElement(By.id("username")).submit();

// click() on submit button — explicitly clicks the button
driver.findElement(By.cssSelector("button[type='submit']")).click();
```

> **Recommendation:** Use `click()` on the submit button for clarity. `submit()` can behave unexpectedly with custom form validation JavaScript.

---

## Interacting with Tables

```java
// Get all rows in a table body
WebElement table = driver.findElement(By.id("data-table"));
List<WebElement> rows = table.findElements(By.cssSelector("tbody tr"));

System.out.println("Total rows: " + rows.size());

// Read specific cell value from a row
for (WebElement row : rows) {
    List<WebElement> cells = row.findElements(By.tagName("td"));
    String id     = cells.get(0).getText();
    String name   = cells.get(1).getText();
    String status = cells.get(2).getText();
    System.out.printf("%-10s %-20s %-10s%n", id, name, status);
}

// Find a row by matching a cell value
WebElement targetRow = rows.stream()
    .filter(row -> row.findElement(By.cssSelector("td:first-child"))
        .getText().equals("ORD-12345"))
    .findFirst()
    .orElseThrow(() -> new NoSuchElementException("Order ORD-12345 not found in table"));

// Click a button in that specific row
targetRow.findElement(By.cssSelector("button.view-btn")).click();

// Get total number of pages from pagination
int totalPages = Integer.parseInt(
    driver.findElement(By.cssSelector(".pagination .total-pages")).getText()
);
```

---

## WebElement Interaction Best Practices

| Practice                                                | Why It Matters                                            |
| ------------------------------------------------------- | --------------------------------------------------------- |
| Always `clear()` before `sendKeys()`                    | Prevents appending to existing content                    |
| Always wait for clickability before clicking            | Avoids `ElementNotInteractableException`                  |
| Use scoped `findElement()` on a parent                  | Faster and avoids false matches on duplicate elements     |
| Read `getText()` only after element is visible          | Text may be empty until element is rendered               |
| Use `getAttribute("value")` for inputs, not `getText()` | Input text is in the `value` attribute, not `textContent` |
| Never rely on element index (e.g., `(//button)[3]`)     | Breaks when UI changes; use attributes instead            |
| Use `isDisplayed()` before reading text or value        | Avoids reading from hidden/stale elements                 |

---

## Summary

WebElement interactions are the actions layer of your tests. Use `click()` for buttons and links, `sendKeys()` with `clear()` for inputs, `Select` for native dropdowns, and custom helpers for modern UI components. Always pair interactions with explicit waits, and read element states with `isDisplayed()`, `isEnabled()`, and `isSelected()` to build robust, flake-resistant test steps.
