# PKI — CSR & Manual Enrollment
Tier: 2
Parent: [[PKI]]
Related: [[pki--x509-certificate]], [[pki--enrollment-protocols]], [[ejbca--end-entities-ra]]
Tags: #pki #csr

## What it does

CSR (Certificate Signing Request) là file mà một entity tự tạo để **xin cấp cert**: chứa public key của entity + thông tin danh tính mong muốn (CN, SAN, O...), được ký bằng chính private key tương ứng (chứng minh entity thực sự sở hữu private key đó — gọi là **Proof of Possession**). CA nhận CSR, xác minh danh tính, rồi ký ra cert.

## Why it exists

CA không tự sinh keypair cho entity (trừ vài trường hợp đặc biệt) vì nếu vậy CA phải biết private key của entity — vi phạm nguyên tắc private key không rời khỏi nơi sinh ra. CSR cho phép entity tự giữ private key từ đầu đến cuối, chỉ gửi phần public cho CA.

## How it works (flow/diagram)

```
1. Entity sinh keypair (private key giữ lại, không gửi đi)
2. Entity tạo CSR = public key + subject info, ký bằng private key
        openssl req -new -key private.key -out request.csr \
          -subj "/CN=app.internal.example.com" \
          -addext "subjectAltName=DNS:app.internal.example.com"
3. Gửi CSR cho RA/CA (qua web UI, API, hoặc protocol tự động — xem
   [[pki--enrollment-protocols]])
4. RA verify danh tính người/hệ thống xin cấp (đây là bước quan trọng
   nhất, quyết định độ tin cậy toàn bộ hệ thống PKI)
5. CA verify chữ ký trên CSR (= entity đúng là chủ private key)
6. CA áp Certificate Profile (giới hạn field nào được chấp nhận,
   override field nào) → ký ra cert
7. Trả cert về cho entity
```

**Định dạng chuẩn:** CSR dùng chuẩn **PKCS#10**.

## Config gotchas

- **CSR chứa field gì không có nghĩa CA sẽ giữ nguyên field đó** — CA/Certificate Profile có thể override (vd entity xin `CA:TRUE` trong CSR nhưng profile không cho phép → CA bỏ qua, đặt `CA:FALSE`). Đừng tưởng CSR là "đơn đặt hàng đúng y", nó chỉ là đề nghị.
- **SAN không tự động đi kèm CN** — nếu tạo CSR chỉ set `-subj CN=...` mà quên `-addext subjectAltName`, cert ra đời sẽ thiếu SAN và bị browser/client hiện đại reject.
- **Reuse CSR cũ để renew** — CSR không có `notBefore`/`notAfter`, chỉ CA quyết định thời hạn, nên có thể tái dùng CSR để renew nếu key chưa đổi, nhưng **best practice là rekey mỗi lần renew** (sinh key mới) để giảm thời gian sống của 1 private key.
- **CSR ký bằng thuật toán yếu** (vd SHA-1) sẽ bị nhiều CA hiện đại từ chối thẳng.

## Security notes

- Bước "verify danh tính" ở RA là điểm yếu con người dễ bị social-engineer nhất trong toàn bộ PKI — CA có thể toán học hoàn hảo nhưng nếu RA duyệt CSR ẩu (không check ai thực sự đứng sau request) thì cả hệ thống vô nghĩa.
- Private key sinh cùng lúc với CSR nên được sinh **tại đúng nơi nó sẽ được dùng** (trên server đích, hoặc trong HSM) — tránh sinh key ở máy trung gian rồi copy qua lại.

## Refs

- [[pki--enrollment-protocols]] — tự động hoá bước gửi CSR + nhận cert (ACME/SCEP/EST/CMP) thay vì làm thủ công.
- [[ejbca--end-entities-ra]] — cách EJBCA quản lý bước RA-approval trước khi ký.
