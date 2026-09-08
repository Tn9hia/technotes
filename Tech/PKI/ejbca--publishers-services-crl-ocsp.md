# EJBCA — Publishers, Services, CRL & OCSP
Tier: 2
Parent: [[EJBCA]]
Related: [[pki--revocation-crl-ocsp]], [[ejbca--ops-runbook]]
Tags: #ejbca #crl #ocsp #publisher

## What it does

- **Publisher** — module đẩy dữ liệu (cert, CRL) từ EJBCA ra hệ thống ngoài (LDAP/AD, database khác, filesystem...) mỗi khi có sự kiện (issue cert, revoke, CRL mới).
- **Service** — tác vụ nền chạy theo lịch bên trong EJBCA (worker), ví dụ: tự sinh CRL định kỳ, gửi cảnh báo cert sắp hết hạn, tự dọn dữ liệu cũ.
- **OCSP Responder** — module trả lời real-time trạng thái revocation của 1 cert, có thể chạy tích hợp trong cùng EJBCA node hoặc tách thành node OCSP riêng (VA) cho hiệu năng/độ sẵn sàng cao hơn.

## Why it exists

CA ký cert xong không tự động nghĩa là hệ thống khác (LDAP directory, firewall, ứng dụng nội bộ dùng cert để xác thực) biết cert đó tồn tại hay đã bị revoke — cần cơ chế chủ động đẩy dữ liệu ra (Publisher) và tác vụ định kỳ đảm bảo CRL luôn cập nhật (Service), thay vì chờ ai đó vào Admin Web check thủ công.

## How it works (flow/diagram)

```
Sự kiện: Issue cert mới / Revoke cert
        │
        ▼
┌───────────────────┐
│  Publisher(s)       │   → LDAP/AD (cho ứng dụng tra cứu cert user)
│  (0 hoặc nhiều,      │   → External Database (hệ thống khác cần biết
│   gắn theo CA hoặc   │      cert nào đang active)
│   Certificate       │   → Custom Publisher (viết plugin riêng nếu
│   Profile)           │      cần tích hợp hệ thống đặc thù)
└───────────────────┘

Song song, độc lập theo lịch:
┌───────────────────┐
│  CRL Update Service │   → CA tự sinh CRL mới theo chu kỳ cấu hình
│  (chạy định kỳ,      │      (vd mỗi vài giờ), publish lên
│   vd mỗi N phút/giờ) │      CRL Distribution Point (HTTP/LDAP)
└───────────────────┘
┌───────────────────┐
│  OCSP Responder      │   → Trả lời real-time mỗi query OCSP,
│  (luôn chạy, đọc      │      đọc trực tiếp trạng thái từ DB
│   trạng thái mới     │      (không cần chờ CRL cycle)
│   nhất từ DB)         │
└───────────────────┘
┌───────────────────┐
│  Expire Notification │   → Quét cert sắp hết hạn (theo ngưỡng
│  Service              │      cấu hình) → gửi email/trigger cảnh báo
└───────────────────┘
```

## Config gotchas

- **CRL Update Service chu kỳ quá dài** so với `overlap`/`nextUpdate` cấu hình trên CA → CRL có thể "hết hạn" trước khi service kịp sinh bản mới, client fail-closed sẽ reject toàn bộ (xem [[pki--revocation-crl-ocsp]]).
- **Publisher fail âm thầm** (vd LDAP đích down) — EJBCA thường log lỗi nhưng không luôn chặn transaction issue cert; cần alert riêng theo dõi Publisher queue/fail count, đừng giả định "issue thành công = publish thành công".
- **OCSP Responder tách node riêng** cần đồng bộ dữ liệu (DB replication hoặc dùng chung DB) với CA node — lệch dữ liệu là nguyên nhân OCSP trả "good" cho cert đã bị revoke ở node khác, hoặc ngược lại.
- **Service bị disable sau maintenance/upgrade** mà không ai để ý — luôn kiểm tra danh sách Service đang "Active" sau mỗi lần thay đổi cấu hình/upgrade lớn.

## Security notes

- OCSP Responder cần cert riêng với EKU `OCSPSigning` — cấu hình sai khiến response OCSP không được client tin tưởng (chain validation fail phía client dù bản thân câu trả lời đúng).
- Publisher đẩy dữ liệu ra hệ thống ngoài là điểm mở rộng attack surface — audit kỹ Custom Publisher tự viết (chạy code trong tiến trình EJBCA, quyền hạn tương đương chính EJBCA).

## Refs

- [[pki--revocation-crl-ocsp]] — lý thuyết nền CRL/OCSP.
- [[ejbca--ops-runbook]] — checklist theo dõi Service/Publisher/OCSP hàng ngày trong vận hành thực tế.
