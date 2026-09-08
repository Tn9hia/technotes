# Trivy — All-in-One Security Scanner (v0.74.0, 2026-08-14)
Tags: #trivy #vulnerability-scanning #sbom #supply-chain #kubernetes #cks #infra
Last updated: 2026-09-06

> Nguồn verify: [trivy.dev/docs](https://trivy.dev/latest/docs/), repo chính thức [github.com/aquasecurity/trivy](https://github.com/aquasecurity/trivy) (nhánh `main`), phiên bản stable mới nhất tại thời điểm viết là **v0.74.0** (14/08/2026, xem [releases](https://github.com/aquasecurity/trivy/releases)). Tất cả CLI flag/default value trong note này lấy trực tiếp từ `docs/guide/references/configuration/cli/trivy_image.md` trên nhánh `main` — nếu version mày dùng khác, chạy `trivy --version` và `trivy image --help` để tự đối chiếu trước khi tin theo note.

---

### **1. What — Nó là cái gì?**

Trivy (Aqua Security, CNCF project) là **all-in-one security scanner**: 1 binary duy nhất quét được vulnerability (CVE), misconfiguration (IaC), exposed secret, và license risk — trên nhiều loại target khác nhau (container image, filesystem, git repo, VM image, Kubernetes cluster, SBOM file). Khác với Falco (runtime/detective), Trivy là **static/preventive scanner**: quét *trước khi* deploy hoặc quét trạng thái tĩnh của cluster, không theo dõi hành vi realtime.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

- Trước Trivy, muốn phủ hết vuln OS package + language dependency (npm, pip, Maven, Go...) + Dockerfile misconfig + secret leak + license compliance, mày phải ghép nhiều tool riêng lẻ (Clair cho OS vuln, tfsec/checkov cho IaC, gitleaks cho secret, license-checker riêng...). Trivy gộp tất cả vào 1 binary, 1 output format thống nhất.
- Nếu không có Trivy: image build xong deploy thẳng, không ai biết bên trong có `openssl` version dính CVE critical hay Dockerfile chạy `USER root`, cho đến khi bị exploit hoặc audit compliance mới phát hiện.
- Trong **CKS (Certified Kubernetes Security Specialist)**: domain **Supply Chain Security chiếm ~20% đề thi** — nặng ngang **Monitoring/Logging & Runtime Security**. Trivy là **tool duy nhất có sẵn trong môi trường thi** để làm các bài image scanning, nên `trivy image`, `--severity`, `--ignore-unfixed`, `--exit-code` phải gõ theo phản xạ, không có thời gian tra docs giữa lúc thi.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi:**
- Cần scan image trước khi push lên registry hoặc trước khi admission vào cluster (CI/CD gate, hoặc kết hợp admission controller).
- Cần audit toàn bộ cluster đang chạy (image nào có CVE, RBAC nào quá rộng, config nào lệch CIS Benchmark) — dùng `trivy k8s` hoặc trivy-operator.
- Cần sinh SBOM (CycloneDX/SPDX) phục vụ compliance/supply-chain attestation.
- Cần quét IaC (Terraform/CloudFormation/Helm/K8s manifest) trước khi apply.

**KHÔNG dùng / cân nhắc kỹ khi:**
- Cần **runtime detection** (ai đó exec shell vào container, syscall bất thường lúc đang chạy) → đó là việc của Falco/Tetragon, không phải Trivy. Trivy quét xong là xong, không theo dõi tiếp.
- Môi trường **air-gapped hoàn toàn không có kế hoạch mirror DB nội bộ** → Trivy mặc định cần internet để tải vulnerability DB/Java DB/checks bundle từ OCI registry công khai (`mirror.gcr.io`, `ghcr.io`). Không chuẩn bị trước = scan fail hoặc chạy với DB cũ/checks embedded (misconfig scanner có fallback embedded checks, nhưng **vulnerability DB thì không có fallback** — không tải được DB thì không scan vuln được). → [[trivy--db-management]]
- Chỉ cần check nhanh 1 CVE cụ thể có ảnh hưởng package nào đó → dùng trực tiếp NVD/OSV tra cứu nhanh hơn là chạy full scan.
- Muốn **enforcement/block** ngay tại admission time dựa trên kết quả Trivy → bản thân `trivy` CLI/operator chỉ **report**, cần ghép thêm OPA Gatekeeper/Kyverno hoặc trivy-operator's compliance report + policy riêng để chặn.
- Pipeline coi kết quả scan là "pass" mặc định → xem gotcha **exit code mặc định là 0** ở mục 6, đây là lỗi rất phổ biến khi mới setup CI.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
                              ┌─────────────────────────────────────────────┐
                              │         External OCI Registries              │
                              │  (mirror.gcr.io ưu tiên, fallback ghcr.io)   │
                              │  - trivy-db (vuln DB, schema v2)             │
                              │  - trivy-java-db (Java index, schema v1)     │
                              │  - trivy-checks (misconfig policy bundle)    │
                              └───────────────────┬───────────────────────────┘
                                                  │ pull khi cần (cache theo TTL)
                                                  ▼
   ┌──────────────┐   ┌──────────────────────────────────────────────────┐
   │  Scan Target  │──▶│                  Trivy Engine                    │
   │ - Image       │   │  ┌────────────┐ ┌───────────────┐ ┌───────────┐ │
   │ - Filesystem  │   │  │ Vuln        │ │ Misconfig      │ │ Secret /  │ │
   │ - Git repo    │   │  │ Scanner     │ │ Scanner (Rego) │ │ License   │ │  → [[trivy--scanners]]
   │ - VM image    │   │  └────────────┘ └───────────────┘ └───────────┘ │
   │ - SBOM file   │   │              Local cache (BoltDB/memory/Redis)   │  → [[trivy--cache-performance]]
   │ - K8s cluster │   └───────────────────┬──────────────────────────────┘
   └──────────────┘                       │ filter theo severity/status/ignore  → [[trivy--filtering-suppression]]
                                           ▼
                              ┌────────────────────────────┐
                              │ Output: table/json/sarif/   │
                              │ cyclonedx/spdx/template      │
                              └───────────────┬────────────┘
                                              ▼
                     ┌───────────────────────────────────────────┐
                     │  CI/CD gate, SIEM, hoặc Kubernetes CRD      │
                     │  (trivy-operator → VulnerabilityReport,      │  → [[trivy--kubernetes-scanning]]
                     │  ConfigAuditReport, ExposedSecretReport...)  │
                     └───────────────────────────────────────────┘
```

- **Vị trí trong hệ thống**: có thể chạy như CLI đơn lẻ trong CI/CD runner, như `trivy server` (client/server mode, dùng chung DB cho nhiều client), hoặc như **Kubernetes Operator** (DaemonSet/Deployment chạy thường trực trong cluster, tự trigger scan khi có Pod mới).
- **Component chính**: scan target adapter (đọc image layer / filesystem / git repo) → 4 scanner engine (vuln, misconfig, secret, license) → local cache (DB + kết quả scan trước) → report formatter.
- **Traffic/data flow**: target → phân tích file/package → so khớp với DB/checks đã cache → filter theo policy (severity/ignore/rego/VEX) → xuất report.
- **Dependency**: cần Docker/containerd/podman socket hoặc quyền pull registry (để lấy image); cần internet ra `mirror.gcr.io`/`ghcr.io` (DB), `repo.maven.apache.org` (Java transitive dependency resolution), `check.trivy.dev` (version check/telemetry, tắt được); trong K8s mode cần RBAC `list` trên hầu hết resource.

### **5. How — Cơ chế hoạt động**

Core concepts quan trọng nhất:

1. **Trivy không tự chứa dữ liệu bảo mật — nó tải "database" về khi cần.** Binary Trivy chỉ là scan engine; 3 loại DB (`trivy-db`, `trivy-java-db`, `trivy-checks`) được đóng gói dưới dạng **OCI artifact**, publish lên container registry công khai, và Trivy tự pull/cache khi scan. Đây là điểm khác biệt lớn nhất so với tool kiểu "signature file đi kèm binary". → [[trivy--db-management]]
2. **4 scanner độc lập, bật/tắt qua `--scanners`.** Mặc định `trivy image` chỉ bật `vuln,secret` — **misconfig và license KHÔNG bật mặc định** (misconfig chỉ mặc định bật khi dùng subcommand `trivy config`). Đây là gotcha hay bị hiểu nhầm nhất khi mới dùng. → [[trivy--scanners]]
3. **Severity của 1 CVE không cố định — phụ thuộc vendor source.** Trivy ưu tiên đánh giá của OS vendor (Debian/RHEL/Alpine security team) hơn NVD, vì vendor biết chính xác họ backport fix ở version nào. Nếu vendor không có dữ liệu mới fallback CVSS/NVD. Vì vậy severity **có thể khác nhau** giữa 2 lần scan nếu nguồn dữ liệu underlying thay đổi.
4. **Filtering là pipeline nhiều tầng, không phải 1 flag.** Thứ tự: Severity → Status (`--ignore-status`/`--ignore-unfixed`) → Finding ID (`.trivyignore`) → Rego policy (`--ignore-policy`) → VEX. Hiểu sai thứ tự này dễ dẫn đến setup ignore rule mà tưởng không hoạt động. → [[trivy--filtering-suppression]]
5. **`trivy k8s` (CLI, on-demand) khác hẳn `trivy-operator` (K8s-native, continuous).** CLI chạy 1 lần rồi thoát; operator là controller chạy thường trực, watch Pod events, tự tạo CRD report (`VulnerabilityReport`, `ConfigAuditReport`...) truy vấn được qua `kubectl get`. → [[trivy--kubernetes-scanning]]

**Scan lifecycle (image scan)**: Trivy lấy image (Docker/containerd/podman socket hoặc pull trực tiếp từ registry theo thứ tự `--image-src`) → giải nén từng layer → phát hiện OS + package manager (apk/dpkg/rpm) và file ngôn ngữ (package-lock.json, go.sum, requirements.txt...) → so khớp version cài đặt với DB đã cache → (nếu bật) chạy Rego check cho Dockerfile/K8s manifest tìm thấy trong image → (nếu bật) quét plaintext tìm secret pattern → filter kết quả theo severity/ignore → format output.

### **6. Key Config — Cấu hình cần nhớ**

- **Default scanners của `trivy image` là `[vuln, secret]`** — không có `misconfig`, không có `license`. Muốn quét đủ 4 loại: `trivy image --scanners vuln,misconfig,secret,license`.
- **Default exit code là 0** dù tìm thấy vuln CRITICAL — đây là **gotcha số 1 khi build CI gate**. Phải tự set `--exit-code 1` (thường kèm `--severity CRITICAL,HIGH`) thì pipeline mới fail đúng lúc. Không set = CI luôn xanh dù image đầy lỗ hổng.
- **Default timeout là `5m0s`** (`--timeout`). Scan image có nhiều JAR/Java library thường timeout ở default này — bump lên `15m` là khuyến nghị chính thức trong troubleshooting docs.
- **Default DB repository từ v0.57.1+**: `mirror.gcr.io/aquasec/trivy-db:2` ưu tiên trước, `ghcr.io/aquasecurity/trivy-db:2` là fallback (đổi thứ tự này sau sự cố rate-limit GHCR — xem mục 9). Đừng cấu hình `--db-repository` đè lên mà quên giữ lại 2 giá trị default nếu chỉ muốn *thêm* mirror riêng.
- **`--cache-backend` mặc định là `fs` (BoltDB) cho `image`/`vm`/`repo`, nhưng mặc định là `memory` cho `fs`/`rootfs`/`config`/`sbom`.** Filesystem cache dùng BoltDB **chỉ cho phép 1 process truy cập cùng lúc** — chạy 2 `trivy image` song song cùng `--cache-dir` sẽ bị lock/hang, không phải bug. → [[trivy--cache-performance]]
- **`--ignore-unfixed` là shorthand nguy hiểm nếu không hiểu rõ**: nó = `--ignore-status affected,will_not_fix,fix_deferred,end_of_life`, nghĩa là **ẩn luôn cả vuln chưa có bản vá** (`affected`), không chỉ ẩn `will_not_fix`. Dùng compliance report mà bật flag này có thể vô tình che mất rủi ro thật sự đang tồn tại.
- **`.trivyignore.yaml` vẫn là EXPERIMENTAL** (tính đến v0.74.0) và **phải khai báo tường minh qua `--ignorefile`**, không tự động load như `.trivyignore` thường. Dễ nhầm là "ignore không hoạt động" vì quên flag này.
- File config `trivy.yaml` (`--config`, default tìm ở thư mục hiện tại) cho phép set mọi flag dưới dạng YAML — hữu ích để version-control policy scan thay vì gõ flag dài trong CI script.

### **7. Security Considerations**

- **Trivy là scanner, không phải control** — kết quả scan chỉ có giá trị nếu có quy trình xử lý theo sau (block deploy, ticket vá lỗi). Chạy Trivy mà không ai đọc report = false sense of security.
- **False negative do DB lag**: CVE mới công bố cần thời gian để vào được `trivy-db` (build hàng ngày từ nhiều feed). Scan "sạch" hôm nay không đồng nghĩa image an toàn tuyệt đối — đặc biệt với zero-day mới công bố trong ngày.
- **`--insecure` và `TRIVY_INSECURE=true`** bỏ qua xác thực TLS certificate khi pull từ registry — chỉ dùng tạm cho debug, tuyệt đối không để trong pipeline production vì mở đường cho MITM khi pull image/DB.
- **`--trace-http`** dùng để debug network/auth issue nhưng **có thể lộ thông tin nhạy cảm trong HTTP header/body** dù Trivy có cố redact — chính docs cảnh báo "Never use this flag in production environments or CI/CD pipelines", và Trivy tự động disable flag này khi phát hiện đang chạy trong CI.
- **Secret scanner dựa trên pattern/regex** — không phải phân tích entropy sâu như một số tool chuyên dụng, nên có thể miss custom secret format hoặc match nhầm allow-list path (`builtin-allow` rules) khiến secret thật bị bỏ qua vì path trùng với rule loại trừ mặc định.
- **RBAC cho `trivy k8s`/trivy-operator đòi hỏi `list` trên gần như toàn bộ resource** (core, apps, batch, networking, rbac API group) — ServiceAccount chạy Trivy trong cluster là mục tiêu hấp dẫn nếu bị compromise (đọc được Secret metadata, RoleBinding...). Hạn chế namespace bằng `--include-namespaces`/`--exclude-namespaces` nếu không cần quét toàn cluster.

**Hardening checklist tối thiểu:**
- [ ] Set `--exit-code` + `--severity` rõ ràng trong mọi CI pipeline — đừng tin exit code mặc định
- [ ] Định kỳ review `.trivyignore`/`.trivyignore.yaml` có `expired_at`/`exp:` — ignore vĩnh viễn không review lại là nợ kỹ thuật bảo mật
- [ ] Không dùng `--insecure`/`TRIVY_INSECURE` ngoài môi trường debug cục bộ
- [ ] Giới hạn RBAC ServiceAccount của trivy-operator theo namespace nếu không cần audit toàn cluster
- [ ] Pin version Trivy CLI trong CI và có kế hoạch upgrade định kỳ — DB schema có vòng đời deprecation (xem mục 9)
- [ ] Không commit `GITHUB_TOKEN`/registry credential vào log khi debug (`--trace-http` chỉ dùng local, không dùng trong CI)

### **8. Ops Runbook — Production Notes**

- **Health check nhanh**: `trivy --version` (kiểm tra version + build metadata), `trivy image --download-db-only` (test riêng khả năng tải DB mà không chạy full scan — cách ly lỗi network vs lỗi scan).
- **Log/error quan trọng cần biết mặt**:
  - `FATAL failed to download vulnerability DB` → thường là firewall chặn `mirror.gcr.io`/`ghcr.io`, xem [[trivy--db-management]]
  - `DENIED: denied` khi pull từ `ghcr.io` → GHCR token cục bộ hết hạn, chạy `docker logout ghcr.io` hoặc `unset GITHUB_TOKEN`
  - `--skip-update cannot be specified with the old DB schema` → binary Trivy quá cũ so với DB schema hiện hành, phải upgrade CLI
  - `cache may be in use by another process` → BoltDB file-lock do chạy scan song song trên cùng `--cache-dir`, xem [[trivy--cache-performance]]
  - `analyze error: timeout: context deadline exceeded` → tăng `--timeout`, đặc biệt image nhiều Java
  - HTTP `429 Too Many Requests` khi scan Java project → Maven Central rate limit, không phải lỗi Trivy — xem [[trivy--troubleshooting-runbook]]
- **Metric/behavior cần theo dõi trong CI ở quy mô lớn**: tần suất pull DB (mỗi job pull riêng = tốn băng thông + dễ dính rate limit) → cân nhắc chạy `trivy server` trung tâm hoặc cache DB layer riêng (image Docker có sẵn DB, refresh định kỳ) thay vì để mỗi runner tự tải.
- **Restart/rollback**: Trivy CLI stateless, không có khái niệm rollback runtime. Với `trivy server`, restart an toàn nếu dùng Redis cache backend (state không mất); nếu dùng filesystem cache, restart có thể cần re-warm DB.
- **Trước khi rollout Trivy CLI version mới trong CI**: đọc release notes để check breaking change (theo `compatibility.md`, breaking change được công bố rõ trong release notes) và check DB schema có bị đổi không.
- Chi tiết troubleshooting từng lỗi cụ thể → [[trivy--troubleshooting-runbook]]

### **9. Gotchas & Lessons Learned**

> ⚠️ Phần dưới đây tổng hợp từ GitHub Issues/Discussions chính thức của `aquasecurity/trivy` (đã verify link, không phải suy đoán) — nhưng vẫn là kinh nghiệm cộng đồng, chưa phải kinh nghiệm vận hành thật của mày trên hệ thống sắp bàn giao. **Cập nhật lại sau khi vận hành thực tế.**

- **Sự cố rate-limit GHCR (cuối 2024 – đầu 2025) là bài học lớn nhất về vận hành Trivy ở quy mô lớn**: tải sinh ra từ cộng đồng dùng Trivy vượt rate limit của GHCR namespace (được ghi nhận vượt mốc hàng chục nghìn request/phút), khiến DB download fail hàng loạt trong CI trên toàn thế giới. Aqua đã: (1) đổi default registry ưu tiên sang `mirror.gcr.io` (không rate-limit vì là mirror Docker Hub qua Google) kể từ **v0.57.1**, dùng `ghcr.io` làm fallback; (2) giảm tần suất build/publish DB mới từ **mỗi 6 giờ xuống mỗi 24 giờ** để giảm tải. Nếu hệ thống mày pin Trivy version cũ hơn v0.57.1 và có nhiều pipeline chạy song song, đây gần như chắc chắn là nguyên nhân khi thấy lỗi tải DB hàng loạt cùng lúc. Nguồn: [Issue #7938](https://github.com/aquasecurity/trivy/issues/7938), [Discussion #8009](https://github.com/aquasecurity/trivy/discussions/8009).
- **`GITHUB_TOKEN` chỉ giúp rate-limit của GitHub API (dùng cho VEX repo), KHÔNG giúp gì cho rate-limit tải vulnerability DB/Java DB/checks bundle** — đây là nhầm lẫn phổ biến, vì DB được phân phối qua OCI registry (GHCR dùng cơ chế token riêng cho container registry, khác API token thông thường). Xem thẳng [Discussion #8009](https://github.com/aquasecurity/trivy/discussions/8009) để tránh set nhầm token mà tưởng đã fix.
- **Exit code mặc định = 0 dù có CRITICAL CVE** là gotcha khiến rất nhiều pipeline "tưởng đã có Trivy gate" nhưng thực chất không chặn gì cả — phải review lại mọi pipeline hiện có xem đã set `--exit-code` đúng nghĩa chưa, đừng giả định người viết CI trước đó đã làm đúng.
- **DB schema có vòng đời riêng, tách khỏi vòng đời CLI**: theo chính sách compatibility chính thức, khi một schema version của `trivy-db`/`trivy-java-db`/`trivy-checks` bị ngừng cập nhật, các bản Trivy CLI cũ phụ thuộc schema đó sẽ **không còn nhận vulnerability data mới** dù binary vẫn chạy bình thường và không báo lỗi rõ ràng — đây là kiểu "âm thầm mù" giống driver Falco chết sau kernel upgrade. Aqua cam kết công bố trước ngày EOL và version tối thiểu cần upgrade, nhưng **không ai tự động nhắc mày** trừ khi theo dõi release notes.
- **Java DB chỉ tải khi cần** (scan JAR/WAR/EAR), nên nhiều người setup `trivy server` xong ngạc nhiên thấy log chỉ pull `trivy-db` mà không thấy `trivy-java-db` — không phải bug, chỉ là chưa scan image nào có Java artifact ([Discussion #7880](https://github.com/aquasecurity/trivy/discussions/7880)).
- **Misconfiguration checks bundle (`trivy-checks`) có fallback embedded trong binary** nếu không tải được từ registry — nhưng **vulnerability DB thì không có fallback tương đương**. Trong air-gapped/network-restricted environment, đừng nhầm lẫn 2 loại DB này có cùng mức độ "chịu lỗi".
- **`--ignore-unfixed` dễ bị hiểu sai phạm vi**: nhiều người dùng nó tưởng chỉ ẩn "chưa có ai định vá (`will_not_fix`)" nhưng thực tế ẩn cả `affected` (đang bị ảnh hưởng, chưa có patch) — nghĩa là **ẩn luôn rủi ro thật đang tồn tại**, không chỉ rủi ro đã được vendor quyết định không vá.

### **10. Resources**

- Official docs: https://trivy.dev/latest/docs/
- Repo chính: https://github.com/aquasecurity/trivy
- Release notes: https://github.com/aquasecurity/trivy/releases
- Database repos: https://github.com/aquasecurity/trivy-db, https://github.com/aquasecurity/trivy-java-db, https://github.com/aquasecurity/trivy-checks
- Trivy Operator (K8s continuous scanning): https://aquasecurity.github.io/trivy-operator/latest/ , repo https://github.com/aquasecurity/trivy-operator
- Trivy Action (GitHub Actions integration): https://github.com/aquasecurity/trivy-action
- Aqua Vulnerability Database (tra CVE theo ID Trivy report): https://avd.aquasec.com
- CKS Supply Chain Security domain (tỷ trọng ~20% đề thi, Trivy là tool có sẵn trong môi trường thi) — luyện tập qua killercoda/killer.sh CKS simulator
