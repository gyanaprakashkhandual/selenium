# Selenium Complete Learning Repository

A comprehensive, open-source repository designed to take learners from beginner to advanced Selenium automation testing. Built with Java and Maven, integrated with Cucumber, Page Object Model, custom reporting, and CI/CD pipelines.

---

## Overview

This repository is a structured learning path for anyone who wants to master Selenium WebDriver. Whether you are a beginner just starting out or an experienced tester looking to adopt best practices, this repository has everything you need in one place.

The repository contains a fully functional portfolio project built on a real-world web application — **kindofacast.personal.app** — demonstrating how professional automation frameworks are built and maintained in production environments.

---

## What This Repository Covers

### Core Technology Stack

- **Language:** Java
- **Build Tool:** Maven
- **Testing Framework:** Selenium WebDriver
- **BDD Framework:** Cucumber
- **Design Pattern:** Page Object Model (POM)
- **Reporting:** Extent Reports with custom report templates
- **CI/CD:** Jenkins and GitHub Actions (X)

### Topics Covered

**Beginner**

- Selenium WebDriver setup and configuration
- Browser drivers and WebDriverManager
- Locators: ID, Name, XPath, CSS Selector, Class Name, Tag Name
- Basic interactions: click, type, navigate, and assertions
- Maven project structure and dependency management

**Intermediate**

- Page Object Model design pattern
- Cucumber BDD with Feature files and Step Definitions
- TestNG and JUnit integration
- Data-driven testing with external data sources
- Handling alerts, frames, and multiple windows
- Waits: Implicit, Explicit, and Fluent

**Advanced**

- Cucumber hooks and tag-based execution
- Custom Extent Reports and reporting utilities
- Parallel test execution
- Cross-browser testing
- Headless browser execution
- Jenkins pipeline setup and configuration
- GitHub Actions CI/CD workflow
- Environment configuration management
- Retry logic and flaky test handling

---

## Repository Structure

```
selenium/
├── test/
│   ├── src/
│   │   ├── main/
│   │   │   └── java/
│   │   │       ├── pages/           # Page Object classes
│   │   │       ├── utils/           # Utility and helper classes
│   │   │       └── config/          # Configuration management
│   │   └── test/
│   │       ├── java/
│   │       │   ├── steps/           # Cucumber step definitions
│   │       │   ├── hooks/           # Cucumber hooks (before/after)
│   │       │   └── runners/         # Test runners
│   │       └── resources/
│   │           └── features/        # Cucumber feature files
│   └── pom.xml
├── docs/                            # Full documentation in Markdown format
├── README.md
├── CODE_OF_CONDUCT.md
└── LICENSE
```

---

## Portfolio Project

The portfolio project inside this repository automates the web application **gyanprakash.vercel.app**. It is a complete reference implementation that demonstrates:

- Real-world Page Object Model implementation
- Clean and maintainable code architecture
- Proper separation of concerns across all layers
- Cucumber feature files written in plain, readable English
- Custom reporting integrated into the full test lifecycle
- CI/CD pipeline configuration ready for Jenkins and GitHub Actions

---

## Documentation

All documentation is written in Markdown format and covers every topic from basic to advanced. The `docs/` folder in this repository contains structured guides aligned with the topics covered in the portfolio project.

**Read the full documentation online:**

[https://gyanprakash.vercel.app/docs/selenium](https://gyanprakash.vercel.app/docs/selenium)

Topics in the documentation include:

- Installation and environment setup
- Framework architecture and design decisions
- Cucumber and BDD fundamentals
- Page Object Model deep dive
- Writing stable and maintainable locators
- Reporting setup and customization
- Jenkins pipeline configuration step by step
- GitHub Actions workflow setup
- Best practices and code standards

---

## Getting Started

### Prerequisites

Ensure the following are installed before running the project:

- Java JDK 11 or higher
- Apache Maven 3.6 or higher
- Google Chrome or Mozilla Firefox (latest stable)
- Git

### Clone the Repository

```bash
git clone https://github.com/gyanaprakashkhandual/selenium.git
cd selenium
```

### Navigate to the Project

```bash
cd test
```

### Install Dependencies

```bash
mvn clean install -DskipTests
```

### Run All Tests

```bash
mvn clean test
```

### Run Tests by Cucumber Tag

```bash
mvn clean test -Dcucumber.filter.tags="@smoke"
mvn clean test -Dcucumber.filter.tags="@regression"
mvn clean test -Dcucumber.filter.tags="@sanity"
```

### Generate Report

```bash
mvn clean test -Preport
```

---

## Browser Configuration

```bash
# Chrome (default)
mvn clean test -Dbrowser=chrome

# Firefox
mvn clean test -Dbrowser=firefox

# Headless Chrome
mvn clean test -Dbrowser=chrome-headless

# Headless Firefox
mvn clean test -Dbrowser=firefox-headless
```

---

## CI/CD Integration

### Jenkins

A `Jenkinsfile` is included in the project. To use it:

1. Create a new Pipeline job in Jenkins.
2. Connect it to this repository.
3. Jenkins will automatically detect the `Jenkinsfile` and execute the pipeline.

### GitHub Actions

A workflow file is provided under `.github/workflows/`. It triggers test execution on every push and pull request to the `main` branch.

---

## Best Practices Followed

- Strict separation of test logic, page objects, and utility classes
- No hard-coded values — all configuration managed through properties files
- Descriptive and readable Cucumber scenarios written in plain English
- Reusable and atomic step definitions
- Explicit waits preferred over implicit waits to reduce flakiness
- Clean browser teardown after every test execution
- Meaningful assertion messages for easier debugging
- Consistent naming conventions across the entire codebase
- Screenshot capture on failure for all test runs

---

## Who Is This For

- **Beginners** who want to learn Selenium from scratch with no prior experience
- **Students** preparing for SDET, QA Engineer, or Test Automation roles
- **Testers** looking to adopt Cucumber and POM in existing projects
- **Developers** who want to understand professional automation testing frameworks

---

## Contributing

Contributions are welcome and appreciated. Please read the [Code of Conduct](CODE_OF_CONDUCT.md) before contributing.

To contribute:

1. Fork the repository.
2. Create a new branch: `git checkout -b feature/your-feature-name`
3. Make your changes and commit: `git commit -m "Add: describe your change"`
4. Push to your branch: `git push origin feature/your-feature-name`
5. Open a Pull Request with a clear description of what was changed and why.

Please ensure your code follows the existing style and that all tests pass before submitting a Pull Request.

---

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for full terms.

This repository is completely free and open source. You are free to use, copy, modify, merge, publish, distribute, sublicense, or sell copies of this work, provided you include the original license notice.

---

## Author

**Gyana Prakash Khandual**

- GitHub: [gyanaprakashkhandual](https://github.com/gyanaprakashkhandual)
- Documentation: [gyanprakash.vercel.app/docs/selenium](https://gyanprakash.vercel.app/docs/selenium)

---

## Acknowledgements

This repository is built with the intent to make Selenium automation accessible to everyone. If you find it helpful, consider giving it a star on GitHub and sharing it with others in the testing and quality assurance community.
