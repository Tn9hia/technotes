# SELinux — Custom Policy Modules
Tier: 2
Parent: [[selinux]]
Related: [[selinux--troubleshooting]], [[selinux--containers-kubernetes]]
Tags: #selinux #policy #module

## What it does

Đóng gói thêm rule TE mới (allow/dontaudit...) thành 1 module `.pp` (policy package) nạp thêm vào policy đang chạy, mà không cần biên dịch lại toàn bộ base policy.

## Why it exists

Base policy (`selinux-policy-targeted`) không thể phủ hết mọi phần mềm 3rd-party/tự viết. Module system cho phép mở rộng policy theo kiểu "thêm mảnh ghép", có thể cài/gỡ độc lập (`semodule -i`/`-r`), không đụng vào base policy gốc — dễ rollback hơn sửa trực tiếp policy nguồn.

## How it works (flow/diagram)

**Luồng chuẩn (từ denial → module, xem thêm [[selinux--troubleshooting]] cho bước trước đó):**
```bash
ausearch -m avc -ts recent | audit2allow -M mymodule   # sinh mymodule.te + mymodule.pp
cat mymodule.te                                         # BẮT BUỘC đọc trước khi nạp
semodule -i mymodule.pp                                 # nạp module đã compile sẵn
```

**Quản lý module đang có:**
```bash
semodule -l                 # liệt kê module + version đang active
semodule -r mymodule        # gỡ module
```

**Viết policy module thủ công (khi cần chính xác hơn audit2allow tự sinh):**
```bash
checkmodule -M -m -o mymodule.mod mymodule.te
semodule_package -o mymodule.pp -m mymodule.mod
semodule -i mymodule.pp
```

## Config gotchas

- Module tự sinh bởi `audit2allow` thường **quá rộng** — nó gộp permission theo pattern thấy trong log, không tối ưu theo nguyên tắc least-privilege. Sau khi review, nên tự tay cắt bớt permission không cần thiết trong file `.te` trước khi compile.
- `semodule -i` không tự động phát hiện version cũ hơn đã load — nếu cần update module cùng tên, `semodule -r` module cũ trước hoặc dùng đúng tên mới, tránh xung đột rule trùng lặp khó debug.
- Priority của module ảnh hưởng thứ tự override — module custom mặc định nạp ở priority 400, base policy modules ở priority thấp hơn; nếu 2 module custom xung đột nhau trên cùng rule, cần hiểu rõ cơ chế priority trước khi debug "tại sao rule tôi viết không có tác dụng".

## Security notes

- Đây là nơi rủi ro cao nhất trong toàn bộ vòng đời SELinux: một module viết/nạp ẩu = lỗ hổng cấp policy, tồn tại vĩnh viễn cho tới khi bị phát hiện và gỡ.
- Luôn version-control các file `.te` custom (Git) kèm comment lý do — không lưu chỉ dưới dạng `.pp` binary đã compile, vì `.pp` không đọc lại được thành rule con người hiểu ngay.
- Định kỳ audit toàn bộ `semodule -l` trên production, đối chiếu với danh sách đã version-control — module lạ xuất hiện ngoài danh sách là tín hiệu cần điều tra ngay.

## Refs

- [Chapter 5 — Troubleshooting problems related to SELinux (RHEL 9), phần audit2allow](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/troubleshooting-problems-related-to-selinux_using-selinux)
