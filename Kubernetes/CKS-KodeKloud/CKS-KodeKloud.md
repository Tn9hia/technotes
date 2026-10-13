---
title: CKS — KodeKloud Course Notes
tags:
  - kubernetes
  - cks
  - security
  - kodekloud
  - index
date: 2026-08-19
---

# CKS — KodeKloud Course Notes

> Ghi chú đầy đủ cho **Certified Kubernetes Security Specialist (CKS)** — gộp từ khóa KodeKloud (qua MCP server `notes-kodekloud`) **và** toàn bộ nội dung của folder `CKS/` cũ (đã merge vào đây, không còn tách riêng nữa). Folder này giờ là bản duy nhất cần đọc — `CKS/` có thể xoá.

**CKS Exam:** 2h, open-book, hands-on clusters. Prerequisite: active CKA.

## Domain theo trọng số

| File                                            | CKS Domain                                                                                          | Weight    | Sessions |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------- | --------- | -------- |
| [[Understanding the Kubernetes Attack Surface]] | Nền tảng — 4C's model, demo tấn công thực tế, kiến trúc AuthN/AuthZ/Admission, checklist production | —         | 3/3      |
| [[Cluster Setup and Hardening]]                 | Cluster Setup + Cluster Hardening                                                                   | 15% + 15% | 32/32    |
| [[System Hardening]]                            | System Hardening                                                                                    | 10%       | 18/18    |
| [[Minimize Microservice Vulnerabilities]]       | Minimize Microservice Vulnerabilities                                                               | 20%       | 31/31    |
| [[Supply Chain Security]]                       | Supply Chain Security                                                                               | 20%       | 7/11     |
| [[Monitoring Logging and Runtime Security]]     | Monitoring, Logging and Runtime Security                                                            | 20%       | 0/7      |


## Nội dung từng file

- **[[Understanding the Kubernetes Attack Surface]]** — mô hình 4C's (Cloud, Cluster, Container, Code), demo tấn công vote app từ Docker port 2375 hở → privileged container → Dirty COW → Dashboard NodePort lộ → DB credentials trong env var; kiến trúc AuthN → AuthZ → Admission Control; bảng attack-vector/mitigation; checklist production đầy đủ.
- **[[Cluster Setup and Hardening]]** — file lớn: TLS/PKI, Authentication/Authorization/RBAC/ServiceAccount/KubeConfig (+ truy vết user/group qua cert CN/O, script audit RBAC, Aggregated ClusterRoles, Dashboard Security), Kubelet security, node metadata protection, NetworkPolicy nâng cao (default-deny, multi-tier, cross-namespace, AND/OR selector), Ingress TLS + cert-manager + mTLS, Docker daemon security, audit logging (+ query cheat-sheet, ship logs ra Fluent Bit/Splunk), CIS Benchmark/kube-bench, cluster upgrade process.
- **[[System Hardening]]** — least privilege, Linux capabilities & privilege escalation (+ capsh decode), Seccomp (+ Security Profiles Operator), AppArmor (+ so sánh với Seccomp), User Namespace (K8s 1.30+), host/OS hardening (open ports, UFW, SSH, obsolete packages), restrict kernel modules, minimize IAM roles.
- **[[Minimize Microservice Vulnerabilities]]** — file lớn nhất: multi-tenancy & isolation, container sandboxing (gVisor/Kata + install steps + decision table), admission controllers, Policy Engines (OPA/Gatekeeper **và Kyverno** — validate/mutate/generate), Pod Security Admission (PSA) + lịch sử PSP + bảng so sánh PSA/OPA/Kyverno, Security Contexts, secrets management đầy đủ (native Secret, encryption at rest, **Sealed Secrets, External Secrets Operator, HashiCorp Vault** Agent Injector & dynamic secrets), pod-to-pod mTLS (Istio PeerAuthentication/AuthorizationPolicy), Cilium (CNI/eBPF + Hubble + L7 policy), resource quotas & QoS.
- **[[Supply Chain Security]]** — image security & minimal base images, **image signing với Cosign** (keyless/key-based + CI pipeline), vulnerability scanning (Trivy, KubeLinter), SBOM (SPDX/CycloneDX, Syft/Grype, cosign attach/attest), **Kyverno image verification** (verifyImages, attestors, SBOM attestation), registry allowlisting (webhook/OPA/ImagePolicyWebhook/Kyverno), ImagePullPolicy & immutability, **Rekor transparency log**.
- **[[Monitoring Logging and Runtime Security]]** — immutable infrastructure & readOnlyRootFilesystem, Falco (install, config, rule anatomy, detect threats), behavioral analytics of syscalls.

