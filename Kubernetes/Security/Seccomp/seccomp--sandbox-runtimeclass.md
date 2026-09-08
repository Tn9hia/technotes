# Seccomp — Sandboxed Containers & RuntimeClass (gVisor, Kata)
Tier: 2
Parent: [[seccomp]]
Related: [[seccomp--kubernetes]], [[seccomp--cks-exam]]
Tags: #kubernetes #sandbox #runtimeclass #isolation
Last updated: 2026-09-06
Verified against: kubernetes.io RuntimeClass docs, CKS Curriculum v1.34 (chính thức)

## What it does

`RuntimeClass` là API Kubernetes để chọn **container runtime handler** khác nhau cho từng Pod (thay vì mọi Pod dùng chung 1 OCI runtime mặc định như runc). Đây là cơ chế Kubernetes dùng để chạy Pod trong sandbox mạnh hơn container thường — ví dụ gVisor hoặc Kata Containers — và nó **độc lập nhưng bổ sung** cho seccomp.

## Why it exists

Seccomp giới hạn syscall nhưng vẫn cho container **share chung kernel host**. Nếu kernel host có 1 bug (0-day hoặc chưa vá) ở đúng syscall được allowlist, container vẫn có thể escape. Với workload chạy code không tin cậy (multi-tenant, chạy code khách hàng, CI runner công khai...), 1 lớp cô lập mạnh hơn kernel-syscall-filter là cần thiết — đây chính là lý do đề mục **"Understand and implement isolation techniques (multi-tenancy, sandboxed containers, etc.)"** nằm trong domain **Minimize Microservice Vulnerabilities (20%)** của CKS curriculum chính thức.

## When — dùng khi nào

- Chạy code không tin cậy từ bên thứ 3 (user-submitted code, CI job công khai, PaaS đa tenant).
- Cụm cluster chia sẻ giữa nhiều team/khách hàng mà **namespace isolation không đủ tin cậy** theo yêu cầu compliance.
- **Không cần** cho workload nội bộ tin cậy, đã qua image scanning, chạy trong network đã segment — thêm sandbox tốn overhead (CPU/latency) không đáng nếu threat model không yêu cầu.

## How it works (2 kiến trúc khác nhau hoàn toàn)

```
Container thường (runc)          gVisor (runsc)                Kata Containers
┌─────────────┐                  ┌─────────────┐               ┌─────────────┐
│  App process │                  │  App process │               │  App process │
└──────┬──────┘                  └──────┬──────┘               └──────┬──────┘
       │ syscall trực tiếp              │ syscall                     │ syscall
       ▼                                ▼                             ▼
┌─────────────┐                  ┌─────────────┐               ┌─────────────┐
│ seccomp-BPF  │ (lọc, không cô  │   Sentry     │ (user-space   │ Guest kernel │ ← kernel
│ filter       │  lập kernel)    │ (app kernel) │  chặn/giả lập)│ (riêng biệt) │   RIÊNG
└──────┬──────┘                  └──────┬──────┘               └──────┬──────┘
       ▼                                ▼                             ▼
   Host kernel                Host kernel (chỉ nhận ~50           KVM hypervisor
  (chia sẻ, mọi                syscall rất hẹp từ Sentry,        (Host kernel chỉ
   container dùng                dùng chính seccomp bên           thấy 1 VM, không
   chung 1 kernel)               trong Sentry để tự giới hạn)     thấy syscall app)
```

- **gVisor**: chặn syscall ở **user-space** bằng 1 "application kernel" tên **Sentry**, viết bằng Go, tự implement lại phần lớn Linux syscall interface. App bên trong tưởng nó nói chuyện với kernel thật, thực ra đang nói chuyện với Sentry. Sentry chỉ cần gọi 1 tập syscall rất hẹp xuống host kernel thật (thường dẫn ví dụ ~50 syscall so với ~150-200+ của 1 seccomp profile tight cho container thường) — **bản thân Sentry cũng tự áp seccomp filter cho chính nó** khi gọi xuống host, tức là gVisor dùng seccomp như 1 lớp phòng thủ nội bộ, không thay thế seccomp.
- **Kata Containers**: mỗi Pod chạy trong **1 VM nhẹ riêng** (qua KVM), có **guest kernel riêng hoàn toàn**. Muốn escape phải qua 2 lớp: thoát guest kernel rồi thoát tiếp hypervisor (QEMU/cloud-hypervisor) — cô lập mạnh hơn về mặt lý thuyết (hardware-enforced) nhưng overhead khởi động/tài nguyên cao hơn gVisor.

### Cấu hình RuntimeClass

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc          # tên handler phải khớp cấu hình containerd/CRI-O trên node
---
apiVersion: v1
kind: Pod
spec:
  runtimeClassName: gvisor
  containers: [...]
```
`handler` phải được cấu hình sẵn trong containerd (`/etc/containerd/config.toml` → `[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]`) hoặc CRI-O tương ứng **trên từng node** — RuntimeClass chỉ là phần khai báo phía Kubernetes API, không tự cài runtime.

## Config gotchas

- `runtimeClassName` sai tên handler chưa được cấu hình trên node → Pod kẹt `CreateContainerError`/`RunPodSandbox` fail tương tự lỗi thiếu seccomp profile — dễ nhầm 2 loại lỗi này nếu chỉ đọc thoáng qua event message.
- Không phải mọi node trong cluster đều có handler này — cần kết hợp `nodeSelector`/taint-toleration để Pod dùng `RuntimeClass` chỉ schedule vào đúng node đã cài gVisor/Kata, nếu không Pod sẽ Pending hoặc fail tuỳ theo cấu hình scheduler.
- gVisor **chưa hỗ trợ 100% syscall/feature Linux** — 1 số workload cần syscall/feature lạ (một số syscall GPU, `io_uring` phiên bản mới, một số cgroup v2 feature) có thể không chạy được hoặc chạy khác hành vi dưới Sentry. Luôn test app cụ thể trước khi áp dụng đại trà, đừng giả định "chạy được trên container thường thì chắc chắn chạy được trên gVisor".
- Kata yêu cầu **nested virtualization** nếu node đã là VM (phổ biến trên cloud) — cần enable qua cấu hình hypervisor cloud provider, nếu không Pod sẽ không khởi động được sandbox.

## Security notes

- RuntimeClass **không thay thế** seccomp/AppArmor/PSS — best practice CKS đề cập rõ là **defense in depth**: vẫn giữ Pod Security Standards + seccomp + capability drop, cộng thêm RuntimeClass cho riêng workload không tin cậy, không phải chọn 1 trong 2.
- gVisor tự giới hạn syscall xuống host bằng chính cơ chế seccomp (bên trong Sentry) — nghĩa là hiểu seccomp-BPF (xem [[seccomp--kernel-bpf]]) giúp hiểu luôn vì sao gVisor an toàn hơn: nó thu hẹp seccomp allowlist xuống mức cực hẹp (~50 syscall) mà vẫn giữ được tương thích ở tầng app nhờ tự emulate syscall interface phía trên.

## Refs
- [Kubernetes — RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)
- [gVisor — Architecture Guide](https://gvisor.dev/docs/architecture_guide/)
- [Kata Containers — Documentation](https://katacontainers.io/docs/)
- CKS Curriculum v1.34 (chính thức, cncf/curriculum) — domain "Minimize Microservice Vulnerabilities"
