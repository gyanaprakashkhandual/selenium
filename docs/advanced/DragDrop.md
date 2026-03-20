# Drag and Drop

## Overview

Drag and drop is one of the most challenging interactions to automate reliably in Selenium. It requires the browser to simulate a coordinated sequence of mouse press, movement, and release events at precise coordinates. The difficulty compounds because modern web applications implement drag and drop in three distinct ways — native HTML5 drag events, JavaScript-based libraries (like Sortable.js, jQuery UI, or React DnD), and CSS-based implementations — each of which responds differently to Selenium's standard `dragAndDrop` method.

This document covers every reliable approach for automating drag and drop, with detection guidance for each UI type.

---

## Understanding Drag and Drop Event Types

### HTML5 Native Drag and Drop

Uses the browser's built-in drag API. An element with `draggable="true"` fires:

- `dragstart` — when drag begins
- `drag` — during movement
- `dragenter` / `dragover` — when hovering a valid drop target
- `drop` — when released on the target
- `dragend` — when drag operation completes

Selenium's standard `dragAndDrop` method fires `mousedown` and `mouseup`, but **does not fire HTML5 drag events**. This means the standard Selenium method often fails on HTML5 drag-and-drop.

### JavaScript Library Drag and Drop

Libraries like jQuery UI Draggable, Sortable.js, React DnD, and Angular CDK respond to `mousemove` events, not drag events. Selenium's `clickAndHold` + `moveToElement` + `release` sequence fires `mousemove` events and generally works with these libraries, though timing and pauses matter.

### How to Identify Which Type Your App Uses

Open browser DevTools and inspect the draggable element:

```
HTML5: Element has draggable="true" attribute
jQuery UI: Element has class "ui-draggable"
Sortable.js: Parent list has data-sortable or class "sortable"
React DnD: Look for data-rbd-draggable-id attributes
Angular CDK: Look for cdkDrag directive
```

---

## Approach 1 — Standard dragAndDrop (Works for Mouse-Event Libraries)

The simplest approach. Use this first. If it works for your application, no further complexity is needed.

```java
public void dragAndDrop(WebElement source, WebElement target) {
    new Actions(driver)
        .dragAndDrop(source, target)
        .perform();
}
```

**Verification:** After calling `dragAndDrop`, assert the expected state change — item moved to new position, list reordered, element dropped in zone.

---

## Approach 2 — Manual Click-and-Hold Sequence

More reliable than `dragAndDrop()` for JavaScript library-based implementations. Pauses between steps give the application time to register each phase of the drag.

```java
public void dragAndDropManual(WebDriver driver, WebElement source, WebElement target) {
    Actions actions = new Actions(driver);
    actions
        .moveToElement(source)
        .pause(Duration.ofMillis(300))
        .clickAndHold(source)
        .pause(Duration.ofMillis(500))
        .moveToElement(target)
        .pause(Duration.ofMillis(500))
        .release(target)
        .pause(Duration.ofMillis(300))
        .perform();
}
```

**Tuning the pauses:** Start with 500ms pauses. If the drag fails, increase to 800ms. If it still fails, the library likely requires HTML5 events — use Approach 3.

---

## Approach 3 — JavaScript HTML5 Drag Event Simulation

This is the only reliable approach for HTML5 native drag and drop. It manually fires the complete event sequence that browsers fire during a real drag operation.

### JavaScript Drag Event Simulator