## Cross-References

- Nền tảng CKA cần vững trước: [[Fundamentals]], [[Security]] (RBAC, Service Account, Secret, TLS...)
- GitOps security (Sealed Secrets, ESO): [[GitOps Security]]
- DevSecOps pipeline (Trivy, Cosign, SBOM in CI): [[DevSecOps]]
- Harbor image registry (scan-on-push, signing): [[Docker Registry Overview]]

## Tools Set

> Đối chiếu với **CNCF CKS Exam Curriculum v1.34** (PDF chính thức, bám sát K8s 1.34) + kinh nghiệm luyện đề cộng đồng (KodeKloud, killer.sh). Tool nào được CNCF **nêu tên trực tiếp** trong curriculum sẽ đánh dấu ✅ **(chính thức)** — gần như chắc chắn xuất hiện. Tool không được nêu tên nhưng là chuẩn luyện thi cộng đồng đánh dấu 🔶 **(de facto)** — rất hay gặp trong đề nhưng không phải "bắt buộc theo văn bản". Phần "Prod-only / biết cho vui" liệt kê tool thực tế hay dùng ở công ty nhưng **không nằm trong scope thi**, khỏi tốn thời gian luyện sâu.

**1. Runtime Security & Threat Detection (20% — Monitoring/Logging/Runtime)**

