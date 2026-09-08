---
tags:
  - cloudstack
  - identity
  - offerings
---

# Service, Disk & Network Offerings

"Offering" trong CloudStack là các **template định nghĩa trước** cho VM/disk/network — tương đương khái niệm "Flavor" bên OpenStack, hoặc gần giống **VM Compute Policy / Storage Policy** kết hợp bên VMware, nhưng bao trùm cả network nữa.

## Service Offering (= "Flavor" của VM)

Định nghĩa kích thước VM: vCPU, RAM, và các thuộc tính nâng cao.

```bash
cmk createServiceOffering name="Standard-4C8G" displaytext="4 vCPU / 8GB RAM" \
  cpunumber=4 cpuspeed=2000 memory=8192 \
  storagetype=shared offerha=true

# Custom offering (cho phép user tự chọn CPU/RAM lúc deploy, giống "custom size" của AWS/GCP)
cmk createServiceOffering name="Custom" iscustomized=true storagetype=shared
```

| Thuộc tính | Ý nghĩa | So sánh VMware |
|---|---|---|
| `offerha` | Bật CloudStack HA — VM tự restart trên host khác nếu host chết | Giống vSphere HA nhưng cấu hình theo từng offering, không phải cluster-wide toggle đơn giản |
| `storagetype` | `shared` hay `local` — quyết định VM có migrate được không | Giống chọn Datastore shared vs local |
| `hosttags` | Ràng buộc VM chỉ chạy trên host có tag tương ứng | Gần giống Host DRS Group/Affinity Rule |
| `deploymentplanner` | Chiến lược đặt VM (spread, pack...) | Gần giống DRS load balancing policy nhưng đơn giản hơn |

> [!warning] Lesson learned: đổi `offerha` sau khi VM đã chạy không có tác dụng hồi tố tự động rõ ràng
> Thay đổi thuộc tính HA trên 1 Service Offering **đã có VM đang dùng** không phải lúc nào cũng áp dụng ngay cho VM cũ — nhiều thuộc tính chỉ chốt tại thời điểm resize/redeploy VM sang offering đó. Muốn chắc chắn 1 VM production có bật HA, kiểm tra trực tiếp thuộc tính của **chính VM đó** (`cmk listVirtualMachines`), đừng suy luận ngược từ tên Service Offering.

## Disk Offering (= "Flavor" của Volume)

```bash
cmk createDiskOffering name="SSD-100GB" disksize=100 \
  storagetype=shared tags=ssd \
  provisioningtype=thin
```

- `provisioningtype`: `thin`, `sparse`, `fat` — tương tự khái niệm Thin/Thick Provision Lazy/Eager Zeroed bên VMware nhưng ít lựa chọn hơn.
- Có thể set **IOPS min/max** nếu storage backend hỗ trợ QoS (VD: một số SAN/Ceph cấu hình đặc biệt) — gần giống Storage I/O Control.

## Network Offering (khái niệm không có tương đương trực tiếp bên VMware thuần)

Định nghĩa **bộ dịch vụ mạng** nào sẽ có sẵn khi tạo 1 network từ offering đó: DHCP, DNS, Firewall, LB, VPN, SourceNAT, StaticNAT, PortForwarding, và **ai cung cấp** dịch vụ đó (Virtual Router hay appliance khác).

```bash
cmk createNetworkOffering name=isolated-with-lb \
  guestiptype=Isolated traffictype=Guest \
  supportedservices=Dhcp,Dns,SourceNat,Firewall,PortForwarding,Lb,UserData,StaticNat \
  serviceproviderlist[0].service=Dhcp serviceproviderlist[0].provider=VirtualRouter \
  ... \
  specifyvlan=false conservemode=true
```

> [!tip] So với NSX Network Profile
> Network Offering gần giống việc định nghĩa trước 1 "gói dịch vụ mạng chuẩn" mà NSX-T có thể làm qua Network/Security Profile — CloudStack đóng gói sẵn thành 1 object duy nhất để chọn lúc tạo network, thay vì lắp ghép nhiều policy rời rạc.

> [!warning] Lesson learned: Network Offering không đổi được `supportedservices` sau khi network đã tạo và có VM
> Một khi network đã "implement" (có VM chạy trong đó), việc đổi sang Network Offering khác có tập dịch vụ khác **rất hạn chế** (một số update được, một số bắt buộc phải tạo network mới rồi di chuyển VM). Thiết kế Network Offering **kỹ ngay từ đầu** (đặc biệt việc có cần LB, VPN, Static NAT hay không) — sửa sau tốn công hơn nhiều so với ước tính ban đầu.

## Bảng tổng hợp 3 loại Offering

| Offering | Định nghĩa cho | Ví dụ thuộc tính chính |
|---|---|---|
| Service Offering | VM (compute) | vCPU, RAM, HA, host tag |
| Disk Offering | Volume | Dung lượng, IOPS, storage tag, provisioning type |
| Network Offering | Network | Dịch vụ mạng (DHCP/FW/LB/VPN...), nhà cung cấp dịch vụ |

---
*Xem thêm: [[Accounts, Domains & Projects (CloudStack)]] | [[CloudStack Network Architecture Overview]] | [[Primary Storage Backends]] | [[Cloudstack|CloudStack]]*
