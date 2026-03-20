# Docker with Selenium

## Overview

Docker provides consistent, reproducible environments for running Selenium tests. Instead of managing browser installations, driver versions, display servers, and dependencies on every machine individually, Docker packages the entire execution environment into images that behave identically on a developer's laptop, a CI agent, and a cloud server. Docker is the backbone of modern Selenium Grid deployments and the standard container strategy for CI/CD test pipelines.

This document covers Docker fundamentals for Selenium, running tests in a containerized test runner, Selenium Grid with Docker Compose, multi-stage Dockerfiles, volume management for reports, and integration with CI/CD platforms.

---

## Why Docker for Selenium Testing

**Environment consistency.** A Docker image that runs tests on your laptop will produce identical results on the CI server. No more "it works on my machine" failures caused by browser version mismatches or missing system dependencies.

**Isolation.** Each test run gets a fresh container. No state leaks between runs. No residual cookies, cached files, or leftover test data from previous executions.

**Browser version control.** Pin your tests to a specific Chrome or Firefox version. Upgrade browser versions deliberately by updating the image tag, not by accepting automatic system updates.

**Scalability.** Spin up 10 Chrome nodes for a large parallel run, tear them down after. Pay only for what you use.

**No display server required.** Selenium Docker images include everything needed for headless browser execution on Linux — no Xvfb, no VNC setup required for CI.

---

## Official Selenium Docker Images

The Selenium project publishes official Docker images at `docker.io/selenium/`:

| Image                         | Purpose                                  |
| ----------------------------- | ---------------------------------------- |
| `selenium/hub`                | Selenium Grid hub (Router + Distributor) |
| `selenium/node-chrome`        | Chrome browser node                      |
| `selenium/node-firefox`       | Firefox browser node                     |
| `selenium/node-edge`          | Edge browser node                        |
| `selenium/standalone-chrome`  | All-in-one Chrome (hub + node)           |
| `selenium/standalone-firefox` | All-in-one Firefox (hub + node)          |
| `selenium/standalone-edge`    | All-in-one Edge (hub + node)             |

All images are versioned by Selenium version: `selenium/node-chrome:4.18.1`. Always pin image versions to avoid breaking changes from `latest`.

---

## Dockerfile for the Test Runner

The test runner container packages your framework source code and runs `mvn test`. It does not contain a browser — browsers run in separate node containers.

```dockerfile
# Dockerfile
FROM maven:3.9.6-eclipse-temurin-11-focal

LABEL maintainer="qa-team@company.com"
LABEL description="Selenium Cucumber Test Runner"
LABEL version="1.0"

# Working directory
WORKDIR /app

# Copy POM first for dependency caching
# Docker caches this layer until pom.xml changes
COPY pom.xml ./

# Download all Maven dependencies in a dedicated layer
# This layer is reused on subsequent builds unless pom.xml changes
RUN mvn dependency:go-offline -q

# Copy the rest of the source code
COPY src/ ./src/
COPY testng.xml ./

# Create output directories with correct permissions
RUN mkdir -p \
    target/cucumber-reports \
    target/allure-results \
    target/extent-reports \
    target/screenshots && \
    chmod -R 777 target/

# Default environment variables — overridable at runtime
ENV BROWSER=chrome
ENV HEADLESS=true
ENV ENV_TARGET=staging
ENV TAGS="@smoke"
ENV GRID_ENABLED=true
ENV GRID_URL=http://selenium-hub:4444
ENV THREAD_COUNT=4
ENV MAVEN_OPTS="-Xmx2048m -XX:MaxMetaspaceSize=512m"

# Default command — runs when container starts with no override
CMD mvn test \
    -Dbrowser=${BROWSER} \
    -Dheadless=${HEADLESS} \
    -Denv=${ENV_TARGET} \
    -Dcucumber.filter.tags="${TAGS}" \
    -Dgrid.enabled=${GRID_ENABLED} \
    -Dgrid.url=${GRID_URL} \
    -Dthread.count=${THREAD_COUNT} \
    --no-transfer-progress
```

### Build the Test Runner Image

```bash
# Build
docker build -t selenium-test-runner:1.0 .

# Tag for registry
docker tag selenium-test-runner:1.0 registry.company.com/qa/selenium-runner:1.0

# Push to registry
docker push registry.company.com/qa/selenium-runner:1.0
```

