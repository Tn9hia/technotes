# PKI — Enrollment Protocols (ACME / SCEP / EST / CMP)
Tier: 2
Parent: [[PKI]]
Related: [[pki--csr-enrollment]], [[pki--certificate-lifecycle]], [[ejbca--api-cli]]
Tags: #pki #protocol #automation

## What it does

Các giao thức chuẩn hoá để **tự động hoá** việc xin cấp/renew cert (thay vì người vào web UI click xin cert thủ công) — máy nói chuyện trực tiếp với CA để lấy cert. Đây là thứ biến PKI từ "quy trình thủ công cồng kềnh" thành "hạ tầng tự vận hành".

## Why it exists

Với vài chục cert, xin cấp thủ công qua web UI ổn. Với hàng nghìn service/thiết bị, hoặc cert thời hạn ngắn phải renew liên tục (best practice bảo mật), thủ công không scale — cần máy tự làm toàn bộ vòng đời enrollment mà không cần người can thiệp mỗi lần.

## How it works (flow/diagram)

**So sánh nhanh 4 giao thức phổ biến:**

| Protocol | Dùng nhiều ở đâu | Đặc điểm |
|---|---|---|
| **ACME** | TLS web (Let's Encrypt khởi xướng), ngày càng phổ biến nội bộ | Tự động hoàn toàn kể cả bước "chứng minh sở hữu domain" (HTTP-01/DNS-01 challenge); không cần pre-shared secret ban đầu |
| **SCEP** | Thiết bị mạng, router, MDM (quản lý điện thoại), IoT | Đơn giản, hỗ trợ rộng ở thiết bị cũ; điểm yếu là cơ chế xác thực ban đầu (pre-shared challenge password) khá yếu so với chuẩn hiện đại |
| **EST** | Kế thừa SCEP, dùng nhiều trong network/IoT hiện đại hơn | Dựa trên TLS + HTTP, hỗ trợ xác thực mạnh hơn SCEP (client cert hoặc credential), có thể renew tự động |
| **CMP** | Môi trường enterprise/telecom, yêu cầu tuân thủ cao | Giao thức đầy đủ tính năng nhất (RFC 4210), hỗ trợ nhiều loại thao tác (issue/revoke/update key) nhưng phức tạp triển khai hơn |

**Flow điển hình (ACME làm ví dụ, vì phổ biến nhất để hiểu tư duy):**
```
1. Client đăng ký account với CA (ACME server)
2. Client xin issue cert cho domain X → CA đưa ra "challenge"
   (yêu cầu client chứng minh sở hữu domain X, vd đặt 1 file tại
   http://X/.well-known/acme-challenge/... hoặc 1 TXT record DNS)
3. Client hoàn thành challenge → CA verify → CA tin client sở hữu domain X
4. Client gửi CSR → CA ký → trả cert
5. (renew) Client tự lặp lại toàn bộ flow trước khi cert hết hạn,
   không cần người can thiệp
```

EJBCA hỗ trợ làm ACME/EST/SCEP/CMP server — tức là công ty có thể để thiết bị/service tự động enrollment thẳng vào EJBCA qua các protocol này, không cần thao tác tay trên Admin Web. Xem [[ejbca--api-cli]].

## Config gotchas

- **Chọn protocol theo khả năng của client**, không phải theo sở thích — thiết bị mạng cũ thường chỉ hỗ trợ SCEP, không tự nhiên nâng lên ACME được.
- **ACME HTTP-01 challenge** đòi port 80 mở ra internet/mạng nội bộ tại đúng thời điểm challenge — vấn đề thường gặp với server nằm sau firewall chặt.
- **SCEP challenge password dùng chung/tĩnh** cho nhiều thiết bị là lỗ hổng phổ biến — nên rotate hoặc theo từng thiết bị nếu CA hỗ trợ.
- **Renew tự động chạy như 1 cronjob/service riêng** — cần giám sát riêng job này (xem [[pki--certificate-lifecycle]] gotcha về renew tự động fail âm thầm).

## Security notes

- Bước xác thực ban đầu (challenge) là điểm yếu nhất của mọi giao thức tự động — nếu challenge dễ giả mạo (vd SCEP static password lộ ra), attacker có thể tự enrollment lấy cert hợp lệ giả danh thiết bị khác.
- Giao thức tự động hoá làm tăng attack surface (endpoint enrollment mở cho máy gọi tới) — cần rate-limit, logging, và giới hạn scope (RA Profile/End Entity Profile ở EJBCA) rõ ràng cho từng protocol.

## Refs

- RFC 8555 (ACME), RFC 8894 (SCEP), RFC 7030 (EST), RFC 4210 (CMP).
- [[ejbca--api-cli]] — bật/cấu hình các protocol này trên EJBCA.
