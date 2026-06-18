# Evidence capture guide

Guide này dùng để **tự capture evidence** cho 3 phần:

1. **Morning** — RBAC + Gatekeeper
2. **Afternoon** — External Secrets + Supply Chain
3. **Payments** — tenant onboarding challenge

Nguyên tắc chung:

- Capture trên **cluster thật của lab** sau khi ArgoCD đã sync xong.
- Mỗi mục nên có ít nhất **1 ảnh chụp màn hình terminal** hoặc **1 đoạn log/output rõ ràng**.
- Nếu một mục có cả **lệnh** và **kết quả hiển thị** trong cùng ảnh thì tốt nhất.
- Không commit secret thật, access key, password, private key.
- Nếu cần redact, chỉ che **giá trị secret**, không che tên resource / namespace / trạng thái / lỗi.

---

## 1. Morning — RBAC + Admission Policy

Mục tiêu chấm:

- 3 role hoạt động đúng
- Gatekeeper controller + constraints đã lên
- 4 manifest vi phạm bị reject
- 1 manifest hợp lệ pass
- 1 custom policy của bạn hoạt động đúng
- Platform vẫn Synced/Healthy sau khi enforce

### 1.1. ArgoCD healthy

Capture màn hình hoặc output của:

```bash
kubectl get applications -n argocd
```

Cần nhìn thấy các app chính ở trạng thái tốt, tối thiểu:

- `rbac`
- `gatekeeper`
- `gatekeeper-policies`
- app workload chính của platform

### 1.2. RBAC — 4 lệnh bắt buộc

Capture 1 ảnh hoặc 1 đoạn terminal có đủ 4 lệnh sau và kết quả của chúng:

```bash
kubectl auth can-i create deployments -n demo --as alice
kubectl auth can-i create deployments -n kube-system --as alice
kubectl auth can-i get pods -A --as bob
kubectl auth can-i delete nodes --as carol
```

Kỳ vọng:

- alice trong `demo` → `yes`
- alice ngoài `demo` → `no`
- bob xem pod toàn cụm → `yes`
- carol xóa node → `no`

### 1.3. Gatekeeper controller + constraints đã cài

Capture output của:

```bash
kubectl get pods -n gatekeeper-system
kubectl get constrainttemplates
kubectl get k8sdisallowedtags
kubectl get k8srequiredresources
kubectl get k8spspallowedusers
kubectl get k8spsphostnetworkingports
kubectl get k8smaxdeploymentreplicas
```

Không nhất thiết phải dùng đúng tên custom constraint ở trên nếu bạn chọn custom policy khác, nhưng cần chứng minh:

- controller đang chạy
- các constraint cần dùng đã tồn tại
- custom policy của bạn đã được apply

### 1.4. 4 manifest vi phạm phải bị reject

Với từng rule, capture lệnh `kubectl apply -f ...` và lỗi reject trả về từ API server:

1. image dùng `:latest`
2. thiếu `resources.limits`
3. `runAsUser: 0`
4. `hostNetwork: true`

Kỳ vọng mỗi case:

- request bị **deny/reject**
- lỗi hiển thị rõ tên policy/constraint hoặc lý do vi phạm

### 1.5. 1 manifest hợp lệ phải pass

Capture:

```bash
kubectl apply -f <manifest-hop-le>.yaml
kubectl get pod -n <namespace>
```

Kỳ vọng:

- apply thành công
- pod/workload tạo được
- manifest hợp lệ có image pinned, limits, non-root, không `hostNetwork: true`

### 1.6. Custom policy — fail + pass

Bạn phải có evidence cho **cả 2 hướng**:

- 1 manifest **vi phạm** custom policy → reject
- 1 manifest **hợp lệ** → pass

Ví dụ nếu custom policy là `replicas > 5`:

- deployment `replicas: 6` → reject
- deployment `replicas: 3` → pass

Ví dụ nếu custom policy là `owner label`:

- thiếu label `owner` → reject
- có label `owner` → pass

### 1.7. Platform không tự làm gãy chính nó

Sau khi bật enforce, capture lại:

```bash
kubectl get applications -n argocd
kubectl get pods -n demo
```

Kỳ vọng:

- ArgoCD app vẫn `Synced` / `Healthy`
- workload chính vẫn chạy bình thường

---

## 2. Afternoon — Secrets Rotation + Supply Chain

Mục tiêu chấm:

