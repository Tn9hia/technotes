# Kata — Hypervisors / VMM Selection
Tier: 2
Parent: [[kata-containers]]
Related: [[kata--architecture-runtime-rs]], [[kata--configuration]], [[kata--security-policy]]
Tags: #kata #hypervisor #qemu #firecracker

## What it does

Kata không tự viết hypervisor — nó điều khiển 1 trong nhiều VMM có sẵn để boot guest VM. Mỗi hypervisor tương ứng 1 file config riêng và 1 (hoặc nhiều) RuntimeClass riêng.

## Why it exists

Không có 1 VMM nào tối ưu cho mọi use-case: QEMU đầy đủ tính năng nhất (GPU, TDX, SEV-SNP) nhưng nặng và attack surface lớn nhất; Firecracker/Cloud Hypervisor nhẹ, tối giản, sinh ra cho microVM density cao nhưng thiếu tính năng (không GPU, không ACPI hotplug ở Firecracker); Dragonball tối ưu riêng cho việc chạy chung process với shim.

## How it works (bảng so sánh chính thức)

| Hypervisor | Ngôn ngữ | Kiến trúc | GPU | Intel TDX | AMD SEV-SNP | Ghi chú |
|---|---|---|---|---|---|---|
| **QEMU** | C | tất cả (x86_64, aarch64, ppc64le, s390x...) | ✅ (NVIDIA, dự án tập trung chủ yếu vào runtime `kata-qemu-nvidia-gpu-*`) | ✅ | ✅ | Best-supported cho GPU và confidential computing; attack surface lớn nhất do nhiều tính năng |
| **Cloud Hypervisor** | Rust | aarch64, x86_64 | ❌ | ❌ | ❌ | Hiện đại, modular |
| **Firecracker** | Rust | aarch64, x86_64 | ❌ | ❌ | ❌ | Tối giản, gốc từ AWS Lambda; **không hỗ trợ CPU/memory/device hotplug (không ACPI)**, không hỗ trợ VFIO |
| **Dragonball** | Rust | aarch64, x86_64 | ❌ | ❌ | ❌ | VMM built-in, chạy chung process với shim (chỉ runtime-rs); dùng **Upcall (vsock-based)** thay ACPI để hotplug — không cần guest-side ACPI state machine |
| StratoVirt | Rust | aarch64, x86_64 | ❌ | ❌ | ❌ | Ít tài liệu vận hành hơn — cần tự kiểm chứng nếu định dùng |

- **VFIO** (device passthrough) chỉ hỗ trợ trên **QEMU và Cloud Hypervisor**, không có ở Firecracker.
- **ACPI** (hotplug CPU/memory/device động) chỉ có ở QEMU và Cloud Hypervisor; Firecracker hoàn toàn không hỗ trợ hotplug.
- Mỗi hypervisor có file config riêng dạng `configuration-<hypervisor>.toml` (Go runtime) hoặc `configuration-<hypervisor>-runtime-rs.toml` (runtime-rs), và Kata chọn qua symlink `configuration.toml` → 1 trong các file này, **hoặc** qua `runtime_path`/`ConfigPath` khai báo thẳng trong containerd/CRI-O RuntimeClass.

## Config gotchas

- Đổi hypervisor **không phải sửa 1 giá trị trong config** — thường là chọn RuntimeClass khác trỏ đúng shim + đúng file config (`kata-deploy` Helm chart quản lý việc này qua `shims.<tên>.enabled`).
- Muốn dùng GPU NVIDIA với Kata → gần như bắt buộc QEMU (`kata-qemu-nvidia-gpu*` RuntimeClass) — đừng mất thời gian thử với Firecracker/Cloud Hypervisor.
- Muốn Confidential Containers (TDX/SEV-SNP) → bắt buộc QEMU, các hypervisor Rust khác **chưa hỗ trợ** tại thời điểm viết.

## Security notes

- `virtio-blk`/`virtio-scsi` backend mặc định chạy trong chính VMM (ring3/userspace) ở cả 3 VMM chính — không dùng `vhost` (kernel-space) cho QEMU dù có sẵn, vì rủi ro cao hơn nếu bị exploit (code chạy trong kernel host).
- `vhost_vsock` (kênh control QEMU↔guest) chạy **trong kernel host**; ở Dragonball/Firecracker/Cloud Hypervisor, vsock backend là **unix-domain-socket ở userspace** — bề mặt tấn công khác nhau đáng kể giữa QEMU và 3 VMM còn lại cho riêng kênh này.
- VFIO đi kèm rủi ro DMA attack, device isolation failure, firmware vulnerability — chỉ bật khi thật sự cần passthrough phần cứng, và đảm bảo IOMMU group cô lập đúng ở tầng host trước.

## Refs

- https://github.com/kata-containers/kata-containers/blob/main/docs/hypervisors.md
- Threat model chi tiết theo từng loại device/VMM: https://github.com/kata-containers/kata-containers/blob/main/docs/threat-model/threat-model.md
