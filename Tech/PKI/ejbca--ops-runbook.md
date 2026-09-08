# EJBCA — Ops Runbook (Production Notes)
Tier: 2
Parent: [[EJBCA]]
Related: [[ejbca--architecture-components]], [[ejbca--publishers-services-crl-ocsp]], [[pki--hsm-key-protection]]
Tags: #ejbca #ops #runbook

## What it does

Checklist vận hành hàng ngày/định kỳ cho 1 hệ thống EJBCA đang chạy production — health check, log cần theo dõi, backup/restore, upgrade, và các tình huống sự cố thường gặp.

## Why it exists

EJBCA là hạ tầng nền (giống DNS) — khi nó hoạt động đúng, không ai để ý; khi nó lỗi, hàng loạt hệ thống phụ thuộc (TLS, mTLS, VPN, 802.1x...) có thể sập cùng lúc. Runbook tồn tại để phát hiện sự cố **trước** khi nó lan rộng, và để người tiếp nhận (không phải người xây dựng ban đầu) biết chính xác phải nhìn vào đâu khi có vấn đề.

## How it works (flow/diagram)

**Health check hàng ngày:**
```
1. App server (WildFly) còn up không, đủ tài nguyên (heap, CPU, disk)?
2. Database connection pool khoẻ không (connection timeout/exhaustion
   là dấu hiệu sớm của sự cố)?
3. Crypto Token/HSM connectivity — CA còn ký được không? (test bằng
   cách thử issue 1 cert test, hoặc check trạng thái Crypto Token
   trên Admin Web — "Active" vs "Offline")
4. CRL của từng CA — nextUpdate còn xa hạn không, Service sinh CRL
   có chạy đúng lịch không?
5. OCSP Responder — trả response đúng không (test bằng openssl ocsp
   hoặc script định kỳ)?
6. Certificate của chính các CA (Issuing CA, và cả cert TLS của
   chính EJBCA Admin/RA Web) — còn hạn xa không?
```

**Log quan trọng cần biết đọc:**
- **Audit log** (trong DB, xem qua Admin Web hoặc export) — ai làm gì, khi nào: tạo/sửa End Entity, issue/revoke cert, thay đổi Role/Profile.
- **Application log** (app server, thường ở thư mục log của WildFly) — lỗi kết nối DB, lỗi Crypto Token, exception khi xử lý request.
- **Log kết nối HSM** (thường riêng, phía HSM vendor cung cấp) — quan trọng khi debug "CA không ký được" mà log EJBCA không rõ nguyên nhân.

**Metric nên có alert:**
- Cert của CA (Issuing/Root nếu chạy trong EJBCA) sắp hết hạn (ngưỡng cảnh báo sớm, vd 90/60/30 ngày).
- CRL sắp/đã quá `nextUpdate`.
- Service (worker) job fail liên tiếp (đặc biệt CRL generation service).
- Crypto Token chuyển trạng thái "Offline"/mất kết nối.
- Disk usage tăng bất thường (audit log/CRL tích luỹ theo thời gian, cần chiến lược archive).
- DB connection pool exhaustion / latency tăng.

**Backup & Restore:**
```
Cần backup đồng bộ (không lệch thời điểm) 3 thứ:
  1. Database (cert, metadata, Profile, Role, audit log)
  2. Crypto Token — nếu soft keystore: file keystore + password
     (lưu tách biệt khỏi DB backup); nếu HSM: quy trình backup
     riêng theo vendor (thường là key-share/smartcard backup từ
     lúc key ceremony)
  3. Cấu hình app server (nếu có custom config ngoài DB)

Restore-test định kỳ (không chỉ backup rồi để đó) — 1 CA "backup
được" nhưng chưa từng restore-test là rủi ro tiềm ẩn lớn, đặc biệt
với Crypto Token/HSM nơi quy trình restore thường phức tạp và ít
khi được luyện tập.
```

**Upgrade:**
- Luôn đọc release notes/upgrade path chính thức (EJBCA có thể yêu cầu upgrade tuần tự qua từng major version, không nhảy cóc được).
- Backup đầy đủ (DB + Crypto Token access) trước khi upgrade.
- Test upgrade trên môi trường staging có dữ liệu tương tự production trước.

## Config gotchas

- **Alert chỉ dựa vào chính EJBCA tự báo** — nếu EJBCA down hoàn toàn, nó không tự cảnh báo được; cần monitoring **độc lập bên ngoài** (external healthcheck, cert expiry scanner quét từ ngoài vào) không phụ thuộc EJBCA còn sống hay không.
- **Quên theo dõi cert TLS của chính Admin Web/RA Web** — nếu cert này hết hạn, admin có thể mất luôn khả năng đăng nhập để xử lý sự cố (tình huống "khoá luôn cả chìa khoá vào trong").

## Security notes

- Runbook nên tách rõ ai được thực hiện thao tác gì trong sự cố khẩn cấp (vd revoke hàng loạt) — tránh tình huống 1 người dưới áp lực tự ý làm thao tác nguy hiểm không qua approval bình thường.

## Refs

- [[ejbca--architecture-components]] — hiểu kiến trúc để biết health check đúng chỗ.
- [[ejbca--publishers-services-crl-ocsp]] — chi tiết Service/Publisher cần theo dõi.
- [[pki--hsm-key-protection]] — quy trình backup/restore Crypto Token liên quan trực tiếp tới key ceremony ban đầu.
