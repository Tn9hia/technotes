# Kyverno — Policy Types: Legacy (JMESPath) vs CEL
Tier: 2
Parent: [[Kyverno]]
Related: [[kyverno--webhooks-admission]], [[kyverno--policy-exceptions]], [[kyverno--cli-testing]]
Tags: #kyverno #policy-types #cel #jmespath #deprecation

## What it does

Kyverno có **2 họ policy CRD song song** hiện nay:

**Legacy (group `kyverno.io`, ngôn ngữ JMESPath) — deprecated từ v1.19, dự kiến gỡ v1.20 (~11/2026):**
- `ClusterPolicy` / `Policy` (`v1`) — chứa rule `validate`, `mutate`, `generate`, `verifyImages` trong cùng 1 object, match bằng `match.resources` + JMESPath variable `{{ request.object... }}`.
- `CleanupPolicy` / `ClusterCleanupPolicy` (`v2`).
- `PolicyException` (`v2beta1`, `kyverno.io`).

**Mới (group `policies.kyverno.io/v1`, ngôn ngữ CEL) — full feature parity với legacy kể từ v1.19:**
- `ValidatingPolicy` — mở rộng từ Kubernetes `ValidatingAdmissionPolicy`, thêm tính năng cho policy-as-code (context bên ngoài, exception...).
- `MutatingPolicy`, `GeneratingPolicy`, `DeletingPolicy` (thêm từ v1.15, 07/2025).
- `ImageValidatingPolicy` — tách riêng verifyImages ra khỏi ClusterPolicy thành CRD độc lập, tập trung supply-chain security.
- Mỗi type đều có biến thể **Namespaced*** (`NamespacedValidatingPolicy`, `NamespacedMutatingPolicy`, ...) chỉ áp dụng trong namespace tạo ra nó — mỗi type là 1 CRD riêng, **không** gộp validate/mutate/generate vào chung 1 object như ClusterPolicy legacy nữa.

## Why it exists

CEL là engine expression đã có sẵn trong Kubernetes core (dùng cho `ValidatingAdmissionPolicy`, CRD validation rules) — dùng chung CEL giúp Kyverno tương thích ngữ nghĩa với native K8s, dễ audit hơn JMESPath (vốn là ngôn ngữ riêng của JMESPath.org, không phải K8s-native), và mở khả năng compile-time type checking mà JMESPath (thuần string interpolation runtime) không có. Việc tách mỗi action (`validate`/`mutate`/`generate`/`delete`) thành 1 CRD riêng thay vì gộp trong `ClusterPolicy` giúp RBAC scope chính xác hơn (vd: cấp quyền tạo `ValidatingPolicy` mà không vô tình cấp luôn quyền viết rule `generate` có thể escalate).

## How it works (flow/diagram)

```
Legacy:  ClusterPolicy { rules: [ {validate}, {mutate}, {generate}, {verifyImages} ] }
                              │
                              ▼  JMESPath: {{ request.object.spec.containers[].image }}
                         engine (kyverno.io/v1 apiVersion, group "kyverno.io")

CEL:     ValidatingPolicy / MutatingPolicy / GeneratingPolicy / DeletingPolicy / ImageValidatingPolicy
         (mỗi cái là 1 CRD riêng, group "policies.kyverno.io/v1")
                              │
                              ▼  CEL: object.spec.containers.all(c, c.image.matches('...'))
                         engine mới (không dùng JMESPath, không có autogen annotation cũ)
```

Autogen cho pod controller (Deployment/StatefulSet/DaemonSet/Job/CronJob) là cơ chế **chỉ tồn tại ở legacy `ClusterPolicy`**: viết rule nhắm `Pod`, Kyverno tự thêm match cho các controller cha, đặt annotation `pod-policies.kyverno.io/autogen-controllers` (default `DaemonSet,Deployment,Job,StatefulSet,CronJob`) lên chính policy đó và tự sinh thêm rule đặt pattern dưới `spec.template.spec` (CronJob cần rule riêng vì lồng thêm 1 cấp `spec.jobTemplate.spec.template.spec`). CEL-based type xử lý pod controller theo cơ chế khác (chưa xác nhận chi tiết trong docs — cần đọc kỹ `docs/policy-types/` bản mới nhất khi thực hành).

## Config gotchas

- **Không trộn field validationFailureAction/failureAction**: legacy dùng `validationFailureAction: audit|enforce` (chữ thường), CEL dùng `failureAction: Audit|Enforce` (chữ hoa). Copy-paste giữa 2 style trong giai đoạn migrate là lỗi gõ phổ biến.
- Autogen annotation-driven → nếu tự đặt tên custom Pod controller trùng tên với 1 trong 5 controller chuẩn (`DaemonSet,Deployment,Job,StatefulSet,CronJob`), autogen có thể xử lý sai (bug đã ghi nhận: kyverno/kyverno#7446).
- Autogen không hoạt động đúng với rule dùng `match.any` kết hợp 1 số điều kiện (bug lịch sử kyverno/kyverno#2337) — nếu policy tưởng chừng nhắm đúng Pod controller mà autogen rule không sinh ra như kỳ vọng, kiểm tra lại cấu trúc `match`.
- `kyverno migrate` CLI hỗ trợ convert field-by-field từ ClusterPolicy sang type CEL tương ứng — dùng công cụ này thay vì viết tay lại từ đầu khi migrate (xem migration guide).

## Security notes

- JMESPath variable resolution có lịch sử DoS: `CVE-2025-47281` — Denial of Service via Improper JMESPath Variable Evaluation; và context variable order trong JMESPath chỉ được reference **tuần tự** (biến sau không dùng được biến định nghĩa sau nó) — viết sai thứ tự gây lỗi runtime khó debug chứ không phải lỗi cú pháp rõ ràng.
- Do 2 engine hoàn toàn khác nhau (JMESPath string-interpolation vs CEL typed-expression), **security review phải làm lại riêng** cho mỗi họ policy khi audit — không thể giả định policy CEL an toàn chỉ vì bản JMESPath tương đương đã được review.
- CVE-2025-46342 (bypass namespace selector trong `match`, do thiếu error propagation khiến Kyverno âm thầm bỏ qua rule) là lỗi riêng của engine legacy — 1 lý do kỹ thuật (ngoài roadmap) khiến team Kyverno đẩy mạnh CEL.

## Refs

- Migration guide: https://kyverno.io/docs/guides/migration-to-cel/
- Autogen (legacy): https://kyverno.io/docs/policy-types/cluster-policy/autogen/
- ValidatingPolicy: https://kyverno.io/docs/policy-types/validating-policy/
- ImageValidatingPolicy: https://kyverno.io/docs/policy-types/image-validating-policy/
- Release 1.14 (CEL validate/image-validate GA): https://kyverno.io/blog/2025/04/25/announcing-kyverno-release-1.14/
