# 🐍 Module 8 — Python Scripting for DevOps

> **JD Alignment:** "Automate infrastructure provisioning and configuration management. Build tooling to support CI/CD pipelines, cluster operations, and observability workflows."

> [← Behavioral](./07-Behavioral.md) | [README](./README.md)

---

## Table of Contents

1. [Python Fundamentals for DevOps](#python-fundamentals-for-devops)
2. [File & System Operations](#file--system-operations)
3. [Working with APIs & HTTP](#working-with-apis--http)
4. [Kubernetes & OpenShift Automation](#kubernetes--openshift-automation)
5. [CI/CD Pipeline Scripting](#cicd-pipeline-scripting)
6. [Data Formats: JSON, YAML, TOML](#data-formats-json-yaml-toml)
7. [Subprocess & Shell Integration](#subprocess--shell-integration)
8. [Error Handling & Logging](#error-handling--logging)
9. [Testing DevOps Scripts](#testing-devops-scripts)
10. [Advanced Patterns](#advanced-patterns)

---

## Python Fundamentals for DevOps

---

### 🟢 Q1. What Python features are most useful in a DevOps context?

**Answer:**

| Feature | DevOps Use Case |
|---------|----------------|
| `subprocess` / `shlex` | Run CLI tools (`kubectl`, `oc`, `helm`, `git`) from scripts |
| `pathlib` / `os` | Manage config files, log dirs, temp workspace |
| `argparse` | Build self-documenting CLI tools for engineers |
| `logging` | Structured output with severity levels |
| `requests` / `httpx` | Call REST APIs — GitHub, Jira, Slack, Vault, Prometheus |
| `pyyaml` / `ruamel.yaml` | Parse and patch Kubernetes/Helm YAML manifests |
| `kubernetes` SDK | Interact with cluster resources without shelling out |
| `boto3` | AWS automation — EC2, S3, EKS, Secrets Manager |
| `dataclasses` / `pydantic` | Typed config objects, JSON schema validation |
| `concurrent.futures` | Parallelise multi-cluster ops |
| `pytest` | Unit-test scripts and automation modules |

**Golden rules for DevOps Python:**
1. **Fail loudly** — raise exceptions or exit non-zero; never swallow errors silently.
2. **Idempotent** — running the script twice should produce the same state.
3. **Config over code** — accept environment variables and flags; avoid hard-coded values.
4. **Log everything** — use `logging`, not `print`, so severity and timestamps are captured.

---

### 🟢 Q2. How do you manage dependencies and virtual environments in a DevOps Python project?

**Answer:**

```bash
# Create isolated environment
python3 -m venv .venv
source .venv/bin/activate          # Linux/macOS
# .venv\Scripts\Activate.ps1       # Windows PowerShell

# Pin dependencies for reproducible builds
pip install -r requirements.txt
pip freeze > requirements.txt
```

**`requirements.txt` for a typical DevOps script:**
```
kubernetes==29.0.0
pyyaml==6.0.1
requests==2.31.0
boto3==1.34.0
click==8.1.7
pydantic==2.6.0
pytest==8.0.0
pytest-mock==3.12.0
```

**Best practice — use `pip-tools` for dependency locking:**
```bash
pip install pip-tools
# requirements.in — direct deps only
echo "kubernetes\nrequests\npyyaml" > requirements.in
pip-compile requirements.in           # generates requirements.txt with pinned hashes
pip-sync requirements.txt             # installs exactly what's in the lock file
```

**In a container / CI job:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENTRYPOINT ["python", "main.py"]
```

---

## File & System Operations

---

### 🟢 Q3. How do you read, modify, and write a YAML Kubernetes manifest with Python?

**Answer:**

```python
import pathlib
import yaml  # pip install pyyaml

def bump_image_tag(manifest_path: str, new_tag: str) -> None:
    """Update the container image tag in a Kubernetes Deployment manifest."""
    path = pathlib.Path(manifest_path)
    manifest = yaml.safe_load(path.read_text())

    containers = (
        manifest
        .get("spec", {})
        .get("template", {})
        .get("spec", {})
        .get("containers", [])
    )

    for container in containers:
        image, _, _ = container["image"].rpartition(":")
        container["image"] = f"{image}:{new_tag}"

    path.write_text(yaml.dump(manifest, default_flow_style=False))
    print(f"Updated {manifest_path} → tag={new_tag}")


if __name__ == "__main__":
    bump_image_tag("deployment.yaml", "v2.4.1")
```

**Handling multi-document YAML (e.g., a Helm render output):**
```python
import yaml

def patch_all_docs(raw_yaml: str, new_tag: str) -> str:
    docs = list(yaml.safe_load_all(raw_yaml))
    for doc in docs:
        if doc and doc.get("kind") == "Deployment":
            for c in doc["spec"]["template"]["spec"]["containers"]:
                image, _, _ = c["image"].rpartition(":")
                c["image"] = f"{image}:{new_tag}"
    return "---\n".join(yaml.dump(d) for d in docs)
```

---

### 🟡 Q4. How do you recursively search a directory for all Kubernetes manifests and validate required labels?

**Answer:**

```python
import pathlib
import sys
import yaml

REQUIRED_LABELS = {"app", "version", "team"}

def validate_manifests(root: str) -> int:
    """Return number of violations found."""
    violations = 0
    for yaml_file in pathlib.Path(root).rglob("*.yaml"):
        docs = yaml.safe_load_all(yaml_file.read_text())
        for doc in docs:
            if not doc or "metadata" not in doc:
                continue
            labels = set(doc["metadata"].get("labels", {}).keys())
            missing = REQUIRED_LABELS - labels
            if missing:
                print(
                    f"[FAIL] {yaml_file} ({doc.get('kind','?')}/{doc['metadata'].get('name','?')})"
                    f" missing labels: {missing}"
                )
                violations += 1
    return violations


if __name__ == "__main__":
    root = sys.argv[1] if len(sys.argv) > 1 else "."
    count = validate_manifests(root)
    if count:
        print(f"\n{count} violation(s) found.")
        sys.exit(1)
    print("All manifests valid.")
```

---

## Working with APIs & HTTP

---

### 🟡 Q5. How do you query the GitHub API to list open pull requests and post a comment?

**Answer:**

```python
import os
import requests

GITHUB_TOKEN = os.environ["GITHUB_TOKEN"]   # never hard-code
REPO         = os.environ.get("GITHUB_REPO", "my-org/my-repo")
BASE_URL     = "https://api.github.com"

session = requests.Session()
session.headers.update({
    "Authorization": f"Bearer {GITHUB_TOKEN}",
    "Accept": "application/vnd.github+json",
    "X-GitHub-Api-Version": "2022-11-28",
})


def list_open_prs() -> list[dict]:
    """Return all open pull requests for the repo."""
    prs, page = [], 1
    while True:
        resp = session.get(
            f"{BASE_URL}/repos/{REPO}/pulls",
            params={"state": "open", "per_page": 100, "page": page},
        )
        resp.raise_for_status()
        batch = resp.json()
        if not batch:
            break
        prs.extend(batch)
        page += 1
    return prs


def post_comment(pr_number: int, body: str) -> None:
    """Post a comment on a pull request."""
    resp = session.post(
        f"{BASE_URL}/repos/{REPO}/issues/{pr_number}/comments",
        json={"body": body},
    )
    resp.raise_for_status()
    print(f"Commented on PR #{pr_number}")


if __name__ == "__main__":
    for pr in list_open_prs():
        print(f"  #{pr['number']} — {pr['title']} ({pr['user']['login']})")
    # post_comment(42, "✅ Automated scan passed.")
```

---

### 🟡 Q6. How do you call the Prometheus HTTP API to check if a service is returning alerts?

**Answer:**

```python
import os
import sys
import requests

PROMETHEUS_URL = os.environ.get("PROMETHEUS_URL", "http://prometheus:9090")


def get_firing_alerts(service: str) -> list[dict]:
    """Return firing alerts for a given service label."""
    resp = requests.get(
        f"{PROMETHEUS_URL}/api/v1/alerts",
        timeout=10,
    )
    resp.raise_for_status()
    alerts = resp.json()["data"]["alerts"]
    return [
        a for a in alerts
        if a["state"] == "firing"
        and a["labels"].get("service") == service
    ]


def query_metric(promql: str) -> list:
    """Run an instant PromQL query."""
    resp = requests.get(
        f"{PROMETHEUS_URL}/api/v1/query",
        params={"query": promql},
        timeout=10,
    )
    resp.raise_for_status()
    return resp.json()["data"]["result"]


if __name__ == "__main__":
    service = sys.argv[1] if len(sys.argv) > 1 else "my-service"
    firing = get_firing_alerts(service)
    if firing:
        for a in firing:
            print(f"[ALERT] {a['labels']['alertname']} — {a['annotations'].get('summary','')}")
        sys.exit(1)
    print(f"No firing alerts for {service}")

    # Example: CPU usage > 80%
    results = query_metric(
        f'avg(rate(container_cpu_usage_seconds_total{{service="{service}"}}[5m])) * 100'
    )
    for r in results:
        print(f"CPU: {float(r['value'][1]):.1f}%")
```

---

## Kubernetes & OpenShift Automation

---

### 🟡 Q7. How do you use the Python Kubernetes SDK to list pods and restart a deployment?

**Answer:**

```python
import os
from kubernetes import client, config

# Load in-cluster config (inside a pod) or kubeconfig (local dev)
def load_k8s():
    if os.getenv("KUBERNETES_SERVICE_HOST"):
        config.load_incluster_config()
    else:
        config.load_kube_config()


def list_pods(namespace: str) -> None:
    load_k8s()
    v1 = client.CoreV1Api()
    pods = v1.list_namespaced_pod(namespace=namespace)
    for pod in pods.items:
        phase = pod.status.phase
        ready = all(cs.ready for cs in (pod.status.container_statuses or []))
        print(f"  {pod.metadata.name:60s} {phase:10s} ready={ready}")


def restart_deployment(namespace: str, name: str) -> None:
    """Trigger a rollout restart by patching the pod template annotation."""
    load_k8s()
    apps_v1 = client.AppsV1Api()
    import datetime

    patch = {
        "spec": {
            "template": {
                "metadata": {
                    "annotations": {
                        "kubectl.kubernetes.io/restartedAt": (
                            datetime.datetime.utcnow().isoformat() + "Z"
                        )
                    }
                }
            }
        }
    }
    apps_v1.patch_namespaced_deployment(name=name, namespace=namespace, body=patch)
    print(f"Restarted deployment {namespace}/{name}")


def scale_deployment(namespace: str, name: str, replicas: int) -> None:
    load_k8s()
    apps_v1 = client.AppsV1Api()
    apps_v1.patch_namespaced_deployment_scale(
        name=name,
        namespace=namespace,
        body={"spec": {"replicas": replicas}},
    )
    print(f"Scaled {namespace}/{name} → {replicas} replicas")


if __name__ == "__main__":
    list_pods("production")
    restart_deployment("production", "my-app")
```

---

### 🟡 Q8. How do you watch a Kubernetes deployment rollout and fail the script if it does not become ready?

**Answer:**

```python
import sys
import time
from kubernetes import client, config, watch


def wait_for_rollout(namespace: str, name: str, timeout_seconds: int = 300) -> bool:
    """
    Wait until a Deployment's desired replicas == ready replicas.
    Returns True on success, False on timeout.
    """
    config.load_kube_config()
    apps_v1 = client.AppsV1Api()
    deadline = time.time() + timeout_seconds

    print(f"Waiting for {namespace}/{name} (timeout={timeout_seconds}s)...")
    while time.time() < deadline:
        deploy = apps_v1.read_namespaced_deployment(name=name, namespace=namespace)
        desired  = deploy.spec.replicas or 0
        ready    = deploy.status.ready_replicas or 0
        updated  = deploy.status.updated_replicas or 0
        available= deploy.status.available_replicas or 0

        print(f"  desired={desired} updated={updated} ready={ready} available={available}")

        if desired == updated == ready == available and desired > 0:
            print(f"✅ Rollout complete.")
            return True

        time.sleep(5)

    print(f"❌ Rollout timed out after {timeout_seconds}s.")
    return False


if __name__ == "__main__":
    ok = wait_for_rollout("production", "my-app", timeout_seconds=300)
    sys.exit(0 if ok else 1)
```

---

### 🔴 Q9. Write a Python script that checks all namespaces for pods NOT in Running/Completed state and sends a Slack alert.

**Answer:**

```python
import os
import sys
import requests
from kubernetes import client, config

SLACK_WEBHOOK = os.environ["SLACK_WEBHOOK_URL"]
IGNORE_PHASES = {"Running", "Succeeded"}
IGNORE_NS_PREFIXES = ("kube-", "openshift-")


def load_k8s():
    if os.getenv("KUBERNETES_SERVICE_HOST"):
        config.load_incluster_config()
    else:
        config.load_kube_config()


def get_unhealthy_pods() -> list[dict]:
    load_k8s()
    v1 = client.CoreV1Api()
    namespaces = [ns.metadata.name for ns in v1.list_namespace().items]
    unhealthy = []

    for ns in namespaces:
        if any(ns.startswith(p) for p in IGNORE_NS_PREFIXES):
            continue
        pods = v1.list_namespaced_pod(namespace=ns)
        for pod in pods.items:
            phase = pod.status.phase or "Unknown"
            if phase not in IGNORE_PHASES:
                unhealthy.append({
                    "namespace": ns,
                    "name": pod.metadata.name,
                    "phase": phase,
                    "reason": (pod.status.reason or ""),
                })
    return unhealthy


def send_slack_alert(pods: list[dict]) -> None:
    lines = [f"*🚨 Unhealthy Pods Detected ({len(pods)})*"]
    for p in pods[:20]:   # cap at 20 to avoid Slack message size limits
        lines.append(
            f"  • `{p['namespace']}/{p['name']}` — *{p['phase']}*"
            + (f" ({p['reason']})" if p["reason"] else "")
        )
    if len(pods) > 20:
        lines.append(f"  _...and {len(pods) - 20} more_")

    resp = requests.post(SLACK_WEBHOOK, json={"text": "\n".join(lines)}, timeout=10)
    resp.raise_for_status()
    print("Slack alert sent.")


if __name__ == "__main__":
    pods = get_unhealthy_pods()
    if pods:
        for p in pods:
            print(f"[{p['phase']}] {p['namespace']}/{p['name']}")
        send_slack_alert(pods)
        sys.exit(1)
    print("All pods healthy.")
```

---

## CI/CD Pipeline Scripting

---

### 🟡 Q10. How do you write a Python script to bump a version tag in `package.json` and push a git tag?

**Answer:**

```python
#!/usr/bin/env python3
"""
bump_version.py — semantic version bumper for package.json
Usage: python bump_version.py [major|minor|patch]
"""
import json
import subprocess
import sys
from pathlib import Path


def bump(version: str, part: str) -> str:
    major, minor, patch = (int(x) for x in version.split("."))
    match part:
        case "major": return f"{major + 1}.0.0"
        case "minor": return f"{major}.{minor + 1}.0"
        case "patch": return f"{major}.{minor}.{patch + 1}"
        case _: raise ValueError(f"Unknown bump part: {part}")


def run(cmd: list[str]) -> str:
    result = subprocess.run(cmd, check=True, capture_output=True, text=True)
    return result.stdout.strip()


def main() -> None:
    part = sys.argv[1] if len(sys.argv) > 1 else "patch"
    pkg_path = Path("package.json")
    pkg = json.loads(pkg_path.read_text())
    old = pkg["version"]
    new = bump(old, part)
    pkg["version"] = new
    pkg_path.write_text(json.dumps(pkg, indent=2) + "\n")

    print(f"Version: {old} → {new}")
    run(["git", "add", "package.json"])
    run(["git", "commit", "-m", f"chore: bump version to {new}"])
    run(["git", "tag", f"v{new}"])
    run(["git", "push", "--follow-tags"])
    print(f"Tagged and pushed v{new}")


if __name__ == "__main__":
    main()
```

---

### 🟡 Q11. How do you write a Python script that checks image vulnerability scan results and fails the pipeline if critical CVEs exist?

**Answer:**

```python
#!/usr/bin/env python3
"""
check_trivy.py — parse Trivy JSON report and gate the pipeline
Usage: python check_trivy.py trivy-report.json [--max-critical 0] [--max-high 5]
"""
import argparse
import json
import sys
from pathlib import Path


def parse_args():
    p = argparse.ArgumentParser()
    p.add_argument("report", help="Path to Trivy JSON report")
    p.add_argument("--max-critical", type=int, default=0)
    p.add_argument("--max-high",     type=int, default=5)
    return p.parse_args()


def count_vulns(report_path: str) -> dict[str, int]:
    data = json.loads(Path(report_path).read_text())
    counts: dict[str, int] = {}
    for result in data.get("Results", []):
        for vuln in result.get("Vulnerabilities") or []:
            sev = vuln.get("Severity", "UNKNOWN")
            counts[sev] = counts.get(sev, 0) + 1
    return counts


def main() -> None:
    args = parse_args()
    counts = count_vulns(args.report)

    print("Vulnerability Summary:")
    for sev in ("CRITICAL", "HIGH", "MEDIUM", "LOW", "UNKNOWN"):
        n = counts.get(sev, 0)
        icon = "🔴" if sev in ("CRITICAL", "HIGH") and n > 0 else "🟢"
        print(f"  {icon} {sev:10s}: {n}")

    failed = False
    if counts.get("CRITICAL", 0) > args.max_critical:
        print(f"\n❌ CRITICAL count {counts['CRITICAL']} exceeds max {args.max_critical}")
        failed = True
    if counts.get("HIGH", 0) > args.max_high:
        print(f"❌ HIGH count {counts['HIGH']} exceeds max {args.max_high}")
        failed = True

    if failed:
        sys.exit(1)
    print("\n✅ Vulnerability gate passed.")


if __name__ == "__main__":
    main()
```

---

## Data Formats: JSON, YAML, TOML

---

### 🟢 Q12. How do you safely parse and validate a JSON config file in Python?

**Answer:**

```python
import json
import sys
from pathlib import Path
from dataclasses import dataclass
from typing import Optional


@dataclass
class AppConfig:
    app_name: str
    namespace: str
    replicas: int
    image_registry: str
    slack_channel: Optional[str] = None

    @classmethod
    def from_dict(cls, data: dict) -> "AppConfig":
        required = {"app_name", "namespace", "replicas", "image_registry"}
        missing = required - data.keys()
        if missing:
            raise ValueError(f"Missing required config keys: {missing}")
        if not isinstance(data["replicas"], int) or data["replicas"] < 1:
            raise ValueError("replicas must be a positive integer")
        return cls(**{k: v for k, v in data.items() if k in cls.__dataclass_fields__})


def load_config(path: str) -> AppConfig:
    raw = Path(path).read_text()
    try:
        data = json.loads(raw)
    except json.JSONDecodeError as exc:
        raise SystemExit(f"Invalid JSON in {path}: {exc}")
    return AppConfig.from_dict(data)


if __name__ == "__main__":
    cfg = load_config(sys.argv[1])
    print(f"App: {cfg.app_name}  NS: {cfg.namespace}  Replicas: {cfg.replicas}")
```

---

### 🟡 Q13. How do you merge multiple YAML config files with override support (like Helm values)?

**Answer:**

```python
import copy
import yaml
from pathlib import Path


def deep_merge(base: dict, override: dict) -> dict:
    """
    Recursively merge `override` into `base`.
    Scalar values in `override` always win; nested dicts are merged.
    """
    result = copy.deepcopy(base)
    for key, val in override.items():
        if key in result and isinstance(result[key], dict) and isinstance(val, dict):
            result[key] = deep_merge(result[key], val)
        else:
            result[key] = copy.deepcopy(val)
    return result


def load_merged_config(*paths: str) -> dict:
    """
    Load YAML files in order; each file overrides the previous.
    First file = base (defaults.yaml), subsequent = environment overrides.
    """
    merged = {}
    for path in paths:
        data = yaml.safe_load(Path(path).read_text()) or {}
        merged = deep_merge(merged, data)
    return merged


if __name__ == "__main__":
    # Example: defaults.yaml → staging.yaml → secrets.yaml
    cfg = load_merged_config("defaults.yaml", "staging.yaml")
    print(yaml.dump(cfg, default_flow_style=False))
```

---

## Subprocess & Shell Integration

---

### 🟡 Q14. How do you safely run shell commands from Python and capture their output?

**Answer:**

```python
import shlex
import subprocess
import sys


def run(cmd: str | list[str], *, check: bool = True, capture: bool = True) -> subprocess.CompletedProcess:
    """
    Run a shell command safely.
    - Accepts a string (auto-split with shlex) or a list.
    - Never uses shell=True to avoid injection.
    - Raises CalledProcessError on non-zero exit when check=True.
    """
    if isinstance(cmd, str):
        cmd = shlex.split(cmd)

    result = subprocess.run(
        cmd,
        check=check,
        capture_output=capture,
        text=True,
    )
    return result


def oc_get_pods(namespace: str) -> str:
    result = run(f"oc get pods -n {namespace} -o wide")
    return result.stdout


def helm_deploy(release: str, chart: str, namespace: str, values_file: str) -> None:
    run([
        "helm", "upgrade", "--install", release, chart,
        "--namespace", namespace,
        "--create-namespace",
        "-f", values_file,
        "--wait",
        "--timeout", "10m0s",
    ])
    print(f"Helm release '{release}' deployed.")


def kubectl_apply(manifest_dir: str, dry_run: bool = False) -> None:
    cmd = ["kubectl", "apply", "-R", "-f", manifest_dir]
    if dry_run:
        cmd += ["--dry-run=client"]
    result = run(cmd)
    print(result.stdout)


if __name__ == "__main__":
    print(oc_get_pods("production"))
```

**What NOT to do:**
```python
# ❌ Never do this — vulnerable to shell injection
import os
namespace = input("Enter namespace: ")
os.system(f"kubectl get pods -n {namespace}")  # attacker can inject "; rm -rf /"

# ✅ Safe
run(["kubectl", "get", "pods", "-n", namespace])
```

---

### 🔴 Q15. How do you run multiple `kubectl` commands in parallel across clusters?

**Answer:**

```python
import concurrent.futures
import subprocess
from dataclasses import dataclass


@dataclass
class ClusterResult:
    cluster: str
    stdout: str
    stderr: str
    returncode: int

    @property
    def ok(self) -> bool:
        return self.returncode == 0


def run_on_cluster(cluster: str, kubeconfig: str, command: list[str]) -> ClusterResult:
    full_cmd = ["kubectl", "--kubeconfig", kubeconfig] + command
    proc = subprocess.run(full_cmd, capture_output=True, text=True)
    return ClusterResult(
        cluster=cluster,
        stdout=proc.stdout,
        stderr=proc.stderr,
        returncode=proc.returncode,
    )


def run_across_clusters(clusters: dict[str, str], command: list[str]) -> list[ClusterResult]:
    """
    clusters: {cluster_name: kubeconfig_path}
    command:  kubectl sub-command (e.g. ["get", "nodes"])
    """
    results = []
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        futures = {
            executor.submit(run_on_cluster, name, kconfig, command): name
            for name, kconfig in clusters.items()
        }
        for future in concurrent.futures.as_completed(futures):
            results.append(future.result())
    return results


if __name__ == "__main__":
    clusters = {
        "prod-eu":  "/home/user/.kube/prod-eu.yaml",
        "prod-us":  "/home/user/.kube/prod-us.yaml",
        "staging":  "/home/user/.kube/staging.yaml",
    }
    results = run_across_clusters(clusters, ["get", "nodes", "--no-headers"])
    for r in results:
        status = "✅" if r.ok else "❌"
        print(f"\n{status} {r.cluster}\n{r.stdout or r.stderr}")
```

---

## Error Handling & Logging

---

### 🟢 Q16. How do you set up structured logging in a DevOps Python script?

**Answer:**

```python
import logging
import sys
import os


def setup_logging(name: str = "devops-script") -> logging.Logger:
    """
    Configure a logger that:
    - Outputs to stdout (not stderr) so CI captures it
    - Uses ISO-8601 timestamps
    - Respects LOG_LEVEL env var (default INFO)
    """
    level = os.environ.get("LOG_LEVEL", "INFO").upper()
    logger = logging.getLogger(name)
    logger.setLevel(level)

    if not logger.handlers:
        handler = logging.StreamHandler(sys.stdout)
        handler.setFormatter(
            logging.Formatter(
                fmt="%(asctime)s %(levelname)-8s %(name)s — %(message)s",
                datefmt="%Y-%m-%dT%H:%M:%S",
            )
        )
        logger.addHandler(handler)

    return logger


log = setup_logging()


def deploy(app: str, version: str) -> None:
    log.info("Starting deployment app=%s version=%s", app, version)
    try:
        # ... deployment logic ...
        log.info("Deployment successful app=%s", app)
    except Exception as exc:
        log.error("Deployment failed app=%s error=%s", app, exc, exc_info=True)
        raise


# JSON structured logging (for log aggregators like Loki/Splunk):
import json


class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        return json.dumps({
            "ts":      self.formatTime(record, "%Y-%m-%dT%H:%M:%S"),
            "level":   record.levelname,
            "logger":  record.name,
            "msg":     record.getMessage(),
            **({"exc": self.formatException(record.exc_info)} if record.exc_info else {}),
        })
```

---

### 🟡 Q17. How do you implement retry logic with exponential back-off for flaky API calls?

**Answer:**

```python
import time
import logging
import functools
from typing import Callable, TypeVar

log = logging.getLogger(__name__)
T = TypeVar("T")


def with_retry(
    max_attempts: int = 5,
    base_delay: float = 1.0,
    backoff_factor: float = 2.0,
    exceptions: tuple = (Exception,),
) -> Callable:
    """
    Decorator: retry a function up to max_attempts times with exponential back-off.
    """
    def decorator(func: Callable[..., T]) -> Callable[..., T]:
        @functools.wraps(func)
        def wrapper(*args, **kwargs) -> T:
            delay = base_delay
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as exc:
                    if attempt == max_attempts:
                        log.error("All %d attempts failed: %s", max_attempts, exc)
                        raise
                    log.warning(
                        "Attempt %d/%d failed (%s), retrying in %.1fs…",
                        attempt, max_attempts, exc, delay,
                    )
                    time.sleep(delay)
                    delay *= backoff_factor
        return wrapper
    return decorator


# Usage
import requests

@with_retry(max_attempts=4, base_delay=2.0, exceptions=(requests.RequestException,))
def fetch_deployment_status(url: str) -> dict:
    resp = requests.get(url, timeout=5)
    resp.raise_for_status()
    return resp.json()
```

---

## Testing DevOps Scripts

---

### 🟡 Q18. How do you write unit tests for a script that calls `kubectl` and the Kubernetes API?

**Answer:**

```python
# test_k8s_ops.py
import pytest
from unittest.mock import MagicMock, patch
from k8s_ops import wait_for_rollout, restart_deployment   # your module


class TestWaitForRollout:

    @patch("k8s_ops.client.AppsV1Api")
    @patch("k8s_ops.config.load_kube_config")
    def test_success_on_first_poll(self, mock_cfg, MockAppsV1):
        deploy = MagicMock()
        deploy.spec.replicas = 2
        deploy.status.ready_replicas = 2
        deploy.status.updated_replicas = 2
        deploy.status.available_replicas = 2
        MockAppsV1.return_value.read_namespaced_deployment.return_value = deploy

        result = wait_for_rollout("prod", "my-app", timeout_seconds=30)
        assert result is True

    @patch("k8s_ops.time.sleep")   # prevent real sleeping in tests
    @patch("k8s_ops.time.time")
    @patch("k8s_ops.client.AppsV1Api")
    @patch("k8s_ops.config.load_kube_config")
    def test_timeout_returns_false(self, mock_cfg, MockAppsV1, mock_time, mock_sleep):
        # Simulate time always past deadline
        mock_time.side_effect = [0, 999]
        deploy = MagicMock()
        deploy.spec.replicas = 2
        deploy.status.ready_replicas = 0
        deploy.status.updated_replicas = 0
        deploy.status.available_replicas = 0
        MockAppsV1.return_value.read_namespaced_deployment.return_value = deploy

        result = wait_for_rollout("prod", "my-app", timeout_seconds=10)
        assert result is False


class TestSubprocessCalls:

    @patch("k8s_ops.subprocess.run")
    def test_helm_deploy_calls_correct_command(self, mock_run):
        from k8s_ops import helm_deploy
        mock_run.return_value = MagicMock(returncode=0, stdout="", stderr="")

        helm_deploy("my-release", "./chart", "production", "values.yaml")

        mock_run.assert_called_once()
        cmd = mock_run.call_args[0][0]
        assert cmd[0] == "helm"
        assert "upgrade" in cmd
        assert "my-release" in cmd
```

**Run tests:**
```bash
pytest tests/ -v --cov=. --cov-report=term-missing
```

---

## Advanced Patterns

---

### 🔴 Q19. How do you build a reusable CLI tool for DevOps automation using `click`?

**Answer:**

```python
#!/usr/bin/env python3
"""
devtool.py — A reusable DevOps CLI built with Click
Usage:
  python devtool.py deploy --app my-app --env staging --tag v1.2.3
  python devtool.py pods --namespace production
  python devtool.py scan --image ghcr.io/org/app:v1.2.3
"""
import sys
import click
import subprocess
import requests
import os

CLUSTER_URL = os.environ.get("OCP_SERVER", "https://api.cluster.example.com:6443")


@click.group()
@click.option("--debug", is_flag=True, help="Enable debug logging")
@click.pass_context
def cli(ctx: click.Context, debug: bool) -> None:
    """DevOps automation tool for OpenShift/Kubernetes operations."""
    ctx.ensure_object(dict)
    ctx.obj["debug"] = debug
    if debug:
        import logging
        logging.basicConfig(level=logging.DEBUG)


@cli.command()
@click.option("--app",  required=True, help="Application name")
@click.option("--env",  required=True, type=click.Choice(["dev", "staging", "production"]))
@click.option("--tag",  required=True, help="Image tag to deploy")
@click.option("--dry-run", is_flag=True, help="Show commands without executing")
@click.pass_context
def deploy(ctx: click.Context, app: str, env: str, tag: str, dry_run: bool) -> None:
    """Deploy an application to an OpenShift namespace."""
    namespace = f"{app}-{env}"
    image = f"ghcr.io/my-org/{app}:{tag}"
    cmd = ["oc", "set", "image", f"deployment/{app}", f"app={image}", "-n", namespace]

    if dry_run:
        click.echo(f"[dry-run] Would run: {' '.join(cmd)}")
        return

    click.echo(f"🚀 Deploying {app}:{tag} → {namespace}")
    try:
        subprocess.run(cmd, check=True)
        click.echo(f"✅ Deployment initiated.")
    except subprocess.CalledProcessError as exc:
        click.secho(f"❌ Deploy failed: {exc}", fg="red", err=True)
        sys.exit(1)


@cli.command()
@click.option("--namespace", "-n", default="default", show_default=True)
def pods(namespace: str) -> None:
    """List pods in a namespace."""
    result = subprocess.run(
        ["kubectl", "get", "pods", "-n", namespace, "-o", "wide"],
        check=True, capture_output=True, text=True,
    )
    click.echo(result.stdout)


@cli.command()
@click.option("--image", required=True, help="Full image reference to scan")
@click.option("--fail-on", default="CRITICAL", type=click.Choice(["CRITICAL","HIGH","MEDIUM"]))
def scan(image: str, fail_on: str) -> None:
    """Run a Trivy vulnerability scan against an image."""
    result = subprocess.run(
        ["trivy", "image", "--exit-code", "1", "--severity", fail_on, image],
        capture_output=False,
    )
    sys.exit(result.returncode)


if __name__ == "__main__":
    cli()
```

---

### 🔴 Q20. How do you write a Python script that automatically rotates a Kubernetes Secret from HashiCorp Vault?

**Answer:**

```python
#!/usr/bin/env python3
"""
rotate_secret.py — Fetch a secret from Vault and update a Kubernetes Secret.
Intended to run as a CronJob inside the cluster (uses in-cluster SA token for Vault auth).
"""
import os
import sys
import base64
import logging
import requests
from kubernetes import client, config

log = logging.getLogger("rotate_secret")
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")

VAULT_ADDR      = os.environ["VAULT_ADDR"]
VAULT_ROLE      = os.environ["VAULT_ROLE"]           # Vault Kubernetes auth role
VAULT_SECRET    = os.environ["VAULT_SECRET_PATH"]    # e.g. secret/data/myapp/db
K8S_NAMESPACE   = os.environ["K8S_NAMESPACE"]
K8S_SECRET_NAME = os.environ["K8S_SECRET_NAME"]
SA_TOKEN_FILE   = "/var/run/secrets/kubernetes.io/serviceaccount/token"


def get_vault_token() -> str:
    """Authenticate to Vault using the pod's ServiceAccount JWT."""
    sa_jwt = open(SA_TOKEN_FILE).read().strip()
    resp = requests.post(
        f"{VAULT_ADDR}/v1/auth/kubernetes/login",
        json={"role": VAULT_ROLE, "jwt": sa_jwt},
        timeout=10,
    )
    resp.raise_for_status()
    token = resp.json()["auth"]["client_token"]
    log.info("Vault token obtained.")
    return token


def read_vault_secret(vault_token: str, path: str) -> dict[str, str]:
    """Read a KV-v2 secret from Vault."""
    resp = requests.get(
        f"{VAULT_ADDR}/v1/{path}",
        headers={"X-Vault-Token": vault_token},
        timeout=10,
    )
    resp.raise_for_status()
    data = resp.json()["data"]["data"]
    log.info("Read Vault secret from %s (%d keys).", path, len(data))
    return data


def update_k8s_secret(namespace: str, name: str, data: dict[str, str]) -> None:
    """Create or update a Kubernetes Secret with base64-encoded values."""
    config.load_incluster_config()
    v1 = client.CoreV1Api()

    encoded = {k: base64.b64encode(v.encode()).decode() for k, v in data.items()}
    body = client.V1Secret(
        metadata=client.V1ObjectMeta(name=name, namespace=namespace),
        data=encoded,
        type="Opaque",
    )

    try:
        v1.read_namespaced_secret(name=name, namespace=namespace)
        v1.replace_namespaced_secret(name=name, namespace=namespace, body=body)
        log.info("Updated existing Secret %s/%s.", namespace, name)
    except client.ApiException as exc:
        if exc.status == 404:
            v1.create_namespaced_secret(namespace=namespace, body=body)
            log.info("Created Secret %s/%s.", namespace, name)
        else:
            raise


def main() -> None:
    try:
        token = get_vault_token()
        secret_data = read_vault_secret(token, VAULT_SECRET)
        update_k8s_secret(K8S_NAMESPACE, K8S_SECRET_NAME, secret_data)
        log.info("Secret rotation complete.")
    except Exception as exc:
        log.error("Secret rotation failed: %s", exc, exc_info=True)
        sys.exit(1)


if __name__ == "__main__":
    main()
```

**Deploy as a Kubernetes CronJob:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: secret-rotator
  namespace: production
spec:
  schedule: "0 */6 * * *"      # every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: vault-rotator-sa
          restartPolicy: OnFailure
          containers:
            - name: rotator
              image: ghcr.io/my-org/rotate-secret:latest
              env:
                - name: VAULT_ADDR
                  value: "https://vault.example.com"
                - name: VAULT_ROLE
                  value: "secret-rotator"
                - name: VAULT_SECRET_PATH
                  value: "secret/data/production/db"
                - name: K8S_NAMESPACE
                  value: "production"
                - name: K8S_SECRET_NAME
                  value: "db-credentials"
              resources:
                requests:
                  cpu: 50m
                  memory: 64Mi
                limits:
                  cpu: 200m
                  memory: 128Mi
```

---

> [← Behavioral](./07-Behavioral.md) | [README](./README.md)
