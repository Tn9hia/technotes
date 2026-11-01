---
type: concept
aliases: [Retention, Downsampling, Retention filters]
tags: [victoriametrics, retention, downsampling]
version: "1.154.1"
verified: 2026-11-02
parent: "[[VictoriaMetrics]]"
related: ["[[victoriametrics--architecture-cluster]]"]
---

# VictoriaMetrics — Retention & Downsampling

## What it does

> Retention giống hạn sử dụng trên hộp sữa — hết hạn là bỏ, không cần nghĩ nhiều. Downsampling giống việc nén ảnh cũ từ RAW xuống JPEG thấp hơn khi dọn ổ cứng — vẫn giữ được hình, chỉ bớt chi tiết, đổi lại nhẹ hơn nhiều để lưu lâu dài.

Retention là khoảng thời gian VictoriaMetrics giữ data trước khi tự xoá. Downsampling (Enterprise) là giảm độ phân giải data cũ (ví dụ gộp mỗi 5 phút thành 1 điểm) sau một mốc thời gian, để giảm dung lượng mà vẫn giữ được xu hướng dài hạn.

## Why it exists

Giữ mọi sample ở độ phân giải gốc (ví dụ mỗi 15s) trong nhiều năm tốn disk không cần thiết — hầu như không ai cần xem chi tiết từng giây của dữ liệu 2 năm trước, chỉ cần biết xu hướng chung. Downsampling giải quyết việc đó, retention giải quyết việc đơn giản hơn: data quá cũ không còn giá trị gì thì xoá hẳn.

## How it works

### Các lựa chọn — Retention

| Lựa chọn | Là gì | Khi nào dùng | Bẫy |
|---|---|---|---|
| `-retentionPeriod` (global, OSS) | 1 giá trị áp dụng cho toàn bộ data trong instance | Mặc định cho mọi use case đơn giản, không cần phân biệt theo metric/tenant | Default là `1` (tháng) — dễ bị hiểu nhầm là 1 ngày hoặc không giới hạn nếu không đọc kỹ |
| **Retention filter theo label** (Enterprise) | Áp retention khác nhau cho từng tập series theo label filter (MetricsQL) | Cần giữ metric quan trọng lâu hơn, xoá sớm metric ít giá trị (ví dụ debug metric) | Chỉ có ở Enterprise — OSS phải tách instance riêng nếu cần retention khác nhau theo nhóm metric |

### Các lựa chọn — Downsampling (Enterprise)

| Lựa chọn | Là gì | Khi nào dùng | Bẫy |
|---|---|---|---|
| `-downsampling.period=offset:interval` | Multi-level: data cũ hơn `offset` được gộp theo `interval` | Giữ data nhiều năm mà không tốn disk như giữ nguyên độ phân giải gốc | Chỉ có ở Enterprise; sau khi downsample, **không thể lấy lại độ phân giải gốc** cho khoảng thời gian đó |
| Nhiều mốc offset:interval cùng lúc (ví dụ `30d:5m`, `180d:1h`) | Downsampling theo nhiều cấp độ càng cũ càng thô | Cần cả chi tiết gần đây và xu hướng dài hạn tiết kiệm disk | Cấu hình nhiều mốc dễ nhầm thứ tự offset, cần kiểm tra kỹ bằng trang debug downsampling filter trong vmui (Enterprise) |

### Ai thắng khi xung đột

Nguyên tắc chung: **retention filter cụ thể hơn thắng `-retentionPeriod` global** (Enterprise); ở OSS không có khái niệm này nên không áp dụng.

| Bên A | Bên B | Kết quả | Im lặng hay báo lỗi? |
|---|---|---|---|
| `-retentionPeriod` global | Retention filter theo label match 1 tập series cụ thể (Enterprise) | Series match filter dùng retention riêng, phần còn lại dùng global | Im lặng — không cảnh báo override |
| Data đã bị downsample (Enterprise) | Query xin lại độ phân giải gốc cho khoảng thời gian đó | Không trả lại được — downsampling là **không đảo ngược** | Im lặng ở tầng query, chỉ thấy số điểm ít hơn kỳ vọng |

## Config gotchas

| Config | Default | Khuyến nghị | Vì sao |
|---|---|---|---|
| `-retentionPeriod` | `1` (tháng = 31 ngày) | Set rõ với đơn vị cụ thể (`-retentionPeriod=12`, hoặc cú pháp hỗ trợ năm/tháng tuỳ bản) | Default ngắn hơn nhiều người tưởng — dễ mất data điều tra incident cũ |
| Deduplication cuối tháng (final dedup pass) | Chạy tự động theo chu kỳ retention | Đảm bảo đủ disk trống **gấp đôi** tạm thời trong lúc pass này chạy | Pass dedup cuối cùng cần không gian tạm bằng dung lượng data tháng trước đó trước khi giải phóng — thiếu disk đúng lúc này dễ gây read-only mode ngoài ý muốn |

## Hay nhầm lẫn

- **Retention vs Downsampling** — retention là "xoá hẳn sau X", downsampling là "giữ lại nhưng giảm độ chi tiết" — downsampling không thay thế retention, vẫn cần set retention cho mốc cuối cùng muốn giữ.
- **Retention filter (Enterprise) vs nhiều instance VictoriaMetrics riêng** — ở OSS không có retention filter, cách thay thế phổ biến là chạy nhiều instance với `-retentionPeriod` khác nhau rồi route data bằng vmauth/relabel theo nhóm metric — phức tạp hơn nhưng khả thi không cần Enterprise.

## Ops notes

- Theo dõi `vm_data_size_bytes` theo thời gian để dự báo khi nào cần tăng retention/disk hoặc cân nhắc downsampling.
- Đổi `-retentionPeriod` không xoá ngay data cũ hơn — việc dọn diễn ra theo chu kỳ nội bộ (thường gắn với compaction/merge), không phải tức thời ngay sau khi đổi flag và restart.

## Refs

- [[VictoriaMetrics]] — note gốc, xem [[VictoriaMetrics#Bẫy vận hành]] cho case disk read-only liên quan retention/churn.
- [[victoriametrics--architecture-cluster]] — retention áp dụng ở vmstorage trong mô hình cluster.
- Official docs: <https://docs.victoriametrics.com/victoriametrics/single-server-victoriametrics/>
