# gVisor — Platforms (systrap / KVM / ptrace)
Tier: 2
Parent: [[gvisor]]
Related: [[gvisor--security-model]], [[gvisor--resource-model]]
Tags: #gvisor #performance #kernel

## What it does

"Platform" là interface nội bộ của gVisor để làm 3 việc: intercept syscall, context switch, quản lý address space cho code chạy trong sandbox. gVisor hỗ trợ 3 implementation: **systrap** (default), **KVM**, **ptrace** (deprecated). Chọn platform không đổi hành vi app, chỉ đổi performance + đôi chút về security surface (khác cơ chế kernel dùng để intercept).

## Why it exists

Không có cách nào "một size fits all" để 1 process userspace (Sentry) bắt được toàn bộ syscall của process khác một cách an toàn và nhanh trên mọi loại máy (bare-metal, VM, VM lồng VM). Mỗi platform đánh đổi khác nhau giữa: có cần hardware virtualization hay không, chạy được trong nested VM hay không, overhead context-switch bao nhiêu.

## How it works (flow/diagram)

- **systrap**: dùng `seccomp`'s `SECCOMP_RET_TRAP`. Khi app gọi syscall bị chặn, kernel gửi `SIGSYS` cho thread đó → Sentry nhận signal, xử lý syscall, trả kết quả lại. **Không cần virtualization**, chạy tốt cả trong VM. Đây là default từ **giữa năm 2023** (thay thế ptrace).
- **KVM**: Sentry đóng vai trò **vừa là guest OS vừa là VMM** thông qua `/dev/kvm`. Không có lớp virtualized hardware (sandbox vẫn là process model), nhưng tận dụng virtualization extension của CPU để tăng tốc address-space switch. Chạy tốt nhất **trên bare-metal**. Chạy được trong nested VM nhưng **chậm hơn systrap** do overhead nested virtualization — và nested virtualization historically có vấn đề bảo mật riêng (vd. CVE-2018-12904), **không khuyến nghị cho production**.
- **ptrace**: dùng `PTRACE_SYSEMU` để chạy code app mà không cho thực thi syscall thật xuống host. Chạy được ở bất kỳ đâu ptrace hoạt động (kể cả VM không nested virtualization) nhưng **overhead context-switch cao nhất**. **Deprecated**, còn tồn tại trong codebase nhưng sẽ bị gỡ, không nên phụ thuộc vào nó cho hệ thống mới.

Quan trọng: "ptrace platform" ở đây **khác** khái niệm "ptrace sandbox" thông thường (loại sandbox chỉ dùng ptrace để authorize/từ chối syscall). gVisor's ptrace platform **không bao giờ để tracee (app) thực thi thẳng xuống host kernel** — mọi syscall vẫn được Sentry diễn giải và xử lý hoàn toàn, giống cơ chế User-Mode Linux (UML), nên tránh được race condition time-of-check/time-of-use mà "ptrace sandbox" cổ điển hay dính.

## Config gotchas

- Chọn platform qua `--platform=systrap|kvm|ptrace` trong `runtimeArgs`.
- **Quy tắc chọn nhanh**: bare-metal → `kvm`. Trong VM (kể cả cloud VM thông thường) → `systrap` (default, không cần đổi gì). Cần KVM trong VM → phải bật nested virtualization trước (`/dev/kvm` phải tồn tại và user thuộc group `kvm`) — nhưng nên tránh, vì lại chậm hơn systrap trong trường hợp này.
- Có thể khai báo **nhiều runtime** cùng lúc trong Docker daemon.json (vd. `runsc-kvm`, `runsc-systrap`) để so sánh A/B trên cùng máy.
- GKE Sandbox **không dùng 1 trong 3 platform chuẩn này theo lựa chọn thủ công** — dùng "an optimized, custom platform" theo tài liệu chính thức, không cần tune platform khi dùng GKE Sandbox.
- Benchmark chính thức trong performance guide chủ yếu chạy trên `ptrace` (baseline tệ nhất, để thấy rõ structural cost) — **đừng lấy số liệu đó làm đại diện cho hiệu năng thực tế nếu bạn dùng systrap/KVM**.

## Security notes

- Cả 3 platform đều giữ nguyên mô hình bảo mật cốt lõi (Sentry không pass-through syscall), nhưng **khác nhau về kernel functionality mà chúng dựa vào** để intercept — nghĩa là bug trong `seccomp-bpf` ảnh hưởng systrap khác với bug trong KVM subsystem ảnh hưởng KVM platform. Không coi 2 platform là "tương đương tuyệt đối" về mặt threat surface, dù chúng transparent với sysadmin.
- Nested virtualization (để chạy KVM platform trong VM) có lịch sử là nguồn gây lỗi bảo mật (CVE-2018-12904) — tránh dùng ở production trừ khi thật sự cần thiết.

## Refs
- `g3doc/architecture_guide/platforms.md`, `g3doc/user_guide/platforms.md` (repo `google/gvisor`, nhánh `master`)
- Blog announcement Systrap: https://gvisor.dev/blog/2023/04/28/systrap-release/
- systrap README (chi tiết implementation): `pkg/sentry/platform/systrap/README.md` trong repo