---

## Multi-Stage Dockerfile (Optimized)

Multi-stage builds produce smaller final images by discarding build-time dependencies:

```dockerfile
# Stage 1 — Download dependencies (cached layer)
FROM maven:3.9.6-eclipse-temurin-11-focal AS dependencies

WORKDIR /app
COPY pom.xml ./
RUN mvn dependency:go-offline -q

# Stage 2 — Compile (separate layer for compilation cache)
FROM dependencies AS builder

COPY src/ ./src/
COPY testng.xml ./
RUN mvn compile test-compile -q

# Stage 3 — Runtime runner (slim image with compiled code)
FROM eclipse-temurin:11-jre-focal AS runner

WORKDIR /app

# Copy Maven local repository and compiled classes
COPY --from=builder /root/.m2 /root/.m2
COPY --from=builder /app /app

RUN mkdir -p target/cucumber-reports target/allure-results target/screenshots target/extent-reports

ENV BROWSER=chrome \
    HEADLESS=true \
    ENV_TARGET=staging \
    TAGS="@smoke" \
    GRID_ENABLED=true \
    GRID_URL=http://selenium-hub:4444 \
    THREAD_COUNT=4

CMD ["mvn", "test", \
     "-Dbrowser=${BROWSER}", \
     "-Dheadless=${HEADLESS}", \
     "-Denv=${ENV_TARGET}", \
     "-Dcucumber.filter.tags=${TAGS}", \
     "-Dgrid.enabled=${GRID_ENABLED}", \
     "-Dgrid.url=${GRID_URL}", \
     "--no-transfer-progress"]
```

---

## Docker Compose — Full Grid Stack

`docker-compose.yml` defines the complete Grid infrastructure and test runner as a coordinated service stack.

```yaml
# docker-compose.yml
version: "3.8"

networks:
  selenium-grid:
    driver: bridge

volumes:
  test-reports:
  screenshots:
  maven-cache:

services:
  # -----------------------------------------------------------------------
  # Selenium Grid Hub
  # -----------------------------------------------------------------------
  selenium-hub:
    image: selenium/hub:4.18.1
    container_name: selenium-hub
    networks:
      - selenium-grid
    ports:
      - "4444:4444" # WebDriver endpoint
      - "4442:4442" # Event bus publish
      - "4443:4443" # Event bus subscribe
    environment:
      - GRID_MAX_SESSION=15
      - GRID_BROWSER_TIMEOUT=300
      - GRID_TIMEOUT=300
      - SE_ENABLE_TRACING=false
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4444/status"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 15s

  # -----------------------------------------------------------------------
  # Chrome Nodes (scalable — use --scale to add more)
  # -----------------------------------------------------------------------
  chrome-node:
    image: selenium/node-chrome:4.18.1
    container_name: chrome-node
    networks:
      - selenium-grid
    depends_on:
      selenium-hub:
        condition: service_healthy
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
      - SE_NODE_MAX_SESSIONS=3
      - SE_NODE_OVERRIDE_MAX_SESSIONS=true
      - SE_NODE_SESSION_TIMEOUT=300
      - SE_VNC_NO_PASSWORD=1
    volumes:
      - /dev/shm:/dev/shm # critical for Chrome stability
    shm_size: "2gb"
    deploy:
      replicas: 2 # start 2 Chrome nodes by default

  # -----------------------------------------------------------------------
  # Firefox Node
  # -----------------------------------------------------------------------
  firefox-node:
    image: selenium/node-firefox:4.18.1
    container_name: firefox-node
    networks:
      - selenium-grid
    depends_on:
      selenium-hub:
        condition: service_healthy
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
      - SE_NODE_MAX_SESSIONS=2
    volumes:
      - /dev/shm:/dev/shm
    shm_size: "2gb"

  # -----------------------------------------------------------------------
  # Test Runner
  # -----------------------------------------------------------------------
  test-runner:
    build:
      context: .
      dockerfile: Dockerfile
      target: runner
    container_name: test-runner
    networks:
      - selenium-grid
    depends_on:
      selenium-hub:
        condition: service_healthy
    environment:
      - BROWSER=chrome
      - HEADLESS=true
      - ENV_TARGET=staging
      - TAGS=@regression and not @wip
      - GRID_ENABLED=true
      - GRID_URL=http://selenium-hub:4444
      - THREAD_COUNT=4
      - APP_BASE_URL=${APP_BASE_URL}
      - TEST_ADMIN_USERNAME=${TEST_ADMIN_USERNAME}
      - TEST_ADMIN_PASSWORD=${TEST_ADMIN_PASSWORD}
    volumes:
      # Mount report directories to host for post-run access
      - ./target/cucumber-reports:/app/target/cucumber-reports
      - ./target/allure-results:/app/target/allure-results
      - ./target/extent-reports:/app/target/extent-reports
      - ./target/screenshots:/app/target/screenshots
      # Cache Maven dependencies on the host
      - maven-cache:/root/.m2
    # Exit code reflects test result: 0 = pass, non-zero = failure
    profiles:
      - test # Only start with: docker-compose --profile test up
```

