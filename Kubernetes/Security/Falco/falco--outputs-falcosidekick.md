# Falco — Outputs & Falcosidekick
Tier: 2
Parent: [[Falco]]
Related: [[falco--rules-syntax]], [[falco--performance-tuning-drops]]
Tags: #falco #alerting #falcosidekick #siem

## What it does

**Output channel** là nơi Falco gửi alert đã format tới sau khi rule match: stdout, file, syslog, gRPC API (structured, dùng cho integration), hoặc HTTP webhook. **Falcosidekick** là 1 service riêng (không phải built-in Falco) nhận alert từ Falco (qua HTTP hoặc gRPC) rồi **route/fan-out** tới hàng chục backend khác nhau (Slack, Elasticsearch, S3, Loki, PagerDuty, SIEM...) mà không cần Falco tự biết cách nói chuyện với từng backend.

## Why it exists

Nếu để Falco tự tích hợp trực tiếp với từng backend (Slack API, ES API, PagerDuty API...) thì core Falco sẽ phình to và phải maintain rất nhiều connector. Falcosidekick tách trách nhiệm: **Falco chỉ cần biết gửi tới 1 endpoint chung (chính nó)**, còn việc route đi đâu, transform format ra sao là việc của Falcosidekick — giống pattern message broker/fan-out.

## How it works (flow/diagram)

```
Falco (outputs.http/grpc enabled)
        │  POST alert (JSON)
        ▼
┌─────────────────────┐
│  Falcosidekick        │
│  - nhận alert          │
│  - filter theo priority/rule (tùy config)
│  - fan-out tới N output│
└──────────┬───────────┘
           ├──▶ Slack/Teams (chat notification)
           ├──▶ Elasticsearch/Loki (long-term storage, search)
           ├──▶ S3/GCS (archive, compliance retention)
           ├──▶ PagerDuty/Opsgenie (on-call paging)
           └──▶ Prometheus (metrics: falcosidekick_outputs_total...)
```

`falcosidekick-ui` (tùy chọn thêm) cung cấp dashboard xem alert trực tiếp không cần cắm SIEM ngay từ đầu — hữu ích cho POC/giai đoạn đầu vận hành.

## Config gotchas

- **`outputs.rate` / `outputs.max_burst` trong `falco.yaml` rate-limit alert tại chính Falco**, trước khi tới Falcosidekick — nếu bị tấn công dồn dập (hoặc rule tune sai gây flood), Falco có thể **âm thầm drop alert** ở tầng này. Log sẽ có dòng cảnh báo rate limiting nhưng dễ bị bỏ qua giữa hàng loạt log khác.
- gRPC output **mặc định có thể chạy plaintext** ở một số cấu hình cũ nếu không set rõ TLS — kiểm tra kỹ trước khi expose ngoài `localhost`/pod network nội bộ.
- Falcosidekick có **queue/buffer nội bộ giới hạn** cho từng output đích — nếu 1 backend (vd Slack) rate-limit hoặc down, alert cho backend đó có thể bị drop trong khi backend khác vẫn nhận bình thường; cần theo dõi metric `falcosidekick_outputs_total{status="error"}` theo từng output riêng, không chỉ nhìn tổng.
- Filter theo priority nên đặt ở Falcosidekick (hoặc SIEM) chứ không tắt hẳn ở Falco — tắt ở Falco nghĩa là **mất hẳn dữ liệu gốc**, filter ở downstream vẫn giữ được raw event để audit sau này nếu cần.
- Nhiều output cùng lúc (`stdout` + `http` + `grpc`) làm tăng CPU/network overhead đáng kể khi volume alert cao — chỉ bật kênh thực sự cần dùng.

## Security notes

- Alert payload thường chứa **command line đầy đủ** (`proc.cmdline`), có thể vô tình chứa secret nếu process được start kèm token/password trên command line (anti-pattern nhưng vẫn phổ biến) → mọi kênh output (đặc biệt Slack/chat công khai trong công ty) cần cân nhắc mức độ nhạy cảm trước khi route.
- Endpoint HTTP/gRPC của Falco và của Falcosidekick đều nên giới hạn bằng NetworkPolicy — chỉ cho phép traffic giữa Falco pod ↔ Falcosidekick pod, không mở ra ngoài.
- Nếu Falcosidekick forward tới S3/external storage cho mục đích compliance retention, cần đảm bảo bucket có encryption + access policy đúng chuẩn (đây là nơi lưu trữ bằng chứng audit, chính nó cũng là target).

## Refs

- https://falco.org/docs/concepts/outputs/
- https://github.com/falcosecurity/falcosidekick
- https://github.com/falcosecurity/falcosidekick-ui
