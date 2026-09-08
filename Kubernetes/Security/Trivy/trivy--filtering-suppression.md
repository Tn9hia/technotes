# Trivy — Filtering & Suppression (Ignore, Rego, VEX, Exit Code)
Tier: 2
Parent: [[Trivy]]
Related: [[trivy--scanners]], [[trivy--troubleshooting-runbook]]
Tags: #trivy #filtering #ignore #vex #ci-cd

## What it does

Trivy filter kết quả qua 1 pipeline nhiều tầng theo thứ tự cố định:

```
Detected Issues → [Prioritization: Severity → Status] → [Suppression: Finding ID → Rego → VEX] → Results
```

- **Prioritization** (thu hẹp theo mức độ): `--severity`, `--ignore-status`/`--ignore-unfixed`.
- **Suppression** (loại bỏ finding cụ thể): `.trivyignore`/`.trivyignore.yaml` (by ID), `--ignore-policy` (Rego), `--vex` (VEX document).
- Song song đó, `--exit-code` quyết định process exit thế nào sau khi có kết quả cuối — **đây không phải bước filter, nhưng thường bị hiểu nhầm là 1 phần của filtering**.

## Why it exists

Không phải tổ chức nào cũng cần block mọi CVE tìm được — có CVE đã accept risk, có CVE không áp dụng trong context cụ thể (VEX `not_affected`), có CVE chưa có bản vá và team quyết định tạm chấp nhận có thời hạn. Nếu không có cơ chế filter linh hoạt, mỗi lần scan sẽ phải xử lý lại y hệt danh sách finding cũ đã review, gây "alert fatigue" y hệt vấn đề Falco gặp phải ở tầng runtime.

## How it works (flow/diagram)

```
$ trivy image --severity HIGH,CRITICAL \
              --ignore-unfixed \
              --ignorefile .trivyignore.yaml \
              --ignore-policy policy.rego \
              --vex repo \
              --exit-code 1 \
              myimage:tag

1. Detect toàn bộ vuln/misconfig/secret/license
2. Lọc severity: chỉ giữ HIGH/CRITICAL
3. Lọc status: --ignore-unfixed = ẩn affected/will_not_fix/fix_deferred/end_of_life
4. Loại theo Finding ID: match CVE-ID/AVD-ID/license-name trong .trivyignore.yaml
5. Loại theo Rego: evaluate package `trivy`, rule `ignore` (input = từng finding JSON)
6. Loại theo VEX: statement not_affected/false_positive từ VEX document
7. Kết quả cuối → nếu còn finding thoả điều kiện → exit code 1 (nếu set)
```

## Config gotchas

- **`--ignore-unfixed` KHÔNG chỉ ẩn "sẽ không vá"** — nó là shorthand của `--ignore-status affected,will_not_fix,fix_deferred,end_of_life`, tức ẩn cả **`affected`** (đang bị ảnh hưởng nhưng CHƯA CÓ patch). Dùng nhầm flag này trong compliance report có thể khiến rủi ro thật biến mất khỏi báo cáo mà không ai nhận ra.
- **Status filter (`--ignore-status`) chỉ support đầy đủ trên RHEL/Debian** — bảng support status (`will_not_fix`, `fix_deferred`, `end_of_life`, `under_investigation`) khác nhau tuỳ OS, OS khác chỉ có `fixed`/`affected` cơ bản. Đừng kỳ vọng hành vi giống hệt nhau giữa các distro.
- **`.trivyignore.yaml` vẫn EXPERIMENTAL và phải khai báo tường minh** qua `--ignorefile ./.trivyignore.yaml` — không tự động được nhận diện như `.trivyignore` (`.txt`-style) mặc định. Đổi extension thành `.yaml` mà quên flag = ignore rule im lặng không áp dụng, dễ tưởng nhầm là bug.
- **`purls` trong `.trivyignore.yaml` chỉ hoạt động cho vulnerabilities**, không áp dụng cho misconfig/secret/license — cần dùng `paths` cho các loại finding khác.
- **`expired_at` (yaml) / `exp:` (trivyignore text) rất dễ bị bỏ quên** — set 1 lần rồi không ai review lại, CVE hết hạn ignore vẫn tiếp tục bị ẩn nếu quên đặt ngày hết hạn ngay từ đầu. Nên coi việc **không set expiry** là code smell cần review.
- **Rego ignore policy yêu cầu đúng package name `trivy` và rule `ignore`** — sai tên package/rule sẽ khiến Rego load "thành công" (không lỗi cú pháp) nhưng không filter được gì, vì Trivy tìm không thấy rule mong đợi.
- **`--show-suppressed`** để debug xem cái gì đã bị ẩn và bởi cơ chế nào (`.trivyignore.yaml`, VEX, CSAF...) — nên dùng flag này khi setup lần đầu để chắc chắn rule ignore hoạt động đúng ý, tránh vừa ẩn nhầm vừa tưởng đang hoạt động tốt.

## Security notes

- **File `.trivyignore`/`.trivyignore.yaml` nằm trong repo = quyết định "chấp nhận rủi ro" trở thành code, review được qua PR** — đây là điểm tốt (audit trail rõ ràng), nhưng cũng là điểm yếu nếu ai đó âm thầm thêm CVE-ID vào ignore list mà không qua review kỹ (case cổ điển: "ẩn lỗi thay vì vá lỗi"). Nên có CODEOWNERS/branch protection riêng cho các file ignore này.
- **Rego ignore-policy là EXPERIMENTAL** — logic filter phức tạp viết bằng Rego khó audit bằng mắt hơn 1 danh sách CVE-ID phẳng; cân nhắc kỹ trước khi dùng cho policy ảnh hưởng security gate quan trọng, và luôn có test riêng cho policy này (không chỉ tin tưởng nó chạy đúng).
- **VEX cho phép bên thứ ba (vendor) tuyên bố "not affected"** — nếu tin tưởng VEX document từ nguồn không đáng tin, có thể vô tình bỏ qua CVE thật sự ảnh hưởng. Chỉ nhận VEX từ nguồn ký số/xác thực được (CSAF có cơ chế ký).
- **`--exit-code` mặc định 0** (xem [[Trivy]] mục 6) là lỗ hổng quy trình phổ biến nhất — không phải lỗi kỹ thuật của Trivy, mà là lỗi thiết kế pipeline khi không set flag này tường minh.

## Refs

- https://trivy.dev/latest/docs/guide/configuration/filtering/
- https://trivy.dev/latest/docs/supply-chain/vex/ (VEX overview)
- https://www.openpolicyagent.org/docs/latest/policy-language/ (Rego syntax)
- Ví dụ Rego ignore-policy chính thức: https://github.com/aquasecurity/trivy/tree/main/examples/ignore-policies
