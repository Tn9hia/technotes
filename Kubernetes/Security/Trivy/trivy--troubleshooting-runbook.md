# Trivy — Troubleshooting Runbook (lỗi thường gặp → nguyên nhân → fix)
Tier: 2
Parent: [[Trivy]]
Related: [[trivy--db-management]], [[trivy--cache-performance]], [[trivy--filtering-suppression]]
Tags: #trivy #troubleshooting #runbook #ops

## What it does

File này là bảng tra nhanh lỗi vận hành thực tế, tổng hợp trực tiếp từ tài liệu troubleshooting chính thức (`docs/guide/references/troubleshooting.md`, verify trên nhánh `main` repo `aquasecurity/trivy`). Mỗi mục: **error message thực tế** → **nguyên nhân gốc** → **cách fix**.

## Why it exists

Trivy chạy trong rất nhiều môi trường khác nhau (CI runner, air-gapped server, laptop dev) nên lỗi phần lớn không phải bug engine mà là **mismatch giữa môi trường và giả định mặc định của Trivy** (network, socket, tmp space, cache concurrency). Có 1 bảng tra sẵn giúp phân biệt nhanh "lỗi do mình cấu hình sai môi trường" vs "lỗi thật của Trivy cần report issue".

## How it works (flow/diagram)

```
Gặp lỗi
  │
  ├─ Liên quan "unable to initialize a scanner" / không tìm thấy image
  │     → kiểm tra Docker/containerd/podman socket, hoặc registry auth/proxy
  │
  ├─ Liên quan "download vulnerability DB" / "DENIED" / rate limit
  │     → network tới mirror.gcr.io/ghcr.io, hoặc GHCR token hết hạn
  │
  ├─ Liên quan "cache may be in use" / "layer cache missing"
  │     → BoltDB file-lock (chạy song song) hoặc thiếu Redis khi multi-server
  │
  ├─ Liên quan "timeout" / "no space left on device" / "/tmp"
  │     → tài nguyên (thời gian, disk, TMPDIR) không đủ cho image lớn/Java
  │
  └─ Liên quan Maven/Java 429
        → rate limit từ Maven Central, không phải lỗi Trivy
```

## Config gotchas — Bảng tra nhanh

### Scan / Target

| Lỗi | Nguyên nhân | Fix |
|---|---|---|
| `analyze error: timeout: context deadline exceeded` | Image nhiều Java, default `--timeout 5m0s` không đủ | Tăng `--timeout 15m` (khuyến nghị chính thức) |
| `unable to initialize an image scanner` + 4 lỗi Docker/containerd/podman/remote liệt kê cùng lúc | Trivy không tìm thấy image ở bất kỳ nguồn nào trong 4 nguồn thử | Kiểm tra: gõ sai tên image / quên registry (default là Docker Hub `index.docker.io`) / `--docker-host` sai / `CONTAINERD_ADDRESS`+`CONTAINERD_NAMESPACE` sai / Podman socket chưa bật / registry cần auth / có proxy chưa set `HTTP_PROXY`/`HTTPS_PROXY` |
| `x509: certificate signed by unknown authority` | Registry dùng self-signed cert | Trust cert qua `SSL_CERT_FILE`/`SSL_CERT_DIR` (Unix) hoặc `--cacert /path/to/ca.pem` (mọi OS); `TRIVY_INSECURE=true` chỉ dùng tạm, không production |
| `write /tmp/fanal-remote...` khi scan git repo | `/tmp` không ghi được/hết chỗ, Trivy clone repo tạm vào `/tmp` | Set `TMPDIR=/my/custom/path` |
| `write /tmp/fanal-...: no space left on device` khi scan image | Layer lớn (JAR/binary) cần ghi tạm ra disk trong lúc stream-scan | Set `TMPDIR` sang disk rộng hơn, giảm `--parallel` (default 5 → 1), hoặc `--skip-files`/`--skip-dirs` loại bớt file không cần scan |
| `failed to analyze ...tools.jar: unable to open ...: stream error ...PROTOCOL_ERROR` | Lỗi biết trước khi mở JAR qua HTTP/2 stream (đang được investigate) | Workaround chính thức: `trivy image --download-java-db-only` trước, rồi scan lại |

### Database

