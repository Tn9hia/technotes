# PKI — CA Hierarchy & Chain of Trust
Tier: 2
Parent: [[PKI]]
Related: [[pki--x509-certificate]], [[pki--hsm-key-protection]], [[ejbca--architecture-components]]
Tags: #pki #ca

## What it does

Mô hình phân tầng CA: **Root CA** (tự ký, là gốc của mọi niềm tin) → **Intermediate/Issuing CA** (được Root ký, dùng để cấp cert hàng ngày) → **Leaf/End-entity cert** (cert thực tế server/user/device dùng). Chain of trust là chuỗi chữ ký nối từ leaf cert ngược lên tới Root CA đã có sẵn trong trust store.

## Why it exists

Nếu chỉ có 1 tầng CA duy nhất và nó phải online 24/7 để ký cert liên tục, private key của nó là mục tiêu tấn công cực lớn — lộ key này là sập toàn bộ hệ thống trust, không cách nào cứu được ngoài build lại từ đầu. Tách tầng cho phép **Root CA nằm offline gần như vĩnh viễn** (rủi ro lộ key giảm mạnh), còn Intermediate CA — dù online và rủi ro hơn — nếu bị compromise thì chỉ cần **revoke + thay intermediate đó**, Root CA vẫn nguyên vẹn, hệ thống hồi phục nhanh hơn nhiều.

## How it works (flow/diagram)

```
Root CA (offline, self-signed, trust anchor)
   │ ký 1 lần, ít khi động vào (vài năm/lần)
   ▼
Intermediate CA A          Intermediate CA B      ← có thể tách theo môi trường
(vd: production)           (vd: staging/internal)     (prod/staging) hoặc theo
   │                            │                      mục đích (TLS/code-signing)
   ▼                            ▼
Leaf cert (server.internal) Leaf cert (app.staging)
```

**Chain validation (client side):**
1. Client nhận leaf cert từ server → server thường gửi kèm cả chain (leaf + intermediate, KHÔNG gửi root — root phải có sẵn ở client).
2. Client dò `Issuer` của leaf → tìm cert intermediate tương ứng trong bundle gửi kèm.
3. Verify chữ ký leaf bằng public key intermediate.
4. Dò `Issuer` của intermediate → so với trust store local, nếu khớp 1 trust anchor → dừng, verify chữ ký intermediate bằng public key root.
5. Toàn chain hợp lệ nếu mọi chữ ký khớp + mọi cert còn hạn + không cert nào bị revoke.

**Cross-signing** — 1 CA có thể được ký bởi nhiều Root khác nhau cùng lúc (2 cert Issuer khác nhau, cùng Subject/key) để hỗ trợ giai đoạn chuyển đổi root cũ→mới mà không phá vỡ trust của client chưa cập nhật trust store.

## Config gotchas

- **Server quên gửi kèm intermediate cert** — lỗi triển khai TLS phổ biến nhất: "cert hợp lệ trên máy tôi" (vì trust store của mình đã cache sẵn intermediate) nhưng fail ở máy khác chưa từng thấy intermediate đó. Luôn cấu hình server gửi full chain (trừ root).
- **`pathLenConstraint` sai** trên intermediate → cho phép tạo sub-CA sâu hơn dự tính, hoặc ngược lại chặn nhầm 1 tầng hợp lệ.
- **Root CA hết hạn mà không ai để ý** vì thời gian sống quá dài (15-25 năm) — không nằm trong chu kỳ rotate thường xuyên nên dễ bị quên theo dõi, đến lúc gần hết hạn mới cuống.
- **Nhiều intermediate cho nhiều mục đích** (TLS server, TLS client, code signing...) là best practice — tách để khi 1 intermediate bị compromise, blast radius chỉ giới hạn ở mục đích đó.

## Security notes

- Root CA private key: **offline, air-gapped, trong HSM**, chỉ bật lên trong "key ceremony" có nhiều người chứng kiến (xem [[pki--hsm-key-protection]]).
- Compromise ở tầng nào, xử lý ở đúng tầng đó: lộ leaf key → revoke leaf. Lộ intermediate key → revoke intermediate + toàn bộ cert nó đã ký (thảm hoạ vận hành). Lộ root key → build lại toàn bộ PKI từ đầu (thảm hoạ toàn diện).
- Giới hạn quyền hạn của mỗi intermediate rõ ràng qua `pathLenConstraint` + `nameConstraints` (chỉ được ký cert cho domain/OU cụ thể) để giảm thiệt hại nếu bị compromise.

## Refs

- [[pki--x509-certificate]] — cấu trúc field Issuer/Subject dùng để build chain.
- [[ejbca--architecture-components]] — cách EJBCA triển khai Root/Sub CA thực tế (thường Root CA là 1 "External CA" hoặc CA offline riêng, Issuing CA chạy trong EJBCA).
