# 🔵 Module 6 — Scenario-Based Questions

> **JD Alignment:** Real-world production problems spanning all JD areas — CI/CD, OpenShift/K8s operations, security, monitoring, automation, and cross-team collaboration.

> [← Infrastructure & Automation](./05-Infrastructure-Automation.md) | [README](./README.md) | [Behavioral →](./07-Behavioral.md)

---

## How to Use This Module

These questions simulate real interview scenarios. **Before reading the answer:**
1. Speak your approach out loud for 2–3 minutes.
2. Think: what would you check first? What commands would you run?
3. Then compare with the full answer.

---

## Scenarios

---

### 🔵 Scenario 1: Production Deployment is Failing — Diagnose and Fix

*"Your team just deployed a new version of the payment service to production. The deployment is stuck, 50% of pods are in CrashLoopBackOff. Customers are being impacted. What do you do?"*

**Answer:**

**Immediate actions (0–2 min) — stop the bleeding:**
```bash
# 1. Immediately rollback — restore service first, investigate second
oc rollout undo deployment/payment-service -n production
oc rollout status deployment/payment-service -n production
# Confirm old pods are Running and serving traffic
```

**Investigation after service is restored (2–15 min):**
```bash
# 2. Get crash logs from the failed pods (they may still exist briefly)
oc get pods -n production | grep payment
oc logs payment-service-<new-hash>-<pod> --previous -n production

# Common findings:
# Exit code 1: Application failed to start — check logs for config errors
# Exit code 137: OOMKilled — memory limit too low for new version
# Exit code 132: SIGILL — binary incompatibility

# 3. Check events
oc get events -n production --sort-by='.lastTimestamp' | grep payment

# 4. Describe failing pod
oc describe pod payment-service-<new-hash>-<pod> -n production
```

**Root cause analysis:**
```bash
# Check what changed between old and new version
oc rollout history deployment/payment-service -n production
oc rollout history deployment/payment-service --revision=5 -n production

# Compare env vars (maybe a new required env var is missing)
oc get deployment payment-service -n production -o yaml | grep -A 30 env

# Test the new image locally / in a debug pod
oc run test-pod --rm -it \
  --image=image-registry.openshift-image-registry.svc:5000/production/payment-service:v2.0 \
  --overrides='{"spec":{"serviceAccountName":"payment-sa"}}' \
  -- /bin/sh
```

**Post-incident:**
```bash
# Fix the root cause (e.g., add missing env var)
oc set env deployment/payment-service NEW_VAR=value -n production

# Re-deploy with monitoring
oc set image deployment/payment-service app=payment-service:v2.0 -n production
watch oc get pods -n production    # Watch rollout

# Add automated smoke test to CI pipeline to prevent recurrence
```

**Communication:** Notify stakeholders immediately. Open incident ticket. Post a post-mortem in 48h.

---

### 🔵 Scenario 2: Pipeline Fails Only in Production — Not in Staging

*"Your Jenkins pipeline succeeds in staging but fails in production at the deploy step with: 'Error from server (Forbidden): deployments.apps is forbidden'. How do you debug this?"*

**Answer:**

**Diagnosis:**
```bash
# The error is RBAC-related — the CI service account lacks permission in production

# 1. Identify which service account the pipeline uses
# In Jenkins pipeline, check: withCredentials, oc login, SA mounting

# 2. Check what SA is used in production
oc get sa -n production | grep jenkins
oc get rolebindings -n production | grep jenkins

# 3. Check exact permissions
oc auth can-i update deployments \
  --as=system:serviceaccount:ci-cd:jenkins-sa \
  -n production
# Expected current result: no

# Compare with staging
oc auth can-i update deployments \
  --as=system:serviceaccount:ci-cd:jenkins-sa \
  -n staging
# Expected: yes

# 4. List what the SA CAN do in production
oc auth can-i --list \
  --as=system:serviceaccount:ci-cd:jenkins-sa \
  -n production
```

**Root cause:** The RoleBinding for the CI service account exists in staging but was never created in production.

