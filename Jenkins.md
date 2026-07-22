---
render_with_liquid: false
layout: default
title: "🔧 Jenkins — DevOps Interview Guide"
---

# 🔧 Jenkins — DevOps Interview Guide

> [← Kubernetes](./Kubernetes.md) | [Main Index](./README.md) | [Terraform →](./Terraform.md)

---

## Table of Contents

1. [Jenkins Architecture](#jenkins-architecture)
2. [Pipeline Fundamentals](#pipeline-fundamentals)
3. [Declarative Pipeline](#declarative-pipeline)
4. [Scripted Pipeline](#scripted-pipeline)
5. [Shared Libraries](#shared-libraries)
6. [Agents & Nodes](#agents--nodes)
7. [Plugins](#plugins)
8. [Security](#security)
9. [Interview Questions](#interview-questions)
10. [Scenario-Based Questions](#scenario-based-questions)
11. [Hands-On Labs](#hands-on-labs)
12. [Cheat Sheet](#cheat-sheet)

---

## Jenkins Architecture

```
┌─────────────────── Jenkins Master ──────────────────────┐
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐  │
│  │  Web UI  │  │  REST API│  │   Build Queue         │  │
│  └──────────┘  └──────────┘  └──────────────────────┘  │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Plugin System (1800+ plugins)                    │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Job / Pipeline Scheduler                         │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────────┬──────────────────────────────────┘
                       │  JNLP / SSH
        ┌──────────────┴──────────────┐
        ▼                             ▼
┌── Agent Node 1 ──┐       ┌── Agent Node 2 ──┐
│  Linux x86_64    │       │  Docker-in-Docker │
│  (build/test)    │       │  (container jobs) │
└──────────────────┘       └──────────────────┘
```

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Job/Project** | A task Jenkins runs (Freestyle, Pipeline, Multibranch) |
| **Build** | A single execution of a job |
| **Workspace** | Directory on agent where build runs |
| **Executor** | A build slot on an agent |
| **Artifact** | Output produced by a build |
| **Fingerprint** | MD5 hash tracking artifact usage across jobs |

---

## Pipeline Fundamentals

### Pipeline Types

| Type | Description | Best For |
|------|-------------|---------|
| **Declarative** | Structured, opinionated syntax | Most pipelines, enforces best practices |
| **Scripted** | Full Groovy, flexible | Complex logic, conditionals |
| **Multibranch** | Auto-detects branches with Jenkinsfile | GitFlow, PR pipelines |
| **Organization Folder** | Scans all repos in GitHub org | Large orgs, auto-discovery |

---

## Declarative Pipeline

### Full Production-Grade Example

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: maven
                    image: maven:3.9-eclipse-temurin-17
                    command: ["sleep", "infinity"]
                  - name: docker
                    image: docker:24-dind
                    securityContext:
                      privileged: true
            '''
            defaultContainer 'maven'
        }
    }

    environment {
        APP_NAME       = 'myapp'
        IMAGE_REGISTRY = 'registry.example.com'
        IMAGE_TAG      = "${env.GIT_COMMIT[0..7]}"
        SONAR_TOKEN    = credentials('sonar-token')
        DOCKER_CREDS   = credentials('registry-creds')
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        disableConcurrentBuilds(abortPrevious: true)
        skipDefaultCheckout(true)
    }

    triggers {
        pollSCM('H/5 * * * *')   // Poll every 5 min (prefer webhooks)
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_AUTHOR = sh(
                        script: 'git log -1 --format="%an"',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'mvn -B test'
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'mvn -B verify -P integration-test'
                    }
                }
            }
        }

        stage('Code Quality') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        mvn sonar:sonar \
                          -Dsonar.projectKey=${APP_NAME} \
                          -Dsonar.token=${SONAR_TOKEN}
                    '''
                }
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                container('docker') {
                    sh '''
                        echo ${DOCKER_CREDS_PSW} | \
                            docker login ${IMAGE_REGISTRY} \
                            -u ${DOCKER_CREDS_USR} --password-stdin
                        docker build -t ${IMAGE_REGISTRY}/${APP_NAME}:${IMAGE_TAG} .
                        docker push ${IMAGE_REGISTRY}/${APP_NAME}:${IMAGE_TAG}
                        docker tag  ${IMAGE_REGISTRY}/${APP_NAME}:${IMAGE_TAG} \
                                    ${IMAGE_REGISTRY}/${APP_NAME}:latest
                        docker push ${IMAGE_REGISTRY}/${APP_NAME}:latest
                    '''
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                sh """
                    kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${IMAGE_REGISTRY}/${APP_NAME}:${IMAGE_TAG} \
                        -n staging
                    kubectl rollout status deployment/${APP_NAME} -n staging
                """
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy ${IMAGE_TAG} to production?"
                ok "Deploy"
                submitter "ops-team"
            }
            steps {
                sh """
                    kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${IMAGE_REGISTRY}/${APP_NAME}:${IMAGE_TAG} \
                        -n production
                    kubectl rollout status deployment/${APP_NAME} -n production \
                        --timeout=5m
                """
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            slackSend(
                channel: '#deploys',
                color: 'good',
                message: "✅ ${APP_NAME}:${IMAGE_TAG} deployed by ${env.GIT_AUTHOR}"
            )
        }
        failure {
            slackSend(
                channel: '#alerts',
                color: 'danger',
                message: "❌ Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
            emailext(
                to: 'devops@example.com',
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: '${DEFAULT_CONTENT}'
            )
        }
    }
}
```

---

## Scripted Pipeline

### Groovy DSL Patterns

```groovy
// Scripted pipeline — full Groovy control
node('linux-agent') {
    def appVersion
    def gitCommit

    try {
        stage('Checkout') {
            checkout scm
            gitCommit = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
            appVersion = readFile('VERSION').trim()
        }

        stage('Build') {
            docker.image('maven:3.9').inside('-v $HOME/.m2:/root/.m2') {
                sh 'mvn -B clean package -DskipTests'
            }
        }

        stage('Test') {
            def testResults = [:]

            // Dynamic parallel stages
            ['unit', 'integration', 'contract'].each { type ->
                testResults[type] = {
                    node('linux-agent') {
                        sh "mvn -B test -P ${type}"
                    }
                }
            }
            parallel testResults
        }

        stage('Publish') {
            docker.withRegistry('https://registry.example.com', 'registry-creds') {
                def image = docker.build("myapp:${gitCommit}")
                image.push()
                image.push('latest')
            }
        }

    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        // Always clean up
        deleteDir()
    }
}
```

---

## Shared Libraries

### Structure

```
jenkins-shared-library/
├── vars/                   # Global variables (DSL steps)
│   ├── deployToK8s.groovy
│   ├── buildDockerImage.groovy
│   └── notifySlack.groovy
├── src/                    # Helper classes
│   └── com/example/
│       ├── Docker.groovy
│       └── Kubernetes.groovy
└── resources/              # Static resources
    └── scripts/
        └── health-check.sh
```

### vars/deployToK8s.groovy

```groovy
// Usage in Jenkinsfile: deployToK8s(app: 'myapp', env: 'production', tag: '1.0')
def call(Map config) {
    def app    = config.app    ?: error("'app' is required")
    def env    = config.env    ?: 'staging'
    def tag    = config.tag    ?: 'latest'
    def ns     = config.ns     ?: env
    def timeout = config.timeout ?: '5m'

    echo "Deploying ${app}:${tag} to ${env}"

    sh """
        kubectl set image deployment/${app} \
            ${app}=${IMAGE_REGISTRY}/${app}:${tag} \
            -n ${ns}
        kubectl rollout status deployment/${app} -n ${ns} --timeout=${timeout}
    """

    // Verify deployment
    def readyReplicas = sh(
        script: "kubectl get deployment/${app} -n ${ns} -o jsonpath='{.status.readyReplicas}'",
        returnStdout: true
    ).trim()

    echo "Ready replicas: ${readyReplicas}"
}
```

### vars/notifySlack.groovy

```groovy
def call(String message, String channel = '#devops', String color = 'good') {
    slackSend(
        channel: channel,
        color: color,
        message: "${message} | Job: ${env.JOB_NAME} #${env.BUILD_NUMBER} | <${env.BUILD_URL}|Open>"
    )
}
```

### Using the Shared Library in Jenkinsfile

```groovy
@Library('jenkins-shared-library@main') _

pipeline {
    agent any

    stages {
        stage('Deploy') {
            steps {
                deployToK8s(
                    app: 'myapp',
                    env: 'production',
                    tag: env.IMAGE_TAG,
                    timeout: '10m'
                )
            }
        }
    }

    post {
        success { notifySlack("✅ Deployment successful") }
        failure { notifySlack("❌ Deployment failed", '#alerts', 'danger') }
    }
}
```

---

## Agents & Nodes

### Static Agent via SSH

```groovy
// In Jenkins global config or via JCasC yaml:
jenkins:
  nodes:
    - permanent:
        name: "build-agent-01"
        remoteFS: "/home/jenkins"
        launcher:
          ssh:
            host: "10.0.1.50"
            credentialsId: "agent-ssh-key"
            sshHostKeyVerificationStrategy: "nonVerifyingKeyVerificationStrategy"
        label: "linux docker maven"
        numExecutors: 4
```

### Dynamic Kubernetes Agents

```groovy
pipeline {
    agent {
        kubernetes {
            cloud 'kubernetes'
            defaultContainer 'jnlp'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-agent
spec:
  serviceAccountName: jenkins-sa
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
  - name: maven
    image: maven:3.9-eclipse-temurin-17
    command: ["sleep", "infinity"]
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "2Gi"
        cpu: "1"
  - name: trivy
    image: aquasec/trivy:latest
    command: ["sleep", "infinity"]
"""
        }
    }
    // ...
}
```

---

## Plugins

### Essential Plugins for DevOps

| Plugin | Purpose |
|--------|---------|
| **Pipeline** | Core pipeline support |
| **Git** | Git SCM integration |
| **GitHub Branch Source** | Multibranch + GitHub PR builds |
| **Kubernetes** | Dynamic K8s agents |
| **Docker Pipeline** | `docker.build`, `docker.withRegistry` |
| **Credentials Binding** | Inject credentials as env vars |
| **SonarQube Scanner** | Code quality gate integration |
| **Slack Notification** | Slack build notifications |
| **JUnit** | Test result publishing |
| **Blue Ocean** | Modern pipeline visualization |
| **Job DSL** | Programmatically create jobs |
| **Configuration as Code (JCasC)** | YAML-based Jenkins config |
| **Artifactory** | Artifact management |
| **AWS Steps** | AWS SDK integration |
| **Kubernetes CLI** | `withKubeConfig` step |
| **Prometheus Metrics** | Expose metrics for Prometheus |

---

## Security

### Jenkins Security Hardening

```groovy
// JCasC - jenkins.yaml
jenkins:
  securityRealm:
    ldap:
      configurations:
        - server: ldap://ldap.example.com
          rootDN: dc=example,dc=com
          userSearchBase: ou=users
          groupSearchBase: ou=groups

  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions:
              - "Overall/Administer"
            assignments:
              - "jenkins-admins"
          - name: "developer"
            permissions:
              - "Job/Build"
              - "Job/Read"
              - "Job/Workspace"
            assignments:
              - "developers"

  globalNodeProperties:
    - envVars:
        env:
          - key: "JAVA_OPTS"
            value: "-Dhudson.model.DirectoryBrowserSupport.CSP=''"
```

### Credentials Management

```groovy
// Types of credentials
// - Secret text: API tokens, passwords
// - Username/password: registry creds
// - SSH key: agent/server access
// - Certificate: PKI
// - Secret file: kubeconfig, service account JSON

// Using credentials in pipeline
withCredentials([
    usernamePassword(
        credentialsId: 'registry-creds',
        usernameVariable: 'REGISTRY_USER',
        passwordVariable: 'REGISTRY_PASS'
    ),
    string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN'),
    file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')
]) {
    sh 'docker login -u $REGISTRY_USER -p $REGISTRY_PASS'
    sh 'kubectl --kubeconfig=$KUBECONFIG get pods'
}
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is a Jenkinsfile and where does it live?**

> A Jenkinsfile is a text file containing the pipeline definition using the Pipeline DSL (Groovy). It lives in the **root of the source code repository** alongside the application code. This is "Pipeline as Code" — the pipeline is versioned with the code, enabling code review of CI changes, branch-specific pipelines, and auditability.

**Q2: What is the difference between Declarative and Scripted pipelines?**

> **Declarative** uses a strict, opinionated structure (`pipeline { agent {} stages {} }`) — easier to read, has built-in error checking, supports `post`, `options`, `triggers`. **Scripted** is pure Groovy (`node { stage { } }`) — more flexible but harder to maintain. Recommendation: always start with Declarative; drop to `script {}` block inside Declarative when you need Groovy logic.

**Q3: What is the `post` block used for?**

> `post` defines actions to run after the pipeline (or stage) finishes. Conditions: `always`, `success`, `failure`, `unstable`, `changed`, `fixed`, `regression`, `aborted`, `cleanup`. Commonly used for: notifications (Slack, email), publishing test results, cleanup (`cleanWs()`), archiving artifacts.

---

### 🟡 Intermediate

**Q4: How do you handle secrets in Jenkins pipelines?**

> Never hardcode secrets. Use:
> 1. **Credentials plugin**: store in Jenkins Credential Store, inject via `withCredentials` or `credentials()` binding — Jenkins masks values in logs
> 2. **External vault**: HashiCorp Vault plugin, AWS Secrets Manager, retrieve at runtime
> 3. **Environment-specific configs**: different credentials for staging vs production
> Never use `echo $SECRET` (masked but may appear in subprocess output). Prefer `--password-stdin` patterns.

**Q5: Explain Jenkins shared libraries and their benefits.**

> Shared libraries are reusable Groovy code stored in a separate Git repository, loaded by multiple Jenkinsfiles via `@Library`. Benefits:
> - **DRY**: eliminate duplicate pipeline code across 100s of repos
> - **Standardization**: enforce organizational standards (security scans, notifications)
> - **Version control**: pin to tag (`@Library('mylib@v2.1')`)
> - **Testing**: unit-test shared library code with `JenkinsPipelineUnit`

**Q6: What is a multibranch pipeline?**

> Multibranch Pipeline automatically discovers branches (and PRs) in a repository that contain a Jenkinsfile and creates a sub-project for each. When a branch is deleted, its project is removed. PRs get their own build with merge preview. This enables:
> - Feature branch isolation
> - PR status checks (blocks merge on failure)
> - Branch-specific behavior via `when { branch 'main' }`

---

### 🔴 Advanced

**Q7: How do you implement Blue-Green deployments in Jenkins?**

```groovy
stage('Blue-Green Deploy') {
    steps {
        script {
            // Determine current active color
            def currentColor = sh(
                script: "kubectl get svc myapp -n prod -o jsonpath='{.spec.selector.color}'",
                returnStdout: true
            ).trim()
            def newColor = (currentColor == 'blue') ? 'green' : 'blue'

            echo "Deploying to ${newColor} (current: ${currentColor})"

            // Deploy new version to inactive color
            sh """
                kubectl set image deployment/myapp-${newColor} \
                    myapp=${IMAGE_REGISTRY}/myapp:${IMAGE_TAG} -n prod
                kubectl rollout status deployment/myapp-${newColor} \
                    -n prod --timeout=5m
            """

            // Health check
            sh "curl -f http://myapp-${newColor}.prod.svc.cluster.local/health"

            // Switch traffic
            sh """
                kubectl patch svc myapp -n prod \
                    -p '{"spec":{"selector":{"color":"${newColor}"}}}'
            """

            // Keep old version for quick rollback (scale down after X minutes)
            sleep(time: 10, unit: 'MINUTES')
            sh "kubectl scale deployment myapp-${currentColor} --replicas=0 -n prod"
        }
    }
}
```

**Q8: How do you scale Jenkins for large organizations?**

> - **Kubernetes agents**: Dynamic, ephemeral agents scale to zero — no idle cost
> - **Horizontal Controller Scaling**: Jenkins HA with multiple controllers (CloudBees)
> - **Folder/Organization structure**: delegate admin to team folders with role-based access
> - **Shared libraries**: reduce pipeline code and execution time
> - **Artifact caching**: mount Maven/npm cache as persistent volume in K8s agents
> - **Build parallelism**: `parallel` stages for test splitting
> - **Thin client**: Jenkins controller only orchestrates; all heavy work on agents

---

## Scenario-Based Questions

### 🔵 Scenario 1: Pipeline Fails Intermittently

*"Your CI pipeline fails with 'connection refused' on the Docker registry push step — but only sometimes."*

```groovy
// Add retry with backoff
stage('Push Image') {
    steps {
        retry(3) {
            script {
                try {
                    sh 'docker push ${IMAGE}'
                } catch (e) {
                    sleep(time: 10, unit: 'SECONDS')
                    throw e
                }
            }
        }
    }
}

// Also check:
// - Registry rate limits (Docker Hub: 100 pulls/6h anonymous)
// - Network flakiness between Jenkins agent and registry
// - Registry health / disk space
// - Credentials expiration
```

### 🔵 Scenario 2: Pipeline Runs Taking Too Long

```groovy
// Before: sequential 45-minute pipeline
// After: parallelized 15-minute pipeline

stage('Tests') {
    parallel {
        stage('Unit') {
            agent { label 'fast-agent' }
            steps { sh 'mvn test -pl module-a,module-b' }
        }
        stage('Integration') {
            agent { label 'fast-agent' }
            steps { sh 'mvn verify -P integration' }
        }
        stage('Security Scan') {
            agent { label 'security-agent' }
            steps { sh 'trivy image ${IMAGE}' }
        }
        stage('Lint') {
            agent { label 'fast-agent' }
            steps { sh 'mvn checkstyle:check' }
        }
    }
}

// Also: use test parallelism plugins
// Maven: -T 4 (4 threads), Surefire fork count
// JUnit 5: parallel execution config
```

---

## Hands-On Labs

### Lab 1: Simple Declarative Pipeline

```groovy
// Save as Jenkinsfile in a test repo
pipeline {
    agent any

    stages {
        stage('Hello') {
            steps {
                echo "Building ${env.JOB_NAME} #${env.BUILD_NUMBER}"
                echo "Branch: ${env.BRANCH_NAME ?: 'unknown'}"
                sh 'echo "Running on: $(hostname)"'
                sh 'echo "Docker: $(docker --version)"'
            }
        }

        stage('Test') {
            steps {
                sh '''
                    echo "Running tests..."
                    sleep 2
                    echo "All tests passed!"
                '''
            }
        }

        stage('Report') {
            steps {
                script {
                    def buildInfo = [
                        job: env.JOB_NAME,
                        build: env.BUILD_NUMBER,
                        url: env.BUILD_URL
                    ]
                    echo "Build info: ${buildInfo}"
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }
    }
}
```

---

## Cheat Sheet

```groovy
// ─── ENVIRONMENT ──────────────────────────────────────
env.JOB_NAME          // Job name
env.BUILD_NUMBER      // Build number
env.WORKSPACE         // Agent workspace path
env.GIT_BRANCH        // Current Git branch
env.GIT_COMMIT        // Full commit SHA
env.BUILD_URL         // URL to this build

// ─── CONDITIONALS ─────────────────────────────────────
when { branch 'main' }
when { environment name: 'DEPLOY', value: 'true' }
when { expression { return params.DEPLOY_TO_PROD } }
when { anyOf { branch 'main'; branch 'develop' } }
when { changeset "**/*.java" }

// ─── PARAMETERS ───────────────────────────────────────
parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'latest')
    booleanParam(name: 'SKIP_TESTS', defaultValue: false)
    choice(name: 'ENV', choices: ['staging', 'production'])
}

// ─── USEFUL STEPS ─────────────────────────────────────
sh 'command'                           // Run shell
sh(script: 'cmd', returnStdout: true)  // Capture output
readFile('file.txt')                   // Read file
writeFile file: 'out.txt', text: 'x'  // Write file
archiveArtifacts 'target/*.jar'        // Archive
stash name: 'built', includes: '**'   // Stash files
unstash 'built'                        // Retrieve stash
input message: 'Proceed?', ok: 'Yes'  // Manual gate
timeout(time: 5, unit: 'MINUTES') {}  // Timeout block
retry(3) { sh 'flaky-command' }       // Retry block
sleep(time: 30, unit: 'SECONDS')      // Wait
```

---

> **Cross-links:** [← Kubernetes](./Kubernetes.md) | [Terraform →](./Terraform.md) | [CI/CD patterns →](./CICD.md) | [Security (pipeline security) →](./Security.md)