### .env File for Secrets

Store environment-specific secrets in a `.env` file (never commit to Git):

```bash
# .env — gitignored
APP_BASE_URL=https://staging.example.com
TEST_ADMIN_USERNAME=admin@example.com
TEST_ADMIN_PASSWORD=SecurePass123
TEST_BUYER_USERNAME=buyer@example.com
TEST_BUYER_PASSWORD=BuyerPass456
```

Add `.env` to `.gitignore`:

```gitignore
.env
*.env
target/
```

---

## Running Tests with Docker Compose

```bash
# Start Grid only (for manual/IDE-based test runs)
docker-compose up -d selenium-hub chrome-node firefox-node

# Wait for Grid to be ready
./scripts/wait-for-grid.sh http://localhost:4444

# Run tests against the running Grid
docker-compose --profile test up test-runner

# Run everything in one command
docker-compose --profile test up --abort-on-container-exit

# Scale Chrome nodes for large parallel runs
docker-compose up -d --scale chrome-node=5

# Watch Grid UI during execution
open http://localhost:4444

# Tear down everything
docker-compose down -v

# Tear down and remove images
docker-compose down --rmi all -v
```

---

## Standalone Chrome — Simpler CI Setup

For simple CI runs without a full Grid, use `standalone-chrome`. All-in-one: hub, node, and browser in a single container.

```yaml
# docker-compose-standalone.yml
version: "3.8"

services:
  chrome:
    image: selenium/standalone-chrome:4.18.1
    container_name: selenium-chrome
    ports:
      - "4444:4444"
    shm_size: "2gb"
    environment:
      - SE_NODE_MAX_SESSIONS=5
      - SE_VNC_NO_PASSWORD=1
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4444/status"]
      interval: 5s
      timeout: 3s
      retries: 12

  tests:
    image: maven:3.9.6-eclipse-temurin-11-focal
    depends_on:
      chrome:
        condition: service_healthy
    volumes:
      - .:/app
      - ~/.m2:/root/.m2
    working_dir: /app
    command: >
      mvn test
      -Dgrid.enabled=true
      -Dgrid.url=http://chrome:4444
      -Dbrowser=chrome
      -Dheadless=true
      -Dcucumber.filter.tags=@smoke
    environment:
      - APP_BASE_URL=${APP_BASE_URL}
      - TEST_ADMIN_USERNAME=${TEST_ADMIN_USERNAME}
      - TEST_ADMIN_PASSWORD=${TEST_ADMIN_PASSWORD}
```

---

## Volume Strategy for Reports

Mount specific output directories from the container to the host. This makes reports available after the container exits:

```bash
# Run with volume mounts for all report directories
docker run --rm \
  --network selenium-grid_selenium-grid \
  -e GRID_URL=http://selenium-hub:4444 \
  -e ENV_TARGET=staging \
  -e TAGS="@regression" \
  -v $(pwd)/target/cucumber-reports:/app/target/cucumber-reports \
  -v $(pwd)/target/allure-results:/app/target/allure-results \
  -v $(pwd)/target/extent-reports:/app/target/extent-reports \
  -v $(pwd)/target/screenshots:/app/target/screenshots \
  -v ~/.m2:/root/.m2 \
  selenium-test-runner:1.0
```

After the run, all reports are on the host at `./target/`.

---

## Docker in GitHub Actions

