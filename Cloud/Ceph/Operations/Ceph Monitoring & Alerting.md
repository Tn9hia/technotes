---
tags:
  - ceph
  - operations
  - monitoring
---

# Ceph Monitoring & Alerting

**Giám sát chủ động** là yếu tố sống còn với Ceph vì cluster có khả năng "tự chữa lành" (self-healing) rất mạnh — điều này vừa là ưu điểm vừa là cái bẫy: nhiều vấn đề nhỏ bị che giấu bởi khả năng tự phục hồi cho tới khi chúng cộng dồn thành sự cố thật. Ceph có sẵn hạ tầng export metrics khá tốt (built-in Prometheus exporter, Dashboard) — không cần công cụ bên thứ ba để lấy dữ liệu, chỉ cần khai thác đúng.

> [!tip] So với VMware vSAN
> vSAN Health Service trong vCenter là 1 bảng tổng hợp check theo nhóm (network, cluster, capacity...) khá "đóng hộp". Ceph cởi mở hơn nhưng cũng đòi hỏi tự lắp ráp nhiều hơn: `ceph health` cho bạn trạng thái tổng quan tương tự, nhưng để có dashboard/alerting nghiêm túc bạn cần bật `prometheus` mgr module rồi tự nối vào Grafana + Alertmanager — không có gì "chạy sẵn out of the box" ở mức production-ready.

## Cluster Health State — 3 mức, hiểu đúng ý nghĩa

| Trạng thái | Ý nghĩa | Hành động |
|---|---|---|
| `HEALTH_OK` | Không có cảnh báo nào đang active | Không cần làm gì, vẫn nên xem trend |
| `HEALTH_WARN` | Có vấn đề — **chưa** gây mất dữ liệu/downtime ngay, nhưng có thể tích lũy thành nghiêm trọng | Điều tra trong ngày làm việc, không được bỏ qua nhiều ngày liền |
| `HEALTH_ERR` | Vấn đề nghiêm trọng — nguy cơ mất dữ liệu hoặc gián đoạn dịch vụ thật | Xử lý ngay, ưu tiên cao nhất |

```bash
ceph health detail          # luôn xem detail, đừng chỉ nhìn 1 dòng tổng
ceph -w                     # theo dõi state transition realtime khi đang xử lý sự cố
```

## Nguồn Metrics chính thức

| Nguồn | Bật bằng | Cổng mặc định | Dùng để |
|---|---|---|---|
| **`prometheus` mgr module** | `ceph mgr module enable prometheus` | `9283` | Metrics cấp cluster/pool/PG — nguồn chính cho Grafana |
| **`ceph-exporter`** | Tự động deploy qua cephadm (`ceph orch apply ceph-exporter`) | `9926` (mỗi host) | Metrics cấp daemon/host chi tiết hơn (per-OSD perf counters), giảm tải cho mgr |
| **Ceph Dashboard** | `ceph mgr module enable dashboard` | `8443` (HTTPS) | Giao diện web trực quan — health check nhanh bằng mắt, không thay thế alerting |
| **`ceph crash` module** | Bật mặc định | — | Theo dõi daemon crash report |
| **Node exporter** (không phải Ceph riêng) | Cài trên từng host | `9100` | CPU/RAM/disk/network thật của host — bổ sung, không phải Ceph-specific |

```bash
# Bật Prometheus exporter (nguồn metrics chuẩn)
ceph mgr module enable prometheus
curl http://<mgr-host>:9283/metrics | head -30   # kiểm tra nhanh có data không

# Bật ceph-exporter per-daemon (khuyến nghị từ Quincy/Reef trở đi)
ceph orch apply ceph-exporter

# Bật Dashboard
ceph mgr module enable dashboard
ceph dashboard create-self-signed-cert
ceph dashboard set-login-credentials <user> -i <(echo "<password>")
ceph mgr services                        # xem URL dashboard đang chạy ở đâu
```

> [!tip] Dashboard vs Prometheus — dùng cho mục đích khác nhau
> Dashboard tốt cho việc **nhìn nhanh** khi đang ngồi debug trực tiếp (topology, PG state, log gần nhất) — giống việc mở vCenter UI nhìn health service. Nhưng nó **không lưu lịch sử dài hạn** và không phải công cụ alerting. Prometheus + Grafana mới là nơi lưu trend theo thời gian và bắn alert qua Alertmanager — 2 công cụ bổ sung nhau, không thay thế nhau.

## Grafana & Alertmanager

Ceph upstream cung cấp sẵn bộ **Grafana dashboard chính thức** (JSON export theo từng phiên bản Ceph) — cephadm có thể tự deploy cả stack monitoring (Prometheus + Grafana + Alertmanager + node-exporter) cùng lúc với cluster:

```bash
ceph orch apply prometheus
ceph orch apply grafana
ceph orch apply alertmanager
ceph orch apply node-exporter

ceph orch ls --service_type grafana        # kiểm tra service đã deploy
ceph dashboard get-grafana-api-url         # lấy URL Grafana đã tích hợp sẵn vào Dashboard
```

