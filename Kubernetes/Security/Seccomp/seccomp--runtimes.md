# Seccomp — Default Profiles theo Container Runtime
Tier: 2
Parent: [[seccomp]]
Related: [[seccomp--profile-json]], [[seccomp--kubernetes]]
Tags: #runtime #docker #containerd #crio
Last updated: 2026-09-06
Verified against: github.com/moby/profiles (seccomp/default.json), github.com/containerd/containerd (contrib/seccomp), kubernetes.io/blog/2024/03/07/cri-o-seccomp-oci-artifacts

## What it does

`RuntimeDefault` trong K8s **không phải một chuẩn duy nhất** — mỗi container runtime tự maintain 1 file JSON default riêng, dùng chung format OCI ([[seccomp--profile-json]]) nhưng nội dung (số syscall, rule theo capability) khác nhau và có thể đổi giữa các version.

## Why it exists

Mỗi runtime phải tự cân bằng giữa "an toàn" (allowlist càng hẹp càng tốt) và "tương thích" (không được chặn syscall mà workload phổ biến cần, nếu không cả hệ sinh thái image công khai sẽ vỡ hàng loạt). Vì độ đánh đổi này chủ quan và cần bảo trì liên tục theo báo cáo CVE/bug, mỗi project (Docker/moby, containerd, CRI-O) tự giữ profile riêng thay vì phụ thuộc vào 1 bên khác.

## How it works — so sánh các runtime

### Docker / Moby (`moby/profiles` — tách khỏi repo `moby/moby` gần đây)

- File: `seccomp/default.json` trong repo [moby/profiles](https://github.com/moby/profiles).
- `defaultAction: SCMP_ACT_ERRNO`, `defaultErrnoRet: 1` (EPERM) — đúng chuẩn deny-by-default.
- Allowlist ~300+ syscall trong 1 nhóm lớn `SCMP_ACT_ALLOW` không điều kiện, cộng thêm ~20-25 nhóm rule có điều kiện theo capability (`includes.caps`).
- Các nhóm syscall bị khoá sau capability cụ thể (rất nên nhớ khi debug "vì sao container thiếu quyền dù không privileged"):

| Capability | Syscalls bị khoá theo nó |
|---|---|
| `CAP_SYS_ADMIN` | `bpf`, `clone`(một số flag), `clone3`, `mount`, `umount`/`umount2`, `unshare`, `setns`, `fsopen`, `fsmount`, `fsconfig`, `perf_event_open` |
| `CAP_SYS_PTRACE` | `ptrace`, `process_vm_readv`, `process_vm_writev`, `kcmp`, `pidfd_getfd` |
| `CAP_SYS_BOOT` | `reboot` |
| `CAP_SYS_MODULE` | `init_module`, `finit_module`, `delete_module` |
| `CAP_SYS_TIME` | `clock_settime`, `settimeofday` |
| `CAP_SYS_CHROOT` | `chroot` |
| `CAP_SYSLOG` | `syslog` |

- `personality` bị giới hạn chỉ cho phép 1 tập giá trị cụ thể (chặn các personality có thể tắt ASLR).

### containerd

- Code: `contrib/seccomp/` trong repo containerd, viết bằng Go (`seccomp_default.go`) chứ không phải 1 file JSON tĩnh — profile được **build tại runtime** dựa trên logic tương tự Docker/moby (kế thừa/tham khảo từ default.json của moby).
- Dùng tự động cho container không khai gì trừ khi bị `privileged: true` (khi đó containerd set `Unconfined`).
- Vì là code Go thay vì file JSON tĩnh, nội dung chính xác **phụ thuộc version containerd đang chạy trên node** — không có 1 file bạn có thể tải về và coi là "chuẩn mãi mãi".

### CRI-O

- CRI-O tự ship default profile riêng, tinh thần tương tự nhưng maintain độc lập với moby/containerd.
- Cấu hình qua `crio.conf`: `seccomp_profile = "/path/to/profile.json"` để đổi default toàn node.
- **Tính năng đáng chú ý (CRI-O, khoảng 2024+)**: phân phối seccomp profile qua **OCI artifact** — thay vì copy file JSON thủ công ra `/var/lib/kubelet/seccomp/` trên từng node, CRI-O có thể pull profile từ 1 OCI registry giống cách pull image, giải quyết đúng gotcha "node mới thiếu profile" đã nêu ở [[seccomp]] mục 9. Đây là tính năng đang phát triển tại thời điểm viết — kiểm tra version CRI-O đang chạy để biết mức hỗ trợ thực tế trước khi phụ thuộc vào nó trong production.

### runc vs crun (OCI runtime — nơi profile JSON thực sự được compile thành BPF)

- **runc** (Go, mặc định phổ biến nhất): dùng `libseccomp-golang` binding, hỗ trợ đầy đủ action set kể cả `SCMP_ACT_NOTIFY`.
- **crun** (C, dùng trong Podman/CRI-O nhiều hơn): nhẹ hơn, nhanh hơn khi start container, hỗ trợ seccomp qua libseccomp trực tiếp (C). Có 1 số khác biệt nhỏ về feature parity theo thời điểm release — nếu gặp hành vi khác nhau giữa 2 cluster dùng runc vs crun với cùng 1 profile JSON, đây là hướng điều tra đầu tiên.

## Config gotchas

- **Đừng hardcode danh sách syscall của "RuntimeDefault" vào tài liệu nội bộ** — nó sẽ lỗi thời khi bump version runtime. Thay vào đó, luôn có cách tự trích xuất profile đang chạy thật trên node (xem [[seccomp--ops-debug]]).
- Cùng 1 Pod spec (`type: RuntimeDefault`) chạy trên 2 node dùng 2 runtime khác nhau (vd. migrate từ Docker-shim cũ sang containerd, hoặc mix containerd/CRI-O trong cùng cluster) có thể có **hành vi syscall khác nhau** — 1 nguồn lỗi khó debug vì Pod spec giống hệt nhau nhưng 1 node chạy được, 1 node lỗi.
- `crio.conf`'s `seccomp_profile` đổi default cho **toàn bộ node CRI-O đó**, tương tự nhưng độc lập với kubelet's `--seccomp-default` (2 lớp cấu hình khác nhau, dễ nhầm là 1).

## Security notes

- Vì mỗi runtime tự bảo trì allowlist, **CVE mới trong 1 syscall hiếm** có thể được vá (thêm rule chặn) ở runtime này trước runtime kia — 1 lý do nữa để luôn cập nhật container runtime, không chỉ update Kubernetes.
- Khi audit 1 cluster nhận bàn giao: việc đầu tiên nên làm là xác định **runtime + version cụ thể trên từng node** (`kubectl get nodes -o wide` → cột `CONTAINER-RUNTIME`), rồi tự trích default profile thật của runtime đó thay vì tin vào tài liệu chung chung.

## Refs
- [moby/profiles — seccomp/default.json](https://github.com/moby/profiles/blob/main/seccomp/default.json)
- [containerd — contrib/seccomp](https://github.com/containerd/containerd/tree/main/contrib/seccomp)
- [Kubernetes blog — CRI-O: Applying seccomp profiles from OCI registries (2024-03-07)](https://kubernetes.io/blog/2024/03/07/cri-o-seccomp-oci-artifacts/)
