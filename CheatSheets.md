---
layout: default
title: "⚡ Cheat Sheets — DevOps Quick Reference"
render_with_liquid: false
---

# ⚡ Cheat Sheets — DevOps Quick Reference

> [← Learning Roadmap](./LearningRoadmap.md) | [Main Index](./README.md)

> **Tip**: Bookmark this file. Review it the day before your interview.

---

## Table of Contents

1. [Linux Cheat Sheet](#linux-cheat-sheet)
2. [Git Cheat Sheet](#git-cheat-sheet)
3. [Docker Cheat Sheet](#docker-cheat-sheet)
4. [Kubernetes (kubectl) Cheat Sheet](#kubernetes-kubectl-cheat-sheet)
5. [Terraform Cheat Sheet](#terraform-cheat-sheet)
6. [Ansible Cheat Sheet](#ansible-cheat-sheet)
7. [AWS CLI Cheat Sheet](#aws-cli-cheat-sheet)
8. [Prometheus/PromQL Cheat Sheet](#prometheuspromql-cheat-sheet)
9. [Jenkins Pipeline Cheat Sheet](#jenkins-pipeline-cheat-sheet)
10. [Security Cheat Sheet](#security-cheat-sheet)
11. [Networking Quick Reference](#networking-quick-reference)
12. [SRE Quick Reference](#sre-quick-reference)

---

## Linux Cheat Sheet

```bash
# ─── FILE SYSTEM ──────────────────────────────────────
ls -lah                      # List with sizes
find / -name "*.conf" -type f 2>/dev/null
find /var/log -mtime -1      # Modified in last day
du -sh /var/log/*            # Disk usage per dir
df -h                        # Disk free all filesystems
df -i                        # Inode usage

# ─── PERMISSIONS ──────────────────────────────────────
chmod 755 file               # rwxr-xr-x
chmod +x script.sh           # Add execute bit
chmod -R 644 /var/www/       # Recursive
chown user:group file        # Change owner
# Permission bits: 4=read 2=write 1=execute
# 755 = rwxr-xr-x (owner:all, group:rx, other:rx)
# 644 = rw-r--r-- (owner:rw, rest:r)

# ─── PROCESSES ────────────────────────────────────────
ps aux                       # All processes
ps aux | grep nginx          # Filter
top -b -n1 | head -20        # Top snapshot
kill -9 PID                  # Force kill
kill -15 PID                 # Graceful (SIGTERM)
pgrep -a nginx               # Find PID by name
lsof -i :80                  # Who uses port 80

# ─── NETWORKING ───────────────────────────────────────
ip addr show                 # IP addresses
ss -tulnp                    # Active listeners (preferred)
netstat -tulnp               # Legacy
ping -c 4 host               # ICMP test
traceroute host              # Trace hops
dig domain.com               # DNS lookup
dig +short domain.com        # IP only
nc -zv host 443              # Test TCP port

# ─── SYSTEM ───────────────────────────────────────────
uptime                       # Load averages
free -h                      # Memory
iostat -xz 1                 # Disk I/O
journalctl -xe               # Systemd journal errors
journalctl -u nginx -f       # Follow service logs
systemctl status nginx       # Service status
systemctl restart nginx      # Restart service

# ─── USEFUL ONE-LINERS ────────────────────────────────
tail -f /var/log/app.log | grep -i error
grep -r "pattern" /etc/ 2>/dev/null
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head
sed -i 's/old/new/g' config.conf
find . -name "*.log" -mtime +7 -delete
tar -czvf backup.tar.gz /path/to/dir
tar -xzvf backup.tar.gz -C /dest/
```

---

## Git Cheat Sheet

```bash
# ─── SETUP ────────────────────────────────────────────
git config --global user.name "Name"
git config --global user.email "email@example.com"
git config --global core.editor vim
git config --global pull.rebase true

# ─── BRANCH ───────────────────────────────────────────
git switch -c feature/new     # Create + switch (modern)
git checkout -b feature/new   # Create + switch (classic)
git branch -a                 # All branches
git branch -d feature         # Delete (safe)
git branch -D feature         # Force delete
git push origin --delete feature  # Delete remote

# ─── STAGING & COMMITS ────────────────────────────────
git add -p                    # Interactive hunk staging
git status -s                 # Short status
git diff HEAD                 # All changes since last commit
git diff --staged             # Staged changes
git commit -m "feat: add login"
git commit --amend            # Fix last commit

# ─── SYNC ─────────────────────────────────────────────
git fetch --prune             # Fetch + remove stale refs
git pull --rebase origin main
git push --force-with-lease   # Safe force push

# ─── UNDO ─────────────────────────────────────────────
git restore file.py           # Discard working changes
git restore --staged file.py  # Unstage
git revert SHA                # Safe undo (creates new commit)
git reset --soft HEAD~1       # Undo commit, keep staged
git reset --hard HEAD~1       # Undo commit, discard everything

# ─── HISTORY ──────────────────────────────────────────
git log --oneline --graph --all
git log --author="Alice" --since="2 weeks ago"
git blame file.py
git reflog                    # All HEAD movements (lifesaver)
git bisect start              # Binary search for bug

# ─── STASH ────────────────────────────────────────────
git stash push -m "WIP: login"
git stash list
git stash pop
git stash apply stash@{1}

# ─── REBASE ───────────────────────────────────────────
git rebase main               # Rebase feature onto main
git rebase -i HEAD~4          # Interactive (squash, reorder)
git rebase --abort
git rebase --continue         # After resolving conflict

# ─── ALIASES ──────────────────────────────────────────
git config --global alias.lg "log --oneline --graph --all"
git config --global alias.undo "reset --soft HEAD~1"
```

---

## Docker Cheat Sheet

```bash
# ─── BUILD ────────────────────────────────────────────
docker build -t name:tag .
docker build --build-arg ENV=prod -t name:tag .
docker build --target stage_name -t name:tag .

# ─── RUN ──────────────────────────────────────────────
docker run -d -p 8080:80 --name myapp nginx
docker run -it --rm ubuntu bash
docker run -v /host:/container -e KEY=val myapp

# ─── LIFECYCLE ────────────────────────────────────────
docker ps                     # Running
docker ps -a                  # All
docker stop/start/restart myapp
docker rm myapp               # Remove stopped
docker rm -f myapp            # Force remove running

# ─── INSPECT ──────────────────────────────────────────
docker logs -f --tail 100 myapp
docker exec -it myapp bash
docker inspect myapp
docker stats myapp
docker top myapp
docker diff myapp             # Changed files in container

# ─── IMAGES ───────────────────────────────────────────
docker images
docker pull nginx:1.25
docker push user/app:tag
docker rmi myapp:old
docker history myapp
docker save myapp > myapp.tar
docker load < myapp.tar

# ─── CLEANUP ──────────────────────────────────────────
docker system prune -a        # Remove ALL unused
docker container prune
docker image prune -a
docker volume prune
docker network prune

# ─── COMPOSE ──────────────────────────────────────────
docker compose up -d          # Start detached
docker compose up --build     # Rebuild first
docker compose down           # Stop + remove
docker compose down -v        # Also remove volumes
docker compose ps
docker compose logs -f app
docker compose exec app bash
docker compose run --rm app pytest

# ─── REGISTRY ─────────────────────────────────────────
docker login
docker tag myapp user/myapp:v1
# ECR login:
aws ecr get-login-password | docker login --username AWS \
  --password-stdin ACCOUNT.dkr.ecr.REGION.amazonaws.com

# ─── MULTI-ARCH ───────────────────────────────────────
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 \
  -t user/app:latest --push .
```

---

## Kubernetes (kubectl) Cheat Sheet

```bash
# ─── CONTEXT ──────────────────────────────────────────
kubectl config get-contexts
kubectl config use-context prod-cluster
kubectl config set-context --current --namespace=production
kubectx               # kubectx plugin for fast switching
kubens                # kubens plugin for namespace switching

# ─── GET ──────────────────────────────────────────────
kubectl get all -n namespace
kubectl get pods -o wide
kubectl get pods -w               # Watch
kubectl get pods -l app=myapp     # Label selector
kubectl get pods --sort-by=.status.startTime
kubectl get events -n ns --sort-by='.lastTimestamp' | tail -20
kubectl get nodes -o wide

# ─── DESCRIBE / LOGS ──────────────────────────────────
kubectl describe pod myapp-abc -n production
kubectl describe node worker-1
kubectl logs myapp-abc -f
kubectl logs myapp-abc --previous -c container-name
kubectl logs -l app=myapp --all-containers=true -f
kubectl top pods -n production --containers

# ─── EXEC / PORT-FORWARD ──────────────────────────────
kubectl exec -it myapp-abc -- bash
kubectl exec -it myapp-abc -c sidecar -- sh
kubectl port-forward pod/myapp-abc 8080:80
kubectl port-forward svc/myapp-svc 8080:80

# ─── APPLY / DELETE ───────────────────────────────────
kubectl apply -f manifest.yaml
kubectl apply -k overlays/production
kubectl delete -f manifest.yaml
kubectl delete pod myapp-abc --grace-period=0 --force

# ─── SCALE / ROLLOUT ──────────────────────────────────
kubectl scale deployment myapp --replicas=5
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp
kubectl rollout undo deployment/myapp --to-revision=3
kubectl rollout restart deployment/myapp
kubectl set image deployment/myapp myapp=myapp:v2

# ─── LABELS / ANNOTATIONS ─────────────────────────────
kubectl label pod myapp-abc env=production
kubectl annotate pod myapp-abc team=platform
kubectl get pods --show-labels

# ─── DEBUG ────────────────────────────────────────────
kubectl run tmp --rm -it --image=nicolaka/netshoot -- bash
kubectl debug node/worker-1 -it --image=ubuntu
kubectl auth can-i get pods --namespace=production
kubectl get rolebindings -A -o wide

# ─── HELM ─────────────────────────────────────────────
helm list -A
helm upgrade --install app ./chart -n ns --atomic --timeout 5m
helm rollback app 2 -n ns
helm history app -n ns
helm diff upgrade app ./chart -f values.yaml    # requires diff plugin
helm template app ./chart -f values.yaml
helm lint ./chart
```

---

## Terraform Cheat Sheet

```bash
# ─── INIT / FORMAT / VALIDATE ─────────────────────────
terraform init
terraform init -upgrade           # Update providers
terraform fmt -recursive          # Format .tf files
terraform validate                # Check syntax

# ─── PLAN / APPLY ─────────────────────────────────────
terraform plan
terraform plan -out=tfplan
terraform plan -var-file=prod.tfvars
terraform apply
terraform apply tfplan
terraform apply -auto-approve     # No confirmation (CI only)
terraform apply -target=aws_instance.web
terraform destroy -auto-approve

# ─── STATE ────────────────────────────────────────────
terraform state list
terraform state show aws_instance.web
terraform state mv old.path new.path
terraform state rm resource.to.unmanage
terraform import aws_instance.web i-1234567890
terraform refresh
terraform force-unlock LOCK-ID

# ─── WORKSPACE ────────────────────────────────────────
terraform workspace list
terraform workspace new dev
terraform workspace select prod
terraform workspace show

# ─── OUTPUT / CONSOLE ─────────────────────────────────
terraform output
terraform output -json | jq
terraform console                 # Interactive

# ─── LINTING / SCANNING ───────────────────────────────
tflint                           # Linting
tfsec .                          # Security scan
checkov -d . --framework terraform
infracost breakdown --path .     # Cost estimation

# ─── HCL QUICK REFERENCE ──────────────────────────────
# count
resource "aws_instance" "app" { count = 3 }
# for_each
resource "aws_subnet" "main" { for_each = toset(var.azs) }
# dynamic block
dynamic "ingress" { for_each = var.ports; content { port = ingress.value } }
# locals
locals { name = "${var.env}-${var.app}" }
# data source
data "aws_ami" "latest" { most_recent = true }
# output
output "vpc_id" { value = aws_vpc.main.id; sensitive = false }
```

---

## Ansible Cheat Sheet

```bash
# ─── AD-HOC ───────────────────────────────────────────
ansible all -m ping
ansible webservers -m setup              # Gather facts
ansible webservers -m shell -a "df -h"
ansible webservers -m copy -a "src=file dest=/tmp/"
ansible webservers -m service -a "name=nginx state=restarted" --become
ansible webservers -m apt -a "name=nginx state=present" --become

# ─── PLAYBOOK ─────────────────────────────────────────
ansible-playbook site.yml
ansible-playbook site.yml -i inventory/
ansible-playbook site.yml --tags deploy
ansible-playbook site.yml --skip-tags test
ansible-playbook site.yml --limit webservers
ansible-playbook site.yml --limit web01
ansible-playbook site.yml --check         # Dry run
ansible-playbook site.yml --diff          # Show file diffs
ansible-playbook site.yml -e "version=2.0"
ansible-playbook site.yml -v/-vv/-vvv     # Verbosity levels
ansible-playbook site.yml --start-at-task "Deploy app"
ansible-playbook site.yml --step          # Interactive step-through

# ─── INVENTORY ────────────────────────────────────────
ansible-inventory --list
ansible-inventory --graph
ansible-inventory -i inventory/aws_ec2.yml --list

# ─── VAULT ────────────────────────────────────────────
ansible-vault encrypt vars/secrets.yml
ansible-vault decrypt vars/secrets.yml
ansible-vault edit vars/secrets.yml
ansible-vault view vars/secrets.yml
ansible-vault encrypt_string 'secret' --name 'db_pass'
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file .vault_pass

# ─── GALAXY ───────────────────────────────────────────
ansible-galaxy role init myrole
ansible-galaxy install geerlingguy.nginx
ansible-galaxy collection install community.docker
ansible-galaxy install -r requirements.yml
```

---

## AWS CLI Cheat Sheet

```bash
# ─── AUTH ─────────────────────────────────────────────
aws configure                        # Set credentials
aws sts get-caller-identity          # Who am I?
aws sts assume-role --role-arn ARN --role-session-name s

# ─── EC2 ──────────────────────────────────────────────
aws ec2 describe-instances --filters "Name=tag:Name,Values=web*" \
  --query 'Reservations[].Instances[].{ID:InstanceId,IP:PrivateIpAddress,State:State.Name}'
aws ec2 start-instances --instance-ids i-1234
aws ec2 stop-instances --instance-ids i-1234
aws ssm start-session --target i-1234     # SSH-less access

# ─── S3 ───────────────────────────────────────────────
aws s3 ls s3://bucket/prefix/
aws s3 cp file.txt s3://bucket/
aws s3 sync ./local s3://bucket/remote --delete
aws s3 presign s3://bucket/file --expires-in 3600

# ─── EKS ──────────────────────────────────────────────
aws eks list-clusters
aws eks update-kubeconfig --name cluster --region us-east-1
aws eks describe-cluster --name cluster

# ─── IAM ──────────────────────────────────────────────
aws iam list-roles | jq '.Roles[].RoleName'
aws iam get-role --role-name MyRole
aws iam list-attached-role-policies --role-name MyRole

# ─── LOGS ─────────────────────────────────────────────
aws logs tail /aws/lambda/fn --follow
aws logs filter-log-events --log-group /app --filter-pattern ERROR

# ─── SECRETSMANAGER ───────────────────────────────────
aws secretsmanager get-secret-value --secret-id prod/myapp/db
aws secretsmanager create-secret --name prod/db --secret-string '{}'

# ─── HELPFUL ──────────────────────────────────────────
aws configure list-profiles
aws ec2 describe-regions --output table
export AWS_PROFILE=production
export AWS_DEFAULT_REGION=us-east-1
```

---

## Prometheus/PromQL Cheat Sheet

```promql
# ─── BASICS ───────────────────────────────────────────
up                                          # Scrape success (1/0)
up{job="myapp"}                            # Filter by label
rate(http_requests_total[5m])              # Rate per second
increase(http_requests_total[1h])          # Total increase

# ─── AGGREGATIONS ─────────────────────────────────────
sum(metric) by (label)                     # Sum grouped by label
avg(metric) without (label)               # Avg, drop label
count(metric)                              # Count time series
topk(5, metric)                            # Top 5
min_over_time(metric[1h])                  # Min in time range

# ─── LATENCY ──────────────────────────────────────────
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# ─── ERROR RATE ───────────────────────────────────────
sum(rate(http_requests_total{code=~"5.."}[5m])) /
sum(rate(http_requests_total[5m]))

# ─── COMMON RESOURCE QUERIES ──────────────────────────
# CPU per pod
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)

# Memory per pod (working set)
sum(container_memory_working_set_bytes{container!=""}) by (pod)

# Node disk usage %
100 - (node_filesystem_avail_bytes / node_filesystem_size_bytes * 100)

# Pod restarts
rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15

# ─── RECORDING RULES ──────────────────────────────────
# In rules YAML:
# - record: job:http_requests:rate5m
#   expr: sum(rate(http_requests_total[5m])) by (job)
```

---

## Jenkins Pipeline Cheat Sheet

```groovy
// ─── ENVIRONMENT ──────────────────────────────────────
env.JOB_NAME          // Job name
env.BUILD_NUMBER      // Build #
env.WORKSPACE         // Agent workspace
env.GIT_COMMIT        // Full SHA
env.GIT_BRANCH        // Current branch
env.BUILD_URL         // Link to this build

// ─── CONDITIONALS ─────────────────────────────────────
when { branch 'main' }
when { not { branch 'main' } }
when { anyOf { branch 'main'; branch 'develop' } }
when { environment name: 'DEPLOY', value: 'true' }
when { expression { return params.DEPLOY } }
when { changeset "**/*.java" }

// ─── CREDENTIALS ──────────────────────────────────────
withCredentials([
  usernamePassword(credentialsId: 'id', usernameVariable: 'USR', passwordVariable: 'PWD'),
  string(credentialsId: 'token', variable: 'TOKEN'),
  file(credentialsId: 'kube', variable: 'KUBECONFIG')
]) { ... }

// ─── COMMON STEPS ─────────────────────────────────────
sh 'command'
sh(script: 'cmd', returnStdout: true).trim()
archiveArtifacts artifacts: '**/*.jar'
junit 'target/surefire-reports/*.xml'
stash name: 'built', includes: 'dist/**'
unstash 'built'
input message: 'Deploy?', ok: 'Yes', submitter: 'ops'
timeout(time: 10, unit: 'MINUTES') { ... }
retry(3) { ... }
sleep(time: 30, unit: 'SECONDS')
cleanWs()

// ─── POST ──────────────────────────────────────────────
post {
  always   { cleanWs() }
  success  { slackSend "✅ Done" }
  failure  { emailext to: 'ops@...', subject: "FAILED: ${JOB_NAME}" }
  unstable { echo "Tests failed" }
  changed  { echo "Status changed" }
}
```

---

## Security Cheat Sheet

```bash
# ─── IMAGE SCANNING ───────────────────────────────────
trivy image myapp:latest
trivy image --severity HIGH,CRITICAL myapp:latest
trivy fs .
docker scout cves myapp:latest
snyk container test myapp:latest

# ─── SECRET SCANNING ──────────────────────────────────
gitleaks detect --source .
detect-secrets scan > .secrets.baseline
trufflehog git https://github.com/org/repo

# ─── INFRA SCANNING ───────────────────────────────────
checkov -d . --framework terraform,dockerfile,kubernetes
tfsec .
kics scan -p .

# ─── K8S SECURITY ─────────────────────────────────────
kubectl auth can-i --list
kubectl auth can-i get pods --as=system:serviceaccount:ns:sa
kubectl get rolebindings,clusterrolebindings -A -o wide | grep sa-name
kubesec scan pod.yaml
kube-bench run --targets node

# ─── VAULT ────────────────────────────────────────────
vault kv get secret/myapp
vault kv put secret/myapp db_pass=secret
vault token lookup
vault policy list
vault auth list

# ─── CONTAINER HARDENING ──────────────────────────────
docker run \
  --read-only \
  --tmpfs /tmp \
  --cap-drop=ALL \
  --security-opt=no-new-privileges:true \
  --pids-limit=100 \
  --memory=512m \
  myapp
```

---

## Networking Quick Reference

### OSI Model Mnemonics
```
All People Seem To Need Data Processing
Layer 7: Application   (HTTP, DNS, FTP)
Layer 6: Presentation  (SSL/TLS, encoding)
Layer 5: Session       (NetBIOS, RPC)
Layer 4: Transport     (TCP, UDP, ports)
Layer 3: Network       (IP, ICMP, routing)
Layer 2: Data Link     (MAC, Ethernet)
Layer 1: Physical      (cables, signals)
```

### Common Ports
| Port | Protocol | Service |
|------|----------|---------|
| 22 | TCP | SSH |
| 25 | TCP | SMTP |
| 53 | TCP/UDP | DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 9200 | TCP | Elasticsearch |
| 9090 | TCP | Prometheus |
| 3000 | TCP | Grafana |

### Subnet Quick Math
```
/32 = 1 host
/30 = 2 hosts
/29 = 6 hosts
/28 = 14 hosts
/27 = 30 hosts
/26 = 62 hosts
/25 = 126 hosts
/24 = 254 hosts
/23 = 510 hosts
/22 = 1022 hosts
/21 = 2046 hosts
/20 = 4094 hosts
/16 = 65,534 hosts

Formula: hosts = 2^(32-prefix) - 2
```

---

## SRE Quick Reference

### DORA Metrics (Elite Performance)
| Metric | Elite | High | Medium | Low |
|--------|-------|------|--------|-----|
| Deploy Frequency | Multiple/day | Once/week | Once/month | Once/6 months |
| Lead Time | < 1 hour | 1 day–1 week | 1 week–1 month | > 6 months |
| MTTR | < 1 hour | < 1 day | < 1 week | > 6 months |
| Change Failure Rate | < 5% | 5–10% | 10–30% | > 30% |

### SLO Error Budget

```
SLO → Error Budget/month
99.999% → 26.3 seconds
99.99%  → 4.4 minutes
99.9%   → 43.8 minutes
99.5%   → 3.65 hours
99.0%   → 7.3 hours
98.0%   → 14.6 hours
```

### Incident Severity
```
SEV-1: Complete outage, data loss     → Page immediately, all hands
SEV-2: Partial outage, major feature  → Page on-call
SEV-3: Degraded, workaround exists    → Slack, fix in hours
SEV-4: Minor, cosmetic                → Ticket, next sprint
```

---

> **Good luck! 🚀** You've got this. Review the relevant topic files for deep dives, and use this cheat sheet for quick day-of-interview review.

*[← Back to Main Index](./README.md)*