| Lỗi | Nguyên nhân | Fix |
|---|---|---|
| `FATAL failed to download vulnerability DB` | Firewall chặn `mirror.gcr.io`/`ghcr.io` | Whitelist host theo [[trivy--db-management]], hoặc self-host DB nội bộ |
| `GET https://ghcr.io/token...: DENIED: denied` | Token GHCR cục bộ hết hạn | `docker logout ghcr.io` hoặc `unset GITHUB_TOKEN` rồi thử lại |
| `--skip-update cannot be specified with the old DB schema` | Binary Trivy quá cũ so với DB schema hiện có local | Upgrade Trivy CLI lên bản hỗ trợ DB v2 (Trivy v0.23.0+ yêu cầu DB v2, các version mới hơn theo schema mới hơn tương ứng) |
| Rate limit khi tải DB hàng loạt trong CI | Tải cộng đồng lớn từng làm GHCR bị rate-limit (đã fix từ v0.57.1 bằng cách ưu tiên `mirror.gcr.io`) | Upgrade Trivy ≥ v0.57.1; nếu vẫn dính, cân nhắc self-host/mirror DB nội bộ + `trivy server` tập trung |
| `API rate limit exceeded` khi dùng `--vex repo` | Rate limit **GitHub API** (khác hẳn rate limit OCI registry) | Set `GITHUB_TOKEN=...`. Lưu ý: token này **không** giúp ích cho rate-limit tải `trivy-db`/`trivy-java-db`/`trivy-checks` |

### Cache / Concurrency

| Lỗi | Nguyên nhân | Fix |
|---|---|---|
| `cache may be in use by another process` | BoltDB file-lock khi 2 process filesystem-cache cùng `--cache-dir` chạy song song | `--cache-backend memory` (đơn giản nhất), cache-dir riêng cho mỗi process, hoặc `trivy server` + Redis backend cho nhu cầu concurrent thật sự |
| `failed to apply layers: layer cache missing: sha256:...` khi dùng nhiều `trivy server` | Nhiều server không chia sẻ cache (mỗi server có filesystem cache riêng) | Chuyển tất cả server sang cùng 1 Redis cache backend |

### Java / Maven

| Lỗi | Nguyên nhân | Fix |
|---|---|---|
| `429 Too Many Requests` khi resolve POM từ Maven Central | `~/.m2` local cache rỗng, Trivy phải tải POM transitive dependency trực tiếp từ Maven Central, dính rate-limit theo IP | Chạy `mvn dependency:resolve` trước để populate `~/.m2`, cache `~/.m2` giữa các lần chạy CI (key theo checksum `pom.xml`); cấu hình mirror (`settings.xml` hoặc `scan.maven.mirrors` trong `trivy.yaml`); hoặc dùng `--offline-scan` (chỉ dùng cache local, **cẩn thận: POM thiếu trong cache sẽ bị âm thầm bỏ qua**, dependency tree không đầy đủ) |

### Khác

| Lỗi | Nguyên nhân | Fix |
|---|---|---|
| Lỗi không rõ nguyên nhân, hành vi bất thường | Cache/DB có thể đã hỏng hoặc lệch version | `trivy clean --all` rồi scan lại từ đầu |
| Cần debug network/auth issue (proxy, registry auth) | Không thấy rõ request/response HTTP thực tế | `--trace-http` (⚠️ CHỈ dùng debug local — có thể lộ header nhạy cảm, tự động bị Trivy disable khi phát hiện đang chạy trong CI) |

## Security notes

- Khi debug bằng `--trace-http` hoặc `--debug`, **không paste log trực tiếp vào ticket/Slack public** trước khi tự rà soát — dù Trivy có cố redact header xác thực phổ biến, vẫn có khả năng sót thông tin nhạy cảm khác trong body request.
- Với lỗi rate-limit (GHCR, Maven Central), **đừng phản xạ retry liên tục** — nhiều dịch vụ áp dụng backoff/block kéo dài hơn nếu tiếp tục request trong lúc đang bị block (`Retry-After` header cho biết thời gian tối thiểu phải chờ).

## Refs

- https://trivy.dev/latest/docs/guide/references/troubleshooting/ (nguồn chính, luôn đối chiếu bản mới nhất vì nội dung này cập nhật thường xuyên theo issue thực tế)
- https://github.com/aquasecurity/trivy/discussions (tìm theo từ khoá lỗi cụ thể nếu không có trong bảng trên)
