# Seccomp — Kernel Mechanics (seccomp-BPF)
Tier: 2
Parent: [[seccomp]]
Related: [[seccomp--profile-json]], [[seccomp--runtimes]]
Tags: #kernel #bpf
Last updated: 2026-09-06
Verified against: man7.org seccomp(2) — nguồn chuẩn nhất cho phần này (man-pages luôn theo sát kernel upstream)

## What it does

Đây là cơ chế kernel-level thực sự đứng sau mọi profile JSON bạn viết. Khi 1 profile được "áp" vào container, cuối cùng nó được libseccomp compile thành 1 chương trình **classic BPF (cBPF)**, và chương trình này chạy trong kernel mỗi lần process gọi syscall, nhận input là `struct seccomp_data { nr, arch, instruction_pointer, args[6] }`.

## Why it exists

Trước seccomp-BPF (kernel < 3.5), seccomp chỉ có "strict mode": literally chỉ cho `read/write/exit/sigreturn`, dùng cho sandbox tính toán thuần (kiểu SETI@home). Không dùng được cho ứng dụng thực tế. Seccomp-BPF ra đời để cho phép **allowlist/denylist theo từng syscall + argument**, đủ linh hoạt cho container/sandbox (Chrome renderer sandbox là use case gốc thúc đẩy tính năng này).

## How it works (flow/diagram)

### 2 mode của syscall `seccomp()` (từ Linux 3.17, trước đó dùng `prctl(PR_SET_SECCOMP,...)`)

| Mode | Hằng số | Hành vi | Kernel version |
|---|---|---|---|
| Strict | `SECCOMP_SET_MODE_STRICT` | Chỉ `read/write/_exit/sigreturn`, syscall khác → kill thread ngay | 2.6.12 (qua prctl) |
| Filter | `SECCOMP_SET_MODE_FILTER` | Chạy chương trình BPF do userspace nạp vào | 3.5 (qua prctl), 3.17 (qua seccomp() syscall) |

Điều kiện để 1 process **không có** `CAP_SYS_ADMIN` được nạp filter: phải set `no_new_privs` trước (`prctl(PR_SET_NO_NEW_PRIVS, 1)`) — đây là lý do container runtime luôn set `no_new_privs` trước khi seccomp filter, và cũng là lý do 1 số ứng dụng setuid bị vỡ trong container hardening (setuid bit vô hiệu khi `no_new_privs=1`).

### Filter stacking & precedence

Nhiều filter có thể được attach chồng lên nhau trong đời process (mỗi lần gọi `seccomp(SECCOMP_SET_MODE_FILTER,...)` thêm 1 filter mới vào chain, filter **không thể gỡ bỏ**, chỉ có thể thêm). Khi 1 syscall được gọi, **toàn bộ chain filter được chạy**, và **action nghiêm khắc nhất trong toàn bộ kết quả thắng** — không phải filter mới nhất, không phải filter cũ nhất.

Thứ tự ưu tiên (cao → thấp), theo man7 seccomp(2):

```
SECCOMP_RET_KILL_PROCESS   (Linux 4.14+)  – kill toàn bộ process, có core dump
SECCOMP_RET_KILL_THREAD    (≈ SECCOMP_RET_KILL, Linux 3.5+) – kill riêng thread gọi
SECCOMP_RET_TRAP                          – gửi SIGSYS cho thread
SECCOMP_RET_ERRNO                         – trả errno chỉ định, KHÔNG thực thi syscall
SECCOMP_RET_USER_NOTIF     (Linux 5.0+)   – forward cho 1 supervisor userspace quyết định
SECCOMP_RET_TRACE                         – báo cho ptrace() tracer
SECCOMP_RET_LOG            (Linux 4.14+)  – log rồi CHO PHÉP chạy
SECCOMP_RET_ALLOW                         – cho chạy bình thường
```

Ý nghĩa thực tế: nếu app tự thêm 1 filter permissive sau khi container runtime đã gắn filter restrictive, **filter của app không thể nới lỏng** filter gốc — chỉ có thể siết chặt thêm. Đây là thuộc tính an toàn cố ý (defense không thể tự gỡ bởi process bị compromise).

### Flags quan trọng của `seccomp()` syscall

