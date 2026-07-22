---
layout: default
title: "📝 Logging — DevOps Interview Guide"
render_with_liquid: false
---

# 📝 Logging — DevOps Interview Guide

> [← Monitoring](./Monitoring.md) | [Main Index](./README.md) | [Security →](./Security.md)

---

## Table of Contents

1. [Logging Fundamentals](#logging-fundamentals)
2. [ELK Stack](#elk-stack)
3. [Loki + Promtail + Grafana](#loki--promtail--grafana)
4. [Fluentd / Fluent Bit](#fluentd--fluent-bit)
5. [Structured Logging](#structured-logging)
6. [Log Analysis & Queries](#log-analysis--queries)
7. [Interview Questions](#interview-questions)
8. [Scenario-Based Questions](#scenario-based-questions)
9. [Hands-On Labs](#hands-on-labs)

---

## Logging Fundamentals

### Log Levels

| Level | Description | Use Case |
|-------|-------------|---------|
| TRACE | Most verbose | Deep debugging, method entries |
| DEBUG | Detailed diagnostic info | Development debugging |
| INFO | Normal operation events | Application milestones, state changes |
| WARN | Potential issues, recoverable | Deprecated usage, high latency |
| ERROR | Errors that need attention | Failed operations, caught exceptions |
| FATAL/CRITICAL | System cannot continue | Unrecoverable errors, crash |

### Structured vs Unstructured Logs

```
# Unstructured (hard to parse, search, and aggregate)
2024-01-15 14:23:01 INFO  User john@example.com logged in from 192.168.1.1

# Structured JSON (easy to query, filter, aggregate)
{
  "timestamp": "2024-01-15T14:23:01.123Z",
  "level": "INFO",
  "message": "User logged in",
  "user_id": "user_123",
  "email": "john@example.com",
  "ip": "192.168.1.1",
  "duration_ms": 45,
  "trace_id": "abc123def456",
  "service": "auth-service",
  "version": "1.2.3",
  "env": "production"
}
```

### Logging Architecture

```
Application Pods
       │ stdout/stderr
       │
   Log Collector (Fluent Bit / Promtail)
   running as DaemonSet on each node
       │
       │ (parse, filter, enrich)
       │
   Log Aggregator (Logstash / Fluentd)
   (optional transformation layer)
       │
       │ (ship to backend)
       │
   Log Storage + Search
   (Elasticsearch, Loki, Splunk, CloudWatch)
       │
       │
   Visualization + Alerting
   (Kibana, Grafana, Splunk UI)
```

---

## ELK Stack

### Components

| Component | Role |
|-----------|------|
| **Elasticsearch** | Distributed search and analytics engine (stores logs) |
| **Logstash** | Data pipeline (ingest, transform, ship) |
| **Kibana** | Visualization and dashboarding |
| **Beats** (Filebeat, Metricbeat) | Lightweight shippers (modern replacement for Logstash) |

### Elasticsearch

```bash
# Index management
# Create index template
curl -X PUT "localhost:9200/_index_template/logs" -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs-policy"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "level": { "type": "keyword" },
        "message": { "type": "text" },
        "service": { "type": "keyword" },
        "trace_id": { "type": "keyword" }
      }
    }
  }
}'

# Search queries
# Find all errors in last hour
curl -X GET "localhost:9200/logs-*/_search" -d'
{
  "query": {
    "bool": {
      "must": [
        { "term": { "level": "ERROR" } },
        { "range": { "@timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "sort": [{ "@timestamp": { "order": "desc" } }],
  "size": 100
}'

# Aggregations — error count per service
curl -X GET "localhost:9200/logs-*/_search" -d'
{
  "query": { "term": { "level": "ERROR" } },
  "aggs": {
    "by_service": {
      "terms": { "field": "service", "size": 10 }
    }
  },
  "size": 0
}'
```

### ILM (Index Lifecycle Management)

```json
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "7d"
          },
          "set_priority": { "priority": 100 }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 },
          "set_priority": { "priority": 50 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {},
          "set_priority": { "priority": 0 }
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": { "delete": {} }
      }
    }
  }
}
```

### Logstash Pipeline

```ruby
# logstash.conf
input {
  beats {
    port => 5044
    ssl  => true
    ssl_certificate => "/etc/logstash/certs/logstash.crt"
    ssl_key         => "/etc/logstash/certs/logstash.key"
  }
  kafka {
    bootstrap_servers => "kafka:9092"
    topics            => ["app-logs"]
    codec             => json
  }
}

filter {
  # Parse JSON logs
  if [message] =~ /^\{/ {
    json {
      source => "message"
    }
  }

  # Parse nginx access logs
  if [fields][log_type] == "nginx" {
    grok {
      match => {
        "message" => '%{IPORHOST:client_ip} - %{DATA:user} \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{DATA:path} HTTP/%{NUMBER:http_version}" %{NUMBER:response_code} %{NUMBER:bytes} "%{DATA:referrer}" "%{DATA:user_agent}" %{NUMBER:response_time}'
      }
    }
    mutate {
      convert => { "response_code" => "integer" }
      convert => { "response_time" => "float" }
    }
  }

  # Add geo IP
  geoip {
    source => "client_ip"
    target => "geoip"
  }

  # Drop health check logs (reduce noise)
  if [path] == "/health" or [path] == "/metrics" {
    drop { }
  }

  # Add environment tag
  mutate {
    add_field => { "environment" => "%{[fields][env]}" }
  }

  # Remove sensitive fields
  mutate {
    remove_field => [ "password", "token", "credit_card" ]
  }

  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts    => ["https://elasticsearch:9200"]
    user     => "logstash_writer"
    password => "${LOGSTASH_PASSWORD}"
    index    => "logs-%{[fields][service]}-%{+YYYY.MM.dd}"
    ilm_rollover_alias => "logs-%{[fields][service]}"
    ilm_policy => "logs-policy"
  }
}
```

### Kibana DSL Queries

```
# KQL (Kibana Query Language — simpler)
level: "ERROR" AND service: "checkout-service"
response_code >= 500 AND response_code < 600
message: "database" AND NOT level: "DEBUG"

# Lucene query syntax
level:ERROR AND service:checkout*
@timestamp:[now-1h TO now] AND level:ERROR

# EQL (Event Query Language — sequences)
sequence by trace_id
  [any where level == "ERROR"]
  [any where level == "FATAL"]
```

---

## Loki + Promtail + Grafana

### Why Loki?

```
Prometheus approach applied to logs:
- Only indexes labels (not full-text indexed like Elasticsearch)
- Much lower resource usage and cost
- Native Grafana integration (same interface as metrics)
- LogQL is similar to PromQL
- Best for Kubernetes environments (natural label integration)
- Trade-off: slower full-text search than Elasticsearch
```

### Loki Configuration

```yaml
# loki-config.yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://alertmanager:9093

limits_config:
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 32
  max_entries_limit_per_query: 5000
  retention_period: 30d
```

### Promtail Configuration (Kubernetes)

```yaml
# promtail-config.yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
    - role: pod

    pipeline_stages:
    # Parse JSON logs
    - json:
        expressions:
          level: level
          msg: message
          trace_id: trace_id

    # Extract log level
    - labels:
        level:
        trace_id:

    # Drop debug logs in production
    - drop:
        expression: '.*level=debug.*'
        drop_counter_reason: debug_log

    relabel_configs:
    - source_labels: [__meta_kubernetes_namespace]
      target_label: namespace
    - source_labels: [__meta_kubernetes_pod_name]
      target_label: pod
    - source_labels: [__meta_kubernetes_pod_label_app]
      target_label: app
    - source_labels: [__meta_kubernetes_pod_container_name]
      target_label: container
```

### LogQL Queries

```logql
# ─── BASIC LOG QUERIES ────────────────────────────────
# All logs from checkout service
{app="checkout", namespace="production"}

# Filter for errors
{app="checkout"} |= "ERROR"

# Exclude health checks
{app="nginx"} != "/health"

# Regex filter
{app="myapp"} |~ "error|exception|panic"

# ─── PARSING ──────────────────────────────────────────
# Parse JSON
{app="myapp"} | json | level="ERROR"

# Parse logfmt
{app="myapp"} | logfmt | duration > 100ms

# Pattern matching
{app="nginx"} | pattern '<ip> - - [<_>] "<method> <path> <_>" <status> <size>'
    | status >= 500

# ─── METRIC QUERIES (LogQL) ───────────────────────────
# Error rate per minute
sum(rate({app="myapp"} |= "ERROR" [1m])) by (pod)

# Request rate from nginx
sum(rate({app="nginx"}
    | pattern `<_> - - [<_>] "<_ <_> <_>" <status> <_>`
    [1m])) by (status)

# Log volume by service
sum(bytes_over_time({namespace="production"}[1h])) by (app)

# 95th percentile duration
quantile_over_time(0.95,
    {app="api"} | json | unwrap duration_ms [5m])
    by (endpoint)
```

---

## Fluentd / Fluent Bit

### Fluent Bit (DaemonSet on Kubernetes)

```yaml
# fluent-bit-config.yaml
[SERVICE]
    Flush         5
    Log_Level     info
    Daemon        off
    Parsers_File  parsers.conf
    HTTP_Server   On
    HTTP_Listen   0.0.0.0
    HTTP_Port     2020

[INPUT]
    Name              tail
    Tag               kube.*
    Path              /var/log/containers/*.log
    Parser            docker
    DB                /var/log/flb_kube.db
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On
    Refresh_Interval  10

[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
    Kube_Tag_Prefix     kube.var.log.containers.
    Merge_Log           On
    Merge_Log_Key       log_processed
    K8S-Logging.Parser  On

[FILTER]
    Name    grep
    Match   kube.*
    Exclude log /health|/metrics|/readiness

[FILTER]
    Name    record_modifier
    Match   kube.*
    Record  cluster production-eks
    Record  region us-east-1

[OUTPUT]
    Name            es
    Match           kube.*
    Host            elasticsearch.logging.svc.cluster.local
    Port            9200
    Logstash_Format On
    Logstash_Prefix k8s-logs
    Include_Tag_Key On
    Time_Key        @timestamp
    Retry_Limit     5
    tls             On
    tls.verify      Off
    HTTP_User       fluent-bit
    HTTP_Passwd     ${ELASTICSEARCH_PASSWORD}
```

---

## Structured Logging

### Application Logging Best Practices

```python
# Python — structlog
import structlog
import logging

structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_log_level,
        structlog.stdlib.add_logger_name,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)

log = structlog.get_logger()

# Request middleware — add trace_id to all logs in request
class LoggingMiddleware:
    async def __call__(self, request, call_next):
        trace_id = request.headers.get("X-Trace-ID", generate_trace_id())

        # Bind context for all subsequent logs in this request
        with structlog.contextvars.bound_contextvars(
            trace_id=trace_id,
            user_id=request.state.user_id if hasattr(request.state, 'user_id') else None,
            path=request.url.path,
            method=request.method,
        ):
            start_time = time.time()
            response = await call_next(request)
            duration_ms = (time.time() - start_time) * 1000

            log.info(
                "request_completed",
                status_code=response.status_code,
                duration_ms=round(duration_ms, 2),
            )

            return response
```

```go
// Go — zerolog
package main

import (
    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
    "os"
    "time"
)

func init() {
    // Configure JSON output
    zerolog.TimeFieldFormat = zerolog.TimeFormatUnixMs
    zerolog.SetGlobalLevel(zerolog.InfoLevel)
}

func processOrder(ctx context.Context, orderID string) error {
    logger := log.With().
        Str("trace_id", getTraceID(ctx)).
        Str("order_id", orderID).
        Logger()

    start := time.Now()

    logger.Info().Msg("processing order")

    result, err := fetchOrder(ctx, orderID)
    if err != nil {
        logger.Error().
            Err(err).
            Dur("duration", time.Since(start)).
            Msg("failed to fetch order")
        return err
    }

    logger.Info().
        Int("item_count", len(result.Items)).
        Float64("total_amount", result.Total).
        Dur("duration", time.Since(start)).
        Msg("order processed successfully")

    return nil
}
```

---

## Log Analysis & Queries

### Common Investigation Patterns

```bash
# ─── COMMAND LINE LOG ANALYSIS ───────────────────────

# Count errors per hour
awk '{print $1}' /var/log/app.log | sort | uniq -c | sort -rn | head -24

# Top 10 error messages
grep "ERROR" /var/log/app.log | awk -F'message":"' '{print $2}' | \
  awk -F'"' '{print $1}' | sort | uniq -c | sort -rn | head -10

# Find slow requests (>1s)
jq 'select(.duration_ms > 1000) | {timestamp, path, duration_ms}' \
  /var/log/app.log | head -20

# Error rate per minute
awk '{print substr($1,1,16)}' /var/log/app.log | \
  sort | uniq -c | awk '$2 ~ /ERROR/'

# Unique users with errors
grep "ERROR" /var/log/app.log | jq -r '.user_id' | \
  sort | uniq -c | sort -rn | head -10
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is the difference between logging and monitoring?**

> **Logging** records discrete events with context — "what happened at this moment" (request received, error thrown, user logged in). Logs are human-readable event records. **Monitoring** aggregates metrics over time — "how is the system performing right now" (CPU 85%, 100 req/s, error rate 0.1%). Logs answer "what happened?"; monitoring answers "how is it going?". Both are needed — monitoring alerts you to a problem, logs help you diagnose it.

**Q2: What is the ELK stack?**

> **E**lasticsearch — full-text search and analytics database that stores logs. **L**ogstash — data pipeline that ingests, parses, transforms, and ships logs to Elasticsearch. **K**ibana — web UI for searching, visualizing, and dashboarding Elasticsearch data. Modern deployments often replace Logstash with the lighter **Beats** (Filebeat for logs, Metricbeat for metrics) and the stack is called Elastic Stack.

**Q3: What is log rotation and why is it important?**

> Log rotation automatically compresses, archives, and deletes old log files to prevent disk space exhaustion. Without rotation, log files grow indefinitely. Tools: `logrotate` (Linux), `systemd` journal size limits. Configuration includes: max file size, max age, number of files to retain, compression. In Kubernetes: containers write to stdout/stderr, and the container runtime handles rotation via `--log-opt max-size` and `--log-opt max-file`.

---

### 🟡 Intermediate

**Q4: How would you centralize logs from a Kubernetes cluster?**

> Deploy a **DaemonSet** running **Fluent Bit** (lightweight) or **Promtail** on every node. It reads container logs from `/var/log/containers/`, enriches them with Kubernetes metadata (namespace, pod, container name, labels), and ships to a centralized backend (Elasticsearch/Loki). Alternatively, use the **sidecar pattern** for applications that can't write to stdout. Cloud providers have native solutions: AWS CloudWatch Container Insights, GCP Cloud Logging, Azure Monitor.

**Q5: What is the difference between Loki and Elasticsearch for log storage?**

> **Elasticsearch** indexes the full text of every log line — enables fast full-text search, complex queries, and analytics. Higher resource usage and cost. Best for compliance, security, and complex queries. **Loki** only indexes labels (similar to Prometheus) — much lower resource usage and cost. Full-text search is done by grep-like scanning, which is slower but acceptable for most use cases. Best for Kubernetes environments already using Prometheus/Grafana.

**Q6: What are structured logs and why are they better than unstructured?**

> Structured logs are machine-parseable records (JSON, logfmt) with well-defined fields. Benefits: **queryable** (filter by `level=ERROR AND user_id=123`), **aggregatable** (count errors by service), **consistent** (all services use same format), **automation-friendly** (dashboards, alerts can use specific fields). Unstructured logs require fragile regex parsing. Always use structured logging in production applications.

---

### 🔴 Advanced

**Q7: How do you handle sensitive data (PII) in logs?**

> - **Prevention**: never log passwords, tokens, credit cards, SSNs at the application level — sanitize before logging
> - **Masking**: log pipeline (Logstash filter) removes or masks sensitive fields: `remove_field => ["password", "token"]`
> - **Hashing**: hash user IDs/emails for correlation without exposing PII: `sha256(user_id)`
> - **Access control**: restrict access to production logs (RBAC on Kibana/Grafana)
> - **Retention limits**: comply with GDPR — define log retention based on data sensitivity
> - **Audit logs**: separate, protected audit trail for security-sensitive actions

**Q8: How would you design a logging system handling 1TB of logs per day?**

> - **Ingestion**: Kafka as buffer (handles bursty log ingestion, provides durability)
> - **Processing**: Logstash/Flink consumers reading from Kafka, parsing and filtering
> - **Hot storage**: Elasticsearch cluster sized for 30-day retention (~30TB); use ILM for warm/cold tiers to S3/GCS
> - **Cold storage**: S3/GCS with Parquet format for long-term compliance storage
> - **Indexing strategy**: index per service per day, enable forcemerge on old indices
> - **Search**: Elasticsearch for interactive queries; Athena/BigQuery for historical analytics
> - **Cost control**: filter noise (health checks, debug) before ingestion; compress old indices

---

## Scenario-Based Questions

### 🔵 Scenario 1: Service Returning 500 Errors

*"Your monitoring shows a spike in 500 errors for the payment service. How do you investigate using logs?"*

```
1. Narrow time range to the spike window in Kibana/Grafana

2. Query:
   {service="payment-service", level="ERROR"}
   | Time range: last 15 minutes

3. Look for patterns:
   - Same error message repeating?
   - Specific endpoint affected?
   - Specific user IDs?
   - Specific trace_ids (find the full request trace)?

4. Kibana example query:
   service: "payment-service" AND level: "ERROR" AND @timestamp: [now-15m TO now]

5. Aggregate to find most common error
6. Take a trace_id from an error → search it across all services (distributed trace)
7. Find root cause: was it this service or upstream dependency?
```

---

## Hands-On Labs

### Lab 1: Run ELK Stack Locally

```yaml
# docker-compose.yml for ELK stack
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    user: root
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - elasticsearch
```

```yaml
# filebeat.yml
filebeat.inputs:
- type: container
  paths:
    - '/var/lib/docker/containers/*/*.log'

processors:
- add_docker_metadata: ~
- decode_json_fields:
    fields: ["message"]
    target: ""

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "filebeat-%{+yyyy.MM.dd}"
```

---

> **Cross-links:** [← Monitoring](./Monitoring.md) | [Security →](./Security.md) | [Kubernetes (log collection on K8s) →](./Kubernetes.md) | [SRE (logging for SLOs) →](./SRE.md)