- ESO sync secret từ nguồn ngoài
- Secret đổi trong `< 60s`
- Pod không restart khi rotate
- `/api/db/health` vẫn tốt sau rotate
- Trivy fail đúng khi có CVE nặng
- Cosign sign + verify thành công
- Unsigned image bị reject
- Signed image pass admission

### 2.1. ESO / Sigstore apps healthy

Capture:

```bash
kubectl get applications -n argocd
kubectl get pods -n external-secrets
kubectl get pods -n cosign-system
```

Kỳ vọng:

- app/operator liên quan đã chạy
- không cần chụp hết mọi pod, chỉ cần đủ chứng minh controller ready

### 2.2. ExternalSecret ready

Capture:

```bash
kubectl get secretstore,externalsecret -n demo
kubectl describe externalsecret web-db-secret -n demo
```

Kỳ vọng:

- `SecretStore` tồn tại
- `ExternalSecret/web-db-secret` ở trạng thái Ready / SecretSynced

### 2.3. Chứng minh rotate không restart

Capture theo thứ tự sau.

#### Bước A — trước khi rotate

```bash
kubectl get secret web-db-secret -n demo -o jsonpath='{.data.host}' | base64 -d && echo
pod=$(kubectl get pod -n demo -l app.kubernetes.io/name=web -o jsonpath='{.items[0].metadata.name}')
kubectl get pod "$pod" -n demo -o jsonpath='{.status.containerStatuses[0].restartCount}' && echo
curl -s http://localhost:30080/api/db/health | jq
```

Capture để thấy:

- secret hiện tại đang tồn tại
- tên pod / restartCount hiện tại
- `/api/db/health` đang pass

#### Bước B — rotate nguồn secret

Capture lệnh update nguồn secret (AWS Secrets Manager hoặc nguồn tương đương bạn dùng trong lab).

Nếu output có giá trị nhạy cảm thì redact **value**, không redact tên secret/resource.

#### Bước C — sau khi rotate

Capture:

```bash
kubectl get secret web-db-secret -n demo -w
```

sau đó capture tiếp:

```bash
kubectl get secret web-db-secret -n demo -o jsonpath='{.data.host}' | base64 -d && echo
kubectl get pod "$pod" -n demo -o jsonpath='{.status.containerStatuses[0].restartCount}' && echo
curl -s http://localhost:30080/api/db/health | jq
```

Kỳ vọng:

- secret trong cluster đã đổi theo nguồn
- restartCount **không tăng**
- `/api/db/health` vẫn pass

### 2.4. Repo / pipeline không lộ secret thật

Chỉ cần 1 bằng chứng rõ ràng là pipeline có secret scanning hoặc repo không chứa secret thật.

Ví dụ tốt nhất:

- ảnh/log step `Scan repo for secrets` trong GitHub Actions pass

Nếu muốn bổ sung thủ công, có thể capture thêm việc grep/audit repo, nhưng không bắt buộc nếu CI proof đã đủ rõ.

### 2.5. Trivy fail khi image có CVE HIGH/CRITICAL

Capture log GitHub Actions hoặc CI tương đương ở step Trivy khi cố tình dùng image/dependency có CVE nặng.

Kỳ vọng:

- step Trivy đỏ
- pipeline fail tại scan
- nhìn thấy `HIGH` / `CRITICAL` hoặc exit code fail

### 2.6. Cosign sign + verify thành công

Capture log của:

- step sign image
- step verify image signature

Kỳ vọng:

- image digest được sign
- verify thành công
- nếu dùng keyless, nên nhìn thấy identity / issuer trong log verify

### 2.7. ClusterImagePolicy hiện diện

Capture:

```bash
kubectl get clusterimagepolicy require-signed-w10-web -o yaml
```

Kỳ vọng:

- policy tồn tại
- mode / image glob / authority hiển thị rõ

### 2.8. Unsigned image bị reject

Capture:

```bash
kubectl -n demo run unsigned-w10-web-test --image=vihn/w10-web:unsigned-test --restart=Never
```

Kỳ vọng:

- admission reject
- lỗi hiển thị do signature/policy không hợp lệ hoặc thiếu signature

### 2.9. Signed image pass

Capture:

```bash
kubectl argo rollouts get rollout web -n demo
kubectl get pods -n demo
```

Kỳ vọng:

- rollout/workload signed image vẫn healthy
- pod chạy bình thường

---

