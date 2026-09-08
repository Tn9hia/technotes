# Kyverno — Security CVEs & Hardening Checklist
Tier: 2
Parent: [[Kyverno]]
Related: [[kyverno--policy-exceptions]], [[kyverno--webhooks-admission]]
Tags: #kyverno #security #cve #rbac #cks

> Nguồn: toàn bộ CVE dưới đây lấy trực tiếp từ GitHub Security Advisories API chính thức của repo `kyverno/kyverno` (`curl https://api.github.com/repos/kyverno/kyverno/security-advisories`), verify ngày 2026-09-06. Không dùng thông tin từ các trang "CVE blogspam" (SEO content mill) vì đã phát hiện có CVE ID không tồn tại trong nguồn chính thức bị các trang này bịa ra khi search — chỉ tin GHSA-* + link `github.com/kyverno/kyverno/security/advisories`.

## What it does

Kyverno's ServiceAccount (đặc biệt Admission Controller) cần quyền rộng để mutate/generate resource cross-namespace, cross-kind — đây là lý do RBAC misconfiguration hoặc bug trong việc enforce namespace boundary của chính Kyverno đã tạo ra 1 chuỗi CVE **cross-namespace privilege escalation** lặp lại nhiều lần qua các version khác nhau.

## Why it exists (tại sao đây là rủi ro cấu trúc, không phải bug ngẫu nhiên)

Cơ chế `apiCall` (gọi API server để lấy context bổ sung cho policy) và context loader (`configMap`, v.v.) đều thực thi **bằng danh tính ServiceAccount của Kyverno**, không phải danh tính user tạo ra policy. Nếu Kyverno không tự validate rằng namespace được policy tham chiếu trùng khớp namespace của chính policy đó, thì 1 user chỉ có quyền tạo `Policy` (namespaced) trong 1 namespace hẹp có thể "mượn" quyền rộng hơn của ServiceAccount Kyverno để đọc/ghi ngoài namespace mình được cấp — đây chính xác là root cause lặp lại của nhiều CVE dưới đây.

## Timeline CVE quan trọng (mới nhất trước, đã verify qua GHSA API)

| GHSA / CVE | Severity | Cơ chế | Version ảnh hưởng | Ghi chú patch |
|---|---|---|---|---|
| GHSA-79gf-7frw-68m9 / CVE-2026-54523 | Critical (9.6) | `NamespacedMutatingPolicy` CEL `generator.apply()` không validate namespace argument → tạo resource ở namespace bất kỳ (kể cả `kube-system`) qua `matchConditions` | v1.18.0, v1.18.1 | Published 2026-07-13; verify bản vá thật qua release notes v1.18.2/v1.19, advisory không ghi rõ `first_patched_version` |
| GHSA-cvq5-hhx3-f99p / CVE-2026-41068 | High (7.7) | "Incomplete fix" của CVE-2026-22039 — ConfigMap context loader (`configMap.namespace`) không validate namespace, trong khi `apiCall.URLPath` đã được vá | ≤ 1.17.0 | Published 2026-04-15; theo mô tả advisory, root cause **cùng pattern** với apiCall bug nhưng ở code path khác — bài học: vá 1 code path không có nghĩa toàn bộ class lỗi đã hết |
| GHSA-8p9x-46gm-qfx2 / CVE-2026-22039 | Critical (9.9) | `apiCall.URLPath` không giới hạn namespace cho namespaced Policy → đọc/ghi cross-namespace, kể cả tạo `ClusterPolicy` (cluster-scoped) từ 1 namespaced policy | ≤ 1.16.2, ≤ 1.15.2 | Published 2026-01-27 |
| GHSA-rggm-jjmc-3394 / CVE-2026-4789 | High (8.5) | SSRF qua CEL `http.Get`/`http.Post` trong `NamespacedValidatingPolicy` — không chặn cloud metadata IP (169.254.0.0/16) hay giới hạn service cùng namespace | ≥1.16.0, <1.17.0 | Published 2026-04-13 |
| GHSA-fpjq-c37h-cqcv / CVE-2026-41485 | High (7.7) | DoS: `forEach` mutation với `patchesJson6902` chứa variable resolve về `nil` → Go panic (type assertion không kiểm tra nil) — **chỉ ảnh hưởng legacy JMESPath engine, CEL-based policy không bị** | ≥1.13.0, ≤1.17.1, ≤1.16.3 | Published 2026-04-22 |
| GHSA-jrr2-x33p-6hvc / CVE-2025-46342 | High (8.5) | Thiếu error propagation trong `GetNamespaceSelectorsFromNamespaceLister` → rule dùng `namespaceSelector` trong `match` bị **âm thầm bỏ qua**, tưởng đang bảo vệ nhưng thực ra không | ≤1.13.4, ≤1.12.7, ≤1.11.5 | — |
| GHSA-gg4x-fgg2-h9w9 | Critical | Bypass bằng 2 PolicyException chồng nhau (chi tiết → [[kyverno--policy-exceptions]]) | v1.9.0–1.12.7 | — |
| GHSA-qjvc-p88j-j9rm / CVE-2024-48921 | Medium | PolicyException tạo được ở namespace bất kỳ (chi tiết → [[kyverno--policy-exceptions]]) | <1.13.0 | — |
| GHSA-hq4m-4948-64cc / CVE-2023-34091 | Low | Resource có `deletionTimestamp` (qua finalizer treo vô hạn) bypass validate/generate/mutate-existing policy dù `Enforce` | <1.10.0 | Fixed 1.10.0 |
| GHSA-33hq-f2mf-jm3c / CVE-2023-33191 | Medium | `validate.podSecurity` subrule không enforce đúng Seccomp check ở baseline level khi dùng `version: latest` | v1.9.2, v1.9.3 | Fixed v1.9.4/v1.10.0 |
| GHSA-m3cq-xcx9-3gvm / CVE-2022-47633 | High | `verifyImages` có thể bị bypass bởi malicious proxy/registry nếu không giới hạn registry tin cậy | v1.8.3, v1.8.4 | Fixed v1.8.5 |