**Fix:**
```yaml
# Create the missing RoleBinding in production
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-deployer
  namespace: production
subjects:
- kind: ServiceAccount
  name: jenkins-sa
  namespace: ci-cd
roleRef:
  kind: Role
  name: deployer-role
  apiGroup: rbac.authorization.k8s.io
```

**Prevention:**
```bash
# Add this to onboarding automation — create RoleBindings in ALL target namespaces
# Add RBAC check to pipeline before deploy:
oc auth can-i update deployments \
  --as=system:serviceaccount:ci-cd:jenkins-sa \
  -n ${TARGET_NS} || {
    echo "ERROR: Missing RBAC in ${TARGET_NS}. Notify platform team."
    exit 1
}
```

---

### 🔵 Scenario 3: Node is Under Memory Pressure — Pods Being Evicted

*"You get a PagerDuty alert: multiple pods in the production namespace have been evicted. You check and see a node has MemoryPressure=True. Walk through your response."*

**Answer:**

```bash
# 1. Check which node is under pressure
oc get nodes
oc describe node worker-3 | grep -A 5 "Conditions:"
# MemoryPressure: True

# 2. Check what's running on this node
oc get pods --all-namespaces \
  -o wide | grep worker-3 | sort -k5

# 3. Check actual memory usage
oc adm top node worker-3
oc adm top pods --all-namespaces --sort-by=memory | head -20

# 4. Find the memory hog
# OOMKilled pods recently?
oc get events --all-namespaces \
  --sort-by='.lastTimestamp' | grep -i "oom\|evict\|memory" | tail -30

# 5. Identify pod consuming most memory
oc adm top pods -n production \
  --sort-by=memory | head -10

# 6. Immediate remediation options:

# Option A: Evict the heavy pod gracefully from this node
oc adm drain worker-3 --ignore-daemonsets \
  --pod-selector="app=memory-heavy-app" \
  --delete-emptydir-data

# Option B: Temporarily scale up to add capacity
oc scale deployment/memory-heavy-app --replicas=5 -n production
# Other replicas land on other nodes, relieving pressure

# Option C: If the app has a memory leak — restart it
oc rollout restart deployment/memory-heavy-app -n production

# 7. Root cause: check if limits are too low
oc get deployment memory-heavy-app -n production \
  -o jsonpath='{.spec.template.spec.containers[0].resources}'

# 8. Set proper limits
oc set resources deployment/memory-heavy-app \
  --requests=memory=512Mi \
  --limits=memory=1Gi
```

**Prevention:**
- Set `LimitRange` defaults in the namespace.
- Create a `MemoryPressure` alert for each node.
- PDB to prevent all pods from being on the same node.

---

### 🔵 Scenario 4: Security Incident — Suspicious Pod Activity Detected

*"Falco alerts: 'Shell spawned in production container'. How do you respond?"*

**Answer:**

**Phase 1 — Contain (0–5 min):**
```bash
# 1. Identify the pod
ALERT_POD="payment-service-abc123-xyz"
ALERT_NS="production"

# 2. Take a snapshot — capture forensic data before touching anything
oc describe pod ${ALERT_POD} -n ${ALERT_NS} > /tmp/pod-snapshot.txt
oc logs ${ALERT_POD} -n ${ALERT_NS} > /tmp/pod-logs.txt
oc get pod ${ALERT_POD} -n ${ALERT_NS} -o yaml > /tmp/pod-full.yaml

# 3. Isolate: apply a deny-all NetworkPolicy to this pod
kubectl label pod ${ALERT_POD} -n ${ALERT_NS} quarantine=true
oc apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: quarantine-pod
  namespace: ${ALERT_NS}
spec:
  podSelector:
    matchLabels:
      quarantine: "true"
  policyTypes:
  - Ingress
  - Egress
  # No ingress/egress rules = deny all
EOF

# 4. Cordon the node if you suspect node compromise
oc adm cordon $(oc get pod ${ALERT_POD} -n ${ALERT_NS} -o jsonpath='{.spec.nodeName}')
```

