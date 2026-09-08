# Falco — Performance Tuning & Syscall Event Drops
Tier: 2
Parent: [[Falco]]
Related: [[falco--drivers]], [[falco--kubernetes-deployment]], [[falco--outputs-falcosidekick]]
Tags: #falco #performance #ops #monitoring

## What it does

Đây là phần "sức khoẻ vận hành" của Falco: overhead CPU/memory do việc bắt syscall liên tục, kích thước ring buffer giữa driver và userspace, và **cơ chế drop event khi buffer đầy** — tức là Falco chủ động bỏ qua syscall thay vì block hệ thống, đổi lại là **mất khả năng nhìn thấy** trong khoảng thời gian đó.

## Why it exists

Bắt mọi syscall trên node có traffic cao (build server, batch job, network-heavy service) tạo ra lượng event khổng lồ. Nếu Falco cố xử lý 100% không drop, nó sẽ **cạnh tranh CPU/memory với chính workload đang chạy** — có thể làm chậm cả node. Falco chọn trade-off: dùng ring buffer có kích thước cố định, khi đầy thì **drop event mới thay vì block syscall của process** (ưu tiên không làm chậm workload hơn là không bỏ sót alert) — đây là quyết định thiết kế quan trọng cần hiểu để không hiểu nhầm hành vi khi troubleshoot.

## How it works (flow/diagram)

```
Syscall xảy ra liên tục
        │
        ▼
┌─────────────────────────┐
│ Ring buffer (per-CPU)     │  ← kích thước cố định (config được)
│ [event][event][event]...  │
└──────────┬───────────────┘
           │ Falco userspace đọc buffer theo tốc độ xử lý được
           ▼
   Buffer đầy? ──Có──▶ DROP event mới (n_drops tăng) — KHÔNG block syscall gốc
           │
          Không
           ▼
   Enrich + evaluate rule → alert (nếu match)
```

Điểm mấu chốt: **drop xảy ra TRƯỚC khi rule engine kịp nhìn thấy event** — nghĩa là nếu syscall attacker gây ra đúng lúc buffer đầy, **không rule nào có thể bắt được nó**, vì đơn giản là Falco chưa từng thấy event đó. Đây khác hẳn với "rule không match" (event có được thấy nhưng không match condition nào).

## Config gotchas

- **`syscall_event_drops` không tự động alert ra ngoài mặc định** ở nhiều setup — team dễ quên bật riêng cảnh báo cho metric này, dẫn tới tình huống "Falco chạy khoẻ, log vẫn ra, nhưng thực ra đang mù một phần" mà không ai biết cho tới khi audit lại sau incident.
- Kích thước ring buffer (`syscall_buf_size_preset` hoặc tương đương tùy version) đánh đổi trực tiếp giữa **memory usage** và **khả năng chịu burst syscall**. Tăng buffer giúp giảm drop khi traffic spike nhưng tăng memory footprint mỗi pod — trên node đông pod, nhân lên đáng kể.
- **Modern eBPF thường có CPU overhead khác kmod/legacy eBPF** tùy workload — không có driver nào "luôn nhanh hơn" tuyệt đối, cần benchmark theo tải thực tế của mày (đặc biệt với node có mật độ syscall cao như CI runner) thay vì tin theo benchmark mặc định trong docs.
- **DaemonSet resource limit quá thấp = OOMKilled = mất giám sát cả node**, không chỉ "chậm đi" như ứng dụng thường — vì Falco restart sẽ có khoảng gap không giám sát (driver phải load lại). Đặt limit dựa theo p99 sử dụng thực tế qua theo dõi 1-2 tuần, không dùng default Helm chart cho node có tải cao.
- Rule quá phức tạp (nhiều điều kiện lồng nhau, dùng nhiều field cần lookup) làm tăng thời gian evaluate mỗi event → gián tiếp làm buffer dễ đầy hơn dưới tải cao, dù bản thân driver/buffer không đổi. Tối ưu rule (đặt điều kiện rẻ trước, dùng macro thay vì lặp lại condition dài) cũng là 1 hình thức performance tuning.

## Security notes

- **Buffer đầy = blind spot có thể bị khai thác chủ động**: attacker biết Falco có drop mechanism có thể cố tình tạo syscall flood (fork bomb nhẹ, spawn nhiều process rapid) để làm đầy buffer, che giấu hành vi độc hại thật sự diễn ra ngay sau/trong lúc đó. Đây là kỹ thuật evasion thực tế, không chỉ lý thuyết — vì vậy alert cho `n_drops` tăng đột biến bản thân nó nên được coi là **tín hiệu đáng ngờ**, không chỉ là vấn đề performance.
- Vì lý do trên, nên treat "drop rate spike" như 1 loại alert bảo mật riêng (không chỉ đưa vào dashboard performance/SRE thông thường).

## Refs

- https://falco.org/docs/concepts/architecture/ (giải thích ring buffer/scap)
- https://github.com/falcosecurity/falco (source `n_drops` metric definitions)
- https://falco.org/docs/reference/rules/supported-fields/ (tối ưu condition — field nào rẻ để evaluate)