## 3. Payments — tenant onboarding challenge

Mục tiêu chấm:

- `payments` có namespace riêng
- RBAC least-privilege đúng
- quota chặn vượt mức
- LimitRange cấp default
- NetworkPolicy chặn gọi chéo namespace
- guardrail cũ tự áp cho team mới
- app team B lên qua GitOps và chạy được

### 3.1. Tenant + app đã sync

Capture:

```bash
kubectl get applications -n argocd
kubectl get ns payments --show-labels
```

Kỳ vọng:

- `payments-tenant` và `payments-app` xuất hiện, healthy nếu có thể
- namespace `payments` tồn tại
- có label để inherited guardrail match

### 3.2. RBAC least-privilege

Capture một terminal có đủ 4 lệnh:

```bash
kubectl auth can-i create deployments -n payments --as=payments-dev
kubectl auth can-i create deployments -n demo --as=payments-dev
kubectl auth can-i get secrets -n payments --as=payments-dev
kubectl auth can-i create rolebindings -n payments --as=payments-dev
```

Kỳ vọng:

- create deployment trong `payments` → `yes`
- create deployment trong `demo` → `no`
- đọc secret trong `payments` → `no`
- tạo rolebinding trong `payments` → `no`

### 3.3. Quota reject pod over budget

Capture:

```bash
kubectl apply -f evidence/payments/pod-over-quota.yaml
```

Kỳ vọng:

- bị từ chối với lỗi `exceeded quota` hoặc tương đương

### 3.4. LimitRange cấp default cho pod thiếu limits

Capture:

```bash
kubectl apply -f evidence/payments/pod-no-limits.yaml
kubectl -n payments get pod no-limits-gets-defaults -o jsonpath='{.spec.containers[0].resources}'
kubectl -n payments delete pod no-limits-gets-defaults
```

Kỳ vọng:

- pod tạo được
- output resources cho thấy default request/limit đã được inject

### 3.5. NetworkPolicy chặn gọi chéo namespace

Capture shell trong pod test:

```bash
kubectl -n payments run curl --image=curlimages/curl:8.10.1 --rm -it --restart=Never -- sh
```

Trong shell đó, capture 2 lệnh:

```bash
curl -m 3 http://web.demo.svc.cluster.local/api/health
curl -m 3 http://payments-api.payments.svc.cluster.local/api/health
```

Kỳ vọng:

- gọi sang `demo` → timeout / blocked
- gọi service cùng namespace `payments-api` → thành công

Lưu ý: proof này chỉ có giá trị nếu CNI đang enforce NetworkPolicy.

### 3.6. Guardrail cũ tự áp cho team mới

Capture:

```bash
kubectl apply -f evidence/payments/bad-latest.yaml
```

Kỳ vọng:

- bị reject bởi constraint hiện có, ví dụ `disallow-latest-tag`
- điều quan trọng là **không cần viết policy mới riêng cho payments**

### 3.7. App team B chạy được qua GitOps

Capture:

```bash
kubectl -n payments get deploy payments-api
kubectl -n payments get pods
kubectl -n payments get svc payments-api
```

Kỳ vọng:

- deployment tồn tại
- pod running/ready
- service tồn tại

### 3.8. Signed image admission trong `payments`

Nếu namespace `payments` đã bật Sigstore include, capture thêm proof rằng app hợp lệ vẫn admit được.

Ví dụ:

```bash
kubectl -n payments get deploy payments-api -o wide
kubectl -n payments describe deploy payments-api
```

Kỳ vọng:

- workload signed image đang chạy bình thường
- nếu có test unsigned matching image thì càng tốt, nhưng không bắt buộc nếu đã có proof reject ở `demo`

---

## 4. Checklist nộp nhanh

Nếu cần tối giản để nộp/chấm nhanh, ít nhất hãy có đủ các nhóm proof sau:

### Morning
- ArgoCD healthy
- 4 lệnh `can-i`
- 4 reject + 1 pass cho Gatekeeper
- custom policy reject + pass

### Afternoon
- ExternalSecret ready
- before/after rotate + restartCount không đổi + `/api/db/health` pass
- Trivy fail
- Cosign sign/verify pass
- unsigned reject + signed pass

### Payments
- 4 lệnh `can-i`
- over-quota reject
- LimitRange default injected
- cross-namespace blocked / same-namespace pass
- inherited guardrail reject
- payments app healthy
