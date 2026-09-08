# gVisor — Common Errors & Lessons Learned
Tier: 2
Parent: [[gvisor]]
Related: [[gvisor--filesystem-gofer]], [[gvisor--networking]], [[gvisor--kubernetes-containerd]], [[gvisor--platforms]]
Tags: #gvisor #troubleshooting #ops #compatibility

## What it does

Danh sách lỗi/gap **đã được xác nhận chính thức** (từ FAQ + compatibility doc của gVisor, không phải suy đoán từ blog cũ), kèm nguyên nhân gốc và cách xử lý.

## Why it exists

gVisor tự re-implement toàn bộ Linux syscall surface — nghĩa là **lỗi "container chạy được với runc nhưng lỗi với runsc" gần như luôn là gap tương thích hoặc feature chưa implement**, không phải bug logic của bản thân container image. Biết trước danh sách gap giúp không mất thời gian debug sai hướng.

## Danh sách gap tương thích đã biết (từ `compatibility.md`, chính thức — không phải "nghe nói")

- **Resource limit trong sandbox chỉ để accounting, không enforce.** cgroup CPU/memory *trong* sandbox tồn tại và đo được, nhưng **không giới hạn giữa các process cạnh tranh trong cùng 1 sandbox**. Muốn giới hạn resource sandbox từ bên ngoài vẫn phải dựa vào cgroup Linux-native ở tầng host (Docker/K8s tự làm việc này).
- **Không mount được block device filesystem** (`fat32`, `ext3`, `ext4`) trực tiếp trong sandbox — không có driver block device trong gVisor kernel. Workaround: mount trên host Linux trước, rồi expose thư mục đã mount cho sandbox (bind mount).
- **`iptables` chỉ hỗ trợ 1 phần** — đủ để chạy "Docker-in-gVisor", không hơn.
- **Device file cho hardware custom không support**, trừ NVIDIA GPU và TPU (2 case đã có driver riêng trong gVisor).
- **`io_uring` tắt mặc định**, khi bật chỉ hỗ trợ I/O cơ bản. Tương tự `nftables`. Lưu ý thực tế: hầu hết runtime/library tự động **probe và fallback** sang syscall I/O khác nếu `io_uring` không khả dụng, nên phần lớn app vẫn chạy được dù benchmark syscall-list trông như thiếu nhiều.
- **Không dùng KVM được từ BÊN TRONG sandbox** (app trong sandbox không tự ảo hoá lồng được) — khác với việc *bản thân gVisor* dùng KVM làm platform (2 khái niệm độc lập, đừng nhầm).
- **Gap riêng khi tích hợp Kubernetes** nằm ở danh sách incompatible feature của GKE Sandbox — xem [[gvisor--kubernetes-containerd]] (mục GKE Sandbox), vì đây là restriction thêm bởi GKE, không phải toàn bộ đều do gVisor upstream.

## Lỗi thường gặp (nguyên văn từ FAQ chính thức, kèm nguyên nhân + fix)

| Triệu chứng | Nguyên nhân | Fix |
|---|---|---|
| Container chạy OK với `runc` nhưng lỗi với `runsc` | Compatibility gap hoặc feature chưa implement | Bật `--debug --strace --debug-log=`, xem file `.boot`; nếu không có issue tương ứng thì file bug kèm log ([[gvisor--debugging-observability]]) |
| `open /run/containerd/.../log.json: no such file` | Host kernel quá cũ, không có `memfd_create` | Nâng kernel (gVisor yêu cầu Linux ≥ 5.6) |
| `flag provided but not defined: -console` | Docker version quá cũ | Nâng Docker theo Docker Quick Start |
| `docker cp` xong không thấy file mới | gVisor cache nội dung thư mục (dentry cache), chưa biết có file mới | Tạo 1 file bất kỳ trong thư mục để force refresh; hoặc bật shared root fs; `kubectl cp` không bị vì nó exec vào sandbox |
| `panic: unable to attach: operation not permitted` / `fork/exec /proc/self/exe: invalid argument` | Permission trên binary `runsc` sai | `chmod a+rx /usr/local/bin/runsc` (runsc cần re-exec chính nó với quyền thấp hơn) |
| `mount submount "/etc/hostname": ... input/output error` | Bug kernel Linux 5.1–5.3.15 / 5.4.2 / 5.5 cụ thể | Nâng kernel, hoặc set `LimitMEMLOCK=infinity` trong `containerd.service` rồi `daemon-reload && restart containerd` |
| `RuntimeHandler "runsc" not supported` | containerd CRI chưa cấu hình đúng runtime handler, hoặc kubeadm ưu tiên Docker thay vì containerd | Kiểm tra containerd config + restart; nếu dùng kubeadm, set rõ `--cri-socket` |
| Container không resolve được tên container khác (Docker user-defined bridge) | Embedded DNS Docker bind ở `127.0.0.10` trên host netns, netstack bị cô lập không với tới | Dùng default bridge + `--link`, hoặc `--network=host` (giảm bảo mật), hoặc IP thẳng, hoặc chuyển K8s |
| `dial unix .../s/....cff: connect: connection refused` khi dùng `gvisor-containerd-shim` | containerd thiếu fix CVE-2020-15257 | Nâng containerd ≥ 1.3.9 hoặc ≥ 1.4.3 |
| `SELinux is not supported: system_u:system_r:container_t:s0:...` | SELinux enforcing, gVisor không support set SELinux label | `--security-opt label=disable` khi tạo container (biết đánh đổi: mất kiểm soát SELinux cho container đó) |
| `error remounting chroot in read-only: permission denied` | gVisor chạy **lồng trong 1 container khác** trên host có SELinux enforcing, thiếu label đúng | Gán label `container_engine_t` cho container NGOÀI: `--security-opt label=type:container_engine_t` (label dành riêng cho việc chạy container engine trong container) |

