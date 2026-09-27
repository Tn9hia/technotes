---
tags:
  - qemu
  - storage
---

# Disk Image Formats — qcow2 vs raw

**raw** là format đơn giản nhất — file (hoặc block device) chứa dữ liệu disk y hệt byte-for-byte, không có metadata gì thêm. **qcow2** (QEMU Copy-On-Write v2) là format riêng của QEMU, hỗ trợ thin-provisioning, internal snapshot, compression, encryption — đánh đổi lấy một lớp overhead translate (cluster mapping) mà raw không có.

## When — chọn format nào

| Tiêu chí | raw | qcow2 |
|---|---|---|
| Hiệu năng thuần túy | Cao nhất (không overhead metadata) | Thấp hơn raw một chút (cluster lookup) |
| Thin-provisioning (chỉ chiếm dung lượng thực dùng) | Không (trừ khi dùng sparse file, quản lý thủ công) | Có, built-in |
| Internal snapshot | Không hỗ trợ | Có (`virsh snapshot-create`) |
| Backing file (chain nhiều lớp COW) | Không | Có — nền cho clone nhanh từ template |
| Dùng trên block device thật (LVM LV, RBD) | Phù hợp nhất — ít lớp translate | Có thể, nhưng thường không cần (block device đã lo phần thin-provisioning ở tầng dưới) |
| Production database/latency-sensitive | Khuyến nghị khi cần hiệu năng tối đa | Vẫn dùng được nếu không cần snapshot chồng chéo |

> [!tip] Quy tắc chọn nhanh
> Nếu backend đã là **block device thật** (LVM LV riêng cho từng VM, hoặc Ceph RBD image riêng — xem [[Ceph RBD with KVM]]), dùng **raw** trực tiếp trên block device đó — tầng dưới (LVM thin, RBD) đã tự lo thin-provisioning/snapshot, thêm qcow2 chồng lên là dư thừa 1 lớp overhead. Nếu backend là **file trên filesystem thường** (ext4/XFS trên local disk hoặc NFS), dùng **qcow2** để có thin-provisioning + snapshot ngay ở tầng ảnh disk.

## How — cấu trúc qcow2

```
qcow2 file
   │
   ├─ Header (version, size, tên backing file nếu có, cluster size)
   ├─ L1/L2 table (2 tầng lookup: offset ảo → cluster thật trong file)
   └─ Data clusters (chỉ tồn tại clusters ĐÃ được viết — đây là bản chất thin-provisioning)
```

Khi guest đọc 1 sector chưa từng viết, QEMU tra L1→L2 table, thấy chưa map → trả về toàn số 0 mà **không cần đọc gì từ disk thật**. Khi guest viết lần đầu vào 1 vùng mới, QEMU cấp phát cluster mới trong file, cập nhật L2 table — đây là lý do file qcow2 "phình" dần theo dữ liệu thực ghi, không chiếm hết dung lượng khai báo ngay từ đầu.

### Backing file — nền cho clone nhanh từ template

```bash
# Tạo base image (template) — chỉ đọc, không sửa trực tiếp sau khi dùng làm backing
qemu-img create -f qcow2 ubuntu-24.04-base.qcow2 20G

# Tạo VM mới CHỈ lưu phần khác biệt so với base — clone gần như tức thì, tiết kiệm dung lượng
qemu-img create -f qcow2 -F qcow2 -b ubuntu-24.04-base.qcow2 vm01-disk.qcow2
```

```
vm01-disk.qcow2 (chỉ chứa phần đã thay đổi so với base)
   │
   └─ backing file → ubuntu-24.04-base.qcow2 (dữ liệu gốc, read-only theo convention)
```

> [!warning] Lesson learned: sửa trực tiếp vào base image sau khi đã có VM dùng nó làm backing file
> Nếu base image bị sửa (VD chạy `apt upgrade` trực tiếp trên file base rồi mount lại) sau khi các VM con đã tạo backing chain từ nó, mọi VM con **đọc dữ liệu không nhất quán** — vì chúng giả định base file bất biến. Luôn coi base/template image là **read-only sau khi đã dùng làm backing** — nếu cần update template, tạo version mới, không sửa file cũ.

## Key Config — cache mode & preallocation

```bash
# Tạo qcow2 với preallocation='metadata' — tạo trước L1/L2 table, giảm fragmentation khi ghi lần đầu
qemu-img create -f qcow2 -o preallocation=metadata disk.qcow2 50G

# Chuyển đổi format (VD raw sang qcow2 để thêm snapshot capability)
qemu-img convert -f raw -O qcow2 disk.raw disk.qcow2

# Kiểm tra thông tin + backing chain của 1 image
qemu-img info disk.qcow2
qemu-img info --backing-chain disk.qcow2
```

Chi tiết cache mode (`cache=none/writeback/writethrough`) áp dụng cho cả raw và qcow2 — xem [[IO Tuning - Cache Modes & IOThreads]].

## Gotchas & Lessons Learned

> [!warning] Lesson learned: cluster size mặc định (64K) không phải luôn tối ưu
> qcow2 mặc định `cluster_size=65536` (64K) — hợp lý cho đa số trường hợp, nhưng với workload random I/O nhỏ (database) cluster size lớn có thể gây write amplification (ghi 4K logic nhưng phải đọc/ghi cả cluster 64K trong một số kịch bản copy-on-write). Không có công thức cố định — nếu nghi ngờ, benchmark thực tế với `fio` trước khi đổi, đừng đổi cluster size "theo cảm tính" trên production.

> [!tip] `qemu-img check` trước khi backup/migrate image quan trọng
> qcow2 có thể bị corrupt L2 table nếu host crash giữa lúc ghi (hiếm nhưng có thể). `qemu-img check disk.qcow2` phát hiện lỗi cấu trúc trước khi tin tưởng backup hoặc migrate image đó sang hệ thống khác.

## Resources

- `man qemu-img`
- QEMU qcow2 format spec: `docs/interop/qcow2.txt` trong source QEMU

---
*Xem thêm: [[Storage Backends Overview]] | [[Snapshots & Backup]] | [[IO Tuning - Cache Modes & IOThreads]] | [[Kvm-virtualization|KVM Virtualization]]*
