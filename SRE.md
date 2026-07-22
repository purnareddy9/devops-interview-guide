---
layout: default
title: "🎯 SRE — Site Reliability Engineering Interview Guide"
render_with_liquid: false
---

# 🎯 SRE — Site Reliability Engineering Interview Guide

> [← Security](./Security.md) | [Main Index](./README.md) | [System Design →](./SystemDesign.md)

---

## Table of Contents

1. [SRE Fundamentals](#sre-fundamentals)
2. [SLIs, SLOs, SLAs & Error Budgets](#slis-slos-slas--error-budgets)
3. [Toil & Automation](#toil--automation)
4. [Incident Management](#incident-management)
5. [Postmortems](#postmortems)
6. [Capacity Planning](#capacity-planning)
7. [Release Engineering](#release-engineering)
8. [Interview Questions](#interview-questions)
9. [Scenario-Based Questions](#scenario-based-questions)
10. [Hands-On Labs](#hands-on-labs)

---

## SRE Fundamentals

### SRE vs DevOps vs Ops

```
Traditional Ops: Manual operations, change-averse, silo from dev
DevOps:          Cultural philosophy, collaboration, automation mindset
SRE:             Specific implementation of DevOps with engineering rigor
                 "SRE is what you get when you treat operations as a software problem"
                 — Ben Treynor Sloss, Google

SRE Key Tenets:
1. Hire software engineers to do operations work
2. Define reliability targets (SLOs) with error budgets
3. Eliminate toil through automation
4. Share on-call between SRE and dev teams
5. Blameless postmortems
6. Gradual rollout with automatic rollback
```

### SRE Responsibilities

| Area | Activities |
|------|-----------|
| **Reliability** | SLO definition, error budget management, reliability reviews |
| **Observability** | Monitoring, alerting, dashboards, distributed tracing |
| **Incident Response** | On-call rotation, incident management, escalation |
| **Postmortems** | Blameless root cause analysis, action items |
| **Capacity Planning** | Growth projections, resource planning, load testing |
| **Release Engineering** | Deployment safety, rollback mechanisms, feature flags |
| **Toil Reduction** | Automating repetitive operational work |

---

## SLIs, SLOs, SLAs & Error Budgets

### Definitions

```
Service Level Indicator (SLI)
  A quantitative measure of service quality, expressed as a ratio:
  
  SLI = (good events) / (total events)

  Examples:
  - Availability:   requests returning 2xx / total requests
  - Latency:        requests completing < 500ms / total requests
  - Freshness:      updates served < 10min old / total served

Service Level Objective (SLO)
  Target value or range for an SLI over a rolling time window:
  
  Availability SLO: 99.9% over 30 days
  Latency SLO:      95% of requests < 200ms over 7 days

Error Budget
  Allowance for unreliability: Error Budget = 1 - SLO
  
  99.9% SLO → 0.1% error budget → 43.8 min/month downtime allowed
  99.5% SLO → 0.5% error budget → 3.65 hours/month
  99.0% SLO → 1.0% error budget → 7.3 hours/month

Service Level Agreement (SLA)
  A business contract with financial consequences for missing SLOs.
  SLA ≤ SLO (your internal target is tighter than what you promise customers)
```

### Error Budget Policy

```
Error Budget = 1 - SLO Target = Allowed unreliability

Budget Status:        Action:
> 50% remaining  →   Normal operations; feature velocity prioritized
25-50% remaining →   Caution; slow down risky changes
< 25% remaining  →   Freeze non-critical deployments; focus on reliability
< 0% (exhausted) →   Halt all feature work; incident review; reliability sprint

Tracking (PromQL):
# Error budget remaining (%)
(1 - (
  sum(rate(http_requests_total{status=~"5.."}[30d]))
  / sum(rate(http_requests_total[30d]))
)) / (1 - 0.999)   # 0.999 = SLO

# Error budget burn rate
sum(rate(http_requests_total{status=~"5.."}[1h])) /
sum(rate(http_requests_total[1h]))
/ (1 - 0.999)
```

### Choosing the Right SLO

```
Too tight (99.999%): 5.26 min/year allowed downtime
  → Extremely costly, may not be achievable
  → Any deployment or maintenance violates SLO

Too loose (90%): 36.5 hours/month
  → Users will notice
  → No incentive to improve

Good process:
1. Measure current performance (your baseline)
2. Set SLO slightly below current performance
3. Review customer satisfaction data
4. Tighten over time as reliability improves
5. Different SLOs for different tiers (critical vs. internal tooling)
```

---

## Toil & Automation

### What is Toil?

```
Toil = Manual, repetitive, automatable operational work

Characteristics:
- Manual: requires human intervention
- Repetitive: done over and over
- Automatable: a computer could do it
- Tactical: reactive work, not proactive
- No lasting value: doesn't improve the system

Examples:
- Manually restarting crashed services
- Running deployment scripts manually
- Manually updating configuration files
- Responding to "is the system up?" pings
- Manual capacity adjustments

SRE principle: Keep toil < 50% of work time
Surplus time → engineering work to eliminate toil
```

### Toil Elimination Examples

```python
# Toil: Manually restarting crashed services
# Elimination: Auto-restart with health checks
# K8s: livenessProbe + restartPolicy: Always

# Toil: Manually scaling during traffic spikes
# Elimination: HPA / Cluster Autoscaler
# K8s: HorizontalPodAutoscaler based on CPU/custom metrics

# Toil: Manually rotating credentials
# Elimination: Dynamic secrets with Vault / automatic rotation
vault write aws/roles/myapp policy=...
# Credentials auto-expire and rotate

# Toil: Manually checking for compliance drift
# Elimination: Policy as Code + automated scanning
# OPA Gatekeeper + CI scanning

# Toil: Manual database backups
# Elimination: Automated scheduled backups + restore testing
# RDS automated backups + Lambda to test restores weekly
```

---

## Incident Management

### Incident Lifecycle

```
Detection → Triage → Mitigation → Resolution → Post-incident

Detection:
  - Alert fires (monitoring)
  - User report
  - Business alert (revenue drop, error rate spike)

Triage (first 5 minutes):
  - Is this a real incident?
  - What is the severity/impact?
  - Who is the Incident Commander (IC)?
  - Notify stakeholders

Mitigation (restore service):
  - NOT root cause analysis — that comes later
  - Rollback if recent deployment
  - Scale up resources
  - Enable circuit breaker
  - Enable feature flag to disable broken feature
  - Failover to secondary

Resolution:
  - Service is stable
  - Monitoring confirms recovery
  - Stakeholder communication

Post-incident:
  - Postmortem (blameless)
  - Action items
  - Incident report to customers (if SLA breach)
```

### Severity Levels

| Level | Impact | Response Time | Example |
|-------|--------|--------------|---------|
| **SEV-1** (Critical) | Total outage or data loss | Immediate (< 5 min) | Site down, data corruption |
| **SEV-2** (High) | Major feature degraded | 15 minutes | Checkout broken, 50% error rate |
| **SEV-3** (Medium) | Minor impact, workaround exists | 1 hour | Slow performance, non-critical feature down |
| **SEV-4** (Low) | Minimal impact | Next business day | Dashboard not loading, cosmetic issue |

### Incident Response Commands

```bash
# Quick system health check during incident
#!/bin/bash
echo "=== Kubernetes Overview ==="
kubectl get nodes
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

echo "=== Recent Events ==="
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -20

echo "=== Service Endpoints ==="
kubectl get endpoints -n production

echo "=== Recent Deployments ==="
kubectl rollout history deployment -n production

echo "=== Resource Usage ==="
kubectl top nodes
kubectl top pods -n production --sort-by=cpu | head -10

echo "=== Recent Errors ==="
kubectl logs -n production -l app=myapp --tail=50 | grep ERROR
```

### Incident Communication Template

```
[Incident] SEV-2: Checkout Service Degraded

Status: INVESTIGATING | IDENTIFIED | MONITORING | RESOLVED
Time: 14:32 UTC

Impact:
  ~30% of checkout attempts are failing
  Estimated 2,000 users affected

Current State:
  Checkout service returning 503 errors
  Error rate: 28% (baseline: 0.1%)
  Latency p99: 8,500ms (baseline: 250ms)

Timeline:
  14:25 - Alert fired (error rate > 5%)
  14:28 - IC assigned: @alice
  14:32 - This update

Actions:
  - Investigating recent deployment (deployed 14:15)
  - DB connection pool metrics being reviewed
  - Rollback prepared, awaiting confirmation

Next Update: 14:45 UTC or sooner if status changes
IC: @alice | Comms: @bob | Tech Lead: @charlie
```

---

## Postmortems

### Blameless Postmortem Principles

```
"The goal is not to find who made the mistake.
The goal is to understand what happened and prevent recurrence."

Key principles:
1. Blameless — no personal blame, psychological safety
2. Assume good intent — everyone did their best with available info
3. Learn — extract learnings, not assign blame
4. Fix systems — humans make errors; fix processes/tools
5. Share widely — learnings benefit everyone
```

### Postmortem Template

```markdown
# Postmortem: [Service] [Brief Description]
**Date**: 2024-01-15
**Duration**: 45 minutes (14:25 - 15:10 UTC)
**Severity**: SEV-2
**Authors**: Alice, Bob
**Status**: COMPLETE — all action items tracked

## Impact
- 28% of checkout requests failed
- ~3,400 users affected
- Estimated revenue impact: $45,000
- 2 SLA breach notifications sent

## Root Cause
Database connection pool exhausted due to a connection leak
introduced in deployment v1.2.5 (14:15 UTC). The leak occurred
in error paths that didn't close connections properly.

## Timeline
| Time (UTC) | Event |
|------------|-------|
| 14:15 | v1.2.5 deployed to production |
| 14:25 | Error rate alert fired (threshold: 5%) |
| 14:28 | Alice paged, acknowledged |
| 14:32 | Incident bridge opened, SEV-2 declared |
| 14:45 | Root cause identified (DB pool exhaustion) |
| 14:58 | Rollback to v1.2.4 initiated |
| 15:05 | Error rate normalizing |
| 15:10 | Incident resolved, error rate < 0.1% |

## What Went Well
- Alert fired within 1 minute of user impact
- Incident bridge started quickly
- Rollback was prepared and executed in < 5 minutes
- Runbook was accurate and helpful

## What Went Wrong
- Connection leak not detected in staging (no load testing)
- DB connection pool metrics not alerted on
- Deployment was not gradually rolled out (all-at-once)

## Action Items
| Action | Owner | Due Date |
|--------|-------|---------|
| Add connection pool exhaustion alert | Bob | Jan 20 |
| Add load/soak testing to staging pipeline | Alice | Jan 25 |
| Enable gradual rollout for all production deploys | Charlie | Jan 18 |
| Add connection leak detection to code review checklist | Team | Jan 17 |
| Review all error paths for connection handling | Alice | Jan 22 |
```

---

## Capacity Planning

### Capacity Planning Process

```
1. Measure current usage
   - CPU, memory, disk, network baseline
   - Request rate, latency percentiles
   - Cost per unit (cost per 1M requests)

2. Forecast growth
   - Historical growth rate
   - Business forecasts (seasonal, launch events)
   - Model: linear, exponential, step-function

3. Calculate headroom needed
   - Peak-to-average ratio
   - N+1 redundancy
   - Emergency capacity reserve (20-30%)

4. Plan infrastructure changes
   - Timeline: when will you hit limits?
   - Lead time: how long to provision?
   - Cost projection

5. Review regularly
   - Monthly capacity review
   - Pre-launch capacity review for big events
```

### Load Testing

```bash
# k6 — modern load testing
cat > load-test.js << 'EOF'
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '2m', target: 100 },    // Ramp up to 100 VUs
    { duration: '5m', target: 100 },    // Hold at 100 VUs
    { duration: '2m', target: 500 },    // Spike to 500 VUs
    { duration: '5m', target: 500 },    // Hold spike
    { duration: '2m', target: 0 },      // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(99)<500'],   // p99 < 500ms
    errors: ['rate<0.01'],              // Error rate < 1%
  },
};

export default function () {
  const response = http.get('https://myapp.example.com/api/products');

  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });

  errorRate.add(response.status !== 200);
  sleep(1);
}
EOF

k6 run load-test.js
```

---

## Interview Questions

### 🟢 Beginner

**Q1: What is the difference between SRE and traditional operations?**

> Traditional ops teams tend to be change-averse (changes cause outages), use manual procedures, and operate in silos from development teams. SRE takes a software engineering approach to operations: reliability is a feature, changes are managed with automation and progressive delivery, toil is eliminated through code, and the team has explicit error budgets that allow controlled change. SRE establishes a structured relationship between reliability and feature velocity.

**Q2: What is an error budget?**

> An error budget is the allowable amount of unreliability in a given time window. It equals `1 - SLO`. For a 99.9% availability SLO, the error budget is 0.1% = 43.8 minutes per month. When the budget is healthy, teams can deploy frequently. When the budget is depleted, all feature work stops and reliability engineering takes priority. This creates a healthy tension: devs want to ship features, but not at the cost of reliability (their budget).

**Q3: What is toil in SRE?**

> Toil is manual, repetitive, automatable operational work that scales linearly with service growth. Characteristics: manual, repetitive, automatable, reactive (tactical), no lasting value. Examples: manually restarting pods, running deployment scripts, manually adding capacity. SRE philosophy: keep toil under 50% of work time; use the remainder for engineering work to eliminate that toil through automation.

---

### 🟡 Intermediate

**Q4: How do you design a good alerting strategy?**

> - **Alert on symptoms, not causes**: alert when users are impacted (high error rate, slow responses), not on individual CPU spikes
> - **Alert on SLO burn rate**: alert when error budget is being consumed faster than expected
> - **Avoid alert fatigue**: every alert should be actionable; silence or eliminate noisy non-actionable alerts
> - **Severity levels**: page for SEV-1/2 (wake someone up); Slack/email for lower severity
> - **Runbooks**: every alert links to a runbook with investigation steps
> - **Multi-window**: use fast + slow windows to reduce false positives (multi-burn-rate alerting)

**Q5: How do you do a blameless postmortem?**

> Focus the postmortem on the **system** and **processes**, not individuals. Use "what happened" framing, not "who failed". Acknowledge that complex systems have latent failures; the triggering event is rarely the root cause. Use the "5 Whys" technique to find systemic causes. Action items should fix systems (add alerting, improve rollback, add tests) not blame people. Share postmortems widely — learnings benefit the whole organization.

**Q6: What is the relationship between SLO and feature velocity?**

> Error budget creates a quantified trade-off between reliability and feature velocity. When the error budget is full (all features are working reliably), teams can move fast and take risks with deployments. When the error budget is depleted (too many incidents), feature deployments are halted until reliability is restored. This gives SRE teams a data-driven way to say "slow down" — instead of just "we feel risky" — and gives developers a clear signal about when they can ship freely.

---

### 🔴 Advanced

**Q7: How would you design an on-call rotation for a global service?**

> - **Follow-the-sun**: rotate across time zones (Americas, EMEA, APAC) so engineers are paged during business hours
> - **Primary + secondary**: one person is paged first; secondary is escalation path
> - **Rotation length**: 1 week is common; avoids burnout from longer rotations
> - **Escalation policy**: if primary doesn't acknowledge in 5 min, page secondary; then management
> - **On-call handoff**: structured handoff meeting covering current incidents, recent changes, known issues
> - **Compensation**: on-call pay or time-off-in-lieu policy
> - **Training**: runbooks, shadowing before solo on-call
> - **Measure**: track on-call burden (number of pages, time spent) — excessive pages indicate toil that should be eliminated

**Q8: How do you implement SLOs as code?**

```yaml
# Sloth — SLO as Code (generates Prometheus rules)
version: "prometheus/v1"
service: myapp
labels:
  owner: platform-team
  tier: critical
slos:
- name: requests-availability
  objective: 99.9
  description: "99.9% of requests succeed"
  sli:
    events:
      error_query: sum(rate(http_requests_total{job="myapp",code=~"5.."}[{{.window}}]))
      total_query: sum(rate(http_requests_total{job="myapp"}[{{.window}}]))
  alerting:
    name: RequestsAvailability
    labels:
      severity: critical
    annotations:
      runbook: https://runbooks.example.com/requests-availability
    page_alert:
      labels:
        severity: critical
    ticket_alert:
      labels:
        severity: warning
```

---

## Scenario-Based Questions

### 🔵 Scenario 1: Error Budget Exhausted

*"Your service exhausted its error budget 3 weeks into the month. The dev team wants to deploy a new feature. What do you do?"*

```
1. HALT feature deployments immediately (per error budget policy)

2. Immediate analysis:
   - Which incidents consumed the budget?
   - Was it one big incident or many small ones?
   - Is the cause fixed or still ongoing?

3. Reliability sprint:
   - Identify top 3 reliability improvements
   - Fix alerting gaps that led to delayed detection
   - Add missing error handling / circuit breakers
   - Improve rollback speed

4. Exception process:
   - If business critical feature needed:
     → Explicit sign-off from business stakeholder
     → Enhanced monitoring during deployment
     → Automatic rollback threshold defined
     → Reduced blast radius (canary/feature flag)

5. Review SLO:
   - Was the SLO realistic?
   - Should it be loosened short-term?
   - Is the service getting enough investment?

6. Communication:
   - Status to stakeholders (budget consumed, what it means)
   - Timeline for return to normal
```

---

## Hands-On Labs

### Lab 1: Calculate Error Budget

```python
#!/usr/bin/env python3
"""Calculate SLO error budget from Prometheus data."""

import requests
from datetime import datetime, timedelta

PROMETHEUS_URL = "http://localhost:9090"
SLO = 0.999  # 99.9%

def query_prometheus(query, start, end, step="60s"):
    resp = requests.get(
        f"{PROMETHEUS_URL}/api/v1/query_range",
        params={"query": query, "start": start.isoformat(), "end": end.isoformat(), "step": step}
    )
    return resp.json()["data"]["result"]

def calculate_error_budget():
    end = datetime.utcnow()
    start = end - timedelta(days=30)

    total = query_prometheus("sum(increase(http_requests_total[30d]))", start, end)
    errors = query_prometheus("sum(increase(http_requests_total{status=~'5..'}[30d]))", start, end)

    total_count = float(total[0]["values"][-1][1]) if total else 0
    error_count = float(errors[0]["values"][-1][1]) if errors else 0

    if total_count == 0:
        print("No data available")
        return

    availability = 1 - (error_count / total_count)
    error_budget = 1 - SLO
    budget_used = (1 - availability) / error_budget * 100 if error_budget > 0 else 0

    print(f"30-day availability:   {availability:.4%}")
    print(f"SLO target:            {SLO:.4%}")
    print(f"Error budget:          {error_budget:.4%}")
    print(f"Budget consumed:       {budget_used:.1f}%")
    print(f"Budget remaining:      {100 - budget_used:.1f}%")
    print(f"Total requests:        {total_count:,.0f}")
    print(f"Error requests:        {error_count:,.0f}")

if __name__ == "__main__":
    calculate_error_budget()
```

---

> **Cross-links:** [← Security](./Security.md) | [System Design →](./SystemDesign.md) | [Monitoring (SLO alerts) →](./Monitoring.md) | [Kubernetes (reliability) →](./Kubernetes.md)