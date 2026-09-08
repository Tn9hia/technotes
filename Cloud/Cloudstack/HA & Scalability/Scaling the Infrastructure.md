---
tags:
  - cloudstack
  - scalability
---

# Scaling the Infrastructure

## Mở rộng theo từng tầng

| Tầng | Cách scale | Giới hạn/lưu ý |
|---|---|---|
| Management Server | Thêm instance MS mới sau LB | Cần `cluster.node.IP` đúng, xem [[CloudStack Management Server]] |
| Database | Thêm Galera node (scale ra), hoặc scale up phần cứng | Galera hợp với vài node (3-5), không phải giải pháp scale ra vô hạn kiểu NoSQL |
| Host | Add host mới vào Cluster có sẵn, hoặc tạo Cluster mới | Cluster mới nếu khác hypervisor/storage backend |
| Storage | Thêm Primary Storage pool mới vào Cluster/Zone | Cân nhắc storage tag để phân luồng workload |
| Zone | Tạo Zone mới hoàn toàn (site mới) | Không thể "mở rộng" 1 zone thành nhiều site — phải tạo zone riêng |

## Thêm Host vào Cluster đang chạy production

```bash
# 1. Chuẩn bị host: cài OS, cấu hình bridge mạng giống các host khác trong cluster
# 2. Cài agent
apt install cloudstack-agent
# 3. Add qua UI/API — CloudStack tự cấu hình agent.properties qua SSH lúc add (hoặc cần cấu hình sẵn tùy version)
cmk addHost zoneid=<zone-id> podid=<pod-id> clusterid=<cluster-id> \
  hypervisor=KVM url="http://<host-ip>" username=root password=<pwd>
```

> [!warning] Lesson learned: mismatch version KVM/libvirt giữa host mới và host cũ trong cùng cluster
> Thêm host chạy OS/`qemu-kvm`/`libvirt` version **khác đáng kể** so với các host hiện có trong cùng cluster có thể gây lỗi live migration (không tương thích CPU feature set, hoặc libvirt version mismatch khi migrate). Nguyên tắc: host trong cùng 1 cluster nên **đồng nhất OS + package version**, nếu cần nâng cấp version mới thì nên tạo **cluster mới** rồi di chuyển VM dần, thay vì trộn lẫn trong 1 cluster.

## CPU Compatibility khi Live Migration

CloudStack (qua libvirt) yêu cầu CPU giữa các host trong cùng cluster **tương thích đủ** để live migration không bị crash VM giữa chừng.

```bash
# Kiểm tra CPU feature trên từng host
virsh capabilities | grep -A 20 "<cpu>"

# Global setting ép CloudStack dùng baseline CPU model chung cho cả cluster (an toàn hơn khi host không đồng nhất)
guest.cpu.mode=custom
guest.cpu.model=<model chung thấp nhất, VD: Haswell-noTSX>
```

> [!tip] So với vSphere EVC (Enhanced vMotion Compatibility)
> Đây chính là tương đương **EVC Mode** bên vSphere — nếu cụm host của bạn có CPU đời khác nhau (mua thêm host mới theo thời gian), nên chủ động set baseline CPU model giống tinh thần EVC, tránh việc migrate ngẫu nhiên thành công/thất bại tùy cặp host nguồn-đích.

## Scale Zone/Pod khi hết chỗ IP quản lý

Pod có dải IP quản lý cố định lúc tạo — hết IP nghĩa là **không thêm được host mới vào Pod đó**.

> [!warning] Lesson learned: quy hoạch dải IP Pod quá chật ngay từ đầu
> Nhiều triển khai ban đầu chỉ dự trù dải `/27` hoặc `/28` cho Pod (vài chục IP), đủ cho vài host + system VM lúc khởi tạo, nhưng khi mở rộng cluster thêm nhiều host, **hết IP quản lý mà không thể mở rộng dải đang dùng** (dải IP Pod không co giãn dễ dàng sau khi tạo, phụ thuộc thiết kế subnet vật lý). Khi thiết kế Pod mới, nên dự trù dư ít nhất gấp 2-3 lần số host dự kiến trong 1-2 năm tới.

## Vertical Scaling (Resize) cho VM đang chạy

```bash
# Resize VM sang Service Offering lớn hơn (cần dừng VM nếu hypervisor/OS không hỗ trợ hot-resize)
cmk scaleVirtualMachine id=<vm-id> serviceofferingid=<new-offering-id>

# Một số hypervisor/OS hỗ trợ scale RAM/CPU khi VM đang chạy (Dynamic Scaling — cần cấu hình VirtIO/driver phù hợp)
```

## Horizontal Scaling qua Autoscaling (VPC/LB)

CloudStack hỗ trợ **AutoScale** (dựa trên NetScaler hoặc LB provider hỗ trợ) để tự thêm/bớt VM theo tải sau Load Balancer — tương tự Auto Scaling Group của AWS nhưng phụ thuộc nhiều vào LB provider có hỗ trợ hay không (không phải mọi Network Offering LB đều support).

---
*Xem thêm: [[CloudStack HA Architecture]] | [[Zones, Pods, Clusters & Hosts]] | [[Hypervisor Support - KVM, VMware & Others]] | [[Cloudstack|CloudStack]]*
