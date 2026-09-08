# Kata — Troubleshooting & Debug
Tier: 2
Parent: [[kata-containers]]
Related: [[kata--configuration]], [[kata--architecture-runtime-rs]], [[kata--k8s-integration]]
Tags: #kata #debug #ops #logs

## What it does

Tập hợp cách bật debug, đọc log, và vào tận bên trong guest VM để soi lỗi khi 1 pod chạy bằng Kata không hoạt động như mong đợi.

## Why it exists

Kata thêm 1 lớp (VM + agent + vsock) giữa containerd và container thật — khi có lỗi, log thông thường của `kubectl logs`/`crictl logs` không đủ vì lỗi có thể nằm ở: shim, hypervisor, kênh vsock, hoặc agent trong guest. Cần biết log của **từng lớp** nằm ở đâu.

## How it works (flow/diagram)

**1. Bật debug** (làm qua drop-in, không sửa file gốc — xem [[kata--configuration]]):

```toml
# Go runtime và runtime-rs đều có enable_debug, nhưng hành vi log khác nhau
[runtime]
enable_debug = true
log_level = "debug"     # chỉ runtime-rs hiểu log_level; Go runtime chỉ có enable_debug (bool)

[hypervisor.qemu]
enable_debug = true      # bật thêm kernel param debug + verbose QEMU invocation
log_level = "debug"       # runtime-rs only

[agent.kata]
enable_debug = true       # tự thêm kernel param agent.log=debug, không cần set tay
log_level = "debug"       # runtime-rs only
```

**2. Đọc log — khác nhau giữa 2 runtime**:

| | Go runtime | runtime-rs |
|---|---|---|
| Log chính | Qua `containerd` log **+** journald (`kata` identifier) | Thẳng vào journald (`kata` identifier), **không cần** bật containerd debug |
| Cần bật containerd debug? | **Có** — `[debug] level = "debug"` trong containerd config, nếu không sẽ thiếu phần lớn log | Không bắt buộc |
| Lệnh xem log | `sudo journalctl -t kata` | `sudo journalctl -t kata` |

**3. Thứ tự debug 1 pod Kata không tạo được (checklist thực dụng)**:
```
1. kubectl get events -w                     # lỗi ở tầng K8s/scheduler trước khi chạm containerd?
2. kubectl get runtimeclasses                 # RuntimeClass có tồn tại, đúng tên?
3. cat /opt/kata/containerd/config.d/kata-deploy.toml   # đang trỏ đúng shim (Go/Rust)?
4. bật enable_debug (xem trên) + tạo lại pod
5. sudo journalctl -t kata --since "5 min ago"
6. ps aux | grep -E 'qemu|cloud-hypervisor|firecracker|dragonball'   # VMM có thực sự start?
7. ls -l /dev/kvm                              # thiếu quyền/thiếu module = fail ngay từ đầu
```

**4. Vào thẳng guest VM (debug console)** — 2 cách:

- **Simple debug console (khuyến nghị, nhanh)**: chỉ cần image guest có `/bin/sh`/`/bin/bash`.
  ```toml
  [agent.kata]
  debug_console_enabled = true
  ```
  Sau đó:
  ```bash
  sudo ctr run --runtime io.containerd.kata.v2 -d docker.io/library/ubuntu:latest testdebug
  kata-runtime exec testdebug   # chỉ có ở Go runtime; runtime-rs equivalent cần tự kiểm chứng thêm
  ```
  Namespace mặc định là `k8s.io` (đúng cho containerd + K8s); với CRI-O phải set `--runtime-namespace default` tường minh — nhầm chỗ này là lý do phổ biến nhất khiến `kata-runtime exec` báo "sandbox not found" dù pod đang chạy tốt.

- **Traditional debug console**: cần build lại rootfs/initrd với shell + `agent.debug_console` trong kernel_params — phức tạp hơn nhiều, chỉ cần khi simple console không đủ (ví dụ cần login qua serial console thật sự, hoặc guest không có systemd để chạy kata-agent theo cách chuẩn).

**5. Thu thập log để báo bug / RCA**:
```bash
sudo kata-collect-data.sh > /tmp/kata-report.log   # script build sẵn từ src/runtime (Go runtime)
```
Sau đó parse log bằng `kata-log-parser` (chuyển log Kata thành JSON/TOML/XML/YAML để dễ grep/phân tích theo timeline).

**6. Debug ở mức source code (attach debugger)**: dùng `dlv` (Delve) attach vào PID shim đang chạy, hoặc chèn 1 RuntimeClass riêng trỏ script `containerd-shim-katadbg-v2` để debug ngay từ lúc shim khởi động (xem [[kata--k8s-integration]]). Lưu ý chính thức: **tại thời điểm viết tài liệu này chỉ có hướng dẫn cho Go shim** — runtime-rs (Rust) chưa có tài liệu debugger tương đương chính thức, chỉ có dev tool `shim-ctl` để test shim độc lập không qua containerd.

## Config gotchas

- **journald rate limiting** che mất log khi bật full debug (lượng log sinh ra rất lớn). Kiểm tra:
  ```bash
  sudo journalctl --since today | grep -F Suppressed
  ```
  Nếu thấy dòng "Suppressed N messages" → log Kata đang bị drop. Fix tạm thời (toàn hệ thống, ảnh hưởng mọi service khác):
  ```ini
  # /etc/systemd/journald.conf
  RateLimitInterval=0s
  RateLimitBurst=0
  ```
  rồi `systemctl restart systemd-journald`. **Nhớ trả lại giá trị cũ sau khi debug xong** — tắt rate limit toàn cục vĩnh viễn có thể khiến 1 service log lỗi loop chiếm hết đĩa.
- Debug console (`debug_console_enabled` hoặc `agent.debug_console`) là **cửa hậu vào guest không cần auth** — chỉ bật tạm thời khi debug, tắt lại ngay sau, không để bật ở production lâu dài.

## Security notes

- Debug console không có cơ chế authentication — bất kỳ ai control được vsock/host namespace của sandbox đó đều exec được vào guest với quyền root. Coi đây là 1 "backdoor tạm thời", quản lý vòng đời bật/tắt chặt như quản lý 1 secret.
- Bật `enable_debug` sinh log verbose có thể vô tình log ra thông tin nhạy cảm (đối số kernel, path, đôi khi cả nội dung request agent) — không để log level debug bật vĩnh viễn ở production, và đảm bảo log pipeline (nếu forward ra ngoài, ví dụ fluentd) có kiểm soát access phù hợp.

## Refs

- https://github.com/kata-containers/kata-containers/blob/main/docs/Developer-Guide.md#troubleshoot-kata-containers (phần chính, bao gồm debug console đầy đủ 2 kiểu)
- https://github.com/kata-containers/kata-containers/blob/main/docs/Debug-shim-guide.md (attach `dlv` debugger vào shim)
- https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-import-kata-logs-with-fluentd.md (nếu cần forward log ra ngoài node)
