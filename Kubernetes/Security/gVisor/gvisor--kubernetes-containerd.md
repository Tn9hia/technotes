# gVisor — Kubernetes / containerd / CRI-O Integration
Tier: 2
Parent: [[gvisor]]
Related: [[gvisor--security-model]], [[gvisor--common-errors-lessons]]
Tags: #gvisor #kubernetes #containerd #cks #runtimeclass

## What it does

Cách gVisor (`runsc`) cắm vào container runtime interface (CRI) để Kubernetes chạy được Pod trong sandbox, thông qua `RuntimeClass`. 3 con đường chính: **GKE Sandbox** (managed, tối ưu sẵn), **containerd + containerd-shim-runsc-v1** (tự vận hành, được support chính thức), **CRI-O** (best-effort, KHÔNG chính thức support).

## Why it exists

Kubernetes cần 1 cách chuẩn để chọn "container này chạy bằng runtime nào" mà không sửa Pod spec nhiều — `RuntimeClass` + `handler: runsc` là cơ chế đó. Đây cũng chính là kỹ thuật được nhắc trong domain **"Minimize Microservice Vulnerabilities"** của kỳ thi **CKS** (chiếm tỷ trọng lớn nhất trong đề thi) — sandbox runtime (gVisor/Kata) + RuntimeClass là 1 trong các công cụ chính để cô lập workload không tin cậy ở cấp cluster.

## How it works (flow/diagram)

```
kubelet → CRI (containerd hoặc CRI-O)
             │
             ├─ runtime "runc"  → chạy thẳng, share kernel host
             └─ runtime "runsc" → containerd-shim-runsc-v1 → runsc → Sentry+Gofer (kernel riêng)
                                                                     │
                                                          RuntimeClass "gvisor"
                                                          (Pod: runtimeClassName: gvisor)
```

### Setup containerd (được support chính thức, tối thiểu containerd **1.3.9 hoặc 1.4.3**)

`/etc/containerd/config.toml`:
```toml
version = 2
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
  runtime_type = "io.containerd.runsc.v1"
```
Cần cài CNI plugin, restart containerd. Sau đó apply `RuntimeClass` (`handler: runsc`) + Pod có `runtimeClassName: gvisor`.

### Setup CRI-O (⚠️ best-effort, KHÔNG được gVisor test/support chính thức — tối thiểu CRI-O v1.37)

Điểm khác biệt quan trọng so với containerd:
- `/etc/containerd/runsc.toml` cần `grouping = true` — **bắt buộc** để các container con trong cùng Pod attach vào chung 1 shim của pause container, thay vì mỗi container tự start 1 shim riêng (nếu thiếu → container start fail).
- CRI-O config cần `selinux = false` — gVisor không support gán SELinux label lên container của nó, để SELinux enabled sẽ khiến tạo container fail.
- CRI-O config cần `runtime_type = "vm"` — báo cho CRI-O biết dùng model "VM shim" (tạo pause/infra container **trước**, gVisor cần nó tồn tại trước khi app container start) — thiếu field này CRI-O bỏ qua bước tạo sandbox và Pod fail.

### Advanced containerd config đáng nhớ

- `[runsc_config]` trong `runsc.toml`: mọi `key = "value"` ở đây tự động thành `--key=value` khi gọi `runsc` — tra flag khả dụng bằng `runsc flags`.
- **Chia sẻ volume giữa nhiều container trong 1 Pod + cần inotify hoạt động đúng** (vd. sidecar watch file do container khác ghi): mặc định gVisor mount volume độc lập theo từng container (không có view chung của cả Pod) → thay đổi từ container A **không tự trigger inotify** ở container B dù cuối cùng cũng đọc được data khi truy cập lại. GKE tự động set "mount hint" annotation cho việc này; trên **EKS hoặc distro khác phải tự cấu hình thủ công**:
  1. Cho phép containerd forward annotation gVisor: `pod_annotations = ["dev.gvisor.*"]` trong config runtime.
  2. Set annotation `dev.gvisor.spec.mount.<NAME>.share: "pod"` (dùng chung trong Pod) + `.type: "tmpfs"` (cho emptyDir, tăng performance) + `.options: "rw,rprivate"`.
