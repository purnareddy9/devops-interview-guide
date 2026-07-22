---
render_with_liquid: false
layout: default
title: "📊 Monitoring — DevOps Interview Guide"
---

# 📊 Monitoring — DevOps Interview Guide

> [← CI/CD](./CICD.md) | [Main Index](./README.md) | [Logging →](./Logging.md)

---

## Table of Contents

1. [Monitoring Fundamentals](#monitoring-fundamentals)
2. [Prometheus](#prometheus)
3. [Grafana](#grafana)
4. [Alerting](#alerting)
5. [SLOs & Error Budgets](#slos--error-budgets)
6. [Distributed Tracing (Jaeger/Tempo)](#distributed-tracing)
7. [Interview Questions](#interview-questions)
8. [Scenario-Based Questions](#scenario-based-questions)
9. [Hands-On Labs](#hands-on-labs)
10. [Cheat Sheet](#cheat-sheet)

---

## Monitoring Fundamentals

### The Four Golden Signals

| Signal | Description | Example Metrics |
|--------|-------------|----------------|
| **Latency** | Time to serve a request (success AND error) | p50, p95, p99 response time |
| **Traffic** | Demand on your system | Requests/second, transactions/sec |
| **Errors** | Rate of failed requests | HTTP 5xx rate, exception count |
| **Saturation** | How full your service is | CPU %, memory %, queue depth |

### RED Method (for services)

```
Rate     → Requests per second
Errors   → Requests that are failing (per second)
Duration → Distribution of response times (latency)
```

### USE Method (for resources)

```
Utilization  → % time the resource is busy
Saturation   → Amount of queued work (waiting)
Errors       → Count of error events
```

### Observability Pillars

```
Metrics   → Aggregated numerical data over time (Prometheus, CloudWatch)
Logs      → Event records with context (Elasticsearch, Loki)
Traces    → Request flow through distributed systems (Jaeger, Zipkin, Tempo)
Profiles  → Code-level CPU/memory usage (Pyroscope, pprof)
```

---

## Prometheus

### Architecture

```
┌──────────────────────────────────────────────────────┐
│                    Prometheus Server                   │
│                                                        │
│  ┌────────────┐  ┌────────────────┐  ┌────────────┐  │
│  │  Retrieval │  │   TSDB (Time   │  │  HTTP API  │  │
│  │ (scrapers) │  │  Series DB)    │  │ + PromQL   │  │
│  └────────────┘  └────────────────┘  └────────────┘  │
│  ┌────────────────────────────────────────────────┐   │
│  │             Alertmanager                        │   │
│  └────────────────────────────────────────────────┘   │
└─────────────────────┬────────────────────────────────┘
                      │ Pull (scrape)
         ┌────────────┼────────────────────────┐
         ▼            ▼                        ▼
   App (metrics)  Node Exporter         kube-state-metrics
   /metrics        (host metrics)       (K8s object metrics)
```

### prometheus.yml Configuration

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: production
    region: us-east-1

rule_files:
  - "alerts/*.yaml"
  - "recording_rules/*.yaml"

alerting:
  alertmanagers:
  - static_configs:
    - targets: ['alertmanager:9093']

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
    - role: pod
    relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)
    - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
      target_label: __address__

  - job_name: 'node-exporter'
    static_configs:
    - targets: ['node-exporter:9100']

  - job_name: 'kube-state-metrics'
    static_configs:
    - targets: ['kube-state-metrics:8080']

  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
    - targets:
      - https://myapp.example.com/health
      - https://api.example.com/status
    relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - source_labels: [__param_target]
      target_label: instance
    - target_label: __address__
      replacement: blackbox-exporter:9115
```

### PromQL Query Language

```promql
# ─── BASIC QUERIES ────────────────────────────────────
# Current value of a metric
http_requests_total

# Filter by label
http_requests_total{job="myapp", status="200"}

# Rate (per second over 5 min window)
rate(http_requests_total[5m])

# Increase (total increase over window)
increase(http_requests_total[1h])

# ─── AGGREGATIONS ─────────────────────────────────────
# Sum by label
sum(rate(http_requests_total[5m])) by (service)

# Average
avg(node_memory_MemAvailable_bytes) by (node)

# 99th percentile latency
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
)

# Top 5 memory consumers
topk(5, sum(container_memory_working_set_bytes) by (pod))

# ─── ERROR RATE ───────────────────────────────────────
# HTTP error rate (5xx)
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))

# ─── RESOURCE QUERIES ─────────────────────────────────
# CPU usage per pod
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)

# Memory usage
sum(container_memory_working_set_bytes{container!=""}) by (pod)

# Disk IOPS
rate(node_disk_io_time_seconds_total[5m])

# ─── RECORDING RULES (pre-computed queries) ───────────
# In recording_rules/app.yaml:
groups:
- name: app_rules
  interval: 30s
  rules:
  - record: job:http_requests:rate5m
    expr: |
      sum(rate(http_requests_total[5m])) by (job)

  - record: job:http_errors:rate5m
    expr: |
      sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
```

### Prometheus Operator (Kubernetes)

```yaml
# ServiceMonitor — tell Prometheus which services to scrape
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
    honorLabels: true
  namespaceSelector:
    matchNames:
    - production

---
# PodMonitor — scrape pods directly
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: myapp-pods
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: myapp
  podMetricsEndpoints:
  - port: metrics
    interval: 15s
```

---

## Grafana

### Dashboard Best Practices

```json
// Grafana dashboard panel structure
{
  "title": "Request Rate",
  "type": "timeseries",
  "targets": [
    {
      "expr": "sum(rate(http_requests_total[5m])) by (service)",
      "legendFormat": "{{service}}"
    }
  ],
  "fieldConfig": {
    "defaults": {
      "unit": "reqps",
      "min": 0,
      "color": { "mode": "palette-classic" },
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 1000 },
          { "color": "red", "value": 5000 }
        ]
      }
    }
  }
}
```

### Grafana Provisioning (as Code)

```yaml
# provisioning/datasources/prometheus.yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: "15s"
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100

  - name: Tempo
    type: tempo
    access: proxy
    url: http://tempo:3200
```

```yaml
# provisioning/dashboards/dashboard-provider.yaml
apiVersion: 1
providers:
  - name: default
    folder: ''
    type: file
    disableDeletion: false
    editable: true
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true
```

---

## Alerting

### Alert Rules

```yaml
# alerts/app.yaml
groups:
- name: application
  rules:

  # High error rate
  - alert: HighErrorRate
    expr: |
      (
        sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
        /
        sum(rate(http_requests_total[5m])) by (service)
      ) > 0.01
    for: 5m
    labels:
      severity: critical
      team: platform
    annotations:
      summary: "High error rate for {{ $labels.service }}"
      description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }} (threshold: 1%)"
      runbook: "https://wiki.example.com/runbooks/high-error-rate"

  # High latency
  - alert: HighLatency
    expr: |
      histogram_quantile(0.95,
        sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)
      ) > 0.5
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "High p95 latency for {{ $labels.service }}"
      description: "p95 latency is {{ $value | humanizeDuration }} (threshold: 500ms)"

  # Pod crashlooping
  - alert: PodCrashLooping
    expr: |
      rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 0
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} is crash looping"

  # Node disk pressure
  - alert: NodeDiskUsageHigh
    expr: |
      (
        (node_filesystem_size_bytes - node_filesystem_avail_bytes)
        / node_filesystem_size_bytes
      ) > 0.85
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High disk usage on {{ $labels.instance }}"
      description: "Disk {{ $labels.mountpoint }} at {{ $value | humanizePercentage }}"

  # No data / service down
  - alert: ServiceDown
    expr: up{job="myapp"} == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Service {{ $labels.job }} is down on {{ $labels.instance }}"
