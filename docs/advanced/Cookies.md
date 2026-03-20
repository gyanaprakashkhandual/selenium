# Cookies Management

## Overview

Cookies are small data fragments stored by the browser on behalf of a website. In web application testing, cookies are used to persist session state, store authentication tokens, remember user preferences, track analytics, and manage consent. Selenium WebDriver provides a complete cookie management API that allows tests to read, add, modify, and delete cookies without going through the UI — enabling powerful techniques such as bypassing login forms, testing cookie consent behavior, and validating that security cookies are set correctly.

---

## Selenium Cookie API

All cookie operations are accessed through `driver.manage()`, which returns an `Options` object with cookie management methods.

| Method                           | Description                                         |
| -------------------------------- | --------------------------------------------------- |
| `addCookie(Cookie cookie)`       | Adds a cookie to the current domain                 |
| `getCookieNamed(String name)`    | Returns a cookie by name, or null                   |
| `getCookies()`                   | Returns all cookies for the current domain as a Set |
| `deleteCookie(Cookie cookie)`    | Deletes a specific cookie object                    |
| `deleteCookieNamed(String name)` | Deletes a cookie by name                            |
| `deleteAllCookies()`             | Deletes all cookies for the current domain          |

---

## The Cookie Class

Selenium's `Cookie` class represents a single browser cookie. It mirrors the attributes of a real HTTP cookie.

```java
import org.openqa.selenium.Cookie;

// Minimal constructor — name and value only
Cookie simple = new Cookie("username", "admin");

// Using the Builder for full control
Cookie full = new Cookie.Builder("auth_token", "eyJhbGciOiJIUzI1NiJ9...")
    .domain("example.com")
    .path("/")
    .expiresOn(new java.util.Date(System.currentTimeMillis() + 3600_000)) // 1 hour
    .isSecure(true)
    .isHttpOnly(true)
    .build();
```

### Cookie Attributes

| Attribute | Builder Method                    | Purpose                                 |
| --------- | --------------------------------- | --------------------------------------- |
| Name      | `new Cookie.Builder(name, value)` | Cookie identifier                       |
| Value     | `new Cookie.Builder(name, value)` | Cookie data                             |
| Domain    | `.domain(String)`                 | Scope to a specific domain              |
| Path      | `.path(String)`                   | Scope to a URL path                     |
| Expiry    | `.expiresOn(Date)`                | Expiration date (null = session cookie) |
| Secure    | `.isSecure(boolean)`              | HTTPS-only flag                         |
| HttpOnly  | `.isHttpOnly(boolean)`            | Inaccessible to JavaScript              |

---

## Reading Cookies

### Get a Specific Cookie

```java
public Cookie getCookieByName(String name) {
    Cookie cookie = driver.manage().getCookieNamed(name);
    if (cookie == null) {
        throw new RuntimeException("Cookie not found: " + name);
    }
    return cookie;
}

// Usage
Cookie sessionCookie = getCookieByName("JSESSIONID");
System.out.println("Session ID: " + sessionCookie.getValue());
System.out.println("Secure: "    + sessionCookie.isSecure());
System.out.println("HttpOnly: "  + sessionCookie.isHttpOnly());
System.out.println("Expires: "   + sessionCookie.getExpiry());
```

### Get All Cookies

```java
public Map<String, String> getAllCookiesAsMap() {
    return driver.manage().getCookies()
        .stream()
        .collect(Collectors.toMap(Cookie::getName, Cookie::getValue));
}

// Log all cookies for debugging
public void logAllCookies() {
    Set<Cookie> cookies = driver.manage().getCookies();
    log.info("Total cookies: {}", cookies.size());
    cookies.forEach(c ->
        log.debug("Cookie: {} = {} [domain={}, path={}, secure={}, httpOnly={}]",
            c.getName(), c.getValue(), c.getDomain(),
            c.getPath(), c.isSecure(), c.isHttpOnly())
    );
}

// Check if a cookie exists
public boolean cookieExists(String name) {
    return driver.manage().getCookieNamed(name) != null;
}
```

---

## Adding Cookies

The browser must be on the target domain before adding a cookie. Selenium restricts cookie additions to the domain of the page currently loaded.

```java
// Always navigate to the domain first
driver.get("https://example.com");

// Then add the cookie
Cookie authCookie = new Cookie.Builder("auth_token", "my-jwt-token-value")
    .domain("example.com")
    .path("/")
    .isSecure(true)
    .isHttpOnly(true)
    .build();

driver.manage().addCookie(authCookie);

// Now navigate to the authenticated page
driver.get("https://example.com/dashboard");
```

---

## Bypassing Login with Cookie Injection

Injecting a session cookie is the fastest way to reach an authenticated state without going through the UI login form. This reduces test execution time significantly when many scenarios require an authenticated starting point.

