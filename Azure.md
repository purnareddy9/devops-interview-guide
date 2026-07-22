# 🔷 Azure — DevOps Interview Guide

> [← AWS](./AWS.md) | [Main Index](./README.md) | [GCP →](./GCP.md)

---

## Table of Contents

1. [Azure Core Concepts](#azure-core-concepts)
2. [Identity & Access (AAD, RBAC)](#identity--access)
3. [Networking (VNet, NSG, Load Balancer)](#networking)
4. [Compute (VMs, AKS, Functions)](#compute)
5. [Storage](#storage)
6. [Azure DevOps](#azure-devops)
7. [ARM Templates & Bicep](#arm-templates--bicep)
8. [Monitoring (Azure Monitor, Log Analytics)](#monitoring)
9. [Interview Questions](#interview-questions)
10. [Scenario-Based Questions](#scenario-based-questions)
11. [Cheat Sheet](#cheat-sheet)

---

## Azure Core Concepts

### Azure Global Infrastructure

```
Geography (Americas, Europe, Asia Pacific)
└── Region (East US, West Europe, Southeast Asia)
    └── Availability Zone (1, 2, 3)
        └── Data Center(s)

Azure Resource Hierarchy:
Management Group
└── Subscription
    └── Resource Group
        └── Resources (VMs, Storage, Networks, etc.)
```

### Resource Organization

| Level | Purpose |
|-------|---------|
| **Management Group** | Apply policies/RBAC across multiple subscriptions |
| **Subscription** | Billing boundary; logical isolation of resources |
| **Resource Group** | Logical container for related resources; lifecycle management |
| **Resources** | Individual services (VM, VNet, Storage Account, etc.) |

---

## Identity & Access

### Azure Active Directory (Entra ID)

```
Azure AD Tenant
├── Users (internal + external/B2B)
├── Groups
├── Service Principals (apps, CI/CD identities)
├── Managed Identities (Azure resource identities)
│   ├── System-assigned (tied to resource lifecycle)
│   └── User-assigned (standalone, reusable)
├── App Registrations
└── Enterprise Applications
```

### RBAC

| Role | Scope | Permissions |
|------|-------|------------|
| **Owner** | Any | Full access + manage access |
| **Contributor** | Any | Full access, cannot manage access |
| **Reader** | Any | View resources only |
| **User Access Admin** | Any | Manage access (not resources) |
| Custom Role | Subscription/RG/Resource | Granular permissions |

```bash
# Assign RBAC role
az role assignment create \
  --assignee "user@example.com" \
  --role "Contributor" \
  --scope "/subscriptions/{subscription-id}/resourceGroups/myapp-rg"

# List role assignments
az role assignment list --resource-group myapp-rg -o table

# Create custom role
az role definition create --role-definition @custom-role.json

# Get current user
az account show
az ad signed-in-user show
```

### Managed Identity for CI/CD

```bash
# Create user-assigned managed identity
az identity create \
  --name github-actions-identity \
  --resource-group devops-rg

# Federated credentials for GitHub Actions (OIDC)
az identity federated-credential create \
  --name github-actions-fed \
  --identity-name github-actions-identity \
  --resource-group devops-rg \
  --issuer "https://token.actions.githubusercontent.com" \
  --subject "repo:my-org/my-repo:ref:refs/heads/main" \
  --audiences "api://AzureADTokenExchange"

# Assign role to managed identity
az role assignment create \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/myapp-rg"
```

---

## Networking

### Virtual Network Architecture

```
VNet: 10.0.0.0/16
├── Subnet: web-tier      10.0.1.0/24  → App Service / VMs
├── Subnet: app-tier      10.0.2.0/24  → Internal services
├── Subnet: data-tier     10.0.3.0/24  → Databases (Private Endpoints)
├── Subnet: AzureBastionSubnet  10.0.4.0/27  → Bastion Host
└── Subnet: AKS           10.0.5.0/22  → AKS node pools

VNet Peering → Connect to other VNets (cross-region possible)
VPN Gateway  → Connect to on-premises (IPSec VPN)
ExpressRoute → Private dedicated link to Azure
```

### NSG (Network Security Group)

```bash
# Create NSG
az network nsg create --name web-nsg --resource-group myapp-rg

# Add inbound rule — allow HTTPS
az network nsg rule create \
  --nsg-name web-nsg \
  --resource-group myapp-rg \
  --name allow-https \
  --priority 100 \
  --protocol tcp \
  --destination-port-range 443 \
  --direction Inbound \
  --access Allow

# Deny all other inbound
az network nsg rule create \
  --nsg-name web-nsg \
  --resource-group myapp-rg \
  --name deny-all \
  --priority 4000 \
  --direction Inbound \
  --access Deny

# Associate NSG with subnet
az network vnet subnet update \
  --vnet-name myapp-vnet \
  --name web-tier \
  --resource-group myapp-rg \
  --nsg web-nsg
```

### Azure Load Balancer Types

| Type | Layer | Use Case |
|------|-------|---------|
| **Application Gateway** | L7 | Web apps, WAF, URL routing, SSL offload |
| **Azure Load Balancer** | L4 | TCP/UDP load balancing, internal/external |
| **Azure Front Door** | L7 (global) | Global HTTP(S), CDN, WAF, geo-routing |
| **Traffic Manager** | DNS | DNS-based global routing (failover, performance) |
| **API Management** | L7 | API gateway, rate limiting, policies |

---

## Compute

### Azure Kubernetes Service (AKS)

```bash
# Create AKS cluster
az aks create \
  --resource-group myapp-rg \
  --name prod-aks \
  --location eastus \
  --kubernetes-version 1.28 \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --enable-managed-identity \
  --enable-auto-upgrade \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --zones 1 2 3 \
  --enable-azure-monitor-metrics

# Get credentials
az aks get-credentials --resource-group myapp-rg --name prod-aks

# Add node pool
az aks nodepool add \
  --cluster-name prod-aks \
  --resource-group myapp-rg \
  --name gpupool \
  --node-vm-size Standard_NC6 \
  --node-count 0 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 5 \
  --node-taints "gpu=true:NoSchedule"

# Upgrade cluster
az aks upgrade --resource-group myapp-rg --name prod-aks --kubernetes-version 1.29
```

### Azure Container Registry (ACR)

```bash
# Create ACR
az acr create \
  --name mycompanyregistry \
  --resource-group devops-rg \
  --sku Premium \
  --admin-enabled false

# Build and push from source (ACR Tasks)
az acr build \
  --registry mycompanyregistry \
  --image myapp:$(git rev-parse --short HEAD) \
  .

# Integrate ACR with AKS (no secrets needed)
az aks update \
  --resource-group myapp-rg \
  --name prod-aks \
  --attach-acr mycompanyregistry

# Import image from Docker Hub
az acr import \
  --name mycompanyregistry \
  --source docker.io/library/nginx:latest \
  --image nginx:latest
```

### Azure Functions

```python
# function_app.py
import azure.functions as func
import logging

app = func.FunctionApp(http_auth_level=func.AuthLevel.FUNCTION)

@app.route(route="process")
def process_message(req: func.HttpRequest) -> func.HttpResponse:
    logging.info('Processing request')
    name = req.params.get('name') or req.get_json().get('name', 'World')
    return func.HttpResponse(f"Hello, {name}!")

@app.queue_trigger(arg_name="msg", queue_name="myqueue",
                   connection="STORAGE_CONNECTION_STRING")
def queue_processor(msg: func.QueueMessage) -> None:
    logging.info(f"Processing message: {msg.get_body().decode('utf-8')}")
```

---

## Storage

### Storage Account Types

| Type | Use Case | Redundancy Options |
|------|---------|-------------------|
| **Blob Storage** | Unstructured data, media, backups | LRS, ZRS, GRS, GZRS |
| **Azure Files** | Shared file system (SMB/NFS) | LRS, ZRS, GRS |
| **Table Storage** | NoSQL key-value | LRS, ZRS, GRS |
| **Queue Storage** | Message queuing | LRS, ZRS, GRS |
| **Data Lake Gen2** | Big data analytics | LRS, ZRS, GRS |

```bash
# Create storage account
az storage account create \
  --name myappstorageaccount \
  --resource-group myapp-rg \
  --sku Standard_ZRS \
  --kind StorageV2 \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false \
  --enable-hierarchical-namespace true

# Upload blob
az storage blob upload \
  --account-name myappstorageaccount \
  --container-name mycontainer \
  --name myfile.txt \
  --file ./local-file.txt \
  --auth-mode login

# Generate SAS token
az storage blob generate-sas \
  --account-name myappstorageaccount \
  --container-name mycontainer \
  --name myfile.txt \
  --permissions r \
  --expiry "2024-12-31T00:00:00Z" \
  --auth-mode login \
  --as-user
```

---

## Azure DevOps

### Pipelines (azure-pipelines.yml)

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include: [main, develop, release/*]
  paths:
    exclude: [docs/, '*.md']

pr:
  branches:
    include: [main]

variables:
  - group: production-secrets     # Variable group from Azure DevOps library
  - name: imageRepository
    value: 'myapp'
  - name: containerRegistry
    value: 'mycompanyregistry.azurecr.io'
  - name: tag
    value: '$(Build.BuildId)'

pool:
  vmImage: ubuntu-latest

stages:
- stage: Build
  displayName: 'Build and Test'
  jobs:
  - job: BuildTest
    steps:
    - task: NodeTool@0
      inputs:
        versionSpec: '20.x'

    - script: |
        npm ci
        npm run lint
        npm run test
      displayName: 'Install, Lint, Test'

    - task: PublishTestResults@2
      inputs:
        testResultsFormat: JUnit
        testResultsFiles: '**/test-results.xml'
      condition: succeededOrFailed()

    - task: PublishCodeCoverageResults@2
      inputs:
        summaryFileLocation: '**/coverage/cobertura-coverage.xml'

- stage: BuildPush
  displayName: 'Build and Push Image'
  dependsOn: Build
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - job: Docker
    steps:
    - task: Docker@2
      displayName: Build and push image
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: Dockerfile
        containerRegistry: $(containerRegistryServiceConnection)
        tags: |
          $(tag)
          latest

    - task: AzureContainerApps@1
      displayName: 'Trivy Security Scan'
      inputs:
        azureSubscription: $(azureServiceConnection)
        imageToDeploy: '$(containerRegistry)/$(imageRepository):$(tag)'

- stage: DeployStaging
  displayName: 'Deploy to Staging'
  dependsOn: BuildPush
  jobs:
  - deployment: Deploy
    environment: staging
    strategy:
      runOnce:
        deploy:
          steps:
          - task: HelmDeploy@0
            inputs:
              connectionType: Azure Resource Manager
              azureSubscriptionEndpoint: $(azureServiceConnection)
              azureResourceGroup: myapp-staging-rg
              kubernetesCluster: staging-aks
              namespace: staging
              command: upgrade
              chartType: FilePath
              chartPath: ./helm/myapp
              releaseName: myapp
              overrideValues: 'image.tag=$(tag)'
              waitForExecution: true

- stage: DeployProd
  displayName: 'Deploy to Production'
  dependsOn: DeployStaging
  jobs:
  - deployment: Deploy
    environment: production          # Requires approval in Azure DevOps
    strategy:
      runOnce:
        deploy:
          steps:
          - task: HelmDeploy@0
            inputs:
              connectionType: Azure Resource Manager
              azureSubscriptionEndpoint: $(azureServiceConnection)
              azureResourceGroup: myapp-prod-rg
              kubernetesCluster: prod-aks
              namespace: production
              command: upgrade
              chartType: FilePath
              chartPath: ./helm/myapp
              releaseName: myapp
              overrideValues: 'image.tag=$(tag),replicaCount=5'
              waitForExecution: true
```

---

## ARM Templates & Bicep

### Bicep Example (Modern IaC for Azure)

```bicep
// main.bicep
@description('Environment name')
@allowed(['dev', 'staging', 'prod'])
param environment string

@description('Azure region')
param location string = resourceGroup().location

param appServicePlanSku object = {
  name: environment == 'prod' ? 'P2v3' : 'B1'
  tier: environment == 'prod' ? 'PremiumV3' : 'Basic'
}

var appName = 'myapp-${environment}-${uniqueString(resourceGroup().id)}'

// App Service Plan
resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: '${appName}-plan'
  location: location
  sku: appServicePlanSku
  kind: 'linux'
  properties: {
    reserved: true       // Required for Linux
  }
}

// App Service
resource webApp 'Microsoft.Web/sites@2022-09-01' = {
  name: appName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'NODE|20-lts'
      minTlsVersion: '1.2'
      ftpsState: 'Disabled'
      appSettings: [
        {
          name: 'NODE_ENV'
          value: environment
        }
        {
          name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: appInsights.properties.ConnectionString
        }
      ]
    }
    httpsOnly: true
  }
  identity: {
    type: 'SystemAssigned'        // Managed identity
  }
}

// Application Insights
resource appInsights 'Microsoft.Insights/components@2020-02-02' = {
  name: '${appName}-insights'
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
  }
}

// Outputs
output webAppUrl string = 'https://${webApp.properties.defaultHostName}'
output webAppPrincipalId string = webApp.identity.principalId
```

```bash
# Deploy Bicep
az deployment group create \
  --resource-group myapp-rg \
  --template-file main.bicep \
  --parameters environment=prod

# What-if (like terraform plan)
az deployment group what-if \
  --resource-group myapp-rg \
  --template-file main.bicep \
  --parameters environment=prod
```

---

## Monitoring

### Azure Monitor Setup

```bash
# Create Log Analytics Workspace
az monitor log-analytics workspace create \
  --resource-group myapp-rg \
  --workspace-name myapp-logs \
  --sku PerGB2018

# Enable diagnostic settings for AKS
az monitor diagnostic-settings create \
  --resource $(az aks show -g myapp-rg -n prod-aks --query id -o tsv) \
  --name aks-diagnostics \
  --workspace $(az monitor log-analytics workspace show -g myapp-rg -n myapp-logs --query id -o tsv) \
  --logs '[{"category":"kube-audit","enabled":true},{"category":"kube-controller-manager","enabled":true}]'

# Create alert rule
az monitor metrics alert create \
  --name "High CPU Alert" \
  --resource-group myapp-rg \
  --scopes $(az aks show -g myapp-rg -n prod-aks --query id -o tsv) \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action $(az monitor action-group show -g myapp-rg -n my-actions --query id -o tsv)
```

### KQL Queries (Kusto Query Language)

```kusto
// Find failed requests in last hour
requests
| where timestamp > ago(1h)
| where success == false
| summarize count() by resultCode, cloud_RoleName
| order by count_ desc

// Pod restart count
KubePodInventory
| where Namespace == "production"
| where PodRestartCount > 0
| summarize max(PodRestartCount) by Name, Namespace
| order by max_PodRestartCount desc

// Error rate over time
ContainerLog
| where LogEntry contains "ERROR"
| summarize ErrorCount = count() by bin(TimeGenerated, 5m)
| render timechart
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is the difference between Azure Active Directory and on-premises Active Directory?**

> **Azure AD (Entra ID)** is a cloud-based identity service — manages users, groups, OAuth/OIDC-based application access, conditional access policies, MFA. It communicates via HTTPS/REST. **On-premises AD** is domain services using LDAP/Kerberos/NTLM for traditional workloads. They can be synchronized via **Azure AD Connect**. Azure AD is not a direct replacement — it lacks Group Policy, LDAP, traditional domain join (Azure AD DS is needed for those).

**Q2: What is a Resource Group in Azure?**

> A Resource Group is a logical container for Azure resources. It serves as a **management boundary**: you can apply policies, RBAC, tags, and locks at the RG level. Resources in an RG share the same lifecycle — when you delete the RG, all resources inside are deleted. Resources can only belong to one RG. Best practice: group resources that share the same lifecycle and belong to the same application/environment.

**Q3: What is the difference between Azure Blob Storage tiers?**

> - **Hot**: Frequently accessed data — highest storage cost, lowest access cost
> - **Cool**: Infrequently accessed (30+ days) — lower storage cost, higher access cost
> - **Cold**: Rarely accessed (90+ days) — even lower storage cost
> - **Archive**: Rarely accessed (180+ days) — lowest storage cost, but requires rehydration (hours) before access
> Lifecycle management policies automatically move blobs between tiers.

---

### 🟡 Intermediate

**Q4: What is a Managed Identity and why is it preferred over service principals with keys?**

> A Managed Identity is an Azure AD identity automatically managed by Azure for an Azure resource (VM, AKS pod, Function, etc.). The platform handles credential rotation automatically — there are **no secrets to store or rotate**. Types: System-assigned (tied to resource, deleted with it) and User-assigned (standalone, can be shared). This eliminates the risk of secret leakage and the toil of secret rotation.

**Q5: Explain VNet Peering vs Azure VPN Gateway.**

> **VNet Peering**: Direct, private connection between two VNets over Microsoft backbone — low latency, no bandwidth limitations, no gateway required. Can be regional or global (cross-region). Not transitive by default (use Route Server or NVA for hub-spoke transitivity). **VPN Gateway**: IPSec tunnel connecting Azure VNet to on-premises or another cloud — useful for hybrid connectivity, up to 10 Gbps (ExpressRoute for higher throughput).

**Q6: What is Azure Kubernetes Service (AKS) and what does Azure manage vs what you manage?**

> AKS is a managed Kubernetes service. **Azure manages**: control plane (API server, etcd, scheduler) — patching, scaling, high availability, certificates. **You manage**: worker node pools, application deployments, networking (CNI plugin), RBAC, monitoring, ingress controllers, and node OS patching (though there are auto-upgrade options).

---

### 🔴 Advanced

**Q7: How would you design a zero-downtime deployment pipeline for AKS using Azure DevOps?**

> 1. **Azure Container Registry** with geo-replication and vulnerability scanning
> 2. **Azure DevOps pipeline** stages: build → test → scan → push → deploy staging → approval gate → deploy prod
> 3. **AKS rolling update** with `maxUnavailable: 0, maxSurge: 1` + proper readiness probes
> 4. **Azure Traffic Manager** or **Front Door** for global traffic shifting (blue-green)
> 5. **Deployment environments** in Azure DevOps with required approvals for production
> 6. **Helm + Helmfile** for declarative deployment configuration
> 7. **Azure Monitor + alerts** + auto-rollback pipeline trigger on error rate spike

**Q8: How does Azure Policy work and how do you use it for governance?**

> Azure Policy evaluates resources against rules (policy definitions). Effect types: `Deny` (prevent non-compliant resource creation), `Audit` (log non-compliance), `Append` (add required fields), `DeployIfNotExists`/`Modify` (auto-remediate). Policies are assigned at management group/subscription/RG scope. **Policy Initiative** (Policy Set) groups related policies. Use cases: enforce tagging, require encryption, restrict allowed VM sizes, mandate private endpoints, require Azure Defender.

---

## Scenario-Based Questions

### 🔵 Scenario 1: AKS Pod Can't Pull Image

```bash
# 1. Check pod status
kubectl describe pod failing-pod -n production
# Look for: "Failed to pull image... unauthorized"

# 2. Verify AKS-ACR integration
az aks check-acr \
  --resource-group myapp-rg \
  --name prod-aks \
  --acr mycompanyregistry.azurecr.io

# 3. Re-attach ACR if needed
az aks update \
  --resource-group myapp-rg \
  --name prod-aks \
  --attach-acr mycompanyregistry

# 4. Check if image exists in ACR
az acr repository show-tags \
  --name mycompanyregistry \
  --repository myapp

# 5. Check managed identity role assignment
KUBELET_IDENTITY=$(az aks show -g myapp-rg -n prod-aks \
  --query identityProfile.kubeletidentity.clientId -o tsv)
az role assignment list --assignee $KUBELET_IDENTITY \
  --scope $(az acr show -n mycompanyregistry --query id -o tsv)
```

---

## Cheat Sheet

```bash
# ─── AUTH ─────────────────────────────────────────────
az login
az account list
az account set --subscription "My Subscription"
az ad signed-in-user show

# ─── RESOURCE GROUPS ──────────────────────────────────
az group create --name myapp-rg --location eastus
az group list -o table
az group delete --name myapp-rg --yes --no-wait

# ─── AKS ──────────────────────────────────────────────
az aks list -o table
az aks get-credentials -g myapp-rg -n prod-aks
az aks show -g myapp-rg -n prod-aks
az aks nodepool list -g myapp-rg --cluster-name prod-aks

# ─── ACR ──────────────────────────────────────────────
az acr list -o table
az acr login --name mycompanyregistry
az acr repository list --name mycompanyregistry

# ─── STORAGE ──────────────────────────────────────────
az storage account list -g myapp-rg -o table
az storage container list --account-name myaccount
az storage blob list --container-name mycontainer --account-name myaccount

# ─── MONITORING ───────────────────────────────────────
az monitor log-analytics query \
  --workspace <workspace-id> \
  --analytics-query "ContainerLog | limit 20"
az monitor metrics list --resource <resource-id>

# ─── BICEP ────────────────────────────────────────────
az bicep build --file main.bicep          # Compile to ARM
az deployment group create -g myapp-rg --template-file main.bicep
az deployment group what-if -g myapp-rg --template-file main.bicep
```

---

> **Cross-links:** [← AWS](./AWS.md) | [GCP →](./GCP.md) | [CI/CD (Azure DevOps) →](./CICD.md) | [Security (Azure security) →](./Security.md) | [Kubernetes (AKS) →](./Kubernetes.md)
