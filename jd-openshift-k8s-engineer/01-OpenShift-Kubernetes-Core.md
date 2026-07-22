---
layout: default
title: "🔴 Module 1 — OpenShift & Kubernetes Core"
render_with_liquid: false
---

# 🔴 Module 1 — OpenShift & Kubernetes Core

> **JD Alignment:** "Develop and maintain OpenShift and Kubernetes infrastructure to support application deployment. Configure and optimize container orchestration for performance and reliability."

> [← README](./README.md) | [CI/CD Pipelines →](./02-CICD-Pipelines.md)

---

## Table of Contents

1. [Architecture Questions](#architecture-questions)
2. [Workloads & Scheduling](#workloads--scheduling)
3. [Networking](#networking)
4. [Storage](#storage)
5. [OpenShift-Specific](#openshift-specific)
6. [High Availability & Scalability](#high-availability--scalability)

---

## Architecture Questions

---

### 🟢 Q1. Explain the Kubernetes architecture and the role of each component.

**Answer:**

Kubernetes has two layers — the **Control Plane** and **Worker Nodes**.

**Control Plane:**

| Component | Role |
|-----------|------|
| `kube-apiserver` | Front-door for all API requests. Handles authentication, authorization, admission control. All other components communicate through it. |
| `etcd` | Distributed key-value store — the single source of truth for all cluster state. Runs as a 3- or 5-node quorum in HA. |
| `kube-scheduler` | Watches for unscheduled pods and selects the best node using filtering (resource requests, affinity, taints) then scoring. |
| `kube-controller-manager` | Runs reconciliation loops: ReplicaSet controller, Node controller, Job controller, Endpoint controller, etc. |
| `cloud-controller-manager` | Integrates with cloud APIs (AWS, Azure, GCP) for load balancers, persistent volumes, and node lifecycle. |

**Worker Node:**

| Component | Role |
|-----------|------|
| `kubelet` | Agent on every node. Receives PodSpecs from the API server and ensures containers are running. Reports node/pod status. |
| `kube-proxy` | Manages iptables/IPVS rules for Service networking. Routes traffic to the correct pod IP. |
| `Container runtime` | CRI-compliant runtime — `containerd` on upstream K8s, `CRI-O` on OpenShift. |

**In OpenShift additionally:**
- The control plane runs on RHCOS (Red Hat CoreOS) managed by the Machine Config Operator.
- HAProxy Router pods (on infra nodes) handle `Route`-based ingress.
- OAuth server handles authentication before the API server.

---

### 🟢 Q2. What is the difference between a Pod, ReplicaSet, and Deployment?

**Answer:**

```
Deployment
  └── manages → ReplicaSet
                  └── manages → Pod(s)
```

| Object | Purpose |
|--------|---------|
| **Pod** | Smallest deployable unit. One or more containers sharing the same network namespace and volumes. Ephemeral — if it dies it is not recreated. |
| **ReplicaSet** | Ensures N identical pods are always running. If a pod dies, it creates a new one. Selects pods by label. Rarely created directly. |
| **Deployment** | Owns ReplicaSets. Adds: declarative updates (rolling/recreate), rollback history, pause/resume rollouts. This is what you always create in practice. |

```yaml
# Deployment creates RS (myapp-7d9f) which creates Pods (myapp-7d9f-abc12)
kubectl get rs
kubectl get pods
kubectl rollout history deployment/myapp    # Deployment-level history
```

---

### 🟡 Q3. How does the Kubernetes scheduler decide which node to assign a pod to?

**Answer:**

The scheduler runs in two phases:

**Phase 1 — Filtering (hard constraints):**
Removes nodes that are ineligible:
- Insufficient CPU/memory for `requests`
- `nodeSelector` or `nodeAffinity` doesn't match
- Node has a `taint` the pod doesn't tolerate
- PVC's PV isn't available on that node
- Pod's `topologySpreadConstraints` would be violated

**Phase 2 — Scoring (soft preferences):**
Ranks remaining nodes (0–100) by:
- `LeastAllocated` — prefer nodes with most free resources
- `ImageLocality` — prefer nodes that already have the container image
- `InterPodAffinity` — prefer nodes near or away from other pods
- `NodeAffinity` preferred rules

The node with the highest score is selected. Ties are broken randomly.

```yaml
# Force a pod to a specific node
spec:
  nodeSelector:
    kubernetes.io/hostname: worker-3

# Prefer GPU nodes but don't require it
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 80
      preference:
        matchExpressions:
        - key: nvidia.com/gpu
          operator: Exists
```

---

### 🟡 Q4. What are taints and tolerations? Give a real-world use case.

**Answer:**

- **Taint** — applied to a **node** to repel pods that don't explicitly tolerate it.
- **Toleration** — applied to a **pod** to allow it to be scheduled on a tainted node.

**Effects:**
| Effect | Behavior |
|--------|---------|
| `NoSchedule` | New pods won't be scheduled unless they tolerate the taint |
| `PreferNoSchedule` | Soft version — scheduler tries to avoid |
| `NoExecute` | New pods not scheduled AND existing pods evicted unless tolerated |

**Real-world use cases:**

1. **Dedicated GPU nodes** — Taint GPU nodes so only ML workloads run there.
2. **Spot/preemptible nodes** — Taint so only fault-tolerant batch jobs land on them.
3. **OpenShift infra nodes** — Infra nodes are tainted so only router/registry/monitoring pods run on them.

```bash
# Taint a node for GPU workloads
kubectl taint node gpu-node-1 nvidia.com/gpu=present:NoSchedule

# Pod with toleration
spec:
  tolerations:
  - key: "nvidia.com/gpu"
    operator: "Equal"
    value: "present"
    effect: "NoSchedule"
  nodeSelector:
    nvidia.com/gpu: "true"
```

---

### 🔴 Q5. Explain etcd and what happens if etcd loses quorum.

**Answer:**

`etcd` is a distributed, strongly-consistent key-value store using the **Raft consensus protocol**. All Kubernetes state (pods, secrets, configmaps, deployments) is stored in etcd.

**Quorum:** A cluster of N nodes requires ⌊N/2⌋ + 1 nodes to agree before committing writes.
- 3-node cluster → needs 2 nodes (can tolerate 1 failure)
- 5-node cluster → needs 3 nodes (can tolerate 2 failures)

**What happens when quorum is lost:**
- The API server becomes read-only — `kubectl get` still works.
- No writes (new pods, deployments, secret updates) are accepted.
- The cluster is stuck — no new scheduling, no updates.

**Recovery steps:**
```bash
# 1. Check etcd health
etcdctl endpoint health --cluster

# 2. If a member is unhealthy, remove it
etcdctl member list
etcdctl member remove <member-id>

# 3. Re-add new member or restore from snapshot
etcdctl snapshot restore snapshot.db \
  --name member1 \
  --data-dir /var/lib/etcd-restored \
  --initial-cluster "member1=https://10.0.0.1:2380"

# In OpenShift: use the etcd operator
oc get etcd -o wide
```

---

## Workloads & Scheduling

---

### 🟢 Q6. What is the difference between a Deployment, StatefulSet, and DaemonSet?

**Answer:**

| Feature | Deployment | StatefulSet | DaemonSet |
|---------|-----------|------------|-----------|
| Pod identity | Random names | Stable, ordered (`pod-0`, `pod-1`) | One pod per node |
| Storage | Shared or ephemeral | Unique PVC per pod | Typically uses hostPath |
| Scaling order | Any order | Ordered (0 → 1 → 2) | Automatic with node count |
| DNS | Single service | Unique DNS per pod | NodeIP-based |
| Use case | Stateless apps (web, API) | Databases, queues, ZooKeeper | Log agents, monitoring, CNI |

```bash
# StatefulSet pod names are predictable
postgres-0, postgres-1, postgres-2

# DaemonSet ensures one pod per node
oc get pods -o wide -l app=fluentd   # one per node
```

---

### 🟡 Q7. How do you implement zero-downtime deployments in Kubernetes/OpenShift?

**Answer:**

Zero-downtime requires three things working together:

**1. Rolling update strategy with maxUnavailable=0:**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1           # Allow 1 extra pod during rollout
    maxUnavailable: 0     # Never remove a pod until new one is ready
```

**2. Readiness probe — gates traffic:**
```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 3
  failureThreshold: 3
# Pod only receives traffic after 3 consecutive passing probes
```

**3. Graceful shutdown — handles in-flight requests:**
```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]  # Let LB drain
terminationGracePeriodSeconds: 60
```

**Rollback if needed:**
```bash
kubectl rollout status deployment/myapp     # Watch progress
kubectl rollout undo deployment/myapp       # Immediate rollback
kubectl rollout undo deployment/myapp --to-revision=3
```

---

### 🔴 Q8. Explain Horizontal Pod Autoscaler (HPA) vs Vertical Pod Autoscaler (VPA) vs KEDA.

**Answer:**

| Autoscaler | Scales | Based On | Use Case |
|-----------|--------|---------|---------|
| **HPA** | Pod count (horizontal) | CPU, memory, custom metrics | Web/API servers with variable load |
| **VPA** | Pod resource requests/limits | Historical usage | Long-running processes needing right-sizing |
| **KEDA** | Pod count, down to zero | External sources (queue depth, DB rows, Kafka lag) | Event-driven workers, batch jobs |

```yaml
# HPA — scale on CPU + custom metric
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 min before scaling down
```

```yaml
# KEDA — scale based on queue depth (SQS example)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
spec:
  scaleTargetRef:
    name: worker-deployment
  minReplicaCount: 0      # Can scale to zero!
  maxReplicaCount: 50
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123/my-queue
      queueLength: "10"
```

**Combined strategy:** Use HPA for reactive scaling during traffic spikes; VPA for right-sizing resource requests; KEDA for event-driven queue workers.

---

## Networking

---

### 🟢 Q9. Explain the four types of Kubernetes Services.

**Answer:**

| Type | Description | Use Case |
|------|-------------|---------|
| **ClusterIP** | Virtual IP reachable only inside the cluster | Service-to-service communication |
| **NodePort** | Opens port 30000–32767 on every node; routes to ClusterIP | Dev/test external access |
| **LoadBalancer** | Provisions cloud LB (AWS ALB/NLB, GCP LB); assigns external IP | Production external access |
| **ExternalName** | CNAME DNS alias to an external hostname | Point cluster traffic to external DB |

```yaml
# LoadBalancer with AWS NLB annotations
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 443
    targetPort: 8443
```

**In OpenShift:** Use `Route` instead of `LoadBalancer` for most external access. The HAProxy Router handles TLS and hostname-based routing more efficiently.

---

### 🟡 Q10. What is an Ingress in Kubernetes? How does OpenShift Route differ from it?

**Answer:**

**Kubernetes Ingress:**
- An API object that defines HTTP routing rules (host/path → service).
- Requires an external **Ingress Controller** (NGINX, Traefik, AWS ALB Controller) to implement the rules.
- TLS supported via `tls.secretName`.

**OpenShift Route:**
- A first-class Route object with a built-in HAProxy Router (no extra controller needed).
- Three TLS modes: `edge` (terminate at router), `passthrough` (TLS to pod), `reencrypt` (TLS to pod with new cert).
- Auto-generates hostname: `<name>-<namespace>.apps.<cluster-domain>`.
- Supports weighted routing for A/B testing (multiple Services with weights).

```yaml
# OpenShift Route — edge TLS with redirect
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myapp
spec:
  host: myapp.apps.mycluster.example.com
  to:
    kind: Service
    name: myapp-svc
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

---

### 🔴 Q11. How do you debug a pod-to-pod networking issue in OpenShift/Kubernetes?

**Answer:**

```bash
# Step 1: Verify both pods are Running and have IPs
oc get pods -o wide -n my-app

# Step 2: Verify the Service exists and has endpoints
oc get svc my-service -n my-app
oc get endpoints my-service -n my-app
# If endpoints list is empty → label selector mismatch!
oc describe svc my-service | grep Selector

# Step 3: Test pod-to-pod directly (bypass Service)
oc exec -it pod-a -- curl http://10.244.1.15:8080

# Step 4: Test via ClusterIP (Service)
oc exec -it pod-a -- curl http://my-service.my-app.svc.cluster.local:8080

# Step 5: Test DNS resolution
oc exec -it pod-a -- nslookup my-service.my-app.svc.cluster.local

# Step 6: Check NetworkPolicies
oc get networkpolicies -n my-app
# If a deny-all policy exists, verify it has the right allow rule

# Step 7: Check OVN/SDN logs
oc logs -n openshift-ovn-kubernetes -l app=ovnkube-node --tail=50

# Step 8: Use netshoot for advanced diagnosis
oc run debug --rm -it --image=nicolaka/netshoot -- bash
# Inside: tcpdump, traceroute, curl, nmap available
```

---

## Storage

---

### 🟡 Q12. Explain PersistentVolume, PersistentVolumeClaim, and StorageClass.

**Answer:**

```
StorageClass (defines HOW storage is provisioned)
    │
    ▼ (dynamic provisioning)
PersistentVolume (PV) — the actual storage resource
    │
    ▼ (bound)
PersistentVolumeClaim (PVC) — a pod's request for storage
    │
    ▼ (mounted)
Pod
```

| Object | Who manages it | Purpose |
|--------|----------------|---------|
| **PV** | Cluster admin (or dynamic provisioner) | Represents a piece of storage (NFS, EBS, Ceph) |
| **PVC** | Developer/app | Requests a volume with specific size and access mode |
| **StorageClass** | Admin | Describes the provisioner and parameters; enables dynamic PV creation |

**Access Modes:**
| Mode | Abbreviation | Description |
|------|-------------|-------------|
| ReadWriteOnce | RWO | One node can read+write |
| ReadOnlyMany | ROX | Many nodes can read |
| ReadWriteMany | RWX | Many nodes can read+write (NFS, CephFS) |
| ReadWriteOncePod | RWOP | Only one pod cluster-wide (K8s 1.22+) |

```yaml
# Dynamic PVC — StorageClass auto-provisions the PV
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3-csi      # AWS ROSA default
  resources:
    requests:
      storage: 20Gi
```

---

## OpenShift-Specific

---

### 🟢 Q13. What is Source-to-Image (S2I) and what problem does it solve?

**Answer:**

**Problem:** Developers shouldn't need to write Dockerfiles or know container internals to deploy their app.

**S2I Solution:**

```
Source code (Git URL)
    +
Builder image (nodejs:18-ubi8, python:3.11, java:17)
    │
    ▼ (assemble script runs inside builder)
    ├── Clone source
    ├── npm install / pip install / mvn package
    └── Output: runnable app image
    │
    ▼
Internal registry → ImageStream → Deployment trigger
```

```bash
# One command to build + deploy a Node.js app from source
oc new-app nodejs:18-ubi8~https://github.com/myorg/my-api.git \
  --name=my-api \
  --env=NODE_ENV=production

# Watch the build
oc logs -f bc/my-api

# Rebuild after code change
oc start-build my-api
```

**Advantages:**
- Reproducible — builder image version = known build environment.
- Upgrades: change `nodejs:16` → `nodejs:18` in BuildConfig, all apps rebuilt automatically.
- No Dockerfile means no developer mistake in image construction.
- Red Hat UBI (Universal Base Image) builder images are enterprise-supported and RHEL-compatible.

---

### 🟡 Q14. What are ImageStreams and why does OpenShift use them?

**Answer:**

An **ImageStream** is an OpenShift abstraction that **tracks image versions by tag** inside the cluster, decoupling deployments from registry URLs.

**Without ImageStreams (pure K8s):**
- Deployment YAML contains `image: quay.io/myorg/myapp:v1.2.3`
- To deploy v1.2.4 you must edit the YAML and apply it.

**With ImageStreams (OpenShift):**
- Deployment references `my-app/myapp:latest` (ImageStream tag)
- When a new build pushes to `myapp:latest`, the **ImageChange trigger** automatically fires the rollout.
- No YAML changes needed.

```bash
# Import and track an external image with scheduled polling
oc import-image nginx:1.25 --from=docker.io/nginx:1.25 \
  --confirm --scheduled=true

# Tag production-approved image
oc tag myapp:v1.2.3 myapp:production    # Only production gets this

# View all tags
oc describe is myapp
```

---

### 🟡 Q15. Explain OpenShift's Security Context Constraints (SCC). How do they differ from Kubernetes PodSecurity?

**Answer:**

**SCC (OpenShift):**
- Enforced at admission time by a built-in admission webhook.
- Evaluated per pod: the pod's service account must have a role binding granting an SCC.
- Default SCC is `restricted-v2` — blocks root, drops all capabilities, sets random UID.
- Highly granular: controls UID ranges, volume types, SELinux, seccomp, capabilities individually.

**Kubernetes PodSecurity (upstream):**
- Three profiles: `privileged`, `baseline`, `restricted`.
- Applied at **namespace** level (namespace label), not per service account.
- Less granular — it's a profile, not a fine-grained policy.

```bash
# Grant service account anyuid SCC (for legacy root apps)
oc adm policy add-scc-to-user anyuid \
  -z my-legacy-app-sa -n my-project

# Check what SCC a running pod actually used
oc get pod my-pod -o jsonpath='{.metadata.annotations.openshift\.io/scc}'

# Dry-run: what SCC would this pod manifest use?
oc adm policy scc-subject-review -f pod.yaml

# Audit all pods and their SCCs in a namespace
oc get pods -n my-project \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.openshift\.io/scc}{"\n"}{end}'
```

---

## High Availability & Scalability

---

### 🔴 Q16. How do you design a highly available OpenShift/Kubernetes deployment?

**Answer:**

HA has multiple layers:

**1. Control Plane HA:**
- 3 master/control-plane nodes (or 5 for higher fault tolerance).
- etcd runs on all 3 masters in quorum.
- API server load-balanced via an external LB (or VIP with keepalived).

**2. Worker Node HA:**
- Minimum 3 worker nodes spread across Availability Zones.
- `topologySpreadConstraints` or `podAntiAffinity` to spread pods across zones.

**3. Application HA:**
```yaml
spec:
  replicas: 3
  strategy:
    rollingUpdate:
      maxUnavailable: 0       # Never drop below desired replicas during updates
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: myapp
            topologyKey: topology.kubernetes.io/zone   # Spread across AZs
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: myapp
```

**4. Pod Disruption Budget (PDB):**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2      # At least 2 pods always running during node drain/upgrade
  selector:
    matchLabels:
      app: myapp
```

**5. Readiness + liveness probes** — ensures LB never sends traffic to unhealthy pods.

**6. Resource requests/limits** — prevents noisy-neighbour starvation.

---

### 🔴 Q17. What is Cluster Autoscaler and how does it work in OpenShift?

**Answer:**

**Cluster Autoscaler** automatically adds/removes worker nodes based on pod scheduling demand.

**Scale-up trigger:** A pod is `Pending` because no node has sufficient resources.
**Scale-down trigger:** A node has been underutilized (< 50% requested) for 10+ minutes AND all pods on it can be safely moved.

**In OpenShift (ROSA/OCP on AWS):**

```yaml
# MachineAutoscaler — attach to a MachineSet
apiVersion: autoscaling.openshift.io/v1beta1
kind: MachineAutoscaler
metadata:
  name: worker-us-east-1a
  namespace: openshift-machine-api
spec:
  minReplicas: 2
  maxReplicas: 10
  scaleTargetRef:
    apiVersion: machine.openshift.io/v1beta1
    kind: MachineSet
    name: mycluster-worker-us-east-1a

# ClusterAutoscaler — global settings
apiVersion: autoscaling.openshift.io/v1
kind: ClusterAutoscaler
metadata:
  name: default
spec:
  resourceLimits:
    maxNodesTotal: 30
  scaleDown:
    enabled: true
    delayAfterAdd: 10m
    unneededTime: 10m
```

```bash
# Check autoscaler status
oc get clusterautoscaler
oc get machineautoscaler -n openshift-machine-api
oc get machineset -n openshift-machine-api
```

---

### 🟡 Q18. How do you perform a rolling cluster upgrade in OpenShift?

**Answer:**

OpenShift upgrades are managed by the **Cluster Version Operator (CVO)** — a fully automated, rolling, node-by-node process.

```bash
# Step 1: Pre-flight checks
oc get clusterversion
oc get co | grep -v "True.*False.*False"   # All operators must be healthy
oc get nodes | grep -v Ready               # All nodes must be Ready

# Step 2: Check available upgrade paths
oc adm upgrade
# Output shows available versions and recommended channel

# Step 3: Initiate upgrade
oc adm upgrade --to-latest=true
# Or to a specific version:
oc adm upgrade --to=4.14.5

# Step 4: Monitor progress
watch oc get clusterversion
oc get co -w        # Cluster operators update one at a time

# Step 5: Monitor node reboots (MCO updates RHCOS)
oc get nodes -w     # Nodes go SchedulingDisabled → Ready as they reboot

# Step 6: Verify completion
oc get clusterversion    # Should show Progressing=False, Available=True
oc get co                # All Available=True, Progressing=False, Degraded=False
```

**Important notes:**
- Cannot skip minor versions (4.12 → 4.14 requires 4.13 unless explicitly in upgrade graph).
- OpenShift drains nodes one at a time during RHCOS update — PDBs are honoured.
- Pause a MachineConfigPool to defer node reboots during a maintenance window.

---

### 🟡 Q19. What is a LimitRange and how does it differ from a ResourceQuota?

**Answer:**

| Object | Scope | Controls |
|--------|-------|---------|
| **LimitRange** | Per-container / per-pod | Default requests/limits; min/max bounds per container |
| **ResourceQuota** | Per-namespace (Project) | Total aggregate resources consumed by the namespace |

```yaml
# LimitRange — sets defaults so developers don't have to specify resources
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: my-app
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "4"
      memory: 4Gi
    min:
      cpu: 50m
      memory: 64Mi

---
# ResourceQuota — namespace-level hard caps
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: my-app
spec:
  hard:
    pods: "20"
    requests.cpu: "8"
    requests.memory: "16Gi"
    limits.cpu: "16"
    limits.memory: "32Gi"
    persistentvolumeclaims: "10"
    requests.storage: "200Gi"
    count/deployments.apps: "10"
```

---

### 🟡 Q20. How do you manage configuration and secrets in Kubernetes/OpenShift?

**Answer:**

**ConfigMap** — for non-sensitive configuration:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
```

**Secret** — for sensitive data (base64-encoded, encrypted at rest in etcd):
```yaml
# Never commit plaintext secrets to Git
# Use external-secrets operator or Vault instead
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:               # OpenShift accepts stringData (auto-encodes)
  DB_PASSWORD: "mysecretpassword"
  DB_URL: "postgres://user:pass@db:5432/mydb"
```

**Best practice — External Secrets Operator:**
```yaml
# Pulls secrets from AWS Secrets Manager / HashiCorp Vault automatically
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: db-secret
  data:
  - secretKey: DB_PASSWORD
    remoteRef:
      key: /production/myapp/db
      property: password
```

```bash
# Mount secret as env vars
oc set env deployment/myapp --from=secret/db-secret

# Mount secret as file
oc set volume deployment/myapp \
  --add --name=certs \
  --type=secret \
  --secret-name=tls-certs \
  --mount-path=/etc/certs
```

---

### 🔴 Q21. Explain how you would optimize container resource usage across a cluster.

**Answer:**

**Step 1 — Right-size with VPA recommendations:**
```bash
# Deploy VPA in recommendation mode (no auto-apply)
kubectl top pods --containers -n my-app    # Current usage
# VPA observes for 24+ hrs and suggests new requests/limits
kubectl get vpa -n my-app -o yaml | grep recommendation
```

**Step 2 — Set LimitRanges as guardrails (prevent unbounded containers).**

**Step 3 — Use Namespace ResourceQuotas per team to prevent noisy neighbours.**

**Step 4 — Enable metrics-server and HPA for reactive scaling.**

**Step 5 — Use Goldilocks (open-source) for cluster-wide VPA dashboard:**
```bash
helm install goldilocks fairwinds-stable/goldilocks
kubectl label namespace my-app goldilocks.fairwinds.com/enabled=true
```

**Step 6 — Check for over-provisioned nodes:**
```bash
oc adm top nodes
# If node CPU usage is 10% but requests are 80%, pods are over-requesting
```

**Step 7 — Use Descheduler to rebalance pods after cluster changes.**

---

### 🟢 Q22. What is the difference between `oc` and `kubectl`?

**Answer:**

`oc` is the OpenShift CLI. It is a superset of `kubectl` — every `kubectl` command works with `oc`. OpenShift adds:

| `oc`-only command | Purpose |
|-------------------|---------|
| `oc new-project` | Create project with RBAC defaults |
| `oc new-app` | Deploy app from source/image with auto-config |
| `oc start-build` | Trigger a BuildConfig build |
| `oc expose` | Create a Route from a Service |
| `oc rollout latest dc/` | DeploymentConfig rollout |
| `oc adm policy add-scc-to-user` | Manage SCC grants |
| `oc adm upgrade` | Cluster version upgrade |
| `oc debug node/<node>` | Node-level privileged debug shell |
| `oc login` | Authenticate to OCP (with OAuth tokens) |
| `oc whoami --show-console` | Get web console URL |
| `oc rsh <pod>` | Shorthand for exec into pod shell |

---

### 🔴 Q23. How do you debug a CrashLoopBackOff pod?

**Answer:**

```bash
# Step 1: Get pod status
oc get pod crashing-pod -n my-app

# Step 2: Describe — check Events section
oc describe pod crashing-pod -n my-app
# Look for: OOMKilled, liveness probe failed, image pull errors

# Step 3: Get current logs
oc logs crashing-pod -n my-app

# Step 4: Get logs from previous (crashed) container
oc logs crashing-pod --previous -n my-app

# Step 5: If image crashes immediately (can't exec into it)
# Override the entrypoint to get a shell
oc debug pod/crashing-pod    # OpenShift creates a debug copy

# Or patch to sleep
kubectl patch deployment myapp -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"app","command":["sleep","3600"]}]}}}}'
```

**Common causes & fixes:**

| Cause | Evidence | Fix |
|-------|---------|-----|
| OOMKilled | `Reason: OOMKilled` in describe | Increase memory limits |
| Bad env var | App logs "cannot connect to DB" | Check secret/configmap references |
| Missing file/mount | App log "file not found" | Check volume mounts |
| Liveness probe too aggressive | `Liveness probe failed` in events | Increase `initialDelaySeconds` |
| Wrong entrypoint | Container exits with code 1 immediately | Fix CMD/ENTRYPOINT in image |

---

### 🟡 Q24. What is a PodDisruptionBudget (PDB) and when would you use it?

**Answer:**

A PDB tells Kubernetes the **minimum number of pods that must remain available** during voluntary disruptions (node drain, cluster upgrade, manual pod eviction).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: my-app
spec:
  minAvailable: 2        # Always keep 2 pods running
  # OR: maxUnavailable: 1   # Allow at most 1 pod to be disrupted
  selector:
    matchLabels:
      app: myapp
```

**Use cases:**
- **Cluster upgrades:** OpenShift respects PDBs when draining nodes for RHCOS updates — it will wait if draining would violate the PDB.
- **Multiple replicas:** If you have 3 replicas and PDB `minAvailable: 2`, you can only drain one node at a time.
- **Stateful apps:** Ensure quorum is never broken (e.g., 3-node Kafka — minAvailable: 2).

**Caution:** A PDB with `minAvailable: 3` on a 3-replica deployment will **block cluster upgrades** entirely — always leave room for at least 1 disruption.

---

### 🟡 Q25. How does OpenShift's internal image registry work?

**Answer:**

OpenShift ships with a built-in image registry (`image-registry.openshift-image-registry.svc:5000`) deployed in the `openshift-image-registry` namespace.

**Key properties:**
- Backed by object storage (S3, Azure Blob, GCS, or PVC)
- Integrated with OpenShift RBAC — pushing/pulling images requires proper role bindings
- ImageStreams reference images by SHA256 digest internally (not mutable tags)
- `image-pusher` role binding allows CI/CD service accounts to push images

```bash
# Check registry status
oc get configs.imageregistry.operator.openshift.io cluster
oc get pods -n openshift-image-registry

# Configure S3 storage backend
oc edit configs.imageregistry.operator.openshift.io cluster
# Set storage.s3.bucket, storage.s3.region

# Allow a service account to push images
oc policy add-role-to-user registry-editor \
  -z pipeline-sa -n my-app

# Build and push via Buildah inside OpenShift pipeline
# IMAGE=image-registry.openshift-image-registry.svc:5000/my-app/myimage:latest
# (Service account token is used for auth automatically)
```

---

> **Next:** [CI/CD Pipelines →](./02-CICD-Pipelines.md)