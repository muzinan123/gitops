# GitOps Main — Production-Grade Kubernetes Full-Stack CI/CD Platform

> Not a collection of YAML files. A complete GitOps engineering practice covering code commit, image build, progressive delivery, and automated rollback.

---

## What Problem Does This Solve

In Kubernetes production environments, there is a massive engineering gap between "able to deploy" and "able to deploy safely and in a controlled manner":

- Post-merge builds and deployments require manual triggering — **how do you achieve full end-to-end automation from git push, with zero human intervention?**
- Standard Kubernetes Deployments are all-or-nothing replacements — **how do you implement progressive traffic shifting to minimize the blast radius of every release?**
- New versions are monitored manually after going live — **how do you let the system automatically evaluate new version health using Prometheus metrics and halt the rollout if thresholds aren't met?**
- Multi-environment configs (dev/test/prod) drift over time when managed separately — **how do you use Git as the single source of truth and keep cluster state always in sync with the repository?**

This project answers each of these with a combination of Argo CD + Argo Rollouts + Tekton + Helm.

✅ GitHub Webhook → Tekton Pipeline — code commits automatically trigger image builds and pushes
✅ Argo CD auto-sync — Git is cluster state, with drift auto-remediation via `selfHeal`
✅ Argo Rollouts progressive delivery — both Canary and Blue/Green strategies supported
✅ AnalysisTemplate + Prometheus metric integration — new version health validation fully automated
✅ Helm multi-environment values separation — dev / test / prod configs managed independently

---

## Three Core Engineering Decisions

### Decision 1: Git as the Single Source of Truth — Manual Deployments Eliminated

**Problem**: Manual `kubectl apply` is unauditable and untraceable. In collaborative environments, cluster state easily drifts from the repository, making root cause analysis difficult.

**Decision**: Use **Argo CD** as the CD engine, creating a hard binding between Git repository state and Kubernetes cluster state.

Key configuration:
```yaml
syncPolicy:
  automated:
    prune: true      # Resources deleted from Git are automatically removed from the cluster
    selfHeal: true   # Manual cluster changes are automatically reverted to Git-defined state
```

- **prune**: Prevents "zombie resources" from lingering — deleted in Git means deleted in cluster
- **selfHeal**: Prevents manual operations from polluting cluster state — any drift is automatically corrected
- **App of Apps pattern**: `application.yaml` acts as the root application, centrally managing sync policies for all child applications

**Outcome**: Cluster state is entirely determined by Git history. Every change has a commit record. Rollback is `git revert` — no kubectl commands to memorize.

---

### Decision 2: Progressive Delivery with Metric-Driven Automated Validation Gates

**Problem**: Standard Kubernetes Deployment rolling updates are full replacements — a bad release affects 100% of traffic. Manual observation windows are short and low-frequency anomalies are easily missed.

**Decision**: Replace standard Deployments with **Argo Rollouts**, combined with **AnalysisTemplate** for metric-driven automated validation.

**Canary Release Flow:**
```
New version deployed
      │
      ▼
Route 10% traffic → Observation window begins
      │
      ▼
AnalysisTemplate queries Prometheus metrics
(error rate / latency / success rate)
      │
      ├── Metrics healthy → Promote (10% → 20% → 50% → 100%)
      │
      └── Metrics degraded → Auto-pause / rollback, zero human intervention
```

**Blue/Green Release Flow:**
```
Green (current production traffic)
Blue  (new version, zero traffic warm-up)
      │
      ▼
Validation passed → Ingress switches all traffic to Blue instantly
Issue detected    → Immediately revert to Green, recovery in seconds
```

**Outcome**: Blast radius of a new release compressed from 100% to 10%. Prometheus metric anomalies trigger automatic braking — no on-call engineer required to watch the dashboard.

---

### Decision 3: Event-Driven CI Pipeline — Webhook-Triggered Automated Builds

**Problem**: Manually triggering CI builds is an efficiency bottleneck and a source of human error. Build processes need to be observable and replayable.

**Decision**: Build an event-driven CI pipeline with **Tekton**, automatically triggered by GitHub Webhooks.

