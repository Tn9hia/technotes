---
type: concept
aliases: [X.509, X509 certificate, certificate extensions]
tags: [pki, x509]
version: "n/a"
verified: 2026-10-04
parent: "[[PKI]]"
related: ["[[pki--ca-hierarchy-trust-chain]]", "[[pki--certificate-lifecycle]]", "[[ejbca--profiles-end-entities]]"]
---

# PKI — X.509 Certificate

## What it does

X.509 (profile trong RFC 5280) là định dạng chuẩn của certificate: buộc 1 public key với 1 danh tính
(Subject + SAN), kèm thời hạn và các **extension** giới hạn cách dùng, tất cả được CA ký lại.

## Why it exists

Chữ ký của CA chỉ có ý nghĩa nếu mọi client hiểu giống nhau "CA đang cam kết điều gì". X.509 chuẩn hoá điều
đó: cert này của ai, dùng vào việc gì, có được ký cert khác không, hết hạn khi nào, hỏi trạng thái ở đâu.
Extension là nơi cert tự giới hạn chính nó — sai extension là sai cam kết.

## How it works

Cấu trúc 1 certificate (ASCII vì đây là layout field, không phải luồng):

```text
Certificate
├── tbsCertificate                  ← phần được CA ký
│   ├── version            v3
│   ├── serialNumber       duy nhất trong phạm vi 1 CA
│   ├── signature          thuật toán CA dùng để ký (vd sha256WithRSAEncryption)
│   ├── issuer             DN của CA ký
│   ├── validity           notBefore / notAfter
│   ├── subject            DN của chủ thể (CN=..., O=...)
│   ├── subjectPublicKeyInfo
│   └── extensions
│       ├── basicConstraints       CA:TRUE/FALSE, pathLen
│       ├── keyUsage               digitalSignature, keyCertSign, cRLSign...
│       ├── extendedKeyUsage       serverAuth, clientAuth, codeSigning, OCSPSigning...
│       ├── subjectAltName         DNS:..., IP:..., email:...
│       ├── crlDistributionPoints  URL tải CRL
│       ├── authorityInfoAccess    URL OCSP + URL tải cert của CA (caIssuers)
│       ├── subjectKeyIdentifier / authorityKeyIdentifier   dùng để build chain
│       └── nameConstraints        (chỉ CA) giới hạn tên CA con được cấp
├── signatureAlgorithm
└── signatureValue                  ← chữ ký của CA trên tbsCertificate
```

1. Client không tin nội dung cert vì nó "nói vậy" — nó tin vì `signatureValue` verify được bằng public key
   của issuer, và issuer chain lên được tới trust anchor.
2. `authorityKeyIdentifier` của cert con khớp `subjectKeyIdentifier` của CA cha — đây là cách client chọn đúng
   CA khi có nhiều CA trùng tên (vd sau khi rekey CA).
3. Extension đánh dấu **critical** bắt buộc client phải hiểu; client không hiểu extension critical phải reject
   cert.

Đọc nhanh 1 cert:

```bash
openssl x509 -in cert.pem -noout -subject -issuer -dates -serial
openssl x509 -in cert.pem -noout -ext subjectAltName,basicConstraints,keyUsage,extendedKeyUsage
```

## Config gotchas

| Config | Default | Khuyến nghị | Vì sao |
|---|---|---|---|
| `subjectAltName` | Không có (OpenSSL không tự thêm) | Luôn có, gồm mọi DNS/IP client dùng để kết nối | Client hiện đại chỉ match SAN; kết nối bằng IP mà SAN không có IP là mismatch |
| `basicConstraints` trên leaf | Không có | `critical, CA:FALSE` | Thiếu hoặc `CA:TRUE` → leaf có thể đóng vai CA |
| `extendedKeyUsage` | Không có | Đúng mục đích, không gom tất cả | mTLS cần `clientAuth` ở phía client; OCSP signer cần `OCSPSigning` |
| `keyUsage` của CA | Tuỳ tool | `critical, keyCertSign, cRLSign` (+ `digitalSignature` nếu CA tự ký OCSP) | Thiếu `cRLSign` → CRL do CA ký bị client reject |
| `serialNumber` | Tuỳ CA | Random, ≥ 64 bit entropy | CA/B Forum yêu cầu với public cert; serial đoán được hỗ trợ tấn công collision |
| Signature hash | SHA-256 | SHA-256+ | SHA-1 bị reject |

## Hay nhầm lẫn

| Dễ nhầm | Thực tế | Hậu quả khi nhầm |
|---|---|---|
| `keyUsage` vs `extendedKeyUsage` | KU: thao tác crypto (ký, mã hoá key). EKU: mục đích ứng dụng (TLS server, client...) | Có KU đúng nhưng thiếu EKU vẫn fail mTLS |
| PEM vs DER | Cùng nội dung; PEM là base64 có header `-----BEGIN`, DER là binary | Import nhầm format → tool báo "unable to load certificate" |
| `.crt` / `.cer` / `.pem` | Đuôi file không quyết định format | Đổi đuôi không chuyển format; phải `openssl x509 -inform der -outform pem` |

## Security notes

- Extension do requester đưa vào CSR **không** được tự động tin — CA/profile quyết định extension nào được
  copy vào cert. Cho phép override là lỗ hổng (xem [[ejbca--profiles-end-entities]]).
- `nameConstraints` trên Issuing CA giới hạn domain CA đó cấp được — giảm thiệt hại nếu Issuing CA bị lạm dụng.

## Refs

- [[PKI]] — note gốc.
- [[pki--ca-hierarchy-trust-chain]] — `basicConstraints`/`pathLen` trong bối cảnh phân tầng CA.
- [RFC 5280 §4 — Certificate and Certificate Extensions Profile](https://datatracker.ietf.org/doc/html/rfc5280#section-4)
