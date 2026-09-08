# SELinux — Systemd Services & Confined Users
Tier: 2
Parent: [[selinux]]
Related: [[selinux--contexts-and-labeling]], [[selinux--port-network-labeling]]
Tags: #selinux #systemd #users

## What it does

Quản lý 2 việc liên quan tới vận hành hàng ngày: (1) đảm bảo service/binary tự viết hoặc cài từ ngoài repo được chạy đúng domain SELinux thay vì rơi vào `unconfined_service_t`; (2) map user Linux đăng nhập vào đúng SELinux user (confined/unconfined) để giới hạn quyền theo phiên làm việc.

## Why it exists

Một service khởi động qua systemd nhưng binary/label không khớp domain nào trong policy sẽ chạy dưới `unconfined_service_t` — về mặt SELinux gần như "miễn nhiễm" MAC, quay lại đúng vấn đề mà SELinux sinh ra để giải quyết. Việc gán đúng domain cho custom service, và đúng SELinux user cho người vận hành, là bước hoàn thiện vòng đời "triển khai an toàn" thay vì chỉ dựa vào policy có sẵn cho service chuẩn (httpd, sshd...).

## How it works (flow/diagram)

**Kiểm tra domain 1 service đang chạy dưới:**
```bash
systemctl status myservice     # cột Context/CGroup show context nếu ps -Z không tiện
ps -eZ | grep myservice
```

**Nếu binary tự build/cài custom, gán label đúng trước khi start:**
```bash
semanage fcontext -a -t bin_t "/opt/myapp/bin/myapp"    # hoặc type domain phù hợp nếu có policy riêng
restorecon -v /opt/myapp/bin/myapp
```
(Nếu không có policy riêng cho app, nó vẫn chạy trong domain cha gọi nó — thường là `init_t`/`unconfined_service_t` tuỳ ngữ cảnh; muốn confine chặt hơn cần viết policy module riêng, xem [[selinux--policy-modules]].)

**Mapping Linux user ↔ SELinux user:**
```bash
semanage login -l                              # xem mapping hiện tại
semanage login -a -s staff_u -r s0 alice        # map user 'alice' vào SELinux user staff_u
semanage login -m -s user_u -r s0 __default__   # đổi default mapping cho user không khai báo riêng
```

**Các SELinux user thường gặp (targeted policy):**

| SELinux user | Đặc điểm |
|---|---|
| `unconfined_u` | Default cho hầu hết user trên RHEL 9 kể cả admin — gần như không giới hạn thêm ngoài DAC |
| `staff_u` | Non-admin, có thể `sudo`/transition lên `sysadm_r` khi cần quyền cao hơn — mô hình "least privilege by default, escalate có kiểm soát" |
| `sysadm_u` | Quyền quản trị rộng; mặc định **không SSH được** — cần bật boolean `ssh_sysadm_login` mới cho phép |
| `user_u` / `guest_u` | Hạn chế mạnh — `guest_u` không có quyền network, giới hạn thực thi trong home + /tmp |

Khi user login, `pam_selinux` tự gán context theo mapping — context này theo suốt session và mọi process con (kể cả service user tự khởi chạy qua `systemd --user`).

## Config gotchas

- Đổi `semanage login` **không ảnh hưởng session đang mở** — user phải logout/login lại (hoặc `newrole`) để nhận context mới.
- `sysadm_u` không SSH được là hành vi **cố ý** (giảm bề mặt tấn công cho tài khoản quyền cao qua network) — đừng tưởng là bug rồi vội bật boolean cho toàn hệ thống; cân nhắc dùng `staff_u` + `sudo` + transition thay vì SSH thẳng bằng `sysadm_u`.
- Custom service không có `.te` riêng vẫn hoạt động bình thường (không bị lỗi start) nhưng **không nhận được lợi ích cô lập MAC** — dễ bị bỏ sót vì "service chạy tốt" không đồng nghĩa "service đã được confine đúng".

## Security notes

- Với hệ thống nhận bàn giao: luôn `semanage login -l` để biết ai đang map vào SELinux user nào — đây là thông tin bàn giao quan trọng, đặc biệt nếu có tài khoản `sysadm_u`/`unconfined_u` không rõ mục đích.
- Custom service chạy quyền cao (root) mà không có policy domain riêng nên nằm trong danh sách ưu tiên viết policy module (qua `udica`-style workflow hoặc `audit2allow` có review) khi có thời gian hardening thêm.

## Refs

- [Chapter 3 — Managing confined and unconfined users (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/managing-confined-and-unconfined-users_using-selinux)
