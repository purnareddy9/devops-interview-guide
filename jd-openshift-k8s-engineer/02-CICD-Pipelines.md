# 🔄 Module 2 — CI/CD Pipelines

> **JD Alignment:** "Design, implement and maintain CI/CD pipelines for automated application deployment. Build and manage CI/CD pipelines using tools such as Jenkins, GitLab CI or similar."

> [← OpenShift & Kubernetes Core](./01-OpenShift-Kubernetes-Core.md) | [README](./README.md) | [Security & Compliance →](./03-Security-Compliance.md)

---

## Table of Contents

1. [CI/CD Fundamentals](#cicd-fundamentals)
2. [Jenkins Pipelines](#jenkins-pipelines)
3. [GitLab CI/CD](#gitlab-cicd)
4. [GitHub Actions](#github-actions)
5. [OpenShift Pipelines (Tekton)](#openshift-pipelines-tekton)
6. [GitOps & ArgoCD](#gitops--argocd)
7. [Deployment Strategies](#deployment-strategies)

---

## CI/CD Fundamentals

---

### 🟢 Q1. What is CI/CD and why is it important in a DevOps culture?

**Answer:**

```
Continuous Integration (CI)
  Developers merge code to a shared branch frequently (daily or multiple times/day).
  Every merge triggers an automated pipeline: build → test → scan → artifact.
  Goal: detect integration problems early, reduce "works on my machine" issues.

Continuous Delivery (CD)
  Every passing CI build is packaged and ready to deploy to production.
  Deployment to production is a conscious manual decision (button click or approval gate).

Continuous Deployment (CD — stronger form)
  Every passing build is automatically deployed to production with no human gate.
  Requires very high automated test coverage and confidence.
```

**Why it matters (JD context):**
- Reduces time-to-market — code changes reach production in minutes, not weeks.
- Reduces risk — small, frequent changes are easier to test and roll back.
- Enables collaboration — Dev and Ops agree on the pipeline definition in Git.
- Enables compliance — every deployment is traceable, auditable, and reproducible.

---

### 🟢 Q2. What are the essential stages in a production-grade CI/CD pipeline?

**Answer:**

```
Code Commit (Git push / Merge Request)
         │
         ▼
    1. SOURCE STAGE
       ├── Checkout code
       └── Set version / tag

         │
         ▼
    2. BUILD STAGE
       ├── Compile / transpile
       ├── Build container image (Buildah / Docker / S2I)
       └── Push image to registry with Git SHA tag

         │
         ▼
    3. TEST STAGE
       ├── Unit tests + coverage report
       ├── Integration tests
       ├── Static code analysis (SonarQube, ESLint)
       └── Dependency vulnerability scan (Trivy, Snyk, OWASP)

         │
         ▼
    4. PUBLISH / PACKAGE STAGE
       ├── Tag image :latest or :v1.2.3
       ├── Sign image (cosign / Notary)
       └── Generate SBOM (Software Bill of Materials)

         │
         ▼
    5. DEPLOY — STAGING
       ├── Update Helm values / Kustomize / GitOps repo
       ├── Wait for ArgoCD/Flux sync
       └── Run smoke tests / acceptance tests

         │
         ▼
    6. DEPLOY — PRODUCTION (manual gate or automated)
       ├── Blue/Green swap or Canary release
       ├── Monitor for errors / rollback threshold
       └── Notify Slack / PagerDuty
```

---

## Jenkins Pipelines

---

### 🟡 Q3. How do you write a production Jenkins Declarative Pipeline for an OpenShift deployment?

**Answer:**

```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins-agent-sa
  containers:
  - name: buildah
    image: quay.io/buildah/stable:latest
    securityContext:
      privileged: false        # OpenShift: use fuse-overlayfs
    volumeMounts:
    - name: varlibcontainers
      mountPath: /var/lib/containers
  - name: oc-cli
    image: registry.redhat.io/openshift4/ose-cli:latest
    command: [cat]
    tty: true
  volumes:
  - name: varlibcontainers
    emptyDir: {}
"""
        }
    }

    environment {
        APP_NAME        = 'my-nodeapp'
        NAMESPACE       = 'production'
        REGISTRY        = 'image-registry.openshift-image-registry.svc:5000'
        IMAGE           = "${REGISTRY}/${NAMESPACE}/${APP_NAME}"
        SONAR_TOKEN     = credentials('sonar-token')
        OC_TOKEN        = credentials('openshift-token')
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG = "${APP_NAME}:${env.GIT_COMMIT_SHORT}"
                }
            }
        }

        stage('Unit Tests') {
            steps {
                container('buildah') {
                    sh 'npm ci && npm test'
                }
            }
            post {
                always {
                    junit 'test-results/*.xml'
                    publishHTML target: [
                        reportDir: 'coverage',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ]
                }
            }
        }

        stage('Code Quality') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh "sonar-scanner -Dsonar.projectKey=${APP_NAME}"
                }
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build & Push Image') {
            steps {
                container('buildah') {
                    sh """
                        buildah bud \
                          --layers \
                          --tag ${IMAGE}:${GIT_COMMIT_SHORT} \
                          --tag ${IMAGE}:latest \
                          .

                        buildah push \
                          --creds=serviceaccount:\$(cat /run/secrets/kubernetes.io/serviceaccount/token) \
                          ${IMAGE}:${GIT_COMMIT_SHORT}

                        buildah push ${IMAGE}:latest
                    """
                }
            }
        }

        stage('Security Scan') {
            steps {
                sh """
                    trivy image \
                      --exit-code 1 \
                      --severity CRITICAL \
                      --ignore-unfixed \
                      ${IMAGE}:${GIT_COMMIT_SHORT}
                """
            }
        }

        stage('Deploy to Staging') {
            steps {
                container('oc-cli') {
                    sh """
                        oc login --token=${OC_TOKEN} --server=https://api.mycluster.example.com:6443
                        oc set image deployment/${APP_NAME} \
                          app=${IMAGE}:${GIT_COMMIT_SHORT} \
                          -n staging
                        oc rollout status deployment/${APP_NAME} -n staging --timeout=5m
                    """
                }
            }
        }

        stage('Approval: Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: "Deploy ${GIT_COMMIT_SHORT} to PRODUCTION?",
                      submitter: 'release-approvers'
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                container('oc-cli') {
                    sh """
                        oc set image deployment/${APP_NAME} \
                          app=${IMAGE}:${GIT_COMMIT_SHORT} \
                          -n production
                        oc rollout status deployment/${APP_NAME} -n production --timeout=10m
                    """
                }
            }
        }
    }

    post {
        success {
            slackSend color: 'good',
                      message: "✅ ${APP_NAME} ${GIT_COMMIT_SHORT} deployed to production"
        }
        failure {
            slackSend color: 'danger',
                      message: "❌ Pipeline FAILED: ${APP_NAME} ${env.STAGE_NAME}"
        }
    }
}
```

---

### 🟡 Q4. How do you handle Jenkins shared libraries?

**Answer:**

Shared libraries allow reusing pipeline logic across multiple Jenkinsfiles.

**Directory structure:**
```
jenkins-shared-library/
├── vars/
│   ├── buildAndPush.groovy     # global functions (used as steps)
│   └── deployToOpenShift.groovy
├── src/
│   └── org/myorg/Utils.groovy  # Groovy classes
└── resources/
    └── k8s/
        └── agent-pod.yaml
```

```groovy
// vars/deployToOpenShift.groovy
def call(Map config) {
    def app     = config.appName
    def ns      = config.namespace
    def image   = config.image
    def timeout = config.get('timeout', '5m')

    container('oc-cli') {
        sh """
            oc set image deployment/${app} app=${image} -n ${ns}
            oc rollout status deployment/${app} -n ${ns} --timeout=${timeout}
        """
    }
}
```

```groovy
// Jenkinsfile consuming the library
@Library('my-shared-library@main') _

pipeline {
    agent { ... }
    stages {
        stage('Deploy') {
            steps {
                deployToOpenShift(
                    appName: 'my-nodeapp',
                    namespace: 'production',
                    image: "registry/my-nodeapp:${GIT_COMMIT_SHORT}"
                )
            }
        }
    }
}
```

**Configure in Jenkins:** Manage Jenkins → System → Global Pipeline Libraries → add Git repo + default version.

---

### 🔴 Q5. How do you manage Jenkins at scale on OpenShift? What are the best practices?

**Answer:**

**Architecture:**
```
Jenkins Controller (1 pod — persistent storage for jobs/config)
    └── Dynamic agents (ephemeral Kubernetes pods — created per build, deleted after)
```

**Best practices:**

1. **Use Kubernetes plugin for dynamic agents** — no static agents, no waste:
```groovy
agent {
    kubernetes {
        yaml readFile('ci/agent-pod.yaml')    # Pod template in source control
        defaultContainer 'builder'
    }
}
```

2. **Controller config-as-code (JCasC):** Store all Jenkins config in YAML managed by Git — no manual UI clicks:
```yaml
# jenkins.yaml (JCasC)
jenkins:
  systemMessage: "Managed by JCasC"
  numExecutors: 0     # No builds on controller
credentials:
  system:
    domainCredentials:
    - credentials:
      - usernamePassword:
          id: "sonar-token"
          username: "admin"
          password: "${SONAR_TOKEN}"   # Env var from Secret
```

3. **Separate controller and agents onto dedicated infra nodes** — taint infra nodes, use tolerations in agent pod templates.

4. **Resource limits on agent pods** — prevent runaway builds from starving the cluster.

5. **Pipeline durability:** Use `durabilityHint('PERFORMANCE_OPTIMIZED')` for most pipelines; only `MAX_SURVIVABILITY` for critical long-running jobs.

6. **Shared libraries in Git** — version-controlled, tested, auditable.

7. **Backup:** Back up `$JENKINS_HOME/jobs` and JCasC YAML regularly.

---

## GitLab CI/CD

---

### 🟡 Q6. Write a GitLab CI pipeline that builds, tests, scans, and deploys to OpenShift.

**Answer:**

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - scan
  - deploy-staging
  - deploy-production

variables:
  APP_NAME: my-nodeapp
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  OCP_SERVER: https://api.mycluster.example.com:6443
  STAGING_NS: staging
  PROD_NS: production

# ── BUILD ─────────────────────────────────────────────
build-image:
  stage: build
  image: quay.io/buildah/stable
  script:
    - buildah bud --layers -t $IMAGE .
    - buildah push --creds=$CI_REGISTRY_USER:$CI_REGISTRY_PASSWORD $IMAGE
  only:
    - main
    - merge_requests

# ── TEST ──────────────────────────────────────────────
unit-tests:
  stage: test
  image: node:18-alpine
  script:
    - npm ci
    - npm test -- --coverage
  coverage: '/Lines\s*:\s*(\d+\.\d+)%/'
  artifacts:
    when: always
    reports:
      junit: test-results/junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

sonarqube:
  stage: test
  image: sonarsource/sonar-scanner-cli
  script:
    - sonar-scanner
      -Dsonar.projectKey=$APP_NAME
      -Dsonar.host.url=$SONAR_HOST_URL
      -Dsonar.login=$SONAR_TOKEN
  allow_failure: false

# ── SCAN ──────────────────────────────────────────────
trivy-scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy image
        --exit-code 1
        --severity CRITICAL,HIGH
        --ignore-unfixed
        $IMAGE
  artifacts:
    reports:
      container_scanning: trivy-report.json

# ── DEPLOY STAGING ─────────────────────────────────────
deploy-staging:
  stage: deploy-staging
  image: registry.redhat.io/openshift4/ose-cli:latest
  environment:
    name: staging
    url: https://myapp-staging.apps.mycluster.example.com
  script:
    - oc login --token=$OCP_TOKEN --server=$OCP_SERVER --insecure-skip-tls-verify
    - oc set image deployment/$APP_NAME app=$IMAGE -n $STAGING_NS
    - oc rollout status deployment/$APP_NAME -n $STAGING_NS --timeout=5m
  only:
    - main

# ── DEPLOY PRODUCTION (manual gate) ──────────────────
deploy-production:
  stage: deploy-production
  image: registry.redhat.io/openshift4/ose-cli:latest
  environment:
    name: production
    url: https://myapp.apps.mycluster.example.com
  when: manual                # Requires human approval in GitLab UI
  script:
    - oc login --token=$OCP_TOKEN_PROD --server=$OCP_SERVER
    - oc set image deployment/$APP_NAME app=$IMAGE -n $PROD_NS
    - oc rollout status deployment/$APP_NAME -n $PROD_NS --timeout=10m
  only:
    - main
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
      allow_failure: false
```

---

### 🟡 Q7. What is the difference between GitLab CI runners and Jenkins agents?

**Answer:**

| Feature | GitLab Runner | Jenkins Agent |
|---------|--------------|--------------|
| Configuration | `.gitlab-ci.yml` in repo | Jenkinsfile + Global config |
| Executor types | Docker, Kubernetes, Shell, SSH, VirtualBox | Kubernetes pod, Docker, SSH, Static node |
| Agent lifecycle | Ephemeral containers (Docker/K8s executor) | Ephemeral (K8s plugin) or persistent |
| Scaling | Runner auto-scales via K8s executor | K8s plugin creates pods on demand |
| Auth | Runner token registered to project/group | SA token in Kubernetes plugin |
| OpenShift native | GitLab Runner image available for OCP | Jenkins Operator available in OperatorHub |

**GitLab Runner on OpenShift:**
```yaml
# values.yaml for gitlab-runner Helm chart
gitlabUrl: https://gitlab.example.com
runnerRegistrationToken: "your-token"
runners:
  executor: kubernetes
  kubernetes:
    namespace: gitlab-runners
    image: ubuntu:22.04
    privileged: false
    serviceAccountName: gitlab-runner-sa
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: "2"
        memory: 2Gi
```

---

## GitHub Actions

---

### 🟢 Q8. What is GitHub Actions and how does it fit into a CI/CD workflow?

**Answer:**

GitHub Actions is a **native CI/CD platform built into GitHub** that lets you automate workflows triggered by repository events (push, pull request, release, schedule, etc.). Workflows are defined as YAML files stored under `.github/workflows/` and consist of **jobs** running on **runners** (GitHub-hosted or self-hosted).

**Key concepts:**

| Concept | Description |
|---------|-------------|
| **Workflow** | YAML file describing the automation (`.github/workflows/*.yml`) |
| **Event** | Trigger — `push`, `pull_request`, `schedule`, `workflow_dispatch`, etc. |
| **Job** | Set of steps that run on the same runner; jobs run in parallel by default |
| **Step** | Individual shell command or reusable **Action** |
| **Action** | Reusable unit of work (composite, Docker, or JavaScript) from the Marketplace |
| **Runner** | VM or container that executes jobs; GitHub-hosted (`ubuntu-latest`, `windows-latest`) or self-hosted |
| **Context** | Runtime variables — `github`, `env`, `secrets`, `vars`, `matrix` |

**Comparison with Jenkins:**

| Feature | GitHub Actions | Jenkins |
|---------|---------------|---------|
| Configuration | YAML in repo (`.github/workflows/`) | Groovy `Jenkinsfile` in repo or SCM |
| Hosting | SaaS (GitHub-hosted) or self-hosted runners | Self-hosted always |
| Marketplace | 20,000+ reusable Actions | Plugin ecosystem (1,800+) |
| Secrets | Encrypted repo / org / environment secrets | Credentials plugin |
| Kubernetes support | Actions Runner Controller (ARC) | Kubernetes plugin |
| Maintenance overhead | Minimal (GitHub-hosted) | High (agents, plugins, upgrades) |
| Audit / OIDC | Built-in OIDC for cloud auth | Manual setup |

---

### 🟡 Q9. Write a production-grade GitHub Actions workflow that builds, tests, scans, and deploys to OpenShift.

**Answer:**

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD – Build, Test, Scan & Deploy to OpenShift

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  APP_NAME: my-nodeapp
  REGISTRY: ghcr.io
  IMAGE: ghcr.io/${{ github.repository }}:${{ github.sha }}
  OCP_SERVER: https://api.mycluster.example.com:6443
  STAGING_NS: staging
  PROD_NS: production

jobs:
  # ── BUILD ────────────────────────────────────────────────
  build:
    name: Build & Push Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write        # Required to push to GHCR
    outputs:
      image: ${{ env.IMAGE }}
    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ${{ env.IMAGE }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ── TEST ─────────────────────────────────────────────────
  test:
    name: Unit Tests & Code Coverage
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests with coverage
        run: npm test -- --coverage --ci

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

      - name: SonarQube Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  # ── SECURITY SCAN ────────────────────────────────────────
  scan:
    name: Container Security Scan
    runs-on: ubuntu-latest
    needs: build
    permissions:
      security-events: write  # Required to upload SARIF to GitHub Security tab
    steps:
      - name: Run Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: '1'
          ignore-unfixed: true

      - name: Upload Trivy SARIF to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif

  # ── DEPLOY STAGING ───────────────────────────────────────
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [test, scan]
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://myapp-staging.apps.mycluster.example.com
    steps:
      - uses: actions/checkout@v4

      - name: Install OpenShift CLI
        uses: redhat-actions/openshift-tools-installer@v1
        with:
          oc: latest

      - name: Authenticate to OpenShift (OIDC)
        uses: redhat-actions/oc-login@v1
        with:
          openshift_server_url: ${{ env.OCP_SERVER }}
          openshift_token: ${{ secrets.OCP_STAGING_TOKEN }}
          insecure_skip_tls_verify: false

      - name: Update image and roll out
        run: |
          oc set image deployment/${{ env.APP_NAME }} \
            app=${{ env.IMAGE }} -n ${{ env.STAGING_NS }}
          oc rollout status deployment/${{ env.APP_NAME }} \
            -n ${{ env.STAGING_NS }} --timeout=5m

  # ── DEPLOY PRODUCTION (manual gate) ─────────────────────
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment:
      name: production            # Requires a reviewer to approve in GitHub UI
      url: https://myapp.apps.mycluster.example.com
    steps:
      - uses: actions/checkout@v4

      - name: Install OpenShift CLI
        uses: redhat-actions/openshift-tools-installer@v1
        with:
          oc: latest

      - name: Authenticate to OpenShift (Production)
        uses: redhat-actions/oc-login@v1
        with:
          openshift_server_url: ${{ env.OCP_SERVER }}
          openshift_token: ${{ secrets.OCP_PROD_TOKEN }}

      - name: Update image and roll out
        run: |
          oc set image deployment/${{ env.APP_NAME }} \
            app=${{ env.IMAGE }} -n ${{ env.PROD_NS }}
          oc rollout status deployment/${{ env.APP_NAME }} \
            -n ${{ env.PROD_NS }} --timeout=10m
```

**Key best practices applied:**
- **OIDC / short-lived tokens** — use `redhat-actions/oc-login` with a service-account token stored as an encrypted secret; rotate regularly.
- **Environment protection rules** — the `production` environment in GitHub Settings requires a designated reviewer before the job runs.
- **Layer caching** — `cache-from/to: type=gha` speeds up repeated builds significantly.
- **SARIF upload** — Trivy findings surface directly in the GitHub Security tab without leaving the platform.
- **`permissions` blocks** — follow the principle of least privilege; each job declares only the token permissions it needs.

---

### 🟡 Q10. How do you use GitHub Actions with self-hosted runners on OpenShift (Actions Runner Controller)?

**Answer:**

**Actions Runner Controller (ARC)** is an operator that manages self-hosted GitHub Actions runners as Kubernetes pods. It supports **ephemeral runners** (scale to zero, one pod per job) which is ideal for OpenShift.

**Install ARC via Helm:**
```bash
helm install arc \
  --namespace arc-systems \
  --create-namespace \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

**Deploy a runner scale set:**
```yaml
# runner-scale-set.yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: arc-runner-set
  namespace: arc-runners
spec:
  chart:
    spec:
      chart: gha-runner-scale-set
      sourceRef:
        kind: HelmRepository
        name: arc
      version: ">=0.9.0"
  values:
    githubConfigUrl: https://github.com/my-org/my-repo
    githubConfigSecret: arc-github-secret   # contains GITHUB_PAT or App credentials
    minRunners: 0
    maxRunners: 10
    containerMode:
      type: kubernetes                        # Kubernetes dind-less mode
    template:
      spec:
        serviceAccountName: arc-runner-sa
        containers:
          - name: runner
            image: ghcr.io/actions/actions-runner:latest
            resources:
              requests:
                cpu: 500m
                memory: 512Mi
              limits:
                cpu: "2"
                memory: 2Gi
```

**Target the self-hosted runner in a workflow:**
```yaml
jobs:
  build:
    runs-on: arc-runner-set     # matches the scale set name
```

**OpenShift-specific considerations:**
- Grant the runner ServiceAccount the `anyuid` SCC only if absolutely necessary; prefer `restricted-v2`.
- Use image pull secrets for private registries.
- Store the GitHub PAT / App private key in an OpenShift `Secret`; never in plaintext.

---

### 🟡 Q11. How do you manage secrets in GitHub Actions securely?

**Answer:**

**Three tiers of secrets storage:**

| Scope | Where set | Accessible by |
|-------|-----------|---------------|
| Repository secret | Repo → Settings → Secrets | All workflows in that repo |
| Environment secret | Repo → Settings → Environments | Jobs targeting that environment |
| Organisation secret | Org → Settings → Secrets | Selected repos in the org |

**Best practices:**
1. **Use OIDC (keyless auth)** — For AWS, Azure, GCP, and Vault, use OIDC federation so no static credentials are stored at all:
```yaml
permissions:
  id-token: write
  contents: read

- name: Configure AWS credentials via OIDC
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions-role
    aws-region: us-east-1
```
2. **Mask dynamic secrets** — `echo "::add-mask::$SECRET_VALUE"` prevents the value from appearing in logs.
3. **Environment protection rules** — Require a manual approval before environment secrets are surfaced to a job.
4. **Rotate regularly** — Automated rotation via Vault Dynamic Secrets or AWS Secrets Manager integration.
5. **Never echo secrets** — GitHub auto-redacts registered secrets in logs but any derived value is not auto-masked.

---

## OpenShift Pipelines (Tekton)

---

### 🟡 Q12. What is Tekton and how does it differ from Jenkins?

**Answer:**

| Feature | Jenkins | Tekton (OpenShift Pipelines) |
|---------|---------|-------------------------------|
| Architecture | Controller + agents | Pure Kubernetes CRDs + pods |
| Pipeline definition | Groovy DSL (Jenkinsfile) | YAML (Task, Pipeline objects) |
| Execution | JVM on controller | Each Task runs as a Kubernetes Pod |
| State | Stored in Jenkins controller | Stored in Kubernetes (TaskRun, PipelineRun CRDs) |
| Scaling | Depends on Jenkins + K8s plugin | Native Kubernetes scaling |
| Reuse | Shared libraries | ClusterTask, Tekton Hub |
| Learning curve | Groovy + Jenkins knowledge | YAML + Kubernetes knowledge |
| OpenShift integration | Via Jenkins Operator | Native, included with OCP |

**When to choose Jenkins:** Existing enterprise adoption, complex Groovy logic, many existing Jenkinsfiles.

**When to choose Tekton:** Greenfield, Kubernetes-native CI/CD, OpenShift-first, GitOps-aligned.

---

### 🟡 Q13. Write a Tekton pipeline for building and deploying a container image.

**Answer:**

```yaml
# Task: run unit tests
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: run-tests
spec:
  workspaces:
  - name: source
  steps:
  - name: test
    image: node:18-alpine
    workingDir: $(workspaces.source.path)
    script: |
      #!/bin/sh
      npm ci
      npm test

---
# Pipeline
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-deploy
  namespace: my-app
spec:
  params:
  - name: GIT_URL
    type: string
  - name: IMAGE
    type: string
  - name: NAMESPACE
    type: string
    default: my-app

  workspaces:
  - name: shared-workspace
  - name: docker-credentials

  tasks:

  - name: clone
    taskRef:
      name: git-clone
      kind: ClusterTask
    workspaces:
    - name: output
      workspace: shared-workspace
    params:
    - name: url
      value: $(params.GIT_URL)
    - name: revision
      value: main

  - name: test
    taskRef:
      name: run-tests
    runAfter: [clone]
    workspaces:
    - name: source
      workspace: shared-workspace

  - name: build-push
    taskRef:
      name: buildah
      kind: ClusterTask
    runAfter: [test]
    workspaces:
    - name: source
      workspace: shared-workspace
    - name: dockerconfig
      workspace: docker-credentials
    params:
    - name: IMAGE
      value: $(params.IMAGE)
    - name: DOCKERFILE
      value: ./Dockerfile
    - name: TLSVERIFY
      value: "false"     # Internal registry

  - name: deploy
    taskRef:
      name: openshift-client
      kind: ClusterTask
    runAfter: [build-push]
    params:
    - name: SCRIPT
      value: |
        oc set image deployment/my-nodeapp \
          app=$(params.IMAGE) \
          -n $(params.NAMESPACE)
        oc rollout status deployment/my-nodeapp \
          -n $(params.NAMESPACE) --timeout=5m

---
# Trigger via webhook
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: github-listener
spec:
  triggers:
  - name: github-push
    interceptors:
    - ref:
        name: github
      params:
      - name: secretRef
        value:
          secretName: github-webhook-secret
          secretKey: token
      - name: eventTypes
        value: ["push"]
    bindings:
    - ref: github-push-binding
    template:
      ref: build-deploy-template
```

---

## GitOps & ArgoCD

---

### 🟡 Q14. What is GitOps and what are its core principles?

**Answer:**

GitOps is an operational model where Git is the **single source of truth** for both application code and infrastructure state.

**Four core principles:**

| Principle | Description |
|-----------|-------------|
| **Declarative** | Desired system state is expressed declaratively (YAML manifests, Helm charts, Kustomize) |
| **Versioned** | Desired state is stored in Git — full audit history, rollback via `git revert` |
| **Automatically pulled** | A software agent (ArgoCD, Flux) continuously compares desired state (Git) to actual state (cluster) |
| **Continuously reconciled** | Divergence is detected and corrected automatically (self-healing) |

**GitOps workflow:**
```
Developer → git push → Git repo (app manifests)
                           │
                    ArgoCD/Flux detects diff
                           │
                    Syncs to OpenShift cluster
                           │
                    Sends notification (Slack/Webhook)
```

**Separation of concerns:**
- **CI pipeline** (Jenkins/GitLab): build image → push to registry → update image tag in GitOps repo.
- **CD agent** (ArgoCD): watches GitOps repo → applies changes to cluster.

---

### 🟡 Q15. Explain ArgoCD sync policies and how you would set up a production GitOps flow.

**Answer:**

**ArgoCD Application with automated sync:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-nodeapp-production
  namespace: openshift-gitops
spec:
  project: default

  source:
    repoURL: https://github.com/myorg/gitops-repo.git
    targetRevision: main
    path: environments/production
    # Kustomize overlay
    kustomize:
      images:
      - myapp=image-registry.openshift-image-registry.svc:5000/prod/myapp:abc1234

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true          # Delete resources removed from Git
      selfHeal: true       # Revert manual oc apply / kubectl apply changes
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
    - ApplyOutOfSyncOnly=true   # Only sync changed resources (faster)
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**GitOps repo structure:**
```
gitops-repo/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── environments/
│   ├── staging/
│   │   ├── kustomization.yaml     # patches staging-specific values
│   │   └── replica-patch.yaml
│   └── production/
│       ├── kustomization.yaml     # patches production-specific values
│       └── replica-patch.yaml
└── apps/
    └── app-of-apps.yaml           # ArgoCD ApplicationSet
```

**CI pipeline updates image tag in GitOps repo (automated PR or direct push):**
```bash
# In CI pipeline after successful build
cd gitops-repo
kustomize edit set image myapp=${REGISTRY}/myapp:${GIT_SHA}
git add .
git commit -m "ci: update myapp to ${GIT_SHA}"
git push
# ArgoCD detects change → syncs to cluster
```

---

## Deployment Strategies

---

### 🟡 Q16. Compare Blue/Green, Canary, and Rolling deployment strategies. When would you use each?

**Answer:**

| Strategy | Mechanism | Downtime | Risk | Rollback speed | Use case |
|----------|-----------|----------|------|----------------|---------|
| **Rolling** | Replace pods N at a time | Zero | Low–Medium | ~minutes | Standard app updates |
| **Blue/Green** | Run two full environments; switch LB | Zero | Low | Instant | Breaking schema changes, high-stakes releases |
| **Canary** | Route % of traffic to new version | Zero | Very low | Instant | New features, gradual risk reduction |
| **Recreate** | Kill all old pods, start new | Yes | High | ~minutes | Dev/test only |

**Blue/Green on OpenShift:**
```bash
# Two Deployments: blue (current) and green (new)
oc apply -f deployment-green.yaml

# Wait for green to be ready
oc rollout status deployment/myapp-green

# Switch Route traffic from blue to green
oc patch route myapp -p '{"spec":{"to":{"name":"myapp-green-svc"}}}'

# Keep blue running for rollback
# oc patch route myapp -p '{"spec":{"to":{"name":"myapp-blue-svc"}}}'
```

**Canary on OpenShift (weighted routes):**
```yaml
# Route splits: 90% to stable, 10% to canary
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myapp
spec:
  to:
    kind: Service
    name: myapp-stable-svc
    weight: 90
  alternateBackends:
  - kind: Service
    name: myapp-canary-svc
    weight: 10
```

---

### 🟡 Q17. How do you implement automated rollback in a CI/CD pipeline?

**Answer:**

**Strategy 1 — Kubernetes rollout undo (fast):**
```bash
# In pipeline post-deploy health check
TIMEOUT=120
for i in $(seq 1 $TIMEOUT); do
    STATUS=$(oc get deployment myapp -n production \
        -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')
    if [ "$STATUS" == "True" ]; then
        echo "✅ Deployment healthy"
        break
    fi
    if [ "$i" -eq "$TIMEOUT" ]; then
        echo "❌ Deployment unhealthy — rolling back"
        oc rollout undo deployment/myapp -n production
        exit 1
    fi
    sleep 1
done
```

**Strategy 2 — GitOps revert (auditable):**
```bash
# In pipeline
git revert HEAD --no-edit
git push
# ArgoCD detects revert → syncs old image tag → cluster rolls back
```

**Strategy 3 — Argo Rollouts (progressive delivery with auto-rollback):**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 100
      analysis:
        templates:
        - templateName: success-rate
      args:
      - name: service-name
        value: myapp-canary-svc
  # If success rate < 95%, auto-rollback to previous version
```

---

### 🟡 Q18. How do you handle secrets in a CI/CD pipeline securely?

**Answer:**

**Anti-patterns to avoid:**
- ❌ Hardcoding secrets in Jenkinsfile or `.gitlab-ci.yml`
- ❌ Storing secrets in Git (even in private repos)
- ❌ Logging secrets in build output
- ❌ Passing secrets as build args in Dockerfiles (they appear in image history)

**Correct patterns:**

**Jenkins Credentials API:**
```groovy
withCredentials([
    string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN'),
    usernamePassword(credentialsId: 'registry-creds',
                     usernameVariable: 'REG_USER',
                     passwordVariable: 'REG_PASS')
]) {
    sh 'buildah push --creds=$REG_USER:$REG_PASS ${IMAGE}'
}
// $SONAR_TOKEN is masked in logs
```

**GitLab CI protected variables:**
- Defined at project/group level in UI → Settings → CI/CD → Variables.
- `Protected` flag: only available on protected branches.
- `Masked` flag: value never printed in job logs.

**OpenShift Pipelines (Tekton) + Secret:**
```yaml
# Mount credentials secret into pipeline workspace
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
  annotations:
    tekton.dev/docker-0: image-registry.openshift-image-registry.svc:5000
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64>
```

**Best practice — HashiCorp Vault + Vault Agent Injector or External Secrets Operator:**
```yaml
# Secret pulled from Vault at pod startup — never stored in etcd
annotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "myapp"
  vault.hashicorp.com/agent-inject-secret-config: "secret/myapp/production"
```

---

### 🔴 Q19. How do you implement multi-environment promotion in a CI/CD pipeline?

**Answer:**

**Recommended pattern: GitOps with promotion via PR/merge:**

```
Git branches / tags:
  feature-*  →  dev namespace     (auto-deploy on push)
  main       →  staging namespace (auto-deploy on merge)
  release/*  →  production namespace (manual approval + PR to prod branch)
```

**GitOps promotion workflow:**
```bash
# CI pipeline after successful staging tests:

# 1. Create promotion PR to production GitOps branch
gh pr create \
  --title "Promote myapp ${GIT_SHA} to production" \
  --body "Tested in staging. Trivy scan passed. SonarQube gate passed." \
  --base production \
  --head main \
  --reviewer "release-approvers"

# 2. Approver reviews diff (just the image tag change) and merges
# 3. ArgoCD on production cluster detects change → deploys
```

**OpenShift-specific: use separate projects per environment:**
```
project: dev        (auto-sync from dev branch)
project: staging    (auto-sync from main branch)
project: production (manual sync, requires PR approval)
```

**Image promotion (no rebuild for production):**
```bash
# Promote the exact same immutable image SHA from staging to production
# Never rebuild for production — use the already-tested image

STAGING_SHA=$(oc get deployment myapp -n staging \
  -o jsonpath='{.spec.template.spec.containers[0].image}' | cut -d: -f2)

# Tag the staging-tested image as production-approved
oc tag staging/myapp:${STAGING_SHA} production/myapp:production-approved
```

---

### 🔴 Q20. How do you measure CI/CD pipeline performance and what are the key DORA metrics?

**Answer:**

**DORA (DevOps Research and Assessment) Metrics:**

| Metric | Definition | Elite benchmark |
|--------|-----------|----------------|
| **Deployment Frequency** | How often you deploy to production | Multiple times per day |
| **Lead Time for Changes** | Time from code commit to production | < 1 hour |
| **Change Failure Rate** | % of deployments causing incidents | < 5% |
| **MTTR** (Mean Time to Restore) | Time to recover from a failure | < 1 hour |

**How to collect:**
- **Deployment Frequency:** Track PipelineRun events in Tekton or Jenkins build history; export to Prometheus.
- **Lead Time:** Commit timestamp (Git webhook) → Deployment timestamp (Kubernetes event) — use Pelorus (OpenShift DORA tool).
- **Change Failure Rate:** Link deployments to incidents (PagerDuty/Jira) — # incidents caused by deployments / total deployments.
- **MTTR:** Incident open time → incident close time in incident management tool.

**OpenShift Pelorus:**
```bash
# Red Hat's DORA metrics tool for OpenShift
helm install pelorus charts/pelorus \
  --namespace pelorus \
  --set exporters.deployment_frequency.enabled=true \
  --set exporters.lead_time.enabled=true
```

---

### 🟡 Q21. What is the difference between `docker build` and `buildah` in an OpenShift context?

**Answer:**

| Feature | docker build | buildah |
|---------|-------------|---------|
| Daemon required | Yes (Docker daemon) | No (daemonless) |
| Root required | Usually yes | No (rootless mode) |
| OpenShift compatible | Limited (Docker socket not available) | Yes (fuse-overlayfs) |
| OCI compliance | Yes | Yes (fully OCI-compliant) |
| Build context | Docker protocol | OCI standard |
| Multi-stage | Yes | Yes |

**Why buildah in OpenShift:**
OpenShift blocks privileged containers by default. Docker-in-Docker (mounting `/var/run/docker.sock`) requires `privileged: true`. Buildah uses `fuse-overlayfs` and works under the `anyuid` SCC without full privilege:

```bash
# Buildah in OpenShift pipeline pod (no privileged needed)
buildah bud \
  --storage-driver=vfs \
  --format=oci \
  -t $IMAGE \
  .

buildah push \
  --tls-verify=false \
  $IMAGE
```

**Alternative: Kaniko** (also daemonless, runs as unprivileged container):
```yaml
- name: build
  image: gcr.io/kaniko-project/executor:latest
  args:
  - --context=dir://$(workspaces.source.path)
  - --destination=$(params.IMAGE)
  - --insecure          # for internal registry
```

---

### 🟡 Q22. How do you trigger a pipeline on a git push / merge request in OpenShift Pipelines?

**Answer:**

OpenShift Pipelines uses **Triggers** (Tekton Triggers):

```yaml
# 1. Secret for webhook validation
apiVersion: v1
kind: Secret
metadata:
  name: github-webhook-secret
stringData:
  token: "mysecrettoken"

---
# 2. TriggerBinding — extract values from the webhook payload
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: github-push-binding
spec:
  params:
  - name: git-url
    value: $(body.repository.clone_url)
  - name: git-revision
    value: $(body.head_commit.id)

---
# 3. TriggerTemplate — creates a PipelineRun
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: build-deploy-template
spec:
  params:
  - name: git-url
  - name: git-revision
  resourcetemplates:
  - apiVersion: tekton.dev/v1
    kind: PipelineRun
    metadata:
      generateName: build-deploy-run-
    spec:
      pipelineRef:
        name: build-deploy
      params:
      - name: GIT_URL
        value: $(tt.params.git-url)
      - name: IMAGE
        value: image-registry.openshift-image-registry.svc:5000/my-app/myapp:$(tt.params.git-revision)
      workspaces:
      - name: shared-workspace
        persistentVolumeClaim:
          claimName: pipeline-pvc

---
# 4. EventListener — HTTP endpoint (becomes a Service + Route)
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: github-listener
spec:
  serviceAccountName: pipeline
  triggers:
  - name: github-push
    interceptors:
    - ref:
        name: github
      params:
      - name: secretRef
        value:
          secretName: github-webhook-secret
          secretKey: token
      - name: eventTypes
        value: [push]
      - name: branches
        value: [main]
    bindings:
    - ref: github-push-binding
    template:
      ref: build-deploy-template
```

```bash
# Get the EventListener URL to configure in GitHub webhook
oc get route el-github-listener -n my-app -o jsonpath='{.spec.host}'
# Set this as the Payload URL in GitHub → Settings → Webhooks
```

---

### 🔴 Q23. How do you ensure pipeline idempotency and handle flaky tests?

**Answer:**

**Pipeline idempotency** means running the pipeline multiple times with the same input produces the same result — no side effects from partial runs.

**Practices:**
1. **Immutable image tags:** Always tag with Git commit SHA, never overwrite `:latest` in CI. `:latest` is for convenience only.
2. **Declarative deployments:** `oc apply` is idempotent — run it 10 times, same result.
3. **Database migrations:** Use tools like Flyway/Liquibase with versioned migration scripts — they track which migrations ran.
4. **Cleanup steps:** Always clean workspace before build (`git clean -fdx`).
5. **Retry logic:** Wrap network-dependent steps (image push, deploy) in retry loops.

**Handling flaky tests:**
```groovy
// Jenkins: retry flaky test stage
stage('Integration Tests') {
    retry(3) {
        sh 'npm run test:integration'
    }
}
```

```yaml
# GitLab CI: retry on failure
integration-tests:
  retry:
    max: 2
    when:
      - script_failure
      - runner_system_failure
```

**Quarantine pattern:** Flaky tests are moved to a `quarantine` suite. They still run but don't block the pipeline. A separate nightly job tracks quarantine tests.

---

### 🔴 Q24. How do you implement compliance gates in a CI/CD pipeline?

**Answer:**

Compliance gates are mandatory checks that must pass before a deployment proceeds.

**Example gates for enterprise environments:**

| Gate | Tool | Blocking |
|------|------|---------|
| SAST (static code analysis) | SonarQube quality gate | Yes |
| Container vulnerability scan | Trivy, Clair, Snyk | Yes (CRITICAL) |
| License compliance | FOSSA, Black Duck | Yes (GPL violations) |
| Secret detection | Gitleaks, GitGuardian | Yes |
| Image signing verification | cosign + Sigstore | Yes |
| Change Management ticket | ServiceNow API check | Yes (prod only) |

```groovy
// Jenkins — enforce change ticket before production deploy
stage('Change Management Gate') {
    when { environment name: 'DEPLOY_ENV', value: 'production' }
    steps {
        script {
            def response = httpRequest(
                url: "https://servicenow.example.com/api/now/table/change_request?number=${CHANGE_TICKET}",
                authentication: 'snow-credentials'
            )
            def data = readJSON text: response.content
            if (data.result[0].state != 'Implement') {
                error("Change ticket ${CHANGE_TICKET} is not in Implement state — aborting")
            }
        }
    }
}
```

**OpenShift image signing with cosign:**
```bash
# Sign the image after build
cosign sign \
  --key k8s://cosign-namespace/cosign-key \
  ${IMAGE}@${IMAGE_DIGEST}

# Verify before deploy (enforced by Kyverno policy)
cosign verify \
  --key k8s://cosign-namespace/cosign-key \
  ${IMAGE}
```

---

> **Next:** [Security & Compliance →](./03-Security-Compliance.md)
