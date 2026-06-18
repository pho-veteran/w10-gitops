# Platform: Day A (RBAC + Admission)

Các guardrail ở cluster-level cho `w10-lab`, deploy qua ArgoCD app-of-apps:

| App (ArgoCD) | Path / Source | Wave | Mục đích |
|--------------|---------------|------|----------|
| `platform-rbac` | `platform/rbac` | 1 | 3 RBAC persona (developer / sre / viewer) trong ns `demo` |
| `gatekeeper` | Helm `gatekeeper` 3.21.0 | 1 | OPA Gatekeeper controller trong ns `gatekeeper-system` |
| `gatekeeper-policies` | `platform/gatekeeper` | 4 | ConstraintTemplate + Constraint áp vào các namespace mang label `app.kubernetes.io/part-of: p2-w10-lab` |

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

## 2. Gatekeeper constraints

| Constraint | Kind | Ràng buộc |
|------------|------|-----------|
| `disallow-latest-tag` | `K8sDisallowedTags` | chặn image dùng tag `latest` |
| `require-limits` | `K8sRequiredResources` | bắt buộc `resources.limits` |
| `disallow-root-user` | `K8sPSPAllowedUsers` | chặn chạy root user |
| `disallow-hostnetwork` | `K8sPSPHostNetworkingPorts` | chặn `hostNetwork: true` |
| `max-deployment-replicas` | `K8sMaxDeploymentReplicas` | Deployment không vượt quá `5` replicas |

Các constraint hiện áp vào các namespace mang label `app.kubernetes.io/part-of: p2-w10-lab`, nên không chỉ `demo` mà cả `payments` cũng tự kế thừa cùng bộ guardrail.

`enforcementAction` hiện ở trạng thái enforce/deny theo manifest live, nên manifest vi phạm sẽ bị chặn ngay ở admission.

## 3. Verify nhanh các guardrail hiện tại

Có thể kiểm nhanh bộ policy live bằng các lệnh sau:

```bash
kubectl get k8sdisallowedtags
kubectl get k8srequiredresources
kubectl get k8spspallowedusers
kubectl get k8spsphostnetworkingports
kubectl get k8smaxdeploymentreplicas
kubectl get k8smaxdeploymentreplicas max-deployment-replicas -o yaml
```

Kỳ vọng:

- tất cả constraint đã tồn tại
- `max-deployment-replicas` có `namespaceSelector` match label `app.kubernetes.io/part-of: p2-w10-lab`
- manifest vi phạm sẽ bị chặn ngay tại admission

Ví dụ kiểm chứng custom policy:

```bash
kubectl apply -f <deployment-replicas-6>.yaml   # reject
kubectl apply -f <deployment-replicas-3>.yaml   # pass
```

## 4. Ý nghĩa với tenant mới như `payments`

Vì policy match theo label namespace chứ không match cứng riêng `demo`, nên namespace mới chỉ cần mang label `app.kubernetes.io/part-of: p2-w10-lab` là tự kế thừa cùng bộ Gatekeeper guardrail. Đây là lý do `payments` bị cùng rule set áp vào mà không cần viết thêm constraint mới.

Để chứng minh end-to-end, xem checklist nộp ở `../../README.md` và guide thao tác ngắn ở `../evidence/README.md`.

> Lưu ý: `Role` + `RoleBinding` giữ quyền ở mức namespace; tránh dùng `ClusterRoleBinding` cho tenant app nếu không thật sự cần, vì rất dễ làm phạm vi quyền rộng quá mức cô lập mong muốn.