```
Developer git push
      │
      ▼
GitHub Webhook → Tekton EventListener
      │
      ▼
TriggerTemplate creates PipelineRun
      │
      ▼
┌─────────────────────────────┐
│       Tekton Pipeline       │
│  fetch-repository           │  ← git-clone Task
│       ↓                     │
│  build-and-push             │  ← docker-build Task (kaniko)
└─────────────────────────────┘
      │
      ▼
New image pushed to Registry
      │
      ▼
Argo CD detects image tag change → triggers sync
```

- **kaniko**: Builds images securely inside ordinary Pods without a Docker daemon — native to Kubernetes environments
- **Tekton Dashboard**: Web UI providing full visibility into every build's logs, status, and parameters
- **Least-privilege RBAC**: ServiceAccount granted only the minimum permissions required by Tekton

**Outcome**: From code commit to new image ready — fully automated, zero human operations. Every build is fully observable with permanent history.

---

## System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                  Complete GitOps Pipeline                    │
│                                                             │
│  Developer                                                  │
│     │ git push                                              │
│     ▼                                                       │
│  GitHub ──Webhook──► Tekton EventListener                   │
│                            │                                │
│                     Tekton Pipeline                         │
│                     (clone → build → push)                  │
│                            │                                │
│                       Registry                              │
│                            │                                │
│                       Argo CD ◄── Git Repo (Helm Charts)    │
│                            │                                │
│                    Argo Rollouts                            │
│               ┌────────────┴────────────┐                   │
│         Canary Delivery           Blue/Green Delivery        │
│               │                         │                   │
│        AnalysisTemplate          Ingress Traffic Switch      │
│        (Prometheus Metrics)                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology | Responsibility |
| :--- | :--- | :--- |
| CI Engine | Tekton + kaniko | Image build and push |
| CD Engine | Argo CD | Git state sync to cluster |
| Delivery Strategy | Argo Rollouts | Canary / Blue/Green progressive delivery |
| Delivery Validation | AnalysisTemplate + Prometheus | Metric-driven automated health validation |
| App Packaging | Helm | Multi-environment configuration management |
| Traffic Ingress | Nginx Ingress Controller | External traffic routing |
| Monitoring | Prometheus + ServiceMonitor | Metrics collection and analysis |
| Trigger Mechanism | Tekton Triggers + GitHub Webhook | Event-driven CI triggering |

---

## Repository Structure

```
gitops-main/
├── argocd/                        # Argo CD core configuration
│   ├── *_deployment.yaml          # Blue / Green / Canary / Prod deployment definitions
│   ├── *_ingress.yaml             # Per-environment Ingress resources
│   ├── canary-rollout.yaml        # Canary progressive delivery strategy
│   ├── rollout-with-analysis.yaml # Rollout config with metric-based validation
│   ├── analysis-success.yaml      # AnalysisTemplate (Prometheus metric definitions)
│   ├── service-monitor.yaml       # Prometheus ServiceMonitor
│   └── ingress-nginx.yaml         # Nginx Ingress Controller install manifest
├── tekton/
│   ├── install/                   # Tekton core component install manifests
│   ├── task/                      # git-clone / docker-build Task definitions
│   ├── pipeline/                  # Pipeline orchestration (fetch → build → push)
│   ├── trigger/                   # EventListener + TriggerTemplate
│   ├── ingress/                   # Tekton Dashboard & Webhook ingress
│   └── other/                     # ServiceAccount + RBAC configuration
├── helm/
│   ├── templates/                 # frontend.yaml / ingress.yaml templates
│   └── env/
│       ├── dev/values.yaml        # Dev environment (replicas: 1)
│       ├── test/values.yaml       # Test environment config
│       └── prod/values.yaml       # Prod environment (replicas: 3, strict resource quotas)
└── gitops-workflow/
    ├── application.yaml           # Argo CD root application (App of Apps)
    ├── repository.yaml            # Git repository connection config
    ├── git-pull-secret.yaml       # Git access credentials
    └── docker-pull-secret.yaml    # Image registry pull secret
```

---

## Roadmap

🚧 Kyverno policy engine integration with OPA compliance enforcement | Multi-cluster Argo CD management | SOPS-encrypted Secrets management | Grafana Dashboard for release state visualization
