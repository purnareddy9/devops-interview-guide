---
layout: default
title: "🚀 DevOps Interview Preparation Guide"
render_with_liquid: false
---

# 🚀 DevOps Interview Preparation Guide

> 🌐 **Live Site:** [View on GitHub Pages](https://your-username.github.io/devops-interview-guide) *(update URL after first deploy)*

## 🚀 Deploy to GitHub Pages

This repository is pre-configured for **GitHub Pages via Jekyll**. Follow these three steps:

### Step 1 — Enable GitHub Pages in your repo
1. Go to your repo → **Settings** → **Pages**
2. Under **Source**, select **GitHub Actions**
3. Save

### Step 2 — Push to `main`
```bash
git add .
git commit -m "Add GitHub Pages site"
git push origin main
```
The workflow at `.github/workflows/pages.yml` will automatically build and deploy.

### Step 3 — Update your URL
Edit `_config.yml` and set:
```yaml
url: "https://<your-github-username>.github.io"
baseurl: "/<your-repo-name>"
```
Then push again — your site will be live in ~60 seconds.

### Local preview (optional)
```bash
cd devops-interview-guide
bundle install
bundle exec jekyll serve --livereload
# open http://localhost:4000
```

---


> A comprehensive, structured guide covering every major DevOps topic — from Linux fundamentals to cloud platforms, CI/CD, SRE, and behavioral interviews.

---

## 📚 Table of Contents

| # | Topic | Description |
|---|-------|-------------|
| 1 | [Linux](./Linux.md) | File system, processes, networking, shell scripting, performance |
| 2 | [Networking](./Networking.md) | OSI model, TCP/IP, DNS, HTTP, load balancing, firewalls |
| 3 | [Git](./Git.md) | Branching, merging, rebasing, workflows, hooks |
| 4 | [Docker](./Docker.md) | Images, containers, Dockerfile, networking, volumes, Compose |
| 5 | [Kubernetes](./Kubernetes.md) | Pods, deployments, services, RBAC, Helm, troubleshooting |
| 6 | [Jenkins](./Jenkins.md) | Pipelines, Groovy DSL, agents, plugins, shared libraries |
| 7 | [Terraform](./Terraform.md) | HCL, state, modules, workspaces, providers, best practices |
| 8 | [Ansible](./Ansible.md) | Playbooks, roles, inventory, variables, Vault, dynamic inventory |
| 9 | [AWS](./AWS.md) | EC2, S3, VPC, IAM, EKS, Lambda, RDS, CloudFormation |
| 10 | [Azure](./Azure.md) | VMs, AKS, ARM, Azure DevOps, Active Directory, Functions |
| 11 | [GCP](./GCP.md) | GCE, GKE, Cloud Run, IAM, Pub/Sub, Cloud Build |
| 12 | [CI/CD](./CICD.md) | Pipelines, strategies, GitOps, ArgoCD, FluxCD |
| 13 | [Monitoring](./Monitoring.md) | Prometheus, Grafana, alerting, SLOs, dashboards |
| 14 | [Logging](./Logging.md) | ELK Stack, Loki, Fluentd, structured logging, log analysis |
| 15 | [Security](./Security.md) | DevSecOps, SAST/DAST, secrets management, RBAC, compliance |
| 16 | [SRE](./SRE.md) | SLIs/SLOs/SLAs, error budgets, toil, incident management |
| 17 | [System Design](./SystemDesign.md) | Scalability, availability, microservices, patterns |
| 18 | [Behavioral](./Behavioral.md) | STAR method, leadership, conflict, failure stories |
| 19 | [OpenShift](./OpenShift.md) | SCC, Routes, S2I, Pipelines, GitOps, Operators, OLM, cluster admin |
| 20 | [Learning Roadmap](./LearningRoadmap.md) | Structured path from beginner to advanced |
| 21 | [Cheat Sheets](./CheatSheets.md) | Quick reference for all major tools |

---

## 🎯 How to Use This Guide

```
Beginner  → Start with Linux → Networking → Git → Docker
Mid-level → Add Kubernetes → Jenkins → Terraform → Ansible
Advanced  → Cloud (AWS/Azure/GCP) → CI/CD → Monitoring → SRE
All Levels → System Design + Behavioral (always practice these)
```

### Study Strategy

1. **Read** the concept explanation
2. **Answer** questions out loud (simulate interview)
3. **Practice** hands-on labs for each topic
4. **Review** cheat sheets before the interview day
5. **Mock interview** using scenario-based questions

---

## 📊 Difficulty Levels

| Symbol | Level |
|--------|-------|
| 🟢 | Beginner |
| 🟡 | Intermediate |
| 🔴 | Advanced |
| 🔵 | Scenario / Troubleshooting |

---

## 🛠️ Prerequisites

- Basic understanding of Linux command line
- A machine with Docker installed (for labs)
- Free-tier AWS/GCP/Azure account (for cloud labs)
- Git installed locally

---

## 📅 Suggested Study Timeline

| Week | Topics |
|------|--------|
| 1–2 | Linux, Networking, Git |
| 3–4 | Docker, Kubernetes basics |
| 5–6 | CI/CD, Jenkins, Terraform |
| 7–8 | Ansible, AWS core services |
| 9–10 | Azure/GCP, Monitoring, Logging |
| 11–12 | Security, SRE, System Design |
| 13–14 | Behavioral, Mock Interviews, Review |

---

## 🔗 External Resources

- [Linux Foundation Training](https://training.linuxfoundation.org/)
- [Kubernetes Official Docs](https://kubernetes.io/docs/)
- [Docker Docs](https://docs.docker.com/)
- [Terraform Registry](https://registry.terraform.io/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Google SRE Book](https://sre.google/sre-book/table-of-contents/)
- [The DevOps Handbook](https://itrevolution.com/the-devops-handbook/)

---

> **Tip:** Bookmark [CheatSheets.md](./CheatSheets.md) for day-of-interview quick reference.

---

*DevOps Interview Preparation Guide — Covers 20+ topics, 500+ questions, hands-on labs, and real-world scenarios.*