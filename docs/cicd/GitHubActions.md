# GitHub Actions Integration

## Overview

GitHub Actions is GitHub's built-in CI/CD platform that runs automated workflows directly from a repository. It requires no external server, no plugin management, and no infrastructure maintenance. Workflows are defined as YAML files in `.github/workflows/` and are triggered by repository events — pushes, pull requests, scheduled times, or manual dispatch. For Selenium Cucumber frameworks hosted on GitHub, Actions is the fastest path to a fully automated test pipeline.

This document covers complete workflow configurations for smoke testing on PRs, nightly regression, cross-browser matrix builds, Selenium Grid service containers, parallel execution, report publishing, and notification strategies.

---

## Workflow File Structure

All GitHub Actions workflows live in `.github/workflows/`. Create one file per workflow purpose:

```
.github/
  workflows/
    smoke-tests.yml        ← on every pull request
    nightly-regression.yml ← scheduled nightly run
    cross-browser.yml      ← matrix across browsers
    manual-run.yml         ← workflow_dispatch with parameters
```

---

## Smoke Test Workflow — On Every Pull Request

This workflow runs on every PR to `main` or `develop`, catching regressions before merge.

```yaml
# .github/workflows/smoke-tests.yml
name: Smoke Tests

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main, develop]

env:
  JAVA_VERSION: "11"
  MAVEN_OPTS: "-Xmx2048m"

jobs:
  smoke-tests:
    name: Smoke Tests — ${{ github.event_name }}
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set Up JDK ${{ env.JAVA_VERSION }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: "temurin"
          cache: "maven"

      - name: Install Google Chrome
        uses: browser-actions/setup-chrome@v1
        with:
          chrome-version: stable

      - name: Cache Maven Dependencies
        uses: actions/cache@v4
        with:
          path: ~/.m2/repository
          key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-maven-

      - name: Run Smoke Tests
        run: |
          mvn test \
            -Denv=staging \
            -Dbrowser=chrome \
            -Dheadless=true \
            -Dcucumber.filter.tags="@smoke" \
            -Dallure.results.directory=target/allure-results
        env:
          APP_BASE_URL: ${{ secrets.STAGING_BASE_URL }}
          TEST_ADMIN_USERNAME: ${{ secrets.STAGING_ADMIN_USERNAME }}
          TEST_ADMIN_PASSWORD: ${{ secrets.STAGING_ADMIN_PASSWORD }}

      - name: Publish JUnit Test Results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Smoke Test Results
          path: target/cucumber-reports/*.xml
          reporter: java-junit
          fail-on-error: false

      - name: Upload Allure Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: allure-results-smoke-${{ github.run_number }}
          path: target/allure-results/
          retention-days: 14

      - name: Upload Failure Screenshots
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: failure-screenshots-${{ github.run_number }}
          path: target/screenshots/
          retention-days: 7
```

---

## Nightly Regression Workflow — Scheduled

