# gVisor — Checkpoint/Restore
Tier: 2
Parent: [[gvisor]]
Related: [[gvisor--resource-model]]
Tags: #gvisor #checkpoint #ha #migration

## What it does

Lưu toàn bộ state của 1 container đang chạy ra file (checkpoint), sau đó nạp lại vào 1 container mới từ file đó (restore) — kiểu CRIU nhưng tự implement bên trong gVisor, không phụ thuộc CRIU.

## Why it exists

Nếu không có tính năng này, muốn "đóng băng" 1 workload lớn (vd. LLM đã load xong model vào RAM) để restart nhanh hoặc migrate sang máy khác, bạn phải khởi động lại từ đầu — tốn "time to first instruction" rất lớn với workload nặng. Checkpoint/restore cho phép tách rời "cold start" (load model) khỏi "warm resume" (chỉ nạp lại state).

## How it works (flow/diagram)

```
runsc run <id>                                  # container đang chạy
runsc checkpoint --image-path=<path> [--leave-running] <id>
                                                 # ghi state ra <path>, mặc định DỪNG container
                                                 # --leave-running: restore-ngay-tại-chỗ, PID có thể đổi
runsc create <new-id>
runsc restore --image-path=<path> <new-id>      # nạp state vào container MỚI
```

Lưu ý bắt buộc: **mỗi `image-path` phải unique** (không ghi đè 2 checkpoint vào cùng thư mục). Khi dùng `--leave-running`, phải truyền lại **toàn bộ top-level flag** giống lúc `run`.

### Optimization flags (đáng nhớ khi cần restore nhanh cho workload lớn)

- `--compression=none|flate-best-speed` — `none` (default) nhanh hơn, cho phép restore kernel + memory **song song**, và là điều kiện bắt buộc để dùng các optimization dưới đây.
- `--exclude-committed-zero-pages` — bỏ qua lưu trang nhớ zero-filled (giảm size checkpoint mạnh với workload có vùng nhớ lớn toàn zero như LLM) — đánh đổi: **checkpoint lâu hơn** (phải scan hết trang để biết trang nào toàn zero).
- `--direct` — dùng `O_DIRECT`, bỏ qua host page cache, cần `--compression=none`. Hữu ích khi snapshot chỉ đọc 1 lần và không restore lại trên máy đó lần nữa (tránh làm bẩn page cache vô ích).
- `--background` (chỉ cho restore) — app **bắt đầu chạy ngay khi kernel state load xong**, phần memory/data còn lại nạp async trong nền. Nếu app chạm vào 1 trang chưa load, gVisor ưu tiên load ngay trang đó để không block app. Giảm mạnh "time to first instruction" cho app lớn.

**Cảnh báo khi dùng `--background`**: file checkpoint có thể vẫn bị sandbox giữ FD mở **sau khi app đã chạy** (đang restore nền) → xoá file pages lúc này **có thể không giải phóng disk ngay** (POSIX fs) hoặc **không xoá được** (non-POSIX fs), và **không unmount được** mount chứa file snapshot cho tới khi restore xong hoàn toàn. Dùng `runsc wait --restore` để chờ restore hoàn tất trước khi dọn dẹp `--image-path`.

### Application-Driven Checkpoint/Restore (app tự trigger, không cần gọi `runsc` từ ngoài)

Cấu hình hoàn toàn qua **OCI annotation**, workload tương tác qua file `/proc/gvisor/checkpoint` (luôn tồn tại, mặc định read-only mode `0444`):
- `dev.gvisor.internal.checkpoint.path` — set trên **container gốc/đầu tiên**, bắt buộc để bật tính năng.
- `dev.gvisor.internal.checkpoint.enable=true` — set **per-container**, cho phép container đó ghi vào `/proc/gvisor/checkpoint` (chuyển mode thành `0666`) để tự trigger checkpoint.

Protocol: mở file (đăng ký nhận checkpoint *kế tiếp*) → ghi `1` để trigger (optional, có thể chỉ đọc để chờ checkpoint do process khác trigger) → đọc, block tới khi xong, trả về `resume` (sandbox tiếp tục chạy) / `restore` (đây là bản đã restore) / `error`. Trigger lần 2 trên cùng FD → lỗi `ENXIO`. File `/proc/gvisor/spec_environ` cho phép đọc lại env var mới nhất sau khi restore (khác với env lúc tạo container).

## Config gotchas

- Networking + checkpoint/restore: support với cả `--network=sandbox` (default), `none`, `host`. Với `--network=host`, **socket host không lưu được**: TCP listening socket được tạo lại (mất backlog đang chờ), socket đã connect trả `ECONNRESET` sau restore — app phải tự reconnect.
- Restore trên máy khác CPU feature khác nhau → gVisor **verify strict**, mặc định fail nếu thiếu CPU feature nào đã bật lúc checkpoint. Dùng annotation `dev.gvisor.internal.cpufeatures` để giới hạn tập feature cho phép, giúp checkpoint/restore portable giữa các máy khác CPU. Xem tập feature bằng `runsc cpu-features`.
- GPU checkpoint/restore dùng `cuda-checkpoint` của NVIDIA qua `--cuda-checkpoint-path` — **không hỗ trợ trên arm64** (giới hạn từ chính cuda-checkpoint, không phải gVisor).
- Docker checkpoint hiện có giới hạn tương thích: Docker cũ (≤18.03.0-ce) bị hang khi checkpoint (bug Moby #37360, đã fix ở bản mới); Docker **không hỗ trợ restore vào container khác** container đã tạo checkpoint (cần cho migration thật sự); Docker chưa support `--checkpoint-dir` (Moby #37344).

## Security notes

- Checkpoint file chứa **toàn bộ memory state** của app — coi nó nhạy cảm tương đương RAM dump, cần kiểm soát quyền truy cập file `--image-path` như dữ liệu bí mật.

## Refs
- `g3doc/user_guide/checkpoint_restore.md` (repo `google/gvisor`, nhánh `master`)
- cuda-checkpoint: https://github.com/NVIDIA/cuda-checkpoint
