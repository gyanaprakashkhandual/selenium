# Actions Class

## Overview

The `Actions` class in Selenium WebDriver provides an API for simulating complex user input that the standard `WebElement.click()` and `sendKeys()` methods cannot replicate. It models a user who can move a mouse precisely, hold modifier keys, perform multi-step gestures, right-click, double-click, drag elements, and type keyboard shortcuts — all as a composed sequence of individual low-level actions.

`Actions` is part of the `org.openqa.selenium.interactions` package and is available in all Selenium 4 WebDriver implementations.

---

## How Actions Works

`Actions` builds a sequence of input actions that are dispatched to the browser as a single atomic operation. You chain method calls on the `Actions` builder, then call `perform()` to execute the entire sequence.

```java
Actions actions = new Actions(driver);

actions
    .moveToElement(targetElement)   // step 1: move mouse
    .pause(Duration.ofMillis(200))  // step 2: brief pause
    .click()                        // step 3: click
    .perform();                     // dispatch the sequence
```

### W3C Actions Model (Selenium 4)

Selenium 4 uses the W3C Actions specification. It models three input sources:

- **Pointer** — mouse, touch, or pen movement and clicks
- **Key** — keyboard key press and release
- **Wheel** — scroll wheel actions

All methods on the `Actions` class translate to one or more of these low-level input events.

---

## Import and Setup

```java
import org.openqa.selenium.interactions.Actions;
import org.openqa.selenium.Keys;
import java.time.Duration;

// In your page class or step definition
Actions actions = new Actions(driver);
```

---

## Mouse Actions

### Hover (Move to Element)

Hovering reveals dropdown menus, tooltips, and pop-over content that only appear on mouse-over.

```java
// Basic hover
public void hoverOver(WebElement element) {
    new Actions(driver)
        .moveToElement(element)
        .perform();
}

// Hover then click a revealed sub-menu item
public void hoverAndClick(WebElement menuItem, WebElement subMenuItem) {
    new Actions(driver)
        .moveToElement(menuItem)
        .pause(Duration.ofMillis(500))   // wait for sub-menu animation
        .moveToElement(subMenuItem)
        .click()
        .perform();
}
```

**Feature file example:**

```gherkin
When the user hovers over the "Products" menu
Then the sub-menu should appear
When the user clicks "Electronics" from the sub-menu
Then the Electronics page should be displayed
```

### Right-Click (Context Menu)

```java
public void rightClick(WebElement element) {
    new Actions(driver)
        .contextClick(element)
        .perform();
}

// Right-click and select from context menu
public void rightClickAndSelect(WebElement element, WebElement menuOption) {
    new Actions(driver)
        .contextClick(element)
        .pause(Duration.ofMillis(300))
        .click(menuOption)
        .perform();
}
```

### Double-Click

```java
public void doubleClick(WebElement element) {
    new Actions(driver)
        .doubleClick(element)
        .perform();
}
```

Common use cases: opening a file/folder in a file browser, triggering inline edit mode on a table cell, selecting a word in a text editor.

### Click and Hold

```java
public void clickAndHold(WebElement element) {
    new Actions(driver)
        .clickAndHold(element)
        .perform();
}

public void release(WebElement element) {
    new Actions(driver)
        .release(element)
        .perform();
}
```

### Move to Offset

Move the mouse to a precise pixel position relative to an element's top-left corner:

```java
public void clickAtOffset(WebElement element, int xOffset, int yOffset) {
    new Actions(driver)
        .moveToElement(element, xOffset, yOffset)
        .click()
        .perform();
}
```

Move to an absolute position on the page:

```java
public void moveToCoordinates(int x, int y) {
    new Actions(driver)
        .moveByOffset(x, y)
        .perform();
}
```

---

## Keyboard Actions

### Key Chord — Modifier + Key

```java
// Select all text: Ctrl+A
public void selectAll() {
    new Actions(driver)
        .keyDown(Keys.CONTROL)
        .sendKeys("a")
        .keyUp(Keys.CONTROL)
        .perform();
}

// Copy: Ctrl+C
public void copy() {
    new Actions(driver)
        .keyDown(Keys.CONTROL)
        .sendKeys("c")
        .keyUp(Keys.CONTROL)
        .perform();
}

// Paste: Ctrl+V
public void paste(WebElement target) {
    new Actions(driver)
        .click(target)
        .keyDown(Keys.CONTROL)
        .sendKeys("v")
        .keyUp(Keys.CONTROL)
        .perform();
}

// Undo: Ctrl+Z
public void undo() {
    new Actions(driver)
        .keyDown(Keys.CONTROL)
        .sendKeys("z")
        .keyUp(Keys.CONTROL)
        .perform();
}

// On macOS use Keys.COMMAND instead of Keys.CONTROL
public void selectAllMac() {
    new Actions(driver)
        .keyDown(Keys.COMMAND)
        .sendKeys("a")
        .keyUp(Keys.COMMAND)
        .perform();
}
```

