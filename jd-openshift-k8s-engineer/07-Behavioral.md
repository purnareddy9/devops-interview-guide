---
render_with_liquid: false
layout: default
title: "🟣 Module 7 — Behavioral Questions"
---

# 🟣 Module 7 — Behavioral Questions

> **JD Alignment:** All behavioral questions are mapped to specific JD requirements. Answers follow the **STAR method**: Situation → Task → Action → Result.

> [← Scenario-Based](./06-Scenario-Based.md) | [README](./README.md)

---

## How to Use STAR

| Component | What to say |
|-----------|------------|
| **S**ituation | Set the context — where, when, what was at stake |
| **T**ask | Your specific responsibility in the situation |
| **A**ction | The concrete steps YOU took (use "I", not "we") |
| **R**esult | Quantifiable outcome + what you learned |

**Tips:**
- Keep each answer 2–3 minutes when spoken aloud.
- Quantify results: "reduced deployment time by 60%", "achieved 99.9% uptime".
- Always end with what you learned or how you'd do it differently.
- Prepare 2 versions of each story — one for positive outcomes, one for "failure/learning".

---

## Questions by JD Area

---

### 🟣 JD: "Design, implement and maintain CI/CD pipelines"

**Q1: Tell me about a CI/CD pipeline you designed from scratch. What were the challenges and how did you overcome them?**

**Strong Answer Framework:**

> **Situation:** At [Company], the team was deploying manually — developers emailed ZIP files to ops, who manually SSH'd to servers. Deployments took 4 hours and had a 30% failure rate. We were on-call every time.
>
> **Task:** I was asked to design and implement an automated CI/CD pipeline for 12 microservices deploying to OpenShift.
>
> **Action:**
> 1. I started by mapping the current manual process to identify all the implicit steps — dependency installation, test execution, image build, registry push, deployment, smoke test. This became the pipeline template.
> 2. I chose GitLab CI + OpenShift Pipelines (Tekton) because GitLab was already used for source control and Tekton was native to OCP.
> 3. I created a shared pipeline template in GitLab that teams could inherit with 3 lines of YAML — reducing duplication across 12 services.
> 4. The hardest part: several legacy services ran as root, which OpenShift's SCC blocked. I worked with each team to fix their Dockerfiles — `USER 1001` directive. For two services with genuine OS-level needs, I created a custom SCC granting only what was needed.
> 5. I added automated rollback: if the post-deploy health check failed within 5 minutes, the pipeline automatically ran `oc rollout undo`.
>
> **Result:** Deployment time dropped from 4 hours to 8 minutes. Failure rate went from 30% to under 3%. The team stopped being on-call for deployments. We went from 2 deploys/week to 15+/week.
>
> **Lesson:** The technical work was 40% of the effort. The other 60% was working with teams to fix their applications and change their habits — communication matters as much as the tooling.

---

**Q2: Describe a time a pipeline failed in production. What happened and what did you do?**

> **Situation:** A Tekton pipeline deployed a new image to production that passed all automated tests in staging but caused 500 errors in production due to a database schema migration that wasn't backward-compatible.
>
> **Task:** Restore service as quickly as possible and prevent recurrence.
>
> **Action:**
> 1. Immediately ran `oc rollout undo deployment/payment-api` — service restored in 90 seconds.
> 2. Coordinated with the database team to roll back the migration (this was the harder part — we had to write a reverse migration script on the fly).
> 3. Did a post-mortem: the root cause was that staging used a 1-week-old copy of the production database, so the schema difference wasn't caught.
> 4. Implemented a fix: added a migration compatibility test to the pipeline — the new version starts alongside the old version (expand phase), both must serve requests; only then does the old version shut down (contract phase — Expand/Contract pattern).
> 5. Also updated staging to refresh with a production snapshot every 24 hours.
>
> **Result:** Zero recurrence of schema-related production incidents in the following 18 months. The expand/contract pattern became a company standard.

