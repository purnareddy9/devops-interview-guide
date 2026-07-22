# 🔴 OpenShift — DevOps Interview Guide

> [← Kubernetes](./Kubernetes.md) | [Main Index](./README.md) | [Jenkins →](./Jenkins.md)

---

## Table of Contents

1. [What is OpenShift?](#what-is-openshift)
2. [Architecture](#architecture)
3. [OpenShift vs Kubernetes](#openshift-vs-kubernetes)
4. [Projects & Namespaces](#projects--namespaces)
5. [Builds & ImageStreams](#builds--imagestreams)
6. [DeploymentConfig vs Deployment](#deploymentconfig-vs-deployment)
7. [Routes & Networking](#routes--networking)
8. [Security Context Constraints (SCC)](#security-context-constraints-scc)
9. [RBAC & Users](#rbac--users)
10. [Operators & OperatorHub](#operators--operatorhub)
11. [OpenShift Pipelines (Tekton)](#openshift-pipelines-tekton)
12. [OpenShift GitOps (ArgoCD)](#openshift-gitops-argocd)
13. [Storage](#storage)
14. [Monitoring & Logging](#monitoring--logging)
15. [Cluster Administration](#cluster-administration)
16. [Interview Questions](#interview-questions)
17. [Scenario-Based Questions](#scenario-based-questions)
18. [Hands-On Labs](#hands-on-labs)
19. [oc CLI Cheat Sheet](#oc-cli-cheat-sheet)

---

## What is OpenShift?

**Red Hat OpenShift** is an enterprise Kubernetes platform that adds security hardening, developer tooling, CI/CD pipelines, and an opinionated operational layer on top of upstream Kubernetes.

```
OpenShift = Kubernetes + Security + Developer Tools + Operations
```

### Editions

| Edition | Description |
|---------|-------------|
| **OCP** (OpenShift Container Platform) | On-premises, self-managed |
| **ROSA** (Red Hat OpenShift Service on AWS) | Fully managed on AWS |
| **ARO** (Azure Red Hat OpenShift) | Fully managed on Azure |
| **OSD** (OpenShift Dedicated) | Red Hat-managed on AWS/GCP |
| **OKD** | Community-supported, upstream of OCP |
| **Microshift** | Edge/single-node variant |

---

## Architecture

```
┌──────────────────────────── OpenShift Cluster ──────────────────────────────┐
│                                                                               │
│  ┌────────────────────── Control Plane (Masters) ───────────────────────┐   │
│  │  ┌──────────────┐  ┌─────────────────┐  ┌──────────────────────┐    │   │
│  │  │  API Server  │  │  etcd (3 nodes) │  │ Controller Manager   │    │   │
│  │  │  + OAuth     │  │  (quorum-based) │  │ + Scheduler          │    │   │
│  │  └──────────────┘  └─────────────────┘  └──────────────────────┘    │   │
│  │  ┌──────────────────────────────────────────────────────────────┐    │   │
│  │  │           OpenShift Controllers (on top of k8s controllers)   │    │   │
│  │  │   BuildConfig · ImageStream · Route · DeploymentConfig       │    │   │
│  │  └──────────────────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌── Infra Node ──────┐   ┌── Worker Node ─────┐  ┌── Worker Node ─────┐   │
│  │  Router (HAProxy)  │   │  kubelet            │  │  kubelet           │   │
│  │  Image Registry    │   │  CRI-O runtime      │  │  CRI-O runtime     │   │
│  │  Monitoring Stack  │   │  Pods               │  │  Pods              │   │
│  └────────────────────┘   └─────────────────────┘  └────────────────────┘   │
│                                                                               │
│  ┌───────────────────── OpenShift Web Console ──────────────────────────┐   │
│  │  Developer View · Administrator View · Topology · Pipelines          │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Role |
|-----------|------|
| **CRI-O** | Default container runtime (not Docker) |
| **HAProxy Router** | Ingress traffic routing via Routes |
| **Internal Registry** | Built-in image registry (`image-registry.openshift-image-registry.svc`) |
| **OAuth Server** | Integrated identity provider / SSO |
| **MCO** (Machine Config Operator) | Manages node OS configuration via MachineConfig |
| **OLM** (Operator Lifecycle Manager) | Manages Operator installation and upgrades |
| **MachineConfigPool** | Groups nodes sharing the same OS config |

---

## OpenShift vs Kubernetes

| Feature | Kubernetes | OpenShift |
|---------|-----------|-----------|
| **Default runtime** | containerd | CRI-O |
| **Ingress** | Ingress object | Route (HAProxy) |
| **Namespaces** | Namespace | Project (namespace + metadata + RBAC) |
| **Security** | No default PSP/PSA enforcement | SCC enforced by default |
| **Image building** | External (Docker/Kaniko) | Native BuildConfig (S2I, Docker, custom) |
| **Pipelines** | External (Jenkins, etc.) | OpenShift Pipelines (Tekton) built-in |
| **GitOps** | External ArgoCD | OpenShift GitOps (ArgoCD) as operator |
| **Web console** | Limited (kuboard, etc.) | Full developer + admin console |
| **User management** | External IdP needed | Built-in OAuth, HTPasswd, LDAP, OIDC |
| **Operators** | Manual Helm/kustomize | OperatorHub + OLM |
| **Root containers** | Allowed by default | Blocked by default (SCC restricted-v2) |
| **Upgrades** | Manual | CVO (Cluster Version Operator) managed |

---

## Projects & Namespaces

A **Project** is an OpenShift extension of a Kubernetes Namespace with added annotations, display name, description, and default RBAC bindings.

```bash
# Create a project
oc new-project my-app \
  --display-name="My Application" \
  --description="Production app workloads"

# Equivalent via YAML
oc apply -f - <<EOF
apiVersion: project.openshift.io/v1
kind: Project
metadata:
  name: my-app
  annotations:
    openshift.io/display-name: "My Application"
    openshift.io/description: "Production app workloads"
EOF

# Switch projects
oc project my-app
oc projects              # List all accessible projects

# Project quotas and limits
oc apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: my-app
spec:
  hard:
    pods: "20"
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    persistentvolumeclaims: "10"
    requests.storage: "100Gi"
EOF
```

### Default Project Templates

OpenShift applies a **project template** to every new project. Admins can customise this:

```bash
# Export default template
oc adm create-bootstrap-project-template -o yaml > project-template.yaml

# Edit and apply as custom template
oc apply -f project-template.yaml -n openshift-config

# Configure cluster to use custom template
oc edit project.config.openshift.io/cluster
# Set: spec.projectRequestTemplate.name: <template-name>
```

---

## Builds & ImageStreams

### Source-to-Image (S2I)

S2I is OpenShift's signature feature: it automatically builds a runnable container image from source code without writing a Dockerfile.

```
Developer pushes code  →  S2I builder detects language
→  Injects source into builder image  →  Runs assemble script
→  Creates application image  →  Pushes to internal registry
→  Triggers deployment
```

```bash
# New app from source (OpenShift auto-detects Node.js)
oc new-app nodejs~https://github.com/myorg/my-nodeapp.git \
  --name=my-nodeapp \
  --env=NODE_ENV=production

# New app from Dockerfile in repo
oc new-app https://github.com/myorg/myapp.git \
  --strategy=docker \
  --name=myapp

# New app from existing image
oc new-app nginx:1.25 --name=my-nginx
```

### BuildConfig

```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: my-nodeapp
  namespace: my-app
spec:
  source:
    type: Git
    git:
      uri: https://github.com/myorg/my-nodeapp.git
      ref: main
    contextDir: /app
  strategy:
    type: Source          # S2I strategy
    sourceStrategy:
      from:
        kind: ImageStreamTag
        name: nodejs:18-ubi8
        namespace: openshift    # Platform-provided builder images
      env:
      - name: NPM_MIRROR
        value: https://registry.npmjs.org
  output:
    to:
      kind: ImageStreamTag
      name: my-nodeapp:latest
  triggers:
  - type: GitHub
    github:
      secret: my-webhook-secret
  - type: ImageChange     # Rebuild when builder image updates
  - type: ConfigChange    # Rebuild when BuildConfig changes
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 2
      memory: 1Gi
```

```bash
# Start a build manually
oc start-build my-nodeapp

# Follow build logs
oc logs -f bc/my-nodeapp

# Start build from local source
oc start-build my-nodeapp --from-dir=./src

# Cancel a running build
oc cancel-build my-nodeapp-3
```

### ImageStream

An **ImageStream** is an OpenShift abstraction that tracks image versions by tag. It decouples deployments from external registry changes.

```yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: my-nodeapp
  namespace: my-app
spec:
  lookupPolicy:
    local: true    # Pods in this namespace can reference by short name
```

```bash
# View image stream tags
oc get is my-nodeapp
oc describe is my-nodeapp

# Import an external image into an ImageStream
oc import-image my-nginx:1.25 \
  --from=docker.io/nginx:1.25 \
  --confirm

# Set up periodic import
oc import-image my-nginx:1.25 \
  --from=docker.io/nginx:1.25 \
  --confirm \
  --scheduled=true

# Tag an image
oc tag my-nodeapp:latest my-nodeapp:v1.2.3
```

---

## DeploymentConfig vs Deployment

OpenShift originally used `DeploymentConfig` (DC). As of OpenShift 4.x, the Kubernetes-native `Deployment` is preferred. DCs are still supported but deprecated in favor of Deployments.

| Feature | Deployment (K8s) | DeploymentConfig (OCP) |
|---------|-----------------|------------------------|
| **Standard** | Kubernetes-native | OpenShift-specific CRD |
| **Rollout controller** | Controller Manager | OpenShift deployer pod |
| **Triggers** | Not built-in | ImageChange, ConfigChange |
| **Lifecycle hooks** | Not built-in | pre, mid, post hooks |
| **Strategy** | RollingUpdate, Recreate | Rolling, Recreate, Custom |
| **Status** | Preferred (4.x+) | Deprecated |

```yaml
# Modern approach: Use Deployment (preferred in OCP 4.x)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nodeapp
  namespace: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-nodeapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-nodeapp
    spec:
      containers:
      - name: app
        image: image-registry.openshift-image-registry.svc:5000/my-app/my-nodeapp:latest
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        securityContext:
          allowPrivilegeEscalation: false
          runAsNonRoot: true
          capabilities:
            drop: ["ALL"]
          seccompProfile:
            type: RuntimeDefault
```

```bash
# Deployment commands
oc rollout status deployment/my-nodeapp
oc rollout undo deployment/my-nodeapp
oc rollout history deployment/my-nodeapp
oc set image deployment/my-nodeapp app=my-nodeapp:v2

# DeploymentConfig (legacy) commands
oc rollout latest dc/my-nodeapp
oc rollout history dc/my-nodeapp
oc rollback dc/my-nodeapp
```

---

## Routes & Networking

### Route

A **Route** exposes a Service externally via the OpenShift HAProxy router. Equivalent to Kubernetes Ingress but with more features.

```yaml
# Edge TLS termination route
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-nodeapp
  namespace: my-app
spec:
  host: my-nodeapp.apps.mycluster.example.com   # Optional: auto-generated if omitted
  to:
    kind: Service
    name: my-nodeapp-svc
    weight: 100
  port:
    targetPort: 8080-tcp
  tls:
    termination: edge          # edge | passthrough | reencrypt
    insecureEdgeTerminationPolicy: Redirect
  wildcardPolicy: None
```

```yaml
# Passthrough route (TLS all the way to pod)
tls:
  termination: passthrough

# Re-encrypt route (new TLS between router and pod)
tls:
  termination: reencrypt
  destinationCACertificate: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
```

```bash
# Expose a service as a route
oc expose svc/my-nodeapp-svc

# Expose with custom hostname
oc expose svc/my-nodeapp-svc --hostname=myapp.example.com

# Create edge TLS route
oc create route edge my-nodeapp \
  --service=my-nodeapp-svc \
  --cert=tls.crt \
  --key=tls.key \
  --ca-cert=ca.crt

# Get route URL
oc get route my-nodeapp -o jsonpath='{.spec.host}'
```

### Network Architecture

```
Internet → External LB → HAProxy Router (Infra node)
         → Route (host-based matching)
         → Service (ClusterIP)
         → Pod
```

### OpenShift SDN / OVN-Kubernetes

| Network Plugin | Description |
|---------------|-------------|
| **OVN-Kubernetes** | Default in OCP 4.12+; eBPF-based, superior performance |
| **OpenShift SDN** | Legacy plugin (deprecated); VXLAN overlay |

```bash
# Check cluster network type
oc get network.config.openshift.io/cluster -o jsonpath='{.spec.networkType}'

# Network policies (same as K8s)
oc get networkpolicies -n my-app
```

---

## Security Context Constraints (SCC)

SCC is OpenShift's primary pod security mechanism (predates and inspired Kubernetes PodSecurity). Every pod must match an allowed SCC.

### Built-in SCCs (most to least permissive)

| SCC | Runnable As | Use Case |
|-----|-------------|---------|
| `privileged` | Any UID, root | System/node-level operators |
| `anyuid` | Any UID | Legacy apps needing specific UIDs |
| `nonroot` | Any non-root UID | Apps that set a non-root UID |
| `nonroot-v2` | Any non-root UID | Stricter nonroot (no privilege escalation) |
| `restricted` | Random UID from namespace range | Default for service accounts |
| `restricted-v2` | Random UID, no privilege escalation | Default in OCP 4.11+ |

```bash
# View all SCCs
oc get scc

# Describe an SCC
oc describe scc restricted-v2

# Check which SCC a running pod uses
oc get pod my-pod -o jsonpath='{.metadata.annotations.openshift\.io/scc}'

# Grant a service account a specific SCC
oc adm policy add-scc-to-user anyuid -z my-service-account -n my-app
oc adm policy add-scc-to-group anyuid system:serviceaccounts:my-app

# Remove SCC from service account
oc adm policy remove-scc-from-user anyuid -z my-service-account -n my-app

# Check what SCC a pod would use (dry-run)
oc adm policy scc-subject-review -f pod.yaml
oc adm policy scc-review -f deployment.yaml
```

### Custom SCC

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: my-custom-scc
allowPrivilegeEscalation: false
allowPrivilegedContainer: false
allowedCapabilities: []
defaultAddCapabilities: []
requiredDropCapabilities:
- ALL
fsGroup:
  type: MustRunAs
  ranges:
  - min: 1000
    max: 65535
runAsUser:
  type: MustRunAsRange
  uidRangeMin: 1000
  uidRangeMax: 65535
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
volumes:
- configMap
- secret
- emptyDir
- persistentVolumeClaim
users: []
groups: []
```

---

## RBAC & Users

### Identity Providers

OpenShift supports configuring external identity providers:

```yaml
# HTPasswd (development/testing)
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: htpasswd_provider
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpasswd-secret   # Secret in openshift-config namespace

  # LDAP example
  - name: ldap_provider
    type: LDAP
    ldap:
      url: ldap://ldap.example.com/ou=users,dc=example,dc=com?uid
      insecure: false

  # OIDC / OAuth2 example (GitHub, GitLab, Okta, etc.)
  - name: github_provider
    type: GitHub
    github:
      clientID: "my-client-id"
      clientSecret:
        name: github-client-secret
      organizations:
      - myorg
```

```bash
# Create HTPasswd secret
htpasswd -c -B -b users.htpasswd admin secretpass
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config

# Add new user to htpasswd secret
htpasswd -b users.htpasswd newuser pass123
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config --dry-run=client -o yaml | oc apply -f -
```

### RBAC

OpenShift extends Kubernetes RBAC with **cluster-level** and **local** roles:

```bash
# Predefined cluster roles
# cluster-admin  → full cluster access
# admin          → full project access
# edit           → create/modify resources, no RBAC changes
# view           → read-only

# Add admin to project
oc adm policy add-role-to-user admin alice -n my-app

# Add view role
oc adm policy add-role-to-user view bob -n my-app

# Make cluster-admin
oc adm policy add-cluster-role-to-user cluster-admin alice

# Remove role
oc adm policy remove-role-from-user admin alice -n my-app

# List role bindings in project
oc get rolebindings -n my-app

# Impersonate a user (for testing)
oc get pods --as=alice -n my-app
oc auth can-i get pods --as=alice -n my-app
```

---

## Operators & OperatorHub

### Operator Lifecycle Manager (OLM)

OLM manages Operator installation, upgrades, and dependency resolution.

```
OperatorHub (catalog)
  └── CatalogSource (index of operators)
       └── Subscription (which operator + channel)
            └── InstallPlan (resolved set of CSVs to install)
                 └── ClusterServiceVersion (CSV) — the Operator itself
                      └── CRDs + Deployments + RBAC
```

```bash
# List available operators
oc get packagemanifests -n openshift-marketplace

# Install an operator (example: cert-manager)
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cert-manager
  namespace: openshift-operators
spec:
  channel: stable
  name: cert-manager
  source: community-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic    # or Manual
EOF

# View installed operators
oc get csv -n openshift-operators

# Check operator status
oc describe csv cert-manager.v1.13.0 -n openshift-operators

# Manual install plan approval
oc get installplan -n openshift-operators
oc patch installplan install-xyz \
  --type merge \
  --patch '{"spec":{"approved":true}}' \
  -n openshift-operators
```

---

## OpenShift Pipelines (Tekton)

OpenShift Pipelines is the Red Hat distribution of **Tekton** — a Kubernetes-native CI/CD system.

### Core Objects

| Object | Description |
|--------|-------------|
| **Task** | Sequence of Steps (containers) performing a unit of work |
| **TaskRun** | Instantiation of a Task |
| **Pipeline** | DAG of Tasks |
| **PipelineRun** | Instantiation of a Pipeline |
| **Workspace** | Shared storage between Tasks |
| **TriggerTemplate** | Creates PipelineRuns from events |
| **EventListener** | HTTP endpoint receiving webhook events |

```yaml
# Simple Pipeline: build and deploy
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-and-deploy
  namespace: my-app
spec:
  params:
  - name: IMAGE
    type: string
  - name: GIT_URL
    type: string
  workspaces:
  - name: shared-workspace
  tasks:
  - name: fetch-source
    taskRef:
      name: git-clone
      kind: ClusterTask
    workspaces:
    - name: output
      workspace: shared-workspace
    params:
    - name: url
      value: $(params.GIT_URL)

  - name: build-image
    taskRef:
      name: buildah
      kind: ClusterTask
    runAfter: [fetch-source]
    workspaces:
    - name: source
      workspace: shared-workspace
    params:
    - name: IMAGE
      value: $(params.IMAGE)

  - name: deploy
    taskRef:
      name: openshift-client
      kind: ClusterTask
    runAfter: [build-image]
    params:
    - name: SCRIPT
      value: |
        oc rollout restart deployment/my-nodeapp -n my-app
        oc rollout status deployment/my-nodeapp -n my-app
```

```bash
# Install OpenShift Pipelines operator (via subscription)
# Then use tkn CLI or oc

# List pipelines
oc get pipelines -n my-app
tkn pipeline list -n my-app

# Run a pipeline
tkn pipeline start build-and-deploy \
  -p IMAGE=image-registry.openshift-image-registry.svc:5000/my-app/my-nodeapp:latest \
  -p GIT_URL=https://github.com/myorg/my-nodeapp.git \
  -w name=shared-workspace,claimName=pipeline-pvc \
  -n my-app

# Watch a pipeline run
tkn pipelinerun logs -f -L -n my-app

# List task runs
tkn taskrun list -n my-app
```

---

## OpenShift GitOps (ArgoCD)

OpenShift GitOps is the Red Hat distribution of **Argo CD**, installed via an Operator.

```bash
# Install OpenShift GitOps operator
# Creates argocd instance in openshift-gitops namespace automatically

# Access ArgoCD UI
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'

# Get admin password
oc get secret openshift-gitops-cluster \
  -n openshift-gitops \
  -o jsonpath='{.data.admin\.password}' | base64 -d

# Create an ArgoCD Application
oc apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/gitops-repo.git
    targetRevision: main
    path: environments/production
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF

# Sync manually
argocd app sync my-app

# Check sync status
oc get application my-app -n openshift-gitops
```

---

## Storage

### Storage Classes

```bash
# View available storage classes
oc get storageclass

# Default storage classes in common environments
# AWS ROSA:     gp3-csi (default), gp2-csi
# Azure ARO:    managed-csi (default), azure-file-csi
# On-premises:  depends on OCS/ODF or NFS provisioner
```

### OpenShift Data Foundation (ODF)

ODF (formerly OpenShift Container Storage / OCS) provides Ceph-based persistent storage:

```bash
# After ODF operator installation
oc get storagecluster -n openshift-storage
oc get cephcluster -n openshift-storage

# Storage classes provided by ODF
# ocs-storagecluster-ceph-rbd   → Block (RWO)
# ocs-storagecluster-cephfs     → File (RWX)
# ocs-storagecluster-ceph-rgw   → Object (S3-compatible)
```

```yaml
# PVC using ODF CephFS (ReadWriteMany for shared access)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
  namespace: my-app
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: ocs-storagecluster-cephfs
  resources:
    requests:
      storage: 10Gi
```

---

## Monitoring & Logging

### Built-in Monitoring (Prometheus + Grafana + Alertmanager)

```bash
# Access monitoring stack
oc get routes -n openshift-monitoring
# prometheus-k8s
# alertmanager-main
# grafana (if enabled)

# Enable user workload monitoring
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF

# Create ServiceMonitor for your app
oc apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-nodeapp
  namespace: my-app
spec:
  endpoints:
  - interval: 30s
    path: /metrics
    port: 8080-tcp
  selector:
    matchLabels:
      app: my-nodeapp
EOF
```

### Logging (OpenShift Logging / Loki / EFK)

```bash
# OpenShift Logging operator provides:
# - log collector (Vector/Fluentd) on every node
# - log store (Elasticsearch or Loki)
# - Kibana or Grafana frontend

# Check logging stack
oc get pods -n openshift-logging

# Forward logs to external Elasticsearch
oc apply -f - <<EOF
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  outputs:
  - name: elasticsearch-external
    type: elasticsearch
    url: https://elasticsearch.example.com:9200
    secret:
      name: elasticsearch-creds
  pipelines:
  - name: app-logs
    inputRefs: [application]
    outputRefs: [elasticsearch-external]
EOF
```

---

## Cluster Administration

### Cluster Version Operator (CVO)

```bash
# Check cluster version
oc get clusterversion

# View available updates
oc adm upgrade

# Trigger upgrade
oc adm upgrade --to-latest=true
oc adm upgrade --to=4.14.5

# Watch upgrade progress
oc get clusterversion -w
oc get clusteroperators    # All operators must be Available=True
```

### Machine Config Operator (MCO)

```bash
# List machine configs
oc get mc

# List machine config pools
oc get mcp

# Custom kernel argument via MachineConfig
oc apply -f - <<EOF
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-custom-kernel-args
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  kernelArguments:
  - "hugepages=1Gi"
  - "hugepages-1Gi=4"
EOF

# Pause a MachineConfigPool (prevent auto-reboot during changes)
oc patch mcp worker --type merge -p '{"spec":{"paused":true}}'
# ... make multiple changes ...
oc patch mcp worker --type merge -p '{"spec":{"paused":false}}'
```

### Node Management

```bash
# List nodes
oc get nodes
oc get nodes -o wide

# Cordon and drain
oc adm cordon <node>
oc adm drain <node> --ignore-daemonsets --delete-emptydir-data

# Uncordon
oc adm uncordon <node>

# Node debugging (creates privileged pod)
oc debug node/<node>

# Label / taint nodes
oc label node worker-1 node-role.kubernetes.io/gpu=
oc adm taint node worker-1 nvidia.com/gpu=:NoSchedule
```

### Certificate Management

```bash
# Approve CSRs (required after node additions)
oc get csr
oc adm certificate approve <csr-name>
oc get csr -o go-template='{{range .items}}{{if not .status.certificate}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve

# Check cert expiry
oc -n openshift-kube-apiserver-operator \
   get secret kube-apiserver-to-kubelet-signer \
   -o jsonpath='{.metadata.annotations.auth\.openshift\.io/certificate-not-after}'
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is the difference between OpenShift and Kubernetes?**

> OpenShift is an enterprise Kubernetes distribution by Red Hat. It adds: built-in Security Context Constraints (SCC) that block root containers by default, S2I builds so developers can build images from source without Dockerfiles, Routes (HAProxy-based ingress), an integrated web console with developer and admin views, built-in OAuth/LDAP identity management, OLM for operator lifecycle management, and a managed upgrade path via the Cluster Version Operator. Kubernetes is the underlying engine; OpenShift wraps it with opinionated security, tooling, and operations.

**Q2: What is a Route in OpenShift?**

> A Route exposes a Service externally through the OpenShift HAProxy Router. It supports TLS termination modes: `edge` (TLS terminated at router, plain HTTP to pod), `passthrough` (TLS all the way to pod, router does not decrypt), and `reencrypt` (TLS to router, new TLS to pod). Routes are auto-assigned a hostname in the format `<name>-<namespace>.apps.<cluster-domain>` if you don't specify one. It is the OpenShift equivalent of Kubernetes Ingress, with richer TLS options and tighter integration.

**Q3: What is SCC and why does OpenShift use it?**

> SCC (Security Context Constraints) is OpenShift's pod security model. Every pod must match at least one SCC that the pod's service account is granted. The default `restricted-v2` SCC prevents running as root, prevents privilege escalation, drops all Linux capabilities, and enforces a random UID from the namespace's assigned range. This "deny-by-default" security posture is what makes OpenShift suitable for enterprise environments. It predates and inspired Kubernetes PodSecurity admission.

---

### 🟡 Intermediate

**Q4: Explain the S2I (Source-to-Image) process.**

> S2I is a framework for building container images from source code:
> 1. Developer provides source code URL (Git) and a builder image (e.g., `nodejs:18-ubi8`).
> 2. OpenShift creates a build pod using the builder image.
> 3. The `assemble` script inside the builder clones the source, installs dependencies, compiles if needed.
> 4. The resulting filesystem is committed as a new image layer.
> 5. The new image is pushed to the internal registry and tagged with the ImageStream.
> 6. An ImageChange trigger fires the Deployment to roll out the new image.
>
> Benefits: no Dockerfile needed, reproducible builds, easy language-version upgrades (update builder image → auto-rebuild all apps).

**Q5: How does OpenShift handle image management differently from vanilla Kubernetes?**

> OpenShift adds two abstractions: **ImageStreams** and **ImageStreamTags**. Instead of referencing a full registry URL in a Deployment, you reference an ImageStream tag. When the image tag is updated (by a build or `oc tag`), the `ImageChange` trigger on a DeploymentConfig automatically triggers a rollout. This decouples deployments from the registry — you don't need to update YAML manifests to deploy a new image version. Additionally, OpenShift's internal registry (`image-registry.openshift-image-registry.svc:5000`) integrates with RBAC so only authorized service accounts can push/pull images.

**Q6: What is OLM and how does it differ from Helm?**

> OLM (Operator Lifecycle Manager) manages operators as first-class citizens: it resolves dependencies between operators, handles upgrades channel by channel (stable, fast, candidate), manages CRD lifecycle, and enforces RBAC for operators. A `Subscription` declares which operator and channel you want; OLM creates an `InstallPlan` and installs the `ClusterServiceVersion` (CSV) which packages the operator's deployment, CRDs, and RBAC.
>
> Helm is a templating-based package manager for Kubernetes manifests — it doesn't have the concept of channels, automatic reconciliation, or dependency management. OLM provides a more robust lifecycle for complex, stateful operators that need to manage their own CRDs and upgrades.

---

### 🔴 Advanced

**Q7: A pod is failing due to "SCC denied" — how do you diagnose and fix it?**

> **Diagnosis:**
> ```bash
> oc describe pod failing-pod    # Look for "Error creating: pods ... is forbidden"
> oc get events -n my-app --sort-by='.lastTimestamp'
> # Common message: "unable to validate against any security context constraint"
>
> # Check what the pod requests vs what SCCs allow
> oc adm policy scc-subject-review -f pod.yaml
> ```
>
> **Resolution options (least to most privileged):**
> 1. Fix the application: set `runAsNonRoot: true`, drop ALL capabilities, set `allowPrivilegeEscalation: false` in the pod spec — this makes it match `restricted-v2` without granting anything extra.
> 2. If the app needs a specific UID: grant `anyuid` to the service account: `oc adm policy add-scc-to-user anyuid -z <sa> -n <ns>`
> 3. Create a custom SCC that grants only what's needed — never use `privileged` unless absolutely necessary.
> 4. For system workloads: `oc adm policy add-scc-to-user privileged -z <sa> -n <ns>`

**Q8: How does the Cluster Version Operator (CVO) manage OpenShift upgrades?**

> The CVO watches the `ClusterVersion` object and a release image. It:
> 1. Downloads the new release image from `quay.io/openshift-release-dev`.
> 2. Validates the upgrade path is supported (upgrade graph).
> 3. Applies manifests for cluster operators (API server, etcd, authentication, console, etc.) in the correct order.
> 4. Each **Cluster Operator** (CO) manages its own component; CVO waits for each CO to report `Available=True, Progressing=False, Degraded=False` before moving to the next.
> 5. Machine Config Operator updates the OS (RHCOS) on each node pool sequentially — nodes reboot with new RHCOS version.
>
> The upgrade is rolling and graceful, but you should drain and check `oc get co` and `oc get nodes` throughout. You cannot skip minor versions (4.12 → 4.14 requires passing through 4.13 unless the upgrade graph explicitly allows it).

**Q9: How would you set up multi-tenancy in OpenShift?**

> Multi-tenancy in OpenShift requires several layers:
> 1. **Projects with ResourceQuotas and LimitRanges** — cap CPU/memory/storage per team.
> 2. **NetworkPolicies** — default-deny all, allow only required cross-project traffic.
> 3. **SCCs** — ensure tenants can't escalate privileges.
> 4. **RBAC** — `admin` role scoped to project; no `cluster-admin` for tenants.
> 5. **Dedicated nodes** (optional): use `taints/tolerations` and `nodeSelector` to isolate critical tenants.
> 6. **HierarchicalNamespaceController** or **OpenShift Project Hierarchies** for organizational grouping.
> 7. **Egress firewall / EgressNetworkPolicy** — control what external IPs projects can reach.

---

## Scenario-Based Questions

### 🔵 Scenario 1: Deploy a Node.js App End-to-End on OpenShift

*"Walk me through deploying a Node.js app from Git to production on OpenShift."*

```bash
# 1. Create project
oc new-project my-nodejs-app --display-name="Node.js Production App"

# 2. Deploy from source (S2I auto-detects Node.js)
oc new-app nodejs:18-ubi8~https://github.com/myorg/my-nodeapp.git \
  --name=my-nodeapp \
  --env=NODE_ENV=production \
  --env=PORT=8080

# 3. Watch build
oc logs -f bc/my-nodeapp

# 4. Once build succeeds, expose as a route
oc expose svc/my-nodeapp

# 5. Get the URL
oc get route my-nodeapp -o jsonpath='{.spec.host}'

# 6. Set up resource limits
oc set resources deployment/my-nodeapp \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=512Mi

# 7. Auto-scale
oc autoscale deployment/my-nodeapp --min=2 --max=10 --cpu-percent=70

# 8. Add edge TLS route
oc create route edge my-nodeapp-tls \
  --service=my-nodeapp \
  --hostname=my-nodeapp.example.com
```

### 🔵 Scenario 2: Application Running as Root — Fix It

*"Your application image runs as root (UID 0). It fails on OpenShift. How do you fix it without changing the Dockerfile?"*

```bash
# Option A: Create a dedicated service account and grant anyuid SCC
oc create serviceaccount my-nodeapp-sa -n my-nodeapp-app

oc adm policy add-scc-to-user anyuid \
  -z my-nodeapp-sa \
  -n my-nodeapp-app

# Patch deployment to use that SA
oc set serviceaccount deployment/my-nodeapp my-nodeapp-sa

# Option B (better long-term): add USER directive to Dockerfile
# FROM node:18
# ...
# RUN chown -R node:node /app
# USER node          ← run as non-root node user

# Option C: Override UID in deployment (if app doesn't need root)
# spec.securityContext.runAsUser: 1001
```

### 🔵 Scenario 3: Debug a Failing Build

```bash
# 1. Check build status
oc get builds -n my-app

# 2. View build logs
oc logs build/my-nodeapp-3

# 3. Common failure: npm install fails
#    → Check network policies blocking npm registry
#    → Check if proxy env vars are needed
oc set env bc/my-nodeapp \
  HTTP_PROXY=http://proxy.example.com:3128 \
  HTTPS_PROXY=http://proxy.example.com:3128 \
  NO_PROXY=.cluster.local,.svc,localhost

# 4. Re-run build
oc start-build my-nodeapp

# 5. Check ImageStream after successful build
oc describe is my-nodeapp
```

### 🔵 Scenario 4: Cluster Upgrade Pre-flight Checks

*"How do you prepare for an OpenShift cluster upgrade?"*

```bash
# 1. Check current version and available upgrades
oc get clusterversion
oc adm upgrade

# 2. All operators must be healthy
oc get co | grep -v "True.*False.*False"   # should return nothing if healthy

# 3. All nodes must be Ready
oc get nodes | grep -v Ready

# 4. Check etcd health
oc get etcd -o wide
oc rsh -n openshift-etcd etcd-master-0 \
  etcdctl member list --cacert /etc/kubernetes/static-pod-certs/configmaps/etcd-serving-ca/ca-bundle.crt \
  --cert /etc/kubernetes/static-pod-certs/secrets/etcd-all-certs/etcd-peer-master-0.crt \
  --key /etc/kubernetes/static-pod-certs/secrets/etcd-all-certs/etcd-peer-master-0.key

# 5. Check PodDisruptionBudgets that could block node drain
oc get pdb --all-namespaces

# 6. Check alerts in monitoring
oc get prometheusrule --all-namespaces | grep -i critical

# 7. Initiate upgrade
oc adm upgrade --to-latest=true
```

---

## Hands-On Labs

### Lab 1: Full App Deployment with Pipelines

```bash
# Prerequisites: OpenShift cluster, oc CLI, tkn CLI

# 1. Create namespaces
oc new-project ci-demo

# 2. Install Pipelines operator (via OperatorHub or CLI)
oc apply -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator
  namespace: openshift-operators
spec:
  channel: latest
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# 3. Wait for operator
oc wait --for=condition=Ready pod -l name=openshift-pipelines-operator \
  -n openshift-operators --timeout=120s

# 4. Create a workspace PVC
oc apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pipeline-workspace
  namespace: ci-demo
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
EOF

# 5. Create and run a simple pipeline
tkn pipeline start build-and-deploy \
  -p IMAGE=image-registry.openshift-image-registry.svc:5000/ci-demo/my-app:latest \
  -p GIT_URL=https://github.com/sclorg/nodejs-ex.git \
  -w name=shared-workspace,claimName=pipeline-workspace \
  -n ci-demo --use-param-defaults

# 6. Monitor
tkn pipelinerun logs -f -L -n ci-demo
```

### Lab 2: RBAC and Multi-Tenancy

```bash
# 1. Create two tenant projects
oc new-project team-alpha
oc new-project team-beta

# 2. Create user accounts (htpasswd)
htpasswd -c -B -b users.htpasswd alice alicepass
htpasswd -b users.htpasswd bob bobpass

oc create secret generic htpasswd-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config

# 3. Grant alice admin on team-alpha, view on team-beta
oc adm policy add-role-to-user admin alice -n team-alpha
oc adm policy add-role-to-user view alice -n team-beta

# 4. Grant bob admin on team-beta only
oc adm policy add-role-to-user admin bob -n team-beta

# 5. Set quotas per project
oc create quota team-quota \
  --hard=pods=10,requests.cpu=2,requests.memory=4Gi \
  -n team-alpha

# 6. Default NetworkPolicy (deny all ingress between projects)
oc apply -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: deny-other-namespaces
  namespace: team-alpha
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}   # Only from same namespace
EOF

# 7. Verify alice cannot access team-beta's deployments
oc get deployments --as=alice -n team-beta   # Should get pods (view role)
oc delete pod --as=alice -n team-beta        # Should be forbidden
```

---

## oc CLI Cheat Sheet

```bash
# ─── LOGIN & CONTEXT ─────────────────────────────────
oc login https://api.mycluster.example.com:6443 --token=<token>
oc login -u admin -p password
oc whoami
oc whoami --show-console
oc config get-contexts
oc project my-app                        # Switch project

# ─── PROJECTS ────────────────────────────────────────
oc new-project my-app
oc projects
oc get projects
oc delete project my-app

# ─── APPS & BUILDS ───────────────────────────────────
oc new-app nodejs~https://github.com/myorg/app.git --name=myapp
oc new-app --template=mysql-ephemeral
oc start-build myapp
oc start-build myapp --from-dir=.
oc logs -f bc/myapp                      # Build logs
oc cancel-build myapp-3
oc get builds
oc get bc                                # BuildConfigs
oc get is                                # ImageStreams

# ─── GET / DESCRIBE ──────────────────────────────────
oc get all -n my-app
oc get pods -o wide
oc get pods -l app=myapp
oc get events --sort-by='.lastTimestamp'
oc describe pod <name>
oc describe node <name>
oc describe route <name>

# ─── LOGS & DEBUG ────────────────────────────────────
oc logs <pod> -f
oc logs <pod> --previous
oc rsh <pod>                             # Remote shell
oc debug <pod>                           # Debug copy of pod
oc debug node/<node>                     # Node-level debug
oc exec -it <pod> -- bash
oc port-forward pod/<pod> 8080:8080
oc port-forward svc/myapp 8080:8080

# ─── ROUTES ──────────────────────────────────────────
oc expose svc/myapp
oc expose svc/myapp --hostname=myapp.example.com
oc get routes
oc create route edge myapp-tls --service=myapp

# ─── DEPLOYMENT / ROLLOUT ────────────────────────────
oc rollout status deployment/myapp
oc rollout undo deployment/myapp
oc rollout history deployment/myapp
oc rollout restart deployment/myapp
oc set image deployment/myapp app=myapp:v2
oc scale deployment/myapp --replicas=5
oc autoscale deployment/myapp --min=2 --max=10 --cpu-percent=70

# ─── RBAC / POLICY ───────────────────────────────────
oc adm policy add-role-to-user admin alice -n my-app
oc adm policy add-scc-to-user anyuid -z my-sa -n my-app
oc adm policy who-can get pods -n my-app
oc auth can-i delete pods --as=alice -n my-app
oc get rolebindings -n my-app

# ─── ADMIN ───────────────────────────────────────────
oc adm top nodes
oc adm top pods -n my-app
oc adm cordon <node>
oc adm drain <node> --ignore-daemonsets --delete-emptydir-data
oc adm uncordon <node>
oc adm upgrade
oc adm certificate approve <csr>
oc get clusterversion
oc get co                                # Cluster Operators
oc get mcp                               # MachineConfigPools
oc get mc                                # MachineConfigs
oc get csr                               # Certificate Signing Requests

# ─── SECRETS & CONFIG ────────────────────────────────
oc create secret generic my-secret --from-literal=password=abc123
oc create secret docker-registry pull-secret \
  --docker-server=quay.io \
  --docker-username=user \
  --docker-password=pass
oc create configmap my-config --from-file=config.yaml
oc set env deployment/myapp DB_URL=postgres://...
oc set env deployment/myapp --from=secret/my-secret
oc set resources deployment/myapp --requests=cpu=100m,memory=128Mi

# ─── OPERATORS / OLM ─────────────────────────────────
oc get csv -n openshift-operators        # ClusterServiceVersions
oc get subscription -n openshift-operators
oc get installplan -n openshift-operators
oc get packagemanifests -n openshift-marketplace
```

---

> **Cross-links:** [← Kubernetes](./Kubernetes.md) | [Jenkins →](./Jenkins.md) | [CI/CD (Tekton/ArgoCD) →](./CICD.md) | [Security (RBAC/SCC) →](./Security.md) | [Monitoring →](./Monitoring.md)