```

### Alertmanager Configuration

```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alerts@example.com'
  slack_api_url: 'https://hooks.slack.com/services/...'

templates:
- '/etc/alertmanager/templates/*.tmpl'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s       # Wait for related alerts to group
  group_interval: 5m    # Wait before sending new group notification
  repeat_interval: 4h   # Wait before re-sending same alert

  receiver: default
  routes:
  - match:
      severity: critical
    receiver: pagerduty-critical
    continue: true    # Also send to default

  - match:
      severity: warning
    receiver: slack-warnings

  - match:
      team: database
    receiver: dba-team
    group_by: ['alertname']

receivers:
- name: default
  slack_configs:
  - channel: '#alerts'
    title: '{{ template "slack.title" . }}'
    text: '{{ template "slack.text" . }}'

- name: pagerduty-critical
  pagerduty_configs:
  - service_key: 'YOUR_PAGERDUTY_KEY'

- name: slack-warnings
  slack_configs:
  - channel: '#warnings'
    send_resolved: true

- name: dba-team
  email_configs:
  - to: 'dba@example.com'

inhibit_rules:
- source_match:
    severity: critical
  target_match:
    severity: warning
  equal: ['alertname', 'instance']  # Suppress warning if critical is firing
```

---

## SLOs & Error Budgets

### Defining SLIs, SLOs, SLAs

```
SLI (Service Level Indicator)
  → Quantitative measure of service performance
  → Examples: availability, latency, error rate, throughput