```yaml
# .github/workflows/nightly-regression.yml
name: Nightly Regression

on:
  schedule:
    # Run at 11:00 PM UTC every Monday through Friday
    - cron: "0 23 * * 1-5"
  # Allow manual trigger from GitHub UI
  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        required: true
        default: "staging"
        type: choice
        options: [staging, dev, uat]
      tags:
        description: "Cucumber tag expression"
        required: false
        default: "@regression and not @wip"
      thread_count:
        description: "Number of parallel threads"
        required: false
        default: "4"

env:
  JAVA_VERSION: "11"
  MAVEN_OPTS: "-Xmx3072m -XX:MaxMetaspaceSize=512m"

jobs:
  regression:
    name: Regression — ${{ github.event.inputs.environment || 'staging' }}
    runs-on: ubuntu-latest
    timeout-minutes: 90

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set Up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: "temurin"
          cache: "maven"

      - name: Install Chrome
        uses: browser-actions/setup-chrome@v1

      - name: Set Execution Parameters
        id: params
        run: |
          ENV="${{ github.event.inputs.environment || 'staging' }}"
          TAGS="${{ github.event.inputs.tags || '@regression and not @wip' }}"
          THREADS="${{ github.event.inputs.thread_count || '4' }}"
          echo "environment=$ENV"    >> $GITHUB_OUTPUT
          echo "tags=$TAGS"          >> $GITHUB_OUTPUT
          echo "thread_count=$THREADS" >> $GITHUB_OUTPUT
          echo "Running: env=$ENV | tags=$TAGS | threads=$THREADS"

      - name: Run Regression Tests
        run: |
          mvn test \
            -Denv=${{ steps.params.outputs.environment }} \
            -Dbrowser=chrome \
            -Dheadless=true \
            -Dthread.count=${{ steps.params.outputs.thread_count }} \
            -Dcucumber.filter.tags="${{ steps.params.outputs.tags }}"
        env:
          APP_BASE_URL: ${{ secrets.STAGING_BASE_URL }}
          TEST_ADMIN_USERNAME: ${{ secrets.STAGING_ADMIN_USERNAME }}
          TEST_ADMIN_PASSWORD: ${{ secrets.STAGING_ADMIN_PASSWORD }}
          TEST_BUYER_USERNAME: ${{ secrets.STAGING_BUYER_USERNAME }}
          TEST_BUYER_PASSWORD: ${{ secrets.STAGING_BUYER_PASSWORD }}

      - name: Generate Allure Report
        if: always()
        run: mvn allure:report -q

      - name: Publish JUnit Results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Regression Test Results
          path: target/cucumber-reports/*.xml
          reporter: java-junit
          fail-on-error: false

      - name: Upload Allure HTML Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: allure-report-${{ github.run_number }}
          path: target/allure-report/
          retention-days: 30

      - name: Upload Allure Results (for trend)
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: allure-results-${{ github.run_number }}
          path: target/allure-results/
          retention-days: 30

      - name: Upload Extent Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: extent-report-${{ github.run_number }}
          path: target/extent-reports/
          retention-days: 30

      - name: Upload Screenshots on Failure
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: screenshots-${{ github.run_number }}
          path: target/screenshots/

      - name: Notify Slack — Success
        if: success()
        uses: slackapi/slack-github-action@v1.25.0
        with:
          payload: |
            {
              "text": "✅ *Nightly Regression PASSED*",
              "attachments": [{
                "color": "good",
                "fields": [
                  {"title": "Environment", "value": "${{ steps.params.outputs.environment }}", "short": true},
                  {"title": "Run",         "value": "#${{ github.run_number }}",               "short": true},
                  {"title": "Report",      "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}", "short": false}
                ]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

      - name: Notify Slack — Failure
        if: failure()
        uses: slackapi/slack-github-action@v1.25.0
        with:
          payload: |
            {
              "text": "❌ *Nightly Regression FAILED*",
              "attachments": [{
                "color": "danger",
                "fields": [
                  {"title": "Environment", "value": "${{ steps.params.outputs.environment }}", "short": true},
                  {"title": "Run",         "value": "#${{ github.run_number }}",               "short": true},
                  {"title": "Logs",        "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}", "short": false}
                ]
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Cross-Browser Matrix Workflow

GitHub Actions matrix strategy runs the same job configuration across multiple browser and environment combinations in parallel.

```yaml
# .github/workflows/cross-browser.yml
name: Cross-Browser Tests

on:
  workflow_dispatch:
  schedule:
    - cron: "0 1 * * 6" # Every Saturday at 1 AM UTC

jobs:
  cross-browser:
    name: ${{ matrix.browser }} on ${{ matrix.env }}
    runs-on: ubuntu-latest
    timeout-minutes: 60

    strategy:
      fail-fast: false # Continue other matrix jobs even if one fails
      matrix:
        browser: [chrome, firefox]
        env: [staging, uat]
        include:
          # Add extra config for specific combinations
          - browser: chrome
            headless: true
          - browser: firefox
            headless: true

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set Up JDK
        uses: actions/setup-java@v4
        with:
          java-version: "11"
          distribution: "temurin"
          cache: "maven"

      - name: Install ${{ matrix.browser }}
        uses: browser-actions/setup-chrome@v1
        if: matrix.browser == 'chrome'

      - name: Install Firefox
        uses: browser-actions/setup-firefox@v1
        if: matrix.browser == 'firefox'

      - name: Run Tests — ${{ matrix.browser }} / ${{ matrix.env }}
        run: |
          mvn test \
            -Denv=${{ matrix.env }} \
            -Dbrowser=${{ matrix.browser }} \
            -Dheadless=true \
            -Dcucumber.filter.tags="@regression" \
            -Djson.output.file=target/cucumber-reports/${{ matrix.browser }}-${{ matrix.env }}.json \
            -Dsurefire.reportsDirectory=target/surefire-${{ matrix.browser }}-${{ matrix.env }}
        env:
          APP_BASE_URL: ${{ secrets[format('{0}_BASE_URL', matrix.env)] }}
          TEST_ADMIN_USERNAME: ${{ secrets.TEST_ADMIN_USERNAME }}
          TEST_ADMIN_PASSWORD: ${{ secrets.TEST_ADMIN_PASSWORD }}

      - name: Upload Results — ${{ matrix.browser }}-${{ matrix.env }}
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: results-${{ matrix.browser }}-${{ matrix.env }}-${{ github.run_number }}
          path: |
            target/cucumber-reports/
            target/allure-results/
            target/screenshots/
          retention-days: 14
