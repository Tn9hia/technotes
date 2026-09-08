# PKI — Revocation (CRL & OCSP)
Tier: 2
Parent: [[PKI]]
Related: [[pki--certificate-lifecycle]], [[ejbca--publishers-services-crl-ocsp]]
Tags: #pki #revocation

## What it does

Cơ chế báo cho client biết "cert này còn hạn nhưng **không còn đáng tin** nữa" — trước khi `notAfter`. Hai cách chính: **CRL** (danh sách offline, CA publish định kỳ) và **OCSP** (hỏi real-time 1 cert cụ thể).

## Why it exists

`notAfter` chỉ xử lý được trường hợp "hết hạn tự nhiên", không xử lý được trường hợp khẩn cấp: private key bị lộ hôm nay nhưng cert còn hạn tới sang năm. Không có revocation, kẻ tấn công cầm private key bị lộ vẫn dùng được cert hợp lệ cho tới ngày hết hạn tự nhiên — không có cách nào "thu hồi sớm".

## How it works (flow/diagram)

**CRL (Certificate Revocation List):**
```
CA publish định kỳ (vd mỗi 24h) 1 file ký sẵn, liệt kê serial number
của mọi cert đã bị revoke + lý do + thời điểm revoke.

Client tải CRL (từ URL trong field `crlDistributionPoints` của cert)
→ cache lại → check serial number của cert đang verify có nằm trong
danh sách không.

Ưu điểm: hoạt động offline sau khi tải, không lộ "ai đang check cert nào"
Nhược điểm: danh sách phình to theo thời gian, độ trễ = chu kỳ publish
(revoke lúc 10h nhưng CRL publish tiếp theo lúc 24h sau → có khoảng hở)
```

**OCSP (Online Certificate Status Protocol):**
```
Client gửi serial number của 1 cert cụ thể tới OCSP responder (VA)
→ responder trả về: good / revoked / unknown (ký bởi responder)

Ưu điểm: real-time, không cần tải cả danh sách
Nhược điểm: cần network tới OCSP responder mỗi lần verify (latency,
điểm chịu tải cao, lộ thông tin "ai đang kết nối tới đâu" cho CA biết)
```

**OCSP stapling** — giải nhược điểm của OCSP thuần: **server** (không phải client) chủ động query OCSP trước, "ghim" (staple) response đã ký vào TLS handshake gửi cho client. Client không cần tự kết nối OCSP responder nữa — vừa nhanh hơn vừa đỡ lộ thông tin.

## Config gotchas

- **Fail-open vs fail-closed khi CRL/OCSP không truy cập được** — đây là quyết định bảo mật quan trọng hay bị bỏ qua: fail-open (coi như hợp lệ nếu không check được) = tiện nhưng có thể bỏ lọt cert bị revoke thật; fail-closed (từ chối nếu không check được) = an toàn hơn nhưng OCSP responder down là cả hệ thống ngừng hoạt động. Phải quyết định rõ ràng theo mức độ rủi ro, không để default ngầm định.
- **CRL quá lớn** (revoke hàng loạt, hoặc lâu năm không dọn) làm client tải chậm/timeout — cần Delta CRL (chỉ chứa thay đổi từ CRL đầy đủ gần nhất) cho hệ thống có volume revoke cao.
- **Quên publish CRL đúng chu kỳ** — CRL có `nextUpdate` field, quá hạn đó mà chưa có bản mới thì client cẩn thận sẽ coi CRL đã "hết hạn" và có thể fail-closed toàn bộ.
- **OCSP responder cũng cần cert riêng** (thường có EKU `OCSPSigning`) — quên cấu hình đúng cert này là lỗi khiến responder trả response không ai tin được.

## Security notes

- Revocation reason nên được ghi rõ (keyCompromise, cessationOfOperation, superseded...) — giúp điều tra sau này, đặc biệt quan trọng khi audit.
- OCSP thuần (không stapling) cho CA biết được pattern truy cập của người dùng cuối (privacy concern) — cân nhắc khi thiết kế cho hệ thống public-facing.

## Refs

- [[pki--certificate-lifecycle]] — revocation là 1 nhánh trong vòng đời cert.
- [[ejbca--publishers-services-crl-ocsp]] — cách EJBCA generate/publish CRL và chạy OCSP responder thực tế.