## Lesson learned vận hành (không phải lỗi cụ thể, mà là bài học rút ra khi đọc kỹ tài liệu chính thức)

1. **"Latest release" của gVisor đổi liên tục và không theo semver** — đừng bao giờ hard-code 1 version trong runbook/note mà không ghi rõ ngày tham chiếu; luôn có bước "kiểm tra lại GitHub Releases" trước khi áp dụng.
2. **Deadline di trú cài đặt (viết tại 2026-09-06)**: trước 2026-07, gVisor release chỉ gồm 2 binary (`runsc`, `containerd-shim-runsc-v1`), phần còn lại nhúng sẵn trong `runsc`. Đang chuyển sang multi-file (`runsc` + shim + thư mục `gvisor-bin/`). Cơ chế auto-download phần thiếu **sẽ bị gỡ cuối tháng 9/2026**. Nếu tiếp nhận hệ thống cũ, đây là việc cần rà soát ngay đầu tiên, KHÔNG để tự khám phá lúc production down.
3. **"Copy runsc đi đâu cũng phải giữ nguyên cấu trúc thư mục cạnh nó"** — từ bản multi-file, `runsc` tìm `gvisor-bin/` **cạnh chính nó**; di chuyển `runsc` sang path khác mà quên mang theo `gvisor-bin/` → lỗi khó hiểu lúc runtime, không phải lỗi cấu hình logic.
4. **`runsc` binary phải readable+executable cho MỌI user** (`chmod a+rx`), vì chính nó re-exec bản thân với quyền thấp hơn như 1 phần cơ chế bảo mật (drop privilege sau khi setup xong) — không phải chi tiết vặt, là **yêu cầu kiến trúc**.
5. **Không nhầm 2 khái niệm "monitoring" khác nhau**: `runsc metric-server` (Prometheus, quan sát nội bộ gVisor) vs Runtime Monitoring (quan sát hành vi workload, cho threat detection) — dùng nhầm cái này để làm việc của cái kia sẽ không đủ dữ liệu cần thiết. Xem [[gvisor--debugging-observability]].
6. **Node/cụm càng "managed" (GKE Sandbox) thì restriction áp thêm càng nhiều** — đừng lấy giới hạn của gVisor upstream để suy luận cho GKE Sandbox hay ngược lại, chúng là 2 danh sách khác nhau. Xem [[gvisor--kubernetes-containerd]].
7. **CRI-O KHÔNG phải đường đi được gVisor test chính thức** — nếu hạ tầng hiện tại dùng CRI-O + gVisor, coi đây là cấu hình "best-effort", có thể vỡ khi 1 trong 2 project cập nhật, và cần theo dõi sát hơn containerd.

## Refs
- `g3doc/user_guide/compatibility.md`, `g3doc/user_guide/FAQ.md` (repo `google/gvisor`, nhánh `master`)
- `g3doc/user_guide/install.md` (mục di trú cài đặt, deadline tháng 9/2026)
- Bug tracker tham chiếu: gvisor issue #268 (memfd_create), #4 (dentry cache/docker cp), #1765 (memlock kernel bug)
