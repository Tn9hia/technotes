# Falco — Runtime Security & Threat Detection (v0.4x)
Tags: #falco #runtime-security #kubernetes #ebpf #cks #infra
Last updated: 2026-09-05

---

### **1. What — Nó là cái gì?**

Falco (CNCF Graduated project) là công cụ **runtime security / threat detection** cho Linux host và container: nó bắt syscall + K8s audit log realtime, so khớp với rule engine, và bắn alert khi phát hiện hành vi bất thường (shell trong container, ghi vào `/etc/shadow`, reverse shell, privilege escalation...). Nghĩ đơn giản: **Falco = IDS chạy ở tầng kernel/syscall cho container workload**, khác hẳn với vulnerability scanner (Trivy) vốn chỉ soi image *trước khi* deploy.

### **2. Why — Tại sao tồn tại / Vấn đề nó giải quyết?**

- Static scanning (Trivy, image signing, admission control) chỉ chặn được cái *biết trước*. Nó không thấy được zero-day exploit, lateral movement, hay insider chạy `kubectl exec` rồi curl reverse shell — những thứ chỉ lộ ra **lúc runtime**.
- Nếu không có Falco: chỉ còn network policy + RBAC + admission controller — toàn bộ đều là **preventive control**, không có **detective control** ở tầng syscall. Khi attacker đã vượt qua được lớp preventive (RBAC lỏng, image bị compromise), không ai biết cho đến khi thiệt hại xảy ra.
- Falco thay thế phần việc mà nếu không có nó, mày phải tự viết auditd rules + eBPF program tay, hoặc mua EDR thương mại (Sysdig Secure, Falcosecurity chính là nguồn gốc mở của việc này).
- Trong CKS: đây chính là domain "Monitoring, Logging and Runtime Security" — câu hỏi kiểu "viết rule Falco để detect ai đó exec shell vào container" là dạng bài chuẩn.

### **3. When — Dùng khi nào / KHÔNG dùng khi nào?**

**Dùng khi:**
- Cần audit trail hành vi ở mức syscall (exec, open, connect, chmod, mount...)
- Compliance yêu cầu runtime detection (PCI-DSS, SOC2, hoặc chuẩn bị CKS)
- Muốn detect: container drift (binary mới xuất hiện sau khi container start), privilege escalation, crypto miner, unexpected outbound connection, sensitive file access

