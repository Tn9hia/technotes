---
tags:
  - ceph
  - mgr
  - architecture
---

# MGR - Manager

**MGR (Manager)** là daemon chịu trách nhiệm thu thập metrics, chạy dashboard, expose API cho orchestrator (`cephadm`), và host các module mở rộng (Prometheus exporter, balancer, alerts...). MGR luôn đi kèm 1 MON trên cùng host và chạy theo mô hình **active/standby**.

> [!tip] So với VMware vSAN
> MGR gần giống tổ hợp **vSAN Health Service + một phần vROps** — nơi tập trung metrics, dashboard trực quan, và các module "thông minh" (balancer tự động cân bằng dữ liệu tương tự tinh thần DRS nhưng ở tầng storage placement). Khác biệt quan trọng: MGR **không nằm trong control plane bắt buộc** để cluster hoạt động — nó là lớp "quan sát + tiện ích", không phải lớp "quyết định trạng thái" như MON. Nếu ví MON như phần lõi giữ quorum trạng thái, MGR giống lớp dashboard/health-service ngồi cạnh quan sát và cung cấp API, không phải bộ não ra quyết định replicate dữ liệu.

## Vai trò cụ thể

- **Thu thập & tổng hợp metrics**: giữ `pgmap` (từ Luminous trở đi, MGR đảm nhiệm việc này thay vì MON để giảm tải cho MON), usage per-pool, per-OSD performance.
- **Dashboard**: web UI quản lý/giám sát built-in (module `dashboard`).
- **Prometheus exporter**: module `prometheus` expose metrics endpoint để scrape.
- **Orchestrator backend**: module `cephadm` — MGR là nơi thực thi các lệnh `ceph orch ...` (add OSD, apply MON, upgrade...).
- **Balancer**: tự động tính lại reweight/upmap để cân bằng dữ liệu đều giữa các OSD.
- **Alerts**: module `alerts` gửi cảnh báo qua email khi health thay đổi.
- Nhiều module khác: `restful`, `insights`, `telemetry`, `iostat`, `pg_autoscaler`...

```bash
# Liệt kê tất cả module, module nào đang bật
ceph mgr module ls

# Bật/tắt 1 module
ceph mgr module enable dashboard
ceph mgr module disable restful

# Xem MGR nào đang active, standby nào sẵn sàng, và endpoint các service (dashboard URL...)
ceph mgr services

# Xem chi tiết trạng thái MGR
ceph -s | grep -A2 "mgr:"
```

## Active/Standby — luôn nên có ít nhất 2

MGR chạy theo mô hình 1 active phục vụ, các MGR còn lại ở standby chờ failover. Khác MON (cần quorum số lẻ để đồng thuận), MGR **không cần Paxos** — chỉ cần MON chỉ định ai là active, failover sang standby diễn ra gần như tức thời khi active chết.

```bash
# Triển khai 2+ MGR qua cephadm (khuyến nghị chuẩn — luôn có standby)
ceph orch apply mgr --placement="mon-a,mon-b"

ceph orch ps --daemon-type mgr
```

## Blast radius: mất MGR khác mất MON như thế nào

Đây là điểm quan trọng nhất cần nắm để không hoảng loạn sai chỗ khi có sự cố:

| Sự kiện | I/O của VM/client (RBD/CephFS/RGW) | Cluster health thay đổi được không | Dashboard/metrics/orchestrator |
|---|---|---|---|
| **Mất quorum MON** (dưới majority) | Vẫn tiếp tục (client dùng map đã cache) | **Không** — mọi thay đổi map bị khóa (tạo pool, OSD lên/xuống không được xử lý đúng) | Vẫn hoạt động nếu MGR OSD MON còn sống, nhưng dữ liệu hiển thị có thể cũ dần |
| **Toàn bộ MGR chết** (không còn active lẫn standby) | **Vẫn tiếp tục bình thường** — MGR không nằm trong data path | Có, MON vẫn xử lý map bình thường | **Mất hoàn toàn**: dashboard không truy cập được, `ceph orch` không chạy được (mọi lệnh orchestrator phụ thuộc MGR), Prometheus không có dữ liệu mới, balancer ngừng hoạt động |

> [!warning] Lesson learned: mất toàn bộ MGR không làm sập I/O nhưng làm "mù" toàn bộ vận hành
> Vì MGR không nằm trong đường dữ liệu, khi tất cả MGR daemon chết, **VM/ứng dụng phía CloudStack vẫn đọc/ghi RBD bình thường** — dễ khiến người vận hành chủ quan tưởng "không sao". Nhưng thực tế bạn đã mất: dashboard, mọi lệnh `ceph orch` (không thêm/xóa OSD, không apply thay đổi placement được), Prometheus exporter (mất alerting nếu phụ thuộc vào đó), và balancer (dữ liệu có thể dần lệch tải mà không ai biết vì không còn ai tính lại). Đây là lý do luôn cần **tối thiểu 2 MGR** (active + standby) trên các host khác nhau, và cần phân biệt rõ 2 loại sự cố khi debug: "cluster không nhận lệnh quản trị mới" (nghi MON trước) và "cluster mất dashboard/API nhưng lệnh `ceph -s` qua CLI trực tiếp trên MON vẫn ra kết quả" (nghi MGR trước).

## Kiểm tra nhanh sức khỏe MGR

```bash
# MGR active hiện tại và standby
ceph mgr dump

# Test dashboard endpoint còn phản hồi không
ceph mgr services | grep dashboard

# Nếu orch không phản hồi (treo), thường do MGR active bị treo/OOM — restart MGR active
# (an toàn vì standby sẽ nhận active gần như ngay lập tức)
ceph orch daemon restart mgr.<id>
```

---
*Xem thêm: [[MON - Monitor]] | [[Ceph Monitoring & Alerting]] | [[RADOS & Cluster Architecture]] | [[Ceph|Ceph]]*