SLO (Service Level Objective)
  → Target value for an SLI
  → Examples: 99.9% availability, p99 latency < 200ms

SLA (Service Level Agreement)
  → Contract with consequences for missing SLOs
  → SLA ≤ SLO (SLO is your internal target; SLA is the customer promise)

Error Budget = 100% - SLO
  → 99.9% SLO = 0.1% error budget = 43.8 min/month of allowed downtime
  → When error budget is exhausted → freeze feature deployments, fix reliability
```

### Error Budget Burn Rate Alert

```yaml
# Multi-window burn rate alerting (Google SRE approach)
- alert: ErrorBudgetBurnRateFast
  expr: |
    (
      # 1h rate > 14x budget burn rate
      sum(rate(http_requests_total{status=~"5.."}[1h])) by (service)
      /
      sum(rate(http_requests_total[1h])) by (service)
    ) > 0.14  # 14 * (1 - 0.999) = 14 * 0.001
    AND
    (
      # 5m rate also elevated (prevents false alarms)
      sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
      /
      sum(rate(http_requests_total[5m])) by (service)
    ) > 0.14
  for: 2m
  labels:
    severity: critical
    page: true
  annotations:
    summary: "Fast burn: {{ $labels.service }} will exhaust error budget in <1h"
```

---

## Distributed Tracing

### OpenTelemetry Instrumentation

```python
# Python FastAPI with OpenTelemetry
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

# Setup tracing
provider = TracerProvider(
    resource=Resource.create({
        SERVICE_NAME: "myapp",
        SERVICE_VERSION: "1.0.0",
        DEPLOYMENT_ENVIRONMENT: "production",
    })
)

provider.add_span_processor(
    BatchSpanProcessor(
        OTLPSpanExporter(endpoint="http://otel-collector:4317")
    )
)

trace.set_tracer_provider(provider)

# Auto-instrument FastAPI
FastAPIInstrumentor.instrument_app(app)
HTTPXClientInstrumentor().instrument()
SQLAlchemyInstrumentor().instrument(engine=engine)

# Manual span
tracer = trace.get_tracer(__name__)

async def process_order(order_id: str):
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)
        span.set_attribute("order.source", "api")

        result = await fetch_order(order_id)
        span.add_event("order_fetched", {"items": len(result.items)})

        return result
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is the difference between monitoring and observability?**

> **Monitoring** tells you when something is wrong — it's about known failure modes you've anticipated (alerts on CPU > 90%, 5xx rate spike). **Observability** is the ability to understand the internal state of a system from its external outputs — it lets you investigate *unknown* failures using metrics, logs, and traces. Monitoring asks "is it down?"; observability asks "why is this happening?" and lets you explore without needing to predict the failure in advance.

**Q2: What is Prometheus and how does it collect metrics?**

> Prometheus is an open-source monitoring system using a **pull model** — it scrapes HTTP `/metrics` endpoints from targets at a configured interval. Metrics are stored in a time-series database. It uses **PromQL** for querying. Pull model advantages: Prometheus controls scrape rate, can detect if a target is down, easier to debug. Pushgateway exists for short-lived jobs that can't be scraped.

**Q3: What are the four types of Prometheus metrics?**

> - **Counter**: monotonically increasing value (total requests, total errors). Use `rate()` or `increase()` to get rate.
> - **Gauge**: value that can go up and down (current CPU%, queue depth, temperature)
> - **Histogram**: distribution of values in configurable buckets (request latency). Use `histogram_quantile()` for percentiles.
> - **Summary**: pre-computed percentiles on the client side (less flexible than histogram for aggregation).

---

### 🟡 Intermediate

**Q4: What is a Grafana datasource vs a dashboard?**

> A **datasource** is the backend Grafana queries for data: Prometheus, Loki, CloudWatch, InfluxDB, etc. A **dashboard** is a visual collection of panels (graphs, tables, gauges) that query datasources using their respective query languages (PromQL, LogQL, etc.). Dashboards can be version-controlled as JSON, provisioned via config files, and shared via Grafana.com.

**Q5: Explain SLI, SLO, and error budget with a practical example.**