**Phase 2 — Investigate (5–30 min):**
```bash
# 5. What process was spawned?
# Check Falco alert details — it includes the exact command

# 6. Check audit logs for the pod's service account
oc adm node-logs --role=master \
  --path=openshift-apiserver/audit.log | \
  jq "select(.user.username | contains(\"${ALERT_NS}:${ALERT_POD}\"))" | head -50

# 7. Check for new files created
oc exec ${ALERT_POD} -n ${ALERT_NS} -- find / \
  -newer /proc/1 -not -path /proc -not -path /sys 2>/dev/null

# 8. Check network connections from the pod (if not yet isolated)
oc exec ${ALERT_POD} -n ${ALERT_NS} -- ss -tulnp
oc exec ${ALERT_POD} -n ${ALERT_NS} -- cat /etc/hosts
```

**Phase 3 — Remediate:**
```bash
# 9. Kill the compromised pod (new pod starts clean from image)
oc delete pod ${ALERT_POD} -n ${ALERT_NS}

# 10. Force restart all pods in the deployment (in case others are affected)
oc rollout restart deployment/payment-service -n ${ALERT_NS}

# 11. Review and tighten SCC
oc adm policy scc-subject-review -f payment-pod.yaml
# Ensure readOnlyRootFilesystem: true to prevent file writes

# 12. Remove quarantine NetworkPolicy after confirming new pods are clean
oc delete networkpolicy quarantine-pod -n ${ALERT_NS}
```

**Post-incident:** Conduct forensics on the captured snapshot. Determine how the shell was spawned (exec from CI/CD? RCE exploit?). File security incident report.

---

### 🔵 Scenario 5: Cluster Upgrade Gone Wrong

*"During a cluster upgrade from 4.13 to 4.14, the upgrade stalls at 60% and cluster operators show Degraded=True. How do you troubleshoot?"*

**Answer:**

```bash
# 1. Check upgrade status
oc get clusterversion
oc describe clusterversion version
# Look for: "Progressing: True" message — what's blocking it?

# 2. Find degraded cluster operators
oc get co | grep -v "True.*False.*False"
# Output shows which operators are not healthy

# 3. Investigate the degraded operator
oc describe co authentication
# Look for Events and Status conditions

# 4. Check the operator's pods
oc get pods -n openshift-authentication
oc logs -n openshift-authentication oauth-openshift-<pod> --previous

# 5. Common stall causes and fixes:

# Cause A: PodDisruptionBudget blocking node drain
oc get pdb --all-namespaces
# Check: minAvailable == current replicas → can't drain
# Temporary fix: lower minAvailable during upgrade window
oc patch pdb my-pdb -n my-app -p '{"spec":{"minAvailable":1}}'

# Cause B: Unhealthy etcd member
oc get etcd -o wide
oc rsh -n openshift-etcd etcd-master-0 -- \
  etcdctl member list --write-out=table

# Cause C: Node not draining (pod stuck in Terminating)
oc get pods --all-namespaces | grep Terminating
oc delete pod stuck-pod -n my-app --force --grace-period=0

# Cause D: Cluster operator has a broken dependency
oc logs deployment/cluster-version-operator -n openshift-cluster-version
oc get co network -o yaml | grep -A 20 conditions

# 6. If safe to force-proceed (rare, only if you understand the risk)
# First: resolve the underlying issue, then let CVO retry automatically
# CVO retries every few minutes — don't force if unsure

# 7. Contact Red Hat Support
oc adm must-gather
# Upload to Red Hat Customer Portal → open a support case
```

---

### 🔵 Scenario 6: 100% CPU Throttling on a Critical Microservice

*"Grafana shows your checkout service is CPU-throttled 90% of the time. Latency has spiked to 3s P99. What do you do?"*

**Answer:**

**Understanding CPU throttling:**
```
CPU request = scheduler guarantee (node allocates this)
CPU limit = hard cap — kernel throttles the container when it tries to exceed this
Throttling % = time spent waiting due to CPU limit / total time
```

```bash
# 1. Confirm via PromQL
# container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total * 100
# If this > 50% → heavily throttled

# 2. Check current limits
oc get deployment checkout -n production \
  -o jsonpath='{.spec.template.spec.containers[0].resources}'

# Example: requests=100m, limits=200m — but actual usage is 400m → heavily throttled

# 3. Check actual CPU usage
oc adm top pods -n production | grep checkout
# Shows: 180m used (close to 200m limit)
```

