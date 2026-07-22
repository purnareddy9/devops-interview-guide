---
render_with_liquid: false
layout: default
title: "🔒 Module 3 — Security & Compliance"

---

# 🔒 Module 3 — Security & Compliance

> **JD Alignment:** "Implement security best practices for containerized environments. Ensure compliance with security standards and implement necessary controls."

> [← CI/CD Pipelines](./02-CICD-Pipelines.md) | [README](./README.md) | [Monitoring & Troubleshooting →](./04-Monitoring-Troubleshooting.md)

---

## Table of Contents

1. [Container Security Fundamentals](#container-security-fundamentals)
2. [OpenShift SCC & Pod Security](#openshift-scc--pod-security)
3. [RBAC](#rbac)
4. [Network Security](#network-security)
5. [Secrets Management](#secrets-management)
6. [Image Security & Supply Chain](#image-security--supply-chain)
7. [Compliance & Audit](#compliance--audit)

---

## Container Security Fundamentals

---

### 🟢 Q1. What is the principle of least privilege in a containerized environment?

**Answer:**

Least privilege means every component — container, service account, user, network connection — has only the permissions it needs to do its job and nothing more.

**Applied to containers:**

```yaml
spec:
  serviceAccountName: myapp-sa     # Dedicated SA — not default
  automountServiceAccountToken: false   # Don't mount SA token unless needed

  securityContext:
    runAsNonRoot: true             # Never run as root
    runAsUser: 1001                # Specific non-root UID
    runAsGroup: 1001
    fsGroup: 1001
    seccompProfile:
      type: RuntimeDefault         # Restrict syscalls to safe set

  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false    # Cannot gain more privileges
      readOnlyRootFilesystem: true       # Immutable filesystem
      capabilities:
        drop: ["ALL"]                    # Drop ALL Linux capabilities
        add: ["NET_BIND_SERVICE"]        # Add only what's needed (if port < 1024)
```

**Applied to service accounts:**
```yaml
# Minimal RBAC — only what the app needs
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["myapp-config"]     # Only this specific ConfigMap
  verbs: ["get"]
```

---

### 🟡 Q2. What is DevSecOps and how do you integrate security into a CI/CD pipeline?

**Answer:**

DevSecOps ("shift-left security") integrates security checks throughout the pipeline instead of at the end:

```
Code           → Secrets detection (Gitleaks, GitGuardian)
               → IDE security plugins (SonarLint)
               ↓
Build          → SAST (SonarQube, Semgrep, Checkmarx)
               → Dependency scan (Snyk, OWASP Dependency-Check, Trivy fs)
               ↓
Container      → Image vulnerability scan (Trivy, Clair, Grype)
               → Dockerfile linting (Hadolint)
               → CIS Benchmark check (Dockle)
               ↓
Deploy         → Kubernetes manifest validation (Kyverno, OPA/Gatekeeper)
               → RBAC audit
               → Network policy verification
               ↓
Runtime        → Runtime security (Falco)
               → Compliance scanning (OpenSCAP, kube-bench)
               → Penetration testing (DAST — OWASP ZAP)
```

```yaml
# GitLab CI integration: Trivy + Gitleaks
gitleaks-scan:
  stage: pre-build
  image: zricethezav/gitleaks:latest
  script:
    - gitleaks detect --source=. --exit-code=1

trivy-image-scan:
  stage: post-build
  image: aquasec/trivy:latest
  script:
    - trivy image
        --format json
        --output trivy-report.json
        --exit-code 1
        --severity CRITICAL
        $IMAGE
  artifacts:
    reports:
      container_scanning: trivy-report.json
```

---

## OpenShift SCC & Pod Security

---

### 🟡 Q3. Walk through all the built-in SCCs in OpenShift and their use cases.

**Answer:**

| SCC | Root | Privilege Escalation | Capabilities | Use Case |
|-----|------|---------------------|-------------|---------|
| `restricted-v2` | ❌ | ❌ | Drop ALL | Default for apps — most secure |
| `restricted` | ❌ | ❌ | Drop ALL | Legacy (pre 4.11) — same as restricted-v2 without seccomp |
| `nonroot-v2` | ❌ | ❌ | Drop ALL | Apps that set their own non-root UID |
| `nonroot` | ❌ | ❌ | None | Legacy nonroot |
| `baseline` | ❌ | ❌ | Limited set | Mapped from K8s PodSecurity baseline |
| `anyuid` | ❌ (any UID) | ❌ | None | Legacy apps needing specific UIDs |
| `privileged` | ✅ | ✅ | ALL | Node-level operators, CNI plugins |
| `hostnetwork-v2` | ❌ | ❌ | `NET_BIND_SERVICE` | Pods needing host network namespace |
| `hostaccess` | ❌ | ❌ | None | Pods needing host paths |

```bash
# Identify the minimum SCC needed
oc adm policy scc-subject-review -f my-deployment.yaml
# Output shows: "AllowedBy: restricted-v2" or "DeniedBy: ..." with reason

# Audit: find pods using overly-permissive SCCs
oc get pods --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.metadata.annotations.openshift\.io/scc}{"\n"}{end}' \
  | grep -E 'privileged|anyuid'
```

---

### 🔴 Q4. How do you design a custom SCC for a specific application requirement?

**Answer:**

**Example: An app that needs to bind port 443 (requires `NET_BIND_SERVICE`) but nothing else.**

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: net-bind-only-scc
  annotations:
    kubernetes.io/description: "Allows NET_BIND_SERVICE only — for TLS termination apps"

# Core restrictions
allowPrivilegedContainer: false
allowPrivilegeEscalation: false
defaultAllowPrivilegeEscalation: false

# Capabilities
allowedCapabilities:
- NET_BIND_SERVICE
defaultAddCapabilities: []
requiredDropCapabilities:
- ALL

# User/Group restrictions
runAsUser:
  type: MustRunAsRange          # Use namespace-assigned UID range
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: MustRunAs
  ranges:
  - min: 1000
    max: 65535
supplementalGroups:
  type: RunAsAny

# Volume restrictions
volumes:
- configMap
- secret
- emptyDir
- persistentVolumeClaim
- projected

# Network
allowHostNetwork: false
allowHostPID: false
allowHostIPC: false
allowHostPorts: false

# SecComp
seccompProfiles:
- runtime/default

users: []
groups: []
priority: null
```

```bash
# Grant to service account
oc adm policy add-scc-to-user net-bind-only-scc \
  -z myapp-sa -n my-app

# Verify pod uses correct SCC
oc get pod myapp-xyz \
  -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
```

---

## RBAC

---

### 🟡 Q5. How do you implement least-privilege RBAC for a CI/CD service account?

**Answer:**

A CI/CD service account (e.g., Jenkins or Tekton pipeline SA) needs to deploy to specific namespaces — nothing more.

```yaml
# Service account for CI/CD in ci-cd namespace
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cicd-deployer
  namespace: cicd

---
# Role in each target namespace — only what deploy needs
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer-role
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "patch", "update"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]

---
# Bind the cicd SA (from cicd namespace) to the role in production
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: cicd-deployer
  namespace: cicd               # SA lives in a different namespace
roleRef:
  kind: Role
  name: deployer-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Verify: what can this SA do?
oc auth can-i --list \
  --as=system:serviceaccount:cicd:cicd-deployer \
  -n production

# Confirm it CANNOT delete pods (not in the Role)
oc auth can-i delete pods \
  --as=system:serviceaccount:cicd:cicd-deployer \
  -n production
# Expected: no
```

---

### 🟡 Q6. How do you audit RBAC permissions in a running OpenShift cluster?

**Answer:**

```bash
# 1. Who has cluster-admin?
oc get clusterrolebindings \
  -o json | jq '.items[] | select(.roleRef.name=="cluster-admin") | .subjects'

# 2. List all role bindings in a namespace
oc get rolebindings,clusterrolebindings \
  --all-namespaces \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,ROLE:.roleRef.name,SUBJECTS:.subjects[*].name'

# 3. What can user alice do?
oc auth can-i --list --as=alice -n my-app

# 4. Who can exec into pods? (common privilege escalation path)
oc policy who-can create pods/exec -n my-app
oc policy who-can get secrets -n my-app

# 5. Check for overly-broad wildcards
oc get clusterroles -o json | \
  jq '.items[] | select(.rules[].verbs[]? == "*") | .metadata.name'

# 6. kube-bench (CIS benchmark RBAC audit)
oc apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-openshift.yaml
oc logs job/kube-bench | grep FAIL
```

---

## Network Security

---

### 🟡 Q7. How do you implement NetworkPolicies for micro-segmentation in OpenShift?

**Answer:**

**Default-deny all, then selectively allow — "zero trust networking":**

```yaml
# Step 1: Default deny all ingress and egress in a namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: my-app
spec:
  podSelector: {}      # Applies to ALL pods in the namespace
  policyTypes:
  - Ingress
  - Egress

---
# Step 2: Allow ingress from the Route/HAProxy router only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-router
  namespace: my-app
spec:
  podSelector:
    matchLabels:
      app: myapp
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          network.openshift.io/policy-group: ingress

---
# Step 3: Allow app pods to reach the database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: my-app
spec:
  podSelector:
    matchLabels:
      app: postgres
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: myapp
    ports:
    - protocol: TCP
      port: 5432

---
# Step 4: Allow DNS egress (required for all pods)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: my-app
spec:
  podSelector: {}
  egress:
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53

---
# Step 5: Allow monitoring scrape from Prometheus
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: my-app
spec:
  podSelector:
    matchLabels:
      app: myapp
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: openshift-monitoring
    ports:
    - port: 8080
      protocol: TCP
```

---

### 🟡 Q8. What is an EgressNetworkPolicy (EgressFirewall) in OpenShift?

**Answer:**

**EgressNetworkPolicy** (called `EgressFirewall` in OVN-Kubernetes) controls which external IPs/DNS names pods in a namespace can reach. This enforces outbound network segmentation.

```yaml
# With OVN-Kubernetes (default in OCP 4.12+): EgressFirewall
apiVersion: k8s.ovn.org/v1
kind: EgressFirewall
metadata:
  name: egress-firewall
  namespace: my-app
spec:
  egress:
  # Allow internal services
  - type: Allow
    to:
      cidrSelector: 10.0.0.0/8

  # Allow specific external APIs
  - type: Allow
    to:
      dnsName: api.github.com

  - type: Allow
    to:
      dnsName: registry.redhat.io

  # Block everything else
  - type: Deny
    to:
      cidrSelector: 0.0.0.0/0
```

```bash
# Check egress firewall status
oc get egressfirewall -n my-app
oc describe egressfirewall egress-firewall -n my-app
```

---

## Secrets Management

---

### 🔴 Q9. What are the security risks of using Kubernetes Secrets and how do you mitigate them?

**Answer:**

**Kubernetes Secret risks:**

| Risk | Description |
|------|-------------|
| **Base64 ≠ encryption** | Secrets are base64-encoded by default — anyone who can `get secret` can decode them |
| **etcd at-rest encryption** | Secrets stored unencrypted in etcd unless explicitly configured |
| **RBAC sprawl** | `get secret` in a namespace → access to ALL secrets in that namespace |
| **Env var exposure** | Secrets injected as env vars may appear in `kubectl describe pod` or logs |
| **Git leakage** | Developers accidentally commit Secrets manifests to Git |

**Mitigations:**

```bash
# 1. Enable etcd encryption at rest (OpenShift)
oc get apiserver cluster -o yaml   # Check encryption config
oc edit apiserver cluster
# Set: spec.encryption.type: aescbc  (or aesgcm)
# OpenShift rotates the encryption key via the API server operator
```

```yaml
# 2. Use External Secrets Operator — secrets never stored in Git or etcd long-term
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: my-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
    deletionPolicy: Delete
  data:
  - secretKey: password
    remoteRef:
      key: secret/production/myapp/db
      property: password
```

```yaml
# 3. Mount as file, not env var — files don't appear in pod describe
volumeMounts:
- name: db-secret
  mountPath: /etc/secrets
  readOnly: true
volumes:
- name: db-secret
  secret:
    secretName: db-credentials
    defaultMode: 0400      # Read-only, owner only
```

```bash
# 4. Restrict RBAC: access only specific secrets by name
# 5. Scan Git repos for accidental secrets — gitleaks in pre-commit hook
# 6. Use Vault Agent Injector for zero-secret-in-etcd pattern
```

---

### 🔴 Q10. How do you integrate HashiCorp Vault with OpenShift?

**Answer:**

**Two patterns:**

**Pattern 1: Vault Agent Injector (sidecar):**
```yaml
# Vault injects secrets as files into pod sidecar — nothing stored in etcd
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp-role"
        vault.hashicorp.com/agent-inject-secret-db-creds: "secret/data/production/myapp/db"
        vault.hashicorp.com/agent-inject-template-db-creds: |
          {{- with secret "secret/data/production/myapp/db" -}}
          export DB_PASSWORD="{{ .Data.data.password }}"
          {{- end }}
    spec:
      serviceAccountName: myapp-sa
      # Vault uses the SA's JWT token to authenticate (Kubernetes auth method)
```

**Pattern 2: External Secrets Operator (ESO):**
```bash
# Install Vault operator + configure K8s auth
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/run/secrets/kubernetes.io/serviceaccount/ca.crt

vault policy write myapp-policy - <<EOF
path "secret/data/production/myapp/*" {
  capabilities = ["read"]
}
EOF

vault write auth/kubernetes/role/myapp-role \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=my-app \
  policies=myapp-policy \
  ttl=1h
```

---

## Image Security & Supply Chain

---

### 🟡 Q11. How do you enforce that only approved images run in OpenShift?

**Answer:**

**Layer 1 — Image signing with cosign:**
```bash
# Generate signing keys (stored as K8s secret)
cosign generate-key-pair k8s://cosign-keys/cosign

# Sign image after build in CI pipeline
cosign sign \
  --key k8s://cosign-keys/cosign \
  ${IMAGE}@${DIGEST}
```

**Layer 2 — Kyverno policy to enforce signed images:**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  rules:
  - name: verify-signature
    match:
      any:
      - resources:
          kinds: [Pod]
          namespaces: [production, staging]
    verifyImages:
    - imageReferences:
      - "image-registry.openshift-image-registry.svc:5000/production/*"
      attestors:
      - entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              ...
              -----END PUBLIC KEY-----
```

**Layer 3 — AllowedRegistries policy:**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  rules:
  - name: allow-only-approved-registries
    match:
      any:
      - resources:
          kinds: [Pod]
    validate:
      message: "Images must come from approved registries"
      pattern:
        spec:
          containers:
          - image: "image-registry.openshift-image-registry.svc:5000/* | registry.redhat.io/* | registry.access.redhat.com/*"
```

**Layer 4 — OpenShift image policy (built-in):**
```bash
# Prevent use of :latest tag in production
oc apply -f - <<EOF
apiVersion: config.openshift.io/v1
kind: Image
metadata:
  name: cluster
spec:
  allowedRegistriesForImport:
  - domainName: image-registry.openshift-image-registry.svc:5000
    insecure: false
  - domainName: registry.redhat.io
  registrySources:
    blockedRegistries:
    - docker.io         # Block Docker Hub in production
EOF
```

---

### 🟡 Q12. What is an SBOM and why is it important for container security?

**Answer:**

**SBOM (Software Bill of Materials)** is a complete list of all components, libraries, and dependencies inside a container image — the "ingredients list" for software.

**Why it matters:**
- When a new CVE is published (e.g., Log4Shell), you can instantly find which images contain the vulnerable library.
- Required by US Executive Order 14028 on Cybersecurity for federal systems.
- Enables faster incident response — no need to scan every image manually.

**Generating SBOM in CI pipeline:**
```bash
# Using Syft (generates CycloneDX or SPDX format)
syft ${IMAGE} -o cyclonedx-json > sbom.json

# Attach SBOM to image (stored alongside in registry)
cosign attach sbom --sbom sbom.json ${IMAGE}

# Verify SBOM presence later
cosign download sbom ${IMAGE}
```

**Trivy scans against SBOM:**
```bash
# Generate and use SBOM for scanning
trivy image --format cyclonedx ${IMAGE} > sbom.cdx.json
trivy sbom ./sbom.cdx.json   # Scan the SBOM for vulnerabilities
```

---

## Compliance & Audit

---

### 🟡 Q13. What compliance frameworks are commonly required in containerized environments?

**Answer:**

| Framework | Focus | How OpenShift helps |
|-----------|-------|---------------------|
| **CIS Kubernetes Benchmark** | K8s hardening (API server flags, etcd, kubelet) | Compliance Operator with CIS profile |
| **NIST SP 800-190** | Container security guidelines | SCC, image signing, network policies |
| **PCI DSS** | Payment card data — network segmentation, encryption | NetworkPolicy, encrypted secrets, audit logs |
| **HIPAA** | Healthcare data — access control, audit trails | RBAC, audit logging, encryption at rest |
| **SOC 2 Type II** | Security, availability, confidentiality | Access control, monitoring, incident management |
| **FedRAMP** | US government cloud — strict controls | OpenShift on GovCloud, FIPS mode |

**OpenShift Compliance Operator:**
```bash
# Install Compliance Operator from OperatorHub
# Runs OpenSCAP scans against CIS/NIST/PCI profiles

# Apply a scan setting
oc apply -f - <<EOF
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSettingBinding
metadata:
  name: cis-compliance
  namespace: openshift-compliance
spec:
  profiles:
  - apiGroup: compliance.openshift.io/v1alpha1
    kind: Profile
    name: ocp4-cis
  - apiGroup: compliance.openshift.io/v1alpha1
    kind: Profile
    name: ocp4-cis-node
  settingsRef:
    apiGroup: compliance.openshift.io/v1alpha1
    kind: ScanSetting
    name: default
EOF

# View results
oc get compliancecheckreports -n openshift-compliance
oc get compliancecheckresult -n openshift-compliance | grep FAIL
```

---

### 🔴 Q14. How do you enable and use Kubernetes/OpenShift audit logs?

**Answer:**

Audit logs record every request made to the API server — who did what, when, and whether it was allowed.

**OpenShift audit policy levels:**
```
None     → Don't log this request
Metadata → Log method, URL, user, status (no body)
Request  → Log metadata + request body
RequestResponse → Log everything including response body
```

```yaml
# Custom audit policy
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Don't log routine read-only requests
- level: None
  verbs: ["get", "list", "watch"]
  resources:
  - group: ""
    resources: ["events"]

# Log all secret access at RequestResponse level
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]

# Log all authentication failures
- level: Metadata
  nonResourceURLs: ["/api*"]
  userGroups: ["system:unauthenticated"]

# Default: log metadata for everything else
- level: Metadata
```

```bash
# In OpenShift, audit logs are in each API server pod
oc adm node-logs --role=master --path=openshift-apiserver/audit.log | head -50

# Parse with jq
oc adm node-logs --role=master --path=openshift-apiserver/audit.log | \
  jq 'select(.verb=="delete" and .objectRef.resource=="secrets")'

# Who accessed secret X?
oc adm node-logs --role=master --path=openshift-apiserver/audit.log | \
  jq 'select(.objectRef.resource=="secrets" and .objectRef.name=="db-password")'
```

---

### 🔴 Q15. How do you use Falco for runtime security in OpenShift?

**Answer:**

**Falco** is a CNCF runtime security tool that monitors Linux system calls to detect threats at runtime.

```yaml
# Falco DaemonSet on OpenShift (needs privileged SCC)
# Install via Helm
helm install falco falcosecurity/falco \
  --namespace falco-system \
  --set falco.jsonOutput=true \
  --set falco.logLevel=info

# Grant privileged SCC to Falco SA
oc adm policy add-scc-to-user privileged \
  -z falco -n falco-system
```

**Custom Falco rules for container environments:**
```yaml
# falco-rules.yaml
- rule: Shell in container
  desc: Detect shell spawned in a container
  condition: >
    spawned_process and
    container and
    shell_procs and
    not user_known_shell_spawn_activities
  output: >
    Shell in container
    (user=%user.name container=%container.name
     image=%container.image.repository
     command=%proc.cmdline)
  priority: WARNING
  tags: [container, shell]

- rule: Write to /etc in container
  desc: Detect write to /etc — possible config tampering
  condition: >
    open_write and
    container and
    fd.name startswith /etc
  output: >
    Write to /etc
    (user=%user.name file=%fd.name container=%container.name)
  priority: ERROR
```

```bash
# View Falco alerts in real-time
kubectl logs -n falco-system -l app.kubernetes.io/name=falco -f | \
  jq 'select(.priority=="CRITICAL" or .priority=="ERROR")'
```

---

### 🟡 Q16. How do you handle certificate management in OpenShift?

**Answer:**

**Internal certificates (cluster):**
OpenShift manages its own PKI — the ingress certificate, API server cert, etcd certs, and service-serving certificates are all auto-rotated by operators.

```bash
# View cert expiry dates
oc get secret -n openshift-ingress-operator \
  router-ca -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates

# Rotate ingress certificate
oc delete secret router-certs-default -n openshift-ingress
# Operator auto-recreates it

# Replace with custom wildcard cert
oc create secret tls custom-certs \
  --cert=wildcard.crt \
  --key=wildcard.key \
  -n openshift-ingress

oc patch ingresscontroller.operator default \
  -n openshift-ingress-operator \
  --type=merge \
  -p '{"spec":{"defaultCertificate":{"name":"custom-certs"}}}'
```

**Application certificates with cert-manager:**
```yaml
# cert-manager Certificate for an app
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-tls
  namespace: my-app
spec:
  secretName: myapp-tls-secret
  duration: 2160h          # 90 days
  renewBefore: 360h        # Renew 15 days before expiry
  dnsNames:
  - myapp.apps.mycluster.example.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

---

### 🔴 Q17. How do you enforce pod security policies organization-wide using Kyverno or OPA/Gatekeeper?

**Answer:**

**Kyverno** is a Kubernetes-native policy engine (operates as admission webhook).

```yaml
# Cluster-wide policy: require readOnlyRootFilesystem
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-readonly-rootfs
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: check-readonly-rootfs
    match:
      any:
      - resources:
          kinds: [Pod]
          namespaces: [production, staging]
    validate:
      message: "readOnlyRootFilesystem must be true in production"
      pattern:
        spec:
          containers:
          - =(securityContext):
              readOnlyRootFilesystem: true

---
# Policy: require resource requests and limits on all containers
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-resources
    match:
      any:
      - resources:
          kinds: [Pod]
    validate:
      message: "CPU and memory requests/limits are required"
      pattern:
        spec:
          containers:
          - resources:
              requests:
                memory: "?*"
                cpu: "?*"
              limits:
                memory: "?*"

---
# Mutating policy: automatically add labels
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-mandatory-labels
spec:
  rules:
  - name: add-labels
    match:
      any:
      - resources:
          kinds: [Deployment, StatefulSet]
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            +(managed-by): "platform-team"
            +(environment): "{{request.namespace}}"
```

```bash
# Check policy reports
kubectl get policyreport -n my-app
kubectl get clusterpolicyreport
```

---

### 🟡 Q18. What is FIPS mode in OpenShift and when is it required?

**Answer:**

**FIPS 140-2/3** (Federal Information Processing Standard) is a US government standard for cryptographic modules. It restricts algorithms to FIPS-approved ones (e.g., AES, SHA-2, RSA) and disables non-approved ones (MD5, DES, RC4).

**When required:**
- US Federal government workloads (FedRAMP)
- Healthcare (HIPAA for some interpretations)
- Defense/intelligence contracts
- Payment systems (some PCI interpretations)

**Enabling FIPS in OpenShift (must be set at install time — cannot be enabled post-install):**
```yaml
# install-config.yaml
apiVersion: v1
fips: true
# ... rest of install config
```

```bash
# Verify FIPS is enabled after install
oc get cm cluster-config-v1 -n kube-system -o jsonpath='{.data.install-config}' | grep fips
# On a node:
oc debug node/worker-1 -- fips-mode-setup --check
```

**Implications:**
- Container images must use FIPS-compliant crypto libraries (OpenSSL in FIPS mode or Go's FIPS build tags).
- Red Hat UBI images are FIPS-compatible.
- Some third-party images may break in FIPS mode.

---

> **Next:** [Monitoring & Troubleshooting →](./04-Monitoring-Troubleshooting.md)