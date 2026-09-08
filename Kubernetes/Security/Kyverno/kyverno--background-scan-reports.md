# Kyverno — Background Scan & Policy Reports
Tier: 2
Parent: [[Kyverno]]
Related: [[kyverno--architecture-controllers]], [[kyverno--webhooks-admission]]
Tags: #kyverno #compliance #reporting

## What it does

Background scan là cơ chế quét định kỳ (mặc định **mỗi 1 giờ**) toàn bộ resource đang tồn tại trong cluster, đối chiếu với `validate`/`verifyImages` rule của mọi policy có `spec.background: true` (mặc định `true`). Kết quả không chặn/xoá gì cả — chỉ ghi nhận vi phạm vào report.

4 loại report trung gian + 2 loại report cuối:

| Report | Nguồn | Scope |
|---|---|---|
| `AdmissionReport` | Kết quả từ admission-time (request đi qua webhook) | Namespaced |
| `ClusterAdmissionReport` | như trên | Cluster-scoped resource |
| `BackgroundScanReport` | Kết quả từ background scan định kỳ | Namespaced |
| `ClusterBackgroundScanReport` | như trên | Cluster-scoped resource |
| **`PolicyReport`** | Reports Controller gộp 2 nguồn admission + background report (namespaced) | Namespaced (final) |
| **`ClusterPolicyReport`** | như trên | Cluster-scoped (final) |

## Why it exists

Không phải lúc nào enforce ngay cũng khả thi — cluster đang chạy production đã có sẵn hàng nghìn resource vi phạm 1 policy mới thì không thể enforce ngay (sẽ không chặn được gì vì admission chỉ áp cho request mới, và không có cơ chế nào "quét ngược" để biết cluster đang compliant tới đâu). Background scan + PolicyReport cho phép: viết policy ở chế độ `audit`, theo dõi compliance qua report trong 1 khoảng thời gian, rồi mới chuyển sang `enforce` khi chắc chắn không phá vỡ workload hiện có — đây là quy trình rollout policy an toàn chuẩn, không phải tính năng phụ.

## How it works (flow/diagram)

```
Policy có background:true, failureAction/validationFailureAction: Enforce
        │
        ├── Request mới (admission-time) ──▶ chặn ngay nếu vi phạm (đúng nghĩa Enforce)
        │                                     + ghi AdmissionReport
        │
        └── Resource CŨ đã tồn tại trước ──▶ Background Controller quét mỗi 1h (default)
             khi policy được tạo                  │
                                                   ▼
                                         vi phạm → BackgroundScanReport
                                         (KHÔNG bị block/xoá, vẫn chạy bình thường)
                                                   │
                                                   ▼
                              Reports Controller gộp AdmissionReport + BackgroundScanReport
                                                   │
                                                   ▼
                                    PolicyReport / ClusterPolicyReport (final, để xem/alert)
```

## Config gotchas

- **Nhầm lẫn phổ biến nhất**: nghĩ rằng bật `background: true` + đặt policy ở `Enforce` sẽ khiến resource cũ vi phạm tự bị dọn/chặn — SAI. Background scan không có khả năng chặn, chỉ report. Muốn dọn resource cũ phải xử lý thủ công hoặc dùng `CleanupPolicy`/`DeletingPolicy` riêng.
- Interval mặc định 1 giờ — với cluster nhiều resource + nhiều policy, chu kỳ này có thể tạo tải đột biến định kỳ lên apiserver/Background Controller; tuning qua container flag (xem docs container flags).
- `AdmissionReport`/`BackgroundScanReport` tồn đọng tăng liên tục = dấu hiệu Reports Controller (chỉ 1 replica active do leader election) bị nghẽn, thường do client-side rate limit thấp (`--clientRateLimitQPS/Burst`) chứ không phải do cần thêm replica — xem [[kyverno--architecture-controllers]].

## Security notes

- PolicyReport là nguồn compliance evidence quan trọng cho audit (CKS thường hỏi về continuous compliance) — nhưng đừng dùng nó như bằng chứng "cluster đang an toàn tại thời điểm hiện tại" nếu interval scan dài (1h): giữa 2 lần scan, resource vi phạm vẫn có thể tồn tại và chạy mà chưa được ghi nhận.
- Report data (`PolicyReport`/`ClusterPolicyReport`) là read model — không phải access control, không nên dùng logic ứng dụng khác dựa vào report để quyết định authorization (report có độ trễ, không real-time).

## Refs

- Policy Reports guide: https://kyverno.io/docs/guides/reports/
- Reports dev docs: https://github.com/kyverno/kyverno/blob/main/docs/dev/reports/README.md
