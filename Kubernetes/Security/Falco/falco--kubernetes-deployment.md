# Falco — Kubernetes Deployment & Audit Log Integration
Tier: 2
Parent: [[Falco]]
Related: [[falco--drivers]], [[falco--falcoctl-plugins]], [[falco--performance-tuning-drops]]
Tags: #falco #kubernetes #cks #daemonset

## What it does

Cách Falco được triển khai trên K8s (thường qua Helm chart `falcosecurity/falco`) dưới dạng **DaemonSet** để có mặt trên mọi node, cộng với (tùy chọn) tích hợp **K8s Audit Log** qua plugin `k8saudit` để bắt được cả sự kiện ở tầng API server (ai tạo/xoá resource gì) chứ không chỉ syscall ở tầng node.

## Why it exists

Syscall-level detection (chạy trên node) không thấy được hành vi ở tầng **Kubernetes API** — ví dụ ai đó dùng `kubectl` tạo 1 pod với `hostPID: true` hoặc gắn ServiceAccount có quyền cao, đó là API call chứ không phải syscall trên node. Muốn có full picture (node + control plane) thì cần cả 2 nguồn: driver (syscall) + k8saudit plugin (API events).

## How it works (flow/diagram)

```
┌─────────────────────────────┐        ┌──────────────────────────────┐
│  K8s API Server               │        │  Mỗi Worker Node               │
│  (Audit Policy + Webhook)     │        │  ┌────────────────────────┐   │
│         │                     │        │  │ Falco DaemonSet Pod     │   │
│         │ audit event (JSON)  │        │  │  - driver (syscall)     │   │
│         ▼                     │        │  │  - k8saudit plugin      │◀──┼── webhook POST
│  ┌────────────────┐           │        │  │    (nếu deploy webhook   │   │
│  │ Falco k8saudit  │◀──────────────────┼──┤    server dạng Service) │   │
│  │ webhook service │           │        │  └────────────────────────┘   │
│  └────────────────┘           │        └──────────────────────────────┘
└─────────────────────────────┘
```

2 cách phổ biến để đưa K8s Audit Log vào Falco:
1. **Webhook backend**: API server config `--audit-webhook-config-file` trỏ tới 1 Service chạy Falco (hoặc falco k8saudit-webhook) nhận POST audit event trực tiếp.
2. **Log file + falcosidekick/log forwarder**: API server ghi audit log ra file, 1 sidecar/agent đọc file rồi forward vào Falco qua plugin.

## Config gotchas

- **Deploy Falco pod thành công KHÔNG có nghĩa là k8saudit hoạt động** — nếu quên cấu hình Audit Policy + webhook ở API server (đây là thay đổi ở **control plane**, không phải ở Falco), Falco sẽ chạy khoẻ re nhưng **không bao giờ nhận được audit event nào**. Dễ nhầm tưởng "Falco bị lỗi" trong khi thực ra là thiếu bước cấu hình phía control plane — đây là 1 trong những nhầm lẫn phổ biến nhất khi mới triển khai.
- Trên **managed K8s (EKS/GKE/AKS)**, mày **không tự sửa được** flag `--audit-webhook-config-file` của API server managed — phải dùng cơ chế riêng của cloud provider (EKS Control Plane Logging → CloudWatch → forwarder; GKE Audit Logs → Cloud Logging → export) thay vì webhook trực tiếp như self-managed cluster. Đừng áp dụng nguyên si hướng dẫn tự-manage cluster.
- **Audit Policy quá rộng** (`level: RequestResponse` cho mọi resource) tạo ra khối lượng audit log khổng lồ → cả API server lẫn Falco đều bị áp lực xử lý. Nên scope audit policy theo resource nhạy cảm (Secret, ClusterRoleBinding, Pod với privileged/hostPath) thay vì log toàn bộ.
- Helm chart default **resource limit khá thấp** cho container Falco — cluster có nhiều syscall traffic (CI/CD runner node, batch job node) dễ bị OOMKilled. Luôn override `resources.limits` theo tải thực tế của node, không dùng default.
- `driver.kind` trong Helm values nên set tường minh `modern_ebpf` thay vì để tự động detect — tự động detect đôi khi fallback về kmod trên node có kernel lạ, kéo theo rủi ro compatibility mô tả ở [[falco--drivers]].
- Falco pod cần mount đúng **container runtime socket** (`/run/containerd/containerd.sock` hay `/var/run/docker.sock` tùy CRI) để enrich metadata container — sai path (phổ biến khi cluster đổi từ Docker sang containerd mà không update Helm values) làm alert thiếu hẳn thông tin container/image, log giống như syscall "trần trụi" từ host.
- `rollingUpdate.maxUnavailable` của DaemonSet: giá trị quá cao khi upgrade Falco version đồng loạt trên nhiều node = gap giám sát lớn tạm thời trên toàn cluster.

## Security notes

- ServiceAccount của Falco chỉ cần quyền **read** (get/list/watch Pod, Namespace, Node...) để enrich metadata — không bao giờ cần quyền write. Audit định kỳ RBAC binding của Falco ServiceAccount, đây là target hấp dẫn nếu bị compromise vì nó có host-level visibility.
- Nếu dùng webhook backend cho k8saudit, endpoint webhook cần **mTLS** — audit event chứa thông tin nhạy cảm (request body có thể chứa Secret data trong 1 số trường hợp `level: RequestResponse`).
- Falco DaemonSet chạy privileged → cân nhắc PodSecurity admission exemption riêng cho namespace Falco, đồng thời giới hạn namespace đó chỉ cluster-admin mới sửa được (đừng để namespace Falco lẫn với namespace ứng dụng thường).

## Refs

- https://github.com/falcosecurity/charts (Helm chart chính thức)
- https://falco.org/docs/plugins/plugin-k8saudit/
- https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/ (K8s Audit Policy chính thức — cần cho phần webhook)
