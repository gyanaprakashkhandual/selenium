# Jenkins Integration

## Overview

Jenkins is the most widely deployed open-source CI/CD automation server. Integrating a Selenium Cucumber framework with Jenkins enables fully automated test execution on every code commit, pull request, scheduled trigger, or manual dispatch. Jenkins orchestrates the full test lifecycle: pulling source code, starting the Selenium Grid, executing tests, publishing reports, archiving artifacts, and sending notifications — all without human intervention.

This document covers the complete Jenkins setup: pipeline configuration using `Jenkinsfile`, parameterized builds, parallel stages, report publishing, Selenium Grid integration, and notification strategies.

---

## Prerequisites

### Required Jenkins Plugins

Install these plugins from `Manage Jenkins → Plugins`:

| Plugin                | Purpose                                    |
| --------------------- | ------------------------------------------ |
| Pipeline              | Declarative and scripted pipeline support  |
| Git                   | Source code checkout from Git repositories |
| Maven Integration     | Maven build lifecycle support              |
| HTML Publisher        | Publish Extent/Cucumber HTML reports       |
| Cucumber Reports      | Native Cucumber JSON report visualization  |
| Allure Jenkins Plugin | Allure report generation and display       |
| JUnit                 | JUnit XML result tracking and trend graphs |
| Email Extension       | Rich email notifications                   |
| Slack Notification    | Slack build status messages                |
| Docker Pipeline       | Docker commands in pipeline steps          |
| Credentials Binding   | Secure credential injection                |
| Timestamper           | Timestamps in console output               |
| AnsiColor             | Colored console output for Maven           |

---

## Basic Declarative Jenkinsfile

Place a `Jenkinsfile` in the root of the project repository. Jenkins detects it automatically when a Pipeline job is configured with `Pipeline script from SCM`.

```groovy
pipeline {
    agent any

    options {
        buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '10'))
        timestamps()
        ansiColor('xterm')
        timeout(time: 60, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    environment {
        MAVEN_OPTS = '-Xmx2048m -XX:MaxMetaspaceSize=512m'
        APP_ENV    = 'staging'
        BROWSER    = 'chrome'
    }

    tools {
        maven 'Maven-3.9'
        jdk   'JDK-11'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Branch: ${env.GIT_BRANCH}"
                echo "Commit: ${env.GIT_COMMIT}"
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile -q'
            }
        }

        stage('Run Tests') {
            steps {
                sh """
                    mvn test \
                      -Denv=${APP_ENV} \
                      -Dbrowser=${BROWSER} \
                      -Dheadless=true \
                      -Dcucumber.filter.tags="@regression and not @wip" \
                      -Dallure.results.directory=target/allure-results
                """
            }
            post {
                always {
                    // JUnit XML — shows test counts in Jenkins build summary
                    junit allowEmptyResults: true,
                          testResults: 'target/cucumber-reports/*.xml'
                }
            }
        }

    }

    post {
        always {
            // Cucumber HTML Report
            cucumber([
                fileIncludePattern:   '**/*.json',
                jsonReportDirectory:  'target/cucumber-reports',
                reportTitle:          'Cucumber Report',
                buildStatus:          'UNSTABLE',
                trendsLimit:          10
            ])

            // Allure Report
            allure([
                includeProperties: false,
                jdk: '',
                properties: [],
                reportBuildPolicy: 'ALWAYS',
                results: [[path: 'target/allure-results']]
            ])

            // Extent HTML Report
            publishHTML([
                allowMissing:          false,
                alwaysLinkToLastBuild: true,
                keepAll:               true,
                reportDir:             'target/extent-reports',
                reportFiles:           'SparkReport.html',
                reportName:            'Extent Report',
                reportTitles:          'Selenium Extent Report'
            ])

            // Archive test artifacts
            archiveArtifacts(
                artifacts:     'target/cucumber-reports/**, target/screenshots/**, target/allure-results/**',
                allowEmptyArchive: true
            )
        }

        failure {
            echo "BUILD FAILED — check reports for details"
            emailext(
                subject: "FAILED: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                body:    """
                    Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}
                    Branch: ${env.GIT_BRANCH}
                    URL: ${env.BUILD_URL}
                    Console: ${env.BUILD_URL}console
                """,
                to: 'qa-team@company.com'
            )
        }

        unstable {
            echo "BUILD UNSTABLE — some tests failed"
        }

        success {
            echo "BUILD SUCCESS — all tests passed"
        }

        cleanup {
            cleanWs()
        }
    }
}
```

---

## Parameterized Pipeline

A parameterized pipeline lets you choose the environment, browser, tag filter, and thread count at build time through Jenkins' UI.

```groovy
pipeline {
    agent any

    parameters {
        choice(
            name:        'ENVIRONMENT',
            choices:     ['staging', 'dev', 'uat', 'prod'],
            description: 'Target environment for test execution'
        )
        choice(
            name:        'BROWSER',
            choices:     ['chrome', 'firefox', 'edge'],
            description: 'Browser to run tests on'
        )
        string(
            name:         'TAGS',
            defaultValue: '@regression and not @wip',
            description:  'Cucumber tag expression'
        )
        string(
            name:         'THREAD_COUNT',
            defaultValue: '4',
            description:  'Number of parallel threads'
        )
        booleanParam(
            name:         'HEADLESS',
            defaultValue: true,
            description:  'Run browser in headless mode'
        )
        booleanParam(
            name:         'USE_GRID',
            defaultValue: false,
            description:  'Run tests on Selenium Grid'
        )
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timestamps()
        ansiColor('xterm')
        timeout(time: 90, unit: 'MINUTES')
    }

    environment {
        MAVEN_OPTS = '-Xmx2048m'
    }

    tools {
        maven 'Maven-3.9'
        jdk   'JDK-11'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    currentBuild.displayName = "#${BUILD_NUMBER} | ${params.ENVIRONMENT} | ${params.BROWSER}"
                    currentBuild.description = "Tags: ${params.TAGS}"
                }
            }
        }

        stage('Validate') {
            steps {
                sh 'mvn validate -q'
                echo "Environment : ${params.ENVIRONMENT}"
                echo "Browser     : ${params.BROWSER}"
                echo "Tags        : ${params.TAGS}"
                echo "Threads     : ${params.THREAD_COUNT}"
                echo "Headless    : ${params.HEADLESS}"
                echo "Use Grid    : ${params.USE_GRID}"
            }
        }

        stage('Start Grid') {
            when {
                expression { params.USE_GRID == true }
            }
            steps {
                sh '''
                    docker-compose -f docker-compose-grid.yml up -d
                    chmod +x scripts/wait-for-grid.sh
                    ./scripts/wait-for-grid.sh http://localhost:4444
                '''
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    def gridArgs = params.USE_GRID
                        ? "-Dgrid.enabled=true -Dgrid.url=http://localhost:4444"
                        : "-Dgrid.enabled=false"

                    sh """
                        mvn test \
                          -Denv=${params.ENVIRONMENT} \
                          -Dbrowser=${params.BROWSER} \
                          -Dheadless=${params.HEADLESS} \
                          -Dthread.count=${params.THREAD_COUNT} \
                          -Dcucumber.filter.tags="${params.TAGS}" \
                          ${gridArgs}
                    """
                }
            }
            post {
                always {
                    junit allowEmptyResults: true,
                          testResults: 'target/cucumber-reports/*.xml'
                }
            }
        }

        stage('Stop Grid') {
            when {
                expression { params.USE_GRID == true }
                always()
            }
            steps {
                sh 'docker-compose -f docker-compose-grid.yml down -v'
            }
        }

    }

    post {
        always {
            cucumber([
                fileIncludePattern:  '**/*.json',
                jsonReportDirectory: 'target/cucumber-reports',
                reportTitle:         "Cucumber — ${params.ENVIRONMENT} / ${params.BROWSER}"
            ])

            allure([
                reportBuildPolicy: 'ALWAYS',
                results: [[path: 'target/allure-results']]
            ])

            publishHTML([
                keepAll:      true,
                reportDir:    'target/extent-reports',
                reportFiles:  'SparkReport.html',
                reportName:   'Extent Report'
            ])

            archiveArtifacts(
                artifacts: 'target/cucumber-reports/**, target/screenshots/**, target/allure-results/**',
                allowEmptyArchive: true
            )
        }

        failure  { notifySlack('FAILED',   '#FF0000') }
        unstable { notifySlack('UNSTABLE', '#FFA500') }
        success  { notifySlack('PASSED',   '#36A64F') }

        cleanup { cleanWs() }
    }
}

def notifySlack(String status, String color) {
    slackSend(
        channel:     '#qa-automation',
        color:       color,
        message:     """
*${status}*: ${env.JOB_NAME} #${env.BUILD_NUMBER}
*Env*: ${params.ENVIRONMENT} | *Browser*: ${params.BROWSER}
*Tags*: ${params.TAGS}
<${env.BUILD_URL}|View Build> | <${env.BUILD_URL}allure/|Allure Report>
        """.stripIndent()
    )
}
```

---

## Parallel Pipeline Stages

Run the same test suite against multiple browsers simultaneously using parallel pipeline stages:

```groovy
stage('Cross-Browser Tests') {
    parallel {

        stage('Chrome') {
            steps {
                sh """
                    mvn test \
                      -Dbrowser=chrome \
                      -Dheadless=true \
                      -Dcucumber.filter.tags="@regression" \
                      -Dsurefire.reportsDirectory=target/surefire-chrome \
                      -Djson.output=target/cucumber-reports/chrome.json
                """
            }
            post {
                always {
                    junit 'target/surefire-chrome/*.xml'
                }
            }
        }

        stage('Firefox') {
            steps {
                sh """
                    mvn test \
                      -Dbrowser=firefox \
                      -Dheadless=true \
                      -Dcucumber.filter.tags="@regression" \
                      -Dsurefire.reportsDirectory=target/surefire-firefox \
                      -Djson.output=target/cucumber-reports/firefox.json
                """
            }
            post {
                always {
                    junit 'target/surefire-firefox/*.xml'
                }
            }
        }

        stage('Edge') {
            steps {
                sh """
                    mvn test \
                      -Dbrowser=edge \
                      -Dheadless=true \
                      -Dcucumber.filter.tags="@smoke" \
                      -Dsurefire.reportsDirectory=target/surefire-edge \
                      -Djson.output=target/cucumber-reports/edge.json
                """
            }
            post {
                always {
                    junit 'target/surefire-edge/*.xml'
                }
            }
        }
    }
}
```

---

## Scheduled Builds

Configure scheduled runs using Jenkins cron syntax:

```groovy
triggers {
    // Run regression suite every night at 11 PM
    cron('0 23 * * *')

    // Run smoke tests every hour during business hours (Mon-Fri)
    cron('0 8-18 * * 1-5')

    // Trigger on SCM change (polling every 5 minutes)
    pollSCM('H/5 * * * *')
}
```

---

## Credentials Management

Never hard-code passwords in `Jenkinsfile`. Store them in Jenkins Credentials Manager and inject them at runtime.

```groovy
environment {
    // Inject credential pair as USERNAME and PASSWORD variables
    APP_CREDS = credentials('staging-app-credentials')
    // APP_CREDS_USR and APP_CREDS_PSW are auto-created

    // Single secret text
    API_TOKEN = credentials('staging-api-token')
}

stage('Run Tests') {
    steps {
        sh """
            mvn test \
              -Dtest.admin.username=${APP_CREDS_USR} \
              -Dtest.admin.password=${APP_CREDS_PSW} \
              -Dapi.token=${API_TOKEN} \
              -Denv=staging
        """
    }
}
```

Store credentials in Jenkins: `Manage Jenkins → Credentials → System → Global credentials → Add Credentials`.

---

## Rerun Failed Tests Stage

Add a rerun stage after the main run to confirm whether failures are genuine defects or infrastructure flakiness:

```groovy
stage('Rerun Failed Tests') {
    when {
        expression {
            // Only rerun if there were failures
            return fileExists('target/cucumber-reports/rerun.txt') &&
                   readFile('target/cucumber-reports/rerun.txt').trim() != ''
        }
    }
    steps {
        echo "Rerunning failed scenarios..."
        sh """
            mvn test \
              -Dcucumber.features="@target/cucumber-reports/rerun.txt" \
              -Denv=${params.ENVIRONMENT} \
              -Dheadless=true \
              -Dallure.results.directory=target/allure-results-rerun
        """
    }
    post {
        always {
            junit allowEmptyResults: true,
                  testResults: 'target/cucumber-reports/*.xml'
        }
    }
}
```

---

## Jenkins Job DSL — Creating Jobs Programmatically

For teams managing many Jenkins jobs, use Job DSL to define jobs as code:

```groovy
// jobs.groovy — processed by Job DSL plugin
pipelineJob('selenium-regression') {
    description('Nightly regression suite for all environments')

    parameters {
        choiceParam('ENVIRONMENT', ['staging', 'dev', 'prod'], 'Target environment')
        choiceParam('BROWSER',     ['chrome', 'firefox'],       'Browser')
        stringParam('TAGS',        '@regression',               'Cucumber tag expression')
    }

    triggers {
        cron('0 22 * * *')
    }

    definition {
        cpsScm {
            scm {
                git {
                    remote {
                        url('https://github.com/company/selenium-framework.git')
                        credentials('github-credentials')
                    }
                    branch('*/main')
                }
            }
            scriptPath('Jenkinsfile')
        }
    }

    logRotator {
        numToKeep(30)
        artifactNumToKeep(10)
    }
}
```

---

## Build Status Badge

Add a Jenkins build status badge to the project `README.md`:

```markdown
[![Build Status](http://jenkins.company.com/buildStatus/icon?job=selenium-regression)](http://jenkins.company.com/job/selenium-regression/)
```

---

## Summary

Jenkins provides the automation backbone for a professional Selenium Cucumber CI/CD pipeline. A declarative `Jenkinsfile` in source control defines the complete build lifecycle: checkout, compile, Grid startup, parallel test execution, report publishing, artifact archiving, Grid teardown, and notifications. Parameterized builds allow any team member to trigger runs against any environment with any tag filter from the Jenkins UI. Scheduled nightly triggers, parallel browser stages, credential management, and failure rerun complete the production-grade pipeline.

---

## Related Topics

- SeleniumGrid.md — Grid started and stopped from Jenkins pipeline stages
- ParallelExecution.md — Thread count configured through Jenkins parameters
- GitHubActions.md — Alternative CI/CD platform to Jenkins
- Docker.md — Docker Compose for Grid management in Jenkins pipelines
- ExtentReports.md — Extent HTML report published via publishHTML plugin
- AllureReports.md — Allure report generated via Jenkins Allure plugin
