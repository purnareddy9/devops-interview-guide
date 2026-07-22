# ☸️ Kubernetes — DevOps Interview Guide

> [← Docker](./Docker.md) | [Main Index](./README.md) | [Jenkins →](./Jenkins.md)

---

## Table of Contents

1. [Architecture](#architecture)
2. [Core Objects](#core-objects)
3. [Workloads](#workloads)
4. [Services & Networking](#services--networking)
5. [Storage](#storage)
6. [Configuration & Secrets](#configuration--secrets)
7. [RBAC & Security](#rbac--security)
8. [Helm](#helm)
9. [Troubleshooting](#troubleshooting)
10. [Interview Questions](#interview-questions)
11. [Scenario-Based Questions](#scenario-based-questions)
12. [Hands-On Labs](#hands-on-labs)
13. [kubectl Cheat Sheet](#kubectl-cheat-sheet)

---

## Architecture

```
┌────────────────────── Kubernetes Cluster ──────────────────────┐
│                                                                  │
│  ┌─────────────── Control Plane ─────────────────┐             │
│  │  ┌──────────────┐  ┌───────────────────────┐  │             │
│  │  │  API Server  │  │  Controller Manager   │  │             │
│  │  │  (kube-api)  │  │  (reconciliation loop)│  │             │
│  │  └──────┬───────┘  └───────────────────────┘  │             │
│  │         │          ┌───────────────────────┐  │             │
│  │  ┌──────▼───────┐  │       Scheduler       │  │             │
│  │  │     etcd     │  │  (pod → node binding) │  │             │
│  │  │  (key-value) │  └───────────────────────┘  │             │
│  │  └──────────────┘                             │             │
│  └───────────────────────────────────────────────┘             │
│                                                                  │
│  ┌── Worker Node 1 ──┐  ┌── Worker Node 2 ──┐                  │
│  │  ┌─────────────┐  │  │  ┌─────────────┐  │                  │
│  │  │   kubelet   │  │  │  │   kubelet   │  │                  │
│  │  ├─────────────┤  │  │  ├─────────────┤  │                  │
│  │  │  kube-proxy │  │  │  │  kube-proxy │  │                  │
│  │  ├─────────────┤  │  │  ├─────────────┤  │                  │
│  │  │ Pod│Pod│Pod │  │  │  │ Pod│Pod│Pod │  │                  │
│  │  └─────────────┘  │  │  └─────────────┘  │                  │
│  └───────────────────┘  └───────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

### Control Plane Components

| Component | Role |
|-----------|------|
| **kube-apiserver** | All API requests, authentication, admission control |
| **etcd** | Distributed key-value store — the source of truth |
| **kube-scheduler** | Assigns pods to nodes based on resources/constraints |
| **kube-controller-manager** | Runs control loops (ReplicaSet, Node, Job controllers) |
| **cloud-controller-manager** | Cloud-provider specific (LB, volumes, routes) |

### Node Components

| Component | Role |
|-----------|------|
| **kubelet** | Pod lifecycle management on the node |
| **kube-proxy** | Network rules (iptables/IPVS) for Services |
| **Container runtime** | CRI-compliant runtime (containerd, CRI-O) |

---

## Core Objects

### Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
    version: "1.0"
  annotations:
    team: platform
spec:
  containers:
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
    env:
    - name: DB_URL
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: url
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
    volumeMounts:
    - name: config
      mountPath: /etc/myapp
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: myapp-config
  serviceAccountName: myapp-sa
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
```

---

## Workloads

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # Extra pods during rollout
      maxUnavailable: 0     # Zero downtime
  template:
    metadata:
      labels:
        app: myapp
    spec:
      affinity:
        podAntiAffinity:    # Spread across nodes
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: myapp
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: myapp:1.0
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

### StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres     # Required for stable DNS
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:     # PVC per pod (postgres-0, postgres-1, etc.)
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

### DaemonSet

```yaml
# Runs one pod per node — useful for logging agents, monitoring
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule     # Run on master nodes too
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

### HPA (Horizontal Pod Autoscaler)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
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
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # Wait 5 min before scale down
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

---

## Services & Networking

### Service Types

| Type | Description | Use Case |
|------|-------------|---------|
| **ClusterIP** | Virtual IP within cluster | Internal service-to-service |
| **NodePort** | Exposes on node IP:port (30000-32767) | Dev/testing, basic external access |
| **LoadBalancer** | Cloud LB provisioned | Production external traffic |
| **ExternalName** | DNS CNAME alias | External service abstraction |

```yaml
# ClusterIP (default)
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP

# LoadBalancer
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 443
    targetPort: 8080
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### NetworkPolicy

```yaml
# Only allow traffic from frontend namespace to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
      podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
```

---

## Storage

### PersistentVolume & PVC

```yaml
# StorageClass (dynamic provisioning)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myapp-data
spec:
  storageClassName: fast-ssd
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
```

---

## Configuration & Secrets

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  APP_ENV: production
  LOG_LEVEL: info
  config.yaml: |
    server:
      port: 8080
    database:
      pool_size: 10
```

### Secret

```bash
# Create secret imperatively
kubectl create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=supersecret \
  --from-file=tls.crt=./cert.pem

# Seal secrets with Sealed Secrets (GitOps-safe)
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
```

---

## RBAC & Security

### RBAC Structure

```
ServiceAccount  ──bind──>  Role/ClusterRole
                           (verbs on resources)

Role          = namespace-scoped
ClusterRole   = cluster-wide

RoleBinding         = binds Role to subject in namespace
ClusterRoleBinding  = binds ClusterRole cluster-wide
```

```yaml
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deploy-sa
  namespace: production

---
# Role
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deploy-sa-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: deploy-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### PodSecurityContext

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
```

---

## Helm

### Helm Concepts

```
Chart        = Package (templates + values + metadata)
Release      = Instance of a chart deployed in a cluster
Repository   = Collection of charts
Values       = Configuration overrides for templates
```

```bash
# Install/upgrade
helm install myapp ./mychart -n production
helm install myapp ./mychart -f values-prod.yaml
helm upgrade myapp ./mychart --atomic --timeout 5m0s
helm upgrade --install myapp ./mychart  # Idempotent

# Inspect
helm list -A
helm status myapp
helm history myapp
helm get values myapp

# Rollback
helm rollback myapp 2

# Template debugging
helm template myapp ./mychart -f values.yaml
helm lint ./mychart
helm diff upgrade myapp ./mychart -f values.yaml  # plugin

# Repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx
```

---

## Troubleshooting

### Pod Troubleshooting Checklist

```bash
# Step 1: Check pod status
kubectl get pods -n namespace
kubectl describe pod <pod-name> -n namespace    # Events section!

# Common issues from Events:
# - ErrImagePull/ImagePullBackOff → image not found or auth issue
# - Pending → no nodes fit (resources, affinity, taints)
# - CrashLoopBackOff → container keeps crashing
# - OOMKilled → out of memory

# Step 2: Check logs
kubectl logs <pod> -n namespace
kubectl logs <pod> -n namespace --previous    # Previous instance
kubectl logs <pod> -n namespace -c container  # Specific container

# Step 3: Exec into pod
kubectl exec -it <pod> -n namespace -- bash
kubectl exec -it <pod> -n namespace -- sh  # For alpine

# Step 4: Check resource usage
kubectl top pods -n namespace
kubectl top nodes

# Step 5: Check node
kubectl describe node <node-name>  # Conditions, capacity, events

# Pending pod: check
kubectl describe pod <pod> | grep -A 10 Events
# - Insufficient CPU/memory: scale nodes or reduce requests
# - NodeSelector/affinity mismatch: check labels
# - Taints: add toleration or untaint node
```

### Common Error Resolution

| Error | Cause | Fix |
|-------|-------|-----|
| `ImagePullBackOff` | Wrong image name, auth failure | Verify image exists; check imagePullSecret |
| `CrashLoopBackOff` | App error, bad config, OOM | Check logs with `--previous` flag |
| `Pending` | Insufficient resources, no nodes match | Check node capacity, affinity, taints |
| `Evicted` | Node disk/memory pressure | Clean up node; add resource limits |
| `OOMKilled` | Container exceeded memory limit | Increase memory limit |
| `Error: container runtime` | CRI issue on node | Check containerd/kubelet status |

---

## Interview Questions

### 🟢 Beginner

**Q1: What is the difference between a Pod and a Deployment?**

> A **Pod** is the smallest deployable unit — one or more containers sharing network/storage. A **Deployment** manages Pods declaratively: desired state, rolling updates, replica count, rollback. Deployments create ReplicaSets which create Pods. In practice, you always use Deployments (or StatefulSets), not bare Pods.

**Q2: What is the difference between `ClusterIP`, `NodePort`, and `LoadBalancer`?**

> - **ClusterIP**: Virtual IP reachable only within the cluster — for internal service communication
> - **NodePort**: Opens a port (30000-32767) on every node — external traffic can reach it, but requires knowing node IP
> - **LoadBalancer**: Provisions a cloud load balancer (AWS ALB/NLB, GCP LB) — production-grade external access

**Q3: What are liveness and readiness probes?**

> - **Liveness probe**: Checks if the container is alive. Failure → container is restarted. Use for deadlock detection.
> - **Readiness probe**: Checks if the container is ready to serve traffic. Failure → removed from Service endpoints (no traffic). Use for startup and overload scenarios.
> - **Startup probe** (newer): Gives slow-starting containers time to start before liveness kicks in.

---

### 🟡 Intermediate

**Q4: How does Kubernetes service discovery work?**

> Kubernetes injects environment variables for each Service at Pod creation time. More powerfully, kube-dns (CoreDNS) provides DNS resolution: a Service named `myapp` in namespace `production` is reachable at `myapp.production.svc.cluster.local` (or just `myapp` within the same namespace). The format is: `<service>.<namespace>.svc.<cluster-domain>`.

**Q5: Explain the difference between StatefulSet and Deployment.**

> | Feature | Deployment | StatefulSet |
> |---------|-----------|------------|
> | Pod identity | Random names | Stable, ordered names (pod-0, pod-1) |
> | Storage | Shared or none | Unique PVC per pod |
> | Scaling | Any order | Ordered (pod-0 before pod-1) |
> | DNS | Single service | Unique DNS per pod |
> | Use case | Stateless apps | Databases, queues, ZooKeeper |

**Q6: What is a Kubernetes Operator?**

> An Operator extends Kubernetes with **custom resources** (CRDs) and a **controller** that encodes operational knowledge. Instead of a human running `kubectl` commands, the Operator watches the custom resource and reconciles the actual state to match desired state. Examples: Prometheus Operator, cert-manager, database operators (PostgreSQL, MongoDB). Built with Operator SDK or kubebuilder.

---

### 🔴 Advanced

**Q7: Explain Kubernetes scheduler decision-making.**

> Scheduler selects a node via two phases:
> 1. **Filtering**: removes nodes that don't satisfy Pod requirements: resource requests, nodeSelector, affinity/anti-affinity, taints/tolerations, PVC availability
> 2. **Scoring**: ranks remaining nodes: resource balance, image locality, pod affinity preferences, spread constraints
> The highest-scoring node is selected. Custom schedulers or scheduler plugins can extend this.

**Q8: How do you debug a networking issue in Kubernetes?**

```bash
# 1. Verify service endpoints exist
kubectl get endpoints myapp-svc

# 2. Test pod-to-pod directly (bypass Service)
kubectl exec -it debug-pod -- curl http://10.244.1.5:8080

# 3. Test via ClusterIP
kubectl exec -it debug-pod -- curl http://myapp-svc.production.svc.cluster.local

# 4. Check CoreDNS
kubectl exec -it debug-pod -- nslookup myapp-svc.production.svc.cluster.local

# 5. Check kube-proxy iptables rules
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs <kube-proxy-pod> -n kube-system

# 6. Network policy blocking?
kubectl get networkpolicies -n production

# 7. Use netshoot for advanced debugging
kubectl run tmp-debug --rm -i --tty \
  --image nicolaka/netshoot -- bash
```

---

## Scenario-Based Questions

### 🔵 Scenario 1: Deploy with Zero Downtime

*"Deploy a new version of your application to production without any downtime."*

```yaml
# Deployment update strategy
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0   # Zero downtime

  # Readiness probe ensures traffic only goes to ready pods
  readinessProbe:
    httpGet:
      path: /health/ready
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 3
    failureThreshold: 3
```

```bash
# Update image
kubectl set image deployment/myapp app=myapp:v2 -n production

# Watch rollout
kubectl rollout status deployment/myapp -n production

# If something goes wrong
kubectl rollout undo deployment/myapp -n production
kubectl rollout undo deployment/myapp --to-revision=3
```

### 🔵 Scenario 2: Pod Stuck in Pending

```bash
kubectl describe pod stuck-pod

# Events show:
# 0/5 nodes are available: 5 Insufficient cpu
# Fix: check node resources
kubectl top nodes
kubectl describe node node-1 | grep -A 5 "Allocated resources"

# Options:
# 1. Reduce resource requests in deployment
# 2. Scale node group (AWS: update ASG)
# 3. Add node with autoscaler annotation
kubectl annotate node node-1 cluster-autoscaler.kubernetes.io/safe-to-evict=true
```

---

## Hands-On Labs

### Lab 1: Deploy a Full Stack Application

```bash
# 1. Create namespace
kubectl create namespace demo

# 2. Deploy PostgreSQL
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POSTGRES_PASSWORD
          value: "password123"
        - name: POSTGRES_DB
          value: "mydb"
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: demo
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
EOF

# 3. Watch everything
kubectl get all -n demo -w
```

---

## kubectl Cheat Sheet

```bash
# ─── GET ─────────────────────────────────────────────
kubectl get all -n namespace
kubectl get pods -o wide
kubectl get pods --sort-by=.status.startTime
kubectl get pods -l app=myapp
kubectl get events -n namespace --sort-by='.lastTimestamp'

# ─── DESCRIBE ────────────────────────────────────────
kubectl describe pod <name>
kubectl describe node <name>
kubectl describe svc <name>

# ─── LOGS ────────────────────────────────────────────
kubectl logs <pod> -f
kubectl logs <pod> --previous -c <container>
kubectl logs -l app=myapp --all-containers=true

# ─── EXEC ────────────────────────────────────────────
kubectl exec -it <pod> -- bash
kubectl port-forward pod/<pod> 8080:80
kubectl port-forward svc/myapp-svc 8080:80

# ─── APPLY/SCALE ─────────────────────────────────────
kubectl apply -f manifest.yaml
kubectl delete -f manifest.yaml
kubectl scale deployment myapp --replicas=5
kubectl rollout restart deployment/myapp
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp

# ─── DEBUG ───────────────────────────────────────────
kubectl run tmp --rm -it --image busybox -- sh
kubectl debug node/<node> -it --image ubuntu
kubectl top pods --containers

# ─── CONTEXT ─────────────────────────────────────────
kubectl config get-contexts
kubectl config use-context prod-cluster
kubectl config set-context --current --namespace=production
```

---

> **Cross-links:** [← Docker](./Docker.md) | [Jenkins →](./Jenkins.md) | [Helm (Kubernetes package manager) →](./CICD.md) | [Monitoring (Prometheus/Grafana on K8s) →](./Monitoring.md) | [Security (K8s RBAC) →](./Security.md) | [OpenShift (enterprise Kubernetes) →](./OpenShift.md)