---

### 🟣 JD: "Manage and administer OpenShift and Kubernetes clusters to ensure high availability and scalability"

**Q3: Tell me about the most complex OpenShift cluster issue you've resolved.**

> **Situation:** During a peak traffic event (Black Friday), our production OpenShift cluster started experiencing cascading pod evictions — 40% of pods across 3 namespaces were evicted within 10 minutes.
>
> **Task:** As the on-call platform engineer, I needed to stabilize the cluster and restore service during the highest-traffic window of the year.
>
> **Action:**
> 1. Immediate: opened war-room Slack channel, looped in app teams to know which services were impacted.
> 2. Ran `oc adm top nodes` — all 6 worker nodes were showing >95% memory.
> 3. The trigger was a batch job that had been mistakenly scheduled with no resource limits — it consumed all available memory on 3 nodes within minutes.
> 4. I deleted the runaway batch job immediately: `oc delete job runaway-job -n processing`.
> 5. Nodes were still under pressure — ran `oc rollout restart` on the most critical services to get them rescheduled on the partially-recovered nodes.
> 6. Added 3 nodes to the cluster via MachineSet to absorb the load while the existing nodes recovered.
>
> **Result:** Full service restoration in 22 minutes. Revenue impact was contained.
>
> **Long-term:** I implemented LimitRange defaults so no container could be created without resource limits. Added a Kyverno policy to reject any Job/CronJob without limits in production. Added node memory alert at 80% (not 95%) to give earlier warning.

---

**Q4: How have you ensured high availability in a containerized environment?**

> **Situation:** My team was running a critical payment processing service with a 99.9% SLA requirement.
>
> **Task:** Design the deployment architecture to meet the SLA and survive node failures and cluster upgrades without downtime.
>
> **Action:**
> 1. Set `replicas: 3` with `podAntiAffinity` requiring pods to be on different availability zones.
> 2. Implemented `PodDisruptionBudget` with `minAvailable: 2` — ensuring at least 2 pods always running during node drains.
> 3. Configured `maxUnavailable: 0` in the rolling update strategy — never reduces below 3 during deployments.
> 4. Added readiness probe that checks DB connectivity — pod only receives traffic when truly ready.
> 5. Prepped a runbook for the quarterly Chaos Engineering exercise — randomly killing pods, draining nodes, and verifying the SLA held.
>
> **Result:** Achieved 99.97% uptime over 12 months. Successfully survived two planned cluster upgrades with zero downtime.

---

### 🟣 JD: "Collaborate with development and operations teams to streamline software delivery"

**Q5: Describe a time you had to influence a team to adopt a DevOps practice they were resistant to.**

> **Situation:** Our development team was resistant to adding security scanning (Trivy) to the CI pipeline. They argued it would slow down the pipeline and "we don't have vulnerabilities."
>
> **Task:** Get the security scan adopted without creating adversarial friction.
>
> **Action:**
> 1. Rather than mandating the scan, I first ran Trivy against all existing production images and presented the results — 3 images had CRITICAL CVEs, including Log4Shell.
> 2. This created urgency. The conversation shifted from "we don't need this" to "how do we fix these?"
> 3. I made the scan non-blocking initially — warnings only, no pipeline failure. This gave teams time to fix images without feeling blocked.
> 4. After 4 weeks, I made it block on CRITICAL only (not HIGH or MEDIUM). This was an agreed-upon threshold with the security team.
> 5. I created a shared Trivy configuration and `.trivyignore` file template so teams could suppress false positives properly.
>
> **Result:** 100% of pipelines have Trivy scanning within 8 weeks. All CRITICAL CVEs were remediated. The team that was most resistant later said it had saved them during a supply chain incident — they were able to identify affected images in minutes instead of hours.

---

**Q6: Tell me about a time you had to work with a difficult stakeholder.**

