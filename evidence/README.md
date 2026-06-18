# Evidence capture guide

Guide này là **bản thao tác ngắn** để tự capture evidence trong repo GitOps.

- **Checklist nộp chính**: xem `../../README.md`
- File này chỉ tóm tắt lại theo hướng thao tác nhanh cho 3 phần:
  1. **Morning / Day A** — RBAC + Gatekeeper
  2. **Afternoon / Day B** — External Secrets + supply chain
  3. **Payments** — tenant onboarding challenge

## Nguyên tắc chung

- Capture trên cluster lab thật sau khi ArgoCD sync xong.
- Mỗi proof nên có đủ **lệnh + kết quả** trong cùng ảnh/log nếu có thể.
- Không commit secret thật, access key, password, private key.
- Nếu cần redact thì chỉ che **value**, không che **resource / namespace / trạng thái / lỗi**.
- Với phần secret rotation: proof phải cho thấy **đổi ở nguồn secret**, không sửa Git manifest giữa lúc demo.

---

## 1. Morning / Day A — RBAC + Gatekeeper

### 1.1. Health

```bash
kubectl get applications -n argocd
kubectl get pods -n gatekeeper-system
```

Cần thấy tối thiểu:

- `platform-rbac`
- `gatekeeper`
- `gatekeeper-policies`

### 1.2. RBAC personas

Repo hiện verify theo **service account persona** trong `demo`:

```bash
kubectl auth can-i create deployments --as=system:serviceaccount:demo:developer-sa -n demo
kubectl auth can-i get secrets --as=system:serviceaccount:demo:developer-sa -n demo
kubectl auth can-i get secrets --as=system:serviceaccount:demo:sre-sa -n demo
kubectl auth can-i create deployments --as=system:serviceaccount:demo:viewer-sa -n demo
```

Kỳ vọng:

- developer create deployment → `yes`
- developer get secret → `no`
- sre get secret → `yes`
- viewer create deployment → `no`

### 1.3. Constraints hiện diện

```bash
kubectl get constrainttemplates
kubectl get k8sdisallowedtags
kubectl get k8srequiredresources
kubectl get k8spspallowedusers
kubectl get k8spsphostnetworkingports
kubectl get k8smaxdeploymentreplicas
```

Nếu cần proof rõ scope policy:

```bash
kubectl get k8smaxdeploymentreplicas max-deployment-replicas -o yaml
```

Cần thấy constraints đang match namespace theo label:

- `app.kubernetes.io/part-of: p2-w10-lab`

### 1.4. Admission reject cases

Capture các case tối thiểu:

1. image dùng `:latest`
2. thiếu `resources.limits`
3. chạy root user / vi phạm non-root rule
4. bật `hostNetwork: true`

Kỳ vọng: admission `deny`, lỗi hiện rõ tên constraint hoặc lý do.

### 1.5. Valid pass + custom policy

- 1 manifest hợp lệ phải pass
- Custom policy hiện tại là `max-deployment-replicas`
  - `replicas: 6` → reject
  - `replicas: 3` → pass

### 1.6. Post-enforce health

```bash
kubectl get applications -n argocd
kubectl get pods -n demo
```

---

## 2. Afternoon / Day B — External Secrets + Supply Chain

### 2.1. Controller health

```bash
kubectl get applications -n argocd
kubectl get pods -n external-secrets
kubectl get pods -n cosign-system
```

Cần thấy:

- External Secrets operator ready
- Sigstore policy-controller ready

### 2.2. ExternalSecret ready

```bash
kubectl get secretstore,externalsecret -n demo
kubectl describe externalsecret web-db-secret -n demo
```

### 2.3. Secret rotation `<60s`, không restart pod

Trước rotate:

```bash
date -u +"%Y-%m-%dT%H:%M:%SZ"
kubectl get secret web-db-secret -n demo -o jsonpath='{.data.host}' | base64 -d && echo
pod=$(kubectl get pod -n demo -l app.kubernetes.io/name=web -o jsonpath='{.items[0].metadata.name}')
kubectl get pod "$pod" -n demo -o jsonpath='{.status.containerStatuses[0].restartCount}' && echo
curl -s http://localhost:30080/api/db/health | jq
```

