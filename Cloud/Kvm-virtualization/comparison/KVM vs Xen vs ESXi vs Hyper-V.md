---
tags:
  - comparison
  - kvm
---

# KVM vs Xen vs ESXi vs Hyper-V

## Bảng so sánh nhanh

| Tiêu chí | **KVM** | **Xen** | **ESXi (VMware)** | **Hyper-V (Microsoft)** |
|---|---|---|---|---|
| Kiến trúc | Kernel module trong Linux (type-1 "lai" — dùng chung kernel host) | Type-1 độc lập thật (microkernel riêng, Dom0 là 1 VM đặc biệt quản lý) | Type-1 độc lập thật (VMkernel riêng hoàn toàn) | Type-1 tích hợp vào Windows Server (root partition đặc biệt) |
| License | Miễn phí, mã nguồn mở (GPL) | Miễn phí, mã nguồn mở | Có phiên bản free giới hạn tính năng, bản đầy đủ cần license trả phí | Đi kèm Windows Server license (Windows Server Datacenter cho unlimited VM) |
| Quản lý tập trung | Không có sẵn — cần thêm libvirt + orchestrator (CloudStack/OpenStack/oVirt) | Tương tự KVM — cần XenServer/XCP-ng hoặc orchestrator riêng | vCenter (tích hợp sẵn, mạnh, nhưng license riêng) | System Center VMM hoặc Windows Admin Center |
| Hiệu năng | Rất tốt (gần native nhờ VT-x/AMD-V trực tiếp) | Rất tốt (PV driver tối ưu, tương tự virtio) | Rất tốt, tối ưu lâu năm cho enterprise workload | Tốt, tối ưu nhất cho workload Windows |
| Cộng đồng/tài liệu | Rất lớn (Linux ecosystem, CloudStack/OpenStack đều dùng) | Nhỏ hơn KVM, nhưng vẫn tích cực (AWS EC2 dùng biến thể Xen tới gần đây, XCP-ng cộng đồng khỏe) | Lớn nhất trong enterprise, tài liệu chính thức rất đầy đủ | Lớn trong ecosystem Windows/Azure |
| Live migration | Có (libvirt), cần tự chuẩn bị shared storage/network | Có, tương tự KVM | Có (vMotion) — tích hợp sẵn, UX mượt nhất | Có (Live Migration), tốt trong ecosystem Windows |
| Container tương thích | Chạy chung host chạy được cả LXC/Docker (cùng kernel Linux) | Cách ly hoàn toàn hơn (Dom0 riêng), ít tương tác trực tiếp với container host | Không áp dụng (VMkernel không phải Linux) | Chạy được Windows Container/Hyper-V Container cùng host |
| Phù hợp nhất khi | Self-hosted, muốn miễn phí, đã quen Linux, cần tích hợp CloudStack/OpenStack | Cần cách ly mạnh hơn KVM (Dom0 tách biệt), hoặc môi trường đã có sẵn Xen/XCP-ng | Doanh nghiệp đã đầu tư ecosystem VMware, cần vCenter/vSAN/NSX tích hợp sẵn | Shop chủ yếu Windows Server, đã có System Center/Azure Stack |

## Điểm khác biệt kiến trúc quan trọng nhất

> [!tip] KVM "mượn" Linux kernel, Xen/ESXi tự viết hypervisor riêng
> Đây là khác biệt gốc rễ nhất: KVM tận dụng lại toàn bộ scheduler/memory manager/driver của Linux (ưu điểm: code nhỏ, tận dụng ecosystem Linux khổng lồ; nhược điểm: bề mặt tấn công bao gồm cả bug của Linux kernel nói chung, không chỉ riêng phần virtualization). Xen và ESXi có hypervisor core **tách biệt hoàn toàn** khỏi OS quản lý (Dom0 của Xen là 1 VM đặc biệt, VMkernel của ESXi không phải Linux/Windows) — về lý thuyết bề mặt tấn công hẹp hơn cho riêng lớp hypervisor, nhưng đánh đổi bằng việc phải tự duy trì toàn bộ driver/scheduler riêng.

## Khi nào chọn KVM

- Hạ tầng **self-hosted**, ưu tiên chi phí (không license), và team đã quen Linux administration.
- Cần tích hợp với **CloudStack** hoặc **OpenStack** — cả 2 đều coi KVM là hypervisor "công dân hạng nhất" (first-class), tài liệu/cộng đồng dày nhất so với hypervisor khác trên 2 platform này (xem [[Hypervisor Support - KVM, VMware & Others]] trong vault CloudStack).
- Cần tận dụng **Ceph** làm storage backend — KVM + Ceph RBD là tổ hợp được cộng đồng self-hosted dùng/patch nhiều nhất (xem [[Ceph RBD with KVM]]).

## Khi nào KHÔNG nên chọn KVM (hoặc cần cân nhắc kỹ)

- Đã đầu tư sâu vào ecosystem VMware (vSAN, NSX, Site Recovery Manager) — chi phí migrate ra khỏi VMware thường lớn hơn lợi ích tiết kiệm license ngắn hạn.
- Team chưa quen Linux administration, chủ yếu Windows-only shop — Hyper-V tích hợp tốt hơn vào ecosystem đó, giảm chi phí học tập.
- Cần GPU vGPU chia sẻ (nhiều VM share 1 GPU) với license/support chính thức từ NVIDIA — ESXi có ecosystem vGPU trưởng thành hơn KVM ở thời điểm hiện tại (KVM hỗ trợ mdev nhưng ecosystem support thương mại mỏng hơn).

## Resources

- Xem thêm [[Ceph vs VMware vSAN & Alternatives]] trong vault Ceph — góc nhìn tương tự ở tầng storage
- Xem thêm [[Hypervisor Support - KVM, VMware & Others]] trong vault CloudStack — góc nhìn từ tầng orchestration

---
*Xem thêm: [[KVM Kernel Module & Hardware Virtualization Extensions]] | [[Kvm-virtualization|KVM Virtualization]]*
