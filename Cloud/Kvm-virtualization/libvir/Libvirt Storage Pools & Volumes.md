---
tags:
  - libvirt
  - storage
---

# Libvirt Storage Pools & Volumes

**Storage pool** là lớp abstraction của libvirt trên một storage backend thật (dir, LVM VG, NFS export, iSCSI target, Ceph RBD pool...) — mục đích để domain XML tham chiếu tới disk mà không cần biết chi tiết backend nằm ở đâu/loại gì. **Volume** là 1 đơn vị lưu trữ cụ thể trong pool đó (1 file qcow2, 1 LVM LV, 1 RBD image...).

> [!tip] So với VMFS/Datastore
> Tương tự khái niệm "Datastore" trong vSphere — pool là nơi chứa, volume là "VMDK" tương ứng. Khác biệt: libvirt storage pool chỉ là lớp khai báo/quản lý mỏng (không có filesystem cluster riêng như VMFS) — với pool loại `dir`, thực chất chỉ là 1 thư mục Linux thường; với pool loại `rbd`, libvirt chỉ gọi thẳng `librbd` xuống Ceph cluster có sẵn.

## Where — các loại pool phổ biến

| Pool type | Backend thật | Khi nào dùng |
|---|---|---|
| `dir` | Thư mục trên filesystem (ext4/XFS) | Mặc định đơn giản nhất, lab/dev, hoặc production nhỏ dùng local disk |
| `logical` | LVM Volume Group | Muốn thin/thick LVM, snapshot ở tầng LVM |
| `netfs` | NFS export | Shared storage nhiều host — cần cho [[Live Migration]] không downtime lớn |
| `iscsi` | iSCSI target/LUN | SAN truyền thống |
| `rbd` | Ceph RBD pool | Shared storage scale-out, xem [[Ceph RBD with KVM]] |

```bash
# Định nghĩa pool loại dir
virsh pool-define-as mypool dir --target /var/lib/libvirt/images/mypool
virsh pool-build mypool
virsh pool-start mypool
virsh pool-autostart mypool

# Định nghĩa pool RBD (yêu cầu Ceph cluster + keyring sẵn có)
cat > rbd-pool.xml <<EOF
<pool type='rbd'>
  <name>ceph-rbd</name>
  <source>
    <name>cloudstack-primary</name>
    <host name='mon1.example.com' port='6789'/>
    <host name='mon2.example.com' port='6789'/>
  </source>
  <auth username='libvirt' type='ceph'>
    <secret uuid='<secret-uuid-đã-tạo-qua-virsh-secret-define>'/>
  </auth>
</pool>
EOF
virsh pool-define rbd-pool.xml
virsh pool-start ceph-rbd
```

## How — vòng đời pool & volume

```
virsh pool-define → pool "inactive" (đã khai báo, chưa mount/connect)
       │
virsh pool-build   → tạo cấu trúc backend nếu cần (VD format LVM VG lần đầu — bỏ qua với rbd/netfs đã có sẵn)
       │
virsh pool-start   → pool "active" — libvirt connect vào backend thật
       │
virsh vol-create-as <pool> <vol-name> <size> → tạo volume mới trong pool
```

```bash
virsh pool-list --all
virsh vol-list ceph-rbd
virsh vol-info --pool ceph-rbd vm01-disk
virsh vol-create-as ceph-rbd vm02-disk 50G --format raw    # RBD image nên dùng raw, không cần qcow2
```

## Key Config — reference volume từ domain XML

```xml
<disk type='network' device='disk'>
  <driver name='qemu' type='raw'/>
  <source protocol='rbd' name='cloudstack-primary/vm01-disk'>
    <host name='mon1.example.com' port='6789'/>
  </source>
  <auth username='libvirt'>
    <secret type='ceph' uuid='<secret-uuid>'/>
  </auth>
  <target dev='vda' bus='virtio'/>
</disk>
```

> [!info] Vì sao RBD volume dùng `type='network'` chứ không phải `type='file'`
> Với pool `dir`/`logical`, libvirt tham chiếu volume qua đường dẫn file/block device local (`type='file'` hoặc `type='block'`). Với RBD, không có file cục bộ nào — QEMU nói chuyện trực tiếp với Ceph cluster qua `librbd` qua network, nên domain XML khai báo `type='network'` kèm protocol `rbd`. Đây cũng là lý do RBD-backed VM **không cần** host KVM mount gì trước — chỉ cần keyring/secret đúng và network tới MON.

## Gotchas & Lessons Learned

> [!warning] Lesson learned: xóa pool (`pool-destroy`) không xóa dữ liệu, nhưng `vol-delete` thì có
> `pool-destroy` chỉ ngắt kết nối pool (giống `pool-stop`), dữ liệu backend thật (file, LVM LV, RBD image) **vẫn còn nguyên** — có thể `pool-start` lại. Ngược lại `vol-delete` xóa thật volume đó khỏi backend — không có cách hoàn tác. Hai lệnh nghe tên giống nhau về mức độ "nguy hiểm" nhưng hậu quả hoàn toàn khác nhau.

> [!warning] Orchestrator (CloudStack/OpenStack) thường tự quản lý pool riêng — đừng tạo pool tay chồng lên
> Nếu KVM host đang được CloudStack quản lý (xem [[Hypervisor Support - KVM, VMware & Others]] trong vault CloudStack), Primary Storage đã được orchestrator tự tạo pool libvirt tương ứng phía sau. Tự tay `virsh pool-define` thêm 1 pool trỏ vào cùng backend dễ gây xung đột định danh (2 pool khác tên cùng trỏ 1 RBD pool) — luôn kiểm tra `virsh pool-list --all` trước khi định nghĩa pool mới trên host do orchestrator quản lý.

## Resources

- `man virsh` — phần `pool-*`, `vol-*`
- Libvirt storage driver docs: https://libvirt.org/storage.html

---
*Xem thêm: [[Storage Backends Overview]] | [[Ceph RBD with KVM]] | [[Libvirt Architecture & Domain XML]] | [[Kvm-virtualization|KVM Virtualization]]*