### Multi-Select with Ctrl+Click

```java
// Select multiple items in a list
public void multiSelectItems(List<WebElement> items) {
    Actions actions = new Actions(driver);
    actions.click(items.get(0));  // click first without modifier
    for (int i = 1; i < items.size(); i++) {
        actions.keyDown(Keys.CONTROL)
               .click(items.get(i))
               .keyUp(Keys.CONTROL);
    }
    actions.perform();
}
```

### Shift+Click (Range Select)

```java
public void shiftClick(WebElement first, WebElement last) {
    new Actions(driver)
        .click(first)
        .keyDown(Keys.SHIFT)
        .click(last)
        .keyUp(Keys.SHIFT)
        .perform();
}
```

### Sending Keys to Focused Element

```java
// Press Tab to move focus
public void pressTab() {
    new Actions(driver)
        .sendKeys(Keys.TAB)
        .perform();
}

// Press Enter to submit
public void pressEnter() {
    new Actions(driver)
        .sendKeys(Keys.ENTER)
        .perform();
}

// Press Escape to close modal or cancel
public void pressEscape() {
    new Actions(driver)
        .sendKeys(Keys.ESCAPE)
        .perform();
}

// Arrow key navigation in date pickers, sliders, select boxes
public void pressArrowDown(int times) {
    Actions actions = new Actions(driver);
    for (int i = 0; i < times; i++) {
        actions.sendKeys(Keys.ARROW_DOWN);
    }
    actions.perform();
}
```

### Typing into a Specific Element

```java
public void typeInElement(WebElement element, CharSequence text) {
    new Actions(driver)
        .click(element)
        .sendKeys(text)
        .perform();
}
```

---

## Drag and Drop

See `DragDrop.md` for the full dedicated treatment. This section covers the fundamental Actions API for dragging.

### dragAndDrop Method

```java
public void dragAndDrop(WebElement source, WebElement target) {
    new Actions(driver)
        .dragAndDrop(source, target)
        .perform();
}
```

### Manual Drag Sequence (More Reliable for Complex UIs)

```java
public void dragAndDropManual(WebElement source, WebElement target) {
    new Actions(driver)
        .moveToElement(source)
        .pause(Duration.ofMillis(300))
        .clickAndHold(source)
        .pause(Duration.ofMillis(300))
        .moveToElement(target)
        .pause(Duration.ofMillis(300))
        .release()
        .perform();
}
```

### Drag to Offset

```java
public void dragToOffset(WebElement source, int xOffset, int yOffset) {
    new Actions(driver)
        .dragAndDropBy(source, xOffset, yOffset)
        .perform();
}
```

---

## Scroll Actions (Selenium 4.2+)

Selenium 4 introduced dedicated scroll actions that are more reliable than JavaScript scroll for viewport-based interactions.

### Scroll to Element

```java
public void scrollToElement(WebElement element) {
    new Actions(driver)
        .scrollToElement(element)
        .perform();
}
```

### Scroll by Amount from Element

```java
// Scroll inside a scrollable container
public void scrollInsideElement(WebElement scrollContainer, int deltaX, int deltaY) {
    new Actions(driver)
        .scrollFromOrigin(
            WheelInput.ScrollOrigin.fromElement(scrollContainer),
            deltaX,
            deltaY
        )
        .perform();
}
```

### Scroll by Viewport Delta

```java
public void scrollByViewport(int deltaX, int deltaY) {
    new Actions(driver)
        .scrollByAmount(deltaX, deltaY)
        .perform();
}
```

---

## Pause and Timing

The `pause()` method introduces a deliberate wait between actions. Use it for animations, transitions, and menu reveals that need a moment before the next step.

```java
new Actions(driver)
    .moveToElement(menuTrigger)
    .pause(Duration.ofMillis(500))   // wait for dropdown animation
    .moveToElement(dropdownItem)
    .pause(Duration.ofMillis(200))
    .click()
    .perform();
```

Do not use `pause` as a general wait substitute. Use `WebDriverWait` for element visibility. `pause` is appropriate only for animation timing — a fixed brief delay between two linked mouse movements.

---

## Composing Complex Action Sequences

