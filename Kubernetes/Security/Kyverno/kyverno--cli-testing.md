# Kyverno — CLI & Policy Testing
Tier: 2
Parent: [[Kyverno]]
Related: [[kyverno--policy-types-cel-vs-legacy]]
Tags: #kyverno #cli #testing #cicd

## What it does

`kyverno` CLI (subproject `kubectl-kyverno`) chạy engine Kyverno **offline, không cần cluster** — dùng để test policy trước khi apply vào production.

- `kyverno apply <policy.yaml> --resource <resource.yaml>` — chạy thử policy lên 1 resource cụ thể, trả kết quả pass/fail/skip (validate) hoặc resource đã mutate (mutate).
- `kyverno test <path>` — chạy bộ test có sẵn: đọc file `kyverno-test.yaml` khai báo policy + resource + kết quả kỳ vọng, so sánh kết quả thật với kỳ vọng, dùng được cho local filesystem lẫn remote git repo.

## Why it exists

Không có CLI, cách duy nhất để biết policy có đúng ý không là apply thẳng vào cluster (thường là dev/staging) rồi quan sát — chậm, và rủi ro nếu policy có lỗi logic (vd JMESPath sai gây bypass âm thầm như CVE-2025-46342, xem [[kyverno--security-cves-hardening]]). CLI cho phép viết test case như unit test bình thường, chạy trong CI/CD trước khi merge policy vào repo GitOps.

## How it works (flow/diagram)

```
policy.yaml + resource.yaml ──▶ kyverno apply ──▶ pass/fail/skip (validate)
                                                    hoặc resource đã mutate (mutate)

kyverno-test.yaml (policy + resource + expected result)
        │
        ▼
   kyverno test <path>  ──▶ so sánh actual vs expected ──▶ CI pass/fail
```

## Config gotchas

- CLI chạy engine **offline** — không có access tới cluster thật, nên context phụ thuộc cluster (`apiCall` tới resource thật, `configMap` context thật) cần mock/stub riêng trong test, không tự động lấy dữ liệu thật; cần đọc kỹ semantics test framework khi policy dùng external context.
- `kyverno-test.yaml` có thể trỏ tới path trên git repo remote — hữu ích để test policy nằm trong 1 repo chính sách chung mà không cần clone thủ công, nhưng cũng có nghĩa CI cần network access tới repo đó.

## Security notes

- CLI test **không thay thế** review bảo mật cho policy dùng `apiCall`/CEL external context — test case thường chỉ verify logic match/mutate đúng ý, không tự động phát hiện rủi ro cross-namespace escalation (đó là lỗi ở tầng engine/RBAC thật khi chạy trong cluster, không phải lỗi logic policy mà CLI test bắt được).
- Nên tích hợp `kyverno test` vào CI cho repo GitOps chứa policy — bắt lỗi cú pháp/logic sớm, nhưng vẫn cần audit riêng theo checklist ở [[kyverno--security-cves-hardening]] trước khi cho phép merge policy dùng context bên ngoài.

## Refs

- Kyverno CLI: https://kyverno.io/docs/subprojects/kyverno-cli/
- Testing Policies guide: https://kyverno.io/docs/guides/testing-policies/
- `kyverno test` reference: https://kyverno.io/docs/kyverno-cli/reference/kyverno_test/
- `kyverno apply` reference: https://kyverno.io/docs/kyverno-cli/reference/kyverno_apply/
