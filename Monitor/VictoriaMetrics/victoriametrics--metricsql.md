---
type: concept
aliases: [MetricsQL]
tags: [victoriametrics, metricsql, query]
version: "1.154.1"
verified: 2026-11-02
parent: "[[VictoriaMetrics]]"
related: ["[[prometheus--promql-recording-rules]]"]
---

# VictoriaMetrics — MetricsQL

## What it does

> MetricsQL giống một phiên bản phương ngữ của PromQL — cùng ngữ pháp gốc, hiểu nhau được hầu hết câu, nhưng có vài từ lóng (hàm/cú pháp mở rộng) và vài chỗ phát âm khác (NaN, rate/increase) mà nếu không biết trước sẽ hiểu nhầm ý.

MetricsQL là ngôn ngữ query của VictoriaMetrics, thiết kế **tương thích ngược với PromQL** — mọi query PromQL hợp lệ chạy được trên MetricsQL, nhưng MetricsQL thêm cú pháp mở rộng riêng (WITH templates, `limit N` trên aggregation, lookbehind window tuỳ chọn) và có vài khác biệt hành vi có chủ đích.

## Why it exists

PromQL thiếu một số tiện ích khi viết query dài/phức tạp (không có khái niệm "biến tạm" như WITH template, cú pháp aggregation hạn chế). MetricsQL giải quyết việc đó bằng cách mở rộng cú pháp, đồng thời cũng chủ động sửa vài hành vi mà tác giả cho là gây nhầm lẫn ở PromQL (ví dụ trả về hàng loạt giá trị NaN) — đổi lại người từ Prometheus chuyển sang phải biết trước những khác biệt này.

## How it works

### Khác biệt hành vi quan trọng (không phải lỗi, là chủ đích)

| Khác biệt | PromQL | MetricsQL | Hậu quả nếu không biết |
|---|---|---|---|
| Giá trị NaN | Trả về series chứa giá trị NaN | Loại bỏ NaN khỏi kết quả — ví dụ `(-1)^0.5` trả **rỗng** thay vì series NaN | Dashboard/alert port từ Prometheus có thể "mất" series tưởng là bug, thực ra là đã bị lọc NaN |
| `rate()`/`increase()` | Có extrapolate ở biên range | Không extrapolate; `increase()` lấy luôn điểm cuối cùng của interval trước đó làm mốc | Số liệu `increase()`/`rate()` giữa Prometheus và VictoriaMetrics có thể lệch nhẹ ở vùng biên thời gian, nhất là range ngắn |
| Scalar type | Khác instant vector không label | Coi scalar giống instant vector không label | Query phân biệt kỹ 2 loại này theo chuẩn PromQL có thể cho kết quả khác đi (thường không ảnh hưởng use case thông thường) |
| Lookbehind window `[...]` | Bắt buộc phải có trong range vector selector | Có thể **bỏ qua**, MetricsQL tự chọn window hợp lý | Query thiếu `[...]` chạy được trên VictoriaMetrics nhưng sẽ lỗi syntax nếu đem thẳng sang Prometheus |

### Các lựa chọn — Mở rộng cú pháp riêng của MetricsQL

| Lựa chọn | Là gì | Khi nào dùng | Bẫy |
|---|---|---|---|
| `WITH` templates | Định nghĩa biểu thức tạm, tái dùng nhiều lần trong 1 query | Query dài, lặp lại cùng 1 biểu thức con nhiều lần | Chỉ chạy được trên MetricsQL — không copy-paste thẳng sang Prometheus được |
| `limit N` trên aggregation | Giới hạn số series trả về sau khi aggregate | Debug nhanh, tránh query trả quá nhiều series làm UI treo | Không phải filter theo giá trị — chỉ cắt số lượng, dễ nhầm là "top N theo giá trị" |
| Hàm mở rộng riêng (ví dụ các hàm rollup/transform không có ở PromQL) | Tập hàm bổ sung cho các nhu cầu phổ biến (ví dụ dự đoán, làm mượt dữ liệu) | Khi PromQL chuẩn không có hàm phù hợp | Dùng nhiều sẽ khoá chặt query vào riêng VictoriaMetrics, khó quay lại Prometheus sau này |

## Config gotchas

Không áp dụng — MetricsQL là ngôn ngữ query, không có config riêng ngoài cách viết query; các khác biệt hành vi đã liệt kê ở bảng trên chính là nơi cần cẩn trọng khi viết query, không phải ở config server.

## Hay nhầm lẫn

- **"Tương thích PromQL" không có nghĩa là "giống 100%"** — đa số query chạy đúng như kỳ vọng, nhưng vùng biên (NaN, rate/increase ở range ngắn) có thể cho số khác. Khi port dashboard/alert từ Prometheus sang VictoriaMetrics, nên test lại các alert nhạy với giá trị biên.
- **MetricsQL extension vs PromQL chuẩn** — query dùng `WITH`/`limit N` chạy tốt trên VictoriaMetrics nhưng sẽ báo lỗi syntax nếu dán thẳng vào Prometheus — cần biết query nào đã "khoá" vào MetricsQL.

## Ops notes

- VictoriaMetrics có trang **vmui** (UI built-in) hỗ trợ viết/test MetricsQL trực tiếp, kèm gợi ý cú pháp — nơi tốt để kiểm tra 1 query trước khi đưa vào dashboard/alert rule.
- Khi debug "dashboard ra số khác so với lúc còn dùng Prometheus", ưu tiên nghi ngờ các hàm `rate()`/`increase()` ở range ngắn hoặc query có khả năng ra NaN trước khi nghi ngờ data bị thiếu.

## Refs

- [[VictoriaMetrics]] — note gốc.
- [[prometheus--promql-recording-rules]] — PromQL gốc mà MetricsQL tương thích ngược, dùng để đối chiếu khác biệt.
- Official docs: <https://docs.victoriametrics.com/metricsql/>
