---
title: Understanding the Kubernetes Attack Surface
tags:
  - kubernetes
  - security
  - cks
  - attack-surface
date: 2026-08-18
---

# Understanding the Kubernetes Attack Surface

## Ý tưởng chính

"Attack surface" của một hệ thống containerized/Kubernetes không chỉ nằm ở application code. Nó trải rộng qua nhiều lớp: hạ tầng cloud, cấu hình cluster, cách container được chạy, và code của ứng dụng. Một kẻ tấn công thường không cần khai thác lỗ hổng code phức tạp — chỉ cần một cấu hình mặc định bị bỏ sót (port mở, thiếu authentication) ở bất kỳ lớp nào cũng đủ để leo thang từ truy cập bên ngoài đến toàn quyền kiểm soát cluster.

## Demo tấn công minh họa: ứng dụng bình chọn Cat vs Dog

KodeKloud minh họa attack surface bằng một kịch bản tấn công thực tế vào một ứng dụng voting (mèo vs chó) chạy trên Kubernetes, gồm `www.vote.com` (nơi bỏ phiếu) và `www.result.com` (nơi xem kết quả). Chuỗi tấn công diễn ra như sau:

1. **Reconnaissance (trinh sát hạ tầng)**: Attacker chỉ có tên miền, không biết gì về stack công nghệ hay nơi hosting. Cô ping cả hai domain và phát hiện chúng resolve về cùng một IP — cho thấy chung hạ tầng hosting.
2. **Port scanning**: Quét port trên IP đó, phát hiện port **2375** (Docker daemon API, không mã hoá) đang mở công khai. Điều này lộ ra rằng ứng dụng chạy trong container.
3. **Khai thác Docker daemon không xác thực**: Vì port 2375 mở mà không yêu cầu authentication, attacker dùng `docker -H <host> ps` và `docker -H <host> version` để liệt kê container và biết thông tin Docker engine mà không cần đăng nhập.
4. **Chạy container privileged**: Attacker khởi chạy một container Ubuntu với `docker -H <host> run --privileged -it ubuntu bash`. Vì Docker daemon không giới hạn privileged mode, cô có ngay shell root bên trong container.
5. **Container escape (Dirty COW)**: Container không có sẵn `curl`/`wget` nhưng không có gì ngăn attacker cài đặt package tùy ý. Cô cài `curl`, tải script khai thác lỗ hổng kernel **Dirty COW**, và thoát (escape) từ container ra host thật.
6. **Host reconnaissance**: Trên host, `df -h` và `hostname` cho thấy đây là một node tên "worker" — tức một worker node của Kubernetes cluster. Các container có tên bắt đầu bằng `k8s` lộ diện, bao gồm cả **Kubernetes Dashboard**.
7. **Dashboard bị public**: Kiểm tra `iptables -L -t nat` xác nhận Kubernetes Dashboard được expose qua NodePort **30080** ra ngoài internet, không có authentication. Attacker truy cập dashboard và thấy toàn bộ thông tin cluster: node, deployment, namespace.
8. **Compromise database**: Từ dashboard, attacker xem được environment variables của DB pod — trong đó có sẵn database credentials (hardcoded). Cô dùng `psql` kết nối trực tiếp vào database, tìm bảng lưu phiếu bầu, và chạy script sửa toàn bộ phiếu "dog" thành "cat" — thay đổi kết quả bầu cử.

Toàn bộ chuỗi tấn công này chỉ khai thác **cấu hình mặc định không an toàn** ở từng lớp — không cần một zero-day hay lỗ hổng code phức tạp nào ở ứng dụng.

## 4C's của Cloud Native Security

Bốn lớp phòng thủ lồng nhau (từ ngoài vào trong): **Cloud → Cluster → Container → Code**. Bảo mật ở lớp ngoài là nền tảng cho các lớp bên trong — nếu Cloud không an toàn thì mọi nỗ lực bảo mật Cluster/Container/Code phía trong đều vô nghĩa.

| Lớp | Phạm vi | Lỗ hổng trong demo | Biện pháp phòng thủ |
|---|---|---|---|
| **1. Cloud** | Hạ tầng vật lý/ảo hoá bên dưới: public cloud, private cloud, on-prem, co-located datacenter | Hạ tầng không được bảo vệ đủ, cho phép truy cập không giới hạn tới các port của cluster (không có firewall) | Network firewall, access control chặt chẽ ở tầng hạ tầng; không expose port ra ngoài một cách không cần thiết |
| **2. Cluster** | Bản thân Kubernetes cluster: API server, dashboard, node | Docker daemon lộ ra công khai không xác thực; Kubernetes Dashboard truy cập được mà không có authentication/authorization | Bảo vệ Docker daemon theo best practice; bảo mật Kubernetes API bằng access control mạnh; giới hạn truy cập dashboard; áp dụng network policy và ingress security |
| **3. Container** | Cách container được build và chạy | Không có giới hạn khi deploy container, cho phép chạy container ở **privileged mode**; không kiểm soát nguồn/tag của image | Chỉ cho phép image từ registry tin cậy (policy enforcement); không cho phép privileged mode; dùng container sandboxing (thêm lớp cô lập) |
| **4. Code** | Application code chạy bên trong container | Không nêu trực tiếp trong demo nhưng là hệ quả tất yếu: hardcode credentials, truyền secret qua environment variable, giao tiếp không mã hoá | Dùng Secrets management / vault; bật mTLS để mã hoá giao tiếp giữa các pod |

Ghi nhớ thứ tự **Cloud, Cluster, Container, Code** — đây là thứ tự từ ngoài vào trong, phản ánh nguyên lý "defense in depth": mỗi lớp phải tự bảo mật độc lập vì attacker có thể chọc thủng từ lớp ngoài cùng rồi leo thang dần vào trong (đúng như trình tự trong demo: Cloud → Cluster (Docker/Dashboard) → Container (privileged, escape) → Code/Data (DB credentials trong env var)).

## CKS Exam Domain Map

**CKS (Certified Kubernetes Security Specialist) — 2h, open-book, hands-on clusters. Prerequisite: active CKA.**

| Domain | Weight | Topics chính |
|---|---|---|
| **Cluster Setup** | 10% | Network policies, TLS, CIS benchmark, ingress TLS, node metadata |
| **Cluster Hardening** | 15% | RBAC, ServiceAccount, API server flags, upgrade frequency |
| **System Hardening** | 15% | OS footprint, AppArmor, seccomp, capabilities |
| **Microservice Vulnerabilities** | 20% | PSA, OPA/Kyverno, secrets mgmt, container sandbox, mTLS |
| **Supply Chain Security** | 20% | Image scan, whitelist registries, sign/verify images |
| **Monitoring & Runtime Security** | 20% | Behavioral analytics (Falco), audit logs, immutable containers |

## Kiến trúc bảo mật: Authentication → Authorization → Admission Control

Mọi request tới API server đi qua 3 lớp kiểm tra tuần tự — hiểu rõ thứ tự này giúp debug nhanh khi request bị reject (401 vs 403 vs 400):

```
Client Request
      │
      ▼
┌─────────────────┐
│  Authentication  │ ← Bạn là ai?
│  - X.509 certs   │   Fail → 401
│  - Service tokens│
│  - OIDC/LDAP     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Authorization   │ ← Bạn được làm gì?
│  - RBAC          │   Fail → 403
│  - ABAC (legacy) │
│  - Node auth     │
│  - Webhook       │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│  Admission Control                       │ ← Request này có nên được phép?
│  Phase 1: Mutating Webhooks              │   Fail → 400/403
│    - Inject sidecars, defaults           │
│    - DefaultStorageClass, ServiceAccount │
│  Phase 2: Validating Webhooks            │
│    - OPA Gatekeeper, Kyverno             │
│    - ResourceQuota, LimitRanger          │
│    - PodSecurity                         │
└────────┬────────────────────────────────┘
         │
         ▼
     etcd (persist)
```

### Principal Types (ai/cái gì được authenticate)

| Principal | Identity source | Use case |
|---|---|---|
| **User** | X.509 cert CN, OIDC sub | Human operators, CI/CD |
| **Group** | X.509 cert O, OIDC groups | Team-based RBAC |
| **ServiceAccount** | K8s resource, JWT token | Pod-to-API auth |
| **Node** | X.509 cert `system:node:<name>` | Kubelet API access |

## Attack Vectors & Mitigations (bảng tra cứu nhanh)

| Attack vector | Mitigation |
|---|---|
| Compromised container | Seccomp, AppArmor, non-root, read-only FS, no privileged |
| Container escape to host | RuntimeClass (gVisor/Kata), user namespace, PSS Restricted |
| Lateral movement | NetworkPolicy deny-all default, namespace isolation, mTLS |
| Privilege escalation | RBAC least privilege, PSA, no-new-privileges, drop caps |
| Secrets exposure | etcd encryption, ESO, Vault, no env-var secrets, RBAC |
| Supply chain | Image signing (Cosign), Trivy scan, private registry, Kyverno image policy |
| Malicious image | Admission webhook, imagePullPolicy Always, registry allowlist |
| API server abuse | Audit logging, Falco, RBAC minimal permissions |
| etcd breach | mTLS etcd, encryption at rest, network isolation |

Mỗi hàng trong bảng này ánh xạ trực tiếp vào một domain CKS — dùng như checklist khi ôn thi.

## Security Checklist (Production)

### Cluster level
- [ ] API server: `--anonymous-auth=false`, `--profiling=false`, audit logging enabled
- [ ] etcd: TLS client auth, encryption at rest enabled
- [ ] Kubelet: `--authorization-mode=Webhook`, `--anonymous-auth=false`
- [ ] CIS Benchmark scan (kube-bench) — all PASS hoặc documented exception
- [ ] RBAC: không có `cluster-admin` binding rộng rãi, review quarterly
- [ ] Network: deny-all NetworkPolicy per namespace, ingress với TLS

### Workload level
- [ ] Pod Security Standards enforced (`restricted` cho prod namespaces)
- [ ] No `hostPID/hostNetwork/hostIPC: true`
- [ ] No `privileged: true`
- [ ] Non-root containers (`runAsNonRoot: true`)
- [ ] Read-only root filesystem (`readOnlyRootFilesystem: true`)
- [ ] Resource limits set (tránh DoS)
- [ ] Seccomp profile: `RuntimeDefault` hoặc custom

### Supply chain
- [ ] Images signed và verified (Cosign + Kyverno)
- [ ] No `:latest` tag trong production
- [ ] Private registry (không pull từ Docker Hub trực tiếp)
- [ ] Trivy scan trong CI — block on CRITICAL
- [ ] ImagePullPolicy: Always

### Runtime
- [ ] Falco deployed và rules configured
- [ ] Audit policy: log `create/delete/update` trên sensitive resources
- [ ] Secrets: encryption at rest hoặc ESO/Vault
- [ ] No secrets in env vars (dùng file mount)

## Gotchas

- Attack surface lớn nhất trong thực tế thường **không phải** là lỗ hổng phức tạp trong application code, mà là **cấu hình mặc định** bị bỏ sót: port Docker daemon (2375) mở công khai, Kubernetes Dashboard không bật authentication, container chạy ở privileged mode.
- Docker daemon lắng nghe trên **TCP port 2375 là không mã hoá và không xác thực theo mặc định** — nếu remote API cần bật, phải dùng TLS (port 2376) và xác thực client cert; không bao giờ để 2375 mở ra internet.
- **Privileged container** = container gần như có toàn quyền như root trên host (bỏ qua phần lớn cô lập của container). Đây là nguyên nhân trực tiếp giúp attacker thực hiện container escape trong demo (kết hợp với lỗ hổng kernel Dirty COW).
- Container escape không chỉ nhờ privileged mode — nó còn cần một lỗ hổng kernel/host (ví dụ Dirty COW). Privileged mode làm tăng khả năng khai thác các lỗ hổng loại này thành công.
- Kubernetes Dashboard bị lộ qua NodePort công khai (30080) mà không có authentication là một trap kinh điển — nhớ rằng dashboard mặc định (tùy phiên bản/cấu hình) có thể có quyền rất rộng trên cluster nếu không giới hạn RBAC cho service account của nó.
- Đừng lưu database credentials (hay bất kỳ secret nào) dưới dạng **plain environment variable** đọc được qua `kubectl describe pod` hoặc `docker inspect` — đây chính là cách attacker lấy được DB credentials trong demo. Dùng Kubernetes Secrets kết hợp với external vault/secret manager thay vì hardcode.
- 4C's không phải checklist rời rạc — chúng lồng nhau (nested), nghĩa là bảo mật Cluster/Container/Code là vô nghĩa nếu lớp Cloud (hạ tầng) đã bị hở do thiếu firewall/access control.
- Trang "A Quick Reminder" trong khoá học chỉ là hướng dẫn quy trình học (ưu tiên làm lab/video trước khi setup môi trường local) — không chứa nội dung kỹ thuật liên quan đến thi CKS nên không được đưa vào ghi chú này.