> For an e-commerce checkout API:
> - **SLI**: `% of checkout requests completing in < 500ms` = `successful_fast_requests / total_requests`
> - **SLO**: That SLI should be ≥ 99.5% over a 30-day rolling window
> - **Error budget**: 0.5% = ~3.6 hours of allowed "budget" per month
> If we've used 80% of the error budget by day 20, we stop new feature deployments and focus on reliability. If budget is healthy, we can take more deployment risks.

**Q6: What is Alertmanager and how does it work with Prometheus?**

> Prometheus evaluates alert rules and fires alerts to **Alertmanager** when conditions are met. Alertmanager handles: **deduplication** (same alert from multiple Prometheus instances), **grouping** (related alerts into one notification), **silencing** (suppress known maintenance alerts), **inhibition** (suppress low-severity if high-severity firing), **routing** (send to PagerDuty/Slack/email based on labels). Prometheus fires and forgets; Alertmanager manages the notification lifecycle.

---

### 🔴 Advanced

**Q7: How do you implement multi-window burn rate alerting?**

> The Google SRE approach: use two time windows to reduce false positives. A **short window** (1h) catches fast-burning issues; a **long window** (6h/24h/72h) catches slow burns. Alert only when both windows exceed a threshold. Burn rate = how fast you're consuming error budget relative to normal. At 14x burn rate for 1 hour = budget exhausted in ~2 hours = page immediately. At 1x burn rate = budget depletes at normal rate = no alert needed.

**Q8: How would you monitor a microservices architecture?**

> **Service-level**: RED metrics per service (rate, errors, duration) via Prometheus + service mesh (Istio/Linkerd auto-generates metrics without code changes).
> **Infrastructure**: USE metrics per node (node-exporter), K8s state (kube-state-metrics).
> **Application**: Custom business metrics (orders/min, payment success rate) in code.
> **Distributed tracing**: OpenTelemetry → Tempo/Jaeger for request correlation across services.
> **Dashboards**: Per-service dashboards + cross-service dependency graphs.
> **Alerting**: Alert on SLO burn rates, not individual symptoms.

---

## Scenario-Based Questions

### 🔵 Scenario 1: Alert Fired — 5xx Rate High

*"You receive a PagerDuty alert at 3 AM: error rate for checkout service > 5%."*

```bash
# 1. Check the alert in Alertmanager/Grafana
# Open runbook link from alert annotation

# 2. Assess impact
# - How many users affected? (request rate * error %)
# - Which specific errors? (check logs)
# - When did it start? (correlate with recent deploys)

# 3. Check service status
kubectl get pods -n production -l app=checkout
kubectl logs -l app=checkout -n production --tail=100 | grep -i error

# 4. Check dependencies (is it upstream/downstream?)
# Look at distributed traces for failed requests
# Check DB connection pool, external API latency

# 5. Check recent deployments
kubectl rollout history deployment/checkout -n production

# 6. Decide: rollback or fix forward
kubectl rollout undo deployment/checkout -n production  # If recent deploy
```

---

## Hands-On Labs

### Lab 1: Deploy Prometheus Stack with Helm

```bash
# Install kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.resources.limits.memory=4Gi

# Access Grafana
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring

# Access Prometheus
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring

# Access Alertmanager
kubectl port-forward svc/monitoring-kube-prometheus-alertmanager 9093:9093 -n monitoring
```

---

## Cheat Sheet

```promql
# ─── COMMON PROMQL PATTERNS ───────────────────────────
# Request rate
rate(http_requests_total[5m])

# Error rate %
sum(rate(http_requests_total{code=~"5.."}[5m])) /
sum(rate(http_requests_total[5m]))

# p99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# CPU by pod
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)

# Memory by pod (working set)
sum(container_memory_working_set_bytes{container!=""}) by (pod)

# Pod restarts
rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15

# Node disk usage %
100 - (node_filesystem_avail_bytes / node_filesystem_size_bytes * 100)

# ─── GRAFANA CLI ──────────────────────────────────────
grafana-cli plugins install grafana-piechart-panel
grafana-cli admin reset-admin-password newpassword

# ─── PROMETHEUS CLI ───────────────────────────────────
# Test PromQL
curl 'localhost:9090/api/v1/query?query=up'

# Check targets
curl 'localhost:9090/api/v1/targets' | jq '.data.activeTargets[] | {job, health}'

# Reload config
curl -X POST localhost:9090/-/reload
```

---

> **Cross-links:** [← CI/CD](./CICD.md) | [Logging →](./Logging.md) | [SRE (SLOs/error budgets) →](./SRE.md) | [Kubernetes (monitoring on K8s) →](./Kubernetes.md)