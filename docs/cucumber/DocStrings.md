# Doc Strings

## Overview

Doc Strings are a Gherkin construct for passing multi-line text content into a step definition. They are enclosed between triple double-quote delimiters `"""` and can span any number of lines. Doc Strings are ideal when a step needs to receive a block of text — a JSON payload, an email body, an SQL query, an HTML snippet, an error message template, or any content that is too long or too structured for an inline step parameter.

---

## Syntax

```gherkin
When the user submits a support ticket with the message:
  """
  I am unable to access my account since the password reset.
  The reset link in the email does not work.
  I have tried three times. Please escalate urgently.
  """
```

The content between the `"""` delimiters is passed verbatim to the step definition as a `String`. The leading indentation is stripped to the level of the opening `"""`, so the content is not over-indented.

The closing `"""` must be on its own line at the same indentation level as the opening `"""`.

---

## Step Definition Implementation

The step definition receives the Doc String content as a `String` parameter. The parameter is automatically appended after any inline parameters extracted from the step text.

```java
import io.cucumber.java.en.When;

@When("the user submits a support ticket with the message:")
public void theUserSubmitsTicketWithMessage(String messageContent) {
    supportPage.fillMessageField(messageContent);
    supportPage.clickSubmit();
}
```

The `String messageContent` parameter receives the entire content between the triple quotes, including newlines but without the leading indentation.

---

## Typed Doc Strings

Cucumber supports typed Doc Strings using a content type annotation between the opening triple quotes and the content. The content type is a hint to both readers and to registered type transformers.

```gherkin
When the API request body is:
  """json
  {
    "username": "admin",
    "password": "secret",
    "rememberMe": true
  }
  """
```

The content type `json` does not cause automatic JSON parsing. You must register a `@DocStringType` transformer to enable that behavior.

---

## @DocStringType Transformer

Register a custom transformer to automatically convert the Doc String content to a specific Java type.

### JSON to Map Transformer

```java
package transformers;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.cucumber.java.DocStringType;

import java.util.Map;

public class DocStringTransformers {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @DocStringType
    public Map<String, Object> json(String content) throws Exception {
        return objectMapper.readValue(content, Map.class);
    }
}
```

With this transformer registered (the class must be in the glue path), a step definition can declare `Map<String, Object>` instead of `String`:

```java
@When("the API request is sent with body:")
public void theApiRequestIsSentWithBody(Map<String, Object> requestBody) {
    String username = (String) requestBody.get("username");
    boolean rememberMe = (boolean) requestBody.get("rememberMe");
    apiClient.login(username, (String) requestBody.get("password"), rememberMe);
}
```

### Custom POJO Transformer

```java
@DocStringType
public LoginRequest loginRequest(String content) throws Exception {
    return objectMapper.readValue(content, LoginRequest.class);
}
```

```java
@When("the user attempts to log in with:")
public void theUserAttemptsToLogIn(LoginRequest request) {
    loginPage.login(request.getUsername(), request.getPassword());
}
```

---

## Common Use Cases

### Verifying Email Content

```gherkin
Then the confirmation email body should contain:
  """
  Thank you for your order.
  Your order number is ORD-1001.
  Estimated delivery: 3-5 business days.
  """
```

```java
@Then("the confirmation email body should contain:")
public void confirmationEmailBodyShouldContain(String expectedContent) {
    String actualEmailBody = emailHelper.getLatestEmailBody("orders@example.com");

    String[] expectedLines = expectedContent.split("\n");
    for (String line : expectedLines) {
        Assert.assertTrue(
            actualEmailBody.contains(line.trim()),
            "Email body did not contain: " + line.trim()
        );
    }
}
```

### Sending an API Request with a JSON Body

```gherkin
When the system sends a notification with payload:
  """json
  {
    "type": "ORDER_SHIPPED",
    "orderId": "ORD-1001",
    "customerEmail": "buyer@example.com",
    "trackingNumber": "TRK-9876543"
  }
  """
```

```java
@When("the system sends a notification with payload:")
public void theSystemSendsNotification(String jsonPayload) {
    notificationService.send(jsonPayload);
}
```

### Submitting a Multi-Line Form Field

```gherkin
When the user enters the following in the "Comments" text area:
  """
  This is my first order and I am very happy with the product quality.
  The delivery was faster than expected.
  I will definitely order again.
  """
```

```java
@When("the user enters the following in the {string} text area:")
public void theUserEntersInTextArea(String fieldName, String content) {
    currentPage.fillTextArea(fieldName, content);
}
```