```java
public void dragAndDropHTML5(WebDriver driver, WebElement source, WebElement target) {
    JavascriptExecutor js = (JavascriptExecutor) driver;

    String dragScript =
        "function simulateDrag(sourceEl, targetEl) {" +
        "  var dataTransfer = {" +
        "    data: {}," +
        "    setData: function(type, val) { this.data[type] = val; }," +
        "    getData: function(type) { return this.data[type]; }," +
        "    clearData: function() { this.data = {}; }," +
        "    dropEffect: 'move'," +
        "    effectAllowed: 'all'" +
        "  };" +
        "" +
        "  function fire(el, eventName, opts) {" +
        "    var event = new DragEvent(eventName, {" +
        "      bubbles: true, cancelable: true," +
        "      ...opts" +
        "    });" +
        "    Object.defineProperty(event, 'dataTransfer', { value: dataTransfer });" +
        "    el.dispatchEvent(event);" +
        "  }" +
        "" +
        "  var srcRect = sourceEl.getBoundingClientRect();" +
        "  var tgtRect = targetEl.getBoundingClientRect();" +
        "" +
        "  fire(sourceEl, 'mousedown', {clientX: srcRect.left + srcRect.width/2, clientY: srcRect.top + srcRect.height/2});" +
        "  fire(sourceEl, 'dragstart', {clientX: srcRect.left + srcRect.width/2, clientY: srcRect.top + srcRect.height/2});" +
        "  fire(sourceEl, 'drag',      {clientX: srcRect.left + srcRect.width/2, clientY: srcRect.top + srcRect.height/2});" +
        "  fire(targetEl, 'dragenter', {clientX: tgtRect.left + tgtRect.width/2, clientY: tgtRect.top + tgtRect.height/2});" +
        "  fire(targetEl, 'dragover',  {clientX: tgtRect.left + tgtRect.width/2, clientY: tgtRect.top + tgtRect.height/2});" +
        "  fire(targetEl, 'drop',      {clientX: tgtRect.left + tgtRect.width/2, clientY: tgtRect.top + tgtRect.height/2});" +
        "  fire(sourceEl, 'dragend',   {clientX: tgtRect.left + tgtRect.width/2, clientY: tgtRect.top + tgtRect.height/2});" +
        "}" +
        "simulateDrag(arguments[0], arguments[1]);";

    js.executeScript(dragScript, source, target);

    // Brief pause after script execution for DOM to update
    try { Thread.sleep(500); } catch (InterruptedException ignored) {}
}
```

---

## Approach 4 — Drag to Pixel Offset

Use this when there is no distinct drop target element — only a target location defined by coordinates. Common in canvas-based UIs, diagram editors, and image crop tools.

```java
public void dragToOffset(WebDriver driver, WebElement source, int xOffset, int yOffset) {
    new Actions(driver)
        .moveToElement(source)
        .pause(Duration.ofMillis(300))
        .clickAndHold(source)
        .pause(Duration.ofMillis(400))
        .moveByOffset(xOffset, yOffset)
        .pause(Duration.ofMillis(300))
        .release()
        .perform();
}
```

**For smooth incremental movement (avoids skipping drag events):**

```java
public void dragToOffsetSmoothly(WebDriver driver, WebElement source,
                                  int totalX, int totalY, int steps) {

    int stepX = totalX / steps;
    int stepY = totalY / steps;

    Actions actions = new Actions(driver);
    actions.moveToElement(source)
           .clickAndHold(source);

    for (int i = 0; i < steps; i++) {
        actions.moveByOffset(stepX, stepY)
               .pause(Duration.ofMillis(50));
    }

    actions.release().perform();
}
```

Smooth movement is important for canvas applications and image editors that track drag distance per event rather than just start and end position.

---

## Complete Drag and Drop Utility Class

