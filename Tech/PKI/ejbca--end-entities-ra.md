# EJBCA — End Entities & RA Workflow
Tier: 2
Parent: [[EJBCA]]
Related: [[ejbca--profiles]], [[pki--csr-enrollment]], [[ejbca--rbac-admin-roles]]
Tags: #ejbca #end-entity #ra

## What it does

**End Entity** là đơn vị "chủ thể được cấp cert" trong EJBCA — mỗi server, user, thiết bị cần cert phải có 1 record End Entity tương ứng trước khi enrollment thành công. RA (Registration Authority) workflow là quy trình tạo/duyệt End Entity đó, thực hiện qua **Admin Web** (đầy đủ quyền) hoặc **RA Web** (giao diện gọn hơn dành riêng cho người làm nhiệm vụ đăng ký).

## Why it exists

CSR/public key thuần không tự nói lên "ai đứng sau request này có được phép xin cert loại này không". End Entity là nơi buộc chặt: danh tính dự kiến (username, DN, SAN...) + quyền hạn (End Entity Profile nào áp dụng, CA nào được dùng) **trước khi** thao tác enrollment thực sự diễn ra — tách rời bước "xác minh & cấp quyền" khỏi bước "sinh key & lấy cert".

## How it works (flow/diagram)

```
1. Admin/RA tạo End Entity mới
   - Chọn End Entity Profile (quyết định field nào phải điền,
     CA + Certificate Profile nào được dùng)
   - Điền username, DN, SAN dự kiến...
   - Chọn Token type: User Generated (client tự sinh CSR),
     P12/JKS (EJBCA tự sinh key + cert, đóng gói sẵn),
     hoặc chỉ enroll qua protocol tự động (SCEP/EST/ACME/CMP)
   - Set trạng thái ban đầu: thường là "New"
   - (tuỳ chọn) set enrollment code — password 1 lần để entity
     tự enroll mà không cần thêm xác thực khác

2. Entity thực hiện enrollment:
   - Qua Admin/RA Web (nhập enrollment code)
   - Qua CLI (`ejbca.sh ra ...`)
   - Qua REST API / SCEP / EST / CMP / ACME (tự động, xem
     [[ejbca--api-cli]])

3. EJBCA verify: End Entity đang ở trạng thái hợp lệ để enroll
   (vd "New" hoặc "Failed" retry) → CSR/thông tin khớp với những gì
   End Entity Profile cho phép → CA ký

4. End Entity chuyển trạng thái "Generated" (đã cấp cert thành
   công) — 1 lần enroll dùng hết, muốn cấp lại (renew) thường cần
   reset trạng thái về "New" hoặc tạo request mới tuỳ cấu hình.
```

**Approval workflow (Approval Profile):** với thao tác nhạy cảm, EJBCA cho phép cấu hình 1 hành động (vd tạo End Entity, revoke cert) cần **admin khác** duyệt mới thực thi được — người tạo request không được tự duyệt request của chính mình.

## Config gotchas

- **Enrollment code (password) yếu hoặc dùng lại nhiều lần** — nếu để entity tự chọn/đoán được code, ai cũng enroll giả danh entity đó được; nên sinh random đủ mạnh và giới hạn 1 lần dùng.
- **Trạng thái End Entity không tự reset sau khi cert hết hạn** — muốn renew phải chủ động thao tác lại (qua Admin Web, API, hoặc tự động hoá qua script/service), không có kiểu "tự renew" nếu không cấu hình thêm.
- **RA Web vs Admin Web nhầm phạm vi quyền** — RA Web nên dùng cho người chỉ cần duyệt/tạo End Entity theo Profile đã định sẵn, không nên cấp quyền Admin Web đầy đủ (có thể sửa cả Profile, CA, Role) cho người chỉ làm nghiệp vụ đăng ký hàng ngày.

## Security notes

- Bước tạo End Entity chính là bước "xác minh danh tính" thực sự của toàn hệ thống PKI (tương đương vai trò RA trong lý thuyết — xem [[PKI]]) — nếu ai cũng tạo được End Entity tuỳ ý, mọi kiểm soát ở tầng Certificate Profile phía sau đều vô nghĩa.
- Approval Profile (2-person rule) nên bắt buộc cho các End Entity Profile cấp cert nhạy cảm (vd cert cho hệ thống payment, cert admin nội bộ).

## Refs

- [[ejbca--profiles]] — End Entity Profile quyết định field/CA/Certificate Profile nào khả dụng.
- [[pki--csr-enrollment]] — lý thuyết nền CSR mà entity gửi lên khi enroll (trường hợp "User Generated").
- [[ejbca--rbac-admin-roles]] — phân quyền ai được thao tác Admin Web/RA Web.
