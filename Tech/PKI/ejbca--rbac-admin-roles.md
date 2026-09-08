# EJBCA — RBAC (Roles & Access Rules), Admin Authentication
Tier: 2
Parent: [[EJBCA]]
Related: [[ejbca--end-entities-ra]], [[pki--hsm-key-protection]]
Tags: #ejbca #rbac #security

## What it does

Hệ thống phân quyền của EJBCA: **Role** (nhóm quyền) + **Access Rule** (quyền cụ thể trên từng resource — CA nào, End Entity Profile nào, chức năng nào) + **Admin** (người/hệ thống được gán vào Role). Đặc biệt: **Admin Web mặc định xác thực bằng client TLS certificate**, không phải username/password.

## Why it exists

CA là hệ thống có quyền lực cao nhất trong toàn bộ hạ tầng bảo mật (nó quyết định ai được tin cậy) — nếu không kiểm soát chặt "ai được làm gì trên EJBCA", việc bảo vệ private key bằng HSM (xem [[pki--hsm-key-protection]]) trở nên vô nghĩa vì attacker chỉ cần chiếm được 1 tài khoản admin quyền cao là điều khiển được CA ký bất cứ thứ gì.

## How it works (flow/diagram)

```
Admin đăng nhập Admin Web
   │
   ▼
Trình duyệt gửi client TLS certificate (Administrator Certificate)
   │
   ▼
EJBCA verify: cert có hợp lệ (chain lên tới CA tin cậy), có match
với 1 Admin đã được gán Role trong hệ thống không
   │
   ▼
Nếu hợp lệ → load Role của admin đó → Access Rule quyết định
admin thấy được menu gì, thao tác được gì (vd chỉ xem, hoặc
tạo End Entity, hoặc full quyền tạo/sửa CA)
```

**Cấu trúc Access Rule điển hình:**
- Theo **resource type**: `/ca/<CA cụ thể>`, `/endentityprofilesrules/<Profile cụ thể>`, `/administrator`, `/ra_functionality/...`
- Theo **hành động**: view / edit / create / approve / revoke...
- **Super Administrator Role** — quyền cao nhất (toàn bộ CA, toàn bộ chức năng) — chỉ nên gán cho rất ít người, dùng cho việc setup ban đầu hoặc khôi phục khẩn cấp, không dùng cho vận hành hàng ngày.

## Config gotchas

- **Gán Super Administrator cho quá nhiều người vì "tiện"** — vi phạm least-privilege nghiêm trọng nhất trong EJBCA; nên tạo Role riêng theo từng nhóm công việc (vd "RA-Operator" chỉ được tạo End Entity theo vài Profile cụ thể, không đụng vào CA config).
- **Administrator Certificate hết hạn không ai để ý** — admin đột nhiên không login được Admin Web, dễ nhầm là lỗi hệ thống trong khi chỉ là cert cá nhân hết hạn — cần theo dõi expiry của chính các cert quản trị này.
- **Quên revoke Administrator Certificate khi nhân viên nghỉ việc** — vì xác thực bằng client cert (không phải password đổi được dễ dàng), quy trình offboarding phải bao gồm bước revoke cert admin, không chỉ khoá tài khoản ở hệ thống khác.
- **Access Rule áp dụng theo nguyên tắc "deny nếu không có rule cho phép rõ ràng"** — khi debug "sao admin X không thấy chức năng Y", thường là thiếu rule cho phép, không phải bug.

## Security notes

- Vì auth bằng client cert, **bảo vệ private key của Administrator Certificate quan trọng ngang với bảo vệ password admin** ở hệ thống khác — không nên để file `.p12` của admin nằm lưu tuỳ tiện trên máy cá nhân không mã hoá.
- Bật audit log chi tiết cho mọi thao tác thuộc Role quyền cao, review định kỳ (không chỉ khi có sự cố).

## Refs

- [[ejbca--end-entities-ra]] — Role/Access Rule áp dụng thực tế lên quyền tạo/duyệt End Entity.
- [[pki--hsm-key-protection]] — RBAC là lớp kiểm soát bổ sung, không thay thế bảo vệ key vật lý qua HSM.
