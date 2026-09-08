# Kata — Configuration (configuration.toml, drop-in, Go vs runtime-rs)
Tier: 2
Parent: [[kata-containers]]
Related: [[kata--architecture-runtime-rs]], [[kata--k8s-integration]], [[kata--security-policy]]
Tags: #kata #configuration #toml

## What it does

Toàn bộ hành vi runtime/agent/hypervisor của Kata được điều khiển bởi 1 (hoặc nhiều, qua drop-in) file TOML — `configuration.toml`. Kata **không có khái niệm "stable branch"**: dự án release rolling, mỗi tháng ra 1 minor version snapshot từ `main`, bug fix không backport về version cũ (theo `Release-Process.md` chính thức) — nghĩa là "phiên bản ổn định nhất" luôn là **bản mới nhất đã release**, không phải 1 nhánh LTS riêng.

## Why it exists

Kata phải hỗ trợ nhiều hypervisor, nhiều kiến trúc CPU, nhiều mức bảo mật (confidential computing) — 1 file config tập trung cho phép chọn combination đó mà không phải build lại binary. Cơ chế drop-in (`config.d/`) tồn tại để **customization sống sót qua upgrade** (kata-deploy tự động ghi lại base config mỗi lần update, sẽ xoá mất mọi sửa đổi tay trực tiếp trong file gốc).

## How it works (flow/diagram)

**Thứ tự tìm config file** (shimv2 runtime, theo `src/runtime/README.md`):
1. Option truyền qua containerd shimv2 (`ConfigPath` trong runtime options).
2. Biến môi trường `KATA_CONF_FILE`.
3. Default path — và ở bước này có 1 quy tắc dễ gây lỗi: **`/etc/kata-containers/configuration.toml` (nếu tồn tại) luôn thắng** `/usr/share/defaults/...` hay `/opt/kata/share/defaults/...` (packaged default).

**Cơ chế drop-in** (khi cài qua `kata-deploy`):
```
/opt/kata/share/defaults/kata-containers/runtimes/<hypervisor>/
├── configuration.toml          ← KHÔNG sửa trực tiếp (bị ghi đè khi upgrade)
└── config.d/
    ├── 10-*.toml                ← reserved: core kata-deploy settings
    ├── 20-*.toml                ← reserved: debug settings
    ├── 30-*.toml                ← reserved: kernel params
    ├── 50-*.toml                ← reserved: settings từ helm chart
    └── 50-89-*.toml              ← khoảng dành cho custom config của bạn
```
File trong `config.d/` được đọc theo thứ tự alphabet, file sau đè giá trị file trước. Không có `config.d/` hoặc thư mục rỗng = không lỗi.

**Kiểm tra đang chạy runtime nào / config nào**:
```bash
# Xem runtime_path để biết Go hay Rust runtime
cat /opt/kata/containerd/config.d/kata-deploy.toml

# Xem toàn bộ path config mà runtime sẽ thử load, theo thứ tự ưu tiên
kata-runtime --show-default-config-paths     # chỉ có ở Go runtime

# Xem config effective đang dùng (bao gồm path file đang load)
kata-runtime env                              # chỉ có ở Go runtime
```
> runtime-rs (tại thời điểm viết) chưa thấy tài liệu về lệnh tương đương `kata-runtime env`/`--show-default-config-paths` — nếu cần kiểm chứng, đọc log khởi động shim (`journalctl -t kata`, shim luôn log rõ path file config đang dùng) hoặc dùng dev tool `shim-ctl` (crate `shim-ctl` trong `src/runtime-rs`).

## Config gotchas — khác biệt Go runtime ↔ runtime-rs (hay bị migrate sai)

Bảng dưới trích từ tài liệu migrate chính thức, chỉ liệt kê nhóm **dễ gây lỗi âm thầm nhất** (option bị đổi tên/đơn vị mà không báo lỗi khi parse — dùng nhầm coi như tính năng đó không tồn tại):

