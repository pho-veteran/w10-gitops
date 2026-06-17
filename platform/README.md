# Platform: Day A (RBAC + Admission)

Các guardrail ở cluster-level cho `w10-lab`, deploy qua ArgoCD app-of-apps:

| App (ArgoCD) | Path / Source | Wave | Mục đích |
|--------------|---------------|------|----------|
| `platform-rbac` | `platform/rbac` | 1 | 3 RBAC persona (developer / sre / viewer) trong ns `demo` |
| `gatekeeper` | Helm `gatekeeper` 3.21.0 | 1 | OPA Gatekeeper controller trong ns `gatekeeper-system` |
| `gatekeeper-policies` | `platform/gatekeeper` | 4 | 4 ConstraintTemplate + Constraint giới hạn trong ns `demo` |

```
platform/
├── rbac/
│   ├── serviceaccounts.yaml     # developer-sa / sre-sa / viewer-sa
│   ├── role-developer.yaml      # CRUD workload, KHÔNG secrets/exec/delete-ns
│   ├── role-sre.yaml            # toàn quyền trong ns, gồm secrets + exec + ns RBAC
│   └── role-viewer.yaml         # bind ClusterRole "view" có sẵn
└── gatekeeper/
    ├── templates/               # 4 ConstraintTemplate (Rego), sync-wave 1
    └── constraints/             # 4 Constraint (dryrun), sync-wave 2
```

## 1. RBAC: 3 persona

| Role | Được phép | Bị chặn |
|------|-----------|---------|
| `developer` | CRUD deployments/rollouts/services/configmaps/pods, đọc `pods/log` | `secrets`, `pods/exec`, `namespaces` |
| `sre` | toàn quyền trong ns gồm `secrets`, `pods/exec`, delete, RBAC ns-scope | cluster-scope (nodes/PV), `escalate`/`bind` |
| `viewer` | read-only qua ClusterRole `view` | mọi thao tác ghi, `secrets`, `exec` |

### Verify bằng impersonation (`kubectl auth can-i --as=...`)

```bash
# developer: CRUD được deployments, bị chặn secrets/exec/delete-ns
kubectl auth can-i create deployments --as=system:serviceaccount:demo:developer-sa -n demo   # yes
kubectl auth can-i get    secrets     --as=system:serviceaccount:demo:developer-sa -n demo   # no
kubectl auth can-i create pods/exec   --as=system:serviceaccount:demo:developer-sa -n demo   # no
kubectl auth can-i delete namespaces  --as=system:serviceaccount:demo:developer-sa           # no

# sre: toàn quyền trong ns, nhưng không có cluster-scope
kubectl auth can-i get    secrets     --as=system:serviceaccount:demo:sre-sa -n demo         # yes
kubectl auth can-i create pods/exec   --as=system:serviceaccount:demo:sre-sa -n demo         # yes
kubectl auth can-i delete deployments --as=system:serviceaccount:demo:sre-sa -n demo         # yes
kubectl auth can-i get    nodes       --as=system:serviceaccount:demo:sre-sa                 # no

# viewer: read-only, không đọc secrets
kubectl auth can-i list   pods        --as=system:serviceaccount:demo:viewer-sa -n demo      # yes
kubectl auth can-i create deployments --as=system:serviceaccount:demo:viewer-sa -n demo      # no
kubectl auth can-i get    secrets     --as=system:serviceaccount:demo:viewer-sa -n demo      # no

# xem toàn bộ ma trận quyền của một persona
kubectl auth can-i --list --as=system:serviceaccount:demo:developer-sa -n demo
```

## 2. Gatekeeper: 4 constraint (khởi đầu ở dryrun)

| Constraint | Kind | Ràng buộc |
|------------|------|-----------|
| `deny-privileged` | `K8sBlockPrivileged` | không cho container `privileged: true` |
| `require-requests-limits` | `K8sRequiredResources` | bắt buộc requests và limits cho cpu+memory |
| `require-run-as-nonroot` | `K8sRequireRunAsNonRoot` | `runAsNonRoot: true` (ở pod hoặc container) |
| `require-min-2-replicas` | `K8sMinReplicas` | Rollout `replicas >= 2` |

Tất cả khởi đầu ở `enforcementAction: dryrun`: audit trước, không bao giờ deny ngay ngày đầu.

### Audit workload `web` đang chạy (dryrun)

Trước khi remediate, Rollout `web` (chưa có securityContext, chưa có resources) vi phạm
3 trong 4 constraint. Vòng audit nền của Gatekeeper (~60s) ghi lại:

```bash
kubectl get k8srequiredresources   require-requests-limits -o jsonpath='{.status.totalViolations}'  # 1
kubectl get k8srequirerunasnonroot require-run-as-nonroot  -o jsonpath='{.status.totalViolations}'  # 1
kubectl get k8sminreplicas         require-min-2-replicas  -o jsonpath='{.status.totalViolations}'  # 0 (đã có 2 replicas)
kubectl get k8sblockprivileged     deny-privileged         -o jsonpath='{.status.totalViolations}'  # 0

kubectl describe k8srequiredresources require-requests-limits   # xem thông điệp vi phạm theo từng pod
```

> Rollout `web` vốn đã chạy `replicas: 2` nên `require-min-2-replicas` sạch ngay từ đầu;
> hai vi phạm còn lại được xử lý ở phần remediate bên dưới.

## 3. Remediate qua GitOps

`workloads/web/base/rollout.yaml` đã được hardening: securityContext (runAsNonRoot, drop
caps, readOnlyRootFilesystem kèm `/tmp` emptyDir) cùng requests/limits cho cpu/memory.
Push lên, ArgoCD self-heal, rồi audit về 0:

```bash
kubectl get k8srequiredresources   require-requests-limits -o jsonpath='{.status.totalViolations}'  # 0
kubectl get k8srequirerunasnonroot require-run-as-nonroot  -o jsonpath='{.status.totalViolations}'  # 0
```

## 4. Chuyển sang Enforce (deny): chỉ khi audit đã sạch

Khi mọi constraint báo 0 vi phạm, đổi `dryrun` thành `deny` rồi commit:

```bash
# từ thư mục gốc repo
sed -i 's/enforcementAction: dryrun/enforcementAction: deny/' platform/gatekeeper/constraints/*.yaml
git commit -am "feat(day-a): enforce Gatekeeper constraints (dryrun -> deny)"
git push origin main
```

Kiểm chứng enforce từ chối manifest xấu ngay tại admission:

```bash
kubectl run bad --image=nginx --privileged -n demo
# Error from server (Forbidden): admission webhook "validation.gatekeeper.sh" denied the request:
# [deny-privileged] Container bad is running privileged, which is prohibited!
```

> ⚠️ Đừng chuyển sang `deny` khi audit chưa sạch: workload đang chạy thì vẫn sống, nhưng
> mọi lần ArgoCD self-heal, rolling update hay scale một workload chưa đạt chuẩn đều bị
> từ chối, khiến service kẹt không cập nhật được.
