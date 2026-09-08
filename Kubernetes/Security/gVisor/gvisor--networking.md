# gVisor — Networking (netstack, host passthrough, traffic shaping)
Tier: 2
Parent: [[gvisor]]
Related: [[gvisor--security-model]], [[gvisor--common-errors-lessons]]
Tags: #gvisor #networking #performance

## What it does

gVisor tự viết network stack riêng gọi là **netstack** (TCP/IP đầy đủ) chạy hoàn toàn bên trong Sentry, tách biệt với network stack của host.

## Why it exists

Nếu để app trong sandbox dùng thẳng socket API của host kernel, thì toàn bộ bug trong host network stack (1 trong những phần code lớn/phức tạp nhất của kernel) sẽ lộ ra cho attacker. Netstack cô lập hoàn toàn: Sentry chỉ có **1 `AF_PACKET` socket raw** duy nhất để gửi/nhận packet ở tầng link layer — không tự tạo socket kernel nào khác (trừ khi bật host networking).

## How it works (flow/diagram)

```
App --socket syscall--> netstack (TCP/UDP/IP logic, 100% trong Sentry)
                              │
                        AF_PACKET socket (raw, tầng link)
                              │
                    veth/loopback device (do Docker/K8s network namespace tạo)
                              │
                          host network
```

- Sentry được start trong 1 network namespace có sẵn veth + loopback (giống Docker thường). gVisor **scrape địa chỉ/route** từ các device đó và tự cấu hình netstack dùng y hệt IP/route đó — app "thấy" mạng y như chạy trực tiếp trên host, nhưng traffic thực chất đi qua netstack trước.
- **Threading**: link endpoint (thường là `fdbased`) có goroutine riêng nhận packet. TCP xử lý bất đồng bộ qua goroutine riêng của chính nó. Packet ra ngoài qua qdisc (mặc định FIFO, có thể chọn TBF).
- Netstack hỗ trợ nhiều link layer: `AF_PACKET`, `AF_XDP`, shared memory, Go channel — và **tự nó là 1 dự án dùng lại được độc lập** ở project khác (API không đảm bảo semver ổn định).

### Network passthrough (`--network=host`)

Bỏ hoàn toàn netstack, dùng thẳng network stack của host (package `hostinet`) — **đổi security/isolation lấy performance**. Chỉ nên dùng khi workload semi-trusted và network performance là ưu tiên số 1.

### Egress traffic shaping (TBF)

`--qdisc=tbf` + `--qdisc-tbf-rate` (bytes/sec) + `--qdisc-tbf-burst` (bytes) — rate-limit egress non-loopback traffic, mô phỏng Linux `tc tbf`. **Cả 2 flag rate/burst đều bắt buộc khi bật tbf**, sandbox từ chối start nếu thiếu. Không nhận unit string kiểu `1mbit` như `tc(8)` — chỉ số nguyên byte/sec thuần. Chỉ áp dụng egress non-loopback; ingress và loopback không bị shape.

Có thể override per-sandbox qua annotation OCI (`dev.gvisor.flag.qdisc`, `dev.gvisor.flag.qdisc-tbf-rate`, `dev.gvisor.flag.qdisc-tbf-burst`) — nhưng annotation chỉ được **hạ thấp hơn hoặc bằng** giá trị đã cấu hình ở runtime, trừ khi bật `--allow-flag-override`. Containerd có thể set ceiling riêng qua `[runsc_config]` để annotation không vượt quá.

## Config gotchas

- `--network=none` vẫn giữ **loopback nội bộ trong sandbox** (không phải cô lập network = mất cả loopback).
- DNS lookup tên container khác lỗi trên Docker **user-defined bridge**: do embedded DNS server của Docker bind ở `127.0.0.10` trên network namespace của **host**, netstack bị cô lập nên không với tới. Fix: dùng default bridge + `--link` (không dùng embedded DNS), hoặc `--network=host` (đánh đổi bảo mật), hoặc dùng IP thẳng, hoặc chuyển sang Kubernetes (name resolution hoạt động bình thường ở đó).
- GSO (`--gso=false`) chỉ cần tắt nếu kernel host quá cũ (< 4.14.77) — performance giảm mạnh với payload lớn khi tắt, chỉ dùng làm workaround tương thích.

## Security notes

- Host passthrough (`--network=host`) là điểm **giảm cô lập rõ ràng nhất** trong toàn bộ cấu hình networking — cần đánh giá kỹ trước khi bật, không nên bật mặc định "cho nhanh".
- TBF chỉ là traffic shaping (QoS), **không phải network policy/firewall** — vẫn cần security group/NetworkPolicy ở tầng khác.

## Refs
- `g3doc/architecture_guide/networking.md`, `g3doc/user_guide/networking.md` (repo `google/gvisor`, nhánh `master`)
- `pkg/tcpip/link/qdisc/tbf` (implementation TBF)