- **Falco** 🔶 — không được CNCF nêu tên, nhưng là tool runtime detection chuẩn của mọi khoá luyện CKS; exam hay hỏi viết custom rules để detect anomaly (shell trong container, write vào /etc, v.v.). Vẫn phải thuần thục.
- **Sysdig** (OSS, Falco's parent project) 🔶 — biết cơ bản `sysdig`/`csysdig` để trace syscall, ít khi là câu hỏi chính nhưng hay bị hỏi lồng trong câu Falco.
- **Tetragon** (Cilium eBPF runtime security) — bonus, KHÔNG có trong curriculum, chưa thấy xuất hiện trong đề thi thật. Biết tồn tại vì đang là trend, không cần luyện tay.

**2. OS-level Hardening (10% — System Hardening)**

- **AppArmor** ✅ (chính thức, CNCF nêu tên) — load profile, apply vào pod qua annotation.
- **Seccomp** ✅ (chính thức, CNCF nêu tên) — viết seccomp profile, restrict syscalls.
- **SELinux** — KHÔNG có trong curriculum, môi trường thi dùng Ubuntu nên gần như không chạm tới. Biết concept là đủ, đừng luyện tay.

**3. Isolation & Sandboxing (20% — Minimize Microservice Vulnerabilities)**

- CNCF ghi rõ: *"Understand and implement isolation techniques (multi-tenancy, sandboxed containers, etc.)"* ✅ — đây là mục hay bị bỏ sót vì không có tool "tên riêng" trong tiêu đề domain.
- **gVisor / Kata Containers** 🔶 — setup RuntimeClass, biết khi nào dùng cái nào (gVisor = user-space kernel, nhẹ hơn; Kata = full VM, cô lập mạnh hơn). Hay ra 1 câu setup RuntimeClass + gán vào pod.
- Multi-tenancy: Namespace + ResourceQuota + NetworkPolicy kết hợp.

**4. Policy as Code & Pod Security**

- **OPA / Gatekeeper** 🔶 — không có tên trực tiếp trong văn bản CNCF nhưng là tool admission-control được dạy ở mọi khoá CKS; ConstraintTemplate + Constraint là phần dễ mất điểm nếu không luyện tay. Vẫn phải luyện kỹ.
- **Kyverno** 🔶 — ít xuất hiện hơn OPA trong đề thi nhưng đáng học vì thực tế dùng nhiều hơn OPA (dễ viết YAML thuần, không cần Rego). Gần đây có xu hướng xuất hiện nhiều hơn trong bộ đề luyện.
- **Pod Security Admission (PSA)** ✅ (chính thức, CNCF ghi *"Use appropriate pod security standards"*) — 3 level `privileged/baseline/restricted`, enforce qua namespace label. PSP đã bị xoá từ 1.25+, đừng học PSP nữa.

**5. Image & Supply Chain Security (20% — Supply Chain Security)**

- **Kubesec, KubeLinter** ✅ (chính thức, CNCF nêu tên trực tiếp: *"static analysis of user workloads and container images e.g. Kubesec, KubeLinter"*) — dễ bị xem nhẹ vì note cũ không nhấn mạnh, nhưng đây là 2 tool được gọi tên rõ ràng nhất trong cả curriculum.
- **Trivy** 🔶 — scan image, phát hiện CVE; không có tên riêng trong văn bản CNCF (curriculum chỉ nói chung "minimize base image footprint") nhưng là tool vulnerability scan chuẩn mọi khoá dạy, must-have.
- **cosign / sigstore** 🔶 — curriculum ghi *"sign and validate artifacts"* nhưng không nêu tên tool cụ thể; cosign là de facto standard và các năm gần đây xuất hiện ngày càng nhiều trong đề (keyless + key-based signing, verify trong CI, Kyverno `verifyImages`). Nên luyện tay thật sự, không chỉ đọc concept.
- **SBOM** ✅ (chính thức, CNCF nêu tên) — SPDX/CycloneDX, tạo bằng Syft, kèm `cosign attach/attest`.

**6. Cluster Hardening (15% — Cluster Hardening + một phần Cluster Setup)**

- **kube-bench** ✅ (chính thức, CNCF ghi *"Use CIS benchmark to review security configuration"*) — chạy CIS benchmark, fix findings.
- **Audit logging** ✅ (chính thức, CNCF ghi *"Use Kubernetes audit logs to monitor access"*) — viết audit policy, biết `audit-log-path`, `audit-policy-file` config trong kube-apiserver.

**7. Network Security (15% — Cluster Setup)**

- **Calico** 🔶 (không phải Cilium) — curriculum chỉ ghi chung "Network security policies", không nêu tên CNI cụ thể, nhưng môi trường lab/exam thực tế (KodeKloud, killer.sh) đều dùng Calico mặc định. Phải thuộc nằm lòng NetworkPolicy syntax, ingress/egress rules.
- **Cilium** ✅ (chính thức, nhưng chỉ cho mục đích khác) — CNCF chỉ nêu tên Cilium trong domain **Minimize Microservice Vulnerabilities** cho *"Pod-to-Pod encryption (Cilium, Istio)"*, KHÔNG phải cho NetworkPolicy chính. Đừng nhầm Cilium là CNI chính để luyện NetworkPolicy — vai trò của nó trong exam là mTLS/eBPF, xem mục 8.

**8. mTLS / Service Mesh (nằm trong 20% — Minimize Microservice Vulnerabilities)**

- **Istio** ✅ (chính thức, CNCF ghi rõ *"Implement Pod-to-Pod encryption (Cilium, Istio)"*) — PeerAuthentication, AuthorizationPolicy, STRICT mTLS mode. Đây là service mesh chính thức cho CKS.
- **Linkerd** ❌ — **KHÔNG nằm trong CKS curriculum**. Linkerd chỉ được CNCF nhắc tên ở phần giới thiệu chung hệ sinh thái CNCF (cùng Fluentd, Prometheus, gRPC...), hoàn toàn không liên quan tới nội dung thi. Đúng như bạn nghi ngờ — bỏ Linkerd ra khỏi list luyện, tập trung Istio.
- **Cilium** (mTLS mode qua Cilium Service Mesh) — thứ 2 sau Istio nếu muốn học thêm, nhưng Istio là trọng tâm vì phổ biến hơn trong tài liệu luyện thi.

**9. Secrets & Encryption (20% — Minimize Microservice Vulnerabilities)**

- **Kubernetes native Secret + etcd encryption at rest (EncryptionConfiguration)** ✅ (chính thức, CNCF ghi *"Manage kubernetes secrets"*) — đây là trọng tâm thật sự của mục secrets trong exam.
- **Vault / Sealed Secrets / External Secrets Operator (ESO)** — KHÔNG nằm trong curriculum, exam không test external secret manager. Đây là kiến thức "biết cho prod" (hay dùng thật ở công ty, GitOps pipeline) chứ không phải thứ cần luyện tay để thi. Có thể đọc qua 1 lần cho biết, đừng dành nhiều giờ luyện.

### Tool KHÔNG cần luyện sâu cho exam (biết concept / dùng ở prod là chính)

- **Linkerd** — không có trong curriculum, dùng Istio thay thế.
- **HashiCorp Vault, Sealed Secrets, ESO** — prod-grade secret management, không test trong exam.
- **SELinux** — môi trường thi là Ubuntu, không chạm tới; biết concept so sánh với AppArmor là đủ.
- **Tetragon** — trend hot nhưng chưa vào curriculum, không cần luyện tay.
- **Prometheus/Grafana** — hữu ích cho observability thực tế nhưng CKS không test monitoring stack, chỉ test audit log + Falco.
- **Notation (notary v2)** — một alternative của cosign cho image signing, ít tài liệu luyện thi nhắc tới hơn cosign; biết tồn tại là đủ.

## Tool by domain

> Trọng số đã sửa đúng theo CNCF CKS Curriculum v1.34: **Cluster Setup 15%** (không phải 10%), **System Hardening 10%** (không phải 15%). ✅ = tool CNCF nêu tên trực tiếp, 🔶 = chuẩn cộng đồng/de facto nhưng không có tên trong văn bản.

```
cks-tools/
├── cluster-setup (15%)
│   ├── calico              🔶 # NetworkPolicy enforcement (CNI mặc định trong lab/exam)
│   ├── kube-bench           ✅ # CIS Benchmark scanner (etcd/kubelet/kubedns/kubeapi)
│   ├── openssl              🔶 # cert generation/inspection
│   ├── kubeadm               🔶 # cluster bootstrap, verify platform binaries trước khi deploy
│   └── ingress-nginx         🔶 # TLS termination, secure ingress (curriculum: "Properly set up Ingress objects with TLS")
│
├── cluster-hardening (15%)
│   ├── kubectl (RBAC)         ✅ # Role/RoleBinding, ClusterRole — minimize exposure
│   ├── audit-log config      ✅ # apiserver --audit-log-path
│   ├── kube-apiserver flags  ✅ # disable anonymous-auth, insecure-port, restrict API access
│   └── service-account       ✅ # disable auto-mount, minimize permissions trên SA mới tạo
│
├── system-hardening (10%)
│   ├── AppArmor                ✅ # MAC profile per pod
│   ├── seccomp                 ✅ # syscall whitelist
│   ├── Linux capabilities     🔶 # drop ALL, add specific (không nêu tên riêng nhưng thuộc "least-privilege")
│   ├── kube-bench              ✅ # node-level CIS check (dùng lại)
│   └── systemd / sysctl         🔶 # kernel & OS hardening, minimize host footprint
│
├── minimize-microservice-vulnerabilities (20%)
│   ├── Pod Security Admission   ✅ # thay PSP đã bị xoá từ 1.25+ ("appropriate pod security standards")
│   ├── OPA / Gatekeeper         🔶 # policy-as-code, admission control (chuẩn cộng đồng)
│   ├── Kyverno                  🔶 # alt policy engine (dễ học hơn OPA, ngày càng phổ biến)
│   ├── gVisor / Kata            ✅ # sandboxed containers, isolation techniques
│   ├── K8s native Secret + etcd encryption ✅ # "manage kubernetes secrets"
│   └── mTLS — Istio hoặc Cilium  ✅ # CNCF chỉ nêu đích danh 2 tool này — KHÔNG có Linkerd
│
├── supply-chain-security (20%)
│   ├── Kubesec, KubeLinter       ✅ # static analysis — CNCF nêu tên trực tiếp
│   ├── Trivy                    🔶 # image + IaC vulnerability scan (chuẩn cộng đồng)
│   ├── docker/buildkit          🔶 # secure image build (no root, multistage, minimal base image)
│   ├── admission-controller      🔶 # ImagePolicyWebhook — "permitted registries"
│   ├── SBOM (Syft/Grype)         ✅ # CNCF nêu tên "SBOM" trực tiếp
│   └── cosign/sigstore           🔶 # sign & validate artifacts — ngày càng hay ra trong đề
│
└── monitor-logging-and-runtime-security (20%)
    ├── Falco                     🔶 # runtime threat detection (syscall-based) — chuẩn cộng đồng, vẫn phải luyện kỹ
    ├── Sysdig                    🔶 # runtime forensics, Falco's parent project
    ├── audit-log (apiserver)     ✅ # "Use Kubernetes audit logs to monitor access"
    ├── readOnlyRootFilesystem    ✅ # "Ensure immutability of containers at runtime"
    └── Tetragon (eBPF)            ❌ # bonus/trend, KHÔNG có trong curriculum — biết cho vui, đừng luyện tay
```

### Tool đã loại khỏi list luyện tay (không thuộc scope CKS)

- **Linkerd** — không có trong CKS curriculum, service mesh chính thức là **Istio** (CNCF nêu tên trực tiếp) hoặc Cilium.
- **HashiCorp Vault / Sealed Secrets / ESO** — prod-grade secret management, không test trong exam (curriculum chỉ test Secret + etcd encryption).
- **SELinux** — môi trường thi dùng Ubuntu, biết concept so sánh AppArmor là đủ.
- **Prometheus/Grafana** — không nằm trong curriculum, CKS chỉ test audit log + runtime detection (Falco).
- **Notation (notary v2)** — alternative của cosign, ít gặp hơn trong đề luyện, biết tồn tại là đủ.