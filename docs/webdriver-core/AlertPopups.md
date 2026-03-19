# Handling Alerts and Popups

## Overview

Web applications use various types of popups and alerts to communicate with users — from native browser dialog boxes to modern JavaScript modal dialogs and browser permission prompts. Each type requires a different handling strategy. This document covers every popup type you will encounter in real-world automation, with complete code and patterns.

---

## JavaScript Browser Alerts

JavaScript provides three native browser dialog types. They block page execution until dismissed and cannot be styled by the application — they look different in every browser. Selenium handles them through the `Alert` interface.

### The Three Alert Types

| Type        | JS Method                              | User Interaction         |
| ----------- | -------------------------------------- | ------------------------ |
| **Alert**   | `window.alert("message")`              | OK button only           |
| **Confirm** | `window.confirm("question")`           | OK + Cancel              |
| **Prompt**  | `window.prompt("question", "default")` | Text input + OK + Cancel |

---

### Handling Simple Alert (window.alert)

```java
import org.openqa.selenium.Alert;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.WebDriverWait;
import java.time.Duration;

WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// Trigger the alert
driver.findElement(By.id("trigger-alert-btn")).click();

// Wait for alert to be present (critical — never switch without waiting)
wait.until(ExpectedConditions.alertIsPresent());

// Switch to the alert
Alert alert = driver.switchTo().alert();

// Read the alert text
String alertMessage = alert.getText();
System.out.println("Alert message: " + alertMessage);
Assert.assertEquals(alertMessage, "Operation successful!");

// Accept (click OK)
alert.accept();

// After accepting, control returns to the main page
System.out.println("Back to page: " + driver.getTitle());
```

---

### Handling Confirm Dialog (window.confirm)

```java
// Trigger confirm dialog
driver.findElement(By.id("delete-btn")).click();

// Wait and switch
wait.until(ExpectedConditions.alertIsPresent());
Alert confirmDialog = driver.switchTo().alert();

// Read the question
String question = confirmDialog.getText();
System.out.println("Confirm text: " + question);
Assert.assertTrue(question.contains("Are you sure"));

// Accept (click OK — confirms the action)
confirmDialog.accept();

// OR dismiss (click Cancel — cancels the action)
// confirmDialog.dismiss();

// Verify the result after confirmation
wait.until(ExpectedConditions.visibilityOfElementLocated(
    By.cssSelector(".success-notification")
));
```

---

### Handling Prompt Dialog (window.prompt)

```java
// Trigger prompt
driver.findElement(By.id("rename-btn")).click();

// Wait and switch
wait.until(ExpectedConditions.alertIsPresent());
Alert promptDialog = driver.switchTo().alert();

// Read the prompt message
System.out.println("Prompt: " + promptDialog.getText());

// Clear any default text and type a new value
promptDialog.sendKeys("New Folder Name");

// Accept with the typed value (click OK)
promptDialog.accept();

// OR dismiss (click Cancel — typed value is discarded)
// promptDialog.dismiss();
```

---

### Alert Utility Class

```java
package com.yourcompany.automation.utils;

import org.openqa.selenium.*;
import org.openqa.selenium.support.ui.*;
import java.time.Duration;

public class AlertHandler {

    private final WebDriver driver;
    private final WebDriverWait wait;

    public AlertHandler(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    /**
     * Accept an alert (click OK). Returns alert text.
     */
    public String acceptAlert() {
        Alert alert = wait.until(ExpectedConditions.alertIsPresent());
        String text = alert.getText();
        alert.accept();
        return text;
    }

    /**
     * Dismiss an alert (click Cancel). Returns alert text.
     */
    public String dismissAlert() {
        Alert alert = wait.until(ExpectedConditions.alertIsPresent());
        String text = alert.getText();
        alert.dismiss();
        return text;
    }

    /**
     * Get alert text without dismissing it.
     */
    public String getAlertText() {
        return wait.until(ExpectedConditions.alertIsPresent()).getText();
    }

    /**
     * Type text in a prompt dialog and accept.
     */
    public void typeInPromptAndAccept(String text) {
        Alert prompt = wait.until(ExpectedConditions.alertIsPresent());
        prompt.sendKeys(text);
        prompt.accept();
    }

    /**
     * Check if an alert is present (non-blocking).
     */
    public boolean isAlertPresent() {
        try {
            new WebDriverWait(driver, Duration.ofSeconds(3))
                .until(ExpectedConditions.alertIsPresent());
            return true;
        } catch (TimeoutException e) {
            return false;
        }
    }

    /**
     * Accept alert if present, do nothing if not.
     */
    public void acceptAlertIfPresent() {
        if (isAlertPresent()) {
            driver.switchTo().alert().accept();
        }
    }
}
```

