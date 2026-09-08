# Trivy — Cache Backend & Performance Tuning
Tier: 2
Parent: [[Trivy]]
Related: [[trivy--db-management]], [[trivy--troubleshooting-runbook]]
Tags: #trivy #cache #performance #boltdb #redis

## What it does

Cache directory của Trivy chứa: scan cache (kết quả phân tích layer/package từ lần scan trước), vulnerability DB, Java DB, misconfiguration checks, VEX repository. Backend lưu scan cache có 3 loại, chọn qua `--cache-backend`:

| Backend | Cơ chế | Mặc định cho | Đặc điểm |
|---|---|---|---|
| `fs` (File System) | BoltDB file trên `--cache-dir` | `image`, `vm`, `repo` | Persistent, nhưng **chỉ 1 process truy cập cùng lúc** |
| `memory` | Lưu trong RAM process | `fs`, `rootfs`, `config`, `sbom` | Mất khi process kết thúc, cho phép chạy song song thoải mái |
| `redis://` | Redis server bên ngoài | Không mặc định, phải chỉ định | Chia sẻ cache giữa nhiều Trivy instance/server |

## Why it exists

Mỗi loại target có đặc điểm khác nhau: scan image lặp lại nhiều lần (CI build cùng base image) hưởng lợi nhiều từ cache persistent, trong khi scan filesystem/SBOM thường chỉ chạy 1 lần nên cache trong RAM đủ dùng và tránh overhead ghi disk. Redis backend giải quyết bài toán scale-out: nhiều `trivy server` hoặc nhiều job CI chạy song song cần dùng chung 1 cache để tránh mỗi node/job tự tải lại DB.

## How it works (flow/diagram)

```
trivy image myimage   (mặc định --cache-backend fs)
        │
        ▼
BoltDB file lock trên --cache-dir
        │
        ├─ Process A đang giữ lock ──▶ Process B chạy cùng --cache-dir cùng lúc
        │                              → BLOCK/HANG chờ A nhả lock, không phải lỗi crash
        │
        └─ A xong, nhả lock ──▶ B tiếp tục bình thường
```

```
trivy server --cache-backend redis://localhost:6379
        │
        ▼
Nhiều client (trivy image --server http://...) dùng chung DB trên Redis
        → không lo file-lock, nhưng cần Redis sẵn sàng + network ổn định
```

## Config gotchas

- **BoltDB file-lock là nguyên nhân số 1 khi thấy lỗi `cache may be in use by another process`** — không phải bug, là thiết kế cố ý của BoltDB (đảm bảo không corrupt data khi nhiều process ghi cùng lúc). Chỉ xảy ra với `fs` backend; `memory`/`redis` không bao giờ gặp lỗi này.
- **Muốn chạy nhiều `trivy image` song song trên cùng máy** (vd nhiều CI job cùng runner) có 3 lựa chọn: (1) `--cache-backend memory` cho từng job (đơn giản nhất, nhưng mỗi job re-scan layer từ đầu, chậm hơn), (2) `--cache-dir` khác nhau cho mỗi job (giữ được cache riêng nhưng **mỗi thư mục tự tải 1 bản DB riêng** — tốn băng thông + disk), (3) dùng `trivy server` + Redis cache backend tập trung (tốt nhất cho quy mô lớn, tránh cả 2 vấn đề trên).
- **Vulnerability DB tự nó mở read-only, không gây lock** — lock chỉ xảy ra ở phần **scan cache** (kết quả phân tích), không phải DB. Hiểu nhầm phổ biến là tưởng DB gây lock nên đi troubleshoot sai hướng.
- **`--parallel`** (default 5) kiểm soát số layer xử lý song song — giảm xuống (vd `--parallel 1`) khi máy ít RAM/disk để tránh out-of-space tạm thời khi layer có file lớn (JAR, binary) cần ghi ra `$TMPDIR` trong lúc xử lý.
- **`$TMPDIR`/`/tmp` đầy là nguyên nhân phổ biến khi scan image lớn** — Trivy stream layer từ registry, nhưng file lớn cần thiết cho phân tích (JAR, binary) phải ghi tạm ra disk. Set `TMPDIR=/path/có/dung/lượng/lớn` nếu `/tmp` mặc định nhỏ (thường gặp trên container CI runner có `/tmp` là tmpfs giới hạn RAM).
- **`--cache-ttl`** chỉ áp dụng cho Redis backend — filesystem cache không có TTL tự động kiểu tương tự, dọn bằng `trivy clean` thủ công/định kỳ (cron) là cách chính để tránh cache phình to vô hạn trên máy CI chạy lâu dài.

## Security notes

- **Redis cache backend không tự bật TLS/auth mặc định** — `--redis-tls` phải bật tường minh, kèm `--redis-ca`/`--redis-cert`/`--redis-key` nếu cần mutual TLS. Redis cache chứa metadata scan (tên package, đôi khi cả path chứa thông tin nhạy cảm) — không nên để Redis endpoint public không xác thực trong network chia sẻ.
- **Cache directory trên filesystem không mã hoá mặc định** — nếu máy CI/server dùng chung cho nhiều team, scan cache của 1 project có thể lộ thông tin (tên package, cấu trúc image) cho project khác nếu share `--cache-dir`. Cân nhắc cache riêng theo team/project nếu boundary bảo mật cần rõ ràng.

## Refs

- https://trivy.dev/latest/docs/guide/configuration/cache/
- https://trivy.dev/latest/docs/guide/references/troubleshooting/#running-in-parallel-takes-same-time-as-series-run
- https://github.com/etcd-io/bbolt (BoltDB, giải thích cơ chế file-lock)
