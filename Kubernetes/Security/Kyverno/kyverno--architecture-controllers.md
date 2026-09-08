# Kyverno — Architecture & Controllers
Tier: 2
Parent: [[Kyverno]]
Related: [[kyverno--webhooks-admission]], [[kyverno--background-scan-reports]], [[kyverno--troubleshooting-runbook]]
Tags: #kyverno #architecture #ha

## What it does

Từ v1.x, Kyverno tách thành 4 controller độc lập (có thể deploy riêng qua Helm sub-chart), thay vì 1 monolith:

| Controller | Xử lý | Leader election? |
|---|---|---|
| **Admission Controller** | Webhook callback (validate/mutate tại admission time), quản lý cert + đăng ký webhook config | Không cho webhook traffic; **có** cho cert/webhook management |
| **Background Controller** | `generate` + `mutate-existing` rules, thông qua resource trung gian `UpdateRequest`; background scan định kỳ | Có (chỉ 1 replica active) |
| **Reports Controller** | Gộp `AdmissionReport`/`ClusterAdmissionReport` (từ admission) + `BackgroundScanReport`/`ClusterBackgroundScanReport` (từ background scan) thành `PolicyReport`/`ClusterPolicyReport` cuối cùng | Có |
| **Cleanup Controller** | Xử lý `CleanupPolicy`/`ClusterCleanupPolicy` — xoá resource theo lịch/điều kiện | Có (cho cert/webhook); là controller duy nhất khác Admission Controller tự quản webhook riêng |

Tất cả trừ Admission Controller là **optional component** — có thể không cài nếu không dùng generate/mutate-existing/cleanup.

## Why it exists

Trước đây (bản cũ hơn) toàn bộ logic nằm trong 1 pod — mọi background scan/report generation cạnh tranh CPU/memory với đường admission-time vốn nhạy latency (webhook có timeout cứng 1–30s). Tách controller cho phép: (1) scale riêng đường admission (nhiều replica song song, không leader election) để tăng throughput thật; (2) các tác vụ nặng nhưng không nhạy latency (background scan, report aggregation, cleanup) chạy leader-election, không cần scale ngang để tăng tốc — chỉ cần failover.

## How it works (flow/diagram)

Xem diagram tổng trong [[Kyverno]] mục 4. Điểm mấu chốt cần nhớ: **Admission Controller không dùng leader election cho việc serve webhook** — nghĩa là tăng replica Admission Controller = tăng throughput admission thật sự (song song xử lý), khác hẳn với Background/Reports/Cleanup Controller — tăng replica ở đó chỉ tăng **độ sẵn sàng** (failover khi leader chết), không tăng tốc xử lý vì tại 1 thời điểm chỉ có 1 leader làm việc.

`UpdateRequest` là resource trung gian: khi 1 policy có rule `generate` hoặc `mutate` nhắm resource đã tồn tại (`mutate-existing`), Background Controller không thực thi ngay mà tạo `UpdateRequest` rồi reconcile — cơ chế này cho phép retry/backoff nếu thao tác thất bại (vd thiếu RBAC).

## Config gotchas

- Helm value cho HA: `admissionController.replicas` (khuyến nghị ≥3), `backgroundController/reportsController/cleanupController.replicas` (khuyến nghị ≥2 cho failover, KHÔNG kỳ vọng tăng tốc xử lý khi tăng số này).
- Nếu thấy `UpdateRequest`/`AdmissionReport` tồn đọng tăng dần mà Admission Controller vẫn khoẻ → nghi ngờ đầu tiên là Background/Reports Controller (chỉ 1 replica active do leader election) đang bị throttle bởi client-side rate limit (`--clientRateLimitQPS`/`Burst` mặc định thấp) — không phải do thiếu replica.
- Insufficient RBAC cho Background Controller là lỗi phổ biến khi custom thêm generate/mutate-existing rule nhắm resource/CRD ngoài phạm vi mặc định — cần tự thêm permission vào ClusterRole tương ứng (Kyverno dùng cluster role aggregation nên có thể extend mà không sửa ClusterRole gốc), verify bằng `kubectl auth can-i --as=system:serviceaccount:kyverno:kyverno-background-controller <verb> <resource>`.

## Security notes

- ServiceAccount của Background Controller có quyền tạo/sửa resource cross-namespace (để generate hoạt động) — đây là attack surface nếu RBAC bị mở rộng quá mức cần thiết (vd cấp `cluster-admin` cho nhanh thay vì scope đúng resource). Xem CVE cụ thể liên quan `apiCall`/CEL context ở [[kyverno--security-cves-hardening]].
- Cleanup Controller có quyền **xoá** resource — misconfigure `CleanupPolicy` match quá rộng (không giới hạn label/namespace) là rủi ro data-loss vận hành thật, không phải lý thuyết.

## Refs

- High Availability: https://kyverno.io/docs/guides/high-availability/
- Controller dev docs: https://github.com/kyverno/kyverno/blob/main/docs/dev/controllers/README.md
- Configuring Kyverno (container flags, rate limit): https://kyverno.io/docs/installation/customization/
