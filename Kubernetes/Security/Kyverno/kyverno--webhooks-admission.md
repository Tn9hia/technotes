# Kyverno — Webhooks & Admission Failure Modes
Tier: 2
Parent: [[Kyverno]]
Related: [[kyverno--architecture-controllers]], [[kyverno--troubleshooting-runbook]]
Tags: #kyverno #webhook #admission #availability

## What it does

Kyverno Admission Controller đăng ký (và tự quản lý qua Webhook Controller con) các `ValidatingWebhookConfiguration`/`MutatingWebhookConfiguration` trỏ về Service `kyverno-svc.kyverno.svc`. Webhook Controller **tự động cập nhật** danh sách resource được match trong webhook config dựa trên policy đang cài (chỉ đăng ký nhận request cho resource kind thực sự có policy match — tối ưu để apiserver không gọi webhook cho mọi request).

## Why it exists

Đây là cơ chế bắt buộc của Kubernetes dynamic admission control — không có cách nào khác để 1 controller "chen vào" giữa request và etcd để validate/mutate ngoài admission webhook. Việc Webhook Controller tự động scope webhook theo policy hiện có (thay vì đăng ký nhận tất cả) giảm overhead không cần thiết lên apiserver.

## How it works (flow/diagram)

```
kubectl apply Pod ──▶ apiserver ──▶ [MutatingWebhook: kyverno] ──▶ [ValidatingWebhook: kyverno] ──▶ etcd
                                          │  timeout: 10s default (1-30s range)         │
                                          │  failurePolicy: Fail (default)              │
                                          ▼                                              ▼
                                   nếu Kyverno pod không phản hồi kịp trong timeoutSeconds:
                                   - failurePolicy=Fail   → apiserver TỪ CHỐI request (fail-closed)
                                   - failurePolicy=Ignore → apiserver CHO QUA request (fail-open)
```

## Config gotchas

- **`failurePolicy` mặc định là `Fail`** — không phải `Ignore`. Đây là điểm khác biệt hay bị nhớ nhầm từ kinh nghiệm dùng policy engine khác. Với `Fail`, nếu Kyverno pod down toàn bộ và policy match rộng (không loại trừ `kube-system`/namespace của Kyverno), apiserver có thể **từ chối mọi request**, kể cả request tạo lại pod Kyverno để tự phục hồi → cluster tự khoá mình.
- `timeoutSeconds` default 10s, giới hạn cứng của Kubernetes là 1–30s — không thể set cao hơn để "chờ" Kyverno xử lý policy phức tạp; nếu policy có `apiCall` external hoặc context nặng, phải tối ưu tốc độ policy chứ không thể kéo dài timeout vô hạn.
- Trên **AKS**: AKS có sẵn mutating webhook "Admissions Enforcer" gây vòng lặp xung đột với webhook của Kyverno nếu không loại trừ — annotation `"admissions.enforcer/disabled": true` cần thêm vào webhook Kyverno (mặc định đã có sẵn từ v1.12+, verify lại nếu chạy version cũ hơn).
- Trên **private GKE**: control plane bị hạn chế giao tiếp tới worker node theo mặc định — cần firewall rule mở port 9443 (port webhook callback) từ control plane range vào node.
- Trên **EKS với custom CNI**: cần `hostNetwork: true` cho Kyverno pod hoặc nâng cấp VPC CNI, và mở inbound rule port 9443 trên security group — nếu không, webhook callback timeout âm thầm dù pod Kyverno vẫn Running khoẻ mạnh (dễ gây nhầm lẫn khi debug vì health check pod pass nhưng webhook vẫn fail).

## Security notes

- `failurePolicy: Ignore` chỉ nên dùng cho policy **không critical về security** — nếu webhook fail, request đi qua mà không được kiểm tra, tức là compliance/security guarantee bị mất âm thầm trong lúc outage, không có cảnh báo rõ ràng trừ khi có alert riêng theo dõi webhook availability.
- `failurePolicy: Fail` đảm bảo enforcement mạnh nhưng đổi lại là **single point of failure thật sự cho toàn cluster** — bất kỳ sự cố nào khiến toàn bộ Kyverno pod down (OOMKill hàng loạt, image pull lỗi sau khi node group đổi, node pool bị drain hết) đều có thể lan thành sự cố toàn cluster. Đây là lý do HA (≥3 replica Admission Controller, PodDisruptionBudget, anti-affinity) không phải "nice to have" mà là điều kiện bắt buộc khi chọn `Fail`.
- Luôn loại trừ `kube-system` và namespace Kyverno khỏi mọi `match` có phạm vi rộng (`*` namespace) — đây là biện pháp phòng ngừa quan trọng nhất để tránh kịch bản tự khoá cluster.

## Refs

- Policy Settings (failurePolicy, webhookTimeoutSeconds): https://kyverno.io/docs/policy-types/cluster-policy/policy-settings/
- Troubleshooting (API server blocked, cloud-specific issues): https://kyverno.io/docs/troubleshooting/
