---
layout: default
title: "🤖 Module 9 — AI for DevOps"
render_with_liquid: false
---

# 🤖 Module 9 — AI for DevOps

> **JD Alignment:** "Leverage AI-assisted tooling, intelligent automation, and ML-powered observability to increase pipeline velocity, reduce toil, and improve system reliability."

> [← Python Scripting](./08-Python-Scripting.md) | [README](./README.md)

---

## Table of Contents

1. [AI & DevOps Fundamentals](#ai--devops-fundamentals)
2. [AI-Assisted Coding & Code Review](#ai-assisted-coding--code-review)
3. [Intelligent CI/CD Pipelines](#intelligent-cicd-pipelines)
4. [AIOps — Anomaly Detection & Alerting](#aiops--anomaly-detection--alerting)
5. [LLM-Powered Automation & Chatbots](#llm-powered-automation--chatbots)
6. [AI in Security & Compliance](#ai-in-security--compliance)
7. [MLOps on Kubernetes & OpenShift](#mlops-on-kubernetes--openshift)
8. [AI-Driven Incident Response](#ai-driven-incident-response)
9. [Responsible AI in DevOps](#responsible-ai-in-devops)

---

## AI & DevOps Fundamentals

---

### 🟢 Q1. What does "AI for DevOps" mean and how does it differ from traditional DevOps?

**Answer:**

AI for DevOps (also called **AIOps**) is the application of machine learning, large language models (LLMs), and intelligent automation to traditionally manual or rule-based DevOps practices. The goal is to reduce **toil**, accelerate **feedback loops**, and move from reactive to **predictive** operations.

**Key shift in mindset:**

| Dimension | Traditional DevOps | AI-Augmented DevOps |
|-----------|-------------------|---------------------|
| Alerting | Threshold rules (`CPU > 80%`) | Anomaly detection on metric patterns |
| Root cause analysis | Manual log grep + intuition | LLM-summarised correlated logs + traces |
| Code review | Human reviewer checks PRs | AI suggests security/quality fixes inline |
| Pipeline optimisation | Manual parallelisation decisions | ML predicts flaky tests → skip or quarantine |
| Incident response | On-call engineer reads runbooks | AI copilot surfaces relevant runbook + past incidents |
| Capacity planning | Manual sizing + spreadsheets | Predictive autoscaling based on traffic forecasts |
| IaC generation | Engineer writes Terraform/Helm | LLM drafts, human reviews |

**Tooling landscape:**

| Category | Tools |
|----------|-------|
| AI coding assistants | GitHub Copilot, Cursor, Amazon CodeWhisperer, Tabnine |
| AIOps platforms | Dynatrace Davis AI, Datadog Watchdog, IBM Watson AIOps, PagerDuty AIOps |
| LLM APIs | OpenAI GPT-4o, Anthropic Claude, Google Gemini, open-source Llama 3 |
| ML on Kubernetes | Kubeflow, OpenShift AI (RHOAI), MLflow, BentoML |
| Observability AI | Grafana ML, Elastic ML, Prometheus + anomaly rules |
| Security AI | Snyk AI, GitHub Advanced Security Copilot, Aqua DTA |

---

### 🟢 Q2. What are the core use cases where AI adds the most value in a DevOps pipeline?

**Answer:**

**High-ROI AI use cases in DevOps (ranked by impact):**

1. **Intelligent alerting & noise reduction**
   - ML clusters correlated alerts into a single incident, suppressing duplicates.
   - Typical outcome: 60-80% alert noise reduction.

2. **Predictive test selection**
   - ML model predicts which tests are likely to fail based on changed files.
   - Only run high-risk tests on PRs → pipeline 3-5× faster.

3. **AI-assisted code review**
   - LLMs detect security misconfigs, logic bugs, and missing error handling before merging.

4. **Automated root cause analysis (RCA)**
   - LLMs correlate logs, metrics, and traces to produce a human-readable incident summary in seconds.

5. **Self-healing infrastructure**
   - Rules engine + ML detects failing pod patterns → triggers automated remediation runbooks.

6. **Capacity forecasting**
   - Time-series ML (e.g. Facebook Prophet, AWS Forecast) predicts resource demand days ahead.

7. **Natural language IaC generation**
   - Prompt: *"Create a Helm chart for a Node.js app with HPA, PodDisruptionBudget, and NetworkPolicy"*
   - LLM drafts the YAML; human reviews and commits.

8. **Chatops integration**
   - Slack/Teams bot backed by an LLM answers *"What's the P50 latency of checkout-service this week?"* in natural language.

---

## AI-Assisted Coding & Code Review

---

### 🟡 Q3. How do you integrate GitHub Copilot or an LLM API into a CI pipeline for automated code review?

**Answer:**

**Approach: LLM-powered PR review bot as a GitHub Actions job**

```yaml
# .github/workflows/ai-review.yml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  ai-review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get PR diff
        id: diff
        run: |
          git diff origin/${{ github.base_ref }}...HEAD > pr.diff
          echo "diff_size=$(wc -c < pr.diff)" >> $GITHUB_OUTPUT

      - name: Run AI Review
        if: steps.diff.outputs.diff_size != '0'
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          PR_NUMBER: ${{ github.event.pull_request.number }}
          REPO: ${{ github.repository }}
        run: |
          pip install openai requests --quiet
          python .github/scripts/ai_review.py
```

**`.github/scripts/ai_review.py`:**
```python
import os
import openai
import requests

client = openai.OpenAI(api_key=os.environ["OPENAI_API_KEY"])

with open("pr.diff") as f:
    diff = f.read()[:12000]          # trim to stay within context window

SYSTEM_PROMPT = """You are a senior DevOps engineer reviewing a pull request diff.
Analyse the changes and report:
1. Security issues (hardcoded secrets, insecure configs, missing RBAC)
2. Reliability risks (missing error handling, no rollback, flaky patterns)
3. Performance concerns (missing resource limits, unbounded loops)
4. Best practice violations (dockerfile best practices, Kubernetes anti-patterns)
Be concise. Use markdown. If no issues, say 'LGTM ✅'.
"""

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user",   "content": f"Review this diff:\n\n```diff\n{diff}\n```"},
    ],
    temperature=0.2,
    max_tokens=1500,
)

review_body = response.choices[0].message.content

# Post as PR comment via GitHub API
headers = {
    "Authorization": f"Bearer {os.environ['GH_TOKEN']}",
    "Accept": "application/vnd.github+json",
}
repo = os.environ["REPO"]
pr   = os.environ["PR_NUMBER"]
requests.post(
    f"https://api.github.com/repos/{repo}/issues/{pr}/comments",
    json={"body": f"### 🤖 AI Code Review\n\n{review_body}"},
    headers=headers,
).raise_for_status()

print("AI review posted.")
```

**Guardrails to apply in production:**
- Gate the AI review as informational only — never block a merge on AI feedback alone.
- Strip secrets from the diff before sending to external LLM APIs.
- Use a self-hosted model (Llama 3, CodeLlama) if your org prohibits sending code to third parties.
- Log all API calls for audit.

---

### 🟡 Q4. How do you use an LLM to auto-generate Kubernetes manifests or Helm values from a natural language description?

**Answer:**

```python
#!/usr/bin/env python3
"""
gen_manifest.py — Generate a Kubernetes manifest from natural language.
Usage: python gen_manifest.py "Deploy a Redis cache with 2 replicas, 256Mi memory limit,
                               and a ClusterIP service on port 6379"
"""
import sys
import openai
import yaml

client = openai.OpenAI()   # reads OPENAI_API_KEY from env

SYSTEM = """You are a Kubernetes YAML expert. When given a description,
output ONLY valid Kubernetes YAML (one or more documents separated by ---).
Include: Deployment, Service, and HorizontalPodAutoscaler where relevant.
Always set resource requests and limits. Never add explanatory text outside the YAML."""

def generate_manifest(description: str) -> str:
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": SYSTEM},
            {"role": "user",   "content": description},
        ],
        temperature=0,
    )
    raw = response.choices[0].message.content.strip()
    # Strip markdown fences if present
    if raw.startswith("```"):
        raw = "\n".join(raw.split("\n")[1:])
        raw = raw.rsplit("```", 1)[0].strip()
    return raw

def validate_yaml(raw: str) -> bool:
    """Basic structural validation before writing to disk."""
    try:
        docs = list(yaml.safe_load_all(raw))
        for doc in docs:
            assert doc.get("apiVersion"), "Missing apiVersion"
            assert doc.get("kind"),       "Missing kind"
        return True
    except Exception as exc:
        print(f"[WARN] YAML validation failed: {exc}")
        return False

if __name__ == "__main__":
    desc = " ".join(sys.argv[1:]) or "A simple nginx deployment with 2 replicas"
    manifest = generate_manifest(desc)
    if validate_yaml(manifest):
        print(manifest)
    else:
        print("[ERROR] Generated YAML is invalid. Manual review required.")
        sys.exit(1)
```

**Best practice workflow:**
```
1. LLM generates YAML  →  2. Validate with kubeconform/kubeval  →  3. Human reviews diff  →  4. Merge via PR
```

Never apply AI-generated manifests directly with `kubectl apply` without a human review gate.

---

## Intelligent CI/CD Pipelines

---

### 🟡 Q5. What is predictive test selection and how do you implement it in a CI pipeline?

**Answer:**

**Predictive test selection** uses an ML model trained on historical test run data to predict which tests are likely to fail for a given code change — so CI only runs the high-risk subset, cutting pipeline time dramatically.

**How it works:**
```
Git diff (changed files)  →  Feature extraction  →  ML model  →  Risk scores per test  →  Run top-N
```

**Implementation using open-source tooling (Launchable or custom):**

```python
# predict_tests.py — Simple file-to-test affinity model
import json
import subprocess
import sys
from pathlib import Path
from collections import defaultdict


def get_changed_files() -> list[str]:
    result = subprocess.run(
        ["git", "diff", "--name-only", "origin/main...HEAD"],
        capture_output=True, text=True, check=True,
    )
    return result.stdout.strip().splitlines()


def load_affinity_map(path: str = ".ci/test-affinity.json") -> dict:
    """
    Affinity map: {source_file: [test_file, ...]}
    Built offline by analysing historical test failure / file-change correlations.
    """
    return json.loads(Path(path).read_text())


def select_tests(changed: list[str], affinity: dict) -> list[str]:
    selected = set()
    for f in changed:
        for test in affinity.get(f, []):
            selected.add(test)
    return sorted(selected)


if __name__ == "__main__":
    changed = get_changed_files()
    print(f"Changed files: {changed}")

    affinity = load_affinity_map()
    tests = select_tests(changed, affinity)

    if not tests:
        print("No targeted tests found — running full suite.")
        sys.exit(0)

    print(f"Running {len(tests)} targeted test(s):")
    for t in tests:
        print(f"  {t}")

    # Write to file for downstream CI step
    Path(".ci/selected-tests.txt").write_text("\n".join(tests))
```

**GitHub Actions integration:**
```yaml
- name: Predict test selection
  run: python .ci/predict_tests.py

- name: Run targeted tests
  run: |
    if [ -f .ci/selected-tests.txt ]; then
      pytest $(cat .ci/selected-tests.txt) -v
    else
      pytest tests/ -v
    fi
```

**Commercial alternatives:** Launchable, BuildPulse, Trunk Flaky Tests — provide pre-built ML models trained on your repo's history.

---

### 🟡 Q6. How do you use AI to detect and quarantine flaky tests automatically?

**Answer:**

A **flaky test detector** tracks per-test pass/fail history and flags tests whose failure rate is above a threshold without correlated code changes.

```python
# flaky_detector.py
import json
from pathlib import Path
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class TestRecord:
    name: str
    runs: list[bool] = field(default_factory=list)  # True=pass, False=fail

    @property
    def flakiness_score(self) -> float:
        if len(self.runs) < 5:
            return 0.0
        failures = self.runs.count(False)
        passes   = self.runs.count(True)
        # Flaky = fails AND passes (not consistently broken)
        if failures == 0 or passes == 0:
            return 0.0
        return failures / len(self.runs)

    @property
    def is_flaky(self) -> bool:
        return 0.05 < self.flakiness_score < 0.95


def load_history(path: str) -> dict[str, TestRecord]:
    raw = json.loads(Path(path).read_text()) if Path(path).exists() else {}
    return {name: TestRecord(name=name, runs=runs) for name, runs in raw.items()}


def report_flaky(records: dict[str, TestRecord], threshold: float = 0.1) -> list[str]:
    flaky = [
        r.name for r in records.values()
        if r.flakiness_score >= threshold
    ]
    return sorted(flaky, key=lambda n: records[n].flakiness_score, reverse=True)


if __name__ == "__main__":
    records = load_history(".ci/test-history.json")
    flaky = report_flaky(records)
    if flaky:
        print("⚠️  Flaky tests detected:")
        for name in flaky:
            score = records[name].flakiness_score
            print(f"  {name:60s}  flakiness={score:.0%}")
        # Write quarantine list
        Path(".ci/quarantine.txt").write_text("\n".join(flaky))
    else:
        print("No flaky tests detected.")
```

**CI usage:**
```yaml
- name: Run tests (excluding quarantine)
  run: |
    QUARANTINE=""
    if [ -f .ci/quarantine.txt ]; then
      QUARANTINE=$(awk '{printf "--deselect %s ", $0}' .ci/quarantine.txt)
    fi
    pytest tests/ $QUARANTINE -v
```

---

## AIOps — Anomaly Detection & Alerting

---

### 🟡 Q7. How does AI-based anomaly detection differ from threshold-based alerting in Prometheus?

**Answer:**

| Aspect | Threshold alerting | AI/ML anomaly detection |
|--------|-------------------|------------------------|
| Rule definition | Manual: `CPU > 80%` | Automatic: learns normal baseline per metric |
| Seasonal awareness | None | Handles daily/weekly patterns (e.g. lower traffic at night) |
| False positive rate | High (weekend traffic looks like incident) | Lower — alert only when pattern deviates from learned norm |
| New services | Must write rules for every metric | Model auto-adapts as data arrives |
| Cold start | Immediate | Needs 1-2 weeks of data to calibrate |
| Tooling | `PrometheusRule` CRDs | Grafana ML, Datadog Watchdog, Elastic ML, custom Prophet |

**Example: Using Facebook Prophet for latency anomaly detection:**

```python
# anomaly_detect.py — Offline training + scoring on Prometheus data
import pandas as pd
from prophet import Prophet
import requests
from datetime import datetime, timedelta


PROMETHEUS_URL = "http://prometheus:9090"


def fetch_metric(query: str, hours: int = 168) -> pd.DataFrame:
    """Fetch time-series data from Prometheus for the last N hours."""
    end   = datetime.utcnow()
    start = end - timedelta(hours=hours)
    resp = requests.get(
        f"{PROMETHEUS_URL}/api/v1/query_range",
        params={
            "query": query,
            "start": start.isoformat() + "Z",
            "end":   end.isoformat() + "Z",
            "step":  "5m",
        },
        timeout=15,
    )
    resp.raise_for_status()
    result = resp.json()["data"]["result"]
    if not result:
        return pd.DataFrame(columns=["ds", "y"])
    values = result[0]["values"]
    df = pd.DataFrame(values, columns=["ts", "y"])
    df["ds"] = pd.to_datetime(df["ts"].astype(float), unit="s", utc=True).dt.tz_localize(None)
    df["y"]  = df["y"].astype(float)
    return df[["ds", "y"]]


def detect_anomalies(df: pd.DataFrame, interval_width: float = 0.99) -> pd.DataFrame:
    """Fit Prophet and flag points outside the prediction interval."""
    model = Prophet(interval_width=interval_width, daily_seasonality=True)
    model.fit(df)
    forecast = model.predict(df)
    merged = df.merge(forecast[["ds", "yhat_lower", "yhat_upper"]], on="ds")
    merged["anomaly"] = (
        (merged["y"] < merged["yhat_lower"]) |
        (merged["y"] > merged["yhat_upper"])
    )
    return merged[merged["anomaly"]]


if __name__ == "__main__":
    df = fetch_metric(
        'histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{service="checkout"}[5m]))'
    )
    anomalies = detect_anomalies(df)
    print(f"Detected {len(anomalies)} anomalous data points:")
    print(anomalies[["ds", "y"]].to_string(index=False))
```

---

### 🔴 Q8. How do you implement an AI-driven alert correlation engine to reduce noise?

**Answer:**

Alert correlation groups related alerts into a single incident using clustering — preventing an on-call engineer from being paged 50 times for a single root cause.

```python
# alert_correlator.py
import hashlib
import json
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from typing import Optional
import requests
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.cluster import DBSCAN
import numpy as np


@dataclass
class Alert:
    name: str
    service: str
    namespace: str
    severity: str
    fired_at: datetime
    summary: str

    def fingerprint(self) -> str:
        raw = f"{self.name}:{self.service}:{self.namespace}"
        return hashlib.md5(raw.encode()).hexdigest()[:8]

    def to_text(self) -> str:
        return f"{self.name} {self.service} {self.namespace} {self.summary}"


def fetch_firing_alerts(prometheus_url: str) -> list[Alert]:
    resp = requests.get(f"{prometheus_url}/api/v1/alerts", timeout=10)
    resp.raise_for_status()
    alerts = []
    for a in resp.json()["data"]["alerts"]:
        if a["state"] != "firing":
            continue
        labels = a["labels"]
        alerts.append(Alert(
            name      = labels.get("alertname", "unknown"),
            service   = labels.get("service", ""),
            namespace = labels.get("namespace", ""),
            severity  = labels.get("severity", "info"),
            fired_at  = datetime.utcnow(),
            summary   = a["annotations"].get("summary", ""),
        ))
    return alerts


def correlate_alerts(alerts: list[Alert]) -> dict[int, list[Alert]]:
    """
    Cluster alerts using TF-IDF text similarity + DBSCAN.
    Returns {cluster_id: [alerts]}.  Cluster -1 = noise (uncorrelated).
    """
    if len(alerts) < 2:
        return {0: alerts}

    texts = [a.to_text() for a in alerts]
    vectors = TfidfVectorizer().fit_transform(texts).toarray()
    labels  = DBSCAN(eps=0.4, min_samples=1, metric="cosine").fit_predict(vectors)

    clusters: dict[int, list[Alert]] = {}
    for alert, label in zip(alerts, labels):
        clusters.setdefault(int(label), []).append(alert)
    return clusters


def summarise_cluster(alerts: list[Alert]) -> str:
    services   = {a.service   for a in alerts}
    namespaces = {a.namespace for a in alerts}
    names      = {a.name      for a in alerts}
    severities = {a.severity  for a in alerts}
    return (
        f"{len(alerts)} alert(s) | "
        f"services={','.join(sorted(services))} | "
        f"namespaces={','.join(sorted(namespaces))} | "
        f"types={','.join(sorted(names))} | "
        f"max_severity={'critical' if 'critical' in severities else 'high' if 'high' in severities else 'warning'}"
    )


if __name__ == "__main__":
    alerts  = fetch_firing_alerts("http://prometheus:9090")
    clusters = correlate_alerts(alerts)
    print(f"Reduced {len(alerts)} alerts → {len(clusters)} incident group(s):\n")
    for cid, group in sorted(clusters.items()):
        print(f"  Incident {cid}: {summarise_cluster(group)}")
```

---

## LLM-Powered Automation & Chatbots

---

### 🟡 Q9. How do you build a ChatOps bot that answers infrastructure questions using natural language?

**Answer:**

**Architecture:**
```
Slack message → Webhook → Python bot → LLM (with tool-calling) → Prometheus/kubectl → Reply
```

```python
# chatops_bot.py — Simplified LLM tool-calling bot
import os
import json
import subprocess
import requests
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
SLACK_WEBHOOK = os.environ["SLACK_WEBHOOK_URL"]

# --- Tool definitions (OpenAI function calling) ---
TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "query_prometheus",
            "description": "Run a PromQL query and return the result as a string.",
            "parameters": {
                "type": "object",
                "properties": {
                    "promql": {"type": "string", "description": "The PromQL expression to evaluate"}
                },
                "required": ["promql"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "kubectl_get",
            "description": "Run 'kubectl get' for a resource in a namespace.",
            "parameters": {
                "type": "object",
                "properties": {
                    "resource":  {"type": "string"},
                    "namespace": {"type": "string"},
                },
                "required": ["resource", "namespace"],
            },
        },
    },
]


def query_prometheus(promql: str) -> str:
    resp = requests.get(
        "http://prometheus:9090/api/v1/query",
        params={"query": promql}, timeout=10,
    )
    resp.raise_for_status()
    results = resp.json()["data"]["result"]
    if not results:
        return "No data found."
    return "\n".join(f"{r['metric']}: {r['value'][1]}" for r in results[:10])


def kubectl_get(resource: str, namespace: str) -> str:
    result = subprocess.run(
        ["kubectl", "get", resource, "-n", namespace, "--no-headers"],
        capture_output=True, text=True,
    )
    return result.stdout or result.stderr


TOOL_DISPATCH = {
    "query_prometheus": query_prometheus,
    "kubectl_get": kubectl_get,
}


def answer_question(user_question: str) -> str:
    messages = [
        {"role": "system", "content": (
            "You are a helpful DevOps assistant. "
            "Use the provided tools to answer infrastructure questions accurately. "
            "Be concise. Use bullet points where helpful."
        )},
        {"role": "user", "content": user_question},
    ]

    # Agentic loop: allow up to 3 tool call rounds
    for _ in range(3):
        response = client.chat.completions.create(
            model="gpt-4o", messages=messages, tools=TOOLS, tool_choice="auto",
        )
        msg = response.choices[0].message

        if msg.tool_calls:
            messages.append(msg)
            for tc in msg.tool_calls:
                fn_name = tc.function.name
                fn_args = json.loads(tc.function.arguments)
                result  = TOOL_DISPATCH[fn_name](**fn_args)
                messages.append({
                    "role": "tool",
                    "tool_call_id": tc.id,
                    "content": result,
                })
        else:
            return msg.content

    return "I couldn't fully answer that. Please check your monitoring dashboard."


def post_slack(text: str) -> None:
    requests.post(SLACK_WEBHOOK, json={"text": text}, timeout=5).raise_for_status()


if __name__ == "__main__":
    question = "What is the P95 latency of the checkout service right now?"
    answer = answer_question(question)
    print(answer)
    post_slack(f"*Bot answer:* {answer}")
```

---

## AI in Security & Compliance

---

### 🟡 Q10. How can AI help with secret detection and security scanning in a CI pipeline?

**Answer:**

**Three layers of AI-enhanced security scanning:**

**Layer 1 — Pre-commit: Detect secrets before they land in git**
```bash
# Install detect-secrets (ML-backed entropy analysis)
pip install detect-secrets
detect-secrets scan > .secrets.baseline
# In pre-commit hook:
detect-secrets audit .secrets.baseline
```

**Layer 2 — CI gate: SAST + AI-prioritised findings**
```yaml
# GitHub Actions — Semgrep AI-enhanced SAST
- name: Semgrep Scan
  uses: semgrep/semgrep-action@v1
  with:
    config: >-
      p/ci
      p/kubernetes
      p/dockerfile
      p/owasp-top-ten
  env:
    SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}
```

**Layer 3 — LLM-powered remediation suggestions**
```python
import openai, json

client = openai.OpenAI()

def suggest_fix(finding: dict) -> str:
    """Given a SAST finding dict, ask an LLM for a concrete fix."""
    prompt = f"""
A security scanner found this issue in our codebase:

Rule:    {finding['check_id']}
Message: {finding['extra']['message']}
File:    {finding['path']}:{finding['start']['line']}
Code:
```
{finding['extra'].get('lines', '')}
```

Provide a concise, actionable fix with a corrected code snippet.
"""
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "You are an application security expert. Be precise and brief."},
            {"role": "user",   "content": prompt},
        ],
        temperature=0.1,
        max_tokens=600,
    )
    return response.choices[0].message.content


# Usage: load Semgrep JSON output
with open("semgrep-results.json") as f:
    findings = json.load(f)["results"]

for finding in findings[:5]:   # top 5 findings
    print(f"\n--- {finding['check_id']} ---")
    print(suggest_fix(finding))
```

---

### 🔴 Q11. How do you implement an AI-driven compliance gate that checks Kubernetes manifests against policy?

**Answer:**

**Approach: OPA/Rego policy evaluation + LLM explanation of violations**

```python
#!/usr/bin/env python3
"""
compliance_gate.py — Evaluate K8s manifests against OPA policies,
then use an LLM to produce human-readable remediation advice.
"""
import subprocess
import json
import sys
import yaml
import openai
from pathlib import Path

client = openai.OpenAI()


def run_conftest(manifest_dir: str, policy_dir: str) -> dict:
    """Run conftest (OPA wrapper) and return parsed JSON output."""
    result = subprocess.run(
        ["conftest", "test", manifest_dir, "--policy", policy_dir, "--output", "json"],
        capture_output=True, text=True,
    )
    return json.loads(result.stdout or "[]")


def llm_remediation(violation: dict, manifest_snippet: str) -> str:
    """Ask LLM to explain violation and provide a concrete fix."""
    prompt = f"""
A Kubernetes policy compliance check flagged this violation:

Policy: {violation.get('rule', 'unknown')}
Message: {violation.get('msg', '')}

Relevant manifest snippet:
```yaml
{manifest_snippet[:1500]}
```

Explain why this is a problem and show the corrected YAML.
"""
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": "You are a Kubernetes security expert. Be brief and practical."},
            {"role": "user",   "content": prompt},
        ],
        temperature=0.1,
        max_tokens=700,
    )
    return response.choices[0].message.content


def main() -> None:
    manifest_dir = sys.argv[1] if len(sys.argv) > 1 else "manifests/"
    policy_dir   = sys.argv[2] if len(sys.argv) > 2 else "policies/"

    results = run_conftest(manifest_dir, policy_dir)
    all_failures = []

    for file_result in results:
        filename = file_result.get("filename", "")
        for failure in file_result.get("failures", []):
            all_failures.append((filename, failure))

    if not all_failures:
        print("✅ All compliance checks passed.")
        sys.exit(0)

    print(f"❌ {len(all_failures)} compliance violation(s):\n")
    for filename, violation in all_failures:
        snippet = Path(filename).read_text() if Path(filename).exists() else ""
        print(f"File: {filename}")
        print(f"Rule: {violation.get('rule','?')}")
        print(f"Msg:  {violation.get('msg','')}")
        print("\n🤖 AI Remediation Advice:")
        print(llm_remediation(violation, snippet))
        print("-" * 60)

    sys.exit(1)


if __name__ == "__main__":
    main()
```

---

## MLOps on Kubernetes & OpenShift

---

### 🟡 Q12. What is MLOps and how does OpenShift AI (RHOAI) support ML workloads?

**Answer:**

**MLOps** applies DevOps practices (versioning, CI/CD, monitoring) to machine learning model lifecycle management.

**ML lifecycle stages and Kubernetes tooling:**

| Stage | What happens | Kubernetes / RHOAI tooling |
|-------|-------------|---------------------------|
| **Data prep** | Feature engineering, dataset versioning | Kubeflow Pipelines, DVC |
| **Training** | Distributed training jobs | RHOAI Training Operator, PyTorchJob, TFJob |
| **Experiment tracking** | Log metrics, params, artefacts | MLflow (deployed as a service on OCP) |
| **Model registry** | Version and approve models | RHOAI Model Registry, MLflow Model Store |
| **Serving** | Expose model as REST/gRPC endpoint | RHOAI ModelMesh Serving, KServe, Seldon |
| **Monitoring** | Track drift, accuracy degradation | Evidently AI, WhyLabs, custom Prometheus metrics |
| **Re-training trigger** | Drift detected → retrain pipeline | Tekton / Argo Workflows triggered by drift alert |

**Minimal KServe InferenceService on OpenShift:**
```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: fraud-detector
  namespace: ml-production
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: s3://ml-models/fraud-detector/v3/
      resources:
        requests:
          cpu: 500m
          memory: 1Gi
        limits:
          cpu: "2"
          memory: 4Gi
  transformer:
    containers:
      - name: preprocessor
        image: ghcr.io/my-org/fraud-preprocessor:v1.2
```

**RHOAI Data Science Pipeline (Kubeflow Pipelines v2):**
```python
from kfp import dsl

@dsl.component(base_image="python:3.11")
def train_model(data_path: str, model_output: dsl.Output[dsl.Model]) -> float:
    import pickle, json
    from sklearn.ensemble import RandomForestClassifier
    from sklearn.datasets import load_iris
    X, y = load_iris(return_X_y=True)
    clf = RandomForestClassifier(n_estimators=100)
    clf.fit(X, y)
    with open(model_output.path, "wb") as f:
        pickle.dump(clf, f)
    return clf.score(X, y)

@dsl.pipeline(name="fraud-training-pipeline")
def fraud_pipeline(data_path: str = "s3://bucket/data/"):
    train_task = train_model(data_path=data_path)

```

---

### 🔴 Q13. How do you monitor an ML model in production for data drift and trigger automatic retraining?

**Answer:**

```python
# drift_monitor.py — Monitor a classification model for data drift using Evidently
import requests
import pandas as pd
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset
from evidently.metrics import DatasetDriftMetric


def fetch_production_data(api_url: str, hours: int = 24) -> pd.DataFrame:
    """Pull recent inference logs from the serving layer."""
    resp = requests.get(f"{api_url}/inference-logs", params={"hours": hours}, timeout=15)
    resp.raise_for_status()
    return pd.DataFrame(resp.json())


def check_drift(reference: pd.DataFrame, current: pd.DataFrame) -> tuple[bool, float]:
    """
    Returns (drift_detected: bool, share_of_drifted_columns: float).
    """
    report = Report(metrics=[DataDriftPreset()])
    report.run(reference_data=reference, current_data=current)
    result = report.as_dict()
    drift_metric = result["metrics"][0]["result"]
    drifted      = drift_metric["dataset_drift"]
    share        = drift_metric["share_of_drifted_columns"]
    return drifted, share


def trigger_retraining(tekton_url: str, pipeline_name: str, token: str) -> None:
    """Start a Tekton PipelineRun to retrain the model."""
    body = {
        "apiVersion": "tekton.dev/v1",
        "kind": "PipelineRun",
        "metadata": {"generateName": f"{pipeline_name}-retrain-"},
        "spec": {"pipelineRef": {"name": pipeline_name}},
    }
    resp = requests.post(
        f"{tekton_url}/apis/tekton.dev/v1/namespaces/ml-production/pipelineruns",
        json=body,
        headers={"Authorization": f"Bearer {token}"},
        timeout=10,
    )
    resp.raise_for_status()
    print(f"Retraining pipeline triggered: {resp.json()['metadata']['name']}")


if __name__ == "__main__":
    reference_df = pd.read_parquet("s3://ml-data/reference/features.parquet")
    current_df   = fetch_production_data("http://model-serving.ml-production.svc")

    drifted, share = check_drift(reference_df, current_df)
    print(f"Drift detected: {drifted}  (share={share:.0%})")

    if drifted and share > 0.3:   # >30% of features drifted
        trigger_retraining(
            tekton_url="https://api.cluster.example.com:6443",
            pipeline_name="fraud-training-pipeline",
            token="<sa-token>",
        )
```

---

## AI-Driven Incident Response

---

### 🔴 Q14. How do you build an AI-powered runbook assistant that helps on-call engineers resolve incidents faster?

**Answer:**

**Architecture:**
```
PagerDuty alert → Webhook → Python service → LLM (RAG over runbooks + past incidents) → Slack thread
```

```python
# runbook_assistant.py — RAG-based incident responder
import os
import json
import requests
from openai import OpenAI
from pathlib import Path

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
SLACK_WEBHOOK = os.environ["SLACK_WEBHOOK_URL"]


def load_runbooks(runbook_dir: str) -> str:
    """Load all runbook markdown files into a single context string."""
    runbooks = []
    for f in sorted(Path(runbook_dir).rglob("*.md")):
        runbooks.append(f"=== {f.name} ===\n{f.read_text()}")
    return "\n\n".join(runbooks)[:15000]   # fit in context window


def fetch_recent_incidents(pd_api_key: str, service_id: str, limit: int = 5) -> str:
    """Pull recent resolved incidents for the same service from PagerDuty."""
    resp = requests.get(
        "https://api.pagerduty.com/incidents",
        headers={
            "Authorization": f"Token token={pd_api_key}",
            "Accept": "application/vnd.pagerduty+json;version=2",
        },
        params={"service_ids[]": service_id, "statuses[]": "resolved", "limit": limit},
        timeout=10,
    )
    resp.raise_for_status()
    incidents = resp.json()["incidents"]
    summaries = []
    for inc in incidents:
        summaries.append(
            f"- [{inc['created_at'][:10]}] {inc['title']} → resolved in "
            f"{inc.get('acknowledgements', [{}])[0].get('summary', 'unknown')} mins"
        )
    return "\n".join(summaries) or "No recent incidents found."


def generate_response_plan(alert: dict, runbooks: str, past_incidents: str) -> str:
    system = """You are an expert SRE incident responder.
Given an alert, relevant runbooks, and past incidents, provide:
1. Most likely root cause (top 3 hypotheses)
2. Immediate diagnostic commands to run
3. Step-by-step remediation (reference runbook sections)
4. Escalation criteria
Be concise and actionable. Use numbered lists."""

    user = f"""
ALERT:
{json.dumps(alert, indent=2)}

RELEVANT RUNBOOKS:
{runbooks}

RECENT SIMILAR INCIDENTS:
{past_incidents}
"""
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": system},
            {"role": "user",   "content": user},
        ],
        temperature=0.2,
        max_tokens=1200,
    )
    return response.choices[0].message.content