> **Situation:** A senior developer consistently bypassed the GitOps deployment process by running `oc apply` directly to production. This created configuration drift and caused two incidents where ArgoCD reverted his manual changes mid-traffic.
>
> **Task:** Align on a process that worked for both parties without creating conflict.
>
> **Action:**
> 1. I started by understanding his reason. The root cause: the GitOps PR review process took 2–4 hours, which was too slow for hotfixes.
> 2. Instead of issuing a policy mandate, I worked with him to design a "hotfix lane" — a fast-track GitOps path that could go from commit to production in < 15 minutes with a single approver.
> 3. I also enabled ArgoCD's `allow_direct_apply` annotation for his service account in break-glass scenarios, but with audit logging and a mandatory incident ticket requirement.
> 4. I presented both options to the team and let them choose — buy-in came from participation in the solution.
>
> **Result:** Manual `oc apply` incidents dropped to zero. The hotfix lane was adopted by 4 other teams. The developer became an advocate for GitOps.

---

### 🟣 JD: "Monitor system performance and troubleshoot issues"

**Q7: Tell me about a time you reduced MTTR (Mean Time to Recovery) for your team.**

> **Situation:** When I joined, the team's MTTR was ~90 minutes. The main bottleneck: on-call engineers spent 30–40 minutes just figuring out which service was broken and on which host.
>
> **Task:** Reduce MTTR by improving observability and runbooks.
>
> **Action:**
> 1. Audited every alert — 60% had no runbook link. I added runbook links to all alerts as a mandatory CI check (alert PR without runbook link = rejected).
> 2. Created a "service health dashboard" in Grafana showing all 12 services in one view — traffic, errors, latency, pod count, and last deployment time.
> 3. Implemented distributed tracing (Jaeger) across the top 5 services — now engineers could trace a slow/failing request across services in one click.
> 4. Wrote and rehearsed incident response playbooks — monthly "fire drills" where we intentionally killed services and timed recovery.
>
> **Result:** MTTR dropped from 90 minutes to 18 minutes over 6 months. Alert-to-runbook coverage reached 100%. The fire drills uncovered 3 gaps in our recovery procedures before they became real incidents.

---

### 🟣 JD: "Implement security best practices for containerized environments"

**Q8: Describe a security incident you handled in a containerized environment. What did you do and what did you learn?**

> **Situation:** Falco detected a cryptocurrency miner running inside a pod in our development namespace. The pod had no resource limits and was maxing out a worker node's CPU.
>
> **Task:** Contain the incident, investigate the compromise, and prevent recurrence.
>
> **Action:**
> 1. Isolated the pod immediately with a deny-all NetworkPolicy.
> 2. Captured forensics: pod logs, process list, network connections before deleting.
> 3. Investigation: the attacker had exploited a publicly-known RCE in an outdated nginx version used as a sidecar. The image had been pulled from Docker Hub without version pinning — `:latest` was 6 months behind.
> 4. Remediation: patched all images, enabled `imagePullPolicy: Always` with SHA pinning (`nginx@sha256:abc123`).
> 5. Prevention: blocked Docker Hub registry in production via OpenShift image policy, mandated Trivy scans blocking on HIGH+ CVEs, added Kyverno policy requiring pinned image digests in production.
>
> **Result:** No further compromise. The incident led to a company-wide "pin your images" initiative. We also implemented Falco across all namespaces with Slack alerting, which has caught 3 additional anomalies in the following year.

---

### 🟣 JD: "Automate infrastructure provisioning and configuration management"

**Q9: Tell me about the most impactful automation you implemented.**

