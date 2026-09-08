# PKI — Certificate Policy (CP) & CPS
Tier: 2
Parent: [[PKI]]
Related: [[pki--ca-hierarchy-trust-chain]], [[ejbca--profiles]]
Tags: #pki #governance #compliance

## What it does

**CP (Certificate Policy)** — tài liệu mô tả "loại cert này được cấp cho ai, theo điều kiện gì, dùng để làm gì" (vd: chính sách cấp cert cho server nội bộ khác với chính sách cấp cert cho VPN client). **CPS (Certification Practice Statement)** — tài liệu mô tả chi tiết "CA thực hiện quy trình đó **như thế nào**" (ai vận hành, quy trình verify danh tính, tần suất audit, xử lý sự cố...). CP nói "làm gì", CPS nói "làm cách nào".

## Why it exists

Cert tự nó chỉ là toán học — nó không nói lên được "CA này có đáng tin không, quy trình xác minh danh tính chặt chẽ tới đâu". CP/CPS là lớp governance để bên thứ ba (auditor, đối tác, hoặc chính nội bộ công ty) đánh giá được mức độ tin cậy của toàn bộ PKI, không chỉ dựa vào "có cert là được".

## How it works (flow/diagram)

```
CP (Policy — "cái gì")
  └── định nghĩa: loại cert, đối tượng được cấp, mức độ verify cần thiết
        (vd: Class 1 chỉ verify email, Class 3 verify danh tính pháp nhân)

CPS (Practice Statement — "làm sao")
  └── mô tả cụ thể: ai vận hành CA, RA verify bằng cách nào,
        HSM dùng loại gì, tần suất audit, quy trình revoke,
        thời gian phản hồi khi có sự cố...

Mỗi CP thường có 1 OID (Object Identifier) riêng, được nhúng vào
certificate qua extension `certificatePolicies` — cho phép máy
(không chỉ người) đọc và biết cert này được cấp theo chính sách nào.
```

Trong thực tế nội bộ công ty (không phải CA công khai cho internet), CP/CPS có thể đơn giản hơn nhiều so với CA thương mại (vd DigiCert, Let's Encrypt phải tuân CA/Browser Forum Baseline Requirements rất chặt) — nhưng vẫn nên có tối thiểu 1 tài liệu ngắn ghi rõ: ai được cấp loại cert nào, quy trình duyệt ra sao, ai chịu trách nhiệm khi có sự cố.

## Config gotchas

- **Certificate Profile trong EJBCA nên map trực tiếp tới 1 CP cụ thể** — nếu để nhân viên tự tạo Certificate Profile tuỳ ý không theo policy nào, hệ thống rất nhanh trở nên hỗn loạn (không ai nhớ vì sao profile X tồn tại, dùng cho việc gì).
- **Thiếu CPS = không ai biết quy trình vận hành thực tế khi người cũ nghỉ việc** — đây chính là rủi ro của "bàn giao công việc" — CP/CPS (dù đơn giản) là tài liệu sống còn giúp người mới không phải đoán mò quy trình.

## Security notes

- Với PKI nội bộ, CP/CPS không cần phức tạp như CA công khai, nhưng tối thiểu phải trả lời được: ai duyệt request cấp cert, quy trình revoke khi nghi ngờ compromise, ai có quyền truy cập HSM/private key CA.

## Refs

- CA/Browser Forum Baseline Requirements — tham khảo chuẩn ngành dù không bắt buộc áp dụng cho CA nội bộ.
- [[ejbca--profiles]] — nơi CP được hiện thực hoá thành cấu hình kỹ thuật cụ thể trong EJBCA.
