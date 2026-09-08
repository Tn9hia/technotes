---
tags:
  - ceph
  - recovery
  - backfill
  - self-healing
---

# Recovery, Backfill & Self-healing

Ceph được thiết kế để **tự phục hồi** khi OSD hoặc cả host chết — không cần operator can thiệp thủ công để dữ liệu trở lại đủ số bản sao. Hiểu đúng vòng đời peering → recovery/backfill, và biết cách throttle nó, là kỹ năng vận hành cốt lõi — vì self-healing mặc định **cạnh tranh trực tiếp** với I/O của VM đang chạy.

> [!tip] So với VMware vSAN
> Tinh thần giống hệt cơ chế "resync" của vSAN khi 1 disk/host bị đưa vào maintenance hoặc chết — vSAN cũng tự động resync component để đạt lại FTT mong muốn, và cũng có tham số throttle (`Resync throttling`). Khác biệt là Ceph tách rõ 2 khái niệm **recovery** và **backfill** với cơ chế và mức độ tốn tài nguyên khác nhau — vSAN không phân biệt rạch ròi 2 khái niệm này ra bên ngoài UI.

## Peering → Recovery vs Backfill

Khi 1 OSD down (hoặc PG bị coi là thiếu bản sao), PG liên quan trải qua các giai đoạn:

```mermaid
graph LR
    A[OSD down] --> B[Peering<br/>các OSD còn lại thống nhất<br/>trạng thái mới nhất của PG]
    B --> C{PG log của replica<br/>còn đủ "gần" primary?}
    C -->|Có, chỉ thiếu vài write gần đây| D[Recovery<br/>replay từ PG log]
    C -->|Không — quá cũ hoặc OSD mới toanh| E[Backfill<br/>copy toàn bộ object]
```

| | Recovery | Backfill |
|---|---|---|
| Khi nào xảy ra | Replica chỉ thiếu 1 số write gần đây (còn nằm trong PG log) | Replica quá cũ / OSD hoàn toàn mới (VD: thêm OSD mới, hoặc down quá lâu vượt `osd_pg_log_dups_tracked`) |
| Cơ chế | Replay các write bị thiếu từ **PG log** | Copy **toàn bộ object** của PG sang OSD đích |
| Chi phí I/O | Nhẹ — chỉ phần chênh lệch | Nặng — bằng cả dung lượng PG đó |
| Thời gian | Nhanh | Chậm hơn nhiều, tỷ lệ thuận với dung lượng dữ liệu |

Đây là lý do vì sao **down 1 OSD trong vài phút rồi lên lại** (VD: restart service, reboot ngắn) thường chỉ trigger recovery nhẹ nhàng, còn **thay hẳn 1 OSD mới** hoặc để OSD down quá lâu sẽ luôn trigger backfill nặng — quyết định maintenance window nên cân nhắc chính khác biệt này.

## Throttling — cân bằng tốc độ hồi phục vs I/O của client

Recovery/backfill mặc định chạy song song với I/O của VM đang chạy — nếu không throttle, chúng có thể chiếm phần lớn băng thông OSD và làm chậm hẳn ứng dụng production trong giờ cao điểm.

```bash
# Số backfill đồng thời tối đa mỗi OSD (mặc định 1 với HDD, có thể cao hơn với SSD/NVMe)
ceph config set osd osd_max_backfills 1

# Số thao tác recovery đồng thời tối đa mỗi OSD
ceph config set osd osd_recovery_max_active 3

# Ưu tiên CPU/queue của recovery op so với client op (giá trị thấp = ưu tiên thấp hơn client)
ceph config set osd osd_recovery_op_priority 3

# Giới hạn băng thông recovery (bytes/sec mỗi OSD) — hữu ích trên cluster HDD chậm
ceph config set osd osd_recovery_max_single_start 1
```

| Tham số | Mặc định (tham khảo) | Tác dụng |
|---|---|---|
| `osd_max_backfills` | 1 | Số PG backfill đồng thời/OSD — tăng để phục hồi nhanh hơn, giảm để bớt ảnh hưởng client |
| `osd_recovery_max_active` | 3 | Số thao tác recovery đồng thời/OSD |
| `osd_recovery_op_priority` | 3 (thấp hơn client op priority) | Mức ưu tiên queue so với client I/O |
| `osd_recovery_sleep` | 0 (HDD thường set > 0) | Thời gian nghỉ giữa các recovery op — thêm độ trễ cố ý để nhường băng thông |

