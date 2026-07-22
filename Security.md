---
layout: default
title: "🔐 Security — DevOps Interview Guide"
render_with_liquid: false
---

# 🔐 Security — DevOps Interview Guide

> [← Logging](./Logging.md) | [Main Index](./README.md) | [SRE →](./SRE.md)

---

## Table of Contents

1. [DevSecOps Fundamentals](#devsecops-fundamentals)
2. [SAST & DAST](#sast--dast)
3. [Container Security](#container-security)
4. [Kubernetes Security](#kubernetes-security)
5. [Secrets Management](#secrets-management)
6. [Network Security](#network-security)
7. [Compliance & Audit](#compliance--audit)
8. [Interview Questions](#interview-questions)
9. [Scenario-Based Questions](#scenario-based-questions)
10. [Cheat Sheet](#cheat-sheet)

---

## DevSecOps Fundamentals

### Shift Left Security

```
Traditional:  Code → Build → Test → Deploy → Security Audit (too late!)
Shift Left:   Security checks at every stage

Dev Phase:
  - Pre-commit hooks (detect-secrets, git-secrets)
  - IDE plugins (SonarLint, Snyk)
  - Dependency scanning (Dependabot, Renovate)

CI Phase:
  - SAST (SonarQube, Semgrep, CodeQL)
  - Dependency audit (npm audit, OWASP Dependency Check)
  - Container scanning (Trivy, Snyk Container)
  - SBOM generation (Syft, Docker Scout)
  - Secret scanning (truffleHog, Gitleaks)

CD Phase:
  - Infrastructure scanning (tfsec, checkov, kics)
  - Container signing (Cosign)
  - Policy enforcement (OPA/Gatekeeper)

Runtime:
  - Runtime threat detection (Falco)
  - Network policies
  - RBAC audit
  - Vulnerability management
```

### Security in CI/CD Pipeline

```yaml
# GitHub Actions security pipeline
- name: Run Gitleaks (secret scanning)
  uses: gitleaks/gitleaks-action@v2

- name: SAST with CodeQL
  uses: github/codeql-action/analyze@v3

- name: Dependency scan
  run: |
    npm audit --audit-level=high
    # or
    pip-audit --requirement requirements.txt

- name: Container scan with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    exit-code: '1'
    severity: CRITICAL,HIGH
    format: sarif
    output: trivy-results.sarif

- name: Infrastructure scan with Checkov
  uses: bridgecrewio/checkov-action@v12
  with:
    directory: .
    framework: terraform,dockerfile,kubernetes
    soft_fail_on: LOW,MEDIUM
    hard_fail_on: HIGH,CRITICAL

- name: Sign image with Cosign
  env:
    COSIGN_PRIVATE_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
  run: |
    cosign sign --key env://COSIGN_PRIVATE_KEY \
      ghcr.io/org/myapp:${{ github.sha }}

- name: Generate SBOM
  uses: anchore/sbom-action@v0
  with:
    image: myapp:${{ github.sha }}
    format: spdx-json
```

---

## SAST & DAST

### SAST (Static Application Security Testing)

| Tool | Languages | Integration |
|------|-----------|-------------|
| **Semgrep** | 30+ languages | CLI, CI, IDE |
| **CodeQL** | 10+ languages | GitHub Actions native |
| **SonarQube** | 30+ languages | Jenkins, GitHub, GitLab |
| **Bandit** | Python | pip install |
| **gosec** | Go | go install |
| **ESLint security plugins** | JavaScript | npm |

```bash
# Semgrep — fast, customizable SAST
semgrep --config=auto .                          # Auto-detect language
semgrep --config=p/owasp-top-ten .             # OWASP Top 10 rules
semgrep --config=p/python .                    # Python-specific rules
semgrep --config=custom-rules.yaml .          # Custom rules

# Custom Semgrep rule
cat > no-hardcoded-secrets.yaml << 'EOF'
rules:
  - id: hardcoded-api-key
    patterns:
      - pattern: $KEY = "..."
    pattern-filter:
      - metavariable-regex:
          metavariable: $KEY
          regex: (api_key|secret|password|token)
    message: "Potential hardcoded secret in $KEY"
    severity: ERROR
    languages: [python, javascript, go]
EOF
```

### DAST (Dynamic Application Security Testing)

```bash
# OWASP ZAP — active scanning against running app
docker run -t owasp/zap2docker-stable zap-api-scan.py \
  -t https://api.example.com/openapi.json \
  -f openapi \
  -r zap-report.html \
  --hook=/zap/auth_hook.py

# Passive scan (proxy mode)
docker run -d --name zap -p 8080:8080 \
  owasp/zap2docker-stable zap.sh -daemon -port 8080

# Nuclei — fast vulnerability scanner
nuclei -u https://myapp.example.com \
  -t cves/ \
  -t exposures/ \
  -severity critical,high
```

---

## Container Security

### Docker Security Best Practices

```dockerfile
# ─── SECURE DOCKERFILE ────────────────────────────────
FROM node:20-alpine

# 1. Run as non-root user
RUN addgroup --system appgroup && \
    adduser --system --ingroup appgroup appuser

WORKDIR /app

# 2. Copy with correct ownership
COPY --chown=appuser:appgroup package*.json ./
RUN npm ci --only=production && npm cache clean --force

COPY --chown=appuser:appgroup . .

# 3. Switch to non-root
USER appuser

# 4. Drop all capabilities (set in docker run or K8s)
EXPOSE 3000

# 5. Read-only filesystem (set in docker run)
# docker run --read-only --tmpfs /tmp myapp

# 6. HEALTHCHECK
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

ENTRYPOINT ["node", "server.js"]
```

```bash
# Run container with security hardening
docker run -d \
  --name myapp \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges:true \
  --security-opt="seccomp=seccomp-profile.json" \
  --pids-limit=100 \
  --memory=512m \
  --cpus="0.5" \
  myapp:latest
```

### Image Scanning

```bash
# Trivy — comprehensive vulnerability scanner
trivy image nginx:latest
trivy image --severity HIGH,CRITICAL myapp:latest
trivy image --format sarif myapp:latest > results.sarif
trivy fs .           # Scan filesystem/repo
trivy k8s --report all cluster  # Scan K8s cluster

# Snyk Container
snyk container test myapp:latest
snyk container monitor myapp:latest

# Docker Scout
docker scout cves myapp:latest
docker scout recommendations myapp:latest
```

---

## Kubernetes Security

### Pod Security Standards

```yaml
# Apply Pod Security Standards at namespace level
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Security Context

```yaml
# Hardened Pod spec
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
    supplementalGroups: []
    sysctls: []

  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      capabilities:
        drop: ["ALL"]
        add: []     # Only add what's absolutely needed
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache

  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

### OPA / Gatekeeper (Policy as Code)

```yaml
# ConstraintTemplate — define a policy
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("Missing required labels: %v", [missing])
        }

---
# Constraint — apply the policy
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-app-labels
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet"]
    namespaces: ["production", "staging"]
  parameters:
    labels: ["app", "version", "team"]
```

### Falco (Runtime Threat Detection)

```yaml
# falco-rules.yaml — custom rules
- rule: Sensitive File Read
  desc: Detect reads of sensitive files
  condition: >
    open_read and
    fd.name in (/etc/shadow, /etc/passwd, /root/.ssh/id_rsa) and
    not proc.name in (authorized-processes)
  output: >
    Sensitive file read (user=%user.name command=%proc.cmdline
    file=%fd.name container=%container.name k8s_pod=%k8s.pod.name)
  priority: WARNING
  tags: [filesystem, security]

- rule: Shell in Container
  desc: Detect shell spawn in container
  condition: >
    spawned_process and
    container and
    proc.name in (shell_binaries) and
    not proc.pname in (allowed_parents)
  output: >
    Shell spawned in container (user=%user.name container=%container.name
    shell=%proc.name parent=%proc.pname)
  priority: WARNING
```

---

## Secrets Management

### HashiCorp Vault

```bash
# Start Vault dev server
vault server -dev

# Basic operations
export VAULT_ADDR='http://127.0.0.1:8200'
vault auth -method=token token=root

# Write and read secrets
vault kv put secret/myapp \
  db_password="supersecret" \
  api_key="key123"

vault kv get secret/myapp
vault kv get -field=db_password secret/myapp

# Dynamic secrets (database)
vault secrets enable database
vault write database/config/postgres \
  plugin_name=postgresql-database-plugin \
  allowed_roles="myapp-role" \
  connection_url="postgresql://{{username}}:{{password}}@postgres:5432/mydb" \
  username="vault_admin" \
  password="vault_password"

vault write database/roles/myapp-role \
  db_name=postgres \
  creation_statements="CREATE ROLE '{{name}}' WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO '{{name}}';" \
  default_ttl="1h" \
  max_ttl="24h"

# Get dynamic credentials
vault read database/creds/myapp-role
```

### Vault in Kubernetes (Vault Agent)

```yaml
# Inject secrets into pods via annotations
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"
        vault.hashicorp.com/agent-inject-secret-config.env: "secret/data/myapp"
        vault.hashicorp.com/agent-inject-template-config.env: |
          {{ with secret "secret/data/myapp" }}
          export DB_PASSWORD="{{ .Data.data.db_password }}"
          export API_KEY="{{ .Data.data.api_key }}"
          {{ end }}
    spec:
      serviceAccountName: myapp
      containers:
      - name: myapp
        command: ["sh", "-c", ". /vault/secrets/config.env && node server.js"]
```

### External Secrets Operator (Kubernetes)

```yaml
# SecretStore — connect to AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa

---
# ExternalSecret — sync to Kubernetes Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: myapp-secret        # Creates this K8s Secret
    creationPolicy: Owner
  data:
  - secretKey: db_password    # K8s secret key
    remoteRef:
      key: production/myapp   # AWS Secrets Manager path
      property: db_password   # JSON property within secret
```

---

## Network Security

### Zero Trust Networking

```
Traditional: Trust internal network, block external
Zero Trust:  Never trust, always verify — even internal traffic

Principles:
1. Verify explicitly (authenticate and authorize every request)
2. Use least privilege access
3. Assume breach (isolate systems, encrypt traffic, monitor)

Kubernetes implementation:
- NetworkPolicies: default-deny, explicit allow
- mTLS: service mesh (Istio, Linkerd) encrypts all service traffic
- OPA: policy enforcement on every request
- Audit logging: every action logged
```

### Network Policy (Zero Trust)

```yaml
# Default deny all ingress/egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}     # Applies to all pods in namespace
  policyTypes:
  - Ingress
  - Egress

---
# Allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:                    # Allow DNS
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is the OWASP Top 10?**

> The OWASP Top 10 is a standard awareness document for web application security. The current top issues include:
> 1. Broken Access Control
> 2. Cryptographic Failures
> 3. Injection (SQL, NoSQL, Command)
> 4. Insecure Design
> 5. Security Misconfiguration
> 6. Vulnerable and Outdated Components
> 7. Identification and Authentication Failures
> 8. Software and Data Integrity Failures
> 9. Security Logging and Monitoring Failures
> 10. Server-Side Request Forgery (SSRF)

**Q2: What is the principle of least privilege?**

> **Least privilege** means granting the minimum permissions necessary for a subject (user, service, process) to perform its function — nothing more. In DevOps: a deployment service account only needs permissions to update specific K8s deployments, not cluster-admin. A Lambda function only needs the S3 bucket it reads from. A database user only has SELECT on tables it queries. This limits the blast radius if a credential is compromised.

**Q3: What is the difference between authentication and authorization?**

> **Authentication (AuthN)**: "Who are you?" — verifying identity (username/password, certificate, token, biometrics). **Authorization (AuthZ)**: "What can you do?" — determining permissions after identity is confirmed (RBAC, ABAC, policies). Example: a user authenticated with valid credentials (AuthN) but only has read access to specific resources (AuthZ).

---

### 🟡 Intermediate

**Q4: How do you store and manage secrets in Kubernetes?**

> Built-in Kubernetes Secrets are base64-encoded (not encrypted) — insufficient for production. Better approaches:
> - **External Secrets Operator**: sync secrets from AWS Secrets Manager, Vault, GCP Secret Manager into K8s Secrets automatically
> - **Vault Agent Sidecar / CSI Driver**: inject secrets directly into pods as files or env vars
> - **Sealed Secrets (Bitnami)**: encrypt Secrets with a cluster-specific key, store encrypted form in Git
> - **Never**: store plaintext secrets in ConfigMaps, env vars in Dockerfiles, or commit `.env` files

**Q5: What is SAST vs DAST?**

> **SAST (Static Application Security Testing)**: analyzes source code or binary without running the application. Catches: SQL injection patterns, hardcoded secrets, insecure dependencies, known vulnerability patterns. Fast, runs in CI. Tools: Semgrep, CodeQL, Bandit. **DAST (Dynamic Application Security Testing)**: tests a running application by sending malicious inputs. Catches: runtime vulnerabilities, authentication issues, business logic flaws. Slower, needs a running environment. Tools: OWASP ZAP, Nuclei.

**Q6: What is supply chain security?**

> Supply chain security protects the entire software delivery pipeline from code to deployment:
> - **SBOM** (Software Bill of Materials): inventory of all components and dependencies
> - **Dependency signing**: verify package integrity (npm integrity hashes, PyPI hash checking)
> - **Image signing**: Cosign signs container images; OPA/Gatekeeper verifies signatures before deploying
> - **Provenance attestation**: SLSA framework documents how artifacts were built
> - **Pin dependencies**: use exact versions or digests, not floating tags
> - **Action pinning**: pin GitHub Actions to full SHA, not tag

---

### 🔴 Advanced

**Q7: Describe a security incident response process for a Kubernetes cluster.**

> **Detection**: Falco alert or unusual activity in audit logs.
> **Isolation**: Cordon and drain affected node; apply NetworkPolicy to isolate compromised pod.
> **Investigation**: `kubectl exec` or forensic container; review audit logs (`kubectl get events`, Cloud audit logs); check for abnormal API calls.
> **Containment**: Force-delete compromised pods; revoke service account credentials; rotate affected secrets.
> **Remediation**: Patch vulnerability; update container images; apply missing security policies.
> **Recovery**: Redeploy clean workloads from GitOps source; verify no persistence mechanisms.
> **Post-incident**: Write incident report; improve detection rules; update RBAC/NetworkPolicy.

**Q8: How does mTLS work in a service mesh?**

> **mTLS (Mutual TLS)**: both client and server authenticate each other with certificates. In Istio/Linkerd: each pod gets a sidecar proxy (Envoy) and a workload certificate issued by the mesh's certificate authority. All service-to-service traffic is encrypted and authenticated at the proxy level — the application doesn't need to implement TLS. This enables: encryption in transit, service identity (not just IP-based), and access policies based on service identity.

---

## Scenario-Based Questions

### 🔵 Scenario 1: Exposed Secret in Git

*"A developer accidentally committed an AWS access key to a public GitHub repository."*

```bash
# 1. IMMEDIATE: Deactivate the key (don't wait!)
aws iam delete-access-key --access-key-id AKIAXXXXXXXX --user-name ci-user
# Or from AWS Console: IAM → Users → Security credentials → Deactivate key

# 2. Rotate: create new key for legitimate use
aws iam create-access-key --user-name ci-user

# 3. Check for unauthorized usage (within 10 min is critical)
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=AKIAXXXXXXXX \
  --start-time 2024-01-01T00:00:00Z

# 4. Remove from Git history
git filter-repo --path-glob "*.env" --invert-paths
# Or use BFG Repo Cleaner

# 5. Force push
git push origin --force --all
git push origin --force --tags

# 6. Prevention:
# - Add .gitignore entries
# - Install pre-commit hooks (detect-secrets, gitleaks)
# - Enable GitHub Secret Scanning
# - Enable Dependabot alerts
```

---

## Cheat Sheet

```bash
# ─── IMAGE SCANNING ───────────────────────────────────
trivy image --severity HIGH,CRITICAL myapp:latest
docker scout cves myapp:latest
snyk container test myapp:latest

# ─── SECRET SCANNING ──────────────────────────────────
gitleaks detect --source .
detect-secrets scan > .secrets.baseline
trufflehog git https://github.com/org/repo

# ─── INFRASTRUCTURE SCANNING ──────────────────────────
checkov -d . --framework terraform
tfsec .
kics scan -p .

# ─── K8S SECURITY ─────────────────────────────────────
kubectl auth can-i --list                    # What can I do?
kubectl auth can-i get pods --as=serviceaccount
kubesec scan pod.yaml                        # Pod security score
kube-bench run --targets node               # CIS benchmark
kubectl get rolebindings,clusterrolebindings -A -o wide

# ─── VAULT ────────────────────────────────────────────
vault kv get secret/myapp
vault kv put secret/myapp key=value
vault token lookup
vault auth list
vault policy list
```

---

> **Cross-links:** [← Logging](./Logging.md) | [SRE →](./SRE.md) | [Kubernetes (K8s security) →](./Kubernetes.md) | [CI/CD (pipeline security) →](./CICD.md) | [AWS (IAM, KMS) →](./AWS.md)