**Immediate fix:**
```bash
# Increase CPU limit
oc set resources deployment/checkout \
  --requests=cpu=200m \
  --limits=cpu=1 \
  -n production

# Watch latency recover in Grafana
```

**Deeper optimization:**
```bash
# Option 1: Remove CPU limit entirely (Kubernetes best practice for latency-sensitive apps)
# With CPU limit removed, container can burst beyond request during idle periods
# Node still protected by requests

oc patch deployment checkout -n production --type='json' -p='[
  {"op": "remove", "path": "/spec/template/spec/containers/0/resources/limits/cpu"}
]'

# Option 2: Use VPA to auto-right-size
kubectl apply -f vpa-checkout.yaml
# VPA observes and recommends: set request=400m, limit=800m

# Option 3: Profile the app
# High CPU usage in checkout could be a code issue (N+1 queries, inefficient algorithms)
# Add CPU profiling: node --prof / pprof / async-profiler

# Option 4: Scale horizontally (HPA on CPU)
oc autoscale deployment/checkout \
  --min=3 --max=10 \
  --cpu-percent=60 \
  -n production
```

---

### 🔵 Scenario 7: Route is Returning 503 Service Unavailable

*"Your application Route returns HTTP 503 intermittently in production. Debug it end-to-end."*

**Answer:**

```bash
# 503 from OpenShift Route = HAProxy can't connect to any backend pod

# Step 1: Check pods are running and ready
oc get pods -n production -l app=myapp
# All Running? Check READY column — 1/1 required

# Step 2: Check Service endpoints (most common cause: 0 endpoints)
oc get endpoints myapp-svc -n production
# If ENDPOINTS shows <none> → label selector mismatch

# Fix label selector mismatch:
oc describe svc myapp-svc | grep Selector
oc get pods -l app=myapp -n production --show-labels
# Selectors must match

# Step 3: Check Route target
oc describe route myapp -n production
# Verify: "to: Service myapp-svc weight=100"
# Verify: port matches

# Step 4: Check readiness probe — pods might be Running but not Ready
oc describe pod myapp-abc -n production | grep -A 10 Readiness
oc get events -n production | grep Readiness

# If readiness probe is too strict → adjust
oc patch deployment myapp -n production --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/readinessProbe/failureThreshold",
    "value": 5
  }
]'

# Step 5: Check HAProxy router logs
oc logs -n openshift-ingress \
  -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default \
  --tail=100 | grep "503\|myapp"

# Step 6: Check if router has enough capacity
oc get pods -n openshift-ingress -o wide
oc adm top pods -n openshift-ingress

# Step 7: Test direct pod access (bypass Service and Route)
POD_IP=$(oc get pod myapp-abc -n production -o jsonpath='{.status.podIP}')
oc run debug --rm -it --image=curlimages/curl -- curl http://${POD_IP}:8080/health

# Step 8: Check if issue is connection reset during rolling deployment
# Add preStop hook + terminationGracePeriodSeconds to drain connections
```

---

### 🔵 Scenario 8: Team Wants Faster Feedback — Optimize CI Pipeline from 45 min to < 15 min

*"The development team says the CI pipeline takes 45 minutes. They're losing productivity. How do you analyze and optimize it?"*

**Answer:**

**Step 1 — Profile the pipeline:**
```bash
# Collect timing data for each stage (Jenkins: Blue Ocean view)
# Or instrument with timestamps:
time npm ci                    # How long is npm install?
time npm test                  # How long are tests?
time docker build .            # How long is image build?
```

**Step 2 — Identify biggest bottlenecks** (typical findings):

| Stage | Before | After | Technique |
|-------|--------|-------|-----------|
| `npm install` | 8 min | 30 sec | Cache `node_modules` in PVC or layer cache |
| Unit tests | 12 min | 3 min | Parallelize test suites |
| Image build | 15 min | 2 min | Layer caching + smaller base image |
| Security scan | 8 min | 2 min | Scan only changed layers (incremental) |
| Deploy | 5 min | 1 min | Pre-pull images, parallel rollout |

**Optimization techniques:**

