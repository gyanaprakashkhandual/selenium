# JavaScript Executor

## Overview

`JavascriptExecutor` is a Selenium WebDriver interface that allows execution of raw JavaScript code directly in the browser from your Java test code. It is one of the most powerful tools in the Selenium toolkit, enabling operations that are impossible or unreliable with standard WebDriver commands — scrolling, DOM manipulation, retrieving hidden element properties, forcing clicks on intercepted elements, bypassing front-end validation, and directly reading or writing page state.

Every WebDriver implementation (ChromeDriver, FirefoxDriver, EdgeDriver, RemoteWebDriver) implements `JavascriptExecutor`, so casting is always safe when using a real browser driver.

---

## How It Works

`JavascriptExecutor` exposes two methods:

```java
// Execute JavaScript and return the result synchronously
Object executeScript(String script, Object... args);

// Execute JavaScript asynchronously (for AJAX callbacks)
Object executeAsyncScript(String script, Object... args);
```

The `args` array passes Java values into the JavaScript as `arguments[0]`, `arguments[1]`, and so on. WebElements passed as arguments are automatically converted to their corresponding DOM node references in the browser.

### Casting Pattern

```java
JavascriptExecutor js = (JavascriptExecutor) driver;
js.executeScript("script here");
```

In a `BasePage` or utility class this is always safe:

```java
private JavascriptExecutor js() {
    return (JavascriptExecutor) driver;
}
```

---

## Return Types

`executeScript` returns a Java object whose actual type depends on what the JavaScript returns:

| JavaScript return type | Java type returned    |
| ---------------------- | --------------------- |
| `string`               | `String`              |
| `number` (integer)     | `Long`                |
| `number` (decimal)     | `Double`              |
| `boolean`              | `Boolean`             |
| DOM element            | `WebElement`          |
| Array                  | `List<Object>`        |
| Object / Map           | `Map<String, Object>` |
| `undefined` / `null`   | `null`                |

Always cast the return value explicitly:

```java
String title  = (String)  js.executeScript("return document.title;");
Long count    = (Long)    js.executeScript("return document.querySelectorAll('li').length;");
Boolean ready = (Boolean) js.executeScript("return document.readyState === 'complete';");
```

---

## Core Use Cases

### 1. Scrolling

**Scroll to element (center of viewport):**

```java
public void scrollToElement(WebElement element) {
    JavascriptExecutor js = (JavascriptExecutor) driver;
    js.executeScript("arguments[0].scrollIntoView({block: 'center', inline: 'center'});", element);
}
```

**Scroll to page top:**

```java
js.executeScript("window.scrollTo(0, 0);");
```

**Scroll to page bottom:**

```java
js.executeScript("window.scrollTo(0, document.body.scrollHeight);");
```

**Scroll by pixel offset:**

```java
js.executeScript("window.scrollBy(0, 500);");   // down 500 pixels
js.executeScript("window.scrollBy(0, -300);");  // up 300 pixels
```

**Get current scroll position:**

```java
Long scrollY = (Long) js.executeScript("return window.pageYOffset;");
Long scrollX = (Long) js.executeScript("return window.pageXOffset;");
```

---

### 2. Click via JavaScript

Standard `click()` throws `ElementClickInterceptedException` when an overlay, modal, or sticky header sits on top of the element. JavaScript click bypasses this.

```java
public void jsClick(WebElement element) {
    ((JavascriptExecutor) driver).executeScript("arguments[0].click();", element);
}
```

**When to use JavaScript click:**

- Element is behind a cookie consent banner
- Element is partially off-screen and standard scroll-and-click fails
- An animation or transition has not completed
- `ElementClickInterceptedException` is thrown consistently

**When NOT to use JavaScript click:**

- For normal interactions that work fine with `.click()` — JS click bypasses browser event listeners and can produce false positives where the test passes but a real user would be blocked
- For testing that a button is actually clickable by a user

---

### 3. Setting Input Values Directly

For date pickers, custom components, and read-only fields that resist `sendKeys`:

```java
public void setInputValue(WebElement element, String value) {
    JavascriptExecutor js = (JavascriptExecutor) driver;
    js.executeScript("arguments[0].value = arguments[1];", element, value);
    // Trigger the change event so the application's JS listeners fire
    js.executeScript(
        "arguments[0].dispatchEvent(new Event('change', { bubbles: true }));",
        element
    );
    js.executeScript(
        "arguments[0].dispatchEvent(new Event('input', { bubbles: true }));",
        element
    );
}
```

Dispatching `change` and `input` events after setting the value is important for React, Angular, and Vue applications — these frameworks track input via DOM events, not just the element's value property.

---

### 4. Removing Obstructions

**Remove readonly attribute:**

```java
js.executeScript("arguments[0].removeAttribute('readonly');", element);
```

**Remove disabled attribute:**