> **Situation:** Onboarding a new application team to OpenShift required 2 days of manual platform-team work: creating namespaces, configuring RBAC, setting up network policies, creating service accounts, configuring pipelines, and documenting everything.
>
> **Task:** Reduce onboarding from 2 days to < 1 hour.
>
> **Action:**
> 1. Mapped every manual step into an Ansible role: `openshift_project`, `openshift_rbac`, `openshift_networking`, `openshift_cicd`.
> 2. Created a Backstage self-service template — developers fill in a form (team name, namespace, language, GitHub URL, approver email). This triggers an Ansible job via AAP.
> 3. The entire provisioning (namespace + quota + RBAC + NetworkPolicy + SA + BuildConfig + ArgoCD Application) completes in ~8 minutes.
> 4. Auto-generated onboarding email includes cluster URL, namespace name, how to get kubeconfig, and link to team documentation.
>
> **Result:** Onboarding time: 2 days → 8 minutes. Platform team capacity freed up by ~15 hours/week. Developer satisfaction survey score for "time to first deployment" went from 2.1/5 to 4.6/5.

---

**Q10: Describe a time you used infrastructure-as-code to solve a problem.**

> **Situation:** The team had 3 OpenShift clusters (dev, staging, prod) that had slowly diverged in configuration over 18 months of manual `oc apply` changes. No one knew exactly what was different between environments, causing "works in staging, fails in prod" incidents weekly.
>
> **Task:** Reconcile all three environments and prevent future drift.
>
> **Action:**
> 1. Used `oc export` to dump current state of all three clusters.
> 2. Used `diff` to identify differences — found 47 meaningful divergences.
> 3. Triaged: which divergences were intentional (prod has more replicas — correct) vs accidental (staging has debug logging enabled in prod config — wrong)?
> 4. Created a Kustomize base + overlays structure in Git. The base = shared config; overlays = environment-specific overrides.
> 5. Deployed ArgoCD to all three clusters pointing at their respective overlay directories. ArgoCD `selfHeal: true` would immediately correct any manual drift.
>
> **Result:** Drift incidents dropped to zero. When a developer asks "why does prod behave differently?" the answer is always in Git. Onboarding new environments takes < 30 minutes.

---

### 🟣 JD: "Provide technical support and guidance to development teams"

**Q11: Tell me about a time you mentored a developer on containerization or DevOps practices.**

> **Situation:** A backend developer was trying to containerize a Java Spring Boot app for the first time. Their Dockerfile was pulling a full JDK image (800MB), running as root, and the app was taking 90 seconds to start — failing readiness probes.
>
> **Task:** Help them build a production-quality container image.
>
> **Action:**
> 1. Pair-programmed the Dockerfile with them rather than just rewriting it — this was a teaching moment.
> 2. Walked through multi-stage builds: compile stage with full JDK, runtime stage with JRE only → image size: 800MB → 120MB.
> 3. Explained why running as root fails on OpenShift (SCC) and showed how to add `RUN useradd -u 1001 appuser && chown -R appuser /app` + `USER 1001`.
> 4. Diagnosed the slow start: Spring Boot's classpath scanning. Fixed with `-Dspring.jmx.enabled=false` and a startup probe (separate from readiness) with 120 seconds initial delay.
>
> **Result:** Image size: 800MB → 120MB. Startup: 90s → 12s. Passed SCC without any special grants. The developer went on to write an internal blog post about it — spread the knowledge to 8 other Java teams.

---

### 🟣 Cross-cutting Behavioral Questions

**Q12: Tell me about a mistake you made in production and what you learned from it.**

> **Situation:** Early in my career, I accidentally ran `oc delete deployment --all -n production` thinking I was in the staging namespace. I had forgotten to switch context.
>
> **Task:** Recover all production deployments as quickly as possible.
>
> **Action:**
> 1. Immediately told my manager — transparency first.
> 2. All deployments were in Git (GitOps), so recovery was: `oc apply -k gitops/overlays/production/` — back in 4 minutes.
> 3. Data was not affected (stateful services had their PVCs intact).
> 4. Wrote a post-mortem focused on systems, not blame.
>
> **Result:** Full restoration in 8 minutes. Zero data loss.
>
> **What I changed immediately:**
> 1. Set `PS1` in my shell to prominently show the current `kubectl` context and namespace in red when `prod` or `production` is in the name.
> 2. Added a namespace confirmation prompt for destructive commands (`kubectl delete`/`oc delete`) via a shell alias.
> 3. Implemented RBAC so the CI/CD service account (which I was using) had more limited permissions than `cluster-admin`.
>
> **Lesson:** Assume humans will make mistakes. Build systems that are hard to break accidentally. GitOps was the hero — without it, recovery would have taken hours.

