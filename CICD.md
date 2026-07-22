---
render_with_liquid: false
layout: default
title: "🔄 CI/CD — DevOps Interview Guide"
---

# 🔄 CI/CD — DevOps Interview Guide

> [← GCP](./GCP.md) | [Main Index](./README.md) | [Monitoring →](./Monitoring.md)

---

## Table of Contents

1. [CI/CD Fundamentals](#cicd-fundamentals)
2. [Pipeline Design Patterns](#pipeline-design-patterns)
3. [Deployment Strategies](#deployment-strategies)
4. [GitOps](#gitops)
5. [ArgoCD](#argocd)
6. [FluxCD](#fluxcd)
7. [GitHub Actions](#github-actions)
8. [Interview Questions](#interview-questions)
9. [Scenario-Based Questions](#scenario-based-questions)
10. [Hands-On Labs](#hands-on-labs)
11. [Cheat Sheet](#cheat-sheet)

---

## CI/CD Fundamentals

### What is CI/CD?

```
Continuous Integration (CI)
  → Developers merge code frequently
  → Automated build + test on every commit
  → Catch bugs early, reduce integration problems

Continuous Delivery (CD)
  → Every passing build is deployable
  → Deployment to production is a manual decision
  → Release is always possible

Continuous Deployment (CD)
  → Every passing build IS deployed to production automatically
  → No human gate
  → Requires very strong automated testing
```

### The CI/CD Pipeline

```
Code Commit
    │
    ▼
Source Control Trigger (webhook)
    │
    ▼
CI Pipeline
├── Build (compile, package)
├── Unit Tests
├── Code Quality (lint, SAST)
├── Security Scan (deps, SBOM)
└── Build & Push Artifact/Image
    │
    ▼
CD Pipeline
├── Deploy to Dev (auto)
├── Integration/E2E Tests
├── Deploy to Staging (auto)
├── Performance Tests
├── Approval Gate (manual / automatic)
└── Deploy to Production
    │
    ▼
Post-Deployment
├── Smoke Tests
├── Monitor Metrics
└── Rollback if needed
```

### Key Metrics

| Metric | Description | Target |
|--------|-------------|--------|
| **Lead Time** | Code commit → production | Hours (elite) |
| **Deployment Frequency** | How often you deploy | Multiple/day (elite) |
| **MTTR** | Mean Time To Restore | < 1 hour (elite) |
| **Change Failure Rate** | % of deploys causing incidents | < 5% (elite) |
| **Build Duration** | How long CI takes | < 10 minutes |
| **Test Coverage** | % of code tested | > 80% |

---

## Pipeline Design Patterns

### Trunk-Based CI Pipeline (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-test:
    runs-on: ubuntu-latest
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
      image_digest: ${{ steps.build.outputs.digest }}

    steps:
    - uses: actions/checkout@v4

    - name: Set up Go
      uses: actions/setup-go@v5
      with:
        go-version: '1.21'
        cache: true

    - name: Run tests
      run: |
        go test ./... -v -coverprofile=coverage.out
        go tool cover -func=coverage.out

    - name: Upload coverage
      uses: codecov/codecov-action@v4
      with:
        file: ./coverage.out

    - name: Run golangci-lint
      uses: golangci/golangci-lint-action@v4

    - name: Security scan (Gosec)
      uses: securego/gosec@master
      with:
        args: './...'

    - name: Build binary
      run: go build -ldflags="-s -w" -o ./dist/myapp ./cmd/myapp

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to GitHub Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Docker metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,prefix=,suffix=,format=short
          type=ref,event=branch
          type=semver,pattern={{version}}

    - name: Build and push
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        push: ${{ github.event_name != 'pull_request' }}
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        sbom: true
        provenance: true

    - name: Scan image with Trivy
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        format: sarif
        output: trivy-results.sarif
        exit-code: '1'
        severity: 'CRITICAL,HIGH'

    - name: Upload Trivy results to GitHub Security
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: trivy-results.sarif

  deploy-staging:
    needs: build-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging.myapp.example.com

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/github-actions-staging
        aws-region: us-east-1

    - name: Deploy to staging EKS
      run: |
        aws eks update-kubeconfig --name staging-cluster --region us-east-1
        helm upgrade --install myapp ./helm/myapp \
          --namespace staging \
          --set image.tag=${{ github.sha }} \
          --atomic --timeout 5m

    - name: Run smoke tests
      run: |
        curl -f https://staging.myapp.example.com/health
        npm run test:smoke -- --url=https://staging.myapp.example.com

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.example.com

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/github-actions-production
        aws-region: us-east-1

    - name: Deploy to production EKS
      run: |
        aws eks update-kubeconfig --name prod-cluster --region us-east-1
        helm upgrade --install myapp ./helm/myapp \
          --namespace production \
          --set image.tag=${{ github.sha }} \
          --set replicaCount=5 \
          --atomic --timeout 10m

    - name: Verify deployment
      run: |
        kubectl rollout status deployment/myapp -n production --timeout=5m
        curl -f https://myapp.example.com/health
```

### Reusable Workflows

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      image_tag:
        required: true
        type: string
      replica_count:
        default: 2
        type: number
    secrets:
      role_arn:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
    - uses: actions/checkout@v4
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ secrets.role_arn }}
        aws-region: us-east-1
    - name: Deploy
      run: |
        helm upgrade --install myapp ./helm/myapp \
          --namespace ${{ inputs.environment }} \
          --set image.tag=${{ inputs.image_tag }} \
          --set replicaCount=${{ inputs.replica_count }} \
          --atomic

# Calling the reusable workflow
# .github/workflows/main.yml
jobs:
  deploy-prod:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
      image_tag: ${{ github.sha }}
      replica_count: 5
    secrets:
      role_arn: ${{ secrets.PROD_ROLE_ARN }}
```

---

## Deployment Strategies

### Strategy Comparison

| Strategy | Downtime | Risk | Rollback | Resource Cost |
|----------|----------|------|---------|---------------|
| **Recreate** | Yes | High | Redeploy | Low |
| **Rolling Update** | No | Medium | Rollback cmd | Low |
| **Blue-Green** | No | Low | Traffic switch | 2x |
| **Canary** | No | Very Low | Traffic shift | Slightly higher |
| **A/B Testing** | No | Very Low | Traffic shift | Slightly higher |
| **Shadow** | No | None | N/A | 2x |

### Rolling Update

```yaml
# Kubernetes Deployment with rolling update
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2           # Can have 2 extra pods during update
      maxUnavailable: 0     # Zero downtime — never reduce below desired count
```

### Canary Deployment with Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:stable
        resources:
          requests: { cpu: 100m, memory: 128Mi }

  strategy:
    canary:
      canaryService: myapp-canary
      stableService: myapp-stable
      trafficRouting:
        nginx:
          stableIngress: myapp-ingress
      steps:
      - setWeight: 5           # 5% traffic to canary
      - pause: {duration: 5m}  # Wait 5 minutes
      - analysis:              # Automated analysis
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: myapp-canary
      - setWeight: 20
      - pause: {duration: 10m}
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100         # Full rollout
      maxSurge: "20%"
      maxUnavailable: 0

  revisionHistoryLimit: 5
```

### Blue-Green with Argo Rollouts

```yaml
strategy:
  blueGreen:
    activeService: myapp-active      # Production traffic
    previewService: myapp-preview    # New version (blue)
    autoPromotionEnabled: false      # Manual promotion
    scaleDownDelaySeconds: 600       # Keep old version 10 min for rollback
    prePromotionAnalysis:
      templates:
      - templateName: smoke-tests
    postPromotionAnalysis:
      templates:
      - templateName: error-rate
```

---

## GitOps

### GitOps Principles

```
1. Declarative: entire system described declaratively in Git
2. Versioned: all desired state stored in Git (immutable, versioned)
3. Pulled automatically: approved changes are automatically applied
4. Continuously reconciled: software agents ensure actual state matches desired
```

### GitOps Repository Structure

```
gitops-repo/
├── apps/
│   ├── base/              # Shared base manifests
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── staging/       # Staging-specific patches
│       │   ├── replicas-patch.yaml
│       │   └── kustomization.yaml
│       └── production/    # Production-specific patches
│           ├── replicas-patch.yaml
│           ├── resources-patch.yaml
│           └── kustomization.yaml
├── infrastructure/
│   ├── cert-manager/
│   ├── nginx-ingress/
│   └── prometheus-stack/
└── clusters/
    ├── staging/
    │   └── flux-system/
    └── production/
        └── flux-system/
```

### Kustomize Overlay Example

```yaml
# apps/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        resources:
          requests:
            cpu: 100m
            memory: 128Mi

---
# apps/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base

images:
  - name: myapp
    newTag: "abc123"           # Updated by CI pipeline

patches:
  - path: replicas-patch.yaml

---
# apps/overlays/production/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: myapp
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: "1"
            memory: 1Gi
```

---

## ArgoCD

### ArgoCD Architecture

```
ArgoCD runs in Kubernetes cluster
├── API Server   — gRPC/REST API, web UI backend
├── Repository Server — clones repos, generates manifests
├── Application Controller — reconciliation loop
└── ApplicationSet Controller — generates Applications

ArgoCD watches Git repos → applies to Kubernetes
"Desired state in Git" = "Actual state in cluster"
```

### Application Definition

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production

  source:
    repoURL: https://github.com/org/gitops-repo
    targetRevision: main
    path: apps/overlays/production

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true          # Delete resources removed from Git
      selfHeal: true       # Revert manual changes
      allowEmpty: false    # Never sync empty state
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  revisionHistoryLimit: 10
```

### ApplicationSet (Multi-Cluster)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: staging
        url: https://staging-k8s.example.com
        env: staging
        replicas: "2"
      - cluster: production
        url: https://prod-k8s.example.com
        env: production
        replicas: "5"

  template:
    metadata:
      name: 'myapp-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/gitops-repo
        targetRevision: main
        path: 'apps/overlays/{{env}}'
      destination:
        server: '{{url}}'
        namespace: '{{env}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### ArgoCD CLI

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Login
argocd login argocd.example.com --sso

# App management
argocd app list
argocd app get myapp-production
argocd app sync myapp-production
argocd app diff myapp-production
argocd app history myapp-production
argocd app rollback myapp-production 3

# Manual sync with dry-run
argocd app sync myapp-production --dry-run

# Force sync (ignore resource differences)
argocd app sync myapp-production --force

# Refresh (re-fetch from Git without syncing)
argocd app get myapp-production --refresh
```

---

## FluxCD

### Flux Bootstrap

```bash
# Bootstrap Flux to GitHub
flux bootstrap github \
  --owner=my-org \
  --repository=gitops-fleet \
  --branch=main \
  --path=clusters/production \
  --personal

# Check status
flux check
flux get all
flux get kustomizations
flux get helmreleases
```

### Flux Components

```yaml
# GitRepository source
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: gitops-repo
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/org/gitops-repo
  ref:
    branch: main
  secretRef:
    name: github-pat

---
# Kustomization (applies manifests)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m
  path: ./apps/overlays/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops-repo
  targetNamespace: production
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: myapp
    namespace: production
  timeout: 5m

---
# HelmRelease
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: myapp
  namespace: production
spec:
  interval: 10m
  chart:
    spec:
      chart: myapp
      version: '>=1.0.0 <2.0.0'
      sourceRef:
        kind: HelmRepository
        name: mycompany-charts
  values:
    replicaCount: 5
    image:
      tag: abc123
  upgrade:
    remediation:
      retries: 3
  rollback:
    timeout: 1m
```

---

## GitHub Actions

### Complete Workflow Patterns

```yaml
# Matrix build strategy
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20, 21]
        os: [ubuntu-latest, windows-latest]
      fail-fast: false    # Don't cancel other matrix jobs on failure

    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
    - run: npm test

# Conditional jobs
jobs:
  needs-approval:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest

# Environment secrets
jobs:
  deploy:
    environment:
      name: production
      url: https://myapp.com
    steps:
    - name: Deploy
      env:
        API_TOKEN: ${{ secrets.PRODUCTION_API_TOKEN }}

# Caching
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: ${{ runner.os }}-npm-

# Artifacts
- uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: test-results/
    retention-days: 30

- uses: actions/download-artifact@v4
  with:
    name: test-results
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is the difference between Continuous Integration, Delivery, and Deployment?**

> - **CI**: Automatically build and test every code commit — catch bugs early
> - **CD (Delivery)**: Every CI-passing build is ready to deploy; deployment to production is a human decision
> - **CD (Deployment)**: Every CI-passing build is automatically deployed to production — no human gate; requires very robust automated testing
> Most teams do CI + Continuous Delivery; Continuous Deployment is rare and requires mature testing.

**Q2: What is a deployment pipeline and what should it include?**

> A deployment pipeline automates the path from code commit to production. Key stages:
> 1. Source trigger (webhook on push/PR)
> 2. Build (compile, package)
> 3. Unit + integration tests
> 4. Code quality (lint, coverage, static analysis)
> 5. Security scanning (SAST, dependency check, container scan)
> 6. Artifact publishing (Docker image, JAR, npm package)
> 7. Deploy to dev/staging
> 8. Automated acceptance/E2E tests
> 9. Approval gate (manual or automated)
> 10. Deploy to production
> 11. Smoke tests + monitoring

**Q3: What is GitOps?**

> GitOps uses Git as the **single source of truth** for infrastructure and application state. All desired state is declared in Git. An agent (ArgoCD, Flux) continuously reconciles the cluster state to match Git. Changes are made by PRs — providing audit trail, review, and rollback capability. Key benefit: you can restore the entire cluster from Git. "If it's not in Git, it doesn't exist."

---

### 🟡 Intermediate

**Q4: Explain the difference between Blue-Green and Canary deployments.**

> **Blue-Green**: two identical environments; all traffic switches from old (blue) to new (green) at once. Instant rollback by switching back. Requires 2x resources. **Canary**: gradually shift traffic from old to new (5% → 20% → 50% → 100%); monitor metrics at each step; rollback by returning traffic. Catches issues affecting a percentage of users before full rollout. Canary is more sophisticated and requires traffic splitting capability (Nginx, Istio, Argo Rollouts).

**Q5: How would you implement automatic rollback in a CI/CD pipeline?**

```yaml
# Example: Rollback if error rate spikes after deploy
- name: Deploy and monitor
  run: |
    helm upgrade myapp ./helm/myapp --set image.tag=$SHA --atomic --timeout 5m
    # --atomic: if any hook or release fails, rollback automatically

# For more sophisticated rollback with metrics:
# 1. Deploy canary with Argo Rollouts
# 2. Set up AnalysisTemplate with Prometheus query
# 3. If error rate > 1%, Argo auto-aborts and rolls back
```

**Q6: What is a feature flag and how does it relate to CI/CD?**

> A feature flag (toggle) lets you deploy code to production without activating it. The code path is gated by a flag value (in config/database/LaunchDarkly). Benefits for CI/CD: enables **trunk-based development** (incomplete features deployed but hidden), **gradual rollout** (enable for 5% of users), **instant rollback** (disable flag instead of redeploying), **A/B testing** (different behavior for segments). Tools: LaunchDarkly, Unleash, AWS AppConfig, Azure App Configuration.

---

### 🔴 Advanced

**Q7: How do you secure a CI/CD pipeline?**

> - **Secrets management**: Never hardcode secrets; use OIDC/Workload Identity instead of long-lived keys; rotate secrets regularly
> - **Least-privilege**: CI agent has only permissions needed for its job; separate roles per environment
> - **Signed artifacts**: Sign images with Cosign; verify signatures before deployment
> - **Supply chain security**: Pin action versions to full SHA; use Dependabot for dependency updates
> - **SBOM generation**: Track all dependencies for vulnerability management
> - **Branch protection**: Require PR reviews, signed commits, passing CI before merge
> - **Artifact scanning**: Trivy/Snyk in pipeline; block on CRITICAL vulnerabilities
> - **Audit logs**: Every pipeline run logged with who triggered it and what changed

**Q8: How would you design a CI/CD system for 500 microservices?**

> - **Mono-repo or poly-repo with path-based triggers**: only rebuild services that changed
> - **Shared CI templates/reusable workflows**: DRY pipeline definitions
> - **Separate CI and CD**: CI per service repo; CD via GitOps (ArgoCD ApplicationSet)
> - **Build caching**: Docker layer cache, npm/Maven dependency cache; per-service cache keys
> - **Test parallelism**: parallel jobs, distributed test execution
> - **Progressive delivery**: canary by default; teams control rollout via Argo Rollouts
> - **Platform team owns**: shared pipeline templates, registry, GitOps repo structure
> - **Self-service**: teams create new service → template generates pipeline automatically

---

## Scenario-Based Questions

### 🔵 Scenario 1: Production Deployment Caused Outage

*"Your latest deployment caused a 500 error spike. What do you do?"*

```bash
# 1. IMMEDIATE: Rollback (don't investigate yet — restore service first)
kubectl rollout undo deployment/myapp -n production
# or
helm rollback myapp -n production
# or (GitOps) revert the PR in the gitops repo and ArgoCD will auto-sync

# 2. Verify rollback succeeded
kubectl rollout status deployment/myapp -n production
curl -f https://myapp.example.com/health

# 3. Investigate (post-incident)
kubectl logs deployment/myapp -n production --previous
kubectl describe pod -l app=myapp -n production

# 4. Find the problematic change
git log --oneline -10
git diff v1.2.3..v1.2.4  # Diff between versions

# 5. Write incident report
# - What happened
# - Timeline
# - Root cause
# - What we'll do to prevent recurrence
```

### 🔵 Scenario 2: CI Pipeline is Too Slow

*"Your CI pipeline takes 45 minutes. Developers are complaining."*

```
Analyze the pipeline stages:
1. Checkout (fast ~10s)
2. npm install (4 min) → CACHE node_modules
3. Build (8 min) → PARALLELIZE by module
4. Unit tests (12 min) → SHARD across multiple runners
5. Integration tests (10 min) → Run in parallel with unit tests
6. Docker build (8 min) → Use BuildKit cache
7. Push to registry (3 min) → Already fast

Optimizations:
- Cache node_modules: -8 minutes
- Parallel unit test shards: -8 minutes
- Parallel integration with unit: -10 minutes
- Docker cache hits: -5 minutes
Target: ~14 minutes total
```

---

## Hands-On Labs

### Lab 1: GitHub Actions with Docker

```yaml
# .github/workflows/docker-ci.yml
name: Docker CI

on:
  push:
    branches: [main]

jobs:
  build-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
    - uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Login to GHCR
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Build and Push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

---

## Cheat Sheet

```bash
# ─── ARGO CD ──────────────────────────────────────────
argocd app list
argocd app sync myapp
argocd app diff myapp
argocd app rollback myapp [revision]
argocd app history myapp

# ─── FLUX ─────────────────────────────────────────────
flux get all
flux reconcile kustomization apps
flux reconcile source git gitops-repo
flux logs --follow
flux suspend kustomization apps    # Pause sync
flux resume kustomization apps     # Resume sync

# ─── HELM ─────────────────────────────────────────────
helm list -A
helm upgrade --install app ./chart --atomic --timeout 5m
helm rollback app 2
helm history app
helm diff upgrade app ./chart -f values.yaml

# ─── KUBECTL ROLLOUT ──────────────────────────────────
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=3
kubectl set image deployment/myapp myapp=myapp:v2

# ─── KUSTOMIZE ────────────────────────────────────────
kubectl apply -k apps/overlays/production
kubectl diff -k apps/overlays/production
kustomize build apps/overlays/production | kubectl apply -f -
```

---

> **Cross-links:** [← GCP](./GCP.md) | [Monitoring →](./Monitoring.md) | [Jenkins (CI engine) →](./Jenkins.md) | [Kubernetes (deployment target) →](./Kubernetes.md) | [Security (pipeline security) →](./Security.md)