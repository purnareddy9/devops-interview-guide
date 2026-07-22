# 🟦 GCP — DevOps Interview Guide

> [← Azure](./Azure.md) | [Main Index](./README.md) | [CI/CD →](./CICD.md)

---

## Table of Contents

1. [GCP Core Concepts](#gcp-core-concepts)
2. [Identity & IAM](#identity--iam)
3. [Networking (VPC, Cloud Load Balancing)](#networking)
4. [Compute (GCE, GKE, Cloud Run)](#compute)
5. [Storage (GCS, Cloud SQL, Spanner)](#storage)
6. [Cloud Build & Artifact Registry](#cloud-build--artifact-registry)
7. [Operations (Cloud Monitoring, Logging)](#operations)
8. [Interview Questions](#interview-questions)
9. [Scenario-Based Questions](#scenario-based-questions)
10. [Cheat Sheet](#cheat-sheet)

---

## GCP Core Concepts

### Resource Hierarchy

```
Organization (company.com)
└── Folder (Division / Team)
    └── Project
        └── Resources (VMs, GKE, Storage, etc.)

IAM policies are inherited down the hierarchy.
```

### GCP vs AWS vs Azure Equivalents

| GCP | AWS | Azure |
|-----|-----|-------|
| Project | Account | Subscription |
| VPC | VPC | VNet |
| GCE | EC2 | VM |
| GKE | EKS | AKS |
| Cloud Run | Fargate/Lambda | Container Apps |
| GCS | S3 | Blob Storage |
| Cloud SQL | RDS | Azure Database |
| Pub/Sub | SNS/SQS | Service Bus |
| Cloud Build | CodeBuild | Azure Pipelines |
| Artifact Registry | ECR | ACR |
| Cloud IAM | AWS IAM | Azure RBAC/AAD |
| Cloud Monitoring | CloudWatch | Azure Monitor |
| BigQuery | Redshift | Synapse Analytics |

---

## Identity & IAM

### IAM Hierarchy

```
Who (Principal) → Can do What (Role) → On Which Resource
```

### Principal Types

| Principal | Description |
|-----------|-------------|
| **Google Account** | Individual user (person@gmail.com) |
| **Service Account** | Non-human identity for apps/VMs |
| **Google Group** | Collection of accounts |
| **Workspace Domain** | All users in a Google Workspace domain |
| **allUsers** | Anyone on the internet (public) |
| **allAuthenticatedUsers** | Any authenticated Google account |

### Service Accounts

```bash
# Create service account
gcloud iam service-accounts create github-actions-sa \
  --display-name="GitHub Actions SA" \
  --project=my-project

# Grant roles
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:github-actions-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/container.developer"

# Workload Identity Federation (OIDC — no key files needed)
gcloud iam workload-identity-pools create github-pool \
  --location=global \
  --project=my-project

gcloud iam workload-identity-pools providers create-oidc github-provider \
  --location=global \
  --workload-identity-pool=github-pool \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --project=my-project

# Bind pool to service account
gcloud iam service-accounts add-iam-policy-binding \
  github-actions-sa@my-project.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/my-org/my-repo"
```

---

## Networking

### VPC Design

```bash
# Create custom VPC
gcloud compute networks create myapp-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=global

# Create subnets with secondary ranges (for GKE pods/services)
gcloud compute networks subnets create web-subnet \
  --network=myapp-vpc \
  --region=us-central1 \
  --range=10.0.1.0/24 \
  --secondary-range=pod-range=172.16.0.0/14,svc-range=172.20.0.0/20 \
  --enable-private-ip-google-access

# Firewall rules
gcloud compute firewall-rules create allow-internal \
  --network=myapp-vpc \
  --allow=tcp,udp,icmp \
  --source-ranges=10.0.0.0/8

gcloud compute firewall-rules create allow-https \
  --network=myapp-vpc \
  --allow=tcp:443 \
  --target-tags=https-server

# Cloud NAT (for private instances to access internet)
gcloud compute routers create myapp-router \
  --network=myapp-vpc \
  --region=us-central1

gcloud compute routers nats create myapp-nat \
  --router=myapp-router \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges \
  --region=us-central1
```

### Cloud Load Balancing

| LB Type | Layer | Scope | Use Case |
|---------|-------|-------|---------|
| Global External L7 | L7 | Global | HTTP(S) with CDN, multi-region |
| Regional External L7 | L7 | Regional | HTTP(S) within region |
| Global External L4 | L4 | Global | TCP/SSL global |
| Regional Internal L4 | L4 | Regional | Internal microservices |
| Regional Internal L7 | L7 | Regional | Internal HTTP APIs |

---

## Compute

### Google Kubernetes Engine (GKE)

```bash
# Create Autopilot cluster (serverless K8s)
gcloud container clusters create-auto prod-cluster \
  --region=us-central1 \
  --project=my-project

# Create Standard cluster
gcloud container clusters create prod-cluster \
  --region=us-central1 \
  --num-nodes=3 \
  --machine-type=n2-standard-4 \
  --disk-size=100 \
  --enable-ip-alias \
  --subnetwork=web-subnet \
  --cluster-secondary-range-name=pod-range \
  --services-secondary-range-name=svc-range \
  --enable-autoscaling \
  --min-nodes=2 \
  --max-nodes=20 \
  --enable-shielded-nodes \
  --workload-pool=my-project.svc.id.goog \
  --enable-master-authorized-networks \
  --master-authorized-networks=10.0.0.0/8 \
  --project=my-project

# Get credentials
gcloud container clusters get-credentials prod-cluster \
  --region=us-central1 \
  --project=my-project

# Node pools
gcloud container node-pools create spot-pool \
  --cluster=prod-cluster \
  --region=us-central1 \
  --machine-type=n2-standard-4 \
  --spot \
  --num-nodes=0 \
  --enable-autoscaling \
  --min-nodes=0 \
  --max-nodes=50
```

### Workload Identity (GKE + IAM)

```bash
# Create KSA (Kubernetes Service Account)
kubectl create serviceaccount myapp-sa -n production

# Bind KSA to GSA (Google Service Account)
gcloud iam service-accounts add-iam-policy-binding \
  myapp-sa@my-project.iam.gserviceaccount.com \
  --role=roles/iam.workloadIdentityUser \
  --member="serviceAccount:my-project.svc.id.goog[production/myapp-sa]"

# Annotate KSA
kubectl annotate serviceaccount myapp-sa \
  --namespace=production \
  iam.gke.io/gcp-service-account=myapp-sa@my-project.iam.gserviceaccount.com
```

### Cloud Run

```bash
# Deploy container to Cloud Run
gcloud run deploy myapp \
  --image=us-central1-docker.pkg.dev/my-project/myrepo/myapp:latest \
  --region=us-central1 \
  --platform=managed \
  --allow-unauthenticated \
  --min-instances=1 \
  --max-instances=100 \
  --concurrency=80 \
  --cpu=2 \
  --memory=512Mi \
  --set-env-vars=ENV=production \
  --service-account=myapp-sa@my-project.iam.gserviceaccount.com \
  --vpc-connector=myapp-connector \
  --vpc-egress=private-ranges-only

# Update traffic split (canary/blue-green)
gcloud run services update-traffic myapp \
  --region=us-central1 \
  --to-revisions=myapp-v2=10,LATEST=90   # 10% to v2, 90% to latest

gcloud run services update-traffic myapp \
  --region=us-central1 \
  --to-latest                             # All traffic to latest
```

---

## Storage

### Google Cloud Storage (GCS)

```bash
# Create bucket
gcloud storage buckets create gs://my-app-data \
  --location=us-central1 \
  --default-storage-class=STANDARD \
  --uniform-bucket-level-access \
  --public-access-prevention=enforced

# Sync directory
gcloud storage rsync ./local-dir gs://my-app-data/backup --recursive

# Set lifecycle policy
gcloud storage buckets update gs://my-app-data \
  --lifecycle-file=lifecycle.json
```

### Cloud SQL

```bash
# Create Cloud SQL for PostgreSQL
gcloud sql instances create prod-postgres \
  --database-version=POSTGRES_15 \
  --tier=db-custom-4-15360 \
  --region=us-central1 \
  --availability-type=REGIONAL \
  --backup-start-time=03:00 \
  --enable-point-in-time-recovery \
  --deletion-protection \
  --database-flags=max_connections=500 \
  --storage-type=SSD \
  --storage-size=100GB \
  --storage-auto-increase

# Create private IP connection
gcloud sql instances patch prod-postgres \
  --network=myapp-vpc \
  --no-assign-ip
```

---

## Cloud Build & Artifact Registry

### Cloud Build Pipeline

```yaml
# cloudbuild.yaml
steps:
  # Run tests
  - name: 'node:20-alpine'
    entrypoint: 'sh'
    args:
      - '-c'
      - |
        npm ci
        npm test
    id: 'test'

  # Build Docker image
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - 'build'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/myrepo/myapp:$SHORT_SHA'
      - '-t'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/myrepo/myapp:latest'
      - '--cache-from'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/myrepo/myapp:latest'
      - '.'
    id: 'build'
    waitFor: ['test']

  # Scan with Cloud Build vulnerability scanning
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    args:
      - 'gcloud'
      - 'artifacts'
      - 'docker'
      - 'images'
      - 'scan'
      - 'us-central1-docker.pkg.dev/$PROJECT_ID/myrepo/myapp:$SHORT_SHA'
      - '--format=json'
    id: 'scan'
    waitFor: ['build']

  # Push image
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '--all-tags',
           'us-central1-docker.pkg.dev/$PROJECT_ID/myrepo/myapp']
    id: 'push'
    waitFor: ['scan']

  # Deploy to GKE
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    args:
      - 'gke-deploy'
      - 'run'
      - '--filename=k8s/'
      - '--image=us-central1-docker.pkg.dev/$PROJECT_ID/myrepo/myapp:$SHORT_SHA'
      - '--cluster=prod-cluster'
      - '--location=us-central1'
      - '--namespace=production'
    id: 'deploy'
    waitFor: ['push']

images:
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/myrepo/myapp:$SHORT_SHA'
  - 'us-central1-docker.pkg.dev/$PROJECT_ID/myrepo/myapp:latest'

options:
  defaultLogsBucketBehavior: REGIONAL_USER_OWNED_BUCKET
  machineType: E2_HIGHCPU_8
  logging: CLOUD_LOGGING_ONLY
```

### Artifact Registry

```bash
# Create registry
gcloud artifacts repositories create myrepo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Docker images for myapp"

# Configure Docker auth
gcloud auth configure-docker us-central1-docker.pkg.dev

# Push image
docker push us-central1-docker.pkg.dev/my-project/myrepo/myapp:latest

# List images
gcloud artifacts docker images list \
  us-central1-docker.pkg.dev/my-project/myrepo

# Set cleanup policy
gcloud artifacts repositories set-cleanup-policies myrepo \
  --location=us-central1 \
  --policy=cleanup-policy.json
```

---

## Operations

### Cloud Monitoring

```bash
# Create uptime check
gcloud monitoring uptime create \
  --display-name="MyApp Health Check" \
  --resource-type=uptime_url \
  --host=myapp.example.com \
  --path=/health \
  --period=60

# Create alerting policy (JSON)
gcloud alpha monitoring policies create --policy-from-file=alert-policy.json
```

### Cloud Logging Queries

```sql
-- Find errors in GKE logs
resource.type="k8s_container"
resource.labels.cluster_name="prod-cluster"
severity>=ERROR
timestamp>="2024-01-01T00:00:00Z"

-- Slow HTTP requests (> 2 seconds)
resource.type="http_load_balancer"
httpRequest.latency>"2s"

-- Service account usage
protoPayload.authenticationInfo.serviceAccountEmail="myapp-sa@my-project.iam.gserviceaccount.com"
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is a GCP Project and what does it represent?**

> A Project is the fundamental organizing unit in GCP. It serves as: billing entity (each project has its own billing), API enablement boundary (APIs are enabled per project), resource isolation boundary (VMs, networks, buckets belong to projects), and IAM boundary. Every resource must belong to exactly one project. You organize projects under folders and folders under an organization.

**Q2: What is the difference between GKE Autopilot and Standard mode?**

> **Standard mode**: You manage worker nodes — choose machine type, disk size, configure node pools, manage node OS patches. Full control, but more operational burden. **Autopilot mode**: GCP manages all node infrastructure — you only deploy workloads. Nodes are provisioned per-pod based on resource requests. You're billed per pod, not per node. Autopilot has stricter security posture (privileged containers disallowed). Choose Autopilot when you want minimal infrastructure management.

**Q3: What is Cloud Pub/Sub?**

> Pub/Sub is a fully managed asynchronous messaging service. Publishers send messages to **topics**; subscribers receive from **subscriptions** (pull or push). Features: at-least-once delivery, message retention (up to 7 days), dead-letter topics, ordering, filtering. Used for decoupling services, event streaming, fanout patterns (one message → multiple subscribers). Equivalent to AWS SNS+SQS or Azure Service Bus.

---

### 🟡 Intermediate

**Q4: Explain Workload Identity in GKE.**

> Workload Identity allows GKE pods to authenticate to Google Cloud APIs as a Google Service Account without key files. A Kubernetes Service Account (KSA) is annotated with a Google Service Account (GSA) email. The GKE metadata server provides OIDC tokens to the pod, which Google's IAM exchanges for GSA credentials. This is the recommended way to give GKE workloads access to GCP services — no secrets to manage or rotate.

**Q5: How does Cloud Run differ from GKE?**

> **Cloud Run** is serverless containers — you provide a container image, Cloud Run handles infrastructure. Scales to zero (no cost when idle), scales horizontally automatically, per-request billing, stateless. **GKE** is managed Kubernetes — full control over pods, deployments, stateful workloads, daemonsets, node configuration, networking, cluster autoscaling. Choose Cloud Run for stateless HTTP services/APIs; GKE for complex microservices, stateful workloads, or when you need Kubernetes-native features.

**Q6: What is VPC Service Controls?**

> VPC Service Controls creates a security perimeter around GCP services, preventing data exfiltration. Even if a service account is compromised, it cannot access data outside the perimeter. You define which projects, users, and networks are inside the perimeter. API calls from outside the perimeter (or to services outside) are blocked. Used for sensitive data compliance (HIPAA, PCI, GDPR).

---

### 🔴 Advanced

**Q7: How would you architect a multi-region, highly available application on GCP?**

> - **Global External Load Balancer**: Anycast IP, routes to nearest healthy backend
> - **GKE clusters** in multiple regions (us-central1, europe-west1, asia-east1)
> - **Cloud Spanner** or **Firestore** for globally consistent, multi-region database
> - **GCS multi-region bucket** for static assets with Cloud CDN
> - **Cloud DNS** with geo-routing as fallback
> - **Cloud Armor** for DDoS protection and WAF at the LB layer
> - **Pub/Sub** for async cross-region event propagation
> - **GitOps** (Config Connector / ArgoCD) for consistent deployments across clusters

**Q8: How do you implement least-privilege IAM in a large GCP organization?**

> - **Organization-level** IAM: only Org Admin role for break-glass accounts
> - **Folder-level** roles: team leads get Folder Admin for their division
> - **Project-level** RBAC: developers get Viewer/Editor on specific projects
> - **Service account least-privilege**: each microservice gets a unique SA with only required roles (Artifact Registry Reader, Cloud SQL Client, etc.)
> - **Workload Identity** instead of SA keys
> - **VPC Service Controls** for data perimeters
> - **IAM Recommender**: ML-based tool suggesting permission reductions based on actual usage

---

## Scenario-Based Questions

### 🔵 Scenario 1: Cloud Run Service Cold Start Latency

```bash
# Problem: Cloud Run service has ~3s cold starts

# Solutions:
# 1. Set minimum instances
gcloud run services update myapp \
  --min-instances=1 \
  --region=us-central1
# Keep at least 1 instance warm at all times

# 2. Optimize container startup
# - Defer expensive initialization (lazy loading)
# - Use multi-stage builds for smaller images
# - Pre-build JVM classes (JVM languages)

# 3. Use CPU always-on during startup
gcloud run services update myapp \
  --cpu-boost \
  --region=us-central1
# Extra CPU during startup phase

# 4. Use startup probes
# Cloud Run v2 supports startup probes

# Monitoring cold starts
gcloud logging read 'resource.type="cloud_run_revision" \
  textPayload:"Cold start"' \
  --freshness=1h \
  --format=json
```

---

## Cheat Sheet

```bash
# ─── AUTH ─────────────────────────────────────────────
gcloud auth login
gcloud auth application-default login
gcloud config set project my-project
gcloud config list

# ─── COMPUTE ──────────────────────────────────────────
gcloud compute instances list
gcloud compute instances create myvm \
  --zone=us-central1-a --machine-type=n2-standard-4
gcloud compute ssh myvm --zone=us-central1-a
gcloud compute instances stop/start/delete myvm

# ─── GKE ──────────────────────────────────────────────
gcloud container clusters list
gcloud container clusters get-credentials cluster-name --region=us-central1
gcloud container clusters resize cluster-name --num-nodes=5 --node-pool=default-pool

# ─── CLOUD RUN ────────────────────────────────────────
gcloud run services list --region=us-central1
gcloud run services describe myapp --region=us-central1
gcloud run revisions list --service=myapp --region=us-central1

# ─── STORAGE ──────────────────────────────────────────
gcloud storage ls gs://my-bucket/
gcloud storage cp file.txt gs://my-bucket/
gcloud storage rsync ./dir gs://my-bucket/dir --recursive

# ─── IAM ──────────────────────────────────────────────
gcloud projects get-iam-policy my-project
gcloud iam service-accounts list
gcloud iam service-accounts keys list \
  --iam-account=sa@my-project.iam.gserviceaccount.com

# ─── LOGS ─────────────────────────────────────────────
gcloud logging read 'severity>=ERROR' --freshness=1h
gcloud logging tail 'resource.type="gke_container"'
```

---

> **Cross-links:** [← Azure](./Azure.md) | [CI/CD →](./CICD.md) | [Kubernetes (GKE) →](./Kubernetes.md) | [Monitoring →](./Monitoring.md)