```java
package com.company.framework.utils;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.JavascriptExecutor;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.interactions.Actions;

import java.time.Duration;

public class DragDropUtils {

    private static final Logger log = LogManager.getLogger(DragDropUtils.class);

    private DragDropUtils() {}

    /**
     * Approach 1 — Standard dragAndDrop.
     * Use for mouse-event library implementations (jQuery UI, some Angular CDK).
     */
    public static void dragAndDrop(WebDriver driver, WebElement source, WebElement target) {
        log.info("Drag and drop (standard): {} -> {}", source, target);
        new Actions(driver).dragAndDrop(source, target).perform();
    }

    /**
     * Approach 2 — Manual click-hold-move-release with pauses.
     * Use for Sortable.js, React DnD, and other mousemove-based libraries.
     */
    public static void dragAndDropManual(WebDriver driver,
                                          WebElement source, WebElement target) {
        log.info("Drag and drop (manual): {} -> {}", source, target);
        new Actions(driver)
            .moveToElement(source)
            .pause(Duration.ofMillis(300))
            .clickAndHold(source)
            .pause(Duration.ofMillis(500))
            .moveToElement(target)
            .pause(Duration.ofMillis(500))
            .release(target)
            .perform();
    }

    /**
     * Approach 3 — JavaScript HTML5 event simulation.
     * Use for native HTML5 draggable="true" implementations.
     */
    public static void dragAndDropHTML5(WebDriver driver,
                                         WebElement source, WebElement target) {
        log.info("Drag and drop (HTML5 JS): {} -> {}", source, target);
        String script = buildHTML5DragScript();
        ((JavascriptExecutor) driver).executeScript(script, source, target);
        try { Thread.sleep(500); } catch (InterruptedException ignored) {}
    }

    /**
     * Approach 4 — Drag to pixel offset from source element.
     * Use for canvas, diagram editors, and coordinate-based drop zones.
     */
    public static void dragToOffset(WebDriver driver,
                                     WebElement source, int xOffset, int yOffset) {
        log.info("Drag to offset ({}, {})", xOffset, yOffset);
        new Actions(driver)
            .moveToElement(source)
            .pause(Duration.ofMillis(300))
            .clickAndHold(source)
            .pause(Duration.ofMillis(400))
            .moveByOffset(xOffset, yOffset)
            .pause(Duration.ofMillis(300))
            .release()
            .perform();
    }

    /**
     * Smooth drag to offset using incremental movement.
     * Use for canvas applications that track movement per event.
     */
    public static void dragToOffsetSmoothly(WebDriver driver, WebElement source,
                                             int totalX, int totalY, int steps) {
        log.info("Smooth drag to offset ({}, {}) in {} steps", totalX, totalY, steps);
        int stepX = totalX / steps;
        int stepY = totalY / steps;

        Actions actions = new Actions(driver);
        actions.moveToElement(source)
               .clickAndHold(source);

        for (int i = 0; i < steps; i++) {
            actions.moveByOffset(stepX, stepY)
                   .pause(Duration.ofMillis(30));
        }
        actions.release().perform();
    }

    private static String buildHTML5DragScript() {
        return
            "function simulateDrag(src, tgt) {" +
            "  var dt={data:{},setData:function(t,v){this.data[t]=v;}," +
            "    getData:function(t){return this.data[t];}," +
            "    clearData:function(){this.data={};}," +
            "    dropEffect:'move',effectAllowed:'all'};" +
            "  function fire(el,name,opts){" +
            "    var e=new DragEvent(name,{bubbles:true,cancelable:true,...opts});" +
            "    Object.defineProperty(e,'dataTransfer',{value:dt});" +
            "    el.dispatchEvent(e);}" +
            "  var s=src.getBoundingClientRect(),t=tgt.getBoundingClientRect();" +
            "  var sx=s.left+s.width/2,sy=s.top+s.height/2;" +
            "  var tx=t.left+t.width/2,ty=t.top+t.height/2;" +
            "  fire(src,'mousedown',{clientX:sx,clientY:sy});" +
            "  fire(src,'dragstart',{clientX:sx,clientY:sy});" +
            "  fire(src,'drag',{clientX:sx,clientY:sy});" +
            "  fire(tgt,'dragenter',{clientX:tx,clientY:ty});" +
            "  fire(tgt,'dragover',{clientX:tx,clientY:ty});" +
            "  fire(tgt,'drop',{clientX:tx,clientY:ty});" +
            "  fire(src,'dragend',{clientX:tx,clientY:ty});}" +
            "simulateDrag(arguments[0],arguments[1]);";
    }
}
```

---

## Page Object Implementation — Kanban Board

A Kanban board is a common drag-and-drop use case: dragging cards between columns.