## Config/Security gotchas rút ra từ nhóm CVE trên

1. **`apiCall`/CEL external-context là class rủi ro riêng, không phải lỗi đơn lẻ** — 4 CVE khác nhau (CVE-2026-22039, CVE-2026-41068, CVE-2026-4789, CVE-2026-54523) đều xoay quanh việc Kyverno thực thi network/API call bằng quyền ServiceAccount của chính nó mà không validate đúng boundary namespace của policy gọi nó. Nếu RBAC cho phép user tạo namespaced Policy/MutatingPolicy dùng `apiCall`/`context.configMap`/CEL `generator`/`http.Get`, coi đó tương đương cấp quyền rộng hơn namespace của họ cho tới khi verify version đang chạy đã vá đủ các advisory trên.
2. **DoS class**: JMESPath variable evaluation (CVE-2025-47281) và `forEach` nil-panic (CVE-2026-41485) — cả 2 đều là lỗi **chỉ ở engine legacy JMESPath**; CEL-based policy được ghi nhận rõ ràng "không bị ảnh hưởng" trong advisory forEach — thêm 1 lý do kỹ thuật (ngoài roadmap deprecation) để ưu tiên CEL cho policy mới.
3. **Silent bypass class**: namespaceSelector bug (CVE-2025-46342) và deletionTimestamp bug (CVE-2023-34091) đều có đặc điểm nguy hiểm nhất — **không có lỗi/log rõ ràng báo hiệu**, policy vẫn "trông như" đang hoạt động (Ready, không báo lỗi) nhưng thực ra âm thầm không áp dụng.

## Hardening checklist tối thiểu (áp dụng khi nhận bàn giao hệ thống)

1. Xác định version đang chạy chính xác đầu tiên (`kubectl get pods -n kyverno -o jsonpath=...` hoặc `helm list`), đối chiếu bảng CVE trên để biết đang lộ những lỗ hổng nào.
2. Nếu version cũ hơn 1.19 và đang dùng `apiCall`/CEL context/`configMap` context trong policy namespaced — audit toàn bộ các policy này thủ công, không chỉ dựa vào việc "đã patch chưa" (vì GHSA-cvq5 cho thấy 1 lần fix có thể không đủ).
3. RBAC: tách nhóm được tạo `Policy`/`PolicyException`/`*MutatingPolicy`/`*GeneratingPolicy` (namespaced) ra khỏi nhóm dev thường — coi các CRD này tương đương quyền có thể escalate, không phải config ứng dụng thông thường.
4. Nếu dùng `verifyImages`, luôn giới hạn registry tin cậy tường minh (không để mặc định chấp nhận registry bất kỳ) — tránh lặp lại lớp lỗi CVE-2022-47633.
5. Không dựa vào `namespaceSelector` trong `match` như lớp bảo vệ duy nhất nếu version chưa xác nhận đã vá CVE-2025-46342 — nên có thêm layer kiểm tra độc lập (vd Kubernetes NetworkPolicy/PSA) thay vì chỉ tin 1 policy.
6. Theo dõi advisory mới định kỳ qua API chính thức thay vì đọc lại blog cũ:
   `curl -s https://api.github.com/repos/kyverno/kyverno/security-advisories`

## Refs

- Security advisories (nguồn xác thực duy nhất nên tin): https://github.com/kyverno/kyverno/security/advisories
- Configuring Kyverno / RBAC extension: https://kyverno.io/docs/installation/customization/