```groovy
// 1. Parallel test execution (Jenkins)
stage('Tests') {
    parallel {
        'Unit Tests': {
            sh 'npm run test:unit'
        },
        'Integration Tests': {
            sh 'npm run test:integration'
        },
        'Linting': {
            sh 'npm run lint'
        }
    }
}
```

```yaml
# 2. GitLab CI parallel matrix
test:
  parallel:
    matrix:
    - TEST_SUITE: [unit, integration, e2e, security]
  script:
    - npm run test:$TEST_SUITE
```

```dockerfile
# 3. Dockerfile layer caching — install dependencies before copying source
FROM node:18-alpine
WORKDIR /app

# Dependencies layer (only rebuilds if package.json changes)
COPY package*.json ./
RUN npm ci --only=production

# Source layer (rebuilds on any source change)
COPY . .
CMD ["node", "server.js"]
```

```yaml
# 4. Kubernetes pipeline pod with PVC for cache
spec:
  volumes:
  - name: npm-cache
    persistentVolumeClaim:
      claimName: npm-cache-pvc    # Shared across builds
  containers:
  - name: builder
    volumeMounts:
    - name: npm-cache
      mountPath: /root/.npm
```

```bash
# 5. Buildah with layer cache registry
buildah bud \
  --cache-from=registry/myapp-cache:latest \
  --cache-to=registry/myapp-cache:latest \
  -t myapp:${SHA} .
```

**Result:** 45 min → 12 min through parallelization + caching.

---

### 🔵 Scenario 9: Multi-Tenant Namespace Isolation — Developer Complaint

*"A developer reports they can curl pods in another team's namespace from their own pod. How do you fix this without breaking legitimate communication?"*

**Answer:**

**Diagnosis:**
```bash
# Verify the issue
oc exec -it myapp-pod -n team-alpha -- \
  curl http://team-beta-service.team-beta.svc.cluster.local
# If this succeeds → no NetworkPolicy isolation

oc get networkpolicies -n team-alpha   # Likely: empty
oc get networkpolicies -n team-beta    # Likely: empty
```

**Fix — apply micro-segmentation without breaking monitoring/pipelines:**

```yaml
# Step 1: Default deny all inter-namespace traffic
# Apply to BOTH namespaces
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: team-beta       # Protect team-beta
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}        # Allow from pods IN SAME namespace only

---
# Step 2: Allow monitoring scrape (don't break Prometheus)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: team-beta
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: openshift-monitoring
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: openshift-user-workload-monitoring

---
# Step 3: Allow Route/HAProxy traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-router
  namespace: team-beta
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          network.openshift.io/policy-group: ingress

---
# Step 4: Allow explicit cross-namespace communication (if team-alpha needs to call team-beta API)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-team-alpha
  namespace: team-beta
spec:
  podSelector:
    matchLabels:
      app: team-beta-public-api    # Only specific service
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: team-alpha
      podSelector:
        matchLabels:
          app: team-alpha-consumer  # Only specific pod
    ports:
    - port: 8080
```

**Verify fix:**
```bash
oc exec -it myapp-pod -n team-alpha -- \
  curl http://team-beta-service.team-beta.svc.cluster.local
# Expected: connection refused / timeout → isolation working

# Test monitoring still works
oc exec -it prometheus-0 -n openshift-user-workload-monitoring -- \
  curl http://team-beta-service.team-beta.svc.cluster.local:8080/metrics
# Expected: metrics response → monitoring still works
```

---

### 🔵 Scenario 10: Automated Compliance Audit Request

*"Management requests a compliance report showing all production workloads comply with the company security policy: no root containers, all images from approved registries, all containers have resource limits. How do you automate this?"*

**Answer:**

```bash
# Option 1: Manual audit commands (for one-time check)

# Check 1: Pods running as root
oc get pods --all-namespaces \
  -o jsonpath='{range .items[?(@.spec.securityContext.runAsUser==0)]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{end}'

# Check 2: Pods using unapproved registries
oc get pods --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}' | \
  grep -v "image-registry.openshift-image-registry\|registry.redhat.io"

# Check 3: Containers without resource limits
oc get pods --all-namespaces \
  -o json | jq -r '
    .items[] |
    .metadata.namespace as $ns |
    .metadata.name as $pod |
    .spec.containers[] |
    select(.resources.limits == null or .resources.limits == {}) |
    [$ns, $pod, .name] | join("\t")
  '
```