```yaml
- name: Start Selenium Grid
  run: |
    docker-compose -f docker-compose.yml up -d selenium-hub chrome-node firefox-node

- name: Wait for Grid
  run: |
    for i in $(seq 1 30); do
      if curl -sf http://localhost:4444/status | grep -q '"ready":true'; then
        echo "Grid ready"; break
      fi
      echo "Waiting... attempt $i"; sleep 5
    done

- name: Run Tests in Docker
  run: |
    docker-compose --profile test up --abort-on-container-exit test-runner

- name: Copy Reports from Container
  if: always()
  run: |
    # Reports are already on the host via volume mounts
    ls -la target/cucumber-reports/
    ls -la target/allure-results/

- name: Stop Grid
  if: always()
  run: docker-compose down -v
```

---

## Docker in Jenkins Pipeline

```groovy
pipeline {
    agent any

    stages {
        stage('Start Grid') {
            steps {
                sh 'docker-compose up -d selenium-hub chrome-node'
                sh '''
                    echo "Waiting for Grid..."
                    for i in $(seq 1 30); do
                        STATUS=$(curl -sf http://localhost:4444/status | grep -o '"ready":true' || true)
                        if [ "$STATUS" = '"ready":true' ]; then
                            echo "Grid ready."
                            break
                        fi
                        sleep 5
                    done
                '''
            }
        }

        stage('Run Tests') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'staging-app-creds',
                        usernameVariable: 'APP_USER',
                        passwordVariable: 'APP_PASS'
                    )
                ]) {
                    sh """
                        docker run --rm \\
                          --network \$(docker-compose ps -q selenium-hub | xargs docker inspect --format '{{range .NetworkSettings.Networks}}{{.NetworkID}}{{end}}' | head -1) \\
                          -e GRID_URL=http://selenium-hub:4444 \\
                          -e ENV_TARGET=staging \\
                          -e TAGS="@regression" \\
                          -e TEST_ADMIN_USERNAME=${APP_USER} \\
                          -e TEST_ADMIN_PASSWORD=${APP_PASS} \\
                          -v \${WORKSPACE}/target:/app/target \\
                          selenium-test-runner:1.0
                    """
                }
            }
        }
    }

    post {
        always {
            sh 'docker-compose down -v || true'
            junit 'target/cucumber-reports/*.xml'
            allure results: [[path: 'target/allure-results']]
        }
    }
}
```

---

## Useful Docker Commands for Selenium Testing

```bash
# View running containers and their status
docker ps

# View Grid logs in real time
docker logs -f selenium-hub

# Inspect Chrome node logs
docker logs -f chrome-node

# Enter a running container for debugging
docker exec -it test-runner /bin/bash

# Check Grid status
curl -s http://localhost:4444/status | python3 -m json.tool

# Scale Chrome nodes up
docker-compose up -d --scale chrome-node=6

# Remove all stopped containers and dangling images
docker system prune -f

# Remove all project containers, networks, and volumes
docker-compose down -v --remove-orphans

# Check image sizes
docker images | grep selenium
```

---

## .dockerignore

Exclude unnecessary files from the Docker build context to speed up builds:

```dockerignore
# Version control
.git
.gitignore

# IDE files
.idea/
*.iml
.vscode/

# Maven build output (never include in context)
target/

# Environment files (security — never in image)
.env
*.env

# Documentation
*.md
docs/

# CI configuration
.github/
Jenkinsfile

# OS files
.DS_Store
Thumbs.db
```

---

## Summary

Docker standardizes every aspect of the Selenium test execution environment. The test runner Dockerfile packages the framework with a pinned Java and Maven version. Docker Compose orchestrates the Grid hub, browser nodes, and test runner as a coordinated stack with health checks and dependency ordering. Volume mounts make reports available on the host after container exit. Environment variables and `.env` files inject configuration and credentials cleanly. The entire stack — start Grid, wait for readiness, run tests, collect reports, tear down — executes as a single `docker-compose` command, making it trivially reproducible on any machine or CI agent that has Docker installed.

---

## Related Topics

- SeleniumGrid.md — Grid architecture and node configuration
- ParallelExecution.md — Thread count tuned to Grid node capacity
- Jenkins.md — Jenkins pipeline that orchestrates Docker Compose
- GitHubActions.md — Service containers and Docker in GitHub workflows
- HeadlessTesting.md — Headless browser configuration inside Docker nodes
