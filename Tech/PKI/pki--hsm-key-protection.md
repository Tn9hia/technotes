# PKI — HSM & Key Protection
Tier: 2
Parent: [[PKI]]
Related: [[pki--asymmetric-crypto]], [[pki--ca-hierarchy-trust-chain]], [[ejbca--architecture-components]]
Tags: #pki #hsm #security

## What it does

HSM (Hardware Security Module) là thiết bị phần cứng chuyên dụng để **sinh, lưu trữ, và thực hiện thao tác crypto (ký/giải mã) với private key mà không bao giờ để key rời khỏi thiết bị**. CA (đặc biệt Root CA) dùng HSM để bảo vệ private key — tài sản quan trọng nhất toàn hệ thống PKI.

## Why it exists

Nếu private key của CA nằm dạng file trên đĩa (dù mã hoá bằng password), nó vẫn có thể bị copy ra ngoài nếu server bị compromise — đến lúc đó attacker chỉ cần crack offline. HSM loại bỏ khả năng đó: thao tác ký diễn ra **bên trong** HSM, hệ điều hành/server chỉ gửi dữ liệu vào và nhận kết quả đã ký ra, không bao giờ thấy private key dạng plaintext.

## How it works (flow/diagram)

```
Server (EJBCA) muốn ký 1 cert:
  1. EJBCA gửi "hash cần ký" tới HSM (qua PKCS#11 interface hoặc
     network HSM protocol)
  2. HSM dùng private key lưu bên trong nó để ký hash đó
  3. HSM trả về chữ ký (không trả private key)
  4. EJBCA gắn chữ ký vào cert, publish ra ngoài

→ Kể cả nếu server chạy EJBCA bị chiếm quyền hoàn toàn, attacker
  cũng chỉ điều khiển được HSM "ký hộ" theo API cho phép, không lấy
  được private key để mang đi nơi khác.
```

**Key ceremony** — quy trình sinh key cho Root CA lần đầu, thường có:
- Nhiều người tham gia (M-of-N control — vd cần 3/5 người cùng có mặt mới thực hiện được thao tác nhạy cảm), giảm rủi ro 1 người duy nhất lạm quyền.
- Ghi hình, biên bản, checklist chi tiết từng bước — vì đây gần như là sự kiện **một lần duy nhất** trong vòng đời PKI (root key hiếm khi sinh lại).
- Backup key theo dạng "key shares" (chia nhỏ key material cho nhiều người giữ, không ai giữ đủ để tự khôi phục 1 mình) thay vì backup nguyên khối.

**Loại HSM:**
- **On-prem/network HSM** (vd Thales Luna, Utimaco) — thiết bị vật lý trong datacenter, kết nối qua PKCS#11/network.
- **Cloud HSM** (AWS CloudHSM, Azure Key Vault Managed HSM, GCP Cloud KMS) — dịch vụ managed, giảm gánh nặng vận hành phần cứng nhưng cần đánh giá mô hình trust với nhà cung cấp cloud.

## Config gotchas

- **HSM không được backup đúng cách trước khi decommission** — mất key vĩnh viễn nếu HSM hỏng mà không có backup/key-share hợp lệ, đồng nghĩa mất luôn khả năng vận hành CA đó.
- **Kết nối EJBCA ↔ HSM qua network HSM bị đứt** — CA không ký được cert mới (không phải lỗi EJBCA, mà lỗi kết nối HSM) — cần health check riêng cho kết nối này (xem [[ejbca--ops-runbook]]).
- **Quyền truy cập HSM quá rộng** — nếu bất kỳ ai có quyền gọi API HSM đều ký được bất cứ thứ gì, HSM chỉ còn bảo vệ được key khỏi bị *đánh cắp*, không bảo vệ khỏi *lạm dụng* — cần giới hạn theo role + audit log riêng cho thao tác ký.

## Security notes

- HSM giải quyết bài toán "key không bị lộ", nhưng **không** giải quyết bài toán "ai được phép yêu cầu HSM ký gì" — đó là trách nhiệm của access control ở tầng ứng dụng (EJBCA RBAC).
- FIPS 140-2/3 Level 3 trở lên là mức chuẩn thường yêu cầu cho HSM bảo vệ Root CA trong môi trường compliance cao.

## Refs

- [[pki--ca-hierarchy-trust-chain]] — lý do Root CA cần bảo vệ key nghiêm ngặt nhất.
- [[ejbca--architecture-components]] — cách EJBCA tích hợp HSM (PKCS#11) trong kiến trúc thực tế.