```java
package pages;

import com.company.framework.utils.DragDropUtils;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;

import java.util.List;

public class KanbanBoardPage extends BasePage {

    @FindBy(css = ".column[data-status='todo'] .card-list")
    private WebElement todoColumn;

    @FindBy(css = ".column[data-status='in-progress'] .card-list")
    private WebElement inProgressColumn;

    @FindBy(css = ".column[data-status='done'] .card-list")
    private WebElement doneColumn;

    public KanbanBoardPage(WebDriver driver) {
        super(driver);
    }

    public WebElement getCardByTitle(String title) {
        String xpath = "//div[contains(@class,'card')]//span[normalize-space()='" + title + "']/ancestor::div[contains(@class,'card')]";
        return waitForVisibility(By.xpath(xpath));
    }

    public void moveCardToInProgress(String cardTitle) {
        WebElement card = getCardByTitle(cardTitle);
        DragDropUtils.dragAndDropManual(driver, card, inProgressColumn);
    }

    public void moveCardToDone(String cardTitle) {
        WebElement card = getCardByTitle(cardTitle);
        DragDropUtils.dragAndDropManual(driver, card, doneColumn);
    }

    public List<String> getCardsInColumn(String status) {
        String css = ".column[data-status='" + status + "'] .card .card-title";
        return driver.findElements(By.cssSelector(css))
            .stream()
            .map(WebElement::getText)
            .collect(java.util.stream.Collectors.toList());
    }

    public boolean isCardInColumn(String cardTitle, String status) {
        return getCardsInColumn(status).contains(cardTitle);
    }
}
```

**Feature file:**

```gherkin
Scenario: Move a task card from To Do to In Progress
  Given the user is on the project Kanban board
  And the card "Implement login API" is in the "todo" column
  When the user drags "Implement login API" to the "in-progress" column
  Then the card "Implement login API" should be visible in the "in-progress" column
  And the card should not be in the "todo" column
```

---

## Page Object Implementation — Sortable List

```java
package pages;

import com.company.framework.utils.DragDropUtils;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;

import java.util.List;
import java.util.stream.Collectors;

public class SortableListPage extends BasePage {

    @FindBy(css = "ul.sortable li")
    private List<WebElement> listItems;

    public SortableListPage(WebDriver driver) {
        super(driver);
    }

    public void moveItemToPosition(String itemText, int targetPosition) {
        WebElement source = getItemByText(itemText);
        WebElement target = listItems.get(targetPosition - 1); // 1-indexed
        DragDropUtils.dragAndDropManual(driver, source, target);
    }

    public List<String> getCurrentOrder() {
        return listItems.stream()
            .map(WebElement::getText)
            .collect(Collectors.toList());
    }

    public int getItemPosition(String itemText) {
        List<String> order = getCurrentOrder();
        return order.indexOf(itemText) + 1; // 1-indexed
    }

    private WebElement getItemByText(String text) {
        return listItems.stream()
            .filter(el -> el.getText().trim().equals(text))
            .findFirst()
            .orElseThrow(() -> new RuntimeException("List item not found: " + text));
    }
}
```

---

## Troubleshooting Drag and Drop

**Symptom: `dragAndDrop` completes without error but nothing moves.**
Cause: The element uses HTML5 drag events, not mouse events.
Fix: Switch to `dragAndDropHTML5()`.

**Symptom: Manual approach starts the drag but releases in the wrong position.**
Cause: The element is not fully in the viewport before dragging.
Fix: Scroll both source and target into view before the drag. Use `JavaScriptUtils.scrollToElement()` on both.

**Symptom: Drag starts but item snaps back.**
Cause: Pauses between `clickAndHold` and `moveToElement` are too short.
Fix: Increase `pause()` values from 300ms to 600–800ms.

**Symptom: HTML5 drag script fires but drop is not registered.**
Cause: The drop zone requires `dragover` to call `event.preventDefault()`. Verify the application has this handler. If not, your drop event will be cancelled by the browser.

**Symptom: Works in Chrome, fails in Firefox.**
Cause: Firefox implements drag events more strictly. Ensure all events in the sequence fire in the correct order. Increase pauses between events in the JS script.

---

## Summary

Drag and drop automation requires selecting the right approach based on how the application implements it. Standard `dragAndDrop()` works for mouse-event libraries. Manual click-hold-move-release with pauses works for Sortable.js and React DnD. JavaScript HTML5 event simulation works for native `draggable="true"` elements. Offset dragging works for canvas and coordinate-based drop zones. Inspecting the application's HTML attributes and CSS classes before choosing an approach saves significant debugging time.

---

## Related Topics

- ActionsClass.md — Full Actions API reference including clickAndHold and moveByOffset
- JavaScriptExecutor.md — JavaScript execution used in HTML5 drag simulation
- Waits.md — Waiting for drag-drop results to appear in the DOM
