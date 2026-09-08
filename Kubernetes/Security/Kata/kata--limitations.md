# Kata — Known Limitations (so với runc)
Tier: 2
Parent: [[kata-containers]]
Related: [[kata-containers]], [[kata--security-policy]]
Tags: #kata #limitations #kubernetes

## What it does

Danh sách hành vi Kata **cố tình khác** hoặc **chưa hỗ trợ** so với `runc` — tài liệu chính thức, được track qua GitHub issue gắn label `limitation`, luôn cập nhật (không phải bài blog cũ).

## Why it exists

Kata thêm 1 lớp VM ở giữa → một số thứ vốn "miễn phí" với `runc` (share network namespace, bind-mount `/proc` tuỳ ý, hotplug host device...) hoặc không khả thi về mặt kiến trúc, hoặc cố tình bị chặn vì lý do bảo mật (CVE cụ thể). Phân biệt 2 loại quan trọng: **limitation có thể fix** (thiếu code) và **limitation kiến trúc** (do bản chất VM, khó/không thể fix).

## How it works — danh sách theo nhóm

### Nhóm "có thể fix trong tương lai"

| Limitation | Chi tiết | Workaround |
|---|---|---|
| Podman | Chưa hỗ trợ chính thức (issue #722, còn mở) | Dùng containerd + `nerdctl`, hoặc Docker ≥ 22.06 |
| `checkpoint`/`restore` | Không có (OCI spec cũng không bắt buộc) | Không có — cân nhắc VM save/restore ở tầng hypervisor nếu cần tương đương |
| `events` command | OOM notification, Intel RDT stats chưa đầy đủ | — |
| `update` command | Chỉ block I/O weight chưa hỗ trợ, còn lại hoạt động | — |

### Nhóm networking (kiến trúc — khó fix)

- **`hostNetwork: true` / `--net=host`**: **KHÔNG hỗ trợ**. Cảnh báo mạnh từ chính docs: cố dùng có thể khiến Kata **sửa đổi/phá luôn cấu hình network của host**. Nếu cần host network cho 1 số pod cụ thể trong cluster có Kata, chạy pod đó bằng `runc` (RuntimeClass mặc định), không ép qua Kata.
- **Share network namespace giữa nhiều container** (`docker run --net=container:X`): không hỗ trợ. Nếu ép 1 Kata container share netns với 1 `runc` container, Kata sẽ "chiếm" toàn bộ network interface của netns đó gắn vào VM → **container runc kia mất kết nối mạng**.
- **`docker run --link`**: không hỗ trợ (bản thân Docker cũng đã deprecate).

### Nhóm storage/volume (kiến trúc)

- **`emptyDir.sizeLimit` với `block-plain`/`block-encrypted` mode**: Kata size block device theo **dung lượng filesystem host chứa emptyDir**, không theo `sizeLimit` thật (shim không nhận được giá trị này qua CRI mount info hiện tại). Overhead ext4 metadata (~0.08% dung lượng logical, số liệu thực nghiệm chính thức: ~2.5GB overhead trên filesystem logical 2.9TB) có thể tự nó vượt 1 `sizeLimit` nhỏ → **pod bị evict dù chưa ghi dữ liệu thực nào**. Chưa có config option nào giới hạn kích thước image backing này (issue #2438 còn mở tại thời điểm viết). Workaround: để dư headroom `sizeLimit` > 0.08% dung lượng host filesystem, hoặc đặt volume kubelet trên 1 filesystem host nhỏ riêng.
- **`volumeMounts.subPath`**: không hỗ trợ (issue #2812, và riêng case emptyDir ở issue #1728).
- **`hostPath` hành vi khác `runc`**:
  - Non-TEE: dùng filesystem sharing (virtio-fs) — file sync 2 chiều bình thường.
  - **TEE (confidential)**: filesystem sharing **bị tắt**, host file được **copy** vào guest lúc container start, **không sync lại** sau đó — thay đổi ở host sau khi container đã start sẽ không bao giờ tới guest.
  - `hostPath` type `BlockDevice`: Kata **hotplug thẳng block device đó vào guest** (khác hẳn semantics thông thường của bind-mount).
  - Path dưới `/dev`: agent bind-mount trực tiếp từ **filesystem của guest** (không phải từ host) nếu path là device/special file.
- **Bind mount `/proc`, `/sysfs`**: chặn nghiêm ngặt vì CVE-2019-16884 (escape qua `/proc` bind mount không đúng) và CVE-2019-19921 (symlink tới proc/sysfs). Chỉ các path sau được phép bind vào `/proc/*`: `cpuinfo`, `diskstats`, `meminfo`, `stat`, `swaps`, `uptime`, `loadavg`, `net/dev`. Tool monitoring cố mount toàn bộ `/proc` từ host (kiểu node-exporter chạy trong pod) **sẽ bị chặn** nếu chạy qua Kata.

### Nhóm privileged/image (đã nêu ở [[kata--security-policy]], liệt kê lại ngắn gọn)

- Privileged container: capability chỉ có hiệu lực **trong guest**, pass host-device mặc định **phải bị tắt** (`privileged_without_host_devices = true`).
- Guest-pulled image (nydus): phải set `runAsUser`/`runAsGroup`/`fsGroup`/`supplementalGroups` tường minh, nếu không container có thể chạy sai UID/GID hoặc bị Agent Policy tự sinh reject.

### Resource constraints — "the constraints challenge"

Áp dụng CPU/memory/storage limit cho 1 workload trên Kata **phức tạp hơn** `runc` vì có nhiều lớp độc lập cần đồng bộ:
```
Trong VM:      guest kernel (kernel cmdline / sysctl sớm)
Trong container: cgroup trong guest (giống runc thông thường)
Ngoài VM:      constraint lên chính process hypervisor trên host
Ngoài VM:      constraint lên toàn bộ process bên trong hypervisor (qua config hypervisor)
```
Đặt limit ở 1 lớp không tự động đồng bộ sang lớp khác — ví dụ set `default_memory` thấp ở hypervisor level vẫn có thể khiến workload OOM dù `resources.limits.memory` ở pod spec cao hơn, vì guest VM vốn dĩ chỉ có từng đó RAM để cấp phát.

## Config gotchas

Không có "config để tắt" phần lớn limitation ở đây — đây là hành vi kiến trúc, không phải bug có thể vá bằng 1 dòng TOML. Cách xử lý thực tế luôn là: (1) tránh pattern đó, hoặc (2) chạy riêng workload cần pattern đó bằng `runc`.

## Security notes

Nhiều "limitation" ở đây **chính là chủ đích bảo mật** (chặn `/proc` bind mount, chặn host network, chặn privileged host device) — đừng coi đây là thiếu sót cần "fix" bằng cách tìm workaround vòng qua, vì làm vậy sẽ xoá bỏ chính lợi ích cách ly mà Kata mang lại.

## Refs

- https://github.com/kata-containers/kata-containers/blob/main/docs/Limitations.md (nguồn chính, luôn cập nhật — kiểm tra lại link issue trước khi dựa vào để ra quyết định vì trạng thái issue có thể đã đổi)
- Danh sách limitation issue còn mở (real-time): https://github.com/pulls?utf8=%E2%9C%93&q=is%3Aopen+label%3Alimitation+org%3Akata-containers
