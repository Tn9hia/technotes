# PKI — Asymmetric Cryptography (Public/Private Key)
Tier: 2
Parent: [[PKI]]
Related: [[pki--x509-certificate]], [[pki--hsm-key-protection]]
Tags: #pki #crypto

## What it does

Asymmetric crypto (public-key crypto) sinh ra 1 **cặp khoá toán học liên quan nhau**: private key (giữ bí mật) và public key (chia sẻ tự do). Dữ liệu mã hoá bằng public key chỉ giải được bằng private key tương ứng, và ngược lại — chữ ký tạo bằng private key thì ai cũng verify được bằng public key. Toàn bộ PKI đứng trên nền tảng này.

## Why it exists

Symmetric crypto (AES...) nhanh nhưng đòi hỏi 2 bên đã có sẵn shared secret — vấn đề "làm sao trao đổi secret lần đầu an toàn" (key distribution problem) không giải được nếu 2 bên chưa từng gặp nhau. Asymmetric crypto giải bài đó: public key phát tán công khai không sợ lộ, chỉ private key cần giữ kín, và nhờ vậy 2 bên lạ vẫn thiết lập được kênh an toàn (TLS handshake dùng asymmetric để trao đổi session key, sau đó chuyển sang symmetric vì nhanh hơn).

## How it works (flow/diagram)

```
Encrypt/Decrypt (confidentiality):
  Alice lấy public key của Bob → encrypt message
  → chỉ Bob (giữ private key) mới decrypt được

Sign/Verify (authenticity + integrity):
  Bob hash message → encrypt hash bằng private key = "chữ ký"
  Alice lấy public key của Bob → decrypt chữ ký ra hash
  → so sánh với hash tự tính từ message → khớp = đúng là Bob ký, message không bị sửa
```

Trong PKI, việc "CA ký cert" chính là thao tác sign ở trên: CA hash toàn bộ nội dung cert (subject, public key của entity, validity...) rồi encrypt hash đó bằng private key của CA. Ai có public key của CA (nằm sẵn trong cert của CA) đều verify được.

**Thuật toán phổ biến:**
- **RSA** — dựa trên độ khó phân tích số nguyên tố lớn. Phổ biến nhất, key size lớn (2048/4096 bit), tương thích rộng nhất với hệ thống cũ.
- **ECC (Elliptic Curve, vd P-256/P-384)** — cùng mức an toàn nhưng key size nhỏ hơn nhiều → nhanh hơn, nhẹ hơn. Ngày càng được ưu tiên cho leaf cert.
- **EdDSA (Ed25519)** — thuật toán ký hiện đại, nhanh và an toàn, nhưng support ở hệ thống enterprise/legacy (kể cả EJBCA/HSM đời cũ) còn hạn chế — kiểm tra compatibility trước khi chọn.

## Config gotchas

- **Key size quá nhỏ** — RSA < 2048 bit coi như unsafe, không dùng kể cả nội bộ.
- **Root/Intermediate CA nên dùng key mạnh hơn leaf** vì thời gian sống dài hơn nhiều (key phải chịu được sức tấn công của 15-20 năm tới, không phải chỉ hiện tại).
- **Trộn thuật toán trong 1 chain** (vd Root RSA, Intermediate ECC) hợp lệ về mặt kỹ thuật nhưng dễ gây nhầm lẫn khi debug — nên đồng nhất trừ khi có lý do rõ ràng.
- **Reuse keypair giữa nhiều mục đích** (vd 1 key dùng cho cả TLS lẫn code signing) là anti-pattern — lộ 1 chỗ là lộ hết, nên tách key theo mục đích qua `KeyUsage`/`EKU`.

## Security notes

- Private key **không bao giờ** rời khỏi nơi sinh ra nó nếu có thể tránh — lý tưởng nhất là sinh và lưu trong HSM, không export dạng plaintext (xem [[pki--hsm-key-protection]]).
- Random number generator (RNG) yếu khi sinh key là lỗ hổng kinh điển (vd Debian OpenSSL bug 2008) — luôn dùng CSPRNG chuẩn của hệ thống/thư viện crypto, không tự chế.
- Với thuật toán lượng tử tương lai (post-quantum), RSA/ECC hiện tại sẽ bị phá — đây là hướng research đang phát triển, chưa cần hành động ngay nhưng nên biết tên (Kyber, Dilithium...) khi đọc roadmap các CA lớn.

## Refs

- [[pki--x509-certificate]] — nơi public key được đóng gói cùng danh tính.
- [[pki--hsm-key-protection]] — cách bảo vệ private key thực tế.