Rotate ở nguồn secret:

```bash
aws secretsmanager put-secret-value --secret-id <secret-id> --secret-string '<json-da-redact>' --region <region>
```

Theo dõi sync:

```bash
kubectl get secret web-db-secret -n demo -w
```

Sau rotate:

```bash
date -u +"%Y-%m-%dT%H:%M:%SZ"
kubectl get secret web-db-secret -n demo -o jsonpath='{.data.host}' | base64 -d && echo
kubectl get pod "$pod" -n demo -o jsonpath='{.status.containerStatuses[0].restartCount}' && echo
curl -s http://localhost:30080/api/db/health | jq
```

Kỳ vọng:

- secret đổi theo nguồn ngoài
- `<60s`
- restartCount không tăng
- `/api/db/health` vẫn pass
- không cần sửa Git manifest để secret mới vào app

### 2.4. CI / supply chain proof

Cần có proof cho các mục sau:

- secret scan trong CI
- Trivy fail khi có `HIGH` / `CRITICAL`
- Cosign sign + verify pass
- verify log hiện `issuer` / `subject` nếu keyless
- `kubectl get clusterimagepolicy require-signed-w10-web -o yaml`
- unsigned image bị reject
- signed workload vẫn healthy

---

## 3. Payments — tenant onboarding challenge

### 3.1. Tenant sync + labels

```bash
kubectl get applications -n argocd
kubectl get ns payments --show-labels
```

Cần thấy:

- `payments-tenant`
- `payments-app`
- label kế thừa guardrail:
  - `app.kubernetes.io/part-of=p2-w10-lab`
- label bật Sigstore scope:
  - `policy.sigstore.dev/include=true`

### 3.2. RBAC least-privilege

```bash
kubectl auth can-i create deployments -n payments --as=payments-dev
kubectl auth can-i create deployments -n demo --as=payments-dev
kubectl auth can-i get secrets -n payments --as=payments-dev
kubectl auth can-i create rolebindings -n payments --as=payments-dev
```

Kỳ vọng:

- trong `payments` quyền đúng nhu cầu
- ra ngoài `payments` bị chặn
- không được đọc secret
- không được tạo rolebinding

### 3.3. Resource isolation

```bash
kubectl apply -f evidence/payments/pod-over-quota.yaml
kubectl apply -f evidence/payments/pod-no-limits.yaml
kubectl -n payments get pod no-limits-gets-defaults -o jsonpath='{.spec.containers[0].resources}'
```

Kỳ vọng:

- over-quota bị reject
- pod thiếu limits được inject default

### 3.4. Network isolation

```bash
kubectl -n payments run curl --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- sh
```

Trong shell:

```bash
curl -m 3 http://web.demo.svc.cluster.local/api/health
curl -m 3 http://payments-api.payments.svc.cluster.local/api/health
```

Kỳ vọng:

- gọi `demo` bị block/timeout
- gọi `payments-api` thành công

### 3.5. Inherited guardrails + signed workload

```bash
kubectl apply -f evidence/payments/bad-latest.yaml
kubectl -n payments get deploy payments-api -o wide
kubectl -n payments describe deploy payments-api
```

Kỳ vọng:

- `bad-latest.yaml` bị reject bởi guardrail sẵn có
- workload signed image trong `payments` vẫn chạy được

---

## 4. Nộp tối thiểu

Nếu cần nộp nhanh, ít nhất phải có:

### Day A
- health
- 4 RBAC checks
- constraint proof
- 4 reject + 1 pass
- custom `max-deployment-replicas` fail + pass

### Day B
- ExternalSecret ready
- before/after rotate có timestamp
- `<60s` + no restart + health pass
- secret scan proof
- Trivy fail
- Cosign sign/verify pass
- unsigned reject + signed pass

### Payments
- tenant/app healthy
- namespace labels
- 4 `can-i`
- quota reject
- LimitRange default
- cross-namespace blocked
- inherited guardrail reject
- signed workload admitted