---

**Q13: Describe a time you had competing priorities and how you handled them.**

> **Situation:** I was mid-way through implementing the cluster autoscaler project (2-week effort) when a critical security CVE (Log4Shell) was disclosed. All production images needed to be patched within 48 hours per our security policy.
>
> **Task:** Balance the ongoing project with the emergency security response.
>
> **Action:**
> 1. Immediately triaged: which services use Log4j? Used Trivy SBOM scan to identify affected images in < 30 minutes.
> 2. Communicated to management: "Log4Shell affects 6 of our 12 services. I'm pausing the autoscaler project to focus on this for 48 hours."
> 3. Prioritized by risk: internet-facing services first (3 services patched in 12 hours), internal services next.
> 4. Updated the CI pipeline to block any new deployment if Trivy found Log4j with the vulnerable version — preventing teams from accidentally deploying more vulnerable images.
> 5. After 48 hours, returned to autoscaler project and completed it within the original week.
>
> **Result:** All 6 services patched within 36 hours. Zero exploitation detected. Autoscaler delivered on time. Management appreciated the transparent communication about reprioritization.

---

**Q14: Where do you see the DevOps/Platform Engineering field heading in the next 3–5 years?**

> **Strong talking points for 2024-2026:**
>
> 1. **Platform Engineering maturity:** Internal Developer Platforms (IDP) — tools like Backstage — will become standard. The platform team's job shifts from "running infra" to "building a product for developers". Golden paths replace DIY.
>
> 2. **GitOps everywhere:** The boundary between CI and CD will blur. ArgoCD/Flux patterns will extend to infrastructure (OpenTofu, Terraform) and policy (Kyverno, OPA). The entire system state — apps, infra, security policies — is in Git.
>
> 3. **eBPF and kernel-level observability:** Tools like Cilium, Hubble, and Tetragon will replace kube-proxy and enable pod-level network observability with near-zero overhead. Falco-like runtime security becomes ubiquitous.
>
> 4. **AI-assisted operations:** LLMs will help with root cause analysis, runbook generation, and alert triaging — but humans will remain in the loop for production decisions. "AI-augmented DevOps", not "AI-replaced DevOps".
>
> 5. **Supply chain security (SLSA):** Image signing, SBOM generation, and provenance attestation will become regulatory requirements (US EO 14028, EU CRA). OpenShift's Tekton Chains addresses this.
>
> 6. **Edge computing:** Kubernetes (MicroShift) and OpenShift at the edge — factories, retail, vehicles. Brings new challenges: low bandwidth, intermittent connectivity, physical security.
>
> **My positioning:** I'm investing in platform engineering skills (Backstage, IDP patterns), GitOps at scale (ACM, ApplicationSets), and supply chain security (cosign, SLSA). These are where the industry is heading.

---

> ✅ **End of Module 7 — You're fully prepared!**
>
> **Recommended final preparation:**
> 1. Record yourself answering Q1, Q3, Q7, Q9, Q12 — watch for filler words and time each answer.
> 2. Do a mock technical interview covering one question from each module.
> 3. Review the [oc CLI Cheat Sheet](../OpenShift.md#oc-cli-cheat-sheet) and [kubectl Cheat Sheet](../Kubernetes.md#kubectl-cheat-sheet) the morning of the interview.
> 4. Prepare 5 questions to ask the interviewer: their tech stack, on-call structure, biggest current challenge, team topology, how success is measured in the first 90 days.