Dashboard chính thức đáng dùng ngay: **Ceph Cluster**, **OSD device details**, **Pool overview**, **RBD overview**, **RGW overview** — import từ [grafana.com](https://grafana.com) theo ID chính thức Ceph publish, hoặc lấy trực tiếp từ package `ceph-grafana-dashboards`.

## Metrics/Alert quan trọng cần cấu hình

| Metric | Ngưỡng gợi ý | Vì sao quan trọng |
|---|---|---|
| Capacity `nearfull` | 85% (mặc định `mon_osd_nearfull_ratio`) | Cảnh báo sớm trước khi chạm `full` |
| Capacity `full` | 95% (mặc định `mon_osd_full_ratio`) | Cluster **ngừng nhận write** khi chạm — cực kỳ khẩn cấp |
| Số OSD `down` | > 0 kéo dài quá vài phút | OSD down ngắn hạn (reboot) bình thường; kéo dài là bất thường |
| PG `not active` | > 0 | PG không active nghĩa là I/O bị chặn cho dữ liệu thuộc PG đó — nghiêm trọng |
| PG `not clean` kéo dài | > vài giờ không giảm | Recovery/backfill bị kẹt hoặc quá chậm, cần điều tra |
| MON quorum size | < số MON cấu hình | Mất 1 MON vẫn còn quorum (với 3 hoặc 5 MON), nhưng cần biết ngay để xử lý trước khi mất thêm |
| Clock skew giữa MON | > vài trăm ms | MON dùng thời gian để đồng thuận (Paxos) — skew lớn có thể khiến quorum không ổn định |
| Daemon down (`ceph orch ps`) | Bất kỳ daemon nào không `running` | Phát hiện sớm trước khi ảnh hưởng dịch vụ |
| `ceph crash` pending | > 0 | Daemon đã crash ít nhất 1 lần — cần biết nguyên nhân dù đã tự restart |

```bash
# Kiểm tra thủ công các ngưỡng full
ceph osd dump | grep -E "full_ratio|nearfull_ratio"
ceph df detail

# Clock skew — nên có alert riêng, không chỉ dựa vào HEALTH_WARN chung
ceph time-sync-status
```

> [!warning] Clock skew — cảnh báo nhỏ nhưng có thể gây hậu quả lớn
> MON dùng thuật toán Paxos để đồng thuận, và có nhạy cảm nhất định với lệch giờ giữa các node MON. Skew nhỏ (vài chục ms) thường vô hại, nhưng skew lớn kéo dài (do NTP/chrony không chạy, hoặc VM MON bị "time drift" trên hypervisor) có thể gây **mất ổn định quorum** khó chẩn đoán — triệu chứng bên ngoài trông giống lỗi network hoặc lỗi MON ngẫu nhiên, rất dễ đi sai hướng debug. Luôn đảm bảo **chrony hoặc NTP chạy và đồng bộ đúng** trên toàn bộ MON host, và có alert riêng cho `ceph time-sync-status` thay vì chỉ dựa vào `HEALTH_WARN` chung chung (đôi khi bị lẫn giữa nhiều cảnh báo khác).

## `ceph crash` — Theo dõi Daemon Crash

```bash
ceph crash ls                      # danh sách crash chưa xử lý
ceph crash ls-new                  # chỉ crash mới kể từ lần xem gần nhất
ceph crash info <crash-id>         # chi tiết stack trace
ceph crash archive <crash-id>      # đánh dấu đã xem
ceph crash archive-all             # archive toàn bộ
```

Nên tích hợp `ceph crash` vào pipeline alert (VD: check `ceph crash ls-new` định kỳ, bắn alert nếu có bản ghi mới) — một daemon crash rồi tự restart thành công **vẫn là tín hiệu quan trọng**, không nên chỉ dựa vào `ceph orch ps` (nó chỉ cho biết trạng thái hiện tại, không cho biết lịch sử crash).

## Bảng tổng hợp dashboard nên có (Grafana)

- Cluster health state theo thời gian (OK/WARN/ERR timeline, không chỉ điểm hiện tại)
- Capacity trend theo pool (used/available, tốc độ tăng trưởng)
- OSD latency (`commit_latency`, `apply_latency`) theo từng OSD — phát hiện disk sắp hỏng
- PG state breakdown (active+clean vs degraded/recovering/backfilling)
- Client I/O throughput & IOPS (theo pool, theo RBD image nếu cần)
- MON quorum size & clock skew theo thời gian
- Số lượng OSD up/in theo thời gian (phát hiện flapping)
- RGW request rate & latency (nếu dùng object storage)

> [!warning] Lesson learned: chỉ alert HEALTH_ERR, bỏ qua HEALTH_WARN kéo dài
> Một cấu hình alerting phổ biến nhưng thiếu an toàn: chỉ bắn page/SMS khi `ceph health == HEALTH_ERR`, coi `HEALTH_WARN` là "bình thường, không cần làm phiền ai". Vấn đề là nhiều sự cố nghiêm trọng **bắt đầu** bằng WARN kéo dài âm thầm — ví dụ: vài PG "not deep-scrubbed in time" tích lũy suốt nhiều tuần vì lịch scrub bị nghẽn do tải cao, rồi đúng lúc đó 1 disk lỗi thật gây corrupt dữ liệu mà lẽ ra deep-scrub định kỳ đã phát hiện sớm hơn nhiều; hoặc clock skew nhỏ kéo dài hàng tháng không ai để ý, tới khi 1 MON restart giữa lúc bảo trì thì cluster mất quorum vì 2 MON còn lại lệch giờ quá ngưỡng. Chiến lược alerting đúng: coi **WARN tồn tại quá X giờ** (không phải WARN xuất hiện tức thời — cái đó có thể tự hết) là điều kiện actionable ngang hàng với ERR, không phải noise để bỏ qua.

---
*Xem thêm: [[Ceph Day 2 Operations]] | [[Ceph Troubleshooting]] | [[Ceph CLI Cheatsheet]] | [[Ceph|Ceph]]*
