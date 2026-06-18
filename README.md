# w10-gitops

GitOps repository for Week 10 lab — deploys a web application with canary releases, SLO-based alerting, and full observability on Kubernetes.

## Architecture

```
ArgoCD (app-of-apps)
├── namespace (demo)
├── platform-rbac (developer / sre / viewer roles)
├── gatekeeper (OPA Gatekeeper controller)
├── argo-rollouts (controller)
├── monitoring (kube-prometheus-stack)
├── web (Argo Rollout — canary)
├── web-monitoring (ServiceMonitor + PrometheusRule + Grafana dashboard)
├── gatekeeper-policies (ConstraintTemplates + Constraints)
├── payments-tenant (namespace / RBAC / quota / NetworkPolicy)
└── payments-app (team B workload)
```

See [`platform/README.md`](platform/README.md) for the Day A RBAC + admission runbook.

## Tech Stack

| Layer | Tool |
|-------|------|
| GitOps | ArgoCD (app-of-apps pattern) |
| Manifests | Kustomize (base + overlays) |
| Progressive Delivery | Argo Rollouts (canary strategy) |
| Monitoring | Prometheus + Grafana (kube-prometheus-stack) |
| Alerting | PrometheusRule with burn-rate SLO alerts |

## Directory Structure

```
apps/
├── root.yaml                  # Bootstrap app-of-apps
└── children/
    ├── kustomization.yaml
    ├── namespace.yaml           # sync-wave -1
    ├── project.yaml             # sync-wave 0
    ├── rbac.yaml                # sync-wave 1
    ├── gatekeeper.yaml          # sync-wave 1
    ├── argo-rollouts.yaml       # sync-wave 1
    ├── monitoring.yaml          # sync-wave 1
    ├── web.yaml                 # sync-wave 2
    ├── web-monitoring.yaml      # sync-wave 3
    └── gatekeeper-policies.yaml # sync-wave 4

platform/                       # Day A — cluster guardrails
├── rbac/                       # 3 personas (developer / sre / viewer)
└── gatekeeper/                 # ConstraintTemplates + Constraints (dryrun)

workloads/
├── web/
│   ├── base/
│   │   ├── rollout.yaml           # Argo Rollout (canary)
│   │   ├── service.yaml           # Stable service
│   │   ├── service-canary.yaml    # Canary service
│   │   └── analysis-template.yaml # Prometheus-based canary checks
│   └── overlays/lab/
│       └── kustomization.yaml     # Image override for lab env
└── web-monitoring/
    ├── base/
    │   ├── servicemonitor.yaml
    │   ├── servicemonitor-canary.yaml
    │   ├── prometheusrule.yaml        # SLO burn-rate alerts
    │   └── dashboard-configmap.yaml   # Grafana dashboard JSON
    └── overlays/lab/
        └── kustomization.yaml
```

## Canary Strategy

Rollout deploys with progressive traffic shifting:

1. **50% traffic** → canary pods (pause 4 minutes for analysis)
2. **100% traffic** → promote if healthy

Automated analysis runs during the pause:
- **Request volume** — canary must receive ≥ 0.1 rps (gates on real traffic)
- **Error rate** — 5xx rate must stay ≤ 0.2% (auto-abort if > 2%)

Both metrics exclude `/api/health` and `/metrics` endpoints.

## Observability

- **ServiceMonitors** scrape both stable and canary services separately
- **PrometheusRule** fires burn-rate alerts when error budget depletes too fast
- **Grafana dashboard** (provisioned via ConfigMap) shows:
  - Success rate
  - Remaining error budget
  - Burn rate
  - Stable vs canary request rate comparison

## Payments tenant challenge

`payments` is onboarded as a second tenant with its own namespace, Role/RoleBinding, ResourceQuota, LimitRange, and NetworkPolicy.

- Guardrails are inherited because the existing cluster-scoped Gatekeeper constraints match namespaces labeled `app.kubernetes.io/part-of=p2-w10-lab`; no tenant-specific constraints are needed.
- `RoleBinding` grants `payments-dev` permissions only inside `payments`; `ClusterRoleBinding` is intentionally avoided because it would grant access across namespaces.
- NetworkPolicy denies cross-namespace traffic and allows only same-namespace traffic plus DNS. Use a CNI that enforces NetworkPolicy.
- Evidence capture guide for the morning lab, afternoon lab, and payments challenge lives in [`evidence/README.md`](evidence/README.md). Negative test manifests for the payments checks live in [`evidence/payments/`](evidence/payments/).

## Deployment

All deployments are GitOps-driven — push to `main` and ArgoCD syncs automatically.

```bash
# Bootstrap the cluster (one-time)
kubectl apply -f apps/root.yaml

# Deploy a new image version
# Update the image ref in workload overlays (tag + digest)
# e.g. workloads/web/overlays/lab/kustomization.yaml or workloads/payments/overlays/lab/kustomization.yaml
# Commit and push — ArgoCD handles the rest
```

## Configuration

- **Namespace:** `demo`
- **App image:** `vihn/w10-web`
- **Replicas:** 2
- **ArgoCD sync:** automated, prune, self-heal
