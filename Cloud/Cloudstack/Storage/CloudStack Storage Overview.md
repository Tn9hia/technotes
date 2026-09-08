---
tags:
  - cloudstack
  - storage
---

# CloudStack Storage Overview

## Phân loại storage trong CloudStack

| Loại | Chứa gì | Gắn vào cấp nào | Tương đương VMware |
|---|---|---|---|
| **Primary Storage** | Disk (volume) của VM đang chạy — root disk + data disk | Cluster (hoặc Zone-wide nếu là "Zone-wide primary storage") | Datastore |
| **Secondary Storage** | Template, ISO, Snapshot, Volume backup | Zone (dùng chung cho mọi Pod/Cluster trong Zone) | Gần giống Content Library + kho snapshot/backup gộp lại |
| **Local Storage** | Disk local trên từng Host, không chia sẻ | Host | Datastore local (ít dùng cho production nghiêm túc) |

> [!tip] Khác biệt tư duy lớn nhất so với VMware
> vSphere coi Datastore là khái niệm tương đối đồng nhất (VMFS/NFS đều là "Datastore", thao tác qua Storage vMotion dễ dàng). CloudStack **tách bạch rõ ràng Primary vs Secondary** — hai thứ có vai trò, giao thức, và vòng đời quản lý khác nhau hẳn. Di chuyển volume giữa Primary và Secondary (VD: khi tạo snapshot, tạo template từ volume) luôn đi qua **SSVM**, không phải thao tác trực tiếp storage-to-storage như Storage vMotion.

## Storage Tags — cơ chế định tuyến volume tới đúng storage

CloudStack dùng **Storage Tags** để quyết định VM/volume nào được đặt vào Primary Storage nào — gắn tag vào cả Primary Storage Pool và Disk/Service Offering, CloudStack chỉ chọn pool có tag khớp.

```bash
# Gắn tag khi thêm primary storage
cmk createStoragePool name=ceph-fast zoneid=<zone-id> podid=<pod-id> \
  clusterid=<cluster-id> url=rbd://... tags=ssd,fast

# Disk Offering yêu cầu tag tương ứng
cmk createDiskOffering name="SSD 100GB" disksize=100 tags=ssd,fast
```

> [!tip] So với VMware Storage Policy
> Storage Tags gần giống **VM Storage Policy** kết hợp với **Storage DRS Datastore Cluster** — cùng mục đích "đặt đúng workload vào đúng tier storage", nhưng CloudStack làm bằng string tag đơn giản, không có rule engine phức tạp như SPBM (Storage Policy Based Management).

> [!warning] Lesson learned: quên tag → CloudStack chọn storage "bất kỳ" khớp điều kiện tối thiểu
> Nếu Disk Offering không set tag, CloudStack sẽ chọn **bất kỳ** Primary Storage nào còn đủ dung lượng trong cluster — kể cả storage tier thấp (HDD chậm) mà bạn dự định chỉ dùng cho backup/archive. Hậu quả: VM production "ngẫu nhiên" nằm trên storage chậm mà không ai chủ đích đặt vào đó. Luôn tag rõ ràng mọi Primary Storage ngay khi thêm vào, đừng để mặc định.

## Zone-wide vs Cluster-wide Primary Storage

- **Cluster-wide**: chỉ host trong 1 cluster thấy được — mặc định, phù hợp NFS/local SAN riêng từng cluster.
- **Zone-wide**: mọi cluster trong zone (cùng hypervisor) đều thấy — thường dùng khi backend là **Ceph/RBD** hoặc **SAN lớn dùng chung**, cho phép migrate VM linh hoạt hơn giữa các cluster.

## Vòng đời 1 volume

```
Tạo VM từ Template
        │
Template được copy từ Secondary Storage → Primary Storage (qua SSVM, lần đầu cho mỗi cluster)
        │
Root Volume được tạo trên Primary Storage (từ template đã cache)
        │
VM chạy, ghi/đọc trực tiếp Primary Storage (KHÔNG qua SSVM khi đang chạy)
        │
Khi tạo Snapshot → Volume snapshot có thể lưu tại Primary (nếu hỗ trợ) rồi backup lên Secondary Storage
```

> [!info] "Template Cache" trên Primary Storage
> Lần đầu deploy VM từ 1 template trong 1 cluster, CloudStack phải **copy nguyên template** từ Secondary Storage vào Primary Storage của cluster đó trước (qua SSVM) — việc này chậm. Các VM sau đó dùng chung bản cache này (copy-on-write tùy hypervisor). Đây là lý do **VM đầu tiên** từ 1 template mới trong 1 cluster luôn deploy chậm hơn hẳn các VM sau.

## Xem chi tiết

- [[Primary Storage Backends]] — NFS, Ceph/RBD, Local, iSCSI/SAN, ưu nhược điểm
- [[Secondary Storage, Snapshots & Backups]] — NFS/S3, snapshot, backup

---
*Xem thêm: [[Zones, Pods, Clusters & Hosts]] | [[System VMs - SSVM, CPVM & Virtual Router]] | [[Cloudstack|CloudStack]]*
