# PKI — Certificate Lifecycle
Tier: 2
Parent: [[PKI]]
Related: [[pki--csr-enrollment]], [[pki--revocation-crl-ocsp]], [[pki--enrollment-protocols]]
Tags: #pki #lifecycle

## What it does

Toàn bộ vòng đời của 1 certificate: **Issuance (cấp mới) → Renewal (gia hạn) → Rekey (đổi key) → Expiry (hết hạn) hoặc Revocation (thu hồi trước hạn)**. Đây là phần "vận hành hàng ngày" của PKI — nơi tốn effort nhất trong thực tế, nhiều hơn hẳn phần thiết kế CA hierarchy ban đầu.

## Why it exists

Cert không thể sống mãi (key có thể bị lộ, danh tính có thể thay đổi, thuật toán cũ có thể bị phá) nên cần vòng đời giới hạn + cơ chế thay mới định kỳ. Vấn đề vận hành cốt lõi PKI phải giải: **làm sao gia hạn hàng nghìn cert đúng hạn mà không có cert nào "rớt" gây downtime** (cert hết hạn đột ngột là nguyên nhân outage kinh điển ở mọi công ty lớn).

## How it works (flow/diagram)

```
        ┌──────────┐   CSR + verify danh tính   ┌──────────┐
  New → │ Issuance │ ─────────────────────────→ │  Active  │
        └──────────┘                             └────┬─────┘
                                                        │
                          gần hết hạn (vd còn 30 ngày)  │
                          ┌─────────────────────────────┤
                          ▼                             │
                   ┌─────────────┐   xin cert mới,      │
                   │   Renewal   │   thường tái dùng    │
                   │  (± Rekey)  │   identity cũ         │
                   └──────┬──────┘                       │
                          │                              │
                          ▼                              ▼
                   ┌─────────────┐              ┌─────────────────┐
                   │   Active    │              │  Expired /       │
                   │ (cert mới)  │              │  Revoked          │
                   └─────────────┘              └──────────────────┘
```

- **Renewal** — xin cert mới cho cùng 1 danh tính khi cert cũ sắp hết hạn, thường **giữ nguyên keypair** nếu chỉ renew thuần.
- **Rekey** — renew kèm **sinh keypair mới** — an toàn hơn vì giới hạn thời gian sống của 1 private key, nên là best practice mỗi lần renew.
- **Expiry** — cert tự động vô hiệu khi qua `notAfter`, không cần hành động gì thêm, nhưng đây cũng là nguyên nhân outage phổ biến nhất nếu quên renew kịp.
- **Revocation** — chủ động vô hiệu hoá cert **trước** khi hết hạn (khi key leak, nhân viên nghỉ việc, thiết bị mất...) → xem chi tiết ở [[pki--revocation-crl-ocsp]].

## Config gotchas

- **Không có cảnh báo trước khi cert hết hạn** là gotcha #1 toàn ngành — luôn cần hệ thống theo dõi expiry (dashboard/alert ở 30/14/7 ngày trước hạn) độc lập với chính CA (vì nếu CA down thì alert nội bộ vẫn phải bắn ra).
- **Renew nhưng quên restart/reload service** — cert file mới nằm trên đĩa nhưng process đang chạy vẫn dùng cert cũ đã load vào memory, đến khi cert cũ hết hạn mới phát hiện.
- **Renewal window quá sát hạn** — nếu chỉ renew đúng ngày hết hạn, bất kỳ trục trặc nào ở CA/network cũng gây downtime; nên renew sớm (vd còn 1/3 thời hạn) để có buffer.
- **Tự động renew (ACME-style) mà không giám sát** — tự động hoá tốt nhưng nếu job renew tự động fail âm thầm (permission, network, rate limit) thì vẫn dẫn tới hết hạn bất ngờ; luôn cần alert riêng cho chính job renew.

## Security notes

- Rekey mỗi lần renew tốt hơn renew thuần vì giảm "thời gian sống hiệu dụng" của 1 private key — nếu key bị lộ mà không ai biết, rekey định kỳ giới hạn thời gian kẻ tấn công có thể lợi dụng.
- Thời hạn cert càng ngắn (vd cert nội bộ ngắn hạn qua ACME tự động) càng giảm rủi ro khi quên revoke kịp thời — đánh đổi lấy gánh nặng vận hành tự động hoá.

## Refs

- [[pki--revocation-crl-ocsp]] — vô hiệu hoá cert trước hạn.
- [[pki--enrollment-protocols]] — tự động hoá renew/rekey.
- [[ejbca--ops-runbook]] — cách theo dõi expiry thực tế trong EJBCA.