**KHÔNG dùng / cân nhắc kỹ khi:**
- Cần deep packet inspection / phân tích payload network → Falco chỉ thấy `connect()`/`accept()` ở mức syscall, không đọc nội dung packet. Dùng Cilium/Tetragon hoặc NIDS riêng.
- Compliance chỉ cần static scanning → đừng cõng thêm Falco nếu team chưa có ai đọc/tune alert. **Alert không ai xử lý = log rác tốn CPU**.
- Cần **chặn (block) ngay lập tức**, không chỉ log → Falco mặc định là **detective, không phải preventive**. Muốn enforcement thật sự cần thêm response engine ([falco-talon](https://github.com/falcosecurity/falco-talon)) hoặc cân nhắc Tetragon (có khả năng block syscall trực tiếp qua eBPF).
- Node có kernel rất cũ (< 4.14 cho eBPF, hoặc kernel không có headers cho kernel-module) → driver có thể không load được, xem [[falco--drivers]].
- Cluster resource rất hạn chế và không có budget cho tuning liên tục → false positive rate ban đầu khá cao nếu dùng nguyên rule mặc định.

### **4. Where - Architecture — Nó nằm ở đâu trong hệ thống?**

```
┌─────────────────────────────── Linux Node / K8s Node ────────────────────────────────┐
│                                                                                        │
│  ┌────────────┐     syscalls      ┌──────────────────────────────┐                   │
│  │ Container A │──────────────────▶│  Driver (chọn 1):             │                   │
│  ├────────────┤                   │  - Kernel module (.ko)        │  → [[falco--drivers]]
│  │ Container B │──────────────────▶│  - Legacy eBPF probe          │                   │
│  ├────────────┤                   │  - Modern eBPF (CO-RE, default)│                   │
│  │ Host process│──────────────────▶│                                │                   │
│  └────────────┘                   └───────────────┬────────────────┘                   │
│                                                    │ scap event (shared ring buffer)     │
│                                                    ▼                                    │
│                                   ┌────────────────────────────────┐                    │
│                                   │  Falco Userspace Engine          │                    │
│                                   │  - libscap (capture)             │                    │
│                                   │  - libsinsp (state/enrichment)   │                    │
│                                   │  - Rule engine (YAML)            │  → [[falco--rules-syntax]]
│                                   │  - Plugin framework              │  → [[falco--falcoctl-plugins]]
│                                   └───────────────┬────────────────┘                    │
│                                                    │ enrich container/pod metadata        │
│                                          (containerd/docker socket, k8s API)             │
│                                                    │ match rule → format alert            │
│                                                    ▼                                     │
│                                   ┌────────────────────────────────┐                    │
│                                   │  Outputs: stdout/file/syslog/    │  → [[falco--outputs-falcosidekick]]
│                                   │  gRPC API / HTTP webhook         │                    │
│                                   └───────────────┬────────────────┘                    │
└────────────────────────────────────────────────────┼─────────────────────────────────────┘
                                                       ▼
                                     ┌────────────────────────────┐
                                     │  Falcosidekick               │
                                     │  → Slack/ES/S3/SIEM/Loki...  │
                                     └──────────────┬─────────────┘
                                                     ▼
                                     ┌────────────────────────────┐
                                     │  falco-talon (response,      │
                                     │  optional — enforcement)     │
                                     └────────────────────────────┘
```

- **Vị trí trong hệ thống**: chạy như **DaemonSet** trên mỗi K8s node (1 pod/node, để bắt syscall của toàn bộ container trên node đó), hoặc `systemd` service trên VM/bare-metal.
- **Component chính**: driver (kernel visibility) → libscap/libsinsp (capture + state tracking process/container) → rule engine → plugin framework (mở rộng nguồn event, vd K8s audit log) → output channel.
- **Traffic/data flow**: syscall → driver → ring buffer (zero-copy, tránh overhead syscall) → userspace parse & enrich → evaluate rule → alert.
- **Dependency**: cần quyền truy cập kernel (privileged container hoặc capability cụ thể tùy driver), mount `/proc`, mount container runtime socket (containerd.sock/docker.sock) để lấy metadata container, tùy chọn kết nối K8s API để enrich pod/namespace label.

### **5. How — Cơ chế hoạt động**

Core concepts quan trọng nhất:

1. **Driver = "mắt" của Falco.** Quyết định compatibility với kernel, performance overhead, và attack surface. 3 loại: kernel module, legacy eBPF, modern eBPF (mặc định từ **Falco 0.35+**, dùng CO-RE nên không cần build lại theo từng kernel version). → [[falco--drivers]]
2. **Rule = condition (cú pháp filter kiểu Sysdig) + output template + priority.** File rule load theo thứ tự, rule cùng tên ở file sau sẽ **override** rule trước — đây là cơ chế để customize mà không sửa trực tiếp rule mặc định. → [[falco--rules-syntax]]
3. **Plugin framework**: từ v0.36 Falco chuyển hướng "plugin-first" — **source plugin** sinh event mới ngoài syscall (k8saudit, cloudtrail, okta...), **extractor plugin** thêm field để filter trên event có sẵn. Đây là cách Falco mở rộng ra ngoài Linux syscall (vd AWS CloudTrail, K8s Audit Log). → [[falco--falcoctl-plugins]]
4. **State engine (libsinsp) giữ context, không nhìn syscall đơn lẻ.** Falco track process tree, container metadata, file descriptor — nên rule có thể check "process cha là gì" (`proc.pname`), "image là gì" (`container.image.repository`), không chỉ 1 syscall trơ trọi.
5. **Falco chỉ detect, không tự chặn** (out-of-band, đọc event *sau khi* syscall đã xảy ra qua ring buffer). Muốn response tự động (kill pod, cordon node) cần thêm [falco-talon](https://github.com/falcosecurity/falco-talon) hoặc custom `program_output`/exec script.

**Request lifecycle**: process gọi syscall → driver bắt event → đẩy vào ring buffer (shared memory) → Falco userspace đọc buffer → enrich metadata (container id qua CRI socket, pod name/namespace qua K8s metadata fetcher nếu enable) → evaluate theo rule engine đã compile thành filter tree → nếu match và không rơi vào `exceptions` → format theo `output:` template → gửi tới output channel đang enable (có thể nhiều channel cùng lúc).

### **6. Key Config — Cấu hình cần nhớ**

- **Đừng sửa trực tiếp `falco_rules.yaml`** (rule mặc định) — sẽ bị ghi đè mất khi upgrade Falco/update rule package qua falcoctl. Luôn override qua rule file riêng (`falco_rules.local.yaml` hoặc custom file) dùng `override:` section. → [[falco--rules-syntax]]
- `priority` threshold trong `falco.yaml` (`priority: debug` default output level) — set quá thấp (informational/debug) khi go-live production = log flood, không ai đọc nổi.
- `outputs.rate` / `outputs.max_burst` — rate-limit alert, default có thể **âm thầm drop alert** khi bị flood (thấy log "rate limiting" tưởng nhầm là bug mất alert).
- `syscall_event_drops` threshold — cảnh báo khi ring buffer đầy và Falco **bỏ qua event** (không phải rule không match, mà là **không thấy** event luôn). Đây là config hay bị bỏ quên nhất — không alert cái này thì có thể "mù" mà không biết.
- `metadata_download` (k8s metadata fetcher) timeout — misconfigure gây pod k8saudit không enrich được `k8s.pod.name`, alert lên thiếu context.
- Resource limit của Falco DaemonSet pod: mặc định trong Helm chart tương đối thấp, dễ OOMKilled khi node có traffic syscall cao (build server, batch job) → mất giám sát cả node mà **không có cảnh báo rõ ràng** nếu không set liveness/PodDisruptionBudget đúng. → [[falco--performance-tuning-drops]]
- `grpc.enabled` / `grpc_output.enabled` không set TLS mặc định là **disabled with plaintext option** ở một số bản cũ — cần verify enable mTLS trước khi expose ra ngoài pod network.

### **7. Security Considerations**

- **Falco daemon chạy privileged** (cần host-level syscall visibility) → chính nó là mục tiêu hấp dẫn: attacker chiếm được 1 container có quyền cao có thể cố gắng tắt/patch Falco pod hoặc sửa ConfigMap rule để evade detection. Cần RBAC hạn chế ai được sửa Falco DaemonSet/ConfigMap/ServiceAccount.
- **gRPC/HTTP output** nếu không có mTLS = lộ toàn bộ alert stream (chứa command line, env var, path — nhiều khi có secret dính trong đó) cho bất kỳ ai reach được endpoint trong cluster network.
- **Kernel module driver** tăng attack surface kernel thật sự (loadable kernel module có bug = kernel panic/exploit). **Modern eBPF** an toàn hơn vì phải pass qua eBPF verifier của kernel trước khi load. Ưu tiên modern eBPF trừ khi có lý do đặc biệt.
- **Rule bypass**: attacker biết rule default dựa nhiều vào path/binary name (`/bin/bash`, `/bin/sh`) có thể copy binary sang tên khác để evade. Rule tốt nên dựa vào **behavior pattern** (syscall sequence, syscall trong container không nằm trong baseline image) thay vì chỉ match tên file.
- Output gửi ra ngoài (Slack webhook, HTTP sink) không mã hoá / không kiểm soát nội dung = rò rỉ thông tin nhạy cảm qua log bên thứ ba.

**Hardening checklist tối thiểu:**
- [ ] Ưu tiên **modern eBPF** driver thay vì kernel module (giảm attack surface kernel)
- [ ] RBAC chặt cho ConfigMap/rule file + ServiceAccount của Falco — không cho user thường sửa
- [ ] Self-monitoring: alert khi Falco pod crash/restart hoặc `syscall_event_drops` tăng cao
- [ ] mTLS bắt buộc cho gRPC/HTTP output nếu expose ngoài localhost
- [ ] Audit quyền `kubectl exec`/node SSH access — đây là vector để tắt driver hoặc kill process Falco
- [ ] Rule không chỉ match theo path/binary name — kết hợp behavior-based detection
- [ ] Rotate/giới hạn quyền ServiceAccount Falco: chỉ cần read pod/node info (get/list/watch), tuyệt đối không cần quyền write

### **8. Ops Runbook — Production Notes**

- **Health check**: `kubectl get pods -n falco` (mọi node phải có 1 pod Running), log phải có dòng `Falco initialized with configuration file` không kèm driver load error.
- **Log quan trọng cần monitor**:
  - `Unable to load the driver` / `bpf_probe_load` failed → driver không tương thích kernel (thường sau khi node upgrade kernel, xem [[falco--drivers]])
  - `Error reading configuration file` / rule YAML parse error → Falco sẽ **crash loop ngay khi start**, không chạy nửa vời
  - `Syscall event drop` / `n_drops` tăng → hệ thống quá tải hoặc buffer quá nhỏ, xem [[falco--performance-tuning-drops]]
- **Metric cần alert** (expose qua `metrics.enabled: true` hoặc qua falcosidekick → Prometheus):
  - `falco_event_drops_total` — spike bất thường = đang "mù" một phần
  - Số alert theo priority (đặc biệt CRITICAL/EMERGENCY) — spike bất thường cần review ngay
  - Pod restart count / CrashLoopBackOff của DaemonSet
  - Driver load failure ngay sau khi node kernel được patch (common risk point, cần checklist riêng khi có kernel upgrade)
- **Restart/rollback**: Falco stateless nên restart pod an toàn, nhưng rolling restart cả DaemonSet tạo **gap giám sát tạm thời** trên từng node — set `maxUnavailable` hợp lý, tránh restart toàn bộ cluster cùng lúc.
- **Trước khi apply rule mới lên production**: luôn `falco --validate <rule-file>` offline trước, và có thể chạy `falco -r <file> -A` để liệt kê toàn bộ rule đang active (kiểm tra không vô tình disable rule quan trọng).
- Chi tiết quy trình deploy/upgrade trên K8s → [[falco--kubernetes-deployment]]

### **9. Gotchas & Lessons Learned**

> ⚠️ Phần này là baseline tổng hợp từ docs/cộng đồng (chưa phải kinh nghiệm vận hành thật của mày) — **verify lại và cập nhật** sau khi bàn giao hệ thống thực tế.

- **Kernel upgrade là rủi ro số 1 với kernel-module/legacy eBPF driver**: node auto-patch kernel (unattended-upgrades, managed K8s node image rotation) → driver cũ không load được → Falco "chết âm thầm" nếu không có alert riêng cho driver load failure. Modern eBPF (CO-RE) giảm rủi ro này đáng kể nhưng không phải 100% miễn nhiễm (vẫn cần BTF info, một số distro custom kernel thiếu BTF).
- **`append: true` đã deprecated từ Falco 0.36**, sẽ bị xoá ở 1.0.0 — dùng `override:` section (`append`/`replace` theo key) thay vì field `append: true` cũ khi viết custom rule mới, để tránh phải sửa lại toàn bộ khi upgrade.
- **Rule mặc định khá noisy lúc mới bật** — nhiều rule (`Terminal shell in container`, `Write below etc`) trigger cả với hoạt động hợp lệ (debug pod, init container). Cần whitelist qua `exceptions:` hoặc macro thay vì tắt hẳn rule.
- **OOMKilled Falco pod = mất giám sát toàn bộ node mà không có gì báo động rõ ràng** nếu không set alert riêng cho việc pod bị restart — dễ bị bỏ sót vì Falco tự khởi động lại (K8s tự restart), nhìn ngoài tưởng "vẫn chạy bình thường".
- **gRPC output để chạy plaintext (không TLS)** ở default config của một số version — cần double-check khi enable, đừng để mặc định lộ endpoint trong cluster.
- **Rule order & override rất dễ nhầm** khi có nhiều file rule (default + local + custom): rule trùng tên ở file load sau sẽ override — nếu không kiểm tra kỹ thứ tự load trong `falco.yaml` (`rules_file:` list), rất dễ vô tình tắt rule quan trọng mà không biết.
- **k8saudit plugin cần K8s Audit Webhook Backend được cấu hình đúng ở API server** — nếu chỉ deploy Falco mà quên cấu hình audit policy/webhook ở control plane, Falco sẽ không nhận được audit event nào dù pod chạy khoẻ mạnh (dễ nhầm là Falco "không hoạt động"). → [[falco--kubernetes-deployment]]

### **10. Resources**

- Official docs: https://falco.org/docs/
- Rule overriding chính thức: https://falco.org/docs/concepts/rules/overriding/
- Repo chính: https://github.com/falcosecurity/falco
- Plugin registry: https://github.com/falcosecurity/plugins
- Falcoctl (quản lý rule/plugin qua OCI artifact): https://github.com/falcosecurity/falcoctl
- Falcosidekick (output routing): https://github.com/falcosecurity/falcosidekick
- Falco-talon (response engine, enforcement): https://github.com/falcosecurity/falco-talon
- CKS practice: killercoda Falco scenarios, killer.sh CKS simulator (có bài liên quan runtime security)