> [!tip] Tăng tốc phục hồi ngoài giờ, giảm lại trong giờ cao điểm
> Một pattern vận hành phổ biến: tăng `osd_max_backfills` và `osd_recovery_max_active` tạm thời (VD: qua đêm hoặc cuối tuần) khi cần phục hồi nhanh sau sự cố lớn (thêm nhiều OSD, thay host), rồi hạ lại về mức bảo thủ khi vào giờ làm việc để tránh ảnh hưởng VM production. Đổi bằng `ceph config set` áp dụng live, không cần restart OSD.

## Theo dõi tiến trình

```bash
# Theo dõi real-time trạng thái cluster, bao gồm % recovery/backfill
ceph -w

# Xem chi tiết trạng thái từng PG (degraded, backfilling, recovering...)
ceph pg dump | grep -E 'backfilling|recovering'

# Tóm tắt nhanh
ceph status
ceph health detail
```

## Scrubbing & Deep-scrubbing — kiểm tra tính toàn vẹn dữ liệu

Ngoài recovery/backfill (phục hồi khi *thiếu* bản sao), Ceph còn có cơ chế **scrub** để phát hiện dữ liệu bị hỏng âm thầm (bit-rot, silent corruption) ngay cả khi đủ số bản sao.

| Loại | Kiểm tra gì | Tần suất mặc định |
|---|---|---|
| **Scrub (light)** | So sánh metadata/checksum object giữa các bản sao | Hàng ngày (`osd_scrub_min_interval`/`osd_scrub_max_interval`) |
| **Deep-scrub** | Đọc và so sánh **toàn bộ nội dung byte** của object giữa các bản sao | Hàng tuần (`osd_deep_scrub_interval`, mặc định 7 ngày) |

```bash
# Xem lịch scrub cấu hình hiện tại
ceph config get osd osd_scrub_begin_hour
ceph config get osd osd_scrub_end_hour
ceph config get osd osd_deep_scrub_interval

# Giới hạn scrub chỉ chạy trong khung giờ thấp điểm (VD: 1h-6h sáng)
ceph config set osd osd_scrub_begin_hour 1
ceph config set osd osd_scrub_end_hour 6

# Scrub thủ công 1 PG cụ thể (hữu ích khi debug)
ceph pg scrub 1.1a
ceph pg deep-scrub 1.1a

# Khi scrub phát hiện lỗi — cluster báo HEALTH_ERR với inconsistent PG
ceph health detail
# HEALTH_ERR 1 scrub errors; Possible data damage: 1 pg inconsistent

# Sửa (dùng bản sao "đúng" theo đa số/checksum để ghi đè bản lỗi)
ceph pg repair 1.1a
```

`HEALTH_ERR` do scrub error nghĩa là Ceph đã **thực sự tìm thấy** sự khác biệt dữ liệu giữa các bản sao (bit-rot, lỗi disk, corruption) — đây không phải cảnh báo phòng hờ, mà là bằng chứng cụ thể cần xử lý. `ceph pg repair` cố gắng tự sửa dựa trên đa số bản sao khớp nhau (hoặc checksum với BlueStore), nhưng nếu tất cả bản sao đều đã hỏng theo cùng cách hiếm gặp, cần khôi phục từ backup.

> [!warning] Lesson learned: tắt scrub/deep-scrub vĩnh viễn "để tiết kiệm IOPS" — phát hiện corruption khi đã quá muộn
> Deep-scrub tốn I/O thật (đọc toàn bộ dữ liệu định kỳ), và trên cluster đã bão hòa IOPS, một số operator chọn giải pháp "tắt hẳn" (`ceph osd set noscrub`, `ceph osd set nodeep-scrub`) thay vì chỉ giới hạn khung giờ — rồi **quên bật lại**. Hậu quả: Ceph mất đi cơ chế phát hiện bit-rot duy nhất của nó. Dữ liệu có thể âm thầm hỏng ở 1 bản sao (lỗi disk, firmware, bit flip) mà không ai biết, vì I/O đọc bình thường của ứng dụng chỉ đọc từ 1 bản sao (thường là primary) chứ không tự so sánh checksum giữa các bản. Corruption tích lũy nhiều tháng, và khi cuối cùng phát hiện ra (thường là lúc 1 OSD khác cũng chết, buộc phải dùng đúng bản sao đã hỏng để recovery), có thể đã **không còn bản sao nào sạch** để phục hồi từ. Đúng cách xử lý áp lực IOPS là **giới hạn khung giờ scrub** (`osd_scrub_begin_hour`/`end_hour`) hoặc giảm interval một cách có kiểm soát, không phải tắt hẳn vô thời hạn — và luôn định kỳ kiểm tra `ceph osd dump | grep -E 'noscrub|nodeep-scrub'` xem có flag nào đang bị set quên tắt hay không.

---
*Xem thêm: [[Pools, Replication & Erasure Coding]] | [[Placement Groups (PG)]] | [[Ceph Troubleshooting]] | [[Ceph|Ceph]]*
