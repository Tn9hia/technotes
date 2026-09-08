---
tags:
  - ceph
  - rbd
  - storage-interfaces
  - block-storage
---

# RBD - Block Storage

RBD (**RADOS Block Device**) là interface **block storage** của Ceph — cung cấp các "ổ đĩa ảo" (image) mà VM/hypervisor gắn vào như một đĩa cứng thật. Đây là interface **quan trọng nhất** với vai trò sắp tới của bạn, vì RBD chính là con đường CloudStack (KVM) dùng làm Primary Storage cho toàn bộ VM disk trong cụm.

> [!tip] So với VMware vSAN
> Một RBD image gần giống một file **VMDK** nằm trên vSAN datastore — nhưng có một khác biệt cấu trúc quan trọng: VMDK là 1 file nằm trên 1 object trong vSAN object store (có thể có nhiều component do FTT, nhưng vẫn là 1 "đơn vị"), còn RBD image **tự nó bị băm nhỏ (striped)** thành hàng nghìn RADOS object 4MB rải khắp toàn bộ OSD của cluster. Không có khái niệm "datastore" giới hạn dung lượng/host — một image 2TB có thể có I/O phục vụ đồng thời bởi hàng chục/hàng trăm OSD khác nhau, không bao giờ bị nghẽn ở "1 node chứa file".

## Anatomy của một RBD image

Khi tạo `rbd create mypool/myimage --size 100G`, Ceph không cấp phát 100GB liền một khối. Image được chia thành các **object 4MB** (mặc định, `object-size`), mỗi object là 1 object RADOS độc lập, tên dạng `rbd_data.<image-id>.<offset-in-hex>`. Object chỉ thực sự được ghi xuống OSD khi có dữ liệu thật ghi vào offset đó — đây là cơ chế **thin provisioning** tự nhiên của RBD.

```bash
# Tạo pool dành cho RBD (xem chi tiết pool ở Pools, Replication & Erasure Coding)
ceph osd pool create rbd-pool 128
rbd pool init rbd-pool

# Tạo image 100GB, thin-provisioned
rbd create rbd-pool/vm-disk-001 --size 100G --image-feature layering,exclusive-lock,object-map,fast-diff,deep-flatten

# Xem thông tin image
rbd info rbd-pool/vm-disk-001

# Resize (mở rộng online — VM cần rescan disk ở guest OS)
rbd resize rbd-pool/vm-disk-001 --size 200G

# Dung lượng thật đã dùng (khác với size cấp phát ảo)
rbd du rbd-pool/vm-disk-001
```

| Khái niệm | Ý nghĩa |
|---|---|
| `size` | Dung lượng ảo (thin), giống "provisioned size" của VMDK |
| `object-size` | Kích thước mỗi object thành phần, mặc định 4MB |
| `rbd du` | Dung lượng thật đã ghi (giống "actual usage" trên vSAN) |
| Số lượng object | `size / object-size` — image 100GB ≈ 25,600 object tiềm năng |

## Image features — vì sao chọn đúng feature quan trọng

| Feature | Vai trò | Ghi chú |
|---|---|---|
| `layering` | Cho phép clone/snapshot | Bắt buộc nếu dùng snapshot/clone (template VM) |
| `exclusive-lock` | Chỉ 1 client được ghi tại 1 thời điểm | Bắt buộc cho live migration an toàn, ngăn split-brain ghi đồng thời |
| `object-map` | Bitmap theo dõi object nào đã cấp phát | Tăng tốc `rbd du`, `rbd resize`, `rbd rm`, `flatten` — không cần scan toàn bộ image |
| `fast-diff` | Tính diff nhanh giữa 2 snapshot | Cần cho backup/replication tăng tốc, phụ thuộc `object-map` |
| `deep-flatten` | Cho phép flatten cả snapshot, không chỉ image | Cần nếu muốn xóa hẳn parent sau khi flatten toàn bộ chain |
| `journaling` | Ghi log thay đổi | Dùng cho RBD mirroring kiểu journal-based |

> [!tip] Vì sao `exclusive-lock` + `object-map` quan trọng với KVM/CloudStack
> Không có `exclusive-lock`, không có gì ngăn 2 host cùng ghi vào 1 image cùng lúc — đúng kịch bản gây hỏng dữ liệu khi live-migration bị lỗi hoặc VM "ma" chạy song song ở 2 nơi. `object-map` giúp các thao tác quản trị (resize, du, clone, rm) không phải quét toàn bộ object của image — với image hàng trăm GB, thiếu object-map khiến các lệnh này chậm bất thường.

## Snapshot & Clone (Copy-on-Write)

RBD snapshot là bản chụp **point-in-time**, gần như tức thời vì không copy dữ liệu — chỉ đánh dấu version tại thời điểm đó trong RADOS. Clone tạo ra 1 image mới, ban đầu **không chiếm thêm dung lượng**, chỉ tham chiếu (COW) tới snapshot gốc — cực kỳ hữu ích để tạo VM từ template.

