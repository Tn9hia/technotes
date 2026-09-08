# Kyverno — Troubleshooting & Ops Runbook
Tier: 2
Parent: [[Kyverno]]
Related: [[kyverno--webhooks-admission]], [[kyverno--architecture-controllers]]
Tags: #kyverno #ops #troubleshooting #runbook

## What it does

Danh sách sự cố thường gặp nhất, tổng hợp từ trang troubleshooting chính thức (https://kyverno.io/docs/troubleshooting/) — đây là nguồn tham chiếu đầu tiên khi debug, ưu tiên hơn blog/Stack Overflow vì được maintainer cập nhật theo version.

## Why it exists

Kyverno vận hành như 1 admission controller có quyền ảnh hưởng tới **mọi** request ghi vào cluster — sự cố của nó lan rất nhanh và rất rộng (khác với 1 microservice thường chỉ ảnh hưởng đúng chức năng của nó). File này tồn tại để tra cứu nhanh khi có alert, không phải để đọc hiểu 1 lần.

## How it works (checklist theo triệu chứng)

**1. Apiserver bị chặn / không tạo được resource nào** (nghiêm trọng nhất)
- Nguyên nhân: toàn bộ Kyverno pod down + policy match rộng + `failurePolicy: Fail`.
- Fix khẩn cấp: xoá tạm `validatingwebhookconfiguration`/`mutatingwebhookconfiguration` của Kyverno → giải phóng apiserver → fix Kyverno pod → webhook tự đăng ký lại (nếu quản lý qua Helm) hoặc cài lại thủ công.

**2. Policy không được áp dụng dù đã tạo**
- Kiểm tra theo thứ tự: pod Kyverno có Running không → policy status `Ready: true` chưa (`kubectl get cpol`) → webhook đã đăng ký đúng resource kind chưa → DNS/network giữa apiserver và `kyverno-svc` thông chưa → `failureAction`/`validationFailureAction` có đang là `Audit` (không phải `Enforce`) không → resource có nằm trong `match`/bị loại trừ bởi user/role exclusion không.

**3. Resource consumption cao / OOMKill**
- Nguyên nhân phổ biến: policy match wildcard quá rộng tạo tải xử lý lớn; resource limit mặc định không đủ cho cluster lớn.
- Fix: tránh wildcard match không cần thiết; xác định đúng controller nào đang quá tải (Admission/Background/Reports) qua metric trước khi tăng resource; tăng `requests/limits`; theo dõi số `UpdateRequest` tồn đọng.

**4. Response chậm / API server bị throttle**
- Nguyên nhân: Kyverno gọi apiserver quá nhiều (nhiều policy, nhiều report) mà client-side rate limit mặc định thấp.
- Fix: tăng `--clientRateLimitQPS` và `--clientRateLimitBurst` (khi report tồn đọng do throttle, cần tăng cao, vd 300–500).

**5. Policy áp dụng "một nửa" / kết quả không như kỳ vọng**
- Fix: xem log ở `-v=6` (rule-level debug).

**6. Kyverno pod crash liên tục**
- Nguyên nhân thường gặp: thiếu memory cho cluster/policy-set lớn.
- Fix: tăng `resources.limits.memory`.

**7. Policy/variable không hoạt động đúng, nghi ngờ do substitution sai**
- Fix: bật flag `dumpPayload` để xem payload thật apiserver gửi; tăng verbosity `-v=4` để xem giá trị variable đã resolve.

**8. AdmissionReport/report tồn đọng tăng liên tục**
- Nguyên nhân: Reports Controller không chạy, hoặc bị throttle khi aggregate.
- Fix: verify Reports Controller đang Running (nhớ: chỉ 1 replica active do leader election, xem pod nào là leader); tăng rate limit như mục 4.

**9. Background/Generate rule không chạy vì thiếu quyền**
- Nguyên nhân: Background Controller ClusterRole chưa được cấp quyền cho resource/CRD custom.
- Fix: bổ sung permission vào ClusterRole tương ứng (Kyverno dùng cluster role aggregation, extend được mà không sửa role gốc); verify bằng `kubectl auth can-i --as=system:serviceaccount:kyverno:kyverno-background-controller <verb> <resource>`.

## Cloud-specific gotchas

| Platform | Vấn đề | Fix |
|---|---|---|
| **AKS** | AKS "Admissions Enforcer" mutating webhook xung đột/loop với webhook Kyverno | Annotation `admissions.enforcer/disabled: true` trên webhook Kyverno (mặc định có sẵn từ v1.12+) |
| **Private GKE** | Control plane không tới được worker node theo mặc định | Firewall rule mở port 9443 từ control plane range vào node |
| **EKS (custom CNI hoặc VPC CNI cũ)** | Webhook callback timeout dù pod Running khoẻ | `hostNetwork: true` cho pod Kyverno, hoặc nâng cấp VPC CNI; mở inbound port 9443 trên security group |

## Config gotchas

- Đừng nhầm "policy Ready" với "policy đang enforce" — policy có thể Ready nhưng đang ở `Audit`, không chặn gì cả dù tưởng đã bảo vệ.
- Log level: `-v=4` cho variable substitution, `-v=6` cho rule-level chi tiết — tăng verbosity tạm thời khi debug, hạ lại sau vì log lớn ảnh hưởng disk/log pipeline production.

## Security notes

- Việc xoá webhook config để "giải cứu" apiserver khi sự cố (mục 1) là thao tác **tạm thời khẩn cấp**, không phải fix — sau khi xoá, cluster tạm thời **không còn được Kyverno bảo vệ** cho tới khi webhook được đăng ký lại; cần theo dõi sát và khôi phục càng nhanh càng tốt, không để trạng thái "không bảo vệ" kéo dài.

## Refs

- Troubleshooting chính thức: https://kyverno.io/docs/troubleshooting/
