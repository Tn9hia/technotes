# gVisor — Debugging & Observability
Tier: 2
Parent: [[gvisor]]
Related: [[gvisor--resource-model]], [[gvisor--common-errors-lessons]]
Tags: #gvisor #debug #observability #ops

## What it does

Tập hợp toàn bộ công cụ để: (1) xem log/strace khi container lỗi, (2) lấy stack trace/profile khi sandbox đang chạy, (3) export metric Prometheus về **bản thân gVisor** (khác với giám sát hành vi workload — xem Runtime Monitoring bên dưới), (4) debug bằng Go debugger (`dlv`).

## Why it exists

Vì Sentry chặn hoàn toàn syscall của app, các công cụ debug thông thường (`strace` trên host, ví dụ) **không thấy được** gì bên trong sandbox — phải dùng cơ chế riêng của `runsc` để lấy log/strace/stack.

## How it works (flow/diagram)

### 1. Debug log + strace

```json
"runtimeArgs": ["--debug-log=/tmp/runsc/", "--debug", "--strace"]
```
- File kết thúc bằng `.boot` = strace của **app** (dùng để tìm syscall thiếu/lỗi trong gVisor).
- File kết thúc bằng `.create` = lý do container **không start được**.
- Trailing `/` trong `--debug-log` → coi là thư mục, tự đặt tên file theo `runsc.log.%TIMESTAMP%.%COMMAND%.txt` — tránh bị nhiều lệnh ghi đè lẫn nhau. Nếu dùng path cụ thể không có biến (`%ID%`, `%COMMAND%`...) và có nhiều sandbox/command chạy đồng thời → **log bị ghi đè/trộn lẫn nhau**.
- `--log-packets` thêm khi debug vấn đề mạng.

### 2. Stack traces khi đang chạy

```bash
runsc --root <root> debug --stacks <container-id>
```
`--root` do Docker cấp, thường là `/var/run/docker/runtime-runsc/moby` — không nhớ được thì tra trong log `runsc`.

### 3. Debugger (dlv)

Build bản debug (`make dev BAZEL_OPTIONS="-c dbg --define gotags=debug"` hoặc dùng nightly release), tìm PID sandbox qua `docker inspect`, attach `dlv` với quyền root, đặt breakpoint thẳng vào hàm Go trong Sentry (vd. `gvisor.dev/gvisor/pkg/sentry/socket/netstack.(*sock).Accept`).

### 4. Profiling

```json
"runtimeArgs": ["--profile"]
```
Rồi dùng `runsc debug --profile-heap=<file>` / `--profile-cpu=<file> --duration=30s`, mở bằng `go tool pprof`.

⚠️ **`--profile` nới lỏng seccomp filter của sandbox — không bật ở production.**

Nếu forward port qua Docker, traffic đi qua `docker-proxy` làm nhiễu profiling — gửi trực tiếp tới IP container (vd. IP `docker0` là `.1` thì container thường là `.2`) để tránh nhiễu.

### 5. Metric server (Prometheus, quan sát chính gVisor — KHÔNG phải hành vi workload)

`runsc metric-server` chạy **unsandboxed** như sidecar, đọc dữ liệu từ `--root` để export `/metrics`. Bật bằng `--metric-server=host:port` trong runtimeArgs (metric mặc định **tắt**). Value flag phải **khớp chính xác** giữa runtime config và lệnh `metric-server`.

Metric hữu ích: `sandbox_presence`, `sandbox_running` (so 2 cái để tìm sandbox "biết nhưng không chạy" = crash âm thầm), `sandbox_metadata` (version, platform, network type đang dùng), `sandbox_capabilities` (audit capability thực tế đang cấp), `num_sandboxes_broken_metrics`.

Có thể chạy metric-server **trong** 1 sandbox khác (`--host-uds=all`) nhưng **cảnh báo chính thức**: làm vậy sandbox đó có full control mọi sandbox khác trên máy — chỉ là defense-in-depth, không phải bảo mật đầy đủ.

Trên Kubernetes, metric-server tự thêm label `pod_name`/`namespace_name` từ annotation containerd (`io.kubernetes.cri.sandbox-name`/`-namespace`) — dễ join với dashboard theo pod.

### 6. Runtime Monitoring (khác mục đích với metric server!)

Dùng để giám sát **hành vi workload bên trong sandbox** (mục tiêu chính: threat detection / intrusion detection), không phải quan sát nội bộ gVisor. Stream "trace point" (mọi syscall + event quan trọng như container start) tới 1 process giám sát ngoài, tách biệt sandbox. Tích hợp được với Falco. Đọc thêm ở file riêng nếu cần triển khai (không đủ chi tiết để tách Tier 2 riêng tại đây — xem link).

## Config gotchas

- `--debug-log` không có trailing `/` + có nhiều container/sandbox cùng lúc = log bị trộn. Luôn dùng biến `%ID%`/`%CID%`/`%COMMAND%` khi cần tách log theo sandbox/container.
- `--metric-server` value string phải giống hệt nhau giữa runtime config và câu lệnh `metric-server` (so sánh string, không phải resolve địa chỉ).

## Security notes

- `--profile` và `--host-uds=all` đều là ví dụ "bật debug/observability = nới lỏng bảo mật" — luôn tắt các flag này khi rời môi trường debug/staging.

## Refs
- `g3doc/user_guide/debugging.md`, `g3doc/user_guide/observability.md`, `g3doc/user_guide/runtime_monitoring.md` (repo `google/gvisor`, nhánh `master`)
- Runtime monitoring overview: `pkg/sentry/seccheck/README.md` trong repo; Falco integration: https://gvisor.dev/docs/tutorials/falco/