```

---

## Selenium Grid with Service Containers

GitHub Actions supports service containers that run alongside the job. Use the official Selenium Docker images as services to provide Grid infrastructure without any manual startup.

```yaml
# .github/workflows/grid-tests.yml
name: Tests with Selenium Grid

on:
  workflow_dispatch:

jobs:
  grid-tests:
    name: Tests on Selenium Grid
    runs-on: ubuntu-latest
    timeout-minutes: 60

    services:
      selenium-hub:
        image: selenium/hub:4.18.1
        ports:
          - 4444:4444
          - 4442:4442
          - 4443:4443

      chrome-node:
        image: selenium/node-chrome:4.18.1
        options: --shm-size="2g"
        env:
          SE_EVENT_BUS_HOST: selenium-hub
          SE_EVENT_BUS_PUBLISH_PORT: 4442
          SE_EVENT_BUS_SUBSCRIBE_PORT: 4443
          SE_NODE_MAX_SESSIONS: 3

      firefox-node:
        image: selenium/node-firefox:4.18.1
        options: --shm-size="2g"
        env:
          SE_EVENT_BUS_HOST: selenium-hub
          SE_EVENT_BUS_PUBLISH_PORT: 4442
          SE_EVENT_BUS_SUBSCRIBE_PORT: 4443
          SE_NODE_MAX_SESSIONS: 2

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set Up JDK
        uses: actions/setup-java@v4
        with:
          java-version: "11"
          distribution: "temurin"
          cache: "maven"

      - name: Wait for Grid to Be Ready
        run: |
          echo "Waiting for Selenium Grid..."
          for i in $(seq 1 30); do
            STATUS=$(curl -s http://localhost:4444/status | grep -o '"ready":true')
            if [ "$STATUS" = '"ready":true' ]; then
              echo "Grid is ready after $i attempts."
              break
            fi
            echo "Attempt $i/30 — waiting 5s..."
            sleep 5
          done
          curl -s http://localhost:4444/status | python3 -m json.tool

      - name: Run Tests on Grid
        run: |
          mvn test \
            -Dgrid.enabled=true \
            -Dgrid.url=http://localhost:4444 \
            -Dbrowser=chrome \
            -Dheadless=true \
            -Dthread.count=4 \
            -Dcucumber.filter.tags="@regression"
        env:
          APP_BASE_URL: ${{ secrets.STAGING_BASE_URL }}
          TEST_ADMIN_USERNAME: ${{ secrets.TEST_ADMIN_USERNAME }}
          TEST_ADMIN_PASSWORD: ${{ secrets.TEST_ADMIN_PASSWORD }}

      - name: Upload Reports
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: grid-test-reports-${{ github.run_number }}
          path: |
            target/cucumber-reports/
            target/allure-results/
            target/extent-reports/
          retention-days: 14
```

---

## Allure Report Published to GitHub Pages

```yaml
# .github/workflows/allure-pages.yml
name: Allure Report to GitHub Pages

on:
  workflow_run:
    workflows: ["Nightly Regression"]
    types: [completed]

jobs:
  publish-allure:
    name: Publish Allure to GitHub Pages
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion != 'cancelled' }}

    steps:
      - name: Checkout gh-pages Branch
        uses: actions/checkout@v4
        with:
          ref: gh-pages
          path: gh-pages

      - name: Download Allure Results
        uses: actions/download-artifact@v4
        with:
          name: allure-results-${{ github.event.workflow_run.run_number }}
          path: allure-results
          run-id: ${{ github.event.workflow_run.id }}

      - name: Set Up JDK
        uses: actions/setup-java@v4
        with:
          java-version: "11"
          distribution: "temurin"

      - name: Install Allure CLI
        run: |
          wget -q https://github.com/allure-framework/allure2/releases/download/2.25.0/allure-2.25.0.tgz
          tar -xzf allure-2.25.0.tgz
          sudo mv allure-2.25.0/bin/allure /usr/local/bin/

      - name: Copy History from Previous Run
        run: |
          if [ -d gh-pages/history ]; then
            cp -r gh-pages/history allure-results/history
          fi

      - name: Generate Allure Report
        run: allure generate allure-results -o allure-report --clean

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: allure-report
          destination_dir: allure-report
          keep_files: true
