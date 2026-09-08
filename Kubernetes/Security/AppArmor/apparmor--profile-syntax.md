# AppArmor — Profile Syntax
Tier: 2
Parent: [[AppArmor]]
Related: [[apparmor--tooling-workflow]], [[apparmor--debugging-runbook]]
Tags: #apparmor #syntax

## What it does
Cú pháp file profile: rule cho file access, network, capability, execute transition, include/abstraction, variable.

## Why it exists
Cần đọc/sửa profile bằng tay khi `aa-logprof` gợi ý sai, hoặc viết profile tối giản trong tình huống không có sẵn tool generate (vd phòng thi CKS).

## How it works (flow/diagram)

### Cấu trúc cơ bản
```
#include <tunables/global>

/usr/sbin/nginx {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  capability net_bind_service,
  capability setuid,
  capability setgid,

  network inet stream,
  network inet6 stream,

  /etc/nginx/** r,
  /var/log/nginx/*.log w,
  /var/run/nginx.pid rw,
  /usr/sbin/nginx mr,

  /usr/bin/dash Cx -> nginx_dash,

  deny /etc/shadow r,
  audit deny /root/** rwx,
}
```

### File permission chars
| Char | Ý nghĩa |
|---|---|
| r | read |
| w | write |
| a | append-only (loại trừ w) |
| l | tạo hard link |
| k | file locking |
| m | mmap với PROT_EXEC (cần cho shared lib / JIT) |
| ix | execute, **kế thừa profile hiện tại** |
| px | execute, chuyển sang profile riêng của binary đích (theo tên) |
| Px | như px nhưng scrub môi trường (bỏ biến env nhạy cảm — an toàn hơn) |
| cx | execute, chuyển sang **child profile** khai báo cùng file (`-> label`) |
| Cx | như cx + scrub env |
| ux | execute **unconfined** — mất toàn bộ protection từ điểm này, hạn chế dùng |
| Ux | như ux + scrub env |

### Owner-conditional rule
```
owner /home/*/.ssh/** r,     # chỉ áp dụng khi file thuộc UID đang chạy process
```
Hữu ích cho service multi-user (Samba, sshd) thay vì liệt kê path tuyệt đối từng user.

### Variables / Tunables
```
@{HOME}=/home/*/ /root/

/usr/bin/foo {
  @{HOME}/.foorc r,
}
```
Định nghĩa 1 lần trong `tunables/`, include lại ở nhiều profile — đổi 1 chỗ, áp dụng toàn bộ.

### Abstractions
File có sẵn trong `/etc/apparmor.d/abstractions/` (`base`, `nameservice`, `openssl`, `python`, ...) — bundle rule dùng chung (resolve DNS, load shared lib chuẩn) để không phải viết lại mỗi profile. `#include <abstractions/base>` gần như bắt buộc ở mọi profile.

### Network rule (AppArmor 3.x)
```
network inet stream,          # cho phép mọi TCP IPv4
```
Network mediation theo **family/type**, KHÔNG lọc theo port/IP như iptables/NetworkPolicy — đừng kỳ vọng AppArmor thay thế firewall hay NetworkPolicy.

## Config gotchas
- Thứ tự rule **không quan trọng** trong 1 profile (khác iptables) — nhưng `deny` luôn thắng nếu conflict tuyệt đối trên cùng path/permission.
- `ix` trên binary đích SUID/SGID có thể fail vì kernel chặn transition giữ nguyên profile cha khi privilege đổi — thường phải dùng `px` cho SUID binary.
- Comment dùng `#`, nhưng mỗi rule vẫn phải kết thúc bằng dấu `,` — thiếu dấu phẩy là lỗi parse khó thấy bằng mắt.
- `m` permission (mmap exec) hay bị quên khi confine app dùng shared library động hoặc runtime có JIT (Java, Node với 1 số flag) → lỗi "Permission denied" mơ hồ, không liên quan trực tiếp tới thao tác file đang debug.

## Security notes
- Rule càng cụ thể càng tốt; tránh `/** rw` ở profile top-level — gần như vô hiệu hoá MAC.
- Luôn thêm `audit deny` cho path nhạy cảm dù đã có rule khác che phủ — log rõ ràng khi có exploit attempt cụ thể, tiện forensics sau này.
- `Px`/`Cx` (scrub env) nên ưu tiên hơn `px`/`cx` khi confine binary nhận input từ network — giảm nguy cơ env-based injection (LD_PRELOAD, PATH poisoning) xuyên qua exec chain.

## Refs
- `man apparmor.d`
- `/etc/apparmor.d/abstractions/` trên máy đã cài apparmor-profiles — đọc ví dụ thật