```yaml
# Option 2: Kyverno PolicyReport (continuous, automated)
# Kyverno runs in audit mode and generates PolicyReports

# View compliance report
kubectl get policyreport -n production
kubectl get clusterpolicyreport

# Detailed view
kubectl get policyreport -n production -o json | jq '
  .results[] |
  select(.result == "fail") |
  {policy: .policy, resource: .resources[0].name, message: .message}
'
```

```bash
# Option 3: kube-bench for CIS benchmark compliance
oc apply -f kube-bench-job-openshift.yaml
oc logs job/kube-bench | grep -E "^\[FAIL\]" | sort | uniq

# Option 4: OpenShift Compliance Operator
oc get compliancecheckresult -n openshift-compliance \
  -l compliance.openshift.io/check-status=FAIL \
  -o custom-columns='RULE:.metadata.name,SEVERITY:.metadata.labels.severity,DESC:.description'
```

```bash
# Scheduled reporting: export to CSV for management
#!/bin/bash
# compliance-report.sh — runs daily, sends report to management

DATE=$(date +%Y-%m-%d)
REPORT_FILE="compliance-report-${DATE}.csv"

echo "Namespace,Pod,Container,Issue" > ${REPORT_FILE}

# No resource limits
oc get pods --all-namespaces -o json | jq -r '
  .items[] |
  .metadata.namespace as $ns |
  .metadata.name as $pod |
  .spec.containers[] |
  select(.resources.limits == null) |
  [$ns, $pod, .name, "Missing resource limits"] | @csv
' >> ${REPORT_FILE}

# Email or upload to S3
aws s3 cp ${REPORT_FILE} s3://compliance-reports/
curl -X POST "${SLACK_WEBHOOK}" \
  -d "{\"text\":\"Daily compliance report uploaded: ${REPORT_FILE}\"}"
```

---

### 🔵 Scenario 11: Onboarding a Development Team — From Zero to CI/CD in a Day

*"A new development team joins. They have a Node.js microservice in GitHub. They need a complete setup: namespace, CI/CD, monitoring, and GitOps by end of day. Walk me through your plan."*

**Answer:**

```bash
# Morning: Infrastructure (1–2 hours)

# 1. Run onboarding Ansible playbook
ansible-playbook onboard-team.yml \
  -e team_name="inventory-team" \
  -e app_namespace="inventory" \
  -e "team_admins=['alice','bob']"

# This creates:
# - Project: inventory
# - ResourceQuota + LimitRange
# - Service accounts: app-sa, pipeline-sa
# - RBAC: alice/bob as admin
# - NetworkPolicies: deny-others, allow-router, allow-monitoring
# - SCC grant: restricted-v2 (default, no change needed)

# Midmorning: CI/CD Setup (1–2 hours)

# 2. Create BuildConfig for S2I
oc new-app nodejs:18-ubi8~https://github.com/myorg/inventory-service.git \
  --name=inventory-service \
  --env=NODE_ENV=production \
  -n inventory

# 3. Watch first build
oc logs -f bc/inventory-service -n inventory

# 4. Expose via Route
oc expose svc/inventory-service -n inventory
oc get route -n inventory

# 5. Set up GitLab CI runner (or Jenkins config) with:
# - OCP token for pipeline-sa
# - Image push permissions
oc policy add-role-to-user registry-editor \
  -z pipeline-sa -n inventory

# Afternoon: GitOps + Monitoring (2 hours)

# 6. Set up ArgoCD Application
oc apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: inventory-service
  namespace: openshift-gitops
spec:
  source:
    repoURL: https://github.com/myorg/gitops-config.git
    path: apps/inventory-service/overlays/production
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: inventory
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

# 7. Enable user workload monitoring for the namespace
oc label namespace inventory openshift.io/user-monitoring=true

# 8. Create ServiceMonitor (team adds /metrics to their app)
oc apply -f serviceMonitor-inventory.yaml -n inventory

# 9. Import standard Grafana dashboard
oc apply -f configmap-dashboard.yaml -n inventory

# 10. Add basic alert
oc apply -f prometheusrule-inventory.yaml -n inventory

# End of day: Handoff
# - Send team: cluster URL, namespace, oc login command
# - Share runbook link
# - Confirm first automated deploy works via GitOps
```

