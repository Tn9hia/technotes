---
tags:
  - ceph
  - operations
  - configuration
---

# Key Configuration Reference (Ceph)

Tổng hợp các file cấu hình và config option **hay phải đụng tới nhất** khi vận hành — dùng để tra cứu nhanh, không cần nhớ hết chi tiết từng note khác.

## File & vị trí dữ liệu quan trọng

| File / Thư mục | Nằm ở đâu | Chứa gì |
|---|---|---|
| `/var/lib/ceph/<fsid>/` | Mọi node trong cluster | Data dir gốc của cephadm — mỗi daemon 1 thư mục con |
| `/var/lib/ceph/<fsid>/<daemon>.<id>/` | Từng node chạy daemon đó | Data thực tế của daemon (VD: `mon.host1/`, `osd.3/`) |
| `/etc/ceph/ceph.conf` | Mọi node có quyền admin/client | **Rất tối giản** — chủ yếu chỉ `fsid` + danh sách `mon host`, vì phần lớn config giờ nằm tập trung trong MON config store |
| `/etc/ceph/ceph.client.admin.keyring` | Node admin | Keyring quyền admin toàn cluster — nhạy cảm, không được để lộ |
| `/etc/ceph/ceph.pub` | Node admin (bootstrap) | SSH public key cephadm dùng để quản lý các node khác trong cluster |
| `/var/log/ceph/<fsid>/` | Mọi node | Log từng daemon (khi dùng cephadm, log cũng đẩy qua journald container) |

> [!tip] So với VMware vSAN
> vSAN gần như không có "file cấu hình" cho admin đụng tới trực tiếp — mọi thứ qua vCenter object model. Ceph ngược lại: `ceph.conf` gần như trống rỗng vì cephadm/Squid trở đi đã chuyển hẳn sang **centralized config store trong MON** (`ceph config set/dump`) — tương tự tinh thần "advanced settings tập trung" nhưng ở đây là bắt buộc, không phải tùy chọn.

```ini
; Ví dụ ceph.conf tối giản điển hình với cephadm (Squid/Tentacle)
[global]
fsid = 6f3a2e1c-9b4d-4a2e-8f1a-2d5c7e9b0a11
mon_host = [v2:10.10.10.11:3300,v1:10.10.10.11:6789] [v2:10.10.10.12:3300,v1:10.10.10.12:6789]
```

## Config option hay dùng nhất (theo nhóm)

### Ngưỡng capacity & health

```ini
mon_osd_nearfull_ratio       # mặc định 0.85 — cảnh báo NEARFULL
mon_osd_full_ratio           # mặc định 0.95 — OSD full, cluster từ chối write mới
mon_osd_backfillfull_ratio   # mặc định 0.90 — OSD không nhận thêm backfill data
```

### Recovery & backfill throttle

```ini
osd_max_backfills            # mặc định 1 — số backfill operation đồng thời mỗi OSD
osd_recovery_max_active      # số recovery operation đồng thời mỗi OSD
osd_recovery_op_priority     # priority I/O của recovery so với client I/O (giá trị thấp hơn = ít ưu tiên hơn client)
```

Chi tiết cơ chế và cách throttle theo khung giờ xem [[Recovery, Backfill & Self-healing]] và [[Ceph Performance Tuning]].

### Placement Group (PG)

```ini
osd_pool_default_pg_autoscale_mode   # on/warn/off — mặc định "on" từ các release gần đây
mon_target_pg_per_osd                # số PG mục tiêu trung bình mỗi OSD, autoscaler dùng làm tham chiếu
```

### Lịch scrub

```ini
osd_scrub_begin_hour         # giờ bắt đầu cho phép scrub (VD: 22 = 10PM)
osd_scrub_end_hour           # giờ kết thúc cửa sổ scrub (VD: 6 = 6AM)
osd_scrub_load_threshold     # load average tối đa cho phép scrub chạy — vượt ngưỡng thì hoãn
```

### cephx (auth)

```ini
auth_cluster_required = cephx   # bắt buộc — daemon-to-daemon
auth_service_required = cephx   # bắt buộc — client-to-service
auth_client_required = cephx    # bắt buộc — client-to-cluster
```

> [!warning] Cả 3 giá trị này KHÔNG bao giờ nên là `none` trên production
> Đặt `none` tắt hẳn cephx authentication — bất kỳ ai truy cập được vào network cluster đều có thể thao tác trực tiếp với RADOS mà không cần key nào. Chi tiết rủi ro và hardening xem [[Ceph Security Considerations]].

### Client (RBD)

```ini
rbd_cache                    # bật/tắt cache phía client — mặc định true
client_mount_timeout         # timeout khi client cố mount/kết nối cluster
```

## Port reference

| Thành phần | Port | Giao thức/Ghi chú |
|---|---|---|
| MON (messenger v2) | 3300 | Giao thức mặc định từ Nautilus trở đi, hỗ trợ msgr2 encryption |
| MON (messenger v1, legacy) | 6789 | Giữ để tương thích ngược, nên dần loại bỏ |
| MGR Dashboard | 8443 | HTTPS UI quản trị |
| Prometheus exporter (MGR module) | 9283 | Scrape metrics cluster-level |
| Node exporter | 9926 (hoặc 9100 tùy bản deploy) | Metrics node-level |
| OSD | 6800-7300 (range) | Mỗi OSD daemon chiếm vài port trong dải này cho public + cluster network |
| RGW | 80 / 443 | S3/Swift API endpoint |
| MDS | 6800+ (chung dải với OSD) | CephFS metadata service |

> [!warning] Đây cũng là checklist rà soát firewall — xem thêm [[Ceph Security Considerations]]
> Danh sách port trên là bề mặt cần đối chiếu khi hardening: port nào không cần mở ra ngoài phạm vi tối thiểu (VD: Dashboard 8443, RGW 80/443 nếu không cần public) thì nên chặn ở tầng network.

## Lệnh tra cứu nhanh

```bash
# Xem toàn bộ config hiện tại (đã áp dụng, khác default)
ceph config dump

# Xem config đang áp dụng cho 1 daemon cụ thể
ceph config show osd.0

# Đặt 1 config option ở scope global (áp dụng toàn cluster)
ceph config set global <key> <value>

# Đặt config chỉ cho 1 loại daemon hoặc 1 daemon cụ thể
ceph config set osd osd_max_backfills 3
ceph config set osd.5 osd_max_backfills 1

# Xem mô tả, kiểu dữ liệu, giá trị mặc định của 1 option
ceph config help osd_max_backfills
```

> [!tip] Khi nhận bàn giao, export toàn bộ config hiện tại ra file ngay ngày đầu
> `ceph config dump > ceph-config-baseline.txt` nên được chạy và lưu lại **ngay ngày đầu tiên** nhận bàn giao cluster. Đây là "dấu vân tay cấu hình" giúp bạn biết chính xác cái gì đã bị tùy chỉnh khác default (và ai đó có lý do gì khi làm vậy) trước khi bạn bắt đầu thay đổi bất cứ thứ gì.

---
*Xem thêm: [[Ceph CLI Cheatsheet]] | [[Ceph Security Considerations]] | [[Lessons Learned & Common Pitfalls (Ceph)]] | [[Ceph|Ceph]]*
