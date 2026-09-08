# AppArmor — Tooling & Workflow
Tier: 2
Parent: [[AppArmor]]
Related: [[apparmor--profile-syntax]], [[apparmor--debugging-runbook]]
Tags: #apparmor #cli #workflow

## What it does
Bộ CLI để tạo, học (learn), chuyển mode, và quản lý vòng đời profile mà không cần viết tay từ đầu.

## Why it exists
Viết profile tay cho app phức tạp (nginx, php-fpm, custom service) rất dễ thiếu rule → app lỗi liên tục khi enforce. Bộ `aa-*` tools tự động hoá quy trình "chạy thử → xem log → generate rule → duyệt".

## How it works (flow)

```
 1. aa-genprof <bin>      → tạo profile rỗng, đặt complain mode, chờ user thao tác app thật
 2. (thao tác mọi tính năng, mọi flow của app) → kernel log mọi hành vi bị chặn dù đang complain
 3. aa-logprof             → đọc log, hỏi Allow/Deny/Glob cho từng dòng, ghi vào profile
 4. lặp lại bước 2-3 tới khi hết sự kiện lạ phát sinh
 5. aa-enforce <profile>   → chuyển sang production mode
 6. theo dõi log DENIED thêm 1 thời gian → phát hiện case chưa cover hết
```

### Bảng lệnh cốt lõi
| Lệnh | Việc làm |
|---|---|
| `aa-status` | Liệt kê toàn bộ profile + mode hiện tại (enforce/complain/unconfined), số process đang bị confine |
| `aa-enforce <path\|profile>` | Chuyển 1 profile sang enforce |
| `aa-complain <path\|profile>` | Chuyển 1 profile sang complain — an toàn để rollback nhanh khi enforce làm app die |
| `aa-disable <path\|profile>` | Gỡ hẳn profile khỏi kernel (unload) — khác complain: complain vẫn load, chỉ không chặn |
| `aa-genprof <bin>` | Sinh profile mới, interactive — dùng lần đầu confine 1 binary |
| `aa-logprof` | Cập nhật profile đang có dựa trên log denial mới — dùng định kỳ, không chỉ lúc tạo mới |
| `aa-easyprof <bin>` | Sinh template profile nhanh, ít câu hỏi hơn genprof — dùng khi đã biết rõ mình cần gì |
| `aa-unconfined` | Liệt kê process đang listen network nhưng KHÔNG có profile nào — audit "cái gì đang chạy trần" |
| `aa-notify` | Desktop notification khi có DENIED — chủ yếu dùng desktop, ít dùng server |
| `apparmor_parser -r <file>` | Reload 1 profile file vào kernel sau khi sửa tay — bắt buộc sau mọi edit thủ công |
| `apparmor_parser -R <file>` | Remove 1 profile khỏi kernel |

## Config gotchas
- `aa-logprof` hỏi rất nhiều, dễ bấm nhầm "Allow" cho action không nên allow khi vội — luôn review file profile bằng mắt sau khi chạy xong, đừng tin tuyệt đối wizard.
- `aa-disable` **không xoá file profile**, chỉ unload khỏi kernel — dễ hiểu nhầm "đã tắt AppArmor cho service" trong khi file vẫn còn, có thể bị load lại nhầm sau reboot nếu có script tự load toàn bộ `/etc/apparmor.d/`.
- Sau `aa-genprof`, mặc định là **complain mode** — hay quên bước cuối `aa-enforce`, để mãi complain = false sense of security.
- `aa-status` cần quyền root để đọc đầy đủ thông tin.

## Security notes
- Chạy `aa-unconfined` định kỳ trên production để phát hiện service mới deploy quên chưa có profile (mặc định unconfined hoàn toàn, không phải deny-all — xem [[AppArmor]]).
- Không chạy `aa-genprof`/`aa-logprof` trực tiếp trên production đang phục vụ traffic thật nếu chưa quen — quá trình duyệt tương tác có thể để app ở complain lâu hơn dự kiến, mất bảo vệ thời gian dài mà không để ý.

## Refs
- `man aa-genprof`, `man aa-logprof`, `man aa-easyprof`
- `man apparmor_parser`