def handle_alert(alert_payload: dict) -> None:
    service_id = alert_payload.get("service", {}).get("id", "")
    runbooks = load_runbooks("./runbooks")
    past = fetch_recent_incidents(
        os.environ["PAGERDUTY_API_KEY"], service_id
    )
    plan = generate_response_plan(alert_payload, runbooks, past)

    message = (
        f"*🚨 Incident: {alert_payload.get('title','?')}*\n"
        f"Service: `{alert_payload.get('service',{}).get('name','?')}`\n\n"
        f"*🤖 AI Response Plan:*\n{plan}"
    )
    requests.post(SLACK_WEBHOOK, json={"text": message}, timeout=5).raise_for_status()
    print("AI incident response plan posted to Slack.")
```

---

## Responsible AI in DevOps

---

### 🟡 Q15. What are the risks of using AI in a DevOps pipeline and how do you mitigate them?

**Answer:**

| Risk | Description | Mitigation |
|------|-------------|------------|
| **Hallucination** | LLM generates plausible but wrong YAML/code | Always validate AI output with `kubeconform`, `terraform validate`, unit tests |
| **Secret leakage** | Source code sent to external LLM API may contain secrets | Strip secrets before sending; use self-hosted models for sensitive repos |
| **Over-automation** | AI auto-applies changes without human approval | Enforce human review gates for infra changes; AI = suggest, not act |
| **Model drift** | Anomaly detection model trained on old traffic patterns becomes stale | Retrain models on a schedule; monitor model accuracy metrics |
| **Vendor lock-in** | Heavy dependency on one LLM provider (OpenAI, etc.) | Abstract LLM calls behind an interface; test with multiple providers |
| **Bias in test selection** | Predictive test selection ML model may have gaps for new code paths | Fall back to full suite for new files with no test history |
| **Audit trail** | AI decisions are opaque | Log every LLM prompt + response with timestamp for audit |
| **Cost runaway** | Uncontrolled LLM API calls in CI can spike costs | Implement per-PR token budgets; cache responses for identical diffs |

**Governance checklist:**
```
☐ All LLM calls logged with input/output for audit
☐ AI review is advisory only — cannot block a merge without human sign-off
☐ Secrets scanned and removed before any data leaves the cluster
☐ Self-hosted model option available for air-gapped / regulated environments
☐ Model performance reviewed quarterly
☐ Incident postmortems review whether AI recommendations were correct
```

---

> [← Python Scripting](./08-Python-Scripting.md) | [README](./README.md)