---

## Modal Dialog (Bootstrap / Custom)

Unlike JavaScript alerts, modal dialogs are HTML elements rendered within the page DOM. They are handled like regular page interactions — with waits and element clicks.

### Bootstrap Modal

```html
<!-- Bootstrap modal structure -->
<div class="modal fade" id="confirmModal" tabindex="-1">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title">Confirm Delete</h5>
      </div>
      <div class="modal-body">Are you sure you want to delete this record?</div>
      <div class="modal-footer">
        <button id="modal-cancel" class="btn btn-secondary">Cancel</button>
        <button id="modal-confirm" class="btn btn-danger">Delete</button>
      </div>
    </div>
  </div>
</div>
```

```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));

// Trigger modal
driver.findElement(By.id("delete-record-btn")).click();

// Wait for modal to be visible
wait.until(ExpectedConditions.visibilityOfElementLocated(By.id("confirmModal")));

// Read modal title and body
String modalTitle = driver.findElement(By.cssSelector("#confirmModal .modal-title")).getText();
String modalBody  = driver.findElement(By.cssSelector("#confirmModal .modal-body")).getText();
System.out.println("Modal title: " + modalTitle);
System.out.println("Modal body:  " + modalBody);

// Click Confirm (Delete)
driver.findElement(By.id("modal-confirm")).click();

// Wait for modal to close
wait.until(ExpectedConditions.invisibilityOfElementLocated(By.id("confirmModal")));

// Verify success
wait.until(ExpectedConditions.visibilityOfElementLocated(
    By.cssSelector(".alert-success")
));
```

### Modal Page Object Component

```java
public class ConfirmModal {

    private final WebDriver driver;
    private final WebDriverWait wait;

    private final By modalLocator    = By.id("confirmModal");
    private final By titleLocator    = By.cssSelector("#confirmModal .modal-title");
    private final By messageLocator  = By.cssSelector("#confirmModal .modal-body");
    private final By confirmBtn      = By.id("modal-confirm");
    private final By cancelBtn       = By.id("modal-cancel");
    private final By closeBtn        = By.cssSelector("#confirmModal .close");

    public ConfirmModal(WebDriver driver) {
        this.driver = driver;
        this.wait = new WebDriverWait(driver, Duration.ofSeconds(10));
    }

    public ConfirmModal waitForDisplay() {
        wait.until(ExpectedConditions.visibilityOfElementLocated(modalLocator));
        return this;
    }

    public String getTitle() {
        return driver.findElement(titleLocator).getText();
    }

    public String getMessage() {
        return driver.findElement(messageLocator).getText();
    }

    public void confirm() {
        driver.findElement(confirmBtn).click();
        wait.until(ExpectedConditions.invisibilityOfElementLocated(modalLocator));
    }

    public void cancel() {
        driver.findElement(cancelBtn).click();
        wait.until(ExpectedConditions.invisibilityOfElementLocated(modalLocator));
    }

    public void closeWithX() {
        driver.findElement(closeBtn).click();
        wait.until(ExpectedConditions.invisibilityOfElementLocated(modalLocator));
    }

    public boolean isVisible() {
        return !driver.findElements(modalLocator).isEmpty()
            && driver.findElement(modalLocator).isDisplayed();
    }
}

// Usage in step definitions
ConfirmModal modal = new ConfirmModal(driver);
modal.waitForDisplay();
Assert.assertEquals(modal.getMessage(), "Are you sure you want to delete this record?");
modal.confirm();
```

---

## Authentication Popups (Basic Auth)

Some applications use HTTP Basic Authentication, which triggers a browser-level login popup. Selenium 4 handles this via URL credential embedding:

```java
// Method 1: Embed credentials in URL (most reliable)
String username = "admin";
String password = "password123";
String baseUrl   = "https://protected.example.com";

String authenticatedUrl = String.format("https://%s:%s@%s",
    username, password, baseUrl.replace("https://", "")
);
driver.get(authenticatedUrl);

// Method 2: Chrome DevTools Protocol (Selenium 4)
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.devtools.DevTools;
import org.openqa.selenium.devtools.v121.network.Network;

ChromeDriver chromeDriver = (ChromeDriver) driver;
DevTools devTools = chromeDriver.getDevTools();
devTools.createSession();
devTools.send(Network.enable(Optional.empty(), Optional.empty(), Optional.empty()));

// Set authorization header for all requests
String credentials = java.util.Base64.getEncoder()
    .encodeToString((username + ":" + password).getBytes());
devTools.send(Network.setExtraHTTPHeaders(
    new org.openqa.selenium.devtools.v121.network.model.Headers(
        Map.of("Authorization", "Basic " + credentials)
    )
));
driver.get(baseUrl);
```

---

## Browser Permission Popups

Modern browsers ask for permissions (camera, microphone, location, notifications). Handle them by denying or allowing via `ChromeOptions` before the browser launches:

```java
import org.openqa.selenium.chrome.ChromeOptions;
import java.util.HashMap;
import java.util.Map;

ChromeOptions options = new ChromeOptions();
Map<String, Object> prefs = new HashMap<>();

// Permission values: 1=allow, 2=deny, 3=ask (default)

// Deny notifications (most common — prevents popup asking for permission)
prefs.put("profile.default_content_setting_values.notifications", 2);

// Deny geolocation
prefs.put("profile.default_content_setting_values.geolocation", 2);

// Allow camera
prefs.put("profile.default_content_setting_values.media_stream_camera", 1);

// Allow microphone
prefs.put("profile.default_content_setting_values.media_stream_mic", 1);

// Deny pop-ups
prefs.put("profile.default_content_setting_values.popups", 2);

options.setExperimentalOption("prefs", prefs);
WebDriver driver = new ChromeDriver(options);
```

---

## Toast Notifications and Snackbars

Toast notifications appear briefly and auto-dismiss. You must capture them before they disappear:

```java
// Wait for toast to appear
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(5));
WebElement toast = wait.until(
    ExpectedConditions.visibilityOfElementLocated(
        By.cssSelector(".toast-message, .snackbar, [data-testid='toast']")
    )
);

String toastText = toast.getText();
System.out.println("Toast: " + toastText);
Assert.assertTrue(toastText.contains("saved successfully"));

// Optionally wait for it to auto-dismiss
wait.until(ExpectedConditions.invisibilityOf(toast));
```

---

## File Download Dialog

Prevent file download dialogs by configuring Chrome preferences (see Browser Commands):

```java
// Pre-configure Chrome to auto-download without dialog
Map<String, Object> prefs = new HashMap<>();
prefs.put("download.default_directory",
    System.getProperty("user.dir") + "/downloads");
prefs.put("download.prompt_for_download", false);
prefs.put("download.directory_upgrade", true);
prefs.put("safebrowsing.enabled", true);

ChromeOptions options = new ChromeOptions();
options.setExperimentalOption("prefs", prefs);

// After clicking download, verify file exists
public boolean waitForDownload(String fileName, int timeoutSeconds) {
    File downloadDir = new File(System.getProperty("user.dir") + "/downloads");
    long startTime = System.currentTimeMillis();
    while (System.currentTimeMillis() - startTime < timeoutSeconds * 1000L) {
        File[] files = downloadDir.listFiles();
        if (files != null) {
            for (File file : files) {
                if (file.getName().equals(fileName) && !file.getName().endsWith(".crdownload")) {
                    return true;
                }
            }
        }
        try { Thread.sleep(500); } catch (InterruptedException ignored) {}
    }
    return false;
}

driver.findElement(By.id("download-report-btn")).click();
Assert.assertTrue(waitForDownload("report.pdf", 30), "File was not downloaded");
```

---

## Summary

JavaScript alerts (alert, confirm, prompt) are handled via `driver.switchTo().alert()` — always wait for the alert to be present first. HTML modal dialogs are standard page elements — use waits and click the appropriate buttons. Browser-level popups (Basic Auth, permissions) are best handled proactively via `ChromeOptions` preferences before launching the browser. Build an `AlertHandler` utility class for reusable alert interactions, and a modal component class for reusable modal interactions within your Page Object Model.
