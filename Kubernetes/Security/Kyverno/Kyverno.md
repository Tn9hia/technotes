# Kyverno — v1.19.0 (stable)
Tags: #kubernetes #admission-controller #policy-as-code #security #cks
Last updated: 2026-09-06

> ⚠️ **Version note**: Tài liệu này chốt theo Kyverno **v1.19.0** (release 2026-08-20, hỗ trợ Kubernetes v1.33–v1.35), là bản stable mới nhất tại thời điểm viết, xác thực qua [kyverno.io/docs](https://kyverno.io/docs/installation/releases/) và [GitHub Releases](https://github.com/kyverno/kyverno/releases). Hệ thống mày sắp nhận bàn giao **gần như chắc chắn không chạy v1.19** (vì đây là bản vừa deprecate toàn bộ API cũ). Việc đầu tiên khi có quyền truy cập cluster: chạy `kubectl get pods -n kyverno -o jsonpath='{.items[0].spec.containers[0].image}'` hoặc `helm list -n kyverno` để biết version thật, rồi đối chiếu lại note này — đừng áp dụng 1:1 mà không kiểm tra.

---

### **1. What — Nó là cái gì?**

Kyverno là policy engine cho Kubernetes, chạy như một **dynamic admission controller** (validating + mutating webhook) kiêm CRD controller, cho phép viết policy bằng YAML thuần (không cần Rego như OPA/Gatekeeper). Từ v1.14 (04/2025) Kyverno bổ sung một họ policy type mới dùng **CEL** (Common Expression Language — cùng cơ chế với Kubernetes ValidatingAdmissionPolicy) song song với engine cũ dùng **JMESPath**; kể từ v1.19 (08/2026) họ CEL đã đạt full feature parity và họ cũ chính thức deprecated.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

Nếu không có Kyverno, mày phải chọn 1 trong các hướng đều tệ hơn:
- Tự viết admission webhook server bằng Go để validate/mutate — tốn effort, phải tự lo cert rotation, tự lo webhook registration.
- Dùng OPA/Gatekeeper — enforce được nhưng phải học Rego, và **không mutate/generate được**, chỉ validate.
- Dùng Kubernetes ValidatingAdmissionPolicy (native, GA từ 1.30) — không cần thêm pod, nhanh hơn, nhưng chỉ validate, không mutate/generate, và policy phức tạp (verify image signature, gọi external API để lấy context) không làm thuần bằng VAP được.

Kyverno lấp khoảng trống: validate + mutate + **generate** (tự sinh resource kèm theo, vd NetworkPolicy mặc định cho mọi Namespace mới) + **verify image signature** (cosign/Notary, chống supply-chain attack) + audit/enforce tách biệt qua Policy Reports — tất cả trong 1 engine, viết bằng YAML.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi:** cần enforce Pod Security Standards có customize; cần supply-chain policy (verifyImages/cosign/SBOM attestation); cần tự động sinh resource kèm sự kiện (namespace tạo mới → tự có ResourceQuota/NetworkPolicy); cần audit-trail (PolicyReport) tách biệt khỏi enforce.

**KHÔNG dùng khi:**
- Cluster rất lớn, cực nhạy latency admission — Kyverno thêm 1 network round-trip (webhook call), K8s VAP (built-in, không cần webhook pod) nhanh hơn cho case chỉ cần validate object schema/CEL đơn giản.
- Logic policy cần general-purpose code phức tạp (loop, external SDK) — nên viết webhook riêng, đừng gò ép vào JMESPath/CEL.
- Không có ai chịu trách nhiệm vận hành thêm 1 stateful-ish component có quyền gần cluster-admin — Kyverno's ServiceAccount là attack surface thật, xem mục 7.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
                          ┌─────────────────────────┐
   kubectl apply  ─────▶ │   kube-apiserver          │
                          │  (admission chain)        │
                          └──────────┬────────────────┘
                       MutatingWebhook│  ValidatingWebhook
                          (kyverno-svc:443, AdmissionReview, timeout mặc định 10s)
                                     ▼
                     ┌───────────────────────────────┐
                     │   Admission Controller (Pod)   │◀── KHÔNG có leader election
                     │   - webhook controller         │    cho webhook traffic
                     │   - cert renewer                │    (mọi replica đều serve
                     │   - engine (validate/mutate)    │     song song → scale ngang
                     └───────────────┬───────────────┘     tăng throughput thật)
                                     │ ghi AdmissionReport /
                                     │ UpdateRequest (generate, mutate-existing)
        ┌────────────────────────────┼─────────────────────────────┐
        ▼                            ▼                              ▼
┌───────────────────┐   ┌────────────────────────┐    ┌───────────────────────┐
│ Background         │   │ Reports Controller      │    │ Cleanup Controller     │
│ Controller         │   │ (có leader election →   │    │ (có leader election    │
│ (có leader election)│  │  chỉ 1 replica active)  │    │  cho webhook/cert;     │
│ xử lý generate/     │   │ gộp AdmissionReport +   │    │  là controller duy    │
│ mutate-existing +   │   │ BackgroundScanReport    │    │  nhất khác Admission  │
│ background scan    │   │ → PolicyReport /         │    │  Controller tự quản   │
│ (mặc định mỗi 1h)  │   │   ClusterPolicyReport    │    │  webhook riêng)        │
└───────────────────┘   └────────────────────────┘    └───────────────────────┘
```

Chi tiết controller + HA sizing → [[kyverno--architecture-controllers]]

### **5. How — Cơ chế hoạt động**

- **2 họ policy song song**: `kyverno.io/v1` `ClusterPolicy`/`Policy` (legacy, JMESPath) — **deprecated từ v1.19, dự kiến gỡ ở v1.20 (~11/2026)** — vs `policies.kyverno.io/v1` `ValidatingPolicy`/`MutatingPolicy`/`GeneratingPolicy`/`ImageValidatingPolicy`/`DeletingPolicy` (mới, CEL). Đây là fact quan trọng nhất cần nhớ khi đọc bất kỳ tài liệu/tutorial nào viết trước 08/2026. → [[kyverno--policy-types-cel-vs-legacy]]
- **Admission-time vs background**: request tạo/sửa resource → chặn/mutate ngay tại webhook (có thể block nếu `enforce` + `failurePolicy: Fail`). Resource **đã tồn tại từ trước** khi policy được tạo → chỉ được quét định kỳ (background scan), **không bao giờ tự bị block/xoá**, chỉ ghi vào report. → [[kyverno--background-scan-reports]]
- **PolicyException**: cơ chế miễn trừ 1 resource/rule cụ thể khỏi policy mà không sửa policy gốc — bề mặt tấn công độc lập, đã có CVE nghiêm trọng thực tế. → [[kyverno--policy-exceptions]]
- **Webhook + failurePolicy**: quyết định cluster fail-open hay fail-closed khi Kyverno pod chết — nguyên nhân phổ biến nhất của sự cố "apiserver bị khoá". → [[kyverno--webhooks-admission]]
- **Autogen**: viết policy nhắm `Pod`, Kyverno tự sinh thêm rule cho Deployment/StatefulSet/DaemonSet/Job/CronJob — tránh phải viết policy trùng lặp cho từng controller. Chỉ áp dụng cho legacy `ClusterPolicy`. → chi tiết trong [[kyverno--policy-types-cel-vs-legacy]]
- **CLI** để test policy offline (không cần cluster) trước khi apply, dùng trong CI/CD. → [[kyverno--cli-testing]]

### **6. Key Config — Cấu hình cần nhớ**

- `failurePolicy` mặc định **`Fail`** (fail-closed) — không phải `Ignore`. Nếu Kyverno pod chết hết + có policy match resource rộng (không loại trừ namespace hệ thống) → apiserver có thể bị khoá hoàn toàn, kể cả không tạo lại được pod Kyverno.
- `background: true` là default — bật quét định kỳ, **mặc định mỗi 1 giờ**, nhưng dù `validationFailureAction`/`failureAction` là Enforce, background scan **chỉ ghi report, không tự chặn/xoá** resource vi phạm đã tồn tại.
- Legacy dùng `validationFailureAction: audit|enforce` (chữ thường), type CEL mới dùng `failureAction: Audit|Enforce` (chữ hoa) — 2 field tên gần giống nhau, dễ gõ nhầm khi hệ thống đang có cả 2 loại policy song song trong giai đoạn migrate.
- `pod-policies.kyverno.io/autogen-controllers` annotation — default `DaemonSet,Deployment,Job,StatefulSet,CronJob`. Set rỗng (`""`) để tắt autogen nếu chỉ muốn áp đúng Pod.
- Helm values HA-critical: `admissionController.replicas` (khuyến nghị tối thiểu 3 — vì không có leader election, mỗi replica thêm là thêm throughput thật); `backgroundController/reportsController/cleanupController.replicas` (khuyến nghị ≥2 cho failover, dù tại 1 thời điểm chỉ 1 replica active do leader election).
- `--clientRateLimitQPS` / `--clientRateLimitBurst` (container flag) — default thấp, cluster nhiều namespace/policy dễ bị client-side throttle → biểu hiện: report tồn đọng, UpdateRequest không được xử lý → tăng lên 300–500 khi gặp.
- Default webhook `timeoutSeconds`: 10s, range hợp lệ 1–30s (K8s giới hạn cứng).

### **7. Security Considerations**

- **Attack surface chính**: ServiceAccount của admission controller thường có quyền rất rộng (để generate/mutate resource cross-namespace, cross-kind). Nếu attacker (hoặc user ít quyền hơn dự kiến) tạo được `Policy`/`PolicyException`/`NamespacedMutatingPolicy` trong namespace của họ, họ có thể lợi dụng `apiCall`, `context.configMap`, hoặc CEL `generator.apply()` để đọc/ghi resource **ngoài** namespace được cấp quyền — đây không phải giả thuyết, là chuỗi CVE đã xảy ra thật nhiều lần (CVE-2026-22039, incomplete-fix GHSA-cvq5-hhx3-f99p, CVE-2026-54523...). Chi tiết đầy đủ từng CVE → [[kyverno--security-cves-hardening]]
- **PolicyException misconfiguration = bypass thật**: mặc định PolicyException tạo được ở **bất kỳ namespace nào** (CVE-2024-48921) — user ít quyền có thể tự miễn trừ chính mình khỏi policy enforce. Nghiêm trọng hơn: 2 exception chồng lên nhau (1 chặt + 1 lỏng) khiến Kyverno áp cái lỏng hơn bất kể exception nào tạo trước — bypass hoàn toàn `enforce` (GHSA-gg4x-fgg2-h9w9, CVSS **critical**, ảnh hưởng v1.9.0–v1.12.7). → [[kyverno--policy-exceptions]]
- **Hardening tối thiểu**: (1) luôn chạy patch mới nhất trong minor đang dùng; (2) RBAC tách biệt: ai được tạo `Policy`/`PolicyException`/`*MutatingPolicy` phải là nhóm khác với dev thường triển khai app; (3) audit mọi policy dùng `apiCall`/`context.configMap`/CEL external context; (4) không dùng wildcard `*` trong `match`/exception, dùng tên cụ thể hoặc label selector; (5) loại trừ `kube-system` và namespace của chính Kyverno khỏi mọi policy match rộng để tránh tự khoá cluster.

### **8. Ops Runbook — Production Notes**

- **Health check**: `kubectl get pods -n kyverno`; `kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations -A | grep kyverno`; kiểm tra cert trong Secret `kyverno-svc.kyverno.svc.<n>.key-pair` còn hạn (cert renewer tự xoay, nhưng phải verify khi nghi ngờ).
- **Log quan trọng**: tăng verbosity `-v=4` để xem variable substitution khi policy không match như kỳ vọng; `-v=6` để debug rule-level chi tiết hơn.
- **Metric cần alert**: `kyverno_admission_review_duration_seconds` (p99 cao kéo dài >5 phút = dấu hiệu Kyverno đang làm chậm apiserver); số lượng `AdmissionReport`/`UpdateRequest`/`BackgroundScanReport` tồn đọng tăng liên tục = reports/background controller đang nghẽn (thường do client-side rate limit).
- **Sự cố nghiêm trọng nhất & cách gỡ**: apiserver bị khoá vì toàn bộ Kyverno pod down + `failurePolicy: Fail` + policy match rộng → xoá tạm `validatingwebhookconfiguration`/`mutatingwebhookconfiguration` của Kyverno để giải phóng apiserver → fix nguyên nhân Kyverno down → cài lại webhook (Kyverno tự đăng ký lại khi pod chạy trở lại nếu dùng Helm-managed webhook).
- Đầy đủ danh sách lỗi thường gặp + fix → [[kyverno--troubleshooting-runbook]]

### **9. Gotchas & Lessons Learned**

- **Bản lề version quan trọng nhất hiện tại**: `ClusterPolicy`/`Policy`/`CleanupPolicy`/legacy `PolicyException` (group `kyverno.io`) đã **deprecated kể từ v1.19 (2026-08-20)**, dự kiến **gỡ bỏ ở v1.20 (~11/2026)**. Gần như 100% tutorial/blog/course ôn CKS hiện có trên mạng viết theo style legacy này vì CEL parity chỉ vừa đạt ở v1.19. Đừng học theo bài viết cũ mà không đối chiếu — nhưng cũng đừng bỏ qua legacy syntax vì hệ thống thật sắp bàn giao gần như chắc chắn đang chạy nó.
- Background scan **không** retroactively enforce — dễ hiểu lầm rằng bật `background: true` + `Enforce` nghĩa là resource cũ vi phạm sẽ tự bị dọn; thực tế nó chỉ nằm trong report, vẫn chạy bình thường.
- `failurePolicy` mặc định `Fail`, không phải `Ignore` — khác với thói quen từ policy engine khác. Cài lần đầu trên cluster đã có traffic mà chưa test kỹ = rủi ro tự khoá cluster.
- (Mục này để tự bổ sung tiếp sau khi thao tác trực tiếp với hệ thống thật — lesson learned thực chiến quan trọng hơn bất kỳ note lý thuyết nào)

### **10. Resources**

- Docs chính thức (đang track v1.19): https://kyverno.io/docs/
- Release/support matrix: https://kyverno.io/docs/installation/releases/
- Release notes v1.19: https://kyverno.io/blog/2026/08/20/announcing-kyverno-release-1.19/
- Migration guide (legacy → CEL): https://kyverno.io/docs/guides/migration-to-cel/
- **Security advisories chính thức** (nguồn CVE xác thực, không phải blog SEO): https://github.com/kyverno/kyverno/security/advisories — dùng `curl -s https://api.github.com/repos/kyverno/kyverno/security-advisories` để lấy list gốc, tránh các trang "CVE blogspam" hay bịa CVE ID không tồn tại trong NVD/GHSA.
