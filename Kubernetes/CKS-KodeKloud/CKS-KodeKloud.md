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
| [[Cluster Setup and Hardening]]                 | Cluster Setup + Cluster Hardening                                                                   | 10% + 15% | 32/32    |
| [[System Hardening]]                            | System Hardening                                                                                    | 15%       | 18/18    |
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
**1. Runtime Security & Threat Detection**

- **Falco** — bắt buộc phải thuần thục, exam hay hỏi viết custom rules để detect anomaly (shell trong container, write vào /etc, v.v.)
- Hiểu cơ bản về **Sysdig** (Falco's parent project) để trace syscall

**2. OS-level Hardening**

- **AppArmor** — load profile, apply vào pod qua annotation
- **Seccomp** — viết seccomp profile, restrict syscalls
- Không cần SELinux sâu (ít xuất hiện hơn) nhưng nên biết concept

**3. Policy as Code**

- **OPA / Gatekeeper** — ConstraintTemplate + Constraint, đây là phần dễ mất điểm nếu không luyện tay
- **Kyverno** ít xuất hiện hơn trong exam nhưng đáng học vì thực tế dùng nhiều hơn OPA (dễ viết YAML thuần, không cần Rego)

**4. Image & Supply Chain Security**

- **Trivy** — scan image, phát hiện CVE (must-have, xuất hiện chắc chắn)
- Image signing concept — cosign/notation (ít khi hỏi sâu nhưng cần hiểu supply chain attack surface)

**5. Cluster Hardening**

- **kube-bench** — chạy CIS benchmark, fix findings
- **Audit logging** — viết audit policy, biết audit-log-path, audit-policy-file config trong kube-apiserver

**6. Network Security**

- **Calico** (không phải Cilium) — NetworkPolicy nâng cao, exam dùng Calico làm CNI mặc định
- Phải thuộc nằm lòng NetworkPolicy syntax, ingress/egress rules

**7. Secrets & Encryption**

- etcd encryption at rest (EncryptionConfiguration)
- Không cần Vault, exam không test external secret manager

## Tool by domain

```
cks-tools/
├── cluster-setup (10%)
│   ├── calico              # NetworkPolicy enforcement (CNI)
│   ├── kube-bench           # CIS Benchmark scanner
│   ├── openssl              # cert generation/inspection
│   ├── kubeadm               # cluster bootstrap security config
│   └── ingress-nginx         # TLS termination, secure ingress
│
├── cluster-hardening (15%)
│   ├── kubectl (RBAC)         # Role/RoleBinding, ClusterRole
│   ├── audit-log config      # apiserver --audit-log-path
│   ├── kube-apiserver flags  # disable anonymous-auth, insecure-port
│   └── service-account       # disable auto-mount, least privilege
│
├── system-hardening (15%)
│   ├── AppArmor                # MAC profile per pod
│   ├── seccomp                 # syscall whitelist
│   ├── Linux capabilities     # drop ALL, add specific
│   ├── kube-bench              # node-level CIS check (dùng lại)
│   └── systemd / sysctl         # kernel & OS hardening
│
├── minimize-microservice-vulnerabilities (20%)
│   ├── OPA / Gatekeeper         # policy-as-code, admission control
│   ├── Kyverno                  # alt policy engine (dễ học hơn OPA)
│   ├── PodSecurityStandards     # (thay PSP đã bị xoá từ 1.25+)
│   ├── Sealed Secrets / SOPS   # encrypt secrets at rest
│   └── mTLS (Istio/Linkerd)      # service mesh security
│
├── supply-chain-security (20%)
│   ├── Trivy                    # image + IaC vulnerability scan
│   ├── docker/buildkit          # secure image build (no root, multistage)
│   ├── admission-controller      # ImagePolicyWebhook
│   ├── kubesec                  # static YAML security scan
│   └── cosign/sigstore           # image signing & verification
│
└── monitor-logging-and-runtime-security (20%)
    ├── Falco                     # runtime threat detection (syscall-based)
    ├── Sysdig                    # runtime forensics
    ├── audit-log (apiserver)     # dùng lại từ cluster-hardening
    ├── Prometheus/Grafana         # observability baseline
    └── eBPF-based tools (Tetragon) # bonus, hot trend giờ hay ra
```