### Step 1 — Obtain the Session Cookie via API Login

```java
package utils;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

import java.util.Map;

public class ApiAuthHelper {

    private static final ObjectMapper mapper = new ObjectMapper();

    public static String getSessionToken(String baseUrl, String email, String password)
            throws Exception {

        try (CloseableHttpClient client = HttpClients.createDefault()) {
            HttpPost post = new HttpPost(baseUrl + "/api/v1/auth/login");
            post.setHeader("Content-Type", "application/json");

            String body = "{\"email\":\"" + email + "\",\"password\":\"" + password + "\"}";
            post.setEntity(new StringEntity(body));

            try (CloseableHttpResponse response = client.execute(post)) {
                String responseBody = EntityUtils.toString(response.getEntity());
                Map<String, Object> json = mapper.readValue(responseBody, Map.class);
                return (String) json.get("token");
            }
        }
    }
}
```

### Step 2 — Inject the Cookie Before Navigating

```java
public DashboardPage loginViaCookie(String token) {
    // Navigate to the domain (any page — just to set the domain context)
    driver.get(ConfigReader.get("app.base.url"));

    // Delete any existing auth cookies
    driver.manage().deleteCookieNamed("auth_token");
    driver.manage().deleteCookieNamed("JSESSIONID");

    // Add the session cookie
    Cookie sessionCookie = new Cookie.Builder("auth_token", token)
        .domain(extractDomain(ConfigReader.get("app.base.url")))
        .path("/")
        .isSecure(true)
        .isHttpOnly(true)
        .build();

    driver.manage().addCookie(sessionCookie);

    // Now navigate directly to the authenticated page
    driver.get(ConfigReader.get("app.base.url") + "/dashboard");
    return new DashboardPage(driver);
}

private String extractDomain(String url) {
    // "https://dev.example.com" → "dev.example.com"
    return url.replace("https://", "").replace("http://", "").split("/")[0];
}
```

### Step 3 — Use in Hooks for @authenticated Scenarios

```java
@Before("@authenticated")
public void loginViaCookie(Scenario scenario) throws Exception {
    String baseUrl = ConfigReader.get("app.base.url");
    String email   = ConfigReader.get("test.admin.username");
    String password = ConfigReader.get("test.admin.password");

    // Get token via API — fast, no browser form submission
    String token = ApiAuthHelper.getSessionToken(baseUrl, email, password);

    // Navigate to the base URL first to set domain context
    context.getDriver().get(baseUrl);

    // Inject the session cookie
    Cookie sessionCookie = new Cookie.Builder("auth_token", token)
        .domain(baseUrl.replaceAll("https?://", "").split("/")[0])
        .path("/")
        .isSecure(true)
        .build();

    context.getDriver().manage().addCookie(sessionCookie);
    log.info("Session cookie injected for: {}", email);
}
```

---

## Saving and Restoring Browser State

Save all cookies after login and restore them later — avoiding repeated login in long test sequences.

```java
// Save all cookies to a file after login
public void saveCookiesToFile(String filePath) throws Exception {
    Set<Cookie> cookies = driver.manage().getCookies();
    ObjectMapper mapper = new ObjectMapper();

    List<Map<String, Object>> cookieList = cookies.stream().map(c -> {
        Map<String, Object> map = new HashMap<>();
        map.put("name", c.getName());
        map.put("value", c.getValue());
        map.put("domain", c.getDomain());
        map.put("path", c.getPath());
        map.put("secure", c.isSecure());
        map.put("httpOnly", c.isHttpOnly());
        if (c.getExpiry() != null) {
            map.put("expiry", c.getExpiry().getTime());
        }
        return map;
    }).collect(Collectors.toList());

    mapper.writeValue(new File(filePath), cookieList);
    log.info("Saved {} cookies to {}", cookieList.size(), filePath);
}

// Restore cookies from file
@SuppressWarnings("unchecked")
public void restoreCookiesFromFile(String filePath) throws Exception {
    ObjectMapper mapper = new ObjectMapper();
    List<Map<String, Object>> cookieList = mapper.readValue(
        new File(filePath), List.class);

    driver.manage().deleteAllCookies();

    for (Map<String, Object> c : cookieList) {
        Cookie.Builder builder = new Cookie.Builder(
            (String) c.get("name"), (String) c.get("value"))
            .domain((String) c.get("domain"))
            .path((String) c.get("path"))
            .isSecure((Boolean) c.getOrDefault("secure", false))
            .isHttpOnly((Boolean) c.getOrDefault("httpOnly", false));

        if (c.containsKey("expiry")) {
            builder.expiresOn(new java.util.Date((Long) c.get("expiry")));
        }

        driver.manage().addCookie(builder.build());
    }

    log.info("Restored {} cookies from {}", cookieList.size(), filePath);
    driver.navigate().refresh();
}
```