```

---

## Secrets Configuration

Store all sensitive values as repository secrets. Navigate to `GitHub Repository → Settings → Secrets and Variables → Actions → New repository secret`.

| Secret Name              | Value                          |
| ------------------------ | ------------------------------ |
| `STAGING_BASE_URL`       | `https://staging.example.com`  |
| `STAGING_ADMIN_USERNAME` | `admin@example.com`            |
| `STAGING_ADMIN_PASSWORD` | `SecurePassword123`            |
| `STAGING_BUYER_USERNAME` | `buyer@example.com`            |
| `STAGING_BUYER_PASSWORD` | `BuyerPassword456`             |
| `SLACK_WEBHOOK_URL`      | Slack Incoming Webhook URL     |
| `GRID_URL`               | `http://grid.company.com:4444` |

Reference secrets in workflows:

```yaml
env:
  APP_BASE_URL: ${{ secrets.STAGING_BASE_URL }}
```

---

## Reusable Workflow

Define a reusable workflow that other workflows call — eliminating duplication across smoke, regression, and cross-browser workflows:

```yaml
# .github/workflows/reusable-test.yml
name: Reusable Test Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      browser:
        required: false
        type: string
        default: chrome
      tags:
        required: false
        type: string
        default: "@smoke"
      thread_count:
        required: false
        type: number
        default: 4
    secrets:
      BASE_URL:
        required: true
      ADMIN_USERNAME:
        required: true
      ADMIN_PASSWORD:
        required: true

jobs:
  run-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: "11"
          distribution: "temurin"
          cache: "maven"
      - uses: browser-actions/setup-chrome@v1
      - name: Run Tests
        run: |
          mvn test \
            -Denv=${{ inputs.environment }} \
            -Dbrowser=${{ inputs.browser }} \
            -Dheadless=true \
            -Dthread.count=${{ inputs.thread_count }} \
            -Dcucumber.filter.tags="${{ inputs.tags }}"
        env:
          APP_BASE_URL: ${{ secrets.BASE_URL }}
          TEST_ADMIN_USERNAME: ${{ secrets.ADMIN_USERNAME }}
          TEST_ADMIN_PASSWORD: ${{ secrets.ADMIN_PASSWORD }}
```

**Caller workflow:**

```yaml
jobs:
  smoke:
    uses: ./.github/workflows/reusable-test.yml
    with:
      environment: staging
      tags: "@smoke"
    secrets:
      BASE_URL: ${{ secrets.STAGING_BASE_URL }}
      ADMIN_USERNAME: ${{ secrets.STAGING_ADMIN_USERNAME }}
      ADMIN_PASSWORD: ${{ secrets.STAGING_ADMIN_PASSWORD }}
```

---

## Summary

GitHub Actions provides a zero-infrastructure CI/CD platform for Selenium Cucumber frameworks. Workflows defined as YAML files in `.github/workflows/` trigger automatically on push, PR, schedule, or manual dispatch. Smoke tests run on every PR, catching regressions before merge. Nightly regression runs the full suite on a schedule. Matrix builds distribute cross-browser coverage in parallel without extra configuration. Service containers bring Selenium Grid infrastructure directly into the workflow job. Secrets inject credentials securely at runtime. Allure results published to GitHub Pages give every team member a live test dashboard with historical trends.

---

## Related Topics

- Jenkins.md — Alternative CI/CD server with more customization options
- SeleniumGrid.md — Grid configuration used as service containers
- ParallelExecution.md — Thread count and test isolation for parallel runs
- Docker.md — Docker images used in service containers
- AllureReports.md — Allure report publishing to GitHub Pages
