---
tags:
  - cloudstack
  - operations
  - monitoring
---

# CloudStack Monitoring & Alerting

## Nguồn dữ liệu giám sát chính

| Nguồn | Cung cấp gì |
|---|---|
| **CloudStack Alert (built-in)** | Cảnh báo capacity threshold, host disconnect, storage issue — xem qua UI/API | 
| **Usage Server** (`cloudstack-usage`) | Tổng hợp usage record (CPU/RAM/Storage/Network theo account) — nền tảng cho billing |
| **CloudStack Management log** | Exception, job failure, chi tiết lỗi API |
| **Prometheus Exporter cộng đồng** (`cloudstack_exporter`) | Expose metrics kiểu Prometheus từ CloudStack API — không phải công cụ chính thức Apache, cần đánh giá độ tin cậy/tần suất update |
| **Metrics ở tầng Host/Hypervisor** (node_exporter, libvirt_exporter) | CPU/RAM/Disk/Network thật của host KVM |
| **Metrics ở tầng Storage** (Ceph `ceph_exporter`, NFS server metrics) | Health & performance storage backend |

> [!warning] CloudStack UI/API alert KHÔNG đủ để vận hành production nghiêm túc
> Alert built-in của CloudStack tập trung vào **capacity threshold** (VD: cluster gần đầy CPU/RAM) và một số sự kiện hệ thống — **không thay thế được** giám sát hạ tầng thật (host CPU load thực tế, disk I/O latency, network packet loss). Cần xây dựng stack giám sát riêng (Prometheus + Grafana là lựa chọn phổ biến nhất trong cộng đồng) đặt bên cạnh, tương tự cách vSphere admin vẫn cần vROps/Grafana dù có vCenter Alarm sẵn.

## Capacity Threshold — cảnh báo sớm trước khi hết tài nguyên

```bash
cmk updateConfiguration name=cluster.cpu.allocated.capacity.notificationthreshold value=0.85
cmk updateConfiguration name=cluster.memory.allocated.capacity.notificationthreshold value=0.85
cmk updateConfiguration name=pool.storage.capacity.notificationthreshold value=0.85
cmk updateConfiguration name=secondary.storage.capacity.notificationthreshold value=0.85
```

> [!tip] Đặt threshold sớm hơn ngưỡng "hoảng loạn" thật
> Vì thêm Host/Storage mới thường mất thời gian đặt hàng/lắp đặt (không như cloud public bấm nút là có ngay), threshold cảnh báo nên đặt ở mức **70-85%** thay vì đợi tới 95%+ mới báo — cho đủ thời gian phản ứng trước khi chạm giới hạn thật.

## Log quan trọng cần biết đường dẫn

| Thành phần | Log path |
|---|---|
| Management Server | `/var/log/cloudstack/management/management-server.log` |
| Usage Server | `/var/log/cloudstack/usage/usage.log` |
| KVM Agent (trên host) | `/var/log/cloudstack/agent/agent.log` |
| Libvirt (trên host) | `/var/log/libvirt/libvirtd.log`, `/var/log/libvirt/qemu/<vm-name>.log` |
| System VM (bên trong VR/SSVM/CPVM) | `/var/log/cloud.log` (SSH vào system VM để xem) |
| MySQL/Galera | `/var/log/mysql/error.log` |

## Usage Server & Billing

```bash
systemctl status cloudstack-usage
tail -f /var/log/cloudstack/usage/usage.log

# Xem usage record qua API (dùng để tích hợp billing riêng)
cmk list usagerecords startdate=2026-09-01 enddate=2026-09-30
```

> [!warning] Lesson learned: Usage Server chạy riêng, chết âm thầm không ảnh hưởng VM nhưng ảnh hưởng billing
> `cloudstack-usage` là **service riêng biệt** với `cloudstack-management` — nó có thể ngừng chạy (crash, hết disk log...) trong khi toàn bộ hạ tầng VM vẫn hoạt động bình thường, không có alert rõ ràng nào báo ngay lập tức trên UI chính. Hậu quả chỉ lộ ra **cuối kỳ tính phí** khi phát hiện thiếu usage record cho cả 1 khoảng thời gian dài. Nên có health-check riêng (systemd watchdog hoặc external monitor) cho service này nếu billing quan trọng với tổ chức.

## Gợi ý dashboard cần có (nếu tự dựng Grafana)

- Capacity theo Zone/Pod/Cluster (CPU/RAM/Storage allocated vs used thật)
- Số lượng VM theo trạng thái (Running/Stopped/Error) theo thời gian
- Trạng thái Host (Up/Down/Alert) theo cluster
- Trạng thái System VM (SSVM/CPVM/VR)
- Async job queue length & failure rate — job kẹt hàng loạt thường là dấu hiệu sớm của sự cố MS/DB
- Galera cluster size & state
- Dung lượng Primary/Secondary Storage còn trống theo thời gian (trend, không chỉ số hiện tại)

---
*Xem thêm: [[CloudStack Day 2 Operations]] | [[CloudStack Troubleshooting]] | [[Database HA - MySQL Galera]] | [[Cloudstack|CloudStack]]*