---

## Deleting Cookies

```java
// Delete a specific cookie by name
driver.manage().deleteCookieNamed("auth_token");

// Delete a specific cookie object
Cookie cookie = driver.manage().getCookieNamed("JSESSIONID");
if (cookie != null) {
    driver.manage().deleteCookie(cookie);
}

// Delete all cookies — full session reset
driver.manage().deleteAllCookies();
```

**In a logout test:**

```java
@Then("the session cookie should be cleared after logout")
public void sessionCookieShouldBeCleared() {
    Cookie sessionCookie = driver.manage().getCookieNamed("auth_token");
    Assert.assertNull(sessionCookie,
        "auth_token cookie still exists after logout");
}
```

---

## Security Cookie Assertions

Validating cookie security attributes is an important part of security testing.

```java
@Then("the authentication cookie should be secure and HttpOnly")
public void authCookieShouldBeSecureAndHttpOnly() {
    Cookie authCookie = driver.manage().getCookieNamed("auth_token");

    Assert.assertNotNull(authCookie, "auth_token cookie was not set");
    Assert.assertTrue(authCookie.isSecure(),
        "auth_token cookie is not marked Secure — transmitted over HTTP");
    Assert.assertTrue(authCookie.isHttpOnly(),
        "auth_token cookie is not HttpOnly — accessible via JavaScript");
    Assert.assertNotNull(authCookie.getExpiry(),
        "auth_token cookie has no expiry — it is a persistent session cookie");
}

@Then("the session cookie should expire within {int} hours")
public void sessionCookieShouldExpireWithinHours(int hours) {
    Cookie sessionCookie = driver.manage().getCookieNamed("JSESSIONID");
    Assert.assertNotNull(sessionCookie, "Session cookie not found");
    Assert.assertNotNull(sessionCookie.getExpiry(), "Session cookie has no expiry");

    long maxExpiryMs = System.currentTimeMillis() + (hours * 3600_000L);
    Assert.assertTrue(
        sessionCookie.getExpiry().getTime() <= maxExpiryMs,
        "Session cookie expiry exceeds " + hours + " hour(s)"
    );
}
```

---

## Cookie Consent Testing

```gherkin
Feature: Cookie Consent

  Scenario: Cookie banner appears on first visit
    Given the user has no cookies set
    When the user visits the home page
    Then the cookie consent banner should be displayed

  Scenario: Accepting cookies sets the consent cookie
    Given the cookie consent banner is displayed
    When the user clicks "Accept All Cookies"
    Then a cookie named "cookie_consent" should exist
    And the cookie "cookie_consent" should have value "accepted"
    And the banner should no longer be displayed

  Scenario: Declining cookies sets the rejection cookie
    Given the cookie consent banner is displayed
    When the user clicks "Reject Non-Essential"
    Then a cookie named "cookie_consent" should exist
    And the cookie "cookie_consent" should have value "rejected"
```

```java
@Given("the user has no cookies set")
public void theUserHasNoCookiesSet() {
    context.getDriver().get(ConfigReader.get("app.base.url"));
    context.getDriver().manage().deleteAllCookies();
    context.getDriver().navigate().refresh();
}

@Then("a cookie named {string} should exist")
public void aCookieNamedShouldExist(String cookieName) {
    Cookie cookie = context.getDriver().manage().getCookieNamed(cookieName);
    Assert.assertNotNull(cookie, "Cookie '" + cookieName + "' was not found");
}

@Then("the cookie {string} should have value {string}")
public void theCookieShouldHaveValue(String name, String expectedValue) {
    Cookie cookie = context.getDriver().manage().getCookieNamed(name);
    Assert.assertNotNull(cookie, "Cookie not found: " + name);
    Assert.assertEquals(cookie.getValue(), expectedValue,
        "Cookie '" + name + "' value mismatch");
}
```

---

## Summary

Selenium's cookie API provides complete control over browser cookies — reading, adding, modifying, and deleting individual cookies or entire sessions. The most impactful use of cookie management in a test framework is bypassing the UI login form by injecting a session cookie obtained via API, which dramatically reduces scenario setup time. Cookie management is also essential for testing logout behavior, verifying security attributes (Secure, HttpOnly, expiry), validating cookie consent flows, and saving and restoring browser state across test sessions.

---

## Related Topics

- Hooks.md — @Before hook with cookie injection for @authenticated scenarios
- JavaScriptExecutor.md — Reading document.cookie and localStorage via JS
- HeadlessTesting.md — Cookie behavior in headless browsers
- CDP.md — Chrome DevTools Protocol for advanced cookie management and network interception
