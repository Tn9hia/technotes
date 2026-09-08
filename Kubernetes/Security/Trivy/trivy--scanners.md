# Trivy — 4 Scanner Engine (Vulnerability / Misconfig / Secret / License)
Tier: 2
Parent: [[Trivy]]
Related: [[trivy--db-management]], [[trivy--filtering-suppression]]
Tags: #trivy #vulnerability #misconfiguration #secret #license

## What it does

Trivy có 4 scanner độc lập, bật/tắt qua flag `--scanners` (giá trị hợp lệ: `vuln,misconfig,secret,license`):

| Scanner | Phát hiện | Nguồn dữ liệu | Mặc định bật ở `trivy image`? |
|---|---|---|---|
| Vulnerability | CVE trong OS package + language dependency | `trivy-db` | ✓ |
| Misconfiguration | Lỗi cấu hình Dockerfile/K8s/Terraform/CloudFormation/Helm/Ansible | `trivy-checks` (Rego) | ✗ |
| Secret | API key, password, token lộ trong plaintext file (kể cả `.pyc`) | Built-in regex rule | ✓ |
| License | Risk classification của license package/source file | Google License Classification | ✗ |

**Default `--scanners` của `trivy image` là `[vuln, secret]`.** Subcommand `trivy config` thì mặc định chỉ bật `misconfig`. Đây là điểm rất dễ nhầm — không có 1 "default chung" cho mọi subcommand.

## Why it exists

Mỗi loại rủi ro cần cơ chế phát hiện khác hẳn nhau: vuln cần so khớp version với CVE database, misconfig cần parse cấu trúc IaC rồi chạy policy engine (Rego/OPA), secret cần pattern matching trên nội dung file, license cần classifier xác suất trên text. Gộp chung 1 command nhưng tách biệt scanner cho phép bật/tắt độc lập theo nhu cầu (vd CI pipeline chỉ cần vuln để nhanh, nhưng audit định kỳ thì bật đủ cả 4).

## How it works (flow/diagram)

```
--scanners vuln,misconfig,secret,license
        │
        ├─▶ Vuln:      detect OS (apk/dpkg/rpm) + lang files (go.sum, package-lock.json...)
        │              → so khớp version với trivy-db → severity theo vendor-priority
        │
        ├─▶ Misconfig: parse Dockerfile/K8s YAML/Terraform/CloudFormation/Helm/Ansible
        │              → evaluate Rego check (trivy-checks) → PASS/FAIL theo severity
        │
        ├─▶ Secret:    quét từng file plaintext theo regex rule (AWS key, GH token...)
        │              → trừ layer/path nằm trong "base image" hoặc "allow rules"
        │
        └─▶ License:   đọc metadata license package (npm/pip/gem/apk...)
                       → classify (Forbidden/Restricted/.../Unknown) → map ra severity
```

Severity của **license** được Trivy tự map cứng: Forbidden→CRITICAL, Restricted→HIGH, Reciprocal→MEDIUM, Notice/Permissive/Unencumbered→LOW, Unknown→UNKNOWN. Không cấu hình lại mapping này được qua flag, chỉ filter theo severity output.

## Config gotchas

- **Quên bật `misconfig` khi scan image** là lỗi phổ biến nhất — nhiều người chạy `trivy image myapp` xong kết luận "không có vấn đề config gì" trong khi thực ra misconfig scanner còn chưa chạy.
- **Secret scanner làm chậm scan đáng kể** với image lớn — chính Trivy tự in gợi ý trong log: `If your scanning is slow, please try '--scanners vuln' to disable secret scanning`. Cân nhắc tách secret scanning ra 1 bước CI riêng thay vì luôn gộp chung.
- **Trivy cố tự phát hiện base image layer và skip secret scan trên layer đó** để tăng tốc — nếu 1 secret nằm trong layer bị nhận nhầm là "base image", nó sẽ **không được phát hiện** dù thực ra secret đó mới được thêm vào. Debug bằng `--debug` để xem Trivy nhận diện layer nào là base.
- **`--license-full` rất tốn thời gian** (quét cả source code, Markdown, text file tìm license, không chỉ metadata package) — chỉ bật khi thực sự cần audit license sâu, không nên để mặc định trong pipeline nhanh.
- **`--license-confidence-level` mặc định 0.9** — hạ xuống để bắt được nhiều license hơn nhưng tăng false positive; license bị Trivy phân loại `UNKNOWN` không có nghĩa là an toàn, chỉ là **classifier không tự tin đủ để gán nhãn**, vẫn cần người review thủ công.
- **`--misconfig-scanners`** cho phép chọn cụ thể loại IaC nào chạy (`azure-arm,cloudformation,dockerfile,helm,kubernetes,terraform,terraformplan-json,terraformplan-snapshot,ansible`) — hữu ích để tắt bớt loại không dùng, giảm thời gian scan trên repo lớn có nhiều loại file lẫn lộn.
- **`--pkg-types`** (default `os,library`) quyết định có quét package OS-level và/hoặc language-level hay không — set thiếu 1 trong 2 sẽ bỏ sót cả 1 tầng dependency mà không có cảnh báo rõ ràng nào ngoài số lượng finding thấp bất thường.

## Security notes

- **Vuln severity không tuyệt đối** — nó phụ thuộc `--vuln-severity-source` (thứ tự ưu tiên nguồn dữ liệu). Đừng coi 1 con số severity là chân lý cố định; luôn xem `SeveritySource` trong JSON output để biết severity đến từ đâu khi cần giải trình cho audit/compliance.
- **Secret scanner dựa trên rule tĩnh (regex)** — kẻ tấn công/dev vô tình có thể encode/obfuscate secret (base64, split string) để né rule. Không nên coi Trivy secret scanner là lớp phòng thủ duy nhất; kết hợp thêm pre-commit hook (gitleaks, git-secrets) ở tầng sớm hơn.
- **Misconfig check dựa trên Rego có thể lỗi thời** so với best practice mới nhất của cloud provider — luôn kiểm tra `trivy-checks` version đang dùng (`trivy --version` show rõ) khi thấy check "thiếu" so với kỳ vọng.
- **License scanning không phải tư vấn pháp lý** — nó chỉ đưa ra classification tham khảo dựa trên text matching, không thay thế review pháp lý thật khi có tranh chấp license nghiêm trọng.

## Refs

- https://trivy.dev/latest/docs/scanner/vulnerability/
- https://trivy.dev/latest/docs/scanner/misconfiguration/
- https://trivy.dev/latest/docs/scanner/secret/
- https://trivy.dev/latest/docs/scanner/license/
- https://trivy.dev/latest/docs/coverage/ (danh sách OS/ngôn ngữ/IaC được hỗ trợ, thay đổi theo version)
