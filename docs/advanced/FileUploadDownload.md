# File Upload and Download

## Overview

File upload and download are among the most common advanced interactions in web application testing. Uploads and downloads involve the operating system file system, native browser dialogs, and server responses — all of which require specific Selenium techniques. Standard WebDriver methods handle the majority of cases cleanly. Edge cases like drag-and-drop upload zones, hidden file inputs, and download verification require additional strategies.

---

## File Upload

### How Web File Upload Works

HTML file inputs render as `<input type="file">` elements. When a user clicks the element, the operating system's native file dialog opens. Selenium cannot interact with native OS dialogs. Instead, it bypasses the dialog entirely by calling `sendKeys()` directly on the file input with the absolute path to the file. The browser treats this exactly as if the user had selected the file through the dialog.

```html
<!-- Standard file input -->
<input type="file" id="fileUpload" name="file" accept=".pdf,.doc,.docx" />

<!-- Multiple file selection -->
<input type="file" id="multiUpload" multiple />
```

### Standard sendKeys Upload

```java
public void uploadFile(WebElement fileInputElement, String absoluteFilePath) {
    // Do NOT click the element — that opens the OS dialog
    // sendKeys directly sets the file path
    fileInputElement.sendKeys(absoluteFilePath);
}
```

**Usage in a page class:**

```java
package pages;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import org.openqa.selenium.support.FindBy;

public class DocumentUploadPage extends BasePage {

    @FindBy(id = "fileUpload")
    private WebElement fileInput;

    @FindBy(id = "uploadBtn")
    private WebElement uploadButton;

    @FindBy(css = ".upload-success-msg")
    private WebElement successMessage;

    @FindBy(css = ".upload-error-msg")
    private WebElement errorMessage;

    @FindBy(css = ".file-preview-name")
    private WebElement filePreviewName;

    public DocumentUploadPage(WebDriver driver) {
        super(driver);
    }

    public void uploadFile(String filePath) {
        fileInput.sendKeys(filePath);
    }

    public void clickUpload() {
        click(uploadButton);
    }

    public DocumentUploadPage uploadAndSubmit(String filePath) {
        uploadFile(filePath);
        clickUpload();
        return this;
    }

    public String getSuccessMessage() {
        return getText(successMessage);
    }

    public String getErrorMessage() {
        return getText(errorMessage);
    }

    public String getUploadedFileName() {
        return getText(filePreviewName);
    }

    public boolean isUploadSuccessful() {
        return isDisplayed(successMessage);
    }
}
```

### Resolving File Paths for Tests

Always use absolute paths. Relative paths cause failures depending on the working directory. Build the absolute path from the project's resource directory:

```java
// Get absolute path of a file in src/test/resources/testfiles/
public static String getTestFilePath(String fileName) {
    return System.getProperty("user.dir")
        + "/src/test/resources/testfiles/"
        + fileName;
}

// Or use ClassLoader for classpath resources
public static String getClasspathFilePath(String resourceName) {
    return Paths.get(
        DocumentUploadPage.class
            .getClassLoader()
            .getResource("testfiles/" + resourceName)
            .toURI()
    ).toAbsolutePath().toString();
}
```

**In the step definition:**

```java
@When("the user uploads the file {string}")
public void theUserUploadsFile(String fileName) throws Exception {
    String filePath = getClasspathFilePath(fileName);
    context.getUploadPage().uploadFile(filePath);
}
```

---

### Uploading Multiple Files

```java
@FindBy(id = "multiFileInput")
private WebElement multiFileInput;

public void uploadMultipleFiles(List<String> filePaths) {
    // Join paths with \n for multiple files
    String allPaths = String.join("\n", filePaths);
    multiFileInput.sendKeys(allPaths);
}
```

```java
// In step definition
@When("the user uploads the following files:")
public void theUserUploadsFiles(List<String> fileNames) throws Exception {
    List<String> absolutePaths = fileNames.stream()
        .map(name -> {
            try { return getClasspathFilePath(name); }
            catch (Exception e) { throw new RuntimeException(e); }
        })
        .collect(Collectors.toList());

    context.getUploadPage().uploadMultipleFiles(absolutePaths);
}
```

---

### Hidden File Input — Making It Visible

Some applications hide the `<input type="file">` and show a styled custom button instead. The hidden input cannot receive `sendKeys` when it is hidden. Make it visible via JavaScript first.

```java
public void uploadToHiddenInput(String cssSelector, String filePath) {
    WebElement hiddenInput = driver.findElement(By.cssSelector(cssSelector));

    // Make the hidden input interactable
    ((JavascriptExecutor) driver).executeScript(
        "arguments[0].style.display = 'block';" +
        "arguments[0].style.visibility = 'visible';" +
        "arguments[0].style.opacity = '1';",
        hiddenInput
    );

    hiddenInput.sendKeys(filePath);
}
```

---

### Drag-and-Drop Upload Zone

Some upload UIs present a drop zone that accepts files dragged from the desktop. These use HTML5 File API events. Simulate them with JavaScript:

```java
public void uploadFileToDragZone(WebElement dropZone, String filePath) throws Exception {
    // Read the file as base64
    byte[] fileBytes = Files.readAllBytes(Paths.get(filePath));
    String base64File = Base64.getEncoder().encodeToString(fileBytes);
    String fileName = Paths.get(filePath).getFileName().toString();
    String mimeType = Files.probeContentType(Paths.get(filePath));

    String script =
        "var b64 = arguments[0];" +
        "var name = arguments[1];" +
        "var type = arguments[2];" +
        "var zone = arguments[3];" +
        "var bytes = atob(b64);" +
        "var arr = new Uint8Array(bytes.length);" +
        "for(var i=0;i<bytes.length;i++) arr[i]=bytes.charCodeAt(i);" +
        "var blob = new Blob([arr],{type:type});" +
        "var file = new File([blob],name,{type:type});" +
        "var dt = new DataTransfer();" +
        "dt.items.add(file);" +
        "var event = new DragEvent('drop',{bubbles:true,cancelable:true,dataTransfer:dt});" +
        "zone.dispatchEvent(event);";

    ((JavascriptExecutor) driver).executeScript(
        script, base64File, fileName, mimeType, dropZone
    );
}
```

---

### Upload Validation Scenarios

```gherkin
Feature: Document Upload

  Background:
    Given the user is logged in
    And the user is on the document upload page

  @smoke @positive
  Scenario: Upload a valid PDF document
    When the user uploads the file "sample-document.pdf"
    And the user clicks the upload button
    Then the success message "File uploaded successfully" should appear
    And the uploaded file name "sample-document.pdf" should be shown

  @negative
  Scenario: Upload fails with unsupported file type
    When the user uploads the file "test-image.exe"
    And the user clicks the upload button
    Then the error message "File type not supported" should appear

  @negative
  Scenario: Upload fails with oversized file
    When the user uploads the file "large-file-25mb.pdf"
    And the user clicks the upload button
    Then the error message "File size exceeds the 10MB limit" should appear
```

---

## File Download

### The Challenge

When Selenium clicks a download link or button, the browser opens a native download dialog or silently downloads to the default folder. Selenium cannot interact with the native OS save dialog, and it cannot introspect the browser's download manager. The solution is to configure the browser to download files automatically to a controlled directory, then verify the downloaded file's presence and content from that directory.

---

### Chrome Download Configuration

Configure Chrome to automatically download files to a specific directory without showing a save dialog.

```java
public static WebDriver createChromeDriverWithDownloads(String downloadDir) {
    // Ensure the directory exists
    new File(downloadDir).mkdirs();

    Map<String, Object> prefs = new HashMap<>();
    prefs.put("download.default_directory", downloadDir);
    prefs.put("download.prompt_for_download", false);   // no save dialog
    prefs.put("download.directory_upgrade", true);
    prefs.put("safebrowsing.enabled", true);
    prefs.put("plugins.always_open_pdf_externally", true); // download PDF instead of preview

    ChromeOptions options = new ChromeOptions();
    options.setExperimentalOption("prefs", prefs);
    options.addArguments("--start-maximized");

    WebDriverManager.chromedriver().setup();
    return new ChromeDriver(options);
}
```

---

### Firefox Download Configuration

```java
public static WebDriver createFirefoxDriverWithDownloads(String downloadDir) {
    new File(downloadDir).mkdirs();

    FirefoxProfile profile = new FirefoxProfile();
    profile.setPreference("browser.download.folderList", 2); // custom dir
    profile.setPreference("browser.download.dir", downloadDir);
    profile.setPreference("browser.download.useDownloadDir", true);
    profile.setPreference("browser.helperApps.neverAsk.saveToDisk",
        "application/pdf,application/octet-stream,text/csv,application/zip," +
        "application/vnd.ms-excel,application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
    profile.setPreference("pdfjs.disabled", true);           // download PDFs
    profile.setPreference("browser.download.manager.showWhenStarting", false);

    FirefoxOptions options = new FirefoxOptions();
    options.setProfile(profile);

    WebDriverManager.firefoxdriver().setup();
    return new FirefoxDriver(options);
}
```

---

### DownloadUtils — Waiting and Verifying Downloads

Downloads are asynchronous. The file appears in the download directory only after the download completes. Poll the directory until the file appears and has finished writing (Chrome writes `.crdownload` files while in progress).

```java
package com.company.framework.utils;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.io.File;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.time.Duration;
import java.time.Instant;
import java.util.Arrays;
import java.util.Optional;

public class DownloadUtils {

    private static final Logger log = LogManager.getLogger(DownloadUtils.class);
    private static final int DEFAULT_WAIT_SECONDS = 30;
    private static final int POLL_INTERVAL_MS = 500;

    private DownloadUtils() {}

    /**
     * Waits for a file matching the given name to appear and finish downloading.
     * Returns the File object when ready.
     */
    public static File waitForDownload(String downloadDir, String expectedFileName) {
        return waitForDownload(downloadDir, expectedFileName, DEFAULT_WAIT_SECONDS);
    }

    public static File waitForDownload(String downloadDir, String expectedFileName,
                                        int timeoutSeconds) {
        log.info("Waiting for download: {} in {}", expectedFileName, downloadDir);
        Instant deadline = Instant.now().plus(Duration.ofSeconds(timeoutSeconds));

        while (Instant.now().isBefore(deadline)) {
            File[] files = new File(downloadDir).listFiles();
            if (files != null) {
                for (File file : files) {
                    String name = file.getName();
                    // Skip Chrome's in-progress marker
                    if (name.endsWith(".crdownload") || name.endsWith(".part")) continue;

                    if (name.equals(expectedFileName) && file.length() > 0) {
                        log.info("Download complete: {} ({} bytes)", name, file.length());
                        return file;
                    }
                }
            }

            try { Thread.sleep(POLL_INTERVAL_MS); }
            catch (InterruptedException e) { Thread.currentThread().interrupt(); break; }
        }

        throw new RuntimeException(
            "Download did not complete within " + timeoutSeconds + "s. " +
            "Expected: " + expectedFileName + " in " + downloadDir
        );
    }

    /**
     * Waits for any new file to appear (useful when exact name is unknown).
     * Takes a snapshot before the action and returns the new file that appeared.
     */
    public static File waitForNewDownload(String downloadDir, File[] filesBefore,
                                           int timeoutSeconds) {
        Instant deadline = Instant.now().plus(Duration.ofSeconds(timeoutSeconds));

        while (Instant.now().isBefore(deadline)) {
            File[] filesNow = new File(downloadDir).listFiles();
            if (filesNow != null) {
                for (File file : filesNow) {
                    if (file.getName().endsWith(".crdownload")) continue;
                    if (file.getName().endsWith(".part")) continue;
                    if (file.length() == 0) continue;

                    boolean isNew = Arrays.stream(filesBefore)
                        .noneMatch(old -> old.getName().equals(file.getName()));

                    if (isNew) {
                        log.info("New download detected: {}", file.getName());
                        return file;
                    }
                }
            }
            try { Thread.sleep(POLL_INTERVAL_MS); }
            catch (InterruptedException e) { Thread.currentThread().interrupt(); break; }
        }

        throw new RuntimeException("No new download appeared within " + timeoutSeconds + "s");
    }

    /**
     * Clears all files in the download directory before a test.
     */
    public static void clearDownloadDirectory(String downloadDir) {
        File dir = new File(downloadDir);
        if (dir.exists()) {
            File[] files = dir.listFiles();
            if (files != null) {
                for (File file : files) {
                    boolean deleted = file.delete();
                    if (deleted) log.debug("Deleted: {}", file.getName());
                }
            }
        }
        dir.mkdirs();
        log.info("Download directory cleared: {}", downloadDir);
    }

    /**
     * Returns true if a file with the given name exists in the directory.
     */
    public static boolean fileExists(String downloadDir, String fileName) {
        return new File(downloadDir + "/" + fileName).exists();
    }

    /**
     * Returns the size of a downloaded file in bytes.
     */
    public static long getFileSize(String downloadDir, String fileName) {
        return new File(downloadDir + "/" + fileName).length();
    }

    /**
     * Reads the first N bytes of a downloaded file as a string (for CSV/text validation).
     */
    public static String readFileContent(String downloadDir, String fileName) throws Exception {
        Path path = Paths.get(downloadDir, fileName);
        return new String(Files.readAllBytes(path));
    }
}
```

---

### Using DownloadUtils in Tests

```java
package stepdefinitions;

import com.company.framework.utils.DownloadUtils;
import io.cucumber.java.en.Then;
import io.cucumber.java.en.When;
import org.testng.Assert;

import java.io.File;

public class DownloadSteps {

    private static final String DOWNLOAD_DIR = System.getProperty("user.dir")
        + "/target/downloads";

    private File[] filesBeforeDownload;

    @When("the user clicks the export button")
    public void theUserClicksExportButton() {
        // Snapshot before the download starts
        filesBeforeDownload = new File(DOWNLOAD_DIR).listFiles();
        if (filesBeforeDownload == null) filesBeforeDownload = new File[0];
        context.getReportsPage().clickExport();
    }

    @Then("the file {string} should be downloaded")
    public void theFileShouldBeDownloaded(String expectedFileName) {
        File downloaded = DownloadUtils.waitForDownload(DOWNLOAD_DIR, expectedFileName);
        Assert.assertTrue(downloaded.exists(),
            "Expected file not found: " + expectedFileName);
        Assert.assertTrue(downloaded.length() > 0,
            "Downloaded file is empty: " + expectedFileName);
    }

    @Then("a CSV file should be downloaded")
    public void aCSVFileShouldBeDownloaded() {
        File downloaded = DownloadUtils.waitForNewDownload(
            DOWNLOAD_DIR, filesBeforeDownload, 30);
        Assert.assertTrue(downloaded.getName().endsWith(".csv"),
            "Expected CSV but got: " + downloaded.getName());
    }

    @Then("the downloaded CSV should contain {string}")
    public void theDownloadedCSVShouldContain(String expectedContent) throws Exception {
        File downloaded = DownloadUtils.waitForNewDownload(
            DOWNLOAD_DIR, filesBeforeDownload, 30);
        String content = DownloadUtils.readFileContent(
            DOWNLOAD_DIR, downloaded.getName());
        Assert.assertTrue(content.contains(expectedContent),
            "CSV does not contain expected content: " + expectedContent);
    }
}
```

---

### Headless Download Configuration

Downloads in headless mode require additional Chrome flags to enable the download directory:

```java
public static WebDriver createHeadlessChromeWithDownloads(String downloadDir) {
    ChromeOptions options = new ChromeOptions();
    options.addArguments("--headless=new");
    options.addArguments("--window-size=1920,1080");
    options.addArguments("--no-sandbox");
    options.addArguments("--disable-dev-shm-usage");

    Map<String, Object> prefs = new HashMap<>();
    prefs.put("download.default_directory", new File(downloadDir).getAbsolutePath());
    prefs.put("download.prompt_for_download", false);
    prefs.put("download.directory_upgrade", true);
    prefs.put("safebrowsing.enabled", false);
    options.setExperimentalOption("prefs", prefs);

    WebDriverManager.chromedriver().setup();
    ChromeDriver driver = new ChromeDriver(options);

    // Additional CDP command needed for headless downloads in older Chrome versions
    Map<String, Object> params = new HashMap<>();
    params.put("behavior", "allow");
    params.put("downloadPath", new File(downloadDir).getAbsolutePath());
    driver.executeCdpCommand("Page.setDownloadBehavior", params);

    return driver;
}
```

---

## Summary

File upload is handled by calling `sendKeys()` directly on the `<input type="file">` element with the absolute file path — bypassing the OS dialog entirely. Hidden inputs are made visible via JavaScript before `sendKeys`. Drag-and-drop upload zones receive simulated HTML5 File API events via JavaScript. File downloads are handled by configuring the browser to save files automatically to a controlled directory, then polling that directory with `DownloadUtils` until the file appears and is complete. Both upload and download flows integrate cleanly into step definitions and verify the actual file system results.

---

## Related Topics

- JavaScriptExecutor.md — JS used for hidden inputs and drag-zone uploads
- HeadlessTesting.md — Special download configuration for headless mode
- CDP.md — Chrome DevTools Protocol for download interception
- Hooks.md — Clearing download directory in @Before hooks
