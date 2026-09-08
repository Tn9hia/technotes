# PKI — X.509 Certificate
Tier: 2
Parent: [[PKI]]
Related: [[pki--asymmetric-crypto]], [[pki--ca-hierarchy-trust-chain]], [[pki--csr-enrollment]]
Tags: #pki #x509

## What it does

X.509 là chuẩn định dạng của certificate số — một file có cấu trúc buộc chặt **public key** với **danh tính** (subject), kèm **chữ ký của CA** để xác nhận sự buộc chặt đó là thật. Đây là "tờ giấy" mà toàn bộ TLS/HTTPS, mTLS, code signing... đều dùng chung format.

## Why it exists

Nếu chỉ gửi public key trần trụi, không ai biết key đó thuộc về ai. X.509 chuẩn hoá cách đóng gói: key + metadata (chủ sở hữu, mục đích dùng, thời hạn, ai xác nhận) thành 1 object có thể verify được bằng toán học (chữ ký số), thay vì phải tin bằng miệng.

## How it works (flow/diagram)

Các field quan trọng trong 1 cert X.509:

```
Certificate
├── Version                    (v1/v2/v3 — hầu như luôn dùng v3 vì có extensions)
├── Serial Number               (định danh duy nhất, CA cấp — dùng để revoke)
├── Signature Algorithm         (thuật toán CA dùng để ký, vd sha256WithRSAEncryption)
├── Issuer                      (ai ký cert này — DN của CA)
├── Validity
│   ├── Not Before
│   └── Not After               (hết hạn — hết là cert vô hiệu, không cần revoke)
├── Subject                     (DN của chủ thể — CN, O, OU, C...)
├── Subject Public Key Info     (public key + thuật toán)
├── Extensions (v3)             ← quan trọng nhất để hiểu hành vi thực tế
│   ├── basicConstraints        (CA:TRUE/FALSE, pathLenConstraint)
│   ├── keyUsage                (digitalSignature, keyEncipherment, keyCertSign...)
│   ├── extendedKeyUsage (EKU)  (serverAuth, clientAuth, codeSigning, emailProtection...)
│   ├── subjectAltName (SAN)    (domain/IP thực tế cert đại diện — browser chỉ tin cái này)
│   ├── authorityKeyIdentifier  (trỏ tới key của CA đã ký — giúp build chain)
│   ├── subjectKeyIdentifier    (định danh key của chính cert này)
│   ├── crlDistributionPoints   (URL để tải CRL)
│   └── authorityInfoAccess     (URL OCSP responder, URL tải cert của CA)
└── Signature                   (chữ ký của CA trên toàn bộ nội dung trên)
```

**Format lưu trữ thường gặp:**
- **PEM** — text, base64, có header `-----BEGIN CERTIFICATE-----`. Phổ biến nhất trên Linux/OpenSSL/EJBCA.
- **DER** — binary, cùng nội dung với PEM nhưng không encode base64.
- **PKCS#7 (.p7b)** — bundle nhiều cert (thường dùng để chuyển cả chain).
- **PKCS#12 (.p12/.pfx)** — bundle cert + private key, có password bảo vệ — cẩn thận khi truyền file này.

## Config gotchas

- **CN vs SAN** — trình duyệt hiện đại (từ ~2017) **bỏ qua CN**, chỉ tin `subjectAltName`. Cert thiếu SAN sẽ bị reject dù CN đúng domain — lỗi rất hay gặp khi ai đó tạo cert theo thói quen cũ.
- **`basicConstraints: CA:TRUE`** đặt nhầm cho leaf cert = lỗ hổng nghiêm trọng (cert đó có thể tự ký cert khác, phá vỡ toàn bộ mô hình trust). Đây là 1 trong những default nguy hiểm nhất khi tự cấu hình certificate profile.
- **`keyUsage`/`EKU` để trống hoặc quá rộng** — nên giới hạn đúng mục đích; 1 cert có cả `serverAuth` lẫn `codeSigning` là dấu hiệu profile bị cấu hình cẩu thả.
- **`pathLenConstraint`** — giới hạn số tầng sub-CA được phép tạo tiếp từ 1 intermediate CA; thiếu constraint này thì intermediate CA có thể đẻ ra chuỗi sub-CA vô hạn ngoài kiểm soát.
- **Serial number phải unique** trong phạm vi 1 CA — CA nào tự sinh serial trùng là bug nghiêm trọng (một số client dùng serial để cache/dedupe).

## Security notes

- Đừng tin `Subject`/`Issuer` string một cách mù quáng — chúng chỉ có ý nghĩa khi chữ ký verify được lên tới trust anchor.
- `notAfter` ngắn hạn giảm thiệt hại khi private key leak nhưng tăng gánh nặng renew — đây là trade-off chính khi thiết kế certificate profile.
- Khi debug 1 cert, luôn xem trực tiếp bằng `openssl x509 -in cert.pem -noout -text` để thấy đúng field thật, đừng suy diễn từ tên file.

## Refs

- RFC 5280 (chuẩn X.509 + CRL profile).
- [[pki--ca-hierarchy-trust-chain]] — Issuer/Subject liên kết thành chain thế nào.
- [[ejbca--profiles]] — cách EJBCA cho cấu hình các field/extension này qua Certificate Profile.
