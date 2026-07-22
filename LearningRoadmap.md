---
layout: default
title: "🗺️ Learning Roadmap — DevOps Interview Preparation"
render_with_liquid: false
---

# 🗺️ Learning Roadmap — DevOps Interview Preparation

> [← Behavioral](./Behavioral.md) | [Main Index](./README.md) | [Cheat Sheets →](./CheatSheets.md)

---

## Table of Contents

1. [Beginner Path (0–6 months)](#beginner-path)
2. [Mid-Level Path (6–18 months)](#mid-level-path)
3. [Advanced Path (18+ months)](#advanced-path)
4. [Role-Based Roadmaps](#role-based-roadmaps)
5. [Certification Roadmap](#certification-roadmap)
6. [Weekly Study Plan](#weekly-study-plan)
7. [Resources by Topic](#resources-by-topic)
8. [Interview Preparation Timeline](#interview-preparation-timeline)

---

## Beginner Path

### Phase 1: Linux & Networking (Weeks 1–4)

```
Milestone: Can navigate Linux CLI confidently, understand network fundamentals

Week 1-2: Linux Fundamentals
  ✅ File system structure, permissions, users
  ✅ Process management (ps, top, kill)
  ✅ Package management (apt/yum)
  ✅ Shell scripting basics (variables, loops, functions)
  ✅ SSH, file transfer (scp, rsync)
  Hands-on: Set up an Ubuntu VM (VirtualBox or cloud free tier)

Week 3-4: Networking
  ✅ OSI model (focus on L3, L4, L7)
  ✅ IP addressing, subnetting, CIDR
  ✅ DNS resolution
  ✅ HTTP/HTTPS, TCP handshake
  ✅ Network tools: ping, traceroute, netstat/ss, tcpdump
  Hands-on: Configure a Linux VM as a basic web server
```

### Phase 2: Git & Docker (Weeks 5–8)

```
Milestone: Can build and run containerized applications, use Git effectively

Week 5-6: Git
  ✅ Clone, branch, commit, push, pull
  ✅ Merge vs rebase
  ✅ Resolving merge conflicts
  ✅ GitFlow or trunk-based development
  ✅ Pre-commit hooks
  Hands-on: Contribute to an open source project (even documentation)

Week 7-8: Docker
  ✅ Images, containers, layers
  ✅ Writing Dockerfiles (multi-stage builds)
  ✅ Docker networking and volumes
  ✅ Docker Compose for multi-service apps
  ✅ Pushing to Docker Hub / GHCR
  Hands-on: Containerize an existing app (your own or a sample)
```

---

## Mid-Level Path

### Phase 3: Kubernetes & CI/CD (Weeks 9–16)

```
Milestone: Can deploy applications to Kubernetes and build CI/CD pipelines

Week 9-12: Kubernetes
  ✅ Architecture: control plane, nodes, etcd
  ✅ Core objects: Pod, Deployment, Service, Ingress
  ✅ ConfigMaps and Secrets
  ✅ RBAC basics
  ✅ Helm basics
  ✅ kubectl proficiency
  Hands-on: Deploy a full stack app to local Kubernetes (minikube/kind)
            Then to a managed cluster (EKS free tier, GKE Autopilot, etc.)

Week 13-16: CI/CD
  ✅ GitHub Actions or Jenkins pipeline
  ✅ Build, test, push Docker image
  ✅ Deploy to Kubernetes from CI
  ✅ Basic GitOps concept (ArgoCD or Flux)
  ✅ Secret management in pipelines
  Hands-on: Build end-to-end pipeline: code push → container → K8s deployment
```

### Phase 4: Infrastructure as Code (Weeks 17–20)

```
Milestone: Can manage cloud infrastructure declaratively

Week 17-18: Terraform
  ✅ HCL syntax, providers, resources
  ✅ State management (remote backend)
  ✅ Variables, outputs, locals
  ✅ Modules
  ✅ Workspaces
  Hands-on: Provision a VPC + EC2 + RDS on AWS with Terraform

Week 19-20: Ansible
  ✅ Inventory, ad-hoc commands
  ✅ Playbooks, tasks, handlers
  ✅ Roles
  ✅ Variables and templates
  ✅ Vault for secrets
  Hands-on: Configure an EC2 instance (nginx + app) with Ansible
```

---

## Advanced Path

### Phase 5: Cloud Platforms (Weeks 21–28)

```
Pick ONE cloud platform to go deep on (usually based on job market in your area)

AWS (Weeks 21-24):
  ✅ Core: EC2, S3, VPC, IAM
  ✅ EKS, RDS, ElastiCache
  ✅ Lambda, SQS, SNS
  ✅ CloudFormation / CDK
  ✅ Well-Architected Framework
  Target cert: AWS Solutions Architect Associate OR DevOps Engineer Professional

Azure (alternative):
  ✅ VMs, VNet, NSG
  ✅ AKS, ACR, Azure DevOps
  ✅ ARM Templates / Bicep
  ✅ Azure AD / Entra ID, RBAC
  Target cert: AZ-104 or AZ-400

GCP (alternative):
  ✅ GCE, GKE, Cloud Run
  ✅ Cloud Build, Artifact Registry
  ✅ IAM, Workload Identity
  ✅ Cloud Monitoring, Logging
  Target cert: GCP Associate Cloud Engineer
```

### Phase 6: Observability & Security (Weeks 29–34)

```
Week 29-31: Monitoring & Logging
  ✅ Prometheus + Grafana + Alertmanager
  ✅ ELK Stack or Loki
  ✅ Distributed tracing (Jaeger/Tempo)
  ✅ SLIs/SLOs/error budgets
  ✅ Alert design (avoid alert fatigue)
  Hands-on: Deploy full observability stack on Kubernetes
            Create dashboards, write alert rules, simulate incidents

Week 32-34: Security (DevSecOps)
  ✅ SAST/DAST tools (Semgrep, Trivy, ZAP)
  ✅ Container security best practices
  ✅ Kubernetes RBAC + NetworkPolicy
  ✅ Secrets management (Vault or cloud-native)
  ✅ Supply chain security (Cosign, SBOM)
  Hands-on: Add security scanning to CI pipeline
            Implement OPA Gatekeeper policies
```

### Phase 7: SRE & System Design (Weeks 35–40)

```
Week 35-37: SRE
  ✅ SLO implementation in Prometheus
  ✅ Incident management process
  ✅ Postmortem writing
  ✅ Error budget policy
  ✅ Toil identification and automation
  Hands-on: Define SLOs for a real or sample service
            Write a sample postmortem

Week 38-40: System Design
  ✅ Scalability patterns
  ✅ Availability patterns (CAP theorem)
  ✅ Microservices patterns (circuit breaker, saga, API gateway)
  ✅ Database scaling (sharding, read replicas, CQRS)
  ✅ Practice designing 5+ systems
  Hands-on: Practice system design with a partner or in front of a mirror
```

---

## Role-Based Roadmaps

### DevOps Engineer

```
Priority order:
1. Linux (foundation)
2. Docker (containerization)
3. Git + CI/CD (automation)
4. Kubernetes (orchestration)
5. ONE cloud platform (AWS or Azure or GCP)
6. Terraform (infrastructure as code)
7. Monitoring (Prometheus/Grafana)
8. Ansible (configuration management)
```

### Platform Engineer / SRE

```
Priority order:
1. Kubernetes (deep — operators, advanced networking)
2. SRE concepts (SLOs, error budgets, toil)
3. Monitoring + Logging (full stack)
4. CI/CD + GitOps (ArgoCD/Flux)
5. Security (RBAC, policies, supply chain)
6. System Design (distributed systems)
7. Terraform (IaC at scale)
8. Go or Python (for building tooling)
```

### Cloud Engineer

```
Priority order:
1. ONE cloud deep (3+ years experience level)
2. IaC (Terraform + cloud-native CDK)
3. Networking (VPC, peering, DNS, LB)
4. Security (IAM, KMS, secrets)
5. Kubernetes on cloud (EKS/AKS/GKE)
6. Cost optimization
7. Multi-cloud basics
8. Migration patterns
```

---

## Certification Roadmap

### By Difficulty and Value

| Cert | Difficulty | Market Value | Time to Prepare |
|------|------------|-------------|-----------------|
| AWS Cloud Practitioner | ⭐ | Low | 2-4 weeks |
| AWS Solutions Architect Associate | ⭐⭐⭐ | High | 2-3 months |
| AWS DevOps Engineer Professional | ⭐⭐⭐⭐ | Very High | 3-4 months |
| CKA (Certified Kubernetes Admin) | ⭐⭐⭐ | High | 2-3 months |
| CKAD (Certified K8s App Developer) | ⭐⭐⭐ | Medium-High | 1-2 months |
| CKS (Certified K8s Security Specialist) | ⭐⭐⭐⭐ | High | 2-3 months |
| Terraform Associate | ⭐⭐ | Medium | 4-6 weeks |
| Azure AZ-104 | ⭐⭐⭐ | High (Azure shops) | 2-3 months |
| GCP ACE | ⭐⭐⭐ | Medium | 2-3 months |

### Recommended Certification Path

```
Beginner:   AWS Cloud Practitioner → AWS SAA
Mid-level:  CKA → AWS DevOps Professional or Terraform Associate
Advanced:   CKS → AWS SAP (Solutions Architect Pro)
Kubernetes: CKA → CKAD → CKS
```

---

## Weekly Study Plan

### 10 Hours/Week Schedule

```
Monday    (2h): Read theory + take notes
Tuesday   (2h): Hands-on practice / lab
Wednesday (1h): Review flashcards / quiz
Thursday  (2h): Practice interview questions (answer out loud)
Friday    (1h): Review week's learnings
Weekend   (2h): Mini-project or full mock interview
```

### Daily Habits (30 min/day)

```
Morning  (15 min): Read one documentation page or article
Evening  (15 min): Review flashcards OR practice one command/concept
Weekly:           One hands-on lab (2-3 hours)
Monthly:          Mock interview or study group session
```

---

## Resources by Topic

### Linux
- **Book**: "The Linux Command Line" — William Shotts (free online)
- **Practice**: OverTheWire Bandit, LinuxJourney.com
- **Course**: Linux Foundation courses (lf.me)

### Docker & Kubernetes
- **Official Docs**: docs.docker.com, kubernetes.io/docs
- **Course**: "Kubernetes the Hard Way" — Kelsey Hightower (GitHub)
- **Practice**: Killercoda.com, Play with Kubernetes, Kodekloud
- **Book**: "Kubernetes in Action" — Luksa

### Terraform
- **Official**: learn.hashicorp.com
- **Book**: "Terraform: Up & Running" — Brikman
- **Practice**: Terraform Registry + AWS free tier

### AWS
- **Official**: aws.amazon.com/training
- **Course**: Adrian Cantrill (cantrill.io) — highly recommended
- **Practice**: AWS free tier + Whizlabs practice exams

### CI/CD & GitOps
- **ArgoCD docs**: argo-cd.readthedocs.io
- **Book**: "The DevOps Handbook" — Kim, Humble, Debois
- **Course**: LinuxFoundation CI/CD courses

### SRE & Reliability
- **Book**: "Site Reliability Engineering" — Google (free at sre.google)
- **Book**: "The SRE Workbook" — Google (free at sre.google)
- **Blog**: SRE Weekly newsletter, Google SRE blog

### System Design
- **Book**: "Designing Data-Intensive Applications" — Kleppmann
- **YouTube**: "System Design Interview" — Alex Xu
- **Practice**: Excalidraw for drawing, mock interviews with peers

---

## Interview Preparation Timeline

### 8-Week Sprint (for upcoming interview)

```
Week 1: Linux + Networking review
  - Re-read Linux.md and Networking.md
  - Practice 20 CLI commands daily
  - Answer all questions in each file out loud

Week 2: Docker + Kubernetes review
  - Re-read Docker.md and Kubernetes.md
  - Run hands-on labs
  - Practice kubectl commands

Week 3: CI/CD + Jenkins/GitHub Actions
  - Build a sample pipeline from scratch
  - Review CICD.md and Jenkins.md
  - Practice Groovy scripting

Week 4: Terraform + Ansible
  - Build a mini infrastructure project
  - Review Terraform.md and Ansible.md
  - Practice IaC patterns

Week 5: Cloud platform (your primary)
  - Review AWS.md or Azure.md or GCP.md
  - Practice cloud CLI commands
  - Review architecture diagrams

Week 6: Monitoring + Security + SRE
  - Review Monitoring.md, Logging.md, Security.md, SRE.md
  - Practice PromQL queries
  - Write 2 sample postmortems

Week 7: System Design + Behavioral
  - Practice 5 system designs (timer: 45 min each)
  - Prepare 8 STAR stories (write them out)
  - Practice out loud

Week 8: Mock Interviews + Review
  - 2-3 full mock interviews (record yourself or use pramp.com)
  - Review CheatSheets.md daily
  - Prepare questions to ask the interviewer
  - Rest and confidence building
```

### Day Before Interview

```
✅ Review CheatSheets.md (quick reference)
✅ Re-read your STAR stories
✅ Review the company's engineering blog / tech stack
✅ Prepare thoughtful questions for the interviewer
✅ Get 8 hours of sleep
✅ Lay out clothes, prepare commute/setup
```

### Day of Interview

```
Morning:
  ✅ Brief review of CheatSheets.md (30 min max)
  ✅ Eat a good breakfast
  ✅ Technical setup (camera, mic, Internet connection)

During interview:
  ✅ Think out loud — interviewers want to see your reasoning
  ✅ Clarify requirements before coding/designing
  ✅ It's OK to say "I'm not sure, but I'd approach it by..."
  ✅ Ask for feedback at end: "Is there anything you'd like me to elaborate on?"

After interview:
  ✅ Send thank-you note within 24 hours
  ✅ Write down questions you couldn't answer → study them
  ✅ Reflect: what went well, what to improve
```

---

> **Cross-links:** [← Behavioral](./Behavioral.md) | [Cheat Sheets →](./CheatSheets.md) | [Back to Main Index →](./README.md)