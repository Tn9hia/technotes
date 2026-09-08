# Trivy — Database Management (vuln-db / java-db / checks-bundle)
Tier: 2
Parent: [[Trivy]]
Related: [[trivy--cache-performance]], [[trivy--troubleshooting-runbook]], [[trivy--kubernetes-scanning]]
Tags: #trivy #database #air-gap #ops

## What it does

Trivy binary tự thân **không chứa dữ liệu bảo mật nào** — nó chỉ là scan engine. Mọi khả năng phát hiện đến từ 3 "database" được đóng gói dưới dạng **OCI artifact** (không phải file tải rời như signature DB truyền thống), publish công khai lên container registry, và Trivy tự pull + cache khi cần:

| DB | Artifact name | Nội dung | Dùng khi nào |
|---|---|---|---|
| Vulnerability DB | `trivy-db` | CVE tổng hợp từ nhiều feed (OS vendor + language advisory) | Mọi lần scan vulnerability |
| Java DB | `trivy-java-db` | Index hash digest của Java artifact | Chỉ khi scan file `.jar/.war/.par/.ear` |
| Checks Bundle | `trivy-checks` | Logic Rego của misconfiguration check | Chỉ khi bật misconfig scanner |

## Why it exists

Vulnerability data thay đổi liên tục (CVE mới mỗi ngày) trong khi binary Trivy release theo chu kỳ chậm hơn nhiều. Tách DB ra khỏi binary và phân phối qua OCI registry cho phép: (1) cập nhật dữ liệu độc lập với việc upgrade CLI, (2) tận dụng hạ tầng registry sẵn có (caching, mirror, versioning) thay vì tự xây CDN riêng, (3) cho phép self-host DB trong môi trường air-gapped bằng cách copy image sang registry nội bộ.

## How it works (flow/diagram)

```
trivy image alpine:3.20
        │
        ▼
Cần trivy-db trong cache? ──có──▶ dùng cache (theo TTL/schema version)
        │ không
        ▼
Thử pull theo thứ tự --db-repository (default):
  1. mirror.gcr.io/aquasec/trivy-db:2      ← ưu tiên (đổi từ v0.57.1, tránh rate-limit GHCR)
  2. ghcr.io/aquasecurity/trivy-db:2       ← fallback nếu (1) lỗi 429/5xx
        │
        ▼
Lưu vào --cache-dir (mặc định theo OS: ~/.cache/trivy trên Linux)
        │
        ▼
Chạy scan, so khớp package version với DB
```

- Tag ảnh không chỉ định version cụ thể sẽ mặc định lấy theo **schema number** (`:2` cho trivy-db, `:1` cho trivy-java-db), không phải `:latest`.
- `trivy-checks` có **embedded fallback ngay trong binary** (build sẵn tại thời điểm release Trivy) — nếu không pull được từ registry, Trivy vẫn chạy misconfig scan bằng checks tại thời điểm binary được build, chỉ là **không có check mới nhất**.
- `trivy-db` và `trivy-java-db` **không có fallback tương đương** — không tải được là không scan vulnerability được, đây là khác biệt quan trọng khi thiết kế air-gapped setup.

## Config gotchas

- **Default registry đổi thứ tự kể từ v0.57.1**: `mirror.gcr.io` được ưu tiên trước `ghcr.io` sau sự cố rate-limit GHCR cuối 2024/đầu 2025 (tải cộng đồng vượt rate limit namespace GHCR). Nếu pin Trivy version cũ hơn, mày vẫn đang gọi thẳng `ghcr.io` trước — dễ dính rate limit hơn version mới. Xem [Issue #7938](https://github.com/aquasecurity/trivy/issues/7938).
- **`--db-repository` override sẽ THAY THẾ hoàn toàn danh sách default**, không phải thêm vào. Muốn giữ fallback về default mà chỉ thêm 1 mirror nội bộ, phải liệt kê đủ cả 3: `--db-repository my.registry.local/trivy-db --db-repository mirror.gcr.io/aquasec/trivy-db:2 --db-repository ghcr.io/aquasecurity/trivy-db:2`.
- **`--checks-bundle-repository` KHÔNG hỗ trợ multi-value fallback** như 2 flag kia (do đã có embedded fallback nên không cần) — set nhiều giá trị cho flag này không có tác dụng như mong đợi.
- **`--skip-db-update` fail cứng nếu DB local đang ở schema version cũ mà CLI đã bump schema mới** (`--skip-update cannot be specified with the old DB schema`). Nghĩa là combo "air-gapped + không update CLI kịp thời" có thể tự làm gãy chính mình.
- **Tần suất publish DB mới đã giảm từ mỗi 6 giờ xuống mỗi 24 giờ** (cùng đợt fix rate-limit) — đừng kỳ vọng CVE mới xuất hiện được vài giờ đã có trong `trivy-db`, độ trễ thực tế tính bằng **ngày**, không phải giờ.
- `trivy clean --vuln-db --java-db --checks-bundle` hoặc `--all` để xoá sạch cache khi nghi ngờ DB hỏng/version lệch — an toàn hơn tự tay xoá thư mục cache vì Trivy biết chính xác path nào cần dọn.
- `--download-db-only` / `--download-java-db-only` hữu ích để pre-warm cache trong build image CI (bake DB vào base image scanner) mà không cần scan target thật.

## Security notes

- Vì DB là OCI artifact công khai, ai cũng pull được — **không có xác thực nội dung mặc định ngoài TLS transport** giữa Trivy và registry. Nếu tổ chức cần đảm bảo integrity cao hơn, cân nhắc self-host DB qua registry nội bộ có kiểm soát access + audit log riêng, thay vì luôn pull thẳng từ internet.
- Air-gapped environment bắt buộc phải có **quy trình đồng bộ định kỳ** (copy OCI artifact từ registry công khai vào registry nội bộ) — nếu quên, hệ thống production sẽ scan với DB "đóng băng" tại thời điểm setup ban đầu, tạo ra false sense of security ngày càng lớn theo thời gian.
- `--insecure`/`TRIVY_INSECURE=true` khi pull DB từ registry tự ký (self-signed) bỏ qua xác thực cert — chỉ chấp nhận được nếu registry nội bộ nằm trong network đã kiểm soát chặt, tuyệt đối không dùng khi vẫn còn pull từ internet.

## Refs

- https://trivy.dev/latest/docs/guide/configuration/db/ (nguồn: `docs/guide/configuration/db.md` trên repo)
- https://trivy.dev/latest/docs/guide/advanced/air-gap/
- https://github.com/aquasecurity/trivy-db
- https://github.com/aquasecurity/trivy/issues/7938 (rate-limit fix, đổi default sang mirror.gcr.io)
- https://github.com/aquasecurity/trivy/discussions/8009 (giải thích GITHUB_TOKEN không giúp rate-limit DB)
