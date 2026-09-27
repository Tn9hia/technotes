---
tags:
  - storage
  - kvm
---

# Storage Backends Overview

KVM/libvirt không tự có "storage stack" riêng — disk của VM luôn cuối cùng là 1 trong 2 dạng: **file** trên 1 filesystem (local hoặc network), hoặc **block device** thô (LVM LV, iSCSI LUN, Ceph RBD image). Chọn backend nào ảnh hưởng trực tiếp tới khả năng live migration, snapshot, và hiệu năng.

> [!tip] So với Datastore trong vSphere
> vSphere gói mọi backend (local disk, SAN, NAS) qua 1 lớp trừu tượng chung là Datastore + VMFS/vSAN filesystem. KVM để lộ rõ backend thật hơn nhiều — bạn biết chính xác disk VM đang nằm trên file ext4 nào, LVM LV nào, hay RBD image nào, không có 1 filesystem cluster chung nào che phần khác biệt này lại (trừ khi tự dùng thêm GFS2/OCFS2 cho shared local storage, hiếm gặp).

## Where — so sánh các backend

| Backend | Loại | Shared giữa nhiều host? | Snapshot | Hiệu năng | Khi nào dùng |
|---|---|---|---|---|---|
| Local file (qcow2/raw trên ext4/XFS) | File | Không | qcow2: có; raw: không | Cao (ít overhead) | Lab, single-host, VM không cần migrate |
| LVM (logical volume, raw) | Block | Không (trừ LVM cluster phức tạp, hiếm dùng) | LVM snapshot (thick, tốn dung lượng ngay khi tạo) | Cao nhất trong nhóm local | Production single-host cần hiệu năng, không cần HA |
| NFS | File (qua network) | **Có** | qcow2: có (file nằm trên NFS) | Trung bình, phụ thuộc network/NFS server | Shared storage đơn giản, cần live migration nhưng không có SAN/Ceph |
| iSCSI | Block (qua network) | **Có** (multi-initiator) | Phụ thuộc SAN backend | Cao, phụ thuộc SAN | Đã có sẵn SAN doanh nghiệp |
| Ceph RBD | Block (qua network, distributed) | **Có**, scale-out | RBD snapshot/clone (copy-on-write, nhanh) | Cao, scale ngang theo cluster | Self-hosted production cần HA + scale-out, xem [[Ceph RBD with KVM]] |

> [!warning] Không có shared storage = không có live migration thật
> Đây là quyết định kiến trúc quan trọng nhất cần chọn **trước khi** build cluster nhiều host: nếu chọn local file/LVM (không shared), mọi VM bị "khóa" vào đúng 1 host — muốn di chuyển phải copy toàn bộ disk qua network (`virsh migrate --copy-storage-all`, chậm) hoặc offline hoàn toàn. Nếu định làm cluster KVM nhiều host có HA/live migration, **bắt buộc** chọn NFS/iSCSI/Ceph RBD ngay từ đầu — xem [[Live Migration]].

## How — kiểm tra backend đang dùng cho 1 VM

```bash
virsh dumpxml vm01 | grep -A3 "<disk"
# type='file' source file='...'      → local file (qcow2/raw)
# type='block' source dev='/dev/...' → block device (LVM LV, iSCSI LUN)
# type='network' source protocol='rbd' → Ceph RBD
```

## Key Config — cache mode ảnh hưởng theo backend

Cache mode (`cache=none/writeback/writethrough`) tương tác khác nhau tùy backend — chi tiết đầy đủ ở [[IO Tuning - Cache Modes & IOThreads]]. Điểm cần nhớ ngay: với **NFS**, `cache=none` (dùng O_DIRECT) đôi khi gặp vấn đề tương thích với 1 số NFS server cũ — luôn test kỹ trước khi đưa vào production.

## Gotchas & Lessons Learned

> [!warning] Lesson learned: trộn lẫn nhiều loại backend trong 1 cluster mà không tài liệu hóa rõ VM nào nằm đâu
> Cluster phát triển theo thời gian dễ dẫn tới tình trạng: VM cũ nằm trên local LVM (tạo thời kỳ đầu, chưa có Ceph), VM mới nằm trên RBD — khi cần migrate hoặc backup đồng bộ toàn cluster, thiếu tài liệu rõ ràng dẫn tới thao tác nhầm (VD chạy script migrate giả định mọi VM đều shared storage, fail hàng loạt với VM cũ). Luôn gắn tag/note rõ backend storage cho từng VM/host ngay từ đầu.

## Resources

- Libvirt storage driver overview: https://libvirt.org/storage.html
- Xem thêm [[Ceph|Ceph vault]] nếu backend là Ceph RBD

---
*Xem thêm: [[Libvirt Storage Pools & Volumes]] | [[Disk Image Formats - qcow2 vs raw]] | [[Ceph RBD with KVM]] | [[Live Migration]] | [[Kvm-virtualization|KVM Virtualization]]*
