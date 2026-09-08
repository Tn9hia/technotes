# SELinux — Contexts & File Labeling
Tier: 2
Parent: [[selinux]]
Related: [[selinux--booleans]], [[selinux--troubleshooting]], [[selinux--modes-and-relabeling]]
Tags: #selinux #labeling

## What it does

Gắn nhãn `user:role:type:level` lên mọi process và mọi object (file, dir, socket, port...). Đây là dữ liệu mà kernel dùng để ra quyết định allow/deny — không có label đúng thì không có rule nào match được.

```
system_u:object_r:httpd_sys_content_t:s0   ← 1 file trong /var/www/html
system_u:system_r:httpd_t:s0               ← process httpd đang chạy
```

Xem context: `ls -Z`, `ps -Z`, `id -Z` (context của session hiện tại).

## Why it exists

Type Enforcement cần một "chìa khoá" gắn cố định lên object để so khớp rule, độc lập với path/tên file. Nhờ vậy, policy không cần biết `/var/www/html` nằm ở đâu — nó chỉ cần biết file đó mang type `httpd_sys_content_t`. Điều này cũng có nghĩa: **label đi theo inode/xattr, không đi theo path**. Đổi path không tự đổi type.

## How it works (flow/diagram)

Có 2 tầng cấu hình dễ nhầm với nhau:

```
semanage fcontext -a -t <type> "path_pattern"
        │  (chỉ ghi vào DB policy: /etc/selinux/targeted/contexts/files/file_contexts.local)
        │  KHÔNG đổi label của file đang tồn tại
        ▼
restorecon -Rv <path>
        │  đọc DB policy ở trên, áp label thực tế lên file/dir đang tồn tại
        ▼
   File thực sự đổi context (xem lại bằng ls -Z)
```

So sánh nhanh 3 lệnh hay bị lẫn:

| Lệnh | Persistent qua relabel? | Dùng khi nào |
|---|---|---|
| `chcon -t type_t path` | ❌ Không | Test nhanh, tạm thời |
| `semanage fcontext -a -t type_t "pattern"` + `restorecon` | ✅ Có | Chuẩn production, custom path lâu dài |
| `restorecon -Rv path` một mình | — | Đưa file về ĐÚNG theo rule đã khai báo sẵn (mặc định hoặc đã `semanage fcontext -a`) |

Regex pattern hay dùng: `"/data/website(/.*)?"` — bắt cả thư mục gốc lẫn mọi thứ bên trong.

Context equivalence (map cả cây thư mục theo context của cây khác):
```bash
semanage fcontext -a -e /var/www /var/test_www
restorecon -Rv /var/test_www
```

Kiểm tra context "đáng lẽ phải là gì" theo policy hiện tại mà không cần đổi gì:
```bash
matchpathcon /var/www/html/index.html
```

## Config gotchas

- `semanage fcontext -a` **không** áp dụng ngay — quên `restorecon` là lỗi phổ biến số 1 khi mới học.
- `cp` (không có `--preserve=context`/`-a`) label file mới theo **thư mục đích**; `mv` giữ nguyên label **cũ** của file. Hai hành vi ngược nhau, dễ gây bug "tưởng move xong là xong".
- Sau khi restore/backup dữ liệu (tar, rsync không giữ xattr) → file thường mất label đúng, phải `restorecon -R` lại toàn bộ sau khi restore.
- `restorecon` không báo gì nếu context **đã đúng sẵn** — đừng hoảng khi thấy output trống, đó là bình thường. Dùng `-v` để thấy dòng log khi có thay đổi thật.
- `-F` flag của `restorecon` reset cả 4 field (user/role/type/level) — cần khi label bị sai cả user/level, không chỉ type.

## Security notes

- Rule chỉ mạnh bằng label đúng — một thư mục dữ liệu nhạy cảm bị label nhầm thành type lỏng lẻo (`var_t`, `unlabeled_t`) coi như vô hiệu hoá phần lớn bảo vệ TE cho nó.
- `unlabeled_t` là dấu hiệu cảnh báo: object chưa từng được label đúng (thường do tạo trên filesystem không hỗ trợ xattr rồi mount sang chỗ khác, hoặc SELinux từng bị Disabled lúc file được tạo).

## Refs

- [Chapter 4 — Configuring SELinux for applications and services (RHEL 9)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/configuring-selinux-for-applications-and-services-with-non-standard-configurations_using-selinux)
