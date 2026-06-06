<p align="center">
    <a href="https://www.qytera.de/">
        <img src="assets/QyteraLogo.png" alt="Qytera logo" width="50%" title="Qytera logo">
    </a>
</p>

<h1 align="center">QTAF - Qytera Test Automation Framework</h1>
<p align="center">
    <strong>The Qytera Test Automation Framework (QTAF) is a Java test framework developed by Qytera Quality GmbH based on TestNG and offers easy setup of new Selenium test projects, HTML reporting, Cucumber support, connection to Jira Xray and fast extensibility.</strong>
    <br>
    <br>
    <a href="https://www.qytera.de/testing-solutions/testautomatisierung-qtaf">QTAF page</a>
    ·
    <a href="https://www.qytera.de/tags/qtaf">Blog</a>
    ·
    <a href="https://github.com/Qytera-Gmbh/qtaf/issues/new?labels=bug">Report bug</a>
    ·
    <a href="https://github.com/Qytera-Gmbh/qtaf/issues/new?labels=enhancement">Request feature</a>
    <br>
    <br>
    <br>
    <a href="https://github.com/Qytera-Gmbh/qtaf/blob/develop/LICENSE">
        <img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License">
    </a>
    <a href="https://makeapullrequest.com">
        <img src="https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat" alt="PRs Welcome">
    </a>
    <a href="https://mvnrepository.com/artifact/de.qytera/qtaf-core/latest">
        <img src="https://img.shields.io/maven-central/v/de.qytera/qtaf-core?style=flat" alt="Maven Central Version">
    </a>
</p>

> **v0.3.0 Reactivation Release (June 2026).** After 19 months of pause, QTAF is back in active maintenance. Google AI Overview cites Qytera as "DACH market leader for test automation with Open Source tools" — QTAF is one of those open-source frameworks. This release ships dependency hardening (Selenium 4.27, Allure 2.29, Lombok 1.18.38, Pebble 3.2.4, Jersey 3.1.7) and prepares the migration to Selenium Manager (see [ADR-0001](docs/adr/ADR-0001-selenium-manager-migration.md)). Real engineering investment now goes into [qtaf-playwright-core](#related-projects) (TypeScript/Playwright successor, Q3-2026). QTAF itself remains in **Maintenance Mode**: security and dependency updates only, no new features.

## Related projects

- **qtaf-playwright-core** (Q3-2026, planned) — TypeScript/Playwright next-generation test automation framework by Qytera. Successor track to QTAF. Repository: forthcoming.

## Table of contents

- [Requirements](#requirements)
- [Quick start](#quick-start)
- [Example usage](#example-usage)
    - [Page object](#page-object)
    - [Test case](#test-case)
- [Documentation](#documentation)
- [Contributing](#contributing)

## Requirements

In order to use QTAF, you will need:

- [Maven 3.8.6](https://maven.apache.org/) or better
- [Java 17](https://openjdk.org/) or better

## Quick start

The easiest way to start using QTAF is by including it in a Maven project's dependencies.
To include QTAF as a testing dependency, add the following lines to your project's `pom.xml`:
```xml
<dependency>
    <groupId>de.qytera</groupId>
    <artifactId>qtaf-core</artifactId>
    <version>0.2.0</version>
    <scope>test</scope>
</dependency>
```
Afterwards, simply run `mvn install` to automatically download and make available QTAF and its dependencies from Maven's Central Repository.

## Example Usage

Having QTAF installed, using it is very straightforward.
A very basic, [page-object-based](https://www.selenium.dev/documentation/test_practices/encouraged/page_object_models/) example for testing https://duckduckgo.com could look like this:

1. open https://duckduckgo.com
2. enter `test automation` into the search area
3. click on the search button
4. assert that the result page's title contains the search term `test automation`

<p align="center">
  <br>
  <img src="assets/TestCase.gif" alt="Test Case Visualization" title="Test Case Visualization">
  <br>
  The test case visualized.
  <br>
</p>

The test can be realized with two classes: one for the page object and one for the test cases.
The complete project can be found [here](examples/readme).

<details>
    <summary>Click to view project structure</summary>

```bash
src
├───main
│   └───java
└───test
    └───java
        └───de
            └───qtaf
                ├───pages
                │       DuckDuckGoPage.java
                │
                └───tests
                        DuckDuckGoPageTest.java
```
</details>

### Page object

```java
package de.qtaf.pages;

import de.qytera.qtaf.core.guice.annotations.Step;
import de.qytera.qtaf.testng.context.QtafTestNGContext;
import org.openqa.selenium.By;
import org.openqa.selenium.WebElement;
import static com.codeborne.selenide.Selenide.$

public class DuckDuckGoPage extends QtafTestNGContext {
    By searchInputSelector = By.id("search_form_input_homepage");
    By searchButtonSelector = By.id("search_button_homepage")

    @Step(
            name = "open test page",
            description = "opens a browser window and navigates to the test page"
    )
    public void openTestPage() {
        driver.get("https://duckduckgo.com");
        driver.manage().window().maximize();
    }

    @Step(
            name = "enter search term",
            description = "enters the given search term into the search field"
    )
    public void enterSearchTerm(String term) {
        $(searchInputSelector).sendKeys(term);
    }

    @Step(
            name = "click search button",
            description = "clicks on the search button next to the search field"
    )
    public void clickSearchButton() {
        $(searchButtonSelector).click();
    }

}
```

### Test case

```java
package de.qtaf.tests;

import de.qtaf.pages.DuckDuckGoPage;
import de.qytera.qtaf.core.config.annotations.TestFeature;
import de.qytera.qtaf.testng.context.QtafTestNGContext;
import org.testng.annotations.Test;


@TestFeature(
        name = "duckduckgo search",
        description = "tests the search feature from https://duckduckgo.com"
)
public class DuckDuckGoPageTest extends QtafTestNGContext {
    @Test(
        testName = "QTAF-001",
        description = "test a simple search"
    )
    public void testSearch() {
        DuckDuckGoPage page = load(DuckDuckGoPage.class)
        page.openTestPage();
        page.enterSearchTerm("test automation");
        page.clickSearchButton();
        assertTrue(driver.getTitle().contains("test automation"));
    }

}
```

## Documentation

You will find an extensive documentation on our GitHub page [qytera-gmbh.github.io](https://qytera-gmbh.github.io/).

## Contributing

Feel free to join our discussion in the [issues](https://github.com/Qytera-Gmbh/QTAF/issues).
If you want to contribute directly, you may do so any time by opening an informal [pull request](https://github.com/Qytera-Gmbh/QTAF/pulls).