| Flag | Kernel | Ý nghĩa |
|---|---|---|
| `SECCOMP_FILTER_FLAG_TSYNC` | sớm | Đồng bộ filter cho **mọi thread** trong process cùng lúc (không TSYNC thì mỗi thread phải tự gọi) |
| `SECCOMP_FILTER_FLAG_LOG` | 4.14 | Ép mọi action không-ALLOW của filter này đều được log (audit), kể cả không phải `SCMP_ACT_LOG` |
| `SECCOMP_FILTER_FLAG_SPEC_ALLOW` | 4.17 | Tắt mitigation Spectre v4 (Speculative Store Bypass) cho process — **cân nhắc kỹ về security trước khi dùng**, đánh đổi hiệu năng lấy an toàn |
| `SECCOMP_FILTER_FLAG_NEW_LISTENER` | 5.0 | Trả về 1 file descriptor để nhận `SECCOMP_RET_USER_NOTIF` — nền tảng cho unotify (bên dưới) |

### Seccomp User-Space Notification (unotify) — cơ chế nâng cao (Linux 5.0+)

Thay vì kernel tự quyết ALLOW/ERRNO/KILL, `SECCOMP_RET_USER_NOTIF` forward syscall cho 1 **process giám sát ở userspace** quyết định thay. Luồng: filter trả `USER_NOTIF` → kernel block thread gọi syscall → supervisor đọc qua `ioctl(SECCOMP_IOCTL_NOTIF_RECV)` → supervisor quyết định (allow giả lập kết quả, deny, hoặc tự thực thi hộ) → trả lời qua `ioctl(SECCOMP_IOCTL_NOTIF_SEND)`.

Use case thực tế: rootless container runtime cần "giả lập" `mount()`/`chown()` thành công cho process con trong user namespace mà không cấp quyền thật (Podman/Docker rootless dùng kỹ thuật này qua `runc`'s seccomp-notify với 1 helper OCI hook chạy quyền cao hơn để proxy syscall). `SECCOMP_ADDFD` (thêm từ Linux 5.9) cho phép supervisor "tiêm" file descriptor vào process bị chặn — hữu ích khi giả lập syscall trả về fd (như `open()`).

**Lưu ý support**: `SCMP_ACT_NOTIFY` không phải mọi container runtime/CRI đều expose ra tận Kubernetes API — đây chủ yếu là cơ chế cấp OCI-runtime/hook, không phải thứ bạn set trực tiếp qua `seccompProfile.type` trong Pod spec. Nếu cần dùng, tự kiểm tra tài liệu runc/crun/CRI-O phiên bản đang chạy.

## Config gotchas

- **Filter không thể gỡ, chỉ thêm** — nếu 1 chain filter lỗi nhẹ (thiếu 1 syscall), cách sửa không phải "detach và attach lại" mà phải **restart process** để filter mới (đã sửa) được gắn từ đầu.
- **`no_new_privs` phá setuid**: nếu image có binary setuid-root (vd. `ping`, `sudo`) và bạn tự thêm layer seccomp/no_new_privs bên trên container runtime (ví dụ dùng `unshare --no-new-privs` thủ công), binary đó mất hiệu lực setuid — dễ nhầm là "seccomp chặn" trong khi thực ra là no_new_privs.
- **32-bit / 64-bit args**: `seccomp_data.args[]` là `uint64_t` nhưng BPF cổ điển chỉ load được 32-bit 1 lần → phải load riêng high/low word. Đây là lý do bạn **không tự viết BPF tay** mà luôn dùng libseccomp (nó lo phần này), tự viết dễ sai leading tới bypass.
- **Kiểm tra `arch` field bắt buộc**: process không check `arch` trong filter tự viết có thể bị bypass qua syscall gọi từ ABI khác (32-bit compat trên máy 64-bit có syscall-number-table khác hẳn). libseccomp xử lý qua field `"architectures"` trong profile JSON.

## Security notes

- Seccomp bảo vệ **theo syscall number + argument value**, không hiểu semantic (không biết `open()` đang mở file gì nếu bạn không so khớp chuỗi path — mà seccomp filter kiểm path string là bất khả thi vì chỉ nhận con trỏ, không đọc nội dung bộ nhớ tại thời điểm filter chạy → **TOCTOU by design**, đừng cố dùng seccomp để chặn theo path).
- `SPEC_ALLOW` flag là 1 trade-off bảo mật thực sự (tắt Spectre mitigation) — không nên bật trừ khi có lý do performance rõ ràng và đã risk-assess.

## Refs
- [seccomp(2) — man7.org](https://man7.org/linux/man-pages/man2/seccomp.2.html)
- [prctl(2) — PR_SET_NO_NEW_PRIVS](https://man7.org/linux/man-pages/man2/prctl.2.html)
