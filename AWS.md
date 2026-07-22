---
layout: default
title: "☁️ AWS — DevOps Interview Guide"
render_with_liquid: false
---

# ☁️ AWS — DevOps Interview Guide

> [← Ansible](./Ansible.md) | [Main Index](./README.md) | [Azure →](./Azure.md)

---

## Table of Contents

1. [Core Services Overview](#core-services-overview)
2. [VPC & Networking](#vpc--networking)
3. [Compute (EC2, ECS, EKS, Lambda)](#compute)
4. [Storage (S3, EBS, EFS)](#storage)
5. [IAM & Security](#iam--security)
6. [Databases (RDS, DynamoDB, ElastiCache)](#databases)
7. [CloudFormation & CDK](#cloudformation--cdk)
8. [Monitoring & Logging](#monitoring--logging)
9. [Interview Questions](#interview-questions)
10. [Scenario-Based Questions](#scenario-based-questions)
11. [Hands-On Labs](#hands-on-labs)
12. [Cheat Sheet](#cheat-sheet)

---

## Core Services Overview

### AWS Global Infrastructure

```
Region (us-east-1)
├── Availability Zone (us-east-1a)
│   └── Data center(s)
├── Availability Zone (us-east-1b)
└── Availability Zone (us-east-1c)

Edge Location (CloudFront CDN PoPs — 400+)
Local Zone (low-latency extension of Region)
Wavelength Zone (5G networks)
```

### Service Categories

| Category | Key Services |
|----------|-------------|
| Compute | EC2, ECS, EKS, Lambda, Fargate, Batch |
| Storage | S3, EBS, EFS, FSx, Glacier |
| Database | RDS, Aurora, DynamoDB, ElastiCache, Redshift |
| Networking | VPC, Route53, CloudFront, ALB/NLB, Direct Connect |
| Security | IAM, KMS, Secrets Manager, WAF, Shield, GuardDuty |
| DevOps | CodePipeline, CodeBuild, CodeDeploy, CodeCommit, ECR |
| Monitoring | CloudWatch, CloudTrail, X-Ray, Config |
| Containers | ECR, ECS, EKS, App Mesh |
| Messaging | SQS, SNS, EventBridge, Kinesis |
| IaC | CloudFormation, CDK, Service Catalog |

---

## VPC & Networking

### Well-Architected VPC Design

```
┌──────────────── VPC: 10.0.0.0/16 ───────────────────┐
│                                                        │
│  ┌── AZ-1a ─────────────────────────────────────┐    │
│  │  Public:  10.0.0.0/24  (ALB, NAT GW, Bastion) │    │
│  │  Private: 10.0.10.0/24 (App servers, EKS)      │    │
│  │  DB:      10.0.20.0/24 (RDS, ElastiCache)      │    │
│  └───────────────────────────────────────────────┘    │
│                                                        │
│  ┌── AZ-1b ─────────────────────────────────────┐    │
│  │  Public:  10.0.1.0/24                          │    │
│  │  Private: 10.0.11.0/24                         │    │
│  │  DB:      10.0.21.0/24                         │    │
│  └───────────────────────────────────────────────┘    │
│                                                        │
│  Internet Gateway ──> Public subnets                   │
│  NAT Gateway ──────> Private subnets (outbound only)   │
│  VPC Endpoints ────> S3, DynamoDB (no NAT needed)      │
└───────────────────────────────────────────────────────┘
```

### VPC Key Components

| Component | Purpose |
|-----------|---------|
| **Internet Gateway** | Allow public subnet traffic in/out of internet |
| **NAT Gateway** | Allow private subnet outbound internet (one-way) |
| **Route Table** | Rules directing where traffic goes |
| **Security Group** | Stateful firewall at instance/ENI level |
| **NACL** | Stateless firewall at subnet level |
| **VPC Endpoint** | Private connection to AWS services (no internet) |
| **VPC Peering** | Connect two VPCs privately (non-transitive) |
| **Transit Gateway** | Hub-and-spoke VPC interconnect |
| **Direct Connect** | Dedicated physical link to AWS |

### Terraform: VPC + Subnets

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "prod-vpc" }
}

# VPC Flow Logs
resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_log.arn
  log_destination = aws_cloudwatch_log_group.flow_log.arn
}
```

---

## Compute

### EC2 Key Concepts

```bash
# Launch EC2 with userdata
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t3.medium \
  --key-name my-key \
  --security-group-ids sg-12345678 \
  --subnet-id subnet-12345678 \
  --iam-instance-profile Name=MyRole \
  --user-data file://user_data.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web01}]'

# Instance metadata (from inside EC2)
curl http://169.254.169.254/latest/meta-data/instance-id
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl http://169.254.169.254/latest/dynamic/instance-identity/document
```

### EC2 Instance Types Quick Reference

| Family | Optimized For | Examples |
|--------|--------------|---------|
| t3/t4g | Burstable general purpose | Web servers, dev |
| m6i/m7g | Balanced compute/memory | App servers |
| c6i/c7g | Compute optimized | CPU-heavy workloads |
| r6i/r7g | Memory optimized | Databases, caches |
| i3/i4i | Storage optimized | Databases, Elasticsearch |
| p3/p4 | GPU | ML training |
| inf1/inf2 | ML inference | Real-time predictions |

### Auto Scaling Group

```hcl
resource "aws_autoscaling_group" "web" {
  name                = "web-asg"
  vpc_zone_identifier = module.vpc.private_subnets
  min_size            = 2
  max_size            = 20
  desired_capacity    = 3

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  target_group_arns = [aws_lb_target_group.web.arn]

  health_check_type         = "ELB"
  health_check_grace_period = 300

  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 90
    }
  }

  tag {
    key                 = "Name"
    value               = "web-asg-instance"
    propagate_at_launch = true
  }
}

resource "aws_autoscaling_policy" "scale_out" {
  name                   = "scale-out"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 70.0
  }
}
```

### EKS

```bash
# Create EKS cluster
eksctl create cluster \
  --name prod-cluster \
  --region us-east-1 \
  --version 1.28 \
  --nodegroup-name standard-workers \
  --node-type m5.large \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 10 \
  --managed

# Update kubeconfig
aws eks update-kubeconfig --region us-east-1 --name prod-cluster

# Enable IRSA (IAM Roles for Service Accounts)
eksctl utils associate-iam-oidc-provider \
  --cluster prod-cluster --approve

eksctl create iamserviceaccount \
  --name s3-reader \
  --namespace default \
  --cluster prod-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

### Lambda

```python
# lambda_function.py
import json
import boto3
import os

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table(os.environ['TABLE_NAME'])

def lambda_handler(event, context):
    """Process SQS messages and store in DynamoDB."""
    for record in event['Records']:
        body = json.loads(record['body'])

        table.put_item(Item={
            'id': record['messageId'],
            'data': body,
            'timestamp': context.aws_request_id
        })

    return {
        'statusCode': 200,
        'body': json.dumps(f"Processed {len(event['Records'])} records")
    }
```

```hcl
# Lambda Terraform
resource "aws_lambda_function" "processor" {
  function_name = "message-processor"
  role          = aws_iam_role.lambda.arn
  handler       = "lambda_function.lambda_handler"
  runtime       = "python3.11"
  timeout       = 30
  memory_size   = 256

  filename         = data.archive_file.lambda.output_path
  source_code_hash = data.archive_file.lambda.output_base64sha256

  environment {
    variables = {
      TABLE_NAME = aws_dynamodb_table.messages.name
    }
  }

  tracing_config {
    mode = "Active"   # X-Ray tracing
  }

  vpc_config {
    subnet_ids         = module.vpc.private_subnets
    security_group_ids = [aws_security_group.lambda.id]
  }
}

resource "aws_lambda_event_source_mapping" "sqs" {
  event_source_arn = aws_sqs_queue.messages.arn
  function_name    = aws_lambda_function.processor.arn
  batch_size       = 10
}
```

---

## Storage

### S3

```bash
# CLI operations
aws s3 ls s3://my-bucket/prefix/
aws s3 cp file.txt s3://my-bucket/
aws s3 sync ./local-dir s3://my-bucket/backup/ --delete
aws s3 presign s3://my-bucket/private-file.pdf --expires-in 3600

# Multipart upload (large files)
aws s3 cp large-file.zip s3://my-bucket/ --multipart-threshold 100MB

# S3 lifecycle policy
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json
```

```hcl
# S3 bucket with security best practices
resource "aws_s3_bucket" "data" {
  bucket = "my-data-${var.account_id}"
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true
  }
}

resource "aws_s3_bucket_public_access_block" "data" {
  bucket                  = aws_s3_bucket.data.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

---

## IAM & Security

### IAM Best Practices

```
Root Account        → Only for billing/account-level; MFA mandatory; no access keys
IAM Users           → For humans (or use SSO/federated identity)
IAM Roles           → For services, EC2, Lambda, cross-account access
IAM Groups          → Assign policies to groups, add users to groups
Service Control Policies (SCP) → Org-level guardrails across all accounts
```

### IAM Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadOnSpecificBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-app-bucket",
        "arn:aws:s3:::my-app-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        },
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    },
    {
      "Sid": "DenyDeleteActions",
      "Effect": "Deny",
      "Action": "s3:Delete*",
      "Resource": "*"
    }
  ]
}
```

### Cross-Account Access

```hcl
# In the target account — trust the source account role
resource "aws_iam_role" "cross_account" {
  name = "cross-account-deployer"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::SOURCE_ACCOUNT_ID:root" }
      Action    = "sts:AssumeRole"
      Condition = {
        StringEquals = {
          "sts:ExternalId" = var.external_id
        }
      }
    }]
  })
}

# In source account — assume the target role
aws sts assume-role \
  --role-arn arn:aws:iam::TARGET_ACCOUNT:role/cross-account-deployer \
  --role-session-name deploy-session \
  --external-id my-external-id
```

---

## Databases

### RDS

```hcl
resource "aws_db_instance" "postgres" {
  identifier        = "${var.environment}-postgres"
  engine            = "postgres"
  engine_version    = "15.3"
  instance_class    = var.environment == "production" ? "db.r6g.large" : "db.t3.medium"
  allocated_storage = 100
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn

  db_name  = "appdb"
  username = "admin"
  password = random_password.db.result

  multi_az               = var.environment == "production"
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:04:00-sun:05:00"

  deletion_protection       = var.environment == "production"
  skip_final_snapshot       = var.environment != "production"
  final_snapshot_identifier = "${var.environment}-final-snapshot"

  performance_insights_enabled = true

  tags = local.common_tags
}
```

---

## CloudFormation & CDK

### CloudFormation Template Structure

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Web application stack

Parameters:
  Environment:
    Type: String
    AllowedValues: [development, staging, production]
  InstanceType:
    Type: String
    Default: t3.medium

Mappings:
  RegionMap:
    us-east-1:
      AMI: ami-0c55b159cbfafe1f0
    eu-west-1:
      AMI: ami-01f14919ba412de34

Conditions:
  IsProduction: !Equals [!Ref Environment, production]

Resources:
  WebInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: !FindInMap [RegionMap, !Ref AWS::Region, AMI]
      Tags:
        - Key: Name
          Value: !Sub "${Environment}-web"

  WebEIP:
    Type: AWS::EC2::EIP
    Condition: IsProduction
    Properties:
      InstanceId: !Ref WebInstance

Outputs:
  InstanceId:
    Value: !Ref WebInstance
    Export:
      Name: !Sub "${AWS::StackName}-InstanceId"
```

---

## Monitoring & Logging

### CloudWatch

```bash
# Create metric alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU-web-asg" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --dimensions Name=AutoScalingGroupName,Value=web-asg \
  --period 300 \
  --evaluation-periods 3 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456:alerts \
  --statistic Average

# Query CloudWatch Logs Insights
aws logs start-query \
  --log-group-name /aws/lambda/processor \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 100'
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is the difference between a Security Group and a NACL?**

> **Security Group**: stateful instance-level firewall — allows rules only; return traffic automatically allowed; evaluated as a whole (all rules). **NACL**: stateless subnet-level firewall — allows and deny rules; return traffic must be explicitly allowed; rules evaluated in number order (lowest first). In practice: use Security Groups for most cases; NACLs as a coarse subnet-level backstop or for explicit denies.

**Q2: What is the difference between S3 and EBS?**

> **S3** is object storage — flat key-value, accessed over HTTPS, virtually unlimited, multi-region, best for data lakes, backups, static assets, artifacts. **EBS** is block storage — behaves like a hard disk, attached to one EC2 instance (usually), same AZ, best for databases, OS volumes, files that need POSIX semantics.

**Q3: Explain the difference between horizontal and vertical scaling in AWS context.**

> **Vertical scaling**: increase instance size (t3.medium → m5.xlarge) — requires downtime, has limits, single point of failure. **Horizontal scaling**: add more instances behind a load balancer — no downtime, theoretically unlimited, requires stateless application design. AWS Auto Scaling Groups automate horizontal scaling. Most cloud-native architectures prefer horizontal.

---

### 🟡 Intermediate

**Q4: How does IAM Role-based access work for EC2 instances?**

> Attach an **IAM Instance Profile** (wraps an IAM Role) to the EC2 instance. The AWS SDK automatically calls the **Instance Metadata Service** (IMDS) at `169.254.169.254` to get temporary credentials. The credentials rotate automatically (typically every 1 hour). No access keys are stored on the instance. Apps use the SDK's default credential chain, which includes IMDS.

**Q5: What is an Elastic Load Balancer? Compare ALB vs NLB.**

> **ALB (Application Load Balancer)**: Layer 7, HTTP/HTTPS, content-based routing (path, host, header), WebSocket, gRPC, WAF integration, target groups. **NLB (Network Load Balancer)**: Layer 4, TCP/UDP/TLS, ultra-high performance, static IP per AZ, preserves source IP, handles millions of req/s. Use ALB for web apps and APIs; NLB for TCP-level routing, gaming, IoT, or when a static IP is required.

**Q6: Explain VPC Peering vs Transit Gateway.**

> **VPC Peering**: direct, private connection between two VPCs (can be cross-account or cross-region). Non-transitive: if A↔B and B↔C, A cannot reach C via B. Scales poorly with many VPCs. **Transit Gateway**: a managed regional router — attach many VPCs and on-premises networks, central routing tables, transitive routing, simpler management for complex topologies. More expensive but far simpler at scale.

---

### 🔴 Advanced

**Q7: How would you design a multi-region active-active architecture on AWS?**

> - **Route53 latency/geolocation routing** → direct users to nearest region
> - **Global DynamoDB tables** or Aurora Global Database for data replication
> - **S3 Cross-Region Replication** for object data
> - **Global Accelerator** for anycast routing and health-based failover
> - **Stateless application tier** (session in DynamoDB/ElastiCache, not local)
> - **EventBridge** event bus cross-region for async coordination
> - **RTO/RPO targets** drive data replication strategy
> - Consider conflict resolution for multi-master writes (last-writer-wins vs business logic)

**Q8: How do you secure AWS access in a CI/CD pipeline?**

> - **Never use long-lived IAM keys** in CI — use OIDC identity federation
> - **GitHub Actions + OIDC**: GitHub issues JWT tokens; AWS verifies via OIDC provider; pipeline assumes a role with `sts:AssumeRoleWithWebIdentity`
> - Scoped roles: separate role per environment (staging/production) with least-privilege policies
> - **Require MFA** for production role assumption
> - **CloudTrail** to audit all API calls
> - **SCPs** to prevent privilege escalation across the org

---

## Scenario-Based Questions

### 🔵 Scenario 1: RDS Failover

*"Your RDS Multi-AZ primary instance fails. What happens and how do you respond?"*

```
Automatic: RDS detects failure (~1-2 min) → promotes standby
         → updates DNS CNAME of endpoint → new primary online

Application impact:
- Connections in-flight are dropped
- App must retry connection (use exponential backoff in connection pool)
- DNS TTL: set to 5-10s for fast failover

Response steps:
1. Check RDS Events in console (RDS → Events)
2. Monitor CloudWatch: DatabaseConnections, CPUUtilization
3. Verify app reconnected (check app logs, metrics)
4. Post-incident: investigate original failure (check OS/hardware metrics)
5. Test: test failover regularly with `aws rds reboot-db-instance --force-failover`
```

### 🔵 Scenario 2: Cost Spike Investigation

```bash
# 1. Check Cost Explorer — which service/account?
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity DAILY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE

# 2. Common culprits and fixes:
# - EC2: orphaned instances → tag all resources; use AWS Config rules
# - NAT Gateway: high data transfer → add S3/DynamoDB VPC endpoints
# - Data Transfer: cross-AZ traffic → co-locate services in same AZ
# - Snapshots: accumulation → lifecycle policies
# - CloudWatch Logs: verbose logging → filter before ingestion; set retention
# - RDS: right-size via Performance Insights

# 3. Set up billing alerts
aws cloudwatch put-metric-alarm \
  --alarm-name "MonthlyBillingAlert" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --period 86400 \
  --threshold 500 \
  --comparison-operator GreaterThanThreshold
```

---

## Cheat Sheet

```bash
# ─── EC2 ──────────────────────────────────────────────
aws ec2 describe-instances --filters "Name=tag:Env,Values=prod"
aws ec2 start-instances --instance-ids i-1234
aws ec2 stop-instances --instance-ids i-1234
aws ec2 describe-instance-status --instance-ids i-1234

# ─── S3 ───────────────────────────────────────────────
aws s3 ls s3://bucket/prefix/
aws s3 cp src s3://bucket/dst
aws s3 sync ./dir s3://bucket/dir
aws s3api get-bucket-policy --bucket my-bucket

# ─── IAM ──────────────────────────────────────────────
aws iam list-roles
aws iam get-role --role-name MyRole
aws iam list-attached-role-policies --role-name MyRole
aws sts get-caller-identity        # Who am I?
aws sts assume-role --role-arn ARN --role-session-name session

# ─── EKS ──────────────────────────────────────────────
aws eks list-clusters
aws eks update-kubeconfig --name cluster --region us-east-1
aws eks describe-cluster --name cluster

# ─── LOGS ─────────────────────────────────────────────
aws logs describe-log-groups
aws logs tail /aws/lambda/my-function --follow
aws logs filter-log-events --log-group /aws/ecs/myapp --filter-pattern ERROR

# ─── SSM (SSH-less access) ─────────────────────────────
aws ssm start-session --target i-1234567890abcdef0
```

---

> **Cross-links:** [← Ansible](./Ansible.md) | [Azure →](./Azure.md) | [Terraform (AWS provider) →](./Terraform.md) | [Security (AWS security services) →](./Security.md) | [Monitoring (CloudWatch) →](./Monitoring.md)