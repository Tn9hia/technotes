# AppArmor — Debugging & Runbook
Tier: 2
Parent: [[AppArmor]]
Related: [[apparmor--tooling-workflow]], [[apparmor--kubernetes-cks]]
Tags: #apparmor #debug #ops

## What it does
Quy trình tra log, đọc thông báo DENIED, phân biệt lỗi do AppArmor vs lỗi app thật, và các bước xử lý sự cố khi 1 profile mới gây down service.

## Why it exists
Triệu chứng phổ biến nhất khi mới bật `enforce`: app báo "Permission denied" dù `chmod`/`chown` hoàn toàn đúng — nếu không biết tra log AppArmor sẽ tốn hàng giờ debug sai hướng (nghĩ do code/app/DAC).

## How it works (flow chẩn đoán)

```
App lỗi "Permission denied" nhưng chmod/chown check OK
        │
        ▼
grep log AppArmor (xem bảng bên dưới) → có dòng apparmor="DENIED" trùng thời điểm không?
        │
   ┌────┴────┐
  Có          Không
   │            │
   ▼            ▼
Đúng do      Lỗi thật ở
AppArmor     tầng khác (DAC,
profile      SELinux, code,...)
   │
   ▼
Đọc field: operation / name / requested_mask / profile
   │
   ▼
Fix: thêm rule vào profile → apparmor_parser -r → test lại
(hoặc tạm aa-complain nếu cần app sống ngay, sửa profile sau)
```

### Lệnh tra log theo hệ thống
| Hệ thống | Lệnh |
|---|---|
| Có auditd | `sudo ausearch -m AVC,USER_AVC -ts recent` hoặc `sudo aureport -a` |
| systemd, không auditd | `journalctl -k --since "-10min" \| grep -i apparmor` |
| Debian/Ubuntu classic | `grep -i apparmor /var/log/kern.log` hoặc `/var/log/syslog` |
| Realtime khi đang test | `sudo tail -f /var/log/kern.log \| grep --line-buffered apparmor` |
| Kernel ring buffer | `dmesg \| grep -i apparmor` |

### Đọc 1 dòng log DENIED
```
apparmor="DENIED" operation="open" profile="usr.sbin.nginx"
name="/etc/nginx/secret.conf" pid=1234 comm="nginx"
requested_mask="r" denied_mask="r" fsuid=33 ouid=33
```
- `operation` — loại syscall (open/exec/mkdir/connect/ptrace/signal...)
- `profile` — profile nào đang chặn (tên khai báo bên trong file, không phải tên file)
- `name` — path bị chặn
- `requested_mask`/`denied_mask` — quyền app xin vs quyền bị từ chối (so sánh để biết thiếu permission nào)
- `comm` — tên process (hữu ích khi 1 profile áp cho nhiều binary qua `cx`/`px`)

## Config gotchas
- `deny` rule mặc định **không log** — nếu tìm mãi không thấy DENIED nhưng vẫn bị chặn, khả năng cao có `deny` rule tường minh; thêm tạm `audit deny <rule>` để thấy log, debug xong bỏ `audit` lại.
- Log AppArmor và log SELinux (AVC) dùng chung style field (`ausearch` đọc được cả 2 nếu hệ thống có cả — hiếm) → đọc kỹ `profile=`/`scontext=` để biết đang debug đúng LSM nào.
- Container: log DENIED của process trong container xuất hiện ở **log của node/host**, KHÔNG nằm trong log container (`kubectl logs` sẽ không thấy) — phải ssh/`kubectl debug node/<name>` rồi grep trên node.
- Sau khi sửa profile và `apparmor_parser -r`, process **đang chạy từ trước** đôi khi không tự áp thay đổi lớn về rule mapping — an toàn nhất là restart process/container sau khi sửa profile đáng kể.

## Security notes
- Spike đột biến DENIED cho path nhạy cảm (`/etc/shadow`, `~/.ssh/`, `docker.sock`) từ 1 profile bình thường không đụng path đó → dấu hiệu exploit attempt, nên có alert riêng thay vì gộp chung noise log.
- Giữ log DENIED lịch sử (ship về SIEM/Loki) tối thiểu vài tuần — hữu ích để review coverage khi viết profile mới và để forensics khi có sự cố.

## Refs
- `man audit.log`, `man ausearch`
- Ubuntu Server Guide — mục "Troubleshooting AppArmor"