---

### 🔵 Scenario 12: Stateful Database Migration — Zero Downtime

*"You need to migrate a PostgreSQL database StatefulSet from one StorageClass to another (gp2 → gp3) without application downtime. How do you do it?"*

**Answer:**

```bash
# Problem: PVCs cannot change StorageClass in place
# Strategy: Blue/Green StatefulSet migration

# Step 1: Scale up a new StatefulSet with the new StorageClass
# (Run alongside existing — use different service for new SS)

# Step 2: Stream data from old to new using pg_basebackup or logical replication
# For PostgreSQL: use logical replication slot

# Connect to OLD postgres
oc exec -it postgres-old-0 -n my-app -- psql -U postgres -c \
  "SELECT pg_create_logical_replication_slot('migration_slot', 'pgoutput');"

oc exec -it postgres-old-0 -n my-app -- psql -U postgres -c \
  "CREATE PUBLICATION migration_pub FOR ALL TABLES;"

# On NEW postgres: create subscription
oc exec -it postgres-new-0 -n my-app -- psql -U postgres -c \
  "CREATE SUBSCRIPTION migration_sub
   CONNECTION 'host=postgres-old.my-app.svc port=5432 user=postgres dbname=mydb'
   PUBLICATION migration_pub
   WITH (slot_name='migration_slot');"

# Step 3: Monitor lag — wait for replication to catch up
oc exec -it postgres-old-0 -- psql -U postgres -c \
  "SELECT pg_current_wal_lsn() - confirmed_flush_lsn AS lag
   FROM pg_replication_slots WHERE slot_name='migration_slot';"
# Wait until lag = 0

# Step 4: Switch over (brief write pause, seconds only)
# a) Set app to read-only or brief maintenance
# b) Wait for 0 lag
# c) Drop subscription on new (makes it primary)
# d) Update Service selector to point to new StatefulSet
oc patch svc postgres -n my-app \
  -p '{"spec":{"selector":{"app":"postgres-new"}}}'

# Step 5: Verify and clean up
# Test app connectivity, then remove old StatefulSet + PVCs after 24h observation
oc delete statefulset postgres-old -n my-app
oc delete pvc data-postgres-old-0 -n my-app
```

---

### 🔵 Scenario 13: CI/CD Pipeline Security Audit Finding

*"A security audit finds your CI/CD pipeline service account has cluster-admin. How do you remediate this while keeping pipelines working?"*

**Answer:**

```bash
# Step 1: Audit what the pipeline SA actually does
# Review all pipeline scripts — list every oc/kubectl command used

# Typical CI/CD actions needed:
# - oc set image deployment/x  → needs: update deployments in target NS
# - oc rollout status          → needs: get deployments in target NS
# - oc start-build             → needs: create builds in build NS
# - oc get pods                → needs: list pods

# Step 2: Create a minimal Role with only those permissions
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-deployer
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "patch", "update"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["replicationcontrollers"]
  verbs: ["get", "list"]
- apiGroups: ["image.openshift.io"]
  resources: ["imagestreams", "imagestreamtags"]
  verbs: ["get", "list"]
EOF

# Step 3: Bind the SA to the minimal role
kubectl create rolebinding cicd-deployer-binding \
  --role=cicd-deployer \
  --serviceaccount=cicd:pipeline-sa \
  -n production

# Step 4: Remove cluster-admin
oc adm policy remove-cluster-role-from-user cluster-admin \
  -z pipeline-sa -n cicd

# Step 5: Test pipeline
# Run pipeline in dry-run / staging first to verify all steps pass

# Step 6: Verify permissions are minimal
oc auth can-i --list \
  --as=system:serviceaccount:cicd:pipeline-sa \
  -n production
# Confirm no cluster-level resources appear

oc auth can-i delete secrets \
  --as=system:serviceaccount:cicd:pipeline-sa \
  -n production
# Expected: no
```

---