```java
js.executeScript("arguments[0].removeAttribute('disabled');", element);
```

**Remove a blocking overlay:**

```java
js.executeScript(
    "var overlay = document.getElementById('cookie-banner'); " +
    "if (overlay) overlay.remove();",
    element
);
```

**Set element style to visible:**

```java
js.executeScript("arguments[0].style.display = 'block';", element);
js.executeScript("arguments[0].style.visibility = 'visible';", element);
```

---

### 5. Reading Element Properties

**Inner HTML:**

```java
String html = (String) js.executeScript("return arguments[0].innerHTML;", element);
```

**Inner text (including text hidden via CSS):**

```java
String text = (String) js.executeScript("return arguments[0].innerText;", element);
```

**Computed CSS property:**

```java
String color = (String) js.executeScript(
    "return window.getComputedStyle(arguments[0]).getPropertyValue('color');",
    element
);
```

**Shadow DOM content (for web components):**

```java
WebElement shadowHost = driver.findElement(By.cssSelector("my-component"));
WebElement shadowContent = (WebElement) js.executeScript(
    "return arguments[0].shadowRoot.querySelector('.inner-element');",
    shadowHost
);
```

---

### 6. Page State and Document Interaction

**Check document ready state:**

```java
public boolean isPageLoaded() {
    return ((JavascriptExecutor) driver)
        .executeScript("return document.readyState")
        .equals("complete");
}
```

**Wait for page to fully load using WebDriverWait:**

```java
public void waitForPageLoad(WebDriver driver) {
    new WebDriverWait(driver, Duration.ofSeconds(30)).until(d ->
        ((JavascriptExecutor) d)
            .executeScript("return document.readyState")
            .equals("complete")
    );
}
```

**Get page title:**

```java
String title = (String) js.executeScript("return document.title;");
```

**Get all cookies as JSON:**

```java
String cookies = (String) js.executeScript(
    "return JSON.stringify(document.cookie);"
);
```

**Get local storage value:**

```java
String token = (String) js.executeScript(
    "return window.localStorage.getItem(arguments[0]);",
    "auth_token"
);
```

**Set local storage value:**

```java
js.executeScript(
    "window.localStorage.setItem(arguments[0], arguments[1]);",
    "feature_flag", "true"
);
```

---

### 7. Highlighting Elements (Debugging Aid)

Highlighting is invaluable during debugging — it visually confirms which element is being interacted with.

```java
public void highlight(WebElement element) {
    JavascriptExecutor js = (JavascriptExecutor) driver;
    String originalStyle = element.getAttribute("style");

    // Apply highlight
    js.executeScript(
        "arguments[0].setAttribute('style', arguments[1]);",
        element,
        "border: 3px solid red; background-color: yellow;"
    );

    // Brief pause so you can see the highlight
    try { Thread.sleep(300); } catch (InterruptedException ignored) {}

    // Restore original style
    js.executeScript(
        "arguments[0].setAttribute('style', arguments[1]);",
        element,
        originalStyle != null ? originalStyle : ""
    );
}
```

---

### 8. Counting and Querying DOM Elements

```java
// Count elements matching a CSS selector
Long count = (Long) js.executeScript(
    "return document.querySelectorAll(arguments[0]).length;",
    "table#results tbody tr"
);

// Get text of all matching elements
@SuppressWarnings("unchecked")
List<Object> texts = (List<Object>) js.executeScript(
    "return Array.from(document.querySelectorAll(arguments[0]))" +
    "    .map(el => el.innerText.trim());",
    "ul.menu li"
);

List<String> menuItems = texts.stream()
    .map(Object::toString)
    .collect(Collectors.toList());
```

---

### 9. executeAsyncScript

`executeAsyncScript` is for JavaScript that completes asynchronously. The script must call a callback (the last item in `arguments`) when it is done. The Java thread waits until the callback is called or the script timeout is reached.

**Use case: waiting for a custom AJAX call to complete:**

```java
driver.manage().timeouts().scriptTimeout(Duration.ofSeconds(30));

js.executeAsyncScript(
    "var callback = arguments[arguments.length - 1]; " +
    "fetch('/api/v1/data').then(r => r.json()).then(data => callback(data));"
);
```

**Use case: waiting for Angular to finish rendering:**

```java
js.executeAsyncScript(
    "var callback = arguments[arguments.length - 1]; " +
    "angular.getTestability(document.body).whenStable(callback);"
);
```

---

## Complete JavaScriptUtils Class

Centralizing all JavaScript operations in one utility class keeps page objects clean and avoids duplicating the cast pattern everywhere.

