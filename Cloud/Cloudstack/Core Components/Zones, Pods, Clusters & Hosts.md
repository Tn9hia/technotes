---
tags:
  - cloudstack
  - core-components
  - infrastructure
---

# Zones, Pods, Clusters & Hosts

Đây là mô hình phân cấp hạ tầng vật lý/logic của CloudStack — **phải hiểu chắc trước khi làm gì khác**, vì gần như mọi resource khác (network, storage, offering) đều "treo" vào một trong 4 tầng này.

## Sơ đồ phân cấp

```
Zone (site/datacenter)
 └─ Pod (rack / L2 segment)
     └─ Cluster (nhóm host share primary storage)
         └─ Host (máy vật lý chạy hypervisor)
             └─ VM Instances
 └─ Primary Storage (gắn vào Cluster)
 └─ Secondary Storage (gắn vào Zone, dùng chung cho mọi Pod/Cluster trong Zone)
 └─ Public IP range, VLAN range (gắn vào Zone hoặc Pod tùy networking mode)
```

## Zone

- Đơn vị cô lập lớn nhất, thường ánh xạ 1-1 với 1 **site/datacenter vật lý**.
- Chứa: các Pod, Secondary Storage, dải Public IP, cấu hình network mode (Basic/Advanced).
- Có 2 loại: **Zone thường** (dùng cho VM) và cân nhắc riêng nếu triển khai multi-zone cho DR.
- Zone có thể **public** (mọi domain thấy) hoặc **private/dedicated** (chỉ domain được gán mới thấy) — dùng cho multi-tenant tách biệt hạ tầng vật lý cho khách VIP.

> [!tip] So với VMware
> Zone gần giống 1 **vCenter Datacenter object**, nhưng khác biệt lớn: trong CloudStack, **networking và storage được định nghĩa ở cấp Zone**, còn vCenter Datacenter chỉ là container tổ chức — networking (dvSwitch) và storage (datastore cluster) là các object độc lập nằm ngoài. CloudStack "buộc" bạn thiết kế network/storage ngay khi tạo Zone.

## Pod

- Nhóm host dùng chung 1 dải mạng quản lý (management network) và DHCP cho System VM — về vật lý thường tương ứng **1 rack** với 1 cặp TOR switch.
- Mỗi Pod có 1 dải IP riêng cho System VM (SSVM, CPVM, VR) nằm trên management network.
- **Không có khái niệm tương đương trực tiếp bên VMware** — gần giống việc nhóm ESXi host theo rack/switch nhưng vCenter không bắt buộc bạn khai báo layer này.

> [!warning] Sai lầm thiết kế thường gặp
> Nhồi quá nhiều host khác switch/khác L2 vào 1 Pod → System VM không lấy được IP quản lý ổn định, hoặc broadcast storm khi Pod quá lớn. Nguyên tắc: **1 Pod = 1 miền broadcast quản lý thực sự**, đừng vẽ Pod theo ý muốn logic mà bỏ qua thực tế switch/VLAN.

## Cluster

- Nhóm host **chia sẻ chung Primary Storage** và cho phép **live migration** qua lại giữa các host trong cluster.
- Mỗi Cluster chỉ dùng **1 loại hypervisor duy nhất** (không trộn KVM + VMware trong cùng cluster).
- Đây là tầng tương đương gần nhất với **vSphere Cluster** — cùng tư duy: "nhóm host + shared storage + di chuyển VM tự do bên trong".

| Đặc điểm | CloudStack Cluster | vSphere Cluster |
|---|---|---|
| Shared storage bắt buộc | Không bắt buộc (có thể dùng local storage) nhưng khuyến nghị | Không bắt buộc nhưng khuyến nghị |
| DRS-like (auto load balancing) | Có nhưng đơn giản hơn (`deployment planner`), không mạnh như DRS | DRS rất mạnh, nhiều policy |
| HA tự động restart VM khi host chết | Có (CloudStack HA), cần cấu hình + fencing | vSphere HA, tích hợp sâu hơn |
| Trộn nhiều hypervisor trong 1 cluster | Không cho phép | N/A (chỉ ESXi) |

## Host

- Máy vật lý chạy hypervisor, được agent CloudStack quản lý:
  - **KVM**: cài `cloudstack-agent`, giao tiếp qua port 8250 (agent) tới Management Server.
  - **VMware**: CloudStack không cài agent lên ESXi, mà **gọi thẳng vCenter API** — nghĩa là bạn vẫn cần vCenter tồn tại, CloudStack chỉ orchestrate ở tầng trên (xem [[Hypervisor Support - KVM, VMware & Others]]).
- Trạng thái Host quan trọng cần biết: `Up`, `Down`, `Disconnected`, `Alert`, `Maintenance`.

```bash
# Liệt kê host và trạng thái
cmk list hosts listall=true

# Đưa host vào maintenance mode (VM sẽ tự động migrate đi nếu HA/cluster cho phép)
cmk prepareHostForMaintenance id=<host-id>

# Đưa host ra khỏi maintenance
cmk cancelHostMaintenance id=<host-id>
```

> [!warning] Lesson learned: Maintenance mode không tự di chuyển VM như DRS
> Khi đưa Host vào maintenance, CloudStack cố gắng **live-migrate** VM sang host khác cùng cluster, nhưng nếu không đủ tài nguyên hoặc VM dùng local storage (không migrate được), quá trình sẽ **treo ở trạng thái "PrepareForMaintenance"** vô thời hạn mà không báo lỗi rõ ràng. Luôn kiểm tra `cmk list vms hostid=<host-id>` trước, và tính toán capacity dư trong cluster trước khi bảo trì — đừng tin tưởng mù quáng như thói quen "cứ enter maintenance mode" bên vSphere.

## Capacity & Resource

CloudStack tính toán capacity theo **overcommit ratio** (CPU/RAM) ở cấp Cluster:

```bash
# Global settings quan trọng liên quan overcommit
cpu.overprovisioning.factor      # mặc định 1 (không overcommit CPU)
mem.overprovisioning.factor      # mặc định 1
storage.overprovisioning.factor  # cho thin-provision primary storage
```

> [!tip] So với VMware
> vSphere overcommit CPU/RAM gần như "ngầm định" và linh hoạt theo thời gian thực (ballooning, TPS...). CloudStack dùng **hệ số cố định (static ratio)** cấu hình sẵn ở Cluster/Zone/Global level — nghĩa là bạn phải chủ động set đúng ratio, CloudStack không tự "co giãn" thông minh như ESXi memory management.

---
*Xem thêm: [[CloudStack Management Server]] | [[Hypervisor Support - KVM, VMware & Others]] | [[Primary Storage Backends]] | [[Cloudstack|CloudStack]]*
