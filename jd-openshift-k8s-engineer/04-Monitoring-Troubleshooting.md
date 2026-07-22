---
render_with_liquid: false
layout: default
title: "📊 Module 4 — Monitoring & Troubleshooting"

---

# 📊 Module 4 — Monitoring & Troubleshooting

> **JD Alignment:** "Monitor system performance and troubleshoot issues related to container orchestration. Monitor cluster health and respond to incidents promptly."

> [← Security & Compliance](./03-Security-Compliance.md) | [README](./README.md) | [Infrastructure & Automation →](./05-Infrastructure-Automation.md)

---

## Table of Contents

1. [Monitoring Stack](#monitoring-stack)
2. [Prometheus & Metrics](#prometheus--metrics)
3. [Grafana Dashboards](#grafana-dashboards)
4. [Alerting](#alerting)
5. [Logging](#logging)
6. [Troubleshooting Methodology](#troubleshooting-methodology)
7. [Common Issues & Resolutions](#common-issues--resolutions)

---

## Monitoring Stack

---

### 🟢 Q1. What is the default monitoring stack in OpenShift and how is it structured?

**Answer:**

OpenShift ships a fully managed monitoring stack based on the **Prometheus Operator** in the `openshift-monitoring` namespace.

```
openshift-monitoring namespace:
  ├── Prometheus (cluster metrics: nodes, pods, K8s objects)
  ├── Alertmanager (deduplication, routing, notifications)
  ├── Thanos (long-term storage, federated queries)
  ├── Grafana (dashboards — read-only by default in OCP 4.x)
  ├── Kube-state-metrics (K8s object state metrics)
  ├── Node Exporter (per-node CPU, memory, disk, network)
  └── Telemeter Client (sends anonymized metrics to Red Hat)

openshift-user-workload-monitoring namespace (enabled separately):
  ├── Prometheus (your app metrics)
  └── Alertmanager (your app alerts)
```

```bash
# Check monitoring stack health
oc get pods -n openshift-monitoring
oc get pods -n openshift-user-workload-monitoring   # If enabled

# Access Prometheus UI
oc get route prometheus-k8s -n openshift-monitoring -o jsonpath='{.spec.host}'

# Access Alertmanager UI
oc get route alertmanager-main -n openshift-monitoring -o jsonpath='{.spec.host}'
```

**Enable user workload monitoring:**
```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
    prometheusK8s:
      retention: 15d
      volumeClaimTemplate:
        spec:
          storageClassName: gp3-csi
          resources:
            requests:
              storage: 100Gi
EOF
```

---

### 🟡 Q2. What are the four Golden Signals and how do you implement them in Kubernetes/OpenShift?

**Answer:**

The four Golden Signals (from the Google SRE Book) are the minimum metrics needed to understand a service's health:

| Signal | Description | Metric Example |
|--------|-------------|----------------|
| **Latency** | Time to serve a request (distinguish success vs error latency) | `http_request_duration_seconds` |
| **Traffic** | Demand on the system (requests/sec) | `http_requests_total` |
| **Errors** | Rate of failed requests | `http_requests_total{status=~"5.."}` |
| **Saturation** | How "full" the service is (CPU, memory, queue depth) | `container_cpu_usage_seconds_total` |

**PromQL queries:**

```promql
# Latency — P99 request duration (last 5 min)
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket{job="myapp"}[5m])
)

# Traffic — requests per second
rate(http_requests_total{job="myapp"}[5m])

# Error rate — 5xx errors as % of total
rate(http_requests_total{job="myapp",status=~"5.."}[5m])
  /
rate(http_requests_total{job="myapp"}[5m])
* 100

# Saturation — CPU usage vs limit
rate(container_cpu_usage_seconds_total{container="myapp"}[5m])
  /
container_spec_cpu_quota{container="myapp"}
  * container_spec_cpu_period{container="myapp"}
* 100
```

**Instrument your app:**
```javascript
// Node.js — prom-client
const client = require('prom-client');
const httpDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5]
});

app.use((req, res, next) => {
  const end = httpDuration.startTimer();
  res.on('finish', () => {
    end({ method: req.method, route: req.route?.path, status: res.statusCode });
  });
  next();
});
```

---

## Prometheus & Metrics

---

### 🟡 Q3. How do you expose custom application metrics to Prometheus in OpenShift?

**Answer:**

**Step 1 — Instrument the application** (expose `/metrics` endpoint in Prometheus format):

```go
// Go example with prometheus/client_golang
import "github.com/prometheus/client_golang/prometheus"

var requestsTotal = prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "myapp_requests_total",
        Help: "Total number of requests",
    },
    []string{"method", "endpoint", "status"},
)

func init() { prometheus.MustRegister(requestsTotal) }
```

**Step 2 — Create a Service with named port:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: my-app
  labels:
    app: myapp
spec:
  selector:
    app: myapp
  ports:
  - name: http-metrics     # Name matters for ServiceMonitor
    port: 8080
    targetPort: 8080
```

**Step 3 — Create a ServiceMonitor:**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: my-app
  labels:
    app: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: http-metrics
    path: /metrics
    interval: 30s
    scheme: http
    tlsConfig: {}
```

**Step 4 — Grant monitoring permission (user workload monitoring):**
```bash
oc adm policy add-role-to-user monitoring-edit \
  -z myapp-sa -n my-app
```

```bash
# Verify metrics are being scraped
oc exec -it prometheus-user-workload-0 \
  -n openshift-user-workload-monitoring -- \
  curl -s http://localhost:9090/api/v1/targets | \
  jq '.data.activeTargets[] | select(.labels.job=="myapp")'
```

---

### 🟡 Q4. Explain the difference between a Counter, Gauge, Histogram, and Summary in Prometheus.

**Answer:**

| Type | Description | Use Case | Example |
|------|-------------|---------|---------|
| **Counter** | Monotonically increasing integer — only goes up (or resets to 0 on restart) | Total requests, errors, bytes sent | `http_requests_total` |
| **Gauge** | Can go up or down — current snapshot | Active connections, memory usage, queue depth | `process_memory_bytes` |
| **Histogram** | Counts observations in configurable buckets + sum + count | Request latency, response size | `http_request_duration_seconds_bucket` |
| **Summary** | Client-side percentile calculation (φ-quantiles) | When percentiles are known in advance | `rpc_duration_seconds{quantile="0.99"}` |

```promql
# Counter — rate of change (per-second)
rate(http_requests_total[5m])               # req/sec over 5 min

# Gauge — current value
avg(process_memory_bytes) by (pod)

# Histogram — P50, P95, P99 latency
histogram_quantile(0.95,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)

# Histogram — average latency
rate(http_request_duration_seconds_sum[5m])
  /
rate(http_request_duration_seconds_count[5m])
```

---

## Grafana Dashboards

---

### 🟡 Q5. How do you create a production-ready Grafana dashboard for a Kubernetes application?

**Answer:**

**Dashboard structure for a typical app:**

```
Row 1: Traffic Overview
  - Requests/sec (rate counter)
  - Error rate % (5xx errors / total)
  - Active connections (gauge)

Row 2: Latency
  - P50, P95, P99 latency (histogram_quantile)
  - Request duration heatmap

Row 3: Saturation
  - CPU usage vs requests vs limits
  - Memory usage vs limits
  - Pod count + HPA status

Row 4: Errors
  - 4xx vs 5xx breakdown
  - Error messages (Loki if integrated)

Row 5: Infrastructure
  - Node CPU/memory/disk
  - Network I/O per node
```

**Dashboard as code (JSON provisioning):**
```bash
# Store Grafana dashboards as ConfigMaps — auto-provisioned
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-dashboard
  namespace: openshift-user-workload-monitoring
  labels:
    grafana_dashboard: "true"    # Grafana sidecar picks this up
data:
  myapp-dashboard.json: |
    {
      "title": "My App — Production Dashboard",
      "panels": [...]
    }
EOF
```

**Key PromQL queries to include:**
```promql
# Request rate
sum(rate(http_requests_total{namespace="production",service="myapp"}[2m]))

# 99th percentile latency
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service="myapp"}[5m])) by (le))

# Error ratio
sum(rate(http_requests_total{service="myapp",status=~"5.."}[5m]))
  /
sum(rate(http_requests_total{service="myapp"}[5m]))

# Memory usage vs limit
container_memory_working_set_bytes{container="myapp"}
  /
container_spec_memory_limit_bytes{container="myapp"}
```

---

## Alerting

---

### 🟡 Q6. How do you write effective Prometheus alerting rules? What makes a good alert?

**Answer:**

**Principles of good alerts:**
1. **Alert on symptoms, not causes** — alert when the user is impacted.
2. **Set meaningful thresholds** — based on SLOs, not arbitrary percentages.
3. **Use `for` duration** — don't fire on transient blips.
4. **Include runbook link** — annotate every alert with a link to remediation steps.
5. **Avoid alert fatigue** — too many alerts = ignored alerts.

```yaml
# PrometheusRule for user workload monitoring
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: my-app
  labels:
    openshift.io/prometheus-rule-evaluation-scope: leaf-prometheus
spec:
  groups:
  - name: myapp.rules
    rules:

    # SLO alert: error rate > 1% for 5 minutes
    - alert: HighErrorRate
      expr: |
        rate(http_requests_total{service="myapp",status=~"5.."}[5m])
          /
        rate(http_requests_total{service="myapp"}[5m]) > 0.01
      for: 5m
      labels:
        severity: critical
        team: myapp-team
      annotations:
        summary: "High error rate on myapp"
        description: "Error rate is {{ $value | humanizePercentage }} over the last 5 minutes"
        runbook_url: "https://wiki.example.com/runbooks/myapp-high-error-rate"

    # Latency SLO alert: P99 > 2 seconds
    - alert: HighLatency
      expr: |
        histogram_quantile(0.99,
          rate(http_request_duration_seconds_bucket{service="myapp"}[5m])
        ) > 2.0
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High P99 latency on myapp"
        description: "P99 latency is {{ $value }}s — SLO is 2s"
        runbook_url: "https://wiki.example.com/runbooks/myapp-high-latency"

    # Pod crash alert
    - alert: PodCrashLooping
      expr: |
        rate(kube_pod_container_status_restarts_total{namespace="production"}[15m]) > 0
      for: 15m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash-looping"
        description: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has been restarting for 15+ minutes"

    # Memory pressure alert
    - alert: HighMemoryUsage
      expr: |
        container_memory_working_set_bytes{container="myapp"}
          /
        container_spec_memory_limit_bytes{container="myapp"} > 0.90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "myapp is using >90% memory limit"
```

---

### 🟡 Q7. How do you configure Alertmanager routing and notifications?

**Answer:**

```yaml
# Alertmanager configuration via Secret in OpenShift
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-main
  namespace: openshift-monitoring
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
      slack_api_url: 'https://hooks.slack.com/services/T00/B00/xxx'

    route:
      group_by: [alertname, namespace, severity]
      group_wait: 30s        # Wait before firing a group
      group_interval: 5m     # Interval for repeat notifications
      repeat_interval: 3h    # Don't repeat for 3 hours
      receiver: default

      routes:
      # Critical alerts → PagerDuty (immediate)
      - match:
          severity: critical
        receiver: pagerduty
        repeat_interval: 1h

      # Warning alerts → Slack
      - match:
          severity: warning
        receiver: slack-warnings

      # Silence noisy alerts during maintenance
      - match_re:
          alertname: "Watchdog|InfoInhibitor"
        receiver: null-receiver

    receivers:
    - name: default
      slack_configs:
      - channel: '#alerts-default'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

    - name: pagerduty
      pagerduty_configs:
      - routing_key: 'your-pagerduty-routing-key'
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'

    - name: slack-warnings
      slack_configs:
      - channel: '#alerts-warning'
        title: '⚠️ {{ .GroupLabels.alertname }}'
        text: |
          *Summary:* {{ .CommonAnnotations.summary }}
          *Runbook:* {{ .CommonAnnotations.runbook_url }}

    - name: null-receiver

    inhibit_rules:
    # If critical fires, suppress the warning for the same service
    - source_match:
        severity: critical
      target_match:
        severity: warning
      equal: [alertname, namespace, service]
```

---

## Logging

---

### 🟡 Q8. What is the logging architecture in OpenShift? How does it differ from a custom EFK stack?

**Answer:**

**OpenShift Logging Stack:**
```
Application Pods
      │ (stdout/stderr)
      ▼
Vector/Fluentd Collector (DaemonSet — one per node)
      │ (structured JSON)
      ├──→ OpenShift Elasticsearch / Loki (log store)
      └──→ External: Splunk, CloudWatch, external ES

Kibana / Grafana (visualization)
```

**vs. Self-managed EFK:**
- OpenShift Logging is managed by an Operator — auto-upgrades, HA config, integrated RBAC.
- Access control: users only see logs from namespaces they have `view` role on.
- Red Hat supports it under the OCP subscription.

```bash
# Install OpenShift Logging operator + instance
oc apply -f - <<EOF
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
  name: instance
  namespace: openshift-logging
spec:
  managementState: Managed
  logStore:
    type: lokistack
    lokistack:
      name: logging-loki
  collection:
    type: vector
  visualization:
    type: ocp-console
EOF
```

---

### 🟡 Q9. How do you forward logs to an external system (e.g., Splunk, CloudWatch)?

**Answer:**

Use `ClusterLogForwarder` to route logs to external destinations:

```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  inputs:
  - name: app-logs
    application:
      namespaces: [production, staging]

  - name: infra-logs
    infrastructure: {}

  outputs:
  - name: splunk-output
    type: splunk
    splunk:
      hecToken:
        secretName: splunk-secret
        key: hecToken
      url: https://splunk.example.com:8088
      index: openshift

  - name: cloudwatch-output
    type: cloudwatch
    cloudwatch:
      groupName: openshift-logs
      region: us-east-1
      authentication:
        type: iam_role
        iamRole:
          roleARN: arn:aws:iam::123456789:role/openshift-logging-role

  pipelines:
  - name: app-to-splunk
    inputRefs: [app-logs]
    outputRefs: [splunk-output]
    filterRefs: [parse-json]

  - name: infra-to-cloudwatch
    inputRefs: [infra-logs]
    outputRefs: [cloudwatch-output]

  filters:
  - name: parse-json
    type: parse
```

---

## Troubleshooting Methodology

---

### 🔴 Q10. What is your systematic approach to troubleshooting a production incident on OpenShift?

**Answer:**

**Incident response methodology (USE + 5 Whys):**

```
1. TRIAGE (0–5 min)
   ├── Is this a complete outage or degradation?
   ├── How many users/services affected?
   └── Check Alertmanager / monitoring dashboards immediately

2. INVESTIGATE (5–30 min)
   ├── Recent changes? (oc rollout history, Git commits)
   ├── Check pod status: oc get pods -n prod --sort-by='.status.startTime'
   ├── Check events: oc get events -n prod --sort-by='.lastTimestamp'
   ├── Check pod logs: oc logs <failing-pod> --previous
   ├── Check node health: oc get nodes; oc adm top nodes
   └── Check cluster operators: oc get co

3. HYPOTHESIZE
   ├── Change-related? Correlate with deployment times
   ├── Resource exhaustion? CPU/memory/disk?
   ├── Network issue? Endpoint missing?
   └── Dependency failure? DB, external API?

4. VERIFY
   ├── kubectl/oc exec to test connectivity
   ├── Check metrics in Grafana
   └── Check logs for error patterns

5. REMEDIATE
   ├── Rollback: oc rollout undo deployment/myapp
   ├── Scale up: oc scale deployment/myapp --replicas=10
   ├── Restart: oc rollout restart deployment/myapp
   └── Document root cause for post-mortem

6. POST-MORTEM (after incident)
   ├── Timeline reconstruction
   ├── Root cause analysis (5 Whys)
   ├── Action items to prevent recurrence
   └── Update runbooks
```

---

### 🟡 Q11. How do you troubleshoot a pod stuck in `Pending` state?

**Answer:**

```bash
# Step 1: Describe the pod — check Events section
oc describe pod stuck-pod -n my-app

# Common events and what they mean:
# "0/5 nodes are available: 5 Insufficient cpu."
#   → Increase CPU requests in deployment, or scale cluster
oc adm top nodes
oc describe node worker-1 | grep -A 10 "Allocated resources"

# "0/5 nodes are available: 5 node(s) didn't match node affinity/selector."
#   → nodeSelector label doesn't exist on any node
oc get nodes --show-labels | grep your-label

# "0/5 nodes are available: 5 node(s) had untolerated taint."
#   → Node has a taint the pod doesn't tolerate
oc describe node worker-1 | grep Taints

# "persistentvolumeclaim 'my-pvc' not found"
#   → PVC doesn't exist or is already bound to another pod (RWO)
oc get pvc my-pvc -n my-app

# "pod has unbound immediate PersistentVolumeClaims"
#   → StorageClass can't provision or no PV available
oc get sc
oc describe pvc my-pvc -n my-app

# "Insufficient memory" — reduce requests or add nodes
oc get pods --all-namespaces \
  -o json | jq '.items[].spec.containers[].resources.requests.memory' | \
  sort | uniq -c
```

---

### 🟡 Q12. How do you debug a pod that starts but has network connectivity issues?

**Answer:**

```bash
# Step 1: Verify pod IP assigned
oc get pod myapp -o wide   # Check IP in output

# Step 2: Check Service endpoints
oc get endpoints myapp-svc -n my-app
# If empty → label selector mismatch
oc describe svc myapp-svc | grep Selector
oc get pod myapp --show-labels

# Step 3: Test pod-to-Service DNS
oc exec -it myapp -- nslookup myapp-svc.my-app.svc.cluster.local
# If DNS fails → CoreDNS issue
oc get pods -n openshift-dns

# Step 4: Test pod-to-pod direct
oc exec -it myapp -- curl http://10.128.0.15:8080

# Step 5: Test pod-to-Service
oc exec -it myapp -- curl http://myapp-svc:8080

# Step 6: Check NetworkPolicies
oc get networkpolicies -n my-app

# Step 7: Check Route
oc get route myapp -n my-app
curl -v https://myapp.apps.mycluster.example.com/health

# Step 8: Run netshoot debug pod
oc run debug --rm -it \
  --image=nicolaka/netshoot \
  --restart=Never -- bash
# Inside: tcpdump -i eth0, traceroute, nmap, ss -tulnp
```

---

## Common Issues & Resolutions

---

### 🟡 Q13. How do you diagnose and resolve an OOMKilled container?

**Answer:**

**Diagnosis:**
```bash
# Check exit reason
oc describe pod myapp -n my-app
# Look for: "OOMKilled" under "Last State"

# Check actual memory usage vs limit
oc adm top pods -n my-app --containers
# If usage ≈ limit → container is hitting the ceiling

# Historical usage via Prometheus
# container_memory_working_set_bytes{container="myapp"} over 24h
```

**Resolution options:**

```bash
# Option 1: Increase memory limit
oc set resources deployment/myapp \
  --requests=memory=512Mi \
  --limits=memory=1Gi

# Option 2: Find memory leak in app
# Check heap dumps, profiling, memory growth trend

# Option 3: Use VPA to auto-right-size
kubectl apply -f - <<EOF
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Initial"    # Only sets on pod creation — no live eviction
  resourcePolicy:
    containerPolicies:
    - containerName: myapp
      maxAllowed:
        memory: 2Gi
EOF
```

---

### 🟡 Q14. How do you handle a cluster with high API server load / etcd latency?

**Answer:**

**Symptoms:** API calls slow, `oc` commands timeout, controllers lagging.

```bash
# Step 1: Check API server metrics
oc get --raw /metrics | grep apiserver_request_duration

# Step 2: Check etcd health and latency
oc get etcd -o yaml
oc -n openshift-etcd exec etcd-master-0 -- etcdctl \
  --cacert /etc/kubernetes/static-pod-certs/configmaps/etcd-serving-ca/ca-bundle.crt \
  --cert /etc/kubernetes/static-pod-certs/secrets/etcd-all-certs/etcd-peer-master-0.crt \
  --key /etc/kubernetes/static-pod-certs/secrets/etcd-all-certs/etcd-peer-master-0.key \
  endpoint status --write-out=table

# Step 3: Check for etcd disk latency (etcd is very I/O sensitive)
# etcd needs < 10ms fsync latency
# Alert: etcd_disk_wal_fsync_duration_seconds > 0.01

# Step 4: Reduce API server load
# Check for misbehaving controllers/operators
oc top pod -n kube-system
oc top pod -n openshift-controller-manager

# Step 5: Reduce watch connections
# Find namespaces with huge object counts
oc get pods --all-namespaces | wc -l
oc get configmaps --all-namespaces | wc -l

# Step 6: etcd compaction (runs automatically every 5 min by default)
# But can run manually:
oc -n openshift-etcd exec etcd-master-0 -- etcdctl \
  defrag --cluster

# Step 7: etcd quota — if db size exceeds quota, writes fail
oc -n openshift-etcd exec etcd-master-0 -- etcdctl \
  endpoint status | grep dbSize
```

---

### 🟡 Q15. How do you diagnose a node in `NotReady` state?

**Answer:**

```bash
# Step 1: Check node status and condition
oc get nodes
oc describe node worker-2 | grep -A 20 Conditions

# Step 2: Common conditions and meanings
# MemoryPressure: True → node is low on memory
# DiskPressure: True → node disk is > 85% full
# PIDPressure: True → too many processes running
# Ready: False → kubelet not reporting

# Step 3: SSH to the node (or oc debug)
oc debug node/worker-2

# Step 4: Check kubelet status on the node
systemctl status kubelet
journalctl -u kubelet --since "1 hour ago" | tail -50

# Step 5: Check CRI-O
systemctl status crio
crictl ps    # List running containers
crictl pods  # List pod sandboxes

# Step 6: Check disk
df -h
du -sh /var/lib/containers   # If full, prune old images

# Step 7: Check memory
free -h
oc adm top node worker-2

# Step 8: Force-drain and replace the node (if unrecoverable)
oc adm cordon worker-2
oc adm drain worker-2 --ignore-daemonsets --delete-emptydir-data
# Delete and replace via MachineSet
oc delete machine worker-2-xxx -n openshift-machine-api
# MachineSet auto-creates a replacement node
```

---

### 🔴 Q16. How do you implement SLOs and error budgets in OpenShift?

**Answer:**

**SLO (Service Level Objective):** A target reliability level expressed as a percentage.
**Error Budget:** `100% - SLO%` — the amount of unreliability you're allowed before the SLO is breached.

```
Example: SLO = 99.9% availability
Error budget = 0.1% = 43.8 minutes/month of acceptable downtime
```

**Implementing SLO tracking with Prometheus:**

```yaml
# PrometheusRule: SLO burn rate alerts (multi-window, multi-burn-rate)
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-slo
  namespace: my-app
spec:
  groups:
  - name: slo.rules
    rules:
    # Recording rule: success rate over 5 min
    - record: job:slo_success_rate:ratio_rate5m
      expr: |
        rate(http_requests_total{service="myapp",status!~"5.."}[5m])
          /
        rate(http_requests_total{service="myapp"}[5m])

    # 1h burn rate alert — consuming error budget 14x faster than normal
    - alert: HighBurnRateMyApp
      expr: |
        job:slo_success_rate:ratio_rate5m < 0.999 and
        job:slo_success_rate:ratio_rate1h < 0.999
      for: 2m
      labels:
        severity: critical
        slo: availability
      annotations:
        summary: "High error budget burn rate — SLO at risk"
        description: "Error budget will be exhausted in less than 1 hour at current rate"

    # 6h burn rate alert — consuming 5x faster
    - alert: MediumBurnRateMyApp
      expr: |
        job:slo_success_rate:ratio_rate5m < 0.9994
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "Elevated error budget consumption"
```

```bash
# Calculate remaining error budget for the month
# Total requests this month: 10,000,000
# Failed requests: 12,000
# Error rate: 0.12% — within 0.1% SLO? No! → SLO breached

# Grafana: error budget panel
# (target_availability - actual_availability) / (1 - target_availability) * 100
# = (0.999 - 0.9988) / 0.001 * 100 = 20% of error budget remaining
```

---

### 🟡 Q17. How do you use `oc adm must-gather` for cluster diagnostics?

**Answer:**

`oc adm must-gather` collects diagnostic data from the cluster into a tarball — used for Red Hat support cases and deep debugging.

```bash
# Basic must-gather (collects from all operators)
oc adm must-gather

# Specify output directory
oc adm must-gather --dest-dir=/tmp/cluster-diagnostics

# Target specific operators
oc adm must-gather \
  --image=registry.redhat.io/openshift4/ose-logging-must-gather:latest \
  -- /usr/bin/gather

# Must-gather for network diagnostics
oc adm must-gather \
  --image=registry.redhat.io/openshift4/network-tools-rhel8:latest

# What gets collected:
# - All cluster operator statuses
# - Node information and logs
# - API resources (pods, deployments, services, events)
# - MachineConfig and MachineConfigPool state
# - Prometheus metrics snapshot
# - Audit logs

# Inspect the output
ls must-gather.local.*/
cat must-gather.local.*/cluster-scoped-resources/config.openshift.io/clusterversions/version.yaml
```

---

### 🔴 Q18. How do you implement distributed tracing in a containerized microservices environment?

**Answer:**

**OpenTelemetry + Jaeger / Tempo + OpenShift:**

```yaml
# Install Red Hat OpenShift distributed tracing operator
# Provides: Jaeger Operator + OpenTelemetry Operator

# Create a Jaeger instance
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
  namespace: openshift-distributed-tracing
spec:
  strategy: production
  storage:
    type: elasticsearch
    elasticsearch:
      nodeCount: 3
      resources:
        requests:
          cpu: 500m
          memory: 2Gi
  ingress:
    enabled: true
```

**Instrument app with OpenTelemetry SDK:**
```javascript
// Node.js auto-instrumentation
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');

const provider = new NodeTracerProvider();
const exporter = new OTLPTraceExporter({
  url: 'http://otel-collector.openshift-distributed-tracing.svc:4318/v1/traces',
});
provider.register();
```

```yaml
# OpenTelemetry Collector (receives, processes, exports traces)
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: my-app
spec:
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
          http:

    processors:
      batch:
        timeout: 1s

    exporters:
      jaeger:
        endpoint: jaeger-production-collector:14250
        tls:
          insecure: true

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [jaeger]
```

---

> **Next:** [Infrastructure & Automation →](./05-Infrastructure-Automation.md)