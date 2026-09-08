---
tags:
  - cloudstack
  - storage
---

# Primary Storage Backends

## So sánh các backend phổ biến

| Backend | Giao thức | Live Migration | Thin Provisioning | Snapshot native | Ghi chú |
|---|---|---|---|---|---|
| **NFS** | NFSv3/v4 | Có (cùng cluster) | Có (tùy filesystem phía sau) | Qua CloudStack (copy file), không native | Dễ setup nhất, phổ biến cho triển khai vừa/nhỏ |
| **Ceph/RBD** | librbd (KVM only) | Có | Có, rất tốt | Có — Ceph snapshot native, nhanh | Phổ biến nhất cho production KVM quy mô lớn, cần cụm Ceph riêng vận hành |
| **Local Storage** | Ổ đĩa local host | **Không** (trừ cơ chế đặc biệt/offline migration) | Tùy filesystem | Không qua CloudStack tốt | Nhanh nhất về IOPS thô, nhưng mất HA/migration — dùng cho workload chấp nhận rủi ro |
| **iSCSI/SAN** (SharedMountPoint hoặc CLVM) | iSCSI | Có | Tùy SAN | Tùy SAN | Phổ biến nếu công ty đã có SAN sẵn từ thời VMware |
| **SolidFire/Datera/StorPool (plugin)** | Vendor-specific API | Có | Có | Native, tích hợp API | Dùng khi có storage vendor hỗ trợ CloudStack plugin chính thức |

> [!tip] So với VMware
> NFS/iSCSI giống hệt cách vSphere mount Datastore — không xa lạ. Ceph/RBD là khái niệm **mới với dân thuần VMware** (VMware có vSAN riêng, không dùng Ceph) — cần hiểu Ceph là 1 hệ phân tán độc lập, CloudStack chỉ là **client** của nó (qua `librbd`), không quản lý vòng đời Ceph cluster. Sự cố Ceph (OSD down, PG degraded...) phải debug ở tầng Ceph, CloudStack chỉ thấy hệ quả (I/O chậm/treo). Xem chi tiết kiến trúc/vận hành Ceph trong vault riêng [[Ceph|Ceph]], đặc biệt [[Ceph with CloudStack]] cho phần tích hợp cụ thể.

## NFS Primary Storage

```bash
# Thêm NFS primary storage vào cluster
cmk createStoragePool name=nfs-primary-01 zoneid=<zone-id> podid=<pod-id> \
  clusterid=<cluster-id> url="nfs://192.168.1.10/export/primary"
```

> [!warning] Lesson learned: NFS export permission sai = toàn cluster "Alert"
> Lỗi phổ biến nhất khi thêm NFS primary storage: quên mở quyền (`rw`, `no_root_squash`) cho **toàn bộ dải IP của các host trong cluster**, không chỉ IP của host đang thêm đầu tiên. Khi host thứ 2/3 trong cluster cố mount và fail, cluster báo "Alert" hàng loạt dù chỉ 1 host thực sự có vấn đề — luôn kiểm tra `/etc/exports` (hoặc config NFS server) cho cả dải IP cluster trước khi thêm storage.

## Ceph/RBD Primary Storage

```bash
cmk createStoragePool name=ceph-primary zoneid=<zone-id> podid=<pod-id> \
  clusterid=<cluster-id> \
  url="rbd://<cephx-user>:<secret>@<mon1>;<mon2>;<mon3>/<pool-name>"
```

Trên mỗi KVM host cần có sẵn:
```bash
# Ceph client packages
ceph-common
# libvirt secret chứa cephx key (CloudStack tự tạo khi add storage, nhưng cần verify)
virsh secret-list
```

> [!warning] Lesson learned: xoay vòng Ceph cephx key mà quên đồng bộ libvirt secret
> Nếu team hạ tầng Ceph xoay vòng (rotate) cephx key định kỳ theo policy bảo mật riêng mà không báo đội CloudStack, **libvirt secret trên từng KVM host sẽ lệch** với key thật trên Ceph cluster → VM đang chạy vẫn OK (đã mount rồi) nhưng **mọi thao tác I/O mới** (tạo volume, snapshot, VM mới) sẽ fail âm thầm hoặc timeout. Đây là 1 trong những sự cố khó chẩn đoán nhất vì triệu chứng xuất hiện trễ và không liên quan trực tiếp tới thao tác vừa làm. Luôn phối hợp chặt giữa đội Ceph và đội CloudStack khi có thay đổi credential.

## Local Storage

```bash
# Bật cho phép dùng local storage ở cấp Zone (global setting)
cmk updateConfiguration name=system.vm.use.local.storage value=false  # tùy nhu cầu
```

> [!warning] Cân nhắc kỹ trước khi dùng Local Storage cho production
> VM trên Local Storage **không live-migrate được** (mặc định), nên: không tận dụng được HA thật sự khi host chết bất ngờ (cần cấu hình đặc biệt để "VM HA on local storage" — vẫn có thời gian downtime để restart trên host khác, dữ liệu volume **mất nếu host chết vĩnh viễn** trừ khi có backup riêng). Chỉ dùng Local Storage cho: workload có thể tái tạo dễ dàng, cache node, hoặc khi đã có storage replication ở tầng ứng dụng (VD: cụm database tự replicate).

## Overprovisioning & Capacity Planning

```properties
storage.overprovisioning.factor=2.0   # thin-provision: cho phép cấp phát ảo gấp 2 lần dung lượng thật
storage.cleanup.enabled=true
storage.cleanup.interval=86400
```

> [!tip] Theo dõi capacity thật, đừng tin số hiển thị UI một cách mù quáng
> UI CloudStack hiển thị capacity dựa trên **allocated** (đã cấp phát ảo), không phải **used thật** (đặc biệt với thin provisioning). Với overprovisioning factor > 1, con số "còn trống" trên UI có thể **lạc quan hơn thực tế disk vật lý** nếu nhiều volume cùng lúc ghi đầy dữ liệu thật. Luôn đối chiếu thêm với công cụ giám sát ở tầng storage thật (Ceph `ceph df`, NFS server `df -h`) — xem thêm [[CloudStack Monitoring & Alerting]].

---
*Xem thêm: [[CloudStack Storage Overview]] | [[Secondary Storage, Snapshots & Backups]] | [[Hypervisor Support - KVM, VMware & Others]] | [[Ceph with CloudStack]] | [[Cloudstack|CloudStack]]*