```java
package com.company.framework.utils;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.openqa.selenium.JavascriptExecutor;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.ui.WebDriverWait;

import java.time.Duration;
import java.util.List;
import java.util.stream.Collectors;

public class JavaScriptUtils {

    private static final Logger log = LogManager.getLogger(JavaScriptUtils.class);

    private JavaScriptUtils() {}

    private static JavascriptExecutor js(WebDriver driver) {
        return (JavascriptExecutor) driver;
    }

    // Scroll
    public static void scrollToElement(WebDriver driver, WebElement element) {
        js(driver).executeScript(
            "arguments[0].scrollIntoView({block:'center',inline:'center'});", element);
    }

    public static void scrollToTop(WebDriver driver) {
        js(driver).executeScript("window.scrollTo(0, 0);");
    }

    public static void scrollToBottom(WebDriver driver) {
        js(driver).executeScript("window.scrollTo(0, document.body.scrollHeight);");
    }

    public static void scrollBy(WebDriver driver, int x, int y) {
        js(driver).executeScript("window.scrollBy(arguments[0], arguments[1]);", x, y);
    }

    // Click
    public static void click(WebDriver driver, WebElement element) {
        log.debug("JavaScript click on element: {}", element);
        js(driver).executeScript("arguments[0].click();", element);
    }

    // Input
    public static void setValue(WebDriver driver, WebElement element, String value) {
        js(driver).executeScript("arguments[0].value = arguments[1];", element, value);
        js(driver).executeScript(
            "arguments[0].dispatchEvent(new Event('input',{bubbles:true}));" +
            "arguments[0].dispatchEvent(new Event('change',{bubbles:true}));",
            element
        );
    }

    public static void removeAttribute(WebDriver driver, WebElement element, String attr) {
        js(driver).executeScript("arguments[0].removeAttribute(arguments[1]);", element, attr);
    }

    // Read
    public static String getInnerText(WebDriver driver, WebElement element) {
        return (String) js(driver).executeScript("return arguments[0].innerText;", element);
    }

    public static String getComputedStyle(WebDriver driver, WebElement element, String property) {
        return (String) js(driver).executeScript(
            "return window.getComputedStyle(arguments[0]).getPropertyValue(arguments[1]);",
            element, property
        );
    }

    public static String getLocalStorageItem(WebDriver driver, String key) {
        return (String) js(driver).executeScript(
            "return window.localStorage.getItem(arguments[0]);", key);
    }

    public static void setLocalStorageItem(WebDriver driver, String key, String value) {
        js(driver).executeScript(
            "window.localStorage.setItem(arguments[0], arguments[1]);", key, value);
    }

    // Page state
    public static boolean isPageLoaded(WebDriver driver) {
        return "complete".equals(
            js(driver).executeScript("return document.readyState;"));
    }

    public static void waitForPageLoad(WebDriver driver) {
        new WebDriverWait(driver, Duration.ofSeconds(30)).until(
            d -> isPageLoaded(d));
    }

    public static Long getElementCount(WebDriver driver, String cssSelector) {
        return (Long) js(driver).executeScript(
            "return document.querySelectorAll(arguments[0]).length;", cssSelector);
    }

    // Highlight
    public static void highlight(WebDriver driver, WebElement element) {
        String original = element.getAttribute("style");
        js(driver).executeScript(
            "arguments[0].style.border='3px solid red';" +
            "arguments[0].style.backgroundColor='yellow';", element);
        try { Thread.sleep(400); } catch (InterruptedException ignored) {}
        js(driver).executeScript(
            "arguments[0].setAttribute('style', arguments[1]);",
            element, original != null ? original : "");
    }
}
```

---

## Common Pitfalls

**Using JS click as the default.** JS click bypasses browser event processing. Always try the standard `.click()` first. Use JS click only as a fallback when the standard click fails with `ElementClickInterceptedException`.

**Not dispatching events after setting values.** Setting `element.value` directly does not trigger `onchange` or `oninput` listeners in JavaScript-heavy frameworks. Always dispatch the events after setting a value programmatically.

**Assuming Long return type for all numbers.** JavaScript integers return as `Long` in Java. Floats return as `Double`. Always check the type before casting.

**Not waiting for page readyState.** After calling `driver.get()` or clicking a link that loads a new page, always wait for `document.readyState === 'complete'` before querying the new page's elements.

---

## Summary

`JavascriptExecutor` is the escape hatch for everything WebDriver's standard API cannot do. It enables scrolling with precision, interacting with intercepted or read-only elements, reading computed styles, accessing local storage, working with shadow DOM, and checking page load state. Used correctly — as a targeted solution for specific problems, not as a replacement for standard WebDriver — it makes automation of complex modern web applications reliable and complete.

---

## Related Topics

- ActionsClass.md — Mouse and keyboard interactions for drag, hover, and key chords
- WaitUtils — Custom wait conditions for AJAX and dynamic content
- HeadlessTesting.md — JavaScript execution behavior in headless mode
- CDP.md — Chrome DevTools Protocol as an alternative for advanced browser control