### 🔵 Scenario 14: High Availability Incident — One Master Node Failed

*"One of your three OpenShift master nodes has failed. The cluster is still running but you've lost quorum redundancy. What do you do?"*

**Answer:**

```bash
# Step 1: Assess the situation
oc get nodes
# master-1: Ready, master-2: NotReady, master-3: Ready
# Cluster still functional (2 of 3 masters = quorum maintained)

oc get co | grep -v "True.*False.*False"   # All operators still healthy?

# Step 2: Check etcd health
oc rsh -n openshift-etcd etcd-master-1 -- \
  etcdctl member list --write-out=table
# Should show: master-1 started, master-2 failed, master-3 started
# etcd quorum: 2/3 still present → cluster is healthy but not resilient

# Step 3: Diagnose the failed master
oc describe node master-2
# Check if VM is still running (check cloud console / vCenter)

# If VM is running — SSH and check:
# systemctl status kubelet
# journalctl -u kubelet -n 50
# crictl ps — is API server container running?

# Step 4a: If node is recoverable (kubelet died)
# SSH to master-2 and restart:
systemctl restart kubelet
# Watch node come back
oc get nodes -w

# Step 4b: If node is unrecoverable (hardware failure / VM deleted)
# OpenShift has a Machine API to replace it automatically

# Check the Machine object
oc get machines -n openshift-machine-api | grep master-2

# Delete the failed machine — MachineSet will create a replacement
oc delete machine master-2-xxxxx -n openshift-machine-api
# Replacement machine provisions in ~10-15 min

# Watch new machine join
oc get machines -n openshift-machine-api -w
oc get nodes -w

# Step 5: After new node joins, verify etcd is healthy
oc rsh -n openshift-etcd etcd-master-1 -- \
  etcdctl endpoint health --cluster

# Step 6: Approve any pending CSRs for the new node
oc get csr | grep Pending
oc adm certificate approve <csr-name>

# Step 7: Verify all cluster operators are healthy
oc get co | grep -v "True.*False.*False"

# Step 8: Incident review
# Why did master-2 fail? Hardware? Network? Disk?
# Update monitoring to alert on master node failures immediately
```

---

### 🔵 Scenario 15: New JD Role — Your First 90 Days Plan

*"You've just joined as a DevOps engineer in this role. Describe your 90-day plan."*

**Answer:**

**Days 1–30 (Learn & Observe):**
```
Week 1–2:
  ✓ Set up local toolchain: oc, kubectl, helm, terraform, ansible, tkn
  ✓ Review existing architecture documentation
  ✓ Shadow oncall rotation — observe incident response
  ✓ Read through all existing Jenkinsfiles/GitLab CI pipelines
  ✓ Review cluster topology, node counts, resource quotas

Week 3–4:
  ✓ Map the application portfolio — which apps exist, which teams own them
  ✓ Review monitoring dashboards — identify gaps (missing alerts, no runbooks)
  ✓ Inventory SCC usage — any team using privileged unnecessarily?
  ✓ Review RBAC — any overly-broad role bindings?
  ✓ Measure DORA metrics baseline (deployment frequency, lead time, MTTR)
```

**Days 31–60 (Small wins & trust-building):**
```
  ✓ Fix the most critical monitoring gaps — add missing alerts with runbooks
  ✓ Automate one manual task (e.g., namespace onboarding)
  ✓ Improve slowest pipeline (caching, parallelization)
  ✓ Document one undocumented runbook
  ✓ Propose and implement one security improvement (e.g., least-privilege RBAC for pipeline SA)
  ✓ Set up Velero/etcd backup validation (verify backups are restorable)
```

**Days 61–90 (Strategic improvements):**
```
  ✓ Design and implement GitOps for production deployments (if not yet in place)
  ✓ Implement SLOs + error budget tracking for the top 3 services
  ✓ Propose cluster upgrade plan to latest OCP version
  ✓ Conduct a full security audit (Compliance Operator, RBAC review)
  ✓ Present DORA metric improvement plan to management
  ✓ Define team standards: Helm chart template, NetworkPolicy defaults, SCC policy
```

---

> **Next:** [Behavioral Questions →](./07-Behavioral.md)
