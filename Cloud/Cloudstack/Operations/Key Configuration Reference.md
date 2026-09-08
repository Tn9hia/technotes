---
tags:
  - cloudstack
  - operations
  - configuration
---

# Key Configuration Reference

Tổng hợp các file cấu hình và Global Settings **hay phải đụng tới nhất** khi vận hành — dùng để tra cứu nhanh, không cần nhớ hết chi tiết từng note khác.

## File cấu hình theo vị trí

| File | Nằm ở đâu | Chứa gì |
|---|---|---|
| `/etc/cloudstack/management/db.properties` | Management Server | Thông tin kết nối MySQL/Galera |
| `/etc/cloudstack/management/server.properties` | Management Server | `cluster.node.IP`, các setting nội bộ MS |
| `/etc/cloudstack/agent/agent.properties` | KVM Host | `guid`, `host` (IP của MS/LB), tên bridge network |
| `/etc/cloudstack/management/log4j-cloud.xml` | Management Server | Cấu hình log level, log rotation |
| `/etc/exports` (hoặc tương đương) | NFS Server | Quyền truy cập NFS Primary/Secondary Storage |
| `grastate.dat` | Mỗi Galera node (`/var/lib/mysql/`) | `seqno` — dùng để xác định node nào mới nhất khi bootstrap lại cluster |

## Global Settings hay dùng nhất (theo nhóm)

### Capacity & Overcommit

```properties
cpu.overprovisioning.factor              # mặc định 1 — hệ số overcommit CPU
mem.overprovisioning.factor              # mặc định 1 — hệ số overcommit RAM
storage.overprovisioning.factor          # hệ số thin-provision primary storage
cluster.cpu.allocated.capacity.notificationthreshold     # % cảnh báo CPU cluster
cluster.memory.allocated.capacity.notificationthreshold  # % cảnh báo RAM cluster
pool.storage.capacity.notificationthreshold               # % cảnh báo primary storage
secondary.storage.capacity.notificationthreshold           # % cảnh báo secondary storage
```

### Lifecycle & Cleanup

```properties
expunge.delay          # giây trước khi VM bị xóa (destroy) bị xóa vĩnh viễn
expunge.interval       # tần suất chạy job expunge
storage.cleanup.enabled
storage.cleanup.interval
```

### Networking

```properties
router.template.kvm / .vmware / .xenserver   # template dùng tạo Virtual Router theo hypervisor
network.dhcp.nondefaultnetwork.setgateway
router.aggregation.command.each.timeout      # timeout khi MS đẩy loạt lệnh xuống VR
network.dns.basiczone.updates
```

### CPU / Migration

```properties
guest.cpu.mode          # custom/host-model/host-passthrough
guest.cpu.model          # baseline CPU model khi mode=custom (tương tự EVC)
```

### API & Session

```properties
integration.api.port
session.timeout
api.request.timeout
```

## Port thường cần mở (firewall/security group giữa các thành phần)

| Từ | Đến | Port | Mục đích |
|---|---|---|---|
| Client/Browser | Management Server | 8080 (HTTP) / 8443 (HTTPS) | UI/API |
| Management Server | KVM Host | 8250 | Agent communication |
| Management Server | KVM Host | 22 | SSH (setup ban đầu, một số thao tác) |
| KVM Host | Management Server | 8250 | Agent kết nối ngược |
| Management Server | vCenter (nếu hypervisor VMware) | 443 | vCenter SDK API |
| Management Server | MySQL/Galera | 3306 | DB |
| Galera node ↔ node | Galera node | 4567, 4568, 4444 | Replication, IST/SST |
| Management Server | System VM (Link Local) | 3922 (SSH) | Debug system VM |
| Bất kỳ | Secondary Storage (NFS) | 2049, 111 | NFS |

> [!warning] Đây là bề mặt tấn công cần rà soát — xem thêm [[CloudStack Security Considerations]]
> Danh sách port trên cũng chính là checklist cần đối chiếu khi làm hardening/audit firewall — port nào không cần mở ra ngoài phạm vi tối thiểu cần thiết thì nên chặn.

## Lệnh tra cứu nhanh

```bash
# Xem 1 global setting
cmk list configurations name=<key>

# Xem toàn bộ setting đã bị đổi khác default (hữu ích khi audit hệ thống bàn giao)
cmk list configurations | jq '.configuration[] | select(.value != .defaultvalue)'

# Đổi setting (nhắc lại: nhiều setting cần restart management service)
cmk updateConfiguration name=<key> value=<value>
systemctl restart cloudstack-management
```

> [!tip] Khi nhận bàn giao, hãy export toàn bộ Global Settings hiện tại ra file
> `cmk list configurations` toàn bộ output nên được lưu lại thành 1 file tham chiếu ngay ngày đầu nhận bàn giao — đây là "dấu vân tay cấu hình" của hệ thống, giúp bạn biết chính xác cái gì đã bị tùy chỉnh khác default trước khi bắt đầu thay đổi bất cứ thứ gì.

---
*Xem thêm: [[CloudStack Management Server]] | [[CloudStack Security Considerations]] | [[Lessons Learned & Common Pitfalls]] | [[Cloudstack|CloudStack]]*
