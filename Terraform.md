---
layout: default
title: "🏗️ Terraform — DevOps Interview Guide"
render_with_liquid: false
---

# 🏗️ Terraform — DevOps Interview Guide

> [← Jenkins](./Jenkins.md) | [Main Index](./README.md) | [Ansible →](./Ansible.md)

---

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [HCL Syntax](#hcl-syntax)
3. [Providers & Resources](#providers--resources)
4. [State Management](#state-management)
5. [Variables & Outputs](#variables--outputs)
6. [Modules](#modules)
7. [Workspaces](#workspaces)
8. [Backends](#backends)
9. [Terraform Cloud & Enterprise](#terraform-cloud--enterprise)
10. [Interview Questions](#interview-questions)
11. [Scenario-Based Questions](#scenario-based-questions)
12. [Hands-On Labs](#hands-on-labs)
13. [Cheat Sheet](#cheat-sheet)

---

## Core Concepts

### Terraform Workflow

```
Write HCL  →  terraform init  →  terraform plan  →  terraform apply  →  terraform destroy
               (init providers)   (preview changes)   (apply changes)    (tear down)
```

### How Terraform Works

```
┌──────────────────────────────────────────────────────┐
│                   .tf Files (desired state)           │
└───────────────────────────┬──────────────────────────┘
                            │
                    terraform plan
                            │ compares against
                            ▼
┌──────────────────────────────────────────────────────┐
│              terraform.tfstate (current state)        │
│               (local file or remote backend)          │
└───────────────────────────┬──────────────────────────┘
                            │ diffs → API calls
                            ▼
┌──────────────────────────────────────────────────────┐
│            Cloud Provider API (AWS/Azure/GCP)         │
└──────────────────────────────────────────────────────┘
```

### Resource Lifecycle

```
Create → Read → Update → Delete
  │        │       │        │
  └────────┴───────┴────────┘
         CRUD operations via provider
```

---

## HCL Syntax

### Building Blocks

```hcl
# ─── TERRAFORM BLOCK ──────────────────────────────────
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"       # >= 5.0.0, < 6.0.0
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
  }

  backend "s3" {
    bucket         = "my-tfstate-bucket"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

# ─── PROVIDER ─────────────────────────────────────────
provider "aws" {
  region  = var.aws_region
  profile = var.aws_profile

  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Environment = var.environment
      Team        = var.team
    }
  }
}

# ─── RESOURCE ─────────────────────────────────────────
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  vpc_security_group_ids = [aws_security_group.web.id]
  subnet_id              = aws_subnet.public.id
  key_name               = var.key_name

  user_data = templatefile("${path.module}/user_data.sh.tpl", {
    app_version = var.app_version
  })

  tags = {
    Name = "${var.environment}-web-${count.index + 1}"
  }

  lifecycle {
    create_before_destroy = true
    ignore_changes        = [ami]        # Don't replace if AMI changes
    prevent_destroy       = true         # Refuse to destroy this resource
  }
}

# ─── DATA SOURCE ──────────────────────────────────────
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# ─── LOCAL VALUES ─────────────────────────────────────
locals {
  name_prefix  = "${var.project}-${var.environment}"
  common_tags  = merge(var.tags, { Environment = var.environment })
  is_prod      = var.environment == "production"
  azs          = slice(data.aws_availability_zones.available.names, 0, 3)
}
```

---

## Providers & Resources

### AWS VPC Module Example (Complete)

```hcl
# vpc.tf — Full VPC with public/private subnets
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = { Name = "${local.name_prefix}-vpc" }
}

resource "aws_subnet" "public" {
  count = length(local.azs)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true

  tags = { Name = "${local.name_prefix}-public-${local.azs[count.index]}" }
}

resource "aws_subnet" "private" {
  count = length(local.azs)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = local.azs[count.index]

  tags = { Name = "${local.name_prefix}-private-${local.azs[count.index]}" }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${local.name_prefix}-igw" }
}

resource "aws_eip" "nat" {
  count  = local.is_prod ? length(local.azs) : 1
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count         = local.is_prod ? length(local.azs) : 1
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  depends_on    = [aws_internet_gateway.main]
  tags          = { Name = "${local.name_prefix}-nat-${count.index}" }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "${local.name_prefix}-public-rt" }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  count  = local.is_prod ? length(local.azs) : 1
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[local.is_prod ? count.index : 0].id
  }

  tags = { Name = "${local.name_prefix}-private-rt-${count.index}" }
}
```

---

## State Management

### State Commands

```bash
# View state
terraform state list                         # All resources
terraform state show aws_instance.web        # Resource detail

# Manipulate state
terraform state mv aws_instance.web aws_instance.app  # Rename
terraform state rm aws_instance.web                    # Remove (unmanage)
terraform import aws_instance.web i-1234567890abcdef0  # Import existing

# Pull/push remote state
terraform state pull > backup.tfstate
terraform state push backup.tfstate

# Refresh (re-read real world into state)
terraform refresh

# Force unlock (if another process crashed)
terraform force-unlock LOCK-ID
```

### State Locking

```hcl
# DynamoDB table for state locking (prevents concurrent applies)
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = { Purpose = "TerraformStateLocking" }
}

resource "aws_s3_bucket" "tfstate" {
  bucket = "my-terraform-state-${var.account_id}"

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

---

## Variables & Outputs

### Variable Types and Validation

```hcl
# variables.tf
variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be: development, staging, or production."
  }
}

variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 2

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 20
    error_message = "Instance count must be between 1 and 20."
  }
}

variable "allowed_ips" {
  description = "CIDR blocks allowed to access the app"
  type        = list(string)
  default     = ["10.0.0.0/8"]
}

variable "tags" {
  description = "Additional resource tags"
  type        = map(string)
  default     = {}
}

variable "database_config" {
  description = "Database configuration"
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
  default = {
    engine         = "postgres"
    engine_version = "15.3"
    instance_class = "db.t3.micro"
    storage_gb     = 20
    multi_az       = false
  }
}

# Sensitive variable — never printed in logs
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}

# outputs.tf
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "IDs of public subnets"
  value       = aws_subnet.public[*].id
}

output "db_endpoint" {
  description = "Database connection endpoint"
  value       = aws_db_instance.main.endpoint
  sensitive   = true     # Won't print in plan/apply output
}
```

### Variable Precedence (lowest → highest)

```
1. Default in variable block
2. terraform.tfvars
3. terraform.tfvars.json
4. *.auto.tfvars (alphabetical order)
5. -var-file flag
6. -var flag
7. TF_VAR_* environment variables
```

---

## Modules

### Module Structure

```
modules/
└── vpc/
    ├── main.tf         # Resources
    ├── variables.tf    # Input variables
    ├── outputs.tf      # Output values
    ├── versions.tf     # Provider requirements
    └── README.md       # Documentation
```

### Calling a Module

```hcl
# root/main.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = local.name_prefix
  cidr = "10.0.0.0/16"

  azs             = local.azs
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = !local.is_prod
  enable_dns_hostnames = true

  tags = local.common_tags
}

# Use module outputs
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "${local.name_prefix}-eks"
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
}
```

### Module Versioning

```hcl
# Local path (development)
source = "./modules/vpc"

# Git repository
source = "git::https://github.com/org/terraform-modules.git//vpc?ref=v2.1.0"

# Terraform Registry (public)
source  = "hashicorp/consul/aws"
version = "0.1.0"

# Private registry
source  = "app.terraform.io/my-org/vpc/aws"
version = "~> 2.0"
```

---

## Workspaces

```bash
# Workspaces = separate state files per workspace
terraform workspace new staging
terraform workspace new production
terraform workspace list
terraform workspace select staging
terraform workspace show      # Current workspace

# Use in code
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "production" ? "t3.large" : "t3.micro"
}

locals {
  env_config = {
    production  = { instance_type = "t3.large",  min_size = 3 }
    staging     = { instance_type = "t3.small",  min_size = 1 }
    development = { instance_type = "t3.micro",  min_size = 1 }
  }
  config = local.env_config[terraform.workspace]
}
```

---

## Backends

### Remote Backend Options

| Backend | State | Locking | Best For |
|---------|-------|---------|---------|
| **local** (default) | Local file | No | Solo dev, learning |
| **S3 + DynamoDB** | S3 | DynamoDB | AWS teams |
| **Azure Blob** | Azure Storage | Blob lease | Azure teams |
| **GCS** | GCS bucket | GCS native | GCP teams |
| **Terraform Cloud** | HCP Terraform | Yes | Teams, enterprises |
| **GitLab** | GitLab managed | Yes | GitLab users |

---

## Interview Questions

### 🟢 Beginner

**Q1: What is Terraform state and why is it important?**

> Terraform state (`terraform.tfstate`) is a JSON file mapping your configuration resources to real infrastructure. It stores IDs, attributes, and metadata. It's important because:
> - Terraform uses it to **detect drift** (real world ≠ config)
> - Enables **incremental updates** (only change what needs changing)
> - Tracks **resource dependencies** for correct ordering
> - Stores **sensitive values** (encrypted in remote backends)
> Without state, Terraform can't know what it manages.

**Q2: What is the difference between `terraform plan` and `terraform apply`?**

> `terraform plan` is a **dry run** — it reads state, queries the provider API, computes a diff, and shows what would change (`+` create, `~` update, `-` destroy). Nothing changes in the real world. `terraform apply` **executes** those changes after prompting for confirmation. In automation, use `terraform plan -out=tfplan` then `terraform apply tfplan` to guarantee what gets applied matches what was reviewed.

**Q3: What is a Terraform provider?**

> A provider is a plugin that handles API communication with a specific platform (AWS, Azure, GCP, GitHub, Datadog, etc.). Providers translate HCL resource blocks into API calls. Each provider has its own authentication configuration. Providers are versioned and downloaded during `terraform init` from the Terraform Registry.

---

### 🟡 Intermediate

**Q4: How do you manage Terraform state in a team?**

> - Use a **remote backend** (S3+DynamoDB, Terraform Cloud) — single source of truth, not a local file committed to Git
> - Enable **state locking** — DynamoDB for S3, preventing concurrent applies that would corrupt state
> - Enable **versioning and encryption** on the state bucket
> - Use **separate state files per environment** (via workspaces or separate backend configs)
> - Never manually edit `tfstate` — use `terraform state` commands

**Q5: Explain `count` vs `for_each` and when to use each.**

> `count` uses an integer — creates N identical resources, indexed by `count.index`. Problem: if you remove item 0, all higher-indexed resources are recreated (order-dependent). `for_each` uses a map or set of strings — creates resources keyed by value, order-independent. Removing one key only removes that resource.
> ```hcl
> # count — risky for lists that change
> resource "aws_subnet" "pub" { count = 3 }
>
> # for_each — safe, stable keys
> resource "aws_subnet" "pub" {
>   for_each = toset(["us-east-1a", "us-east-1b", "us-east-1c"])
>   availability_zone = each.key
> }
> ```

**Q6: What is `terraform taint` (now `terraform apply -replace`)?**

> Forces recreation of a specific resource even if its config hasn't changed. Used when a resource is in a broken state but Terraform doesn't detect it (e.g., corrupted EC2 instance). Modern syntax: `terraform apply -replace="aws_instance.web"`. The old `terraform taint` command was deprecated in Terraform 0.15.2.

---

### 🔴 Advanced

**Q7: How would you refactor Terraform code to use modules without destroying resources?**

```bash
# Problem: moved resources into a module, now Terraform wants to destroy+recreate

# Solution: terraform state mv
# Original address: aws_vpc.main
# New module address: module.network.aws_vpc.main

terraform state mv \
  'aws_vpc.main' \
  'module.network.aws_vpc.main'

terraform state mv \
  'aws_subnet.public[0]' \
  'module.network.aws_subnet.public[0]'

# Verify plan shows no changes
terraform plan
# Should show: No changes. Infrastructure is up-to-date.
```

**Q8: Explain Terraform's dependency graph and how to handle circular dependencies.**

> Terraform builds a **directed acyclic graph (DAG)** of resources based on explicit references (`aws_subnet.main.id`) and `depends_on`. It applies resources in parallel when no dependency exists. Circular dependencies are a configuration error. Fixes:
> - Extract a resource to break the cycle
> - Use `data` sources instead of resource references where possible
> - Use `depends_on` carefully (it's a full dependency, not just ordering)
> - Split into separate Terraform configurations communicating via remote state: `data "terraform_remote_state"`

---

## Scenario-Based Questions

### 🔵 Scenario 1: Production State Corruption

*"Someone ran `terraform apply` with the wrong workspace. Resources were accidentally destroyed. How do you recover?"*

```bash
# 1. STOP: don't run any more terraform commands

# 2. Identify what was destroyed from the plan output in CI logs

# 3. Restore state from S3 versioning
aws s3api list-object-versions \
  --bucket my-tfstate-bucket \
  --prefix production/terraform.tfstate

aws s3api get-object \
  --bucket my-tfstate-bucket \
  --key production/terraform.tfstate \
  --version-id <version-id-before-incident> \
  restored.tfstate

terraform state push restored.tfstate

# 4. Import any resources that still exist but were removed from state
terraform import aws_instance.web i-1234567890abcdef0

# 5. Plan to verify state matches reality
terraform plan   # Should show no changes if import was complete

# 6. For truly deleted resources: re-create via apply

# Prevention:
# - Enable prevent_destroy on critical resources
# - Require PR review for production applies
# - Separate state files per environment
# - Enable S3 versioning + MFA delete on tfstate bucket
```

### 🔵 Scenario 2: Drift Detection

*"An engineer made manual changes in the AWS console. How do you detect and reconcile?"*

```bash
# 1. Check for drift
terraform plan     # Will show changes needed to match config

# 2. Options:
# A) Accept manual changes — import into state and update .tf files
terraform import aws_security_group_rule.ssh sgr-abc123
# Then edit .tf to reflect new config

# B) Reject manual changes — run apply to revert
terraform apply    # Reverts to declared config

# 3. Automated drift detection in CI
# Run terraform plan on a schedule (daily cron in GitHub Actions)
# Alert if plan output is non-empty
terraform plan -detailed-exitcode
# Exit code 0 = no changes, 1 = error, 2 = changes pending
```

---

## Hands-On Labs

### Lab 1: Bootstrap Remote State

```bash
# Step 1: Bootstrap with local state, then migrate
# bootstrap/main.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws" }
  }
}

resource "aws_s3_bucket" "tfstate" {
  bucket = "tfstate-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_versioning" "tfstate" {
  bucket = aws_s3_bucket.tfstate.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_dynamodb_table" "lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  attribute { name = "LockID"; type = "S" }
}

data "aws_caller_identity" "current" {}

# Step 2: Apply with local state
terraform init && terraform apply

# Step 3: Add backend block and re-init to migrate state
# terraform init -migrate-state
```

---

## Cheat Sheet

```bash
# ─── INIT / FORMAT / VALIDATE ─────────────────────────
terraform init                    # Initialize + download providers
terraform init -upgrade           # Upgrade provider versions
terraform fmt -recursive          # Format all .tf files
terraform validate                # Syntax + config validation

# ─── PLAN / APPLY / DESTROY ───────────────────────────
terraform plan
terraform plan -out=tfplan        # Save plan
terraform plan -var-file=prod.tfvars
terraform apply
terraform apply tfplan            # Apply saved plan
terraform apply -auto-approve     # Skip confirmation (CI only)
terraform apply -target=aws_instance.web  # Target single resource
terraform destroy -auto-approve

# ─── STATE ────────────────────────────────────────────
terraform state list
terraform state show <resource>
terraform state mv <src> <dst>
terraform state rm <resource>
terraform import <resource> <id>

# ─── WORKSPACE ────────────────────────────────────────
terraform workspace new dev
terraform workspace select prod
terraform workspace list

# ─── OUTPUT / CONSOLE ─────────────────────────────────
terraform output
terraform output vpc_id
terraform console              # Interactive expression evaluation

# ─── GRAPH ────────────────────────────────────────────
terraform graph | dot -Tsvg > graph.svg
```

---

> **Cross-links:** [← Jenkins](./Jenkins.md) | [Ansible →](./Ansible.md) | [AWS (cloud resources) →](./AWS.md) | [CI/CD (Terraform in pipelines) →](./CICD.md)