- NVIDIA container runtime + runsc: `nvidia-ctk runtime configure` tạo config mặc định **không tương thích thẳng với runsc** — phải tự sửa `runtime_type` thành `io.containerd.runsc.v1` và chuyển cách truyền config từ `BinaryName` trong `options` sang `ConfigPath` trỏ tới file `runsc.toml` (trong đó mới ghi `binary_name = "/usr/bin/nvidia-container-runtime"`).

## Config gotchas

- Lỗi `RuntimeHandler "runsc" not supported`: thường do **Docker vẫn cài song song containerd** trên node dùng `kubeadm`, và kubeadm ưu tiên Docker nếu cả 2 cùng tồn tại → phải chỉ rõ `--cri-socket=/var/run/containerd/containerd.sock` khi `kubeadm init`, hoặc sửa `/var/lib/kubelet/kubeadm-flags.env` (`--container-runtime=remote`, `--container-runtime-endpoint=...containerd.sock`) cho cluster đã tồn tại.
- Container name lookup (DNS) qua Docker user-defined bridge **lỗi** do embedded DNS ở host network namespace (netstack cô lập không với tới) — trên **Kubernetes thì không bị vấn đề này**, name resolution hoạt động bình thường.

## GKE Sandbox — khác biệt so với gVisor "trần" (quan trọng nếu vận hành trên GKE)

Theo tài liệu GKE chính thức (`docs.cloud.google.com/kubernetes-engine/docs/concepts/sandbox-pods`, kiểm tra lại ngày 2026-09-06):
- **Yêu cầu cluster**: Standard cluster cần **ít nhất 2 node pool** (1 pool phải KHÔNG bật GKE Sandbox); **không bật được trên default node pool**; node phải chạy `cos_containerd` (Container-Optimized OS); **không hỗ trợ Windows node pool**.
- **GPU**: chỉ từ GKE `1.29.2-gke.1108000+`, chỉ workload CUDA, **không hỗ trợ NVIDIA V100/P100**, chỉ 1 số driver version nhất định, **GPU time-sharing không khuyến nghị** (Pod isolation không đầy đủ khi share GPU), **RDMA/IMEX không hỗ trợ native**.
- **TPU**: từ `1.31.3-gke.1111001+`, hỗ trợ V4/V4lite/V5litepod/V5pod/V6e — nhưng **GKE Sandbox không giảm thiểu hết mọi lỗ hổng TPU driver**.
- **Feature Kubernetes KHÔNG tương thích khi bật GKE Sandbox**: hostPath volume, privileged container, VolumeDevices, port-forward, **Seccomp/AppArmor/SELinux profile riêng** (gVisor tự quản lý cô lập, không cho set thêm các LSM này), sysctl, `NoNewPrivileges`, bidirectional MountPropagation, ProcMount, Traffic Director, Cloud Service Mesh (trên Autopilot).
- **Capability**: mặc định **chặn raw socket**; muốn dùng phải thêm `NET_RAW` tường minh — nhưng trên **Autopilot, `NET_RAW` bị chặn cứng**, không xin thêm được.

Đây là danh sách **restriction riêng của GKE Sandbox** (do policy của GKE áp thêm), khác với giới hạn chung của gVisor upstream ở [[gvisor--common-errors-lessons]] — khi audit 1 cluster GKE, phải đối chiếu cả 2 danh sách.

## Refs
- `g3doc/user_guide/containerd/quick_start.md`, `.../containerd/crio.md`, `.../containerd/configuration.md`, `.../quick_start/kubernetes.md` (repo `google/gvisor`, nhánh `master`)
- GKE Sandbox docs (kiểm tra lại định kỳ vì GKE version support thay đổi liên tục): https://docs.cloud.google.com/kubernetes-engine/docs/concepts/sandbox-pods
