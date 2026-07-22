---
render_with_liquid: false
layout: default
title: "⚙️ Module 5 — Infrastructure & Automation"

---

{% raw %}

# ⚙️ Module 5 — Infrastructure & Automation

> **JD Alignment:** "Automate infrastructure provisioning and configuration management. Collaborate with cross-functional teams to integrate DevOps best practices. Maintain documentation related to deployment processes and system architecture."

> [← Monitoring & Troubleshooting](./04-Monitoring-Troubleshooting.md) | [README](./README.md) | [Scenario-Based →](./06-Scenario-Based.md)

---

## Table of Contents

1. [Terraform](#terraform)
2. [Ansible](#ansible)
3. [Helm](#helm)
4. [Kustomize](#kustomize)
5. [GitOps at Scale](#gitops-at-scale)
6. [OpenShift Cluster Provisioning](#openshift-cluster-provisioning)

---

## Terraform

---

### 🟡 Q1. How do you provision an OpenShift cluster on AWS (ROSA) using Terraform?

**Answer:**

```hcl
# main.tf — ROSA cluster via Terraform (using rhcs provider)
terraform {
  required_providers {
    rhcs = {
      source  = "terraform-redhat/rhcs"
      version = "~> 1.5"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "rosa/terraform.tfstate"
    region = "us-east-1"
    encrypt = true
  }
}

provider "rhcs" {
  token = var.rhcs_token     # Red Hat cloud.redhat.com token
  url   = "https://api.openshift.com"
}

provider "aws" {
  region = var.aws_region
}

# VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  name    = "${var.cluster_name}-vpc"
  cidr    = "10.0.0.0/16"
  azs     = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  enable_nat_gateway = true
  single_nat_gateway = false    # HA: one NAT GW per AZ
}

# ROSA Cluster
resource "rhcs_cluster_rosa_classic" "rosa" {
  name               = var.cluster_name
  cloud_region       = var.aws_region
  multi_az           = true
  version            = "4.14.5"
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
  aws_account_id     = data.aws_caller_identity.current.account_id

  sts = {
    oidc_config_id       = rhcs_oidc_config.oidc.id
    operator_role_prefix = var.cluster_name
    role_arn             = aws_iam_role.installer.arn
    support_role_arn     = aws_iam_role.support.arn
    instance_iam_roles = {
      master_role_arn = aws_iam_role.controlplane.arn
      worker_role_arn = aws_iam_role.worker.arn
    }
  }

  aws_subnet_ids = concat(
    module.vpc.private_subnets,
    module.vpc.public_subnets
  )

  compute_machine_type = "m5.xlarge"
  replicas             = 3

  properties = {
    rosa_creator_arn = data.aws_caller_identity.current.arn
  }
}

# Machine Pool for application workloads
resource "rhcs_machine_pool" "app_pool" {
  cluster      = rhcs_cluster_rosa_classic.rosa.id
  name         = "app-workers"
  machine_type = "m5.2xlarge"
  replicas     = 3
  labels = {
    "workload-type" = "application"
  }
}
```

```bash
# Apply
terraform init
terraform plan -var-file=prod.tfvars
terraform apply -var-file=prod.tfvars
```

---

### 🟡 Q2. How do you manage Terraform state for team-based infrastructure?

**Answer:**

**Remote state with locking (S3 + DynamoDB):**
```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "openshift/production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"   # Prevents concurrent applies
    kms_key_id     = "arn:aws:kms:us-east-1:123:key/abc123"
  }
}
```

**Workspace strategy for multi-environment:**
```bash
# Separate state per environment
terraform workspace new production
terraform workspace new staging
terraform workspace select production

# Reference workspace in config
locals {
  env_config = {
    production = { replicas = 6, instance_type = "m5.2xlarge" }
    staging    = { replicas = 3, instance_type = "m5.xlarge" }
  }
  config = local.env_config[terraform.workspace]
}
```

**Best practices:**
- Never store state in Git — it contains secrets in plaintext.
- Use `-target` sparingly — for break-glass scenarios only.
- Pin provider versions in `required_providers` and `terraform.lock.hcl` in Git.
- Use `terraform plan -out=tfplan` + `terraform apply tfplan` in CI — apply the exact same plan that was reviewed.
- Separate Terraform modules per component (vpc, cluster, dns, monitoring).

---

### 🔴 Q3. How do you handle Terraform drift detection and remediation?

**Answer:**

**Drift** = actual cloud state diverges from Terraform state (manual changes, auto-scaling, external processes).

```bash
# Detect drift
terraform plan     # Shows any differences between state and real infra

# Refresh state to match reality (without making changes)
terraform refresh  # Updates state file from real infra; deprecated in TF 1.x
terraform plan -refresh-only   # Newer approach: see what refreshed state would be

# Import manually-created resources into state
terraform import aws_instance.web i-1234567890abcdef0

# Automated drift detection in CI (run nightly)
terraform plan -detailed-exitcode
# Exit code 0: no changes; 1: error; 2: changes detected
if [ $? -eq 2 ]; then
  notify-slack "DRIFT DETECTED in production infrastructure!"
fi
```

**In GitOps with Atlantis:**
```yaml
# atlantis.yaml — auto-plan on PR, manual apply
version: 3
projects:
- name: openshift-production
  dir: terraform/openshift
  workspace: production
  autoplan:
    when_modified: ["*.tf", "*.tfvars"]
    enabled: true
  apply_requirements: [approved, mergeable]
```

---

## Ansible

---

### 🟡 Q4. How do you use Ansible for OpenShift post-installation configuration?

**Answer:**

```yaml
# playbooks/configure-openshift.yml
---
- name: Configure OpenShift cluster post-install
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    cluster_api: "https://api.mycluster.example.com:6443"
    kubeconfig: "~/.kube/config"

  tasks:

  - name: Create standard namespaces
    kubernetes.core.k8s:
      state: present
      definition:
        apiVersion: project.openshift.io/v1
        kind: Project
        metadata:
          name: "{{ item }}"
          annotations:
            openshift.io/display-name: "{{ item | capitalize }}"
    loop:
      - production
      - staging
      - monitoring

  - name: Apply ResourceQuotas to all namespaces
    kubernetes.core.k8s:
      state: present
      definition:
        apiVersion: v1
        kind: ResourceQuota
        metadata:
          name: compute-quota
          namespace: "{{ item }}"
        spec:
          hard:
            pods: "50"
            requests.cpu: "10"
            requests.memory: "20Gi"
            limits.cpu: "20"
            limits.memory: "40Gi"
    loop:
      - production
      - staging

  - name: Configure HTPasswd identity provider
    kubernetes.core.k8s:
      state: present
      definition:
        apiVersion: config.openshift.io/v1
        kind: OAuth
        metadata:
          name: cluster
        spec:
          identityProviders:
          - name: htpasswd
            type: HTPasswd
            htpasswd:
              fileData:
                name: htpasswd-secret

  - name: Deploy cluster-wide NetworkPolicy defaults
    kubernetes.core.k8s:
      state: present
      src: "files/networkpolicies/{{ item }}.yaml"
    loop:
      - deny-other-namespaces
      - allow-from-router
      - allow-monitoring

  - name: Install operators via Subscriptions
    kubernetes.core.k8s:
      state: present
      definition: "{{ lookup('file', 'files/subscriptions/' + item + '.yaml') | from_yaml }}"
    loop:
      - openshift-pipelines
      - openshift-gitops
      - cert-manager
      - external-secrets

  - name: Configure cluster logging
    kubernetes.core.k8s:
      state: present
      src: files/logging/clusterlogging-instance.yaml

  - name: Set cluster-wide proxy (if needed)
    kubernetes.core.k8s:
      state: present
      definition:
        apiVersion: config.openshift.io/v1
        kind: Proxy
        metadata:
          name: cluster
        spec:
          httpProxy: "{{ http_proxy | default('') }}"
          httpsProxy: "{{ https_proxy | default('') }}"
          noProxy: ".cluster.local,.svc,localhost,10.0.0.0/8"
    when: http_proxy is defined
```

```bash
# Run playbook
ansible-playbook playbooks/configure-openshift.yml \
  -e @vars/production.yml \
  --diff
```

---

### 🟡 Q5. How do you use Ansible roles and best practices for maintainability?

**Answer:**

```
roles/
└── openshift-project/
    ├── tasks/
    │   ├── main.yml
    │   ├── create-project.yml
    │   └── configure-rbac.yml
    ├── defaults/
    │   └── main.yml    # Default variable values
    ├── vars/
    │   └── main.yml    # Variables that override defaults
    ├── templates/
    │   ├── quota.yaml.j2
    │   └── rolebinding.yaml.j2
    ├── files/
    │   └── networkpolicy.yaml
    ├── handlers/
    │   └── main.yml    # Triggered by notify
    ├── meta/
    │   └── main.yml    # Role dependencies
    └── README.md
```

```yaml
# roles/openshift-project/defaults/main.yml
openshift_project_quota_cpu_requests: "4"
openshift_project_quota_memory_requests: "8Gi"
openshift_project_quota_pods: "20"
openshift_project_admin_users: []

# roles/openshift-project/tasks/main.yml
---
- name: Create project
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: project.openshift.io/v1
      kind: Project
      metadata:
        name: "{{ openshift_project_name }}"
  tags: [project]

- name: Apply quota
  kubernetes.core.k8s:
    state: present
    template: quota.yaml.j2
  tags: [quota]

- include_tasks: configure-rbac.yml
  tags: [rbac]
```

**Ansible Vault for secrets:**
```bash
# Encrypt sensitive vars
ansible-vault encrypt_string 'super-secret-password' --name 'db_password'
# Store encrypted value in vars file — safe to commit to Git

# Run playbook with vault password
ansible-playbook playbook.yml --ask-vault-pass
# Or from file:
ansible-playbook playbook.yml --vault-password-file ~/.vault-pass
```

---

## Helm

---

### 🟡 Q6. Explain Helm chart structure and how you use it for OpenShift deployments.

**Answer:**

```
my-app-chart/
├── Chart.yaml           # Chart metadata (name, version, appVersion)
├── values.yaml          # Default configuration values
├── values-production.yaml  # Production overrides (in source control)
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── route.yaml       # OpenShift Route
│   ├── serviceaccount.yaml
│   ├── rbac.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   ├── _helpers.tpl     # Template helper functions
│   └── NOTES.txt        # Post-install instructions
├── charts/              # Chart dependencies (sub-charts)
└── .helmignore
```

```yaml
# Chart.yaml
apiVersion: v2
name: my-nodeapp
description: Node.js application for OpenShift
type: application
version: 1.2.3           # Chart version
appVersion: "2.0.1"      # App version being packaged

# values.yaml
image:
  repository: image-registry.openshift-image-registry.svc:5000/my-app/my-nodeapp
  tag: "latest"
  pullPolicy: IfNotPresent

replicaCount: 2

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

route:
  enabled: true
  host: ""               # Auto-generated if empty
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-nodeapp.fullname" . }}
  labels: {{ include "my-nodeapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels: {{ include "my-nodeapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels: {{ include "my-nodeapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        resources: {{ toYaml .Values.resources | nindent 10 }}
```

```bash
# Install to OpenShift
helm install my-nodeapp ./my-app-chart \
  -f values-production.yaml \
  -n production \
  --create-namespace

# Upgrade
helm upgrade my-nodeapp ./my-app-chart \
  -f values-production.yaml \
  -n production

# Dry-run before upgrade
helm upgrade --dry-run --debug my-nodeapp ./my-app-chart -f values-production.yaml

# Rollback
helm rollback my-nodeapp 2 -n production

# Diff (helm-diff plugin)
helm diff upgrade my-nodeapp ./my-app-chart -f values-production.yaml
```

---

## Kustomize

---

### 🟡 Q7. What is Kustomize and how does it differ from Helm? When would you choose each?

**Answer:**

| Feature | Helm | Kustomize |
|---------|------|-----------|
| Templating | Go templates — full logic | JSON/YAML patches — no templates |
| Values | `values.yaml` | `kustomization.yaml` overlays |
| Complexity | Higher (Go templating) | Lower (YAML patching) |
| Packaging | Charts — versioned, shareable via registry | Directories — Git-native |
| K8s integration | External tool | Built into `kubectl -k` and `oc` |
| OpenShift Routes | Can template Route YAML | Can patch Route YAML |
| Best for | Complex apps with many configurable options | Environment-specific overlays, GitOps |

**Kustomize structure:**
```
k8s/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── route.yaml
└── overlays/
    ├── staging/
    │   ├── kustomization.yaml
    │   ├── replica-patch.yaml    # Override replicas
    │   └── resource-patch.yaml  # Override resources
    └── production/
        ├── kustomization.yaml
        ├── replica-patch.yaml
        └── image-patch.yaml     # Set production image
```

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production
namePrefix: prod-

resources:
- ../../base

images:
- name: myapp
  newName: image-registry.openshift-image-registry.svc:5000/production/myapp
  newTag: "v1.2.3"

patchesStrategicMerge:
- replica-patch.yaml

patches:
- target:
    kind: Deployment
    name: myapp
  patch: |
    - op: replace
      path: /spec/template/spec/containers/0/resources/limits/memory
      value: 1Gi
```

```bash
# Apply with kustomize
oc apply -k k8s/overlays/production/

# Preview output
oc kustomize k8s/overlays/production/

# ArgoCD uses kustomize natively — just point to the overlay dir
```

---

## GitOps at Scale

---

### 🔴 Q8. How do you manage hundreds of applications across multiple clusters with GitOps?

**Answer:**

**ApplicationSet** — ArgoCD resource that generates Applications from templates:

```yaml
# ApplicationSet: deploy to all clusters matching labels
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: all-apps
  namespace: openshift-gitops
spec:
  generators:
  # Matrix generator: all apps × all clusters
  - matrix:
      generators:
      # List of apps from Git directory structure
      - git:
          repoURL: https://github.com/myorg/gitops-config.git
          revision: main
          directories:
          - path: apps/*

      # Clusters from ArgoCD cluster secrets
      - clusters:
          selector:
            matchLabels:
              environment: production

  template:
    metadata:
      name: "{{path.basename}}-{{name}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/gitops-config.git
        targetRevision: main
        path: "{{path}}/overlays/production"
      destination:
        server: "{{server}}"
        namespace: "{{path.basename}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

**App-of-Apps pattern:**
```yaml
# Root application that manages all other ArgoCD Applications
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: openshift-gitops
spec:
  source:
    path: clusters/production/apps    # Directory of Application manifests
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Multi-cluster strategy:**
```
gitops-repo/
├── clusters/
│   ├── production-us-east-1/
│   │   ├── apps/           # ArgoCD Applications for this cluster
│   │   ├── infra/          # Cluster-level infra (monitoring, logging)
│   │   └── policies/       # Kyverno/OPA policies
│   ├── production-eu-west-1/
│   └── staging/
└── apps/
    ├── my-nodeapp/
    │   ├── base/
    │   └── overlays/
    └── my-api/
```

---

## OpenShift Cluster Provisioning

---

### 🔴 Q9. Explain the OpenShift IPI vs UPI installation methods.

**Answer:**

| Method | Stands For | Use Case |
|--------|-----------|---------|
| **IPI** | Installer-Provisioned Infrastructure | Installer creates all cloud infra + cluster. Fastest, least control. Good for: AWS, Azure, GCP, vSphere. |
| **UPI** | User-Provisioned Infrastructure | You provision infra (VMs, LBs, DNS, VPC); installer only creates the OpenShift layer. More control. Required for: bare-metal, restricted networks, custom networking. |
| **Assisted Installer** | — | Web/API-based installation for bare-metal without existing Kubernetes. |
| **ACM** (Advanced Cluster Management) | — | Multi-cluster provisioning and lifecycle management. |

**IPI on AWS (install-config.yaml):**
```yaml
apiVersion: v1
baseDomain: example.com
metadata:
  name: mycluster
platform:
  aws:
    region: us-east-1
    userTags:
      Environment: production
      Team: platform
controlPlane:
  replicas: 3
  platform:
    aws:
      type: m5.2xlarge
compute:
- name: worker
  replicas: 3
  platform:
    aws:
      type: m5.xlarge
      zones: [us-east-1a, us-east-1b, us-east-1c]
networking:
  networkType: OVNKubernetes
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16
  machineNetwork:
  - cidr: 10.0.0.0/16
pullSecret: '{"auths": {...}}'
sshKey: 'ssh-rsa AAAA...'
fips: false
```

```bash
# Run IPI install
openshift-install create cluster --dir ./install-config/
openshift-install wait-for install-complete --dir ./install-config/
```

---

### 🔴 Q10. How do you implement GitOps-driven cluster configuration with ACM (Advanced Cluster Management)?

**Answer:**

Red Hat ACM (Advanced Cluster Management) manages multiple OpenShift clusters from a single hub.

```yaml
# ACM Policy: enforce PodSecurity across all managed clusters
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: enforce-pod-security
  namespace: policies
  annotations:
    policy.open-cluster-management.io/standards: NIST-CSF
    policy.open-cluster-management.io/categories: PR.PT
spec:
  remediationAction: enforce   # or inform (audit-only)
  disabled: false
  policy-templates:
  - objectDefinition:
      apiVersion: policy.open-cluster-management.io/v1
      kind: ConfigurationPolicy
      metadata:
        name: pod-security-restricted
      spec:
        remediationAction: enforce
        severity: high
        object-templates:
        - complianceType: musthave
          objectDefinition:
            apiVersion: v1
            kind: Namespace
            metadata:
              labels:
                pod-security.kubernetes.io/enforce: restricted

---
# PolicyBinding: apply to all clusters with label env=production
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: pod-security-binding
  namespace: policies
placementRef:
  apiGroup: cluster.open-cluster-management.io
  kind: PlacementRule
  name: all-production-clusters
subjects:
- apiGroup: policy.open-cluster-management.io
  kind: Policy
  name: enforce-pod-security
```

```bash
# Check policy compliance across all clusters
oc get policy -n policies
oc get placementdecision -n policies
```

---

### 🟡 Q11. How do you handle Day-2 operations for OpenShift using Ansible Automation Platform?

**Answer:**

Red Hat Ansible Automation Platform (AAP) integrates with OpenShift via the Automation controller (formerly Ansible Tower) and the `kubernetes.core` collection.

**Common Day-2 tasks automated with AAP:**

```yaml
# Job Template: Rotate all service account tokens in a namespace
- name: Rotate SA tokens
  hosts: localhost
  tasks:
  - name: Get all service accounts
    kubernetes.core.k8s_info:
      api_version: v1
      kind: ServiceAccount
      namespace: "{{ target_namespace }}"
    register: sa_list

  - name: Delete SA secrets to force token rotation
    kubernetes.core.k8s:
      state: absent
      api_version: v1
      kind: Secret
      name: "{{ item }}"
      namespace: "{{ target_namespace }}"
    loop: "{{ sa_list.resources | map(attribute='secrets') | flatten | map(attribute='name') | list }}"
```

**AAP integration with OpenShift:**
```bash
# Install AAP operator from OperatorHub
# Create AutomationController instance
# Create execution environments (EE) with kubernetes.core collection

# Webhook trigger: trigger Ansible job from OpenShift event
# (e.g., when new namespace is created → run namespace-bootstrap playbook)
```

---

### 🟡 Q12. How do you document infrastructure and keep documentation current?

**Answer:**

**Documentation strategy (living documentation):**

1. **Architecture as code → Architecture as document:**
```bash
# Generate architecture diagram from Terraform
terraform graph | dot -Tpng > architecture.png

# Or use Diagrams-as-code (Python diagrams library)
# diagram.py generates PNG architecture diagram
```

2. **Runbooks in Git alongside code:**
```
docs/
├── runbooks/
│   ├── cluster-upgrade.md
│   ├── incident-response.md
│   ├── node-replacement.md
│   └── certificate-rotation.md
├── architecture/
│   ├── cluster-overview.md
│   └── network-topology.md
└── onboarding/
    └── new-developer-guide.md
```

3. **Automated README generation for Terraform:**
```bash
# terraform-docs generates README from module variables/outputs
terraform-docs markdown table ./modules/openshift-cluster > README.md
```

4. **OpenAPI/Swagger for service APIs:**
```bash
# Auto-generate from code annotations
# Publish to developer portal (Backstage on OpenShift)
```

5. **Changelog with conventional commits:**
```bash
# git cliff generates CHANGELOG.md from conventional commits
git cliff --unreleased --tag v1.2.0 -o CHANGELOG.md
```

**Measure documentation quality:**
- Link checks in CI (broken links = outdated docs).
- Alert when runbook URL referenced in a Prometheus alert returns 404.
- Require runbook link in every new alert PR.

---

### 🔴 Q13. How do you implement infrastructure cost optimization for OpenShift on cloud?

**Answer:**

**Strategies:**

**1. Right-size compute with Cluster Autoscaler:**
```bash
# Scale down unused node pools automatically
# Enable scale-down with short delay for non-critical environments
oc edit clusterautoscaler default
# scaleDown.delayAfterAdd: 2m (staging) vs 10m (production)
```

**2. Use spot/preemptible instances for non-critical workloads:**
```yaml
# MachineSet with spot instances (AWS)
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  name: worker-spot-us-east-1a
spec:
  template:
    spec:
      providerSpec:
        value:
          spotMarketOptions:
            maxPrice: "0.10"    # Max price per hour
```

```yaml
# Toleration for spot nodes
spec:
  tolerations:
  - key: "spot-instance"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

**3. Cluster hibernation for non-production:**
```bash
# ROSA: hibernate dev/staging clusters during off-hours
rosa hibernate cluster -c my-dev-cluster

# Resume Monday morning
rosa resume cluster -c my-dev-cluster
# Saves ~70% cost for 8×5 dev clusters
```

**4. Monitor resource utilization:**
```promql
# Identify over-provisioned namespaces
# CPU requested vs actual usage
sum(kube_pod_container_resource_requests{resource="cpu",namespace="my-app"})
  /
sum(rate(container_cpu_usage_seconds_total{namespace="my-app"}[1h]))
# If ratio > 5 → pods are over-requesting CPU
```

**5. Use Goldilocks + VPA for recommendations:**
```bash
kubectl label namespace my-app goldilocks.fairwinds.com/enabled=true
kubectl port-forward svc/goldilocks-dashboard 8080 -n goldilocks
# Open http://localhost:8080 → shows VPA recommendations per deployment
```

**6. Storage lifecycle policies:**
```bash
# Delete unused PVCs (orphaned after pod deletion)
oc get pvc --all-namespaces | grep Released
# EBS volumes remain after PVC deletion — attach lifecycle policy

# S3 storage for internal registry: lifecycle policy to delete old layers
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-registry-bucket \
  --lifecycle-configuration file://registry-lifecycle.json
```

---

### 🟡 Q14. How do you implement configuration management across multiple OpenShift clusters?

**Answer:**

**Using ACM + Git (GitOps at cluster level):**

```yaml
# ACM Channel: points to Git repo
apiVersion: apps.open-cluster-management.io/v1
kind: Channel
metadata:
  name: gitops-config-channel
  namespace: default
spec:
  type: Git
  pathname: https://github.com/myorg/cluster-config.git
  secretRef:
    name: github-credentials

---
# Subscription: deploys resources from Git to managed clusters
apiVersion: apps.open-cluster-management.io/v1
kind: Subscription
metadata:
  name: cluster-base-config
  namespace: default
spec:
  channel: default/gitops-config-channel
  placement:
    placementRef:
      name: all-production-clusters
      kind: PlacementRule
  packageFilter:
    version: ">=1.0.0"
  overrides: []
```

**Using SealedSecrets for environment-specific secrets in Git:**
```bash
# Encrypt secret with cluster public key — safe to store in Git
kubeseal --controller-namespace kube-system \
  --controller-name sealed-secrets \
  < secret.yaml > sealed-secret.yaml

git add sealed-secret.yaml    # Safe to commit
# SealedSecrets operator decrypts on the target cluster using its private key
```

---

### 🟡 Q15. Describe your process for onboarding a new application team onto OpenShift.

**Answer:**

**Onboarding checklist and automation:**

```yaml
# Ansible playbook: onboard-team.yml
# Triggered by: ServiceNow ticket or self-service portal
---
- name: Onboard new application team
  hosts: localhost
  vars:
    team_name: "{{ team_name }}"          # e.g., "payments-team"
    app_namespace: "{{ app_namespace }}"  # e.g., "payments"
    team_admins: "{{ team_admins }}"      # List of usernames
    cpu_quota: "{{ cpu_quota | default('8') }}"
    memory_quota: "{{ memory_quota | default('16Gi') }}"

  tasks:
  - name: Create project (namespace)
    kubernetes.core.k8s:
      definition:
        apiVersion: project.openshift.io/v1
        kind: Project
        metadata:
          name: "{{ app_namespace }}"
          annotations:
            openshift.io/display-name: "{{ team_name }}"
            openshift.io/requester: "{{ team_admins[0] }}"

  - name: Apply ResourceQuota
    # ...

  - name: Apply LimitRange defaults
    # ...

  - name: Create service accounts
    # pipeline-sa, app-sa

  - name: Grant RBAC
    # admin role to team_admins
    # edit role to team_members

  - name: Configure NetworkPolicies
    # deny-other-namespaces, allow-from-router

  - name: Set up CI/CD service account
    # grant pipeline-sa the deployer role

  - name: Create default ImagePullSecret
    # for pulling from internal registry

  - name: Notify team via Slack
    community.general.slack:
      token: "{{ slack_token }}"
      channel: "#platform-onboarding"
      msg: "✅ Namespace '{{ app_namespace }}' ready for team {{ team_name }}"
```

**Self-service portal integration:**
- Backstage Template → form submission → triggers Ansible job → creates namespace.
- Team receives email/Slack with: cluster URL, namespace name, how to get kubeconfig, documentation links.

---

### 🔴 Q16. How do you use OpenShift's MachineConfig Operator to customize node OS configuration?

**Answer:**

The MCO manages RHCOS (Red Hat CoreOS) node configuration declaratively.

```yaml
# Custom kernel arguments for high-performance workloads
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-performance-tuning
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  kernelArguments:
  - "transparent_hugepages=always"
  - "numa_balancing=0"
  kernelType: default

---
# Custom sysctl settings
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-sysctl
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,bmV0LmlwdjQudGNwX3R3X3JldXNlPTEKbmV0LmNvcmUuc29tYXhibWVtPTI2MjE0NDAK
          # net.ipv4.tcp_tw_reuse=1
          # net.core.somaxconn=262144
        mode: 0644
        path: /etc/sysctl.d/99-custom.conf

---
# Add custom CA certificate to all nodes
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-custom-ca
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
      - contents:
          source: data:text/plain;base64,LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...
        mode: 0644
        path: /etc/pki/ca-trust/source/anchors/my-internal-ca.crt
  extensions:
  - update-ca-trust   # Runs update-ca-trust after file placement
```

```bash
# Monitor MCO rollout (nodes reboot one by one)
oc get mcp worker -w
# MACHINECOUNT READYMACHINECOUNT UPDATEDMACHINECOUNT DEGRADEDMACHINECOUNT
# 3            3                 3                   0   ← Done

# Pause MCO to batch changes (apply multiple MachineConfigs, then reboot once)
oc patch mcp worker --type merge -p '{"spec":{"paused":true}}'
oc apply -f mc-1.yaml
oc apply -f mc-2.yaml
oc apply -f mc-3.yaml
oc patch mcp worker --type merge -p '{"spec":{"paused":false}}'
# Nodes now reboot once with all three changes applied
```

---

### 🟡 Q17. What is the OpenShift Node Feature Discovery (NFD) operator and how do you use it?

**Answer:**

**NFD** automatically detects hardware features on nodes (CPU capabilities, GPU presence, NIC types, etc.) and adds them as node labels. This enables workloads to be scheduled on nodes with the right hardware.

```bash
# Install NFD operator
# After installation, NFD labels nodes automatically

# Example labels added by NFD
kubectl get node worker-gpu-1 --show-labels | tr ',' '\n' | grep feature
# feature.node.kubernetes.io/cpu-cpuid.AVX512F=true
# feature.node.kubernetes.io/pci-10de.present=true  ← NVIDIA GPU detected
# feature.node.kubernetes.io/system-os_release.ID=rhcos

# Schedule GPU workloads using NFD labels
spec:
  nodeSelector:
    feature.node.kubernetes.io/pci-10de.present: "true"
```

```yaml
# NFD + NVIDIA GPU Operator combination
# NFD detects GPU node → NVIDIA operator installs drivers → workload runs
spec:
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
  containers:
  - name: gpu-workload
    image: nvcr.io/nvidia/cuda:12.0-base
    resources:
      limits:
        nvidia.com/gpu: 1    # Request 1 GPU
```

---

### 🟡 Q18. How do you implement disaster recovery for an OpenShift cluster?

**Answer:**

**RPO/RTO targets define the strategy:**
- **RPO** (Recovery Point Objective): How much data loss is acceptable? (hours, minutes?)
- **RTO** (Recovery Time Objective): How quickly must the cluster be restored?

**Three tiers:**

**Tier 1: etcd backup + restore (RTO: 1–2 hours):**
```bash
# Automated etcd backup (should run daily via CronJob)
oc apply -f - <<EOF
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: openshift-etcd
spec:
  schedule: "0 2 * * *"    # 2am daily
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          nodeSelector:
            node-role.kubernetes.io/master: ""
          tolerations:
          - effect: NoSchedule
            operator: Exists
          containers:
          - name: backup
            image: registry.redhat.io/openshift4/ose-cli:latest
            command: ["/bin/bash", "-c"]
            args:
            - |
              /usr/local/bin/cluster-backup.sh /home/core/backup
              aws s3 cp /home/core/backup/ s3://my-backup-bucket/etcd/ --recursive
EOF

# Restore from etcd snapshot
openshift-install --dir=./backup restore-etcd \
  --backup-dir=/home/core/backup
```

**Tier 2: Velero backup (application data + PVs):**
```bash
# Install Velero with AWS plugin
velero install \
  --provider aws \
  --bucket my-backup-bucket \
  --backup-location-config region=us-east-1

# Backup a namespace
velero backup create my-app-backup \
  --include-namespaces=my-app \
  --wait

# Scheduled backup
velero schedule create daily-backup \
  --schedule="@daily" \
  --include-namespaces=production,staging

# Restore
velero restore create --from-backup my-app-backup
```

**Tier 3: Multi-cluster active-active (RTO: seconds via DNS failover):**
- Two clusters in different regions/clouds.
- ArgoCD deploys same apps to both clusters.
- Global LB (Route 53, Cloudflare) does health-check-based failover.
- Stateful data replicated via ODF stretched cluster or external DB replication.

---

> **Next:** [Scenario-Based Questions →](./06-Scenario-Based.md)

{% endraw %}