### Verifying Error Log Output

```gherkin
Then the application log should contain the entry:
  """
  ERROR: Payment gateway timeout after 30 seconds.
  Order ID: ORD-1001
  Retry attempt: 3 of 3
  Status: FAILED
  """
```

```java
@Then("the application log should contain the entry:")
public void applicationLogShouldContainEntry(String expectedLogEntry) {
    String logContent = LogHelper.readLatestLogFile();
    Assert.assertTrue(
        logContent.contains(expectedLogEntry.trim()),
        "Log entry not found.\nExpected:\n" + expectedLogEntry
            + "\n\nActual log:\n" + logContent
    );
}
```

### Inserting SQL for Test Data Setup

```gherkin
Given the database contains the following record:
  """sql
  INSERT INTO orders (id, customer_id, status, total)
  VALUES ('ORD-9001', 'CUST-101', 'PENDING', 149.97);
  """
```

```java
@Given("the database contains the following record:")
public void theDatabaseContainsRecord(String sql) {
    DatabaseHelper.execute(sql);
}
```

### Verifying Page Source or HTML Snippet

```gherkin
Then the page should contain the following HTML element:
  """html
  <div class="alert alert-success">
    Your order has been placed successfully.
  </div>
  """
```

```java
@Then("the page should contain the following HTML element:")
public void pageShouldContainHtmlElement(String expectedHtml) {
    String pageSource = Hooks.getDriver().getPageSource();
    // Normalize whitespace for comparison
    String normalizedSource = pageSource.replaceAll("\\s+", " ");
    String normalizedExpected = expectedHtml.replaceAll("\\s+", " ").trim();
    Assert.assertTrue(
        normalizedSource.contains(normalizedExpected),
        "Expected HTML not found in page source"
    );
}
```

---

## Indentation Behavior

Cucumber strips the leading indentation from Doc String content based on the position of the opening `"""`. The content preserves relative indentation within the block.

```gherkin
When the user pastes the following JSON:
  """
  {
    "key": "value",
    "nested": {
      "inner": "data"
    }
  }
  """
```

The step definition receives:

```
{
  "key": "value",
  "nested": {
    "inner": "data"
  }
}
```

The two-space indentation of the content relative to `"""` is removed. The relative indentation within the JSON is preserved.

---

## Doc Strings vs Data Tables

Both Doc Strings and Data Tables pass structured content to a step. Choose based on the nature of the data.

| Aspect            | Doc String                               | Data Table                  |
| ----------------- | ---------------------------------------- | --------------------------- |
| Structure         | Free-form multi-line text                | Tabular rows and columns    |
| Java type         | String (or custom via @DocStringType)    | DataTable, List, Map        |
| Best for          | JSON, XML, email bodies, SQL, plain text | Records, form fields, lists |
| Header row        | N/A                                      | Optional                    |
| Type-safe rows    | N/A                                      | Yes with asList(Pojo.class) |
| Human readability | High for text blocks                     | High for structured records |

---

## Best Practices

**Use Doc Strings for content, not data.** If the content is a structured set of field-value pairs, a Data Table is clearer. Doc Strings suit free-form text, message bodies, and code snippets.

**Keep the content realistic.** Doc String content is specification. It should contain realistic examples that the business would recognize — not placeholder text like "some message here."

**Use content type annotations for documentation.** Adding `"""json` or `"""sql` after the opening quotes signals the content format to readers, even if no transformer is registered.

**Trim the content in step definitions.** The Doc String may have a trailing newline depending on formatting. Call `.trim()` on the string if the step requires exact matching.

```java
@Then("the displayed message should be:")
public void theDisplayedMessageShouldBe(String expectedMessage) {
    String actual = messagePage.getMessageText().trim();
    String expected = expectedMessage.trim();
    Assert.assertEquals(actual, expected);
}
```

---

## Summary

Doc Strings provide a clean way to pass multi-line text content — JSON, email bodies, SQL, HTML, or plain text — directly into Cucumber step definitions. They preserve line breaks and relative indentation, support content type annotations for documentation, and can be automatically transformed into custom Java types using `@DocStringType`. When a step needs more than a single inline string value, Doc Strings are the appropriate mechanism.

---

## Related Topics

- DataTables.md — Structured tabular data for steps
- GherkinSyntax.md — Gherkin keyword and construct reference
- StepDefinitions.md — Step definition parameter handling
- CustomParameterTypes.md — Custom parameter type converters
