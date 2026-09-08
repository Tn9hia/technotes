---
tags:
  - cloudstack
  - core-components
  - management-server
---

# CloudStack Management Server

Management Server (MS) là **control plane** của CloudStack — một ứng dụng Java (Spring/Tomcat) chạy độc lập, chịu trách nhiệm nhận API request, orchestration, scheduling, và ghi state vào MySQL.

> [!tip] So với vCenter
> MS đóng vai trò tương tự **vCenter Server**, nhưng có 2 khác biệt lớn:
> 1. **Stateless & scale-out**: MS không lưu state cục bộ (mọi state nằm trong MySQL), nên có thể chạy **nhiều instance MS song song sau load balancer** để HA/scale — không giống vCenter (1 instance, HA bằng vCenter HA/VCHA riêng biệt, phức tạp hơn nhiều).
> 2. **Không nằm trong data path**: giống vCenter, MS chết không làm VM đang chạy bị ảnh hưởng — chỉ mất khả năng control (tạo/xóa/thay đổi).

## Kiến trúc

```
        Load Balancer (VIP, tùy chọn nhưng khuyến nghị production)
              │
      ┌───────┴───────┐
   MS instance 1   MS instance 2  ...
      │                  │
      └────────┬─────────┘
                │
        MySQL/MariaDB (cloud, cloud_usage database)
                │
        (Galera cluster nếu cần HA DB)
```

### Thành phần bên trong MS

| Thành phần | Chức năng |
|---|---|
| API Server | Nhận REST API (HMAC-signed hoặc OAuth/keystone tùy config), validate |
| Orchestration Engine | Điều phối vòng đời VM, network, storage |
| Allocators/Planners | Chọn host/storage để đặt VM (tương tự DRS placement nhưng đơn giản hơn) |
| Job/Async Framework | Toàn bộ tác vụ dài (deploy VM, snapshot...) chạy async, trả về `jobid` để poll |
| Usage Server | Tính toán usage record để billing (thường chạy như 1 service riêng `cloudstack-usage`) |

> [!info] Async Job — điều khác biệt lớn khi thao tác API
> Hầu hết API CloudStack **trả về ngay `jobid`**, không đợi tác vụ hoàn tất (khác PowerCLI/vSphere API vốn có API đồng bộ lẫn bất đồng bộ tùy hàm). Bạn luôn phải `queryAsyncJobResult` để biết kết quả thật sự — quên bước này là nguyên nhân phổ biến khiến script tự động "tưởng thành công" trong khi job đang fail.

## VM Lifecycle qua Management Server

```
   deployVirtualMachine (API)
          │  validate quota, offering, network
          │
   Deployment Planner ──► chọn Zone/Pod/Cluster/Host phù hợp
          │
   ghi state vào DB (Starting)
          │
   gửi lệnh tới Host Agent (KVM) hoặc vCenter API (VMware)
          │
   Host tạo VM qua libvirt/qemu (KVM) hoặc vCenter tạo (VMware)
          │
   Agent báo trạng thái ngược về MS qua heartbeat
          │
   VM = Running ✓
```

## Cấu hình quan trọng

File cấu hình chính: `/etc/cloudstack/management/server.properties` và `db.properties`.

```properties
# db.properties — kết nối MySQL
db.cloud.host=127.0.0.1
db.cloud.port=3306
db.cloud.name=cloud
db.cloud.driver=jdbc:mysql

# server.properties
cluster.node.IP=<IP của chính MS này>   # bắt buộc đúng khi chạy nhiều MS!
```

> [!warning] Lesson learned: `cluster.node.IP` sai là "quả bom hẹn giờ"
> Khi scale ra nhiều MS, nếu `cluster.node.IP` bị cấu hình sai (ví dụ copy nguyên file server.properties từ node 1 sang node 2 mà quên sửa IP), các MS sẽ **đăng ký nhầm địa chỉ trong bảng `mshost`**, dẫn đến hiện tượng: job bị "kẹt" ở trạng thái pending vì MS khác tưởng job thuộc về node đã chết. Triệu chứng rất khó đoán ra nguyên nhân gốc nếu không biết trước. Luôn kiểm tra `SELECT * FROM cloud.mshost;` sau khi thêm MS mới.

## Global Settings

Global Settings (`cmk list configurations` hoặc UI: Infrastructure > Global Settings) là kho cấu hình runtime quan trọng nhất — tương tự **Advanced System Settings** của vCenter nhưng phạm vi ảnh hưởng rộng hơn nhiều (network, storage, security, API...).

```bash
# Xem 1 setting
cmk list configurations name=expunge.delay

# Đổi 1 setting (thường cần restart management service để áp dụng!)
cmk updateConfiguration name=expunge.delay value=86400
systemctl restart cloudstack-management
```

> [!warning] Không phải Global Setting nào cũng apply ngay
> Rất nhiều setting **chỉ có hiệu lực sau khi restart `cloudstack-management`**, một số khác chỉ áp dụng cho resource **tạo mới** (không hồi tố resource cũ). Đây là nguồn gốc của rất nhiều câu hỏi "tôi đã đổi config sao không thấy hiệu lực" — luôn kiểm tra changelog/docs của từng setting trước khi kỳ vọng nó apply real-time. Xem thêm [[Key Configuration Reference]].

## Quản lý dịch vụ

```bash
systemctl status cloudstack-management
systemctl restart cloudstack-management
journalctl -u cloudstack-management -f

# Log chính
tail -f /var/log/cloudstack/management/management-server.log
```

## HA cho Management Server

- Không cần cluster software đặc biệt (không giống Pacemaker cho OpenStack) — chỉ cần:
  1. Nhiều MS instance trỏ chung 1 MySQL (Galera).
  2. Load Balancer (HAProxy/F5/keepalived) đặt VIP phía trước, health-check port 8080/8443.
  3. Đảm bảo NTP đồng bộ giữa các MS (async job & session timestamp rất nhạy với lệch giờ).

Xem chi tiết ở [[CloudStack HA Architecture]].

---
*Xem thêm: [[Database HA - MySQL Galera]] | [[CLI & API - CloudMonkey]] | [[Key Configuration Reference]] | [[Cloudstack|CloudStack]]*