```bash
# Tạo snapshot
rbd snap create rbd-pool/golden-template@v1

# Protect snapshot (bắt buộc trước khi clone)
rbd snap protect rbd-pool/golden-template@v1

# Clone ra image mới (COW — chỉ ghi phần khác biệt)
rbd clone rbd-pool/golden-template@v1 rbd-pool/vm-disk-002

# "Cắt dây" khỏi parent — copy toàn bộ dữ liệu từ parent vào chính nó
rbd flatten rbd-pool/vm-disk-002

# Xóa snapshot sau khi không còn clone nào phụ thuộc
rbd snap unprotect rbd-pool/golden-template@v1
rbd snap rm rbd-pool/golden-template@v1
```

> [!tip] So với VMware vSAN
> Tinh thần giống snapshot/linked-clone của vSphere (COW, gần như tức thời), nhưng cơ chế nền khác hẳn: vSphere dùng delta-disk file (.vmdk redo log), RBD dùng **object versioning trong RADOS** — không có "chain file" trên filesystem để bạn tự dò bằng `ls`, mọi quan hệ parent/child chỉ tồn tại trong metadata Ceph (`rbd info`, `rbd children`).

## Luồng dữ liệu: libvirt/QEMU và librbd

QEMU trên KVM host kết nối thẳng tới Ceph cluster qua thư viện **librbd** (userspace), không cần mount block device qua kernel — libvirt chỉ cần biết pool, image name và cephx secret. Đây là điểm khác biệt lớn so với iSCSI/NFS: không có tầng "mount filesystem trung gian" trên host, QEMU nói chuyện thẳng với RADOS. Chi tiết cách CloudStack cấu hình `virsh secret`, storage pool `rbd://` và các gotcha đặc thù CloudStack nằm ở [[Ceph with CloudStack]] — note này chỉ nói tới bản chất RBD.

## RBD Mirroring — DR giữa 2 cluster

RBD hỗ trợ replicate image sang 1 cluster Ceph khác (thường ở site khác) để làm DR, thông qua daemon `rbd-mirror`.

| Mode | Cơ chế | Đặc điểm |
|---|---|---|
| **Journal-based** | Ghi mọi write vào 1 journal object trước, `rbd-mirror` đọc journal để replay ở site đích | RPO gần real-time, nhưng overhead ghi double (ghi journal + ghi data) |
| **Snapshot-based** | Định kỳ tạo snapshot, chỉ đồng bộ phần diff (`fast-diff`) giữa 2 snapshot | RPO = chu kỳ snapshot (VD: 5-15 phút), overhead thấp hơn, phổ biến hơn trong bản Ceph hiện đại |

```bash
# Bật mirroring ở mức pool (mode: image hoặc pool)
rbd mirror pool enable rbd-pool image

# Bật mirroring cho 1 image cụ thể, kiểu snapshot-based
rbd mirror image enable rbd-pool/vm-disk-001 snapshot

# Kiểm tra trạng thái đồng bộ
rbd mirror image status rbd-pool/vm-disk-001
```

Có thể cấu hình **one-way** (site chính → DR) hoặc **two-way** (active-active ở mức image, nhưng cần ứng dụng tự tránh ghi đồng thời 2 site). Với CloudStack, mirroring thường dùng ở tầng "DR cho toàn bộ pool VM disk" chứ không cấu hình per-VM thủ công.

> [!warning] Lesson learned: `rbd rm` thất bại vì còn clone/snapshot phụ thuộc
> Một trong những lỗi phổ biến nhất khi dọn dẹp template/VM cũ: chạy `rbd rm pool/image` và nhận lỗi `image has snapshot(s)` hoặc cố xóa snapshot gốc trong khi vẫn còn clone (VM) đang tham chiếu tới nó — Ceph sẽ từ chối (`cannot unprotect: at least 1 snapshot is in use`). Quy trình đúng: (1) tìm toàn bộ clone phụ thuộc bằng `rbd children pool/image@snap`, (2) với mỗi clone quan trọng cần giữ lại, chạy `rbd flatten` để tách khỏi parent, (3) chỉ sau đó mới `rbd snap unprotect` rồi `rbd snap rm`, cuối cùng mới `rbd rm` image gốc. Xóa nhầm theo thứ tự ngược lại không mất dữ liệu ngay (Ceph chặn lại), nhưng gây hoang mang và tốn thời gian điều tra "tại sao xóa hoài không được" — đặc biệt nguy hiểm nếu ai đó cố "ép" bằng `--force` mà không hiểu hậu quả với các clone đang là VM chạy production.

---
*Xem thêm: [[Ceph with CloudStack]] | [[Pools, Replication & Erasure Coding]] | [[CephFS - File Storage]] | [[Ceph|Ceph]]*