Actions chains can be as long as needed. They execute as a single dispatched input sequence.

### Filling a Rich Text Editor

Rich text editors like TinyMCE, Quill, or CKEditor use custom key handling:

```java
public void typeInRichTextEditor(WebElement editorBody, String text) {
    new Actions(driver)
        .click(editorBody)
        .keyDown(Keys.CONTROL)
        .sendKeys("a")            // select all
        .keyUp(Keys.CONTROL)
        .sendKeys(Keys.DELETE)    // delete
        .sendKeys(text)           // type new content
        .perform();
}
```

### Navigation via Keyboard in a Table

```java
public void navigateTableCells(WebElement firstCell, int rowsDown, int columnsRight) {
    Actions actions = new Actions(driver);
    actions.click(firstCell);

    for (int r = 0; r < rowsDown; r++) {
        actions.sendKeys(Keys.ARROW_DOWN);
    }
    for (int c = 0; c < columnsRight; c++) {
        actions.sendKeys(Keys.ARROW_RIGHT);
    }

    actions.perform();
}
```

### Resizing a Resizable Element

```java
public void resizeElement(WebElement resizeHandle, int deltaX, int deltaY) {
    new Actions(driver)
        .moveToElement(resizeHandle)
        .clickAndHold()
        .moveByOffset(deltaX, deltaY)
        .release()
        .perform();
}
```

---

## ActionsPage Example — Menu Page Object

```java
package pages;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.interactions.Actions;
import org.openqa.selenium.support.FindBy;

import java.time.Duration;

public class NavigationMenuPage extends BasePage {

    @FindBy(css = "nav li.has-submenu:nth-child(1)")
    private WebElement productsMenu;

    @FindBy(css = "nav li.has-submenu:nth-child(2)")
    private WebElement servicesMenu;

    @FindBy(css = ".submenu-products a[href='/electronics']")
    private WebElement electronicsLink;

    @FindBy(css = ".submenu-products a[href='/furniture']")
    private WebElement furnitureLink;

    @FindBy(css = ".submenu-services a[href='/consulting']")
    private WebElement consultingLink;

    public NavigationMenuPage(WebDriver driver) {
        super(driver);
    }

    public ProductListPage goToElectronics() {
        Actions actions = new Actions(driver);
        actions.moveToElement(productsMenu)
               .pause(Duration.ofMillis(400))
               .moveToElement(electronicsLink)
               .click()
               .perform();
        return new ProductListPage(driver);
    }

    public ProductListPage goToFurniture() {
        Actions actions = new Actions(driver);
        actions.moveToElement(productsMenu)
               .pause(Duration.ofMillis(400))
               .moveToElement(furnitureLink)
               .click()
               .perform();
        return new ProductListPage(driver);
    }

    public void openContextMenuOn(WebElement element) {
        new Actions(driver).contextClick(element).perform();
    }
}
```

---

## Common Issues and Solutions

**Actions chain does nothing.** You must call `.perform()` at the end. Forgetting it builds the chain but never dispatches it.

**Hover does not open the sub-menu.** Add a `pause()` between `moveToElement(menu)` and `moveToElement(subItem)`. CSS transitions take time, and the sub-item is not yet present in the DOM when the mouse arrives.

**Drag and drop fails on HTML5 elements.** Some drag-and-drop implementations use HTML5 drag events that the standard `dragAndDrop` does not trigger correctly. Use JavaScript to fire `dragstart`, `dragover`, and `drop` events manually, or use the manual click-and-hold sequence with pauses.

**Key chords do not work in Firefox.** Ensure `keyDown` and `keyUp` wrap exactly the keys you intend. Firefox is stricter than Chrome about key state management.

**`MoveTargetOutOfBoundsException`.** The element is not in the viewport. Scroll to it before moving the mouse: `JavaScriptUtils.scrollToElement(driver, element)` then build the Actions chain.

---

## Summary

The `Actions` class is the Selenium API for all advanced user input — hover, right-click, double-click, keyboard chords, multi-select, drag, and scroll. Actions chains are composable, readable, and execute atomically. They are essential for testing hover-dependent menus, keyboard navigation, drag-and-drop interfaces, rich text editors, and any interaction that requires precisely coordinated mouse and keyboard events.

---

## Related Topics

- DragDrop.md — Full drag-and-drop implementation guide
- JavaScriptExecutor.md — JS alternatives for actions that fail in certain browsers
- WebElementInteractions.md — Standard click and sendKeys for simple interactions
- Waits.md — WebDriverWait for elements revealed by Actions (hover menus)