| Go runtime | runtime-rs | Khác biệt |
|---|---|---|
| `[hypervisor.qemu] seccompsandbox` | `seccomp_sandbox` | Đổi tên (thêm `_`) |
| `[hypervisor.qemu] hypervisor_loglevel` (số `uint32`) | `log_level` (string: trace/debug/info/warn/error/critical) | Đổi kiểu dữ liệu |
| `[runtime] guest_selinux_label` | `[hypervisor.qemu] selinux_label` | Đổi cả tên lẫn bảng TOML chứa nó |
| `[runtime] create_container_timeout` (giây) | `[agent.kata] create_container_timeout` (giây trong file, lưu nội bộ dạng ms) | Đổi bảng |
| `[agent.kata] dial_timeout` (giây) | `dial_timeout_ms` (mili-giây) | Đổi tên + đơn vị |
| `[factory]` (bảng top-level, VM templating) | `[hypervisor.qemu.factory]` | Chuyển vào trong bảng hypervisor; field VMCache (`vm_cache_number`, `vm_cache_endpoint`) **bị drop hẳn**, không còn tương đương |
| `[runtime] experimental_force_guest_pull` (bool) | `[runtime] experimental = ["force_guest_pull"]` | Đổi từ boolean sang list tính năng |

Nhóm **chỉ có ở Go runtime, chưa được implement ở runtime-rs** (không phải bug, chỉ là chưa code xong): `enable_numa`, `numa_mapping`, toàn bộ `net_rate_limiter_bw/ops_*` (rate limit mạng theo băng thông — `disk_rate_limiter_*` thì **có** ở cả hai).

Nhóm **chỉ có ở runtime-rs** (không tồn tại ở Go, nên đừng tìm): `vm_rootfs_driver`, `queue_size`/`num_queues` (block multi-queue), `network_queues`, `hugepage_type`, `rootless_user` (structured, thay vì chỉ 1 boolean `rootless` ở Go), toàn bộ bảng `[agent.kata.mem_agent]` (memory agent — tối ưu compact/reclaim RAM guest, không có tương đương Go).

> ⚠️ **Lesson learned quan trọng nhất phần này**: `guest_swap_*` tồn tại trong file template QEMU của runtime-rs nhưng **QEMU plugin sẽ reject `enable_guest_swap = true` khi validate** — set giá trị này với QEMU trên runtime-rs coi như vô tác dụng (không lỗi rõ ràng), tài liệu chính thức khuyến nghị xoá hẳn các field `guest_swap_*` khỏi template QEMU runtime-rs vì chúng chỉ có ý nghĩa với hypervisor khác có hỗ trợ guest swap.

## Security notes

- `enable_annotations` = whitelist tên annotation hypervisor được phép override per-pod — mặc định rất hạn chế. Annotation liên quan tới đường dẫn binary (`path`, `jailer_path`, `virtio_fs_daemon`, `vhost_user_store_path`) đều được đánh dấu **(R) restricted** — bắt buộc phải khớp với 1 whitelist giá trị riêng (`valid_hypervisor_paths`, `valid_jailer_paths`...) trong configuration.toml, không chỉ whitelist tên annotation. Thiếu 1 trong 2 lớp whitelist này = lỗ hổng cho phép chỉ định binary tuỳ ý.
- `entropy_source` annotation cũng bị restricted (R) — kiểm soát nguồn random dùng cho guest, không nên cho phép trỏ tuỳ ý tới path bất kỳ trên host.

## Refs

- https://github.com/kata-containers/kata-containers/blob/main/docs/runtime-configuration.md (drop-in mechanism)
- https://github.com/kata-containers/kata-containers/blob/main/docs/migrating-config-go-runtime-to-runtime-rs.md (bảng migrate đầy đủ — hiện chỉ cover QEMU, hypervisor khác đang được bổ sung dần)
- https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-set-sandbox-config-kata.md (toàn bộ danh sách pod annotation + phần Restricted annotations)
- https://github.com/kata-containers/kata-containers/blob/main/src/runtime/README.md (thứ tự tìm config file, `kata-runtime env`)
- https://github.com/kata-containers/kata-containers/blob/main/docs/Release-Process.md (xác nhận: không có stable branch, chỉ